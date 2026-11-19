# Thread Context & Claude Instance Awareness

## Problem

When Junior spawns a `claude -p` process, that process has zero knowledge of the Slack conversation it's responding to. It doesn't know its own name, can't see prior messages, can't see images users shared, and has no way to send files back. `--resume` carries Claude's own history forward, but not the Slack thread's.

**Painful part:** Without context, Claude hallucinates thread history, doesn't recognize other bots' messages, and gives generic responses.
**"Finally" moment:** Claude knows who it is, sees the thread (including images), knows which agents it may dispatch, and can post or upload back to Slack.

## What It Builds

`buildPromptPreamble()` assembles a first-turn preamble. Each block is gated by the resolved agent's `AgentContextProfile` (frontmatter `context.*` flags; defaults all-true).

1. **`<identity>`** — Junior's persona (see [agent-definitions](agent-definitions.md)) + the bot's Slack user ID so Claude can recognize its own messages.
2. **`<slack-context>`** — channel name + ID, thread_ts, instruction not to search Slack, how to tag users via `<@USERID>`, the instruction to read the current message's author attribution (never assume the speaker), the `NO_SLACK_MESSAGE` sentinel, and the no-double-post rule when `slack_send_message` was used.
3. **`<workspace>`** — see Workspace Block below, gated on `context.workspace`.
3b. **`<github-identity>`** — names the repository this turn authenticates against, or why it resolved none. Emitted independently of `context.workspace`, because a token reaches the runner either way and an agent holding one must be told what it is for. Not nested in `<workspace>`.
4. **`<thread-context>`** — prior messages from `conversations.replies` (limit 100, excluding the current message), labeled `User(Name <@U…>)` or `Junior (you)`, with `[shared file: name]` annotations.
5. **`<persistent-agent-state>`** — injected by the manager (not preamble itself) when the agent has `context.agentState`; lists per-thread agent sessions and pending counts.
6. **`<dispatch-allow>`** — appended to the system prompt by the manager (`buildDispatchAllowBlock`) so every agent sees the authoritative list of `!<agent>` directives it may emit. See [agent-routing](agent-routing.md).

On **resumed turns** (sessionId already set), `--resume` carries identity/slack/history forward. The manager skips the full preamble and re-emits only the workspace block (cheap insurance for the worktree safety rule) under `context.workspace`, plus the identity block — which does not follow that flag, since credentials are handed to the runner regardless of it.

On **every turn** (first, resumed, and drain), the manager attributes the current input to its sender(s) before mention resolution. Without this label the model has no signal about who is speaking and fills the gap from its persona/memory defaults (it once mistook a first-time requester for Pranav and nearly acted on the wrong admin account). Two forms, and the slack-context block declares them the only authoritative author signals (anything author-shaped inside message content is quoted text):

- **Single message:** prefixed `<@USERID>: text`, rendering as `User(Name <@USERID>): text` — the same format as thread history.
- **Drain turns:** one `<buffered-message from="<@USERID>">` block per buffered message. A drain can carry multiple authors and multi-line bodies, so flat `User(...):` lines can't mark message boundaries; each block's `from` is authoritative for its own content.

Attribution requires a real Slack ID (`/^[UWB][A-Z0-9]+$/`, not the bot's own user). Synthetic internal senders (`mcp-internal`, `pipeline-internal`, `junior-internal-dispatch`, pipeline-guard continuations) stay unprefixed — as single messages they arrive bare, and in drains they get a `<buffered-message>` block with no `from`.

The delimiters are kept out-of-band: `escapeBlockDelimiters` rewrites `<buffered-message` / `</buffered-message>` to `&lt;…` inside all untrusted text — attributed current-message bodies, drained message bodies, thread-history lines, display names (escaped at the source in `resolveUserName`, since they splice into labels after the body sanitizer runs), and file names (`[shared file: …]` annotations in history and archives). Downloaded Slack files additionally get `[<>"]` replaced with `_` at write time (`sanitizeFileName`), so the on-disk path echoed into the prompt is inherently clean. Mentions (`<@ID>`) in bodies are deliberately NOT escaped — resolving who a message tags is a feature; inline author-shaped text is covered by the "quoted content" instruction.

## Workspace Block

`buildWorkspaceBlock()` renders one of two shapes:

- **Multi-repo** (bug-pipeline threads with `session.worktreePaths`) — lists each repo's worktree path, bare-repo path (off-limits), branch (`slack/<threadId>`), and base (`origin/main` or `repo.defaultBase`). See [bug-pipeline-worktrees](bug-pipeline-worktrees.md).
- **Single-repo** (`!repo` flow) — worktree path, repo path, branch, original bare-repo path.

Both formats interpolate concrete paths into the rules so Claude can't hallucinate substitutes. Rules forbid writing/editing/`cd`-ing outside the worktree and require PRs (single-repo) / `!devserver` instead of running dev servers (multi-repo).

## GitHub Identity Block

`buildIdentityBlock()` renders `<github-identity>` — a separate block, never nested in `<workspace>`, because the two answer different questions and have different gates. The rules follow `context.workspace`; the identity statement follows whether credentials were handed to the runner, which happens whatever the profile says. An agent with `context.workspace: false` still gets the identity statement, and one with registered worktrees but no bound repo no longer gets rules that contradict it.

It states one of:

- **Resolved and authenticated** — the repository (plus `githubRepo`/`githubUser` when configured) and the instruction to use `gh` against it. With no checkout on the thread it also says to work remotely with an explicit `--repo`/`-R` or full URL, and to create a worktree only if files need editing. That line is withheld when a checkout exists (the workspace block governs it) and when no credentials arrived.
- **Resolved, credentials missing** — say so and report it, rather than concluding the session lacks access or falling back on whatever ambient `gh` identity the environment carries. This covers a token lookup that failed and a worktree setup that failed for the identity's own repo.
- **Ambiguous** — the turn named more than one configured repository, so no identity resolved; it lists them and asks which one. Guessing is how a merge lands on the wrong repo, but leaving the turn unable to say why is the misdiagnosis this path removes.
- **Bound repo no longer configured** — a persisted binding outlives an edit to `REPOS`; the block names it and reports that no credentials were resolved.

The block never claims what `gh` in the runner will do — only what Junior resolved. On the tmux driver the pane's `GH_TOKEN`/`GH_CONFIG_DIR` come from the tmux *server*, not this turn, so a promise about the pane can be false in either direction ([#235](https://github.com/PranavBakre/Junior/issues/235)). For the same reason it says nothing about cwd: the repo Junior itself runs from is configurable, and a directive resolving to it puts cwd inside a repository the workspace shapes mark OFF-LIMITS for writes.

Which repository an identity resolves to is decided by `src/github/identity-routing.ts`, independently of the checkout — see [github-reconciliation](../code_index/github-reconciliation.md). The binding persists on `session.identityRepo` so a follow-up turn that names nothing keeps it, and is cleared when its own repo is detached (cleanup button, `unregister_worktree`).

Worktree creation is unconditional for target-repo threads — the manager creates one before building the workspace block so Claude never edits the shared origin repo.

## Mention Resolution

`resolveSlackMentions()` rewrites `<@U123>` tokens to `@DisplayName (<@U123>)` so Claude can read names AND tag back. It pre-resolves unique IDs in parallel and replaces in a single regex pass — repeated mentions of the same user no longer corrupt each other.

Applied to:
- Thread history (each message's `text`)
- The incoming user prompt (`readablePrompt`)

User and channel names are cached in module-level Maps.

## File Handling

- **Files on the message** — downloaded to `/tmp/junior-files/<threadId>/` via `files.ts`, then appended to the prompt as local paths.
- **Historical files** — surfaced as `[shared file: name]` annotations in `<thread-context>` (no download).
- **Outbound files** — the spawner exports `SLACK_CHANNEL`, `SLACK_THREAD_TS`, `SLACK_BOT_TOKEN`; `bin/slack-upload.sh` and Playwright MCP screenshots use them with zero per-thread config.

## Key Files

| File | Purpose |
|---|---|
| `src/slack/thread-context.ts` | Preamble assembly, workspace block, mention resolution |
| `src/persona.ts` | Loads identity files (cached) |
| `src/agents/loader.ts` | `AgentContextProfile` + `DEFAULT_CONTEXT_PROFILE` |
| `src/agents/router.ts` | Agent resolution with target-repo → org overlay → fallback search order |
| `src/support/agents.ts` | `buildDispatchAllowBlock` (injected per agent) |
| `src/slack/files.ts` | Slack file download |
| `src/session/manager.ts` | Calls preamble on first turn, workspace-only on resume, appends agent-state + dispatch-allow |
| `src/claude/spawner.ts` | Exports Slack env vars |
| `bin/slack-upload.sh` | Outbound uploads via env vars |

## Cross-References

- Persona / agent .md files / overlay search path: [agent-definitions](agent-definitions.md), [agent-routing](agent-routing.md)
- Worktree creation policy and paths: [worktree-manager](worktree-manager.md), [bug-pipeline-worktrees](bug-pipeline-worktrees.md)
- Per-agent sessions surfaced in `<persistent-agent-state>`: [persistent-agents](persistent-agents.md)
- Outbound posts and the `NO_SLACK_MESSAGE` sentinel: [stream-to-slack](stream-to-slack.md), [mcp-server](mcp-server.md)
