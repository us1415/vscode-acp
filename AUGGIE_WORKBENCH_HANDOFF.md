# Auggie Workbench Handoff

Last updated: 2026-07-19

## Repository

Work in:

```text
C:\Users\us141\Documents\codebase\vscode-acp
```

Important: some previous tool sessions had a stale cwd pointing at `AugmentCode-Free`. Always verify the cwd before editing.

## Product Goal

Build a personal Augment-like VS Code sidebar/workbench for Auggie CLI over ACP.

Boundary: reproduce visible workflow and user experience with our own implementation. Do not copy proprietary extension code. Public Auggie docs and the public `augmentcode/auggie` repo are okay as behavior/reference sources.

## User Preferences

- Explain VS Code/dev-host steps very plainly.
- Keep progress in `AUGGIE_SESSION_NOTES.md`.
- Keep the checklist in `AUGGIE_WORKBENCH_TODO.md`.
- Favor bigger functionality over tiny UI nits.
- Visible terminal execution is higher priority than polishing command cards.
- User usually reviews code changes through VS Code Git, so Edits tab is useful but not the core workflow.
- Keep the inferred edit bridge documented; it is intentional fallback behavior.

## Key Files

- `src/ui/ChatWebviewProvider.ts`: webview provider and extension-to-webview state bridge.
- `media/chatWebview.js`: main webview UI script.
- `src/core/SessionManager.ts`: ACP session lifecycle, session load/resume/new, MCP server attachment.
- `src/config/AgentConfig.ts`: agent config plus MCP server normalization.
- `src/handlers/TerminalHandler.ts`: ACP terminal support and visible-command runner.
- `src/core/TerminalMcpBridge.ts`: localhost bridge that lets MCP call the VS Code terminal runner.
- `scripts/auggie-terminal-mcp.js`: stdio MCP helper exposing terminal tools to Auggie.
- `AUGGIE_SESSION_NOTES.md`: running session notes.
- `AUGGIE_WORKBENCH_TODO.md`: project checklist.

## What Is Done

### Core Auggie Workbench

- Extension branded as Auggie Workbench.
- Default agent command is `npx @augmentcode/auggie@latest --acp`.
- Chat webview moved into `media/chatWebview.js`, fixing the old VS Code `document.write` webview parse failure.
- Latest Auggie conversation auto-restores on dev-host startup.
- Main sidebar shell has:
  - Thread / Tasks / Edits tabs
  - connected agent header
  - Augment-like composer controls
  - model/mode-ish lower controls

### Composer / Context

- `@` menu opens an Augment-style context menu.
- Context chips can be added and removed.
- `+` attaches files.
- Selected-code button adds editor selection or shows no-selection state.
- New thread uses a warning flow.

### Tasks

- ACP plan updates populate Tasks tab.
- Tasks persist through extension host reload using webview state and extension workspaceState.
- Fallback parser can recover task titles from assistant text if plan updates are not replayed.
- Fixed hyphenated task title truncation.

### Activity Cards

- Tool calls render as expandable action cards.
- Inline working indicator shows elapsed time and recent tool activity.
- `agent_thought_chunk` is shown only if Auggie actually emits it; no fake hidden reasoning.

### Edits

- Edits tab shows changed files.
- Git-backed file list and line deltas via `git diff --numstat HEAD --`.
- Expandable inline diff previews.
- Open and Diff buttons work.
- Inferred edit rows remain as fallback when tool-call data implies an edit before Git-backed state catches up.

### MCP Config

- `acp.mcpServers` accepts both:
  - ACP-style array
  - Auggie-style object keyed by server name
- Env can be object or `{ name, value }` array.
- `${workspaceFolder}` expands in `command`, `args`, and `url`.

### Visible Terminal MCP Bridge

This is the biggest recent milestone.

Implemented:

- `TerminalMcpBridge` starts a localhost-only HTTP bridge on `127.0.0.1` with a random bearer token.
- `scripts/auggie-terminal-mcp.js` exposes an MCP tool named `run_command_in_vscode_terminal`.
- `SessionManager` appends this built-in MCP server to Auggie sessions.
- `TerminalHandler.runVisibleCommand` executes via VS Code shell integration first, falling back to the existing spawn/pseudoterminal path.

Important fixes:

- First smoke test failed because the MCP helper timed out during startup.
- The helper now accepts:
  - CRLF `Content-Length` framing
  - LF-only `Content-Length` framing
  - newline-delimited JSON framing

Successful user smoke test:

- Prompt:

```text
Use the run_command_in_vscode_terminal MCP tool to run node --version.
```

- Result:
  - Auggie called `run_command_in_vscode_terminal_auggie-vscode-terminal`.
  - VS Code opened a visible terminal named `Auggie: node`.
  - Terminal ran `node --version`.
  - Auggie returned `v22.14.0`.

This confirms the full path works:

```text
Auggie -> MCP helper -> extension localhost bridge -> VS Code terminal -> captured output -> Auggie
```

## Latest Verification

After MCP helper startup-timeout fix:

```text
node --check scripts\auggie-terminal-mcp.js
npm run compile
npm run lint
```

All passed.

After the final user smoke test, no additional code verification was run because no code changed after the smoke test except documentation/checklist notes.

## Known Caveats

RESOLVED since 2026-07-04 (kept for history):

- ~~Auggie only used the terminal tool when explicitly asked by exact tool name.~~ Alias tools were added, and as of 2026-07-19 the extension also ships `rules/visible-terminal.md` injected via `--rules`, so Auggie chooses the visible terminal unprompted (verified: `TERM_PROGRAM=[vscode]`).
- ~~MCP bridge availability does not force Auggie away from `launch-process`.~~ Still true at the protocol level, but the bundled rule now steers it and tells it to STOP AND ASK rather than fall back invisibly. Tool choice remains model-driven, so keep regression smoke tests.
- ~~Keep/discard Edits actions are still pending.~~ Implemented (`Discard All` still untested against non-disposable changes).
- ~~Recent conversation tree is still pending.~~ Implemented.
- ~~Packaging should be checked.~~ Verified; `media/`, `scripts/`, and `rules/` are included in the VSIX.

Still open:

- The terminal screenshot showed the project venv activating after the command. Only investigate if that extra terminal output becomes noisy.
- Auggie can emit the SAME command twice within one turn (observed with `git status`). The turn-scoped duplicate-command guard is designed but not built.
- If shell integration is unavailable the extension falls back to a hidden process; this is now loudly surfaced (toast + banner + `Auggie (hidden):` tab) but Ctrl+C and shell history still do not work in that mode.

## Best Next Steps

Items 1-5 from the 2026-07-04 version of this list (terminal MCP alias tools, terminal/action cards, Edits polish, session/thread tree, package smoke test) are all DONE. Current list:

1. Turn-scoped duplicate-command guard (top priority; designed, not built).
   - Auggie can emit the same command twice in one turn (two `tool_call` events, two terminals).
   - Suppress a duplicate identical command (same command+args+cwd) WITHIN one agent turn; return the first result for the repeat.
   - ANY new user message resets the turn scope, so a deliberate "check it again" always re-runs.
   - Needs turn-awareness wired from the ACP session/prompt layer to `TerminalMcpBridge`; an in-flight check alone is not enough (the observed duplicate was sequential).

2. Housekeeping.
   - Delete/archive old milestone branches (`auggie-package-smoke` and the older stacked branches); `main` is the baseline.
   - Move the untracked screenshot zips out of the repo root.
   - Re-package and install-smoke a fresh VSIX (current artifact `0.2.3` predates the 2026-07-19 hardening pass).

3. More test seams (the MCP alias test landed 2026-07-19).
   - Action-card parsing.
   - Changed-file snapshot parsing.
   - A regression smoke checklist for the terminal-routing contract after Auggie CLI updates.

4. Split `ChatWebviewProvider.ts` (~2900 lines) before growing slash-menu/message-rendering/visual-polish surface.

5. Deliberately scope checkpoints/revert before implementing.

## Instructions For The Next Agent

Tell the next agent:

```text
We are continuing the Auggie Workbench project in C:\Users\us141\Documents\codebase\vscode-acp.

First read:
- AUGGIE_WORKBENCH_HANDOFF.md
- AUGGIE_SESSION_NOTES.md
- AUGGIE_WORKBENCH_TODO.md

Do not work in AugmentCode-Free. Verify cwd is vscode-acp.

Current state: all work is on `main`. The visible-terminal path is working end to end
- Auggie chooses the visible VS Code terminal on its own (bundled rules/visible-terminal.md
injected via --rules), output is sanitized in the action cards, and a fallback to hidden
execution is loudly surfaced. Test suite is 21 passing.

The current priority is the turn-scoped duplicate-command guard: Auggie can emit the
same command twice within one turn. Suppress within-turn duplicate identical commands
and reset the scope on any new user message, so a deliberate "check it again" still
re-runs. This needs turn-awareness from the ACP session layer down to TerminalMcpBridge.

Workflow expectations:
- Run npm test (compile + lint + tests) before proposing anything as done.
- Do NOT push to origin/main until the user has smoke-tested in the F5 dev host;
  automated tests passing is necessary but not sufficient.
- Keep commits small and separated by logical change.
- At the end of the session, add a dated entry to the Session Log in
  AUGGIE_SESSION_NOTES.md, tick off completed items in AUGGIE_WORKBENCH_TODO.md,
  and update this handoff if priorities changed.
```

