# agent-line-feed-guard

Hook-only plugin — no binary, no MCP server. `hooks/hooks.json` wires three events to a single script (`scripts/line-feed-guard-hook.mjs`) that converts CRLF to LF in the files Claude Code parses: `.claude/{agents,skills,commands,hooks}/**`, `.claude/*.json`, and root `CLAUDE.md` / `AGENTS.md` / `CLAUDE.local.md` / `.mcp.json`, in the project cwd and in the user config dir. It also ships two skills: `line-feed-guard` (preventive — the rule, before the agent writes) and `scan` (on-demand sweep, `--scan` CLI mode).

## Contracts an agent won't infer from the tree

- **Release is orphan-branch + marketplace dispatch.** `release.yml` (manual: Actions → release → `version=X.Y.Z`) stamps the version into both manifests, force-pushes an orphan `release` branch holding only install-ready files, tags `agent-line-feed-guard--vX.Y.Z`, and POSTs a dispatch (`category: hook`) to `Seretos/agent-marketplace`. `main` and `release` share no history.
- **Required secret:** `MARKETPLACE_DISPATCH_TOKEN` — fine-grained PAT, `Contents: RW` + `Pull requests: RW` on `Seretos/agent-marketplace` only.
- **`assets/icon.png` and `description.md` are release artifacts, not just repo files.** The dispatch payload sends `raw.githubusercontent.com/${repo}/${TAG}/…` URLs for both, so they must live on the orphan `release` branch at the tagged commit — that is why `release.yml`'s stage step copies them. Replace the placeholder icon and fill in `description.md`'s Key Features before cutting v0.0.1.

## Why the design looks like this

- **Event confidence is tiered, and the tiers are load-bearing.**
  - *Tier 1* — `PostToolUse` and `SessionStart`. Publicly documented, and the plugin's guarantees rest on them.
  - *Tier 2* — `FileChanged`. Verified present in the shipped Claude Code binary (2.1.222) but publicly undocumented. It catches CRLF from writes no tool call made: an external editor, `git checkout`, another terminal. It is deliberately built so that if it silently stops firing, nothing breaks — the next `SessionStart` or `Bash` call catches the same file. Do not move any guarantee onto it.
- **`FileChanged` must be configured without a `matcher`.** The binary derives watch paths from each `FileChanged` hook's `matcher` split on `|` and joined to cwd — a matcher would register junk watch paths. Both code paths handle an absent matcher (it then matches everything), so omitting it is correct and intended.
- **`FileChanged` cannot talk to the model.** Its `hookSpecificOutput` union member carries only `watchPaths` — there is no `additionalContext`. Top-level `systemMessage` is its only channel, which is why that branch emits `systemMessage` alone and no `hookSpecificOutput` at all.
- **`hookEventName` must match the firing event exactly.** The hook runner throws `Hook returned incorrect event name` otherwise. This is why the three branches never share an output builder.
- **There is deliberately no `git status` and no mtime window.** The scope is not "what changed" — it is a statically known set of a few dozen small files. A git spawn costs 20–150 ms on Windows plus an index refresh, more than reading the whole scope (1–3 ms); it never reports gitignored files such as `.claude/settings.local.json`; it does not work outside a working tree; and it answers the wrong question, since only the bytes on disk say what has CRLF *now*. Dropping it also removes the entire mtime/state apparatus and every loop risk from the hook's own writes. **Do not reintroduce it as an "optimisation" — it is a pessimisation here.**
- **Idempotence is the loop-prevention mechanism.** `stripCrlf` returns `null` when there is no CRLF and the caller writes only on a non-null result, so a second pass is a no-op with empty stdout. The `FileChanged` self-trigger therefore terminates in one step.
- **Writes are plain `writeFileSync`, not temp-file + rename — on purpose.** A rename replaces the inode: it would break symlink identity (this workspace symlinks `mcp-test/plugins/*` into `plugins/`), drop explicit ACLs, and fail with `EPERM`/`EBUSY` on Windows when an editor holds a share-deny handle. The files are well under 100 KB, so the truncate window is microseconds.
- **The user config dir is scanned narrowly and watched even more narrowly.** Only `{agents,skills,commands}`, top-level `*.json` and `CLAUDE.md` — never `projects/`, `file-history/`, `shell-snapshots/` or `plugins/`. Those churn constantly and would make both the scan and the chokidar watcher expensive. The scan bound is what keeps this plugin viable inside Unity projects and `node_modules` trees: it descends only into `.claude/`, which neither of those lives under.
- **UTF-16 is reported but never transcoded.** PowerShell 5.1's `Out-File` and `>` default to UTF-16LE, which Claude Code also cannot read — same symptom, different cause. Surfacing it is cheap; transcoding is out of scope for a line-ending guard and would be a much riskier write.
- **`systemMessage` is spent only on `agents/**` fixes.** Agent definitions are read once at startup, so a silent failure there survives the whole session and is worth interrupting the user for. Everything else stays agent-facing. That is the entire noise budget.
- **Unverified, and the messages reflect it:** whether SessionStart's `reloadSkills` also re-reads the *subagent* registry. Its docstring names only "skill and command directories". The plugin therefore promises nothing and tells the user to restart.

## Testing

`scripts/line-feed-guard-hook.test.mjs` — plain Node, no framework, no dependencies, `fs.mkdtempSync` per test, exit 0 = pass.

**Every end-to-end test must inject `CLAUDE_CONFIG_DIR` and `LINE_FEED_GUARD_STATE_DIR` into a sandbox** (`sandboxEnv()` does this). Without it the hook enumerates — and rewrites — the real `~/.claude` tree. The symlink-recursion test skips itself when symlink creation is unprivileged, which is the normal case on Windows without Developer Mode.
