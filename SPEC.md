# AgentOS — Build Specification (v1)

A private, single-user command center for AI agents, running on one workstation.
Inspired by the architecture analysed in [`agentos-teardown.md`](./agentos-teardown.md). That document is background only. **This spec is the source of truth.**

---

## 0. Decisions (fixed for v1)

| Topic | Decision |
|---|---|
| Users | Exactly one (the owner). No multi-tenancy. |
| Prerequisites | Node LTS, pnpm, git, tmux (`brew install node pnpm git tmux`), Xcode Command Line Tools (needed to build `node-pty`), the agent CLIs installed and logged in |
| Network exposure | **Loopback only** (`127.0.0.1`). Never public. Remote access, if ever wanted, goes through Tailscale or SSH tunnel later. |
| OS | **macOS** (Apple Silicon assumed) is the only target that must work. Keep code POSIX-friendly so Linux works with little effort; Windows is not a goal. |
| Subscription agents | **Claude Code**, **Antigravity**, **Hermes Agent**, driven by spawning their **official CLIs** under the user's own login. The app never reads, copies or proxies their OAuth tokens. |
| API-key providers | **OpenRouter**, **DeepSeek**, any other **OpenAI-compatible** broker, via one generic HTTP adapter. |
| Local models | **Ollama** and **LM Studio**, via their OpenAI-compatible endpoints (same generic adapter). |
| v1 features | 1. Chat with agents · 2. Multi-agent room · 3. Pipeline / Kanban · 4. Terminal |
| Not in v1 | Voice, media/content studios, SEO tools, preview iframes for generated apps, mobile PWA, scheduled jobs |

---

## 1. Architecture

```
┌────────────────────────── Workstation (single user) ──────────────────────────┐
│                                                                               │
│  Browser ──HTTP/SSE/WS (127.0.0.1:7777, token auth)──▶  agentosd (daemon)     │
│  (React SPA)                                            Node + Fastify        │
│                                                          │                    │
│                         ┌────────────────────────────────┼──────────────┐     │
│                         ▼                ▼               ▼              ▼     │
│                   RunManager       RoomOrchestrator  BoardOrchestrator  TerminalManager
│                         │                │               │              │     │
│                         └──── AgentAdapters ─────────────┘          node-pty  │
│                              │        │          │                  (+tmux)   │
│              claude CLI  hermes CLI  antigravity  OpenAI-compatible HTTP      │
│              (spawn)     (spawn)     (spawn/pty)  OpenRouter · DeepSeek ·     │
│                                                   Ollama · LM Studio · …      │
│                                                                               │
│  State: SQLite (~/.agentos/agentos.db) · Vault (Obsidian-compatible markdown) │
│  Code work: git worktrees under ~/.agentos/worktrees/<project>/<card>         │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Why a separate daemon (and not Next.js route handlers like the original)

- **Runs must outlive the HTTP request and the browser tab.** A 20-minute builder run can't die because the tab was reloaded or a dev server hot-reloaded. The daemon owns every run, PTY and orchestrator loop. The browser only *subscribes*.
- **Native WebSockets** for the terminal, with no SSE+POST workaround.
- **One long-lived process** holding in-memory state (PTY buffers, run registry, queues), with SQLite for durability.
- The UI is a static SPA served by the daemon. Nothing needs SSR.

### 1.2 Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Runtime | Node.js current LTS, TypeScript `strict` | pnpm workspaces monorepo |
| Daemon HTTP | Fastify + `@fastify/websocket` + `@fastify/static` | |
| Validation | zod at every boundary (HTTP bodies, config, adapter output, planner/reviewer JSON) | Schemas live in `packages/shared` |
| DB | SQLite via `better-sqlite3`, plain SQL migrations | WAL mode. FTS5 for vault search. |
| Process spawn | `execa` + process-group kill (`detached: true`, `process.kill(-pid)`) | **Never** `shell: true` with user input. Resolve CLIs via the user's login-shell `PATH` (see §9 note) |
| PTY | **tmux** (default backend) attached through `node-pty`; plain `node-pty` fallback | `brew install tmux` is a prerequisite |
| Logging | `pino` (JSON) to `~/.agentos/logs/` | Redact secrets |
| Frontend | Vite + React + TypeScript, Tailwind CSS v4, Radix primitives (shadcn/ui-style components), `cmdk`, TanStack Query, `zustand` (UI state only) | |
| Terminal UI | `@xterm/xterm` + `@xterm/addon-fit` + `@xterm/addon-web-links` | |
| Markdown | `react-markdown` + `remark-gfm` + a syntax highlighter | |
| Prompt templates | Handlebars (`{{var}}`, `{{#if}}`) | Templates in `prompts/*.md`, rendered in the daemon |
| Diff view | `diff2html` or Monaco diff editor | For the review stage |
| Tests | Vitest (unit/integration), Playwright (smoke e2e) | |

### 1.3 Repository layout

```
apps/
  daemon/            # agentosd
    src/
      server.ts            # Fastify bootstrap, auth hooks
      config.ts            # load + validate agentos.config.yaml
      db/                  # migrations, queries
      adapters/            # one file per adapter kind + registry
      runs/                # RunManager, event store, SSE
      room/                # RoomOrchestrator
      board/               # BoardOrchestrator, git worktree helpers, checks
      terminal/            # TerminalManager (node-pty / tmux)
      vault/               # markdown writer, FTS indexer, search
      routes/              # HTTP + WS routes
    test/fixtures/         # recorded CLI outputs (ndjson/txt)
  web/               # React SPA
packages/
  shared/            # zod schemas + TS types: events, API DTOs, config
prompts/             # editable system prompts loaded at runtime
config/
  agentos.config.example.yaml
```

---

## 2. Security baseline (applies to every milestone, starting at M0)

Local-only is **not** the same as safe. Any web page you visit can fire requests at `localhost`, and this daemon can spawn shells. All of these are required:

1. **Bind `127.0.0.1` only.** Refuse to start on `0.0.0.0` unless `--unsafe-bind` is passed explicitly, and log a warning if it is.
2. **Access token.** On first start, generate a 256-bit random token, saved to `~/.agentos/token` with `0600` permissions. The daemon prints a one-time URL (`http://127.0.0.1:7777/auth?t=…`), Jupyter-style, which sets an `HttpOnly; SameSite=Strict` cookie. Every HTTP route and the WS upgrade require that cookie or an `Authorization: Bearer` header.
3. **DNS-rebinding protection.** Reject any request whose `Host` isn't `127.0.0.1:<port>` or `localhost:<port>`.
4. **CSRF / cross-origin.** For state-changing requests and WS upgrades, require `Origin` to match the daemon's own origin. No CORS headers at all.
5. **Secrets stay server-side.** API keys are read from the **macOS Keychain** (`security find-generic-password -s agentos -a <ENV_NAME> -w`), falling back to `~/.agentos/.env` (`0600`). They are never sent to the browser and are redacted in logs and run events.
6. **Agent permissions are explicit.** Each agent has `readonly` and `build` mode profiles in config (§4.4). The room always uses `readonly`. Only the board's builder and explicit chat sessions marked **Build** may write files.
7. **Code changes are isolated.** Board builds run in **git worktrees on a dedicated branch**, never in the user's checkout. Merging is a human click.
8. **No dangerous defaults.** Never pass flags that skip all permission checks (e.g. Claude's `--dangerously-skip-permissions`) unless the user opts in per agent in config, and the UI shows a red badge when they do.

---

## 3. Configuration

File: `~/.agentos/agentos.config.yaml` (example in `config/agentos.config.example.yaml`). Validated with zod at startup. Invalid config means the daemon refuses to start and prints a clear error.

```yaml
server: { host: 127.0.0.1, port: 7777 }
dataDir: ~/.agentos
vault:
  dir: ~/Obsidian/MyVault/AgentOS   # a subfolder of an existing vault is fine
projects:
  - id: firstrepo
    name: firstrepo
    path: ~/code/firstrepo
    defaultBranch: main
    checks:                         # run after each build, all must exit 0
      - pnpm install --frozen-lockfile
      - pnpm typecheck
      - pnpm test
terminal: { backend: tmux }       # tmux | pty
board:
  plannerAgent: claude
  builderAgent: claude
  reviewerAgent: deepseek          # must differ from builderAgent (validated)
  maxParallelBuilds: 1             # subscription CLIs have usage limits; start low
  maxIterations: 3
  passScore: 80
room: { defaultJudge: claude, maxRounds: 1, contextMessages: 20 }
agents: [ ... ]                    # see §4.4; full example in config/agentos.config.example.yaml
```

Hot reload of the config is **not** required for v1 (a restart is fine). There is a "Reload config" button that restarts the adapter registry.

---

## 4. Agent adapters

### 4.1 Normalized event schema (`packages/shared/src/events.ts`)

Every adapter emits only these events. The UI, room and board understand only these.

```ts
export type AgentEvent =
  | { type: "run.started"; nativeSessionId?: string; model?: string }
  | { type: "text.delta"; text: string }
  | { type: "text.done"; text: string }              // full assistant text for the turn
  | { type: "thinking.delta"; text: string }
  | { type: "tool.call"; id: string; name: string; input: unknown }
  | { type: "tool.result"; id: string; output: string; isError?: boolean }
  | { type: "stderr"; text: string }
  | { type: "usage"; inputTokens?: number; outputTokens?: number; costUsd?: number; costIsEstimate?: boolean }
  | { type: "run.finished"; status: "ok" | "error" | "cancelled"; error?: string; exitCode?: number };

export interface RunEventEnvelope { runId: string; seq: number; ts: string; event: AgentEvent }
```

### 4.2 Adapter interface (`apps/daemon/src/adapters/types.ts`)

```ts
export type RunMode = "chat" | "readonly" | "build";

export interface RunRequest {
  prompt: string;
  history?: { role: "user" | "assistant"; content: string }[]; // for stateless adapters
  nativeSessionId?: string;     // for adapters that support resume
  cwd: string;                  // project path or worktree
  systemPrompt?: string;        // appended to the agent's own system prompt
  mode: RunMode;
}

export interface ProbeResult { ok: boolean; version?: string; detail?: string; latencyMs: number }

export interface AgentAdapter {
  readonly id: string;
  readonly capabilities: { resume: boolean; tools: boolean; headless: boolean; reportsCost: boolean };
  probe(): Promise<ProbeResult>;
  run(req: RunRequest, signal: AbortSignal): AsyncIterable<AgentEvent>;
}
```

### 4.3 Adapter kinds

| `kind` | Used for | Transport | Session continuity |
|---|---|---|---|
| `claude-cli` | Claude Code (subscription) | spawn `claude` in headless mode, parse `stream-json` | `--resume <session_id>` captured from the init event |
| `cli-json` | Any CLI with a JSON-lines output mode (configurable field mapping) | spawn, parse NDJSON | if the CLI supports resume, otherwise history is prepended |
| `cli-text` | CLIs with only plain-text headless output | spawn; stdout → `text.delta`, stderr → `stderr` | last N turns prepended to the prompt |
| `pty` | CLIs with **no** headless mode | not a chat adapter: "Open in terminal" launches it in a Terminal tab | n/a |
| `openai-compatible` | OpenRouter, DeepSeek, other brokers, Ollama, LM Studio | HTTP `POST {baseUrl}/chat/completions`, `stream: true`, SSE | daemon keeps history in SQLite |

#### `claude-cli` details

- **Command (verify against the installed version first):**

  ```
  claude -p --output-format stream-json --verbose --include-partial-messages \
         [--resume <id>] --permission-mode <mode> [--allowedTools …] [--append-system-prompt …]
  ```

  Pass the prompt on **stdin** rather than argv, which avoids argument-length limits and quoting problems.
- **Event mapping:**
  - `system` (subtype `init`) → `run.started` (`session_id`, `model`)
  - `stream_event` with a text delta → `text.delta`; with a thinking delta → `thinking.delta`
  - `assistant` message → `tool.call` for each `tool_use` block, and `text.done` for the text blocks
  - `user` message with `tool_result` blocks → `tool.result`
  - `result` → `usage` (`total_cost_usd`, token usage; `costIsEstimate: true` on subscription), then `run.finished` (`is_error` → `error`)
- **Mode mapping:**
  - `readonly` → `--permission-mode plan`
  - `build` → `--permission-mode acceptEdits` plus the agent's `allowedTools` from config
- **Process handling:**
  - Unknown event types are logged and ignored, never fatal.
  - Cancel = SIGTERM to the process group, SIGKILL after 5 s, then `run.finished{status:"cancelled"}`.

#### Hermes Agent and Antigravity

The exact headless flags of these CLIs are **not known yet and must not be guessed.** In milestone M2:
1. Run `<cli> --help` (and its subcommands' help) locally and paste the output into `apps/daemon/test/fixtures/<cli>/help.txt`.
2. Pick the best kind, in this order: `cli-json` (structured output available), then `cli-text`, then `pty`.
3. Record real outputs as fixtures, then write the mapping and tests.

Until then both are configured as `pty`, so they're usable straight away from the Terminal.

#### `openai-compatible` details

- **Request:** `{ model, messages, stream: true, stream_options: { include_usage: true } }`. Omit `stream_options` if the endpoint rejects it (configurable flag).
- **Streaming:**
  - Parse SSE `data:` lines and stop on `[DONE]`.
  - `choices[0].delta.content` → `text.delta`
  - `delta.reasoning_content` or `delta.reasoning` → `thinking.delta` (DeepSeek, OpenRouter reasoning models)
  - The final `usage` object → `usage`
- **History:** the system prompt plus history trimmed to `maxContextTokens` (approximate as chars/4) plus the new prompt.
- **Tools:** none in v1. These are chat-only models. Room and board treat them as readonly participants.
- **Health probe:**
  - Ollama / LM Studio: `GET {baseUrl}/models`
  - Cloud: `GET {baseUrl}/models` with the key, or a 1-token completion if `/models` isn't supported
- **OpenRouter:** sends `X-Title: AgentOS` (optional `HTTP-Referer`).

### 4.4 Agent config entries

```yaml
agents:
  - id: claude
    name: Claude Code
    kind: claude-cli
    command: claude
    color: "#e07a5f"
    modes:
      readonly: { permissionMode: plan }
      build:
        permissionMode: acceptEdits
        allowedTools: ["Read", "Edit", "Write", "Glob", "Grep", "Bash(git status:*)", "Bash(git diff:*)", "Bash(pnpm test:*)"]

  - id: hermes
    name: Hermes Agent
    kind: pty                 # upgrade to cli-json / cli-text in M2 after verifying flags
    command: hermes
    color: "#8b5cf6"

  - id: antigravity
    name: Antigravity
    kind: pty                 # upgrade in M2 after verifying flags
    command: antigravity      # verify actual binary name
    color: "#22c55e"

  - id: openrouter-default
    name: OpenRouter · <model>
    kind: openai-compatible
    baseUrl: https://openrouter.ai/api/v1
    apiKeyEnv: OPENROUTER_API_KEY
    model: "<provider/model-slug>"
    color: "#38bdf8"

  - id: deepseek
    name: DeepSeek
    kind: openai-compatible
    baseUrl: https://api.deepseek.com
    apiKeyEnv: DEEPSEEK_API_KEY
    model: "<deepseek-model-id>"
    color: "#3b82f6"

  - id: ollama
    name: Ollama · <model>
    kind: openai-compatible
    baseUrl: http://127.0.0.1:11434/v1
    model: "<local-model>"
    color: "#f59e0b"

  - id: lmstudio
    name: LM Studio · <model>
    kind: openai-compatible
    baseUrl: http://127.0.0.1:1234/v1
    model: "<local-model>"
    color: "#ec4899"
```

Adding a new OpenAI-compatible broker is a config change only, no code.

### 4.5 Mock adapter (required)

`kind: mock` emits a scripted sequence of events with delays, configurable to fail or hang. All UI, room and board tests use it, so **nothing in CI or tests ever spends tokens or needs a login.**

---

## 5. Runs (the backbone)

- A **run** is one invocation of one adapter. Chat turns, room replies, planner, builder and reviewer calls, and check executions are all runs.
- `RunManager.start(agentId, RunRequest, meta)` returns a `runId` **immediately**. Execution happens in the daemon.
- Every event is persisted to `run_events (run_id, seq, ts, type, payload)` **before** fan-out.
- **Streaming to the browser:** `GET /api/runs/:id/events` (SSE). It honours `Last-Event-ID` and replays from `seq`. Reconnects, reloads and late subscribers all get the full stream.
- **Cancel:** `POST /api/runs/:id/cancel`.
- **Timeouts:** per-run wall-clock timeout (config, default 30 min for build, 5 min otherwise). Idle-output watchdog (default 5 min with no events) marks the run as errored.
- **On daemon start:** any run still `running` in the DB is marked `error: "daemon restarted"`.
- **Cost ledger:** `usage` events are aggregated per agent per day and shown on Mission Control. Subscription costs are labelled "estimated".

---

## 6. Feature 1 — Chat with agents

### Behaviour
- **Session list** (left): grouped by agent. Each session is pinned to **one agent** and **one project** (cwd), or "no project" (a scratch dir in `~/.agentos/scratch/<session>`).
- **Mode toggle** per session: **Chat** (default) / **Build**. Build is only available for `claude-cli` and verified tool-capable CLI kinds, and it shows a warning badge.
- **Sending a message:**
  1. `POST /api/sessions/:id/messages {text}` → `{runId}`.
  2. The UI subscribes to the run's SSE.
  3. On finish, the assistant message is stored.
- **Transcript rendering:**
  - Markdown with code highlighting.
  - **Collapsible tool calls** (name, input, result, error state).
  - Thinking shown collapsed.
  - Live token stream with a cancel button.
- **Continuity:** `claude-cli` reuses `nativeSessionId`, and HTTP adapters get trimmed history.
- **Vault export:** each session is mirrored to `vault/Chats/<YYYY-MM-DD> <title>.md` with front-matter (`agent`, `sessionId`, `project`, `created`), updated after each turn.
- **"Send to…" actions on a message:** Room (start a room with this as the opener), Board (create an Inbox card), Vault (save as a note).

### Acceptance (M1)
- [ ] Chat with `mock`, `ollama` (or `lmstudio`), `openrouter-default`, `deepseek` and `claude` works end to end with live streaming.
- [ ] Reloading the page mid-stream resumes the same run without losing or duplicating text.
- [ ] Cancel stops the process (no orphan `claude` process left, checked with `ps` / Task Manager).
- [ ] A second message in a Claude session resumes the same native session.
- [ ] Transcript appears in the vault as markdown.
- [ ] Parser tests pass against recorded `claude` fixtures.

---

## 7. Feature 2 — Multi-agent room

### Concepts
- A **room** has participants (a subset of agents, each with an optional persona line) and a **mode**.
- **Modes:**
  - **Riff (sequential).** For each user message, the targeted participants reply **in order**. Each one sees the transcript so far, including replies earlier in the same round.
  - **Council (parallel + judge).** All participants answer **independently and in parallel**. Then a designated **judge** agent synthesizes a verdict: agreements, disagreements, recommendation, confidence.
- **Targeting:** `@id` mentions restrict a turn to the mentioned agents. No mention means all participants.
- **Rounds:** `maxRounds` (default 1, max 3). With `autoContinue`, if a reply @mentions another participant, that one replies next, until `maxRounds` or a per-turn budget of N replies (default 6).

### Mechanics
- Every reply is a run in `readonly` mode, with the `prompts/room.md` system prompt filled with: room title, own name, other participant names, persona, and "reply concisely (≤ 150 words unless asked), add something new, disagree when warranted, address others with @name".
- **Context for each participant:** the last N messages of the room (configurable, default 20, trimmed by token budget) plus any pinned vault notes.
- **Stream:** `GET /api/rooms/:id/stream` (SSE). Events: `turn.started`, `agent.typing{agentId}`, `agent.delta{agentId,text}`, `agent.message{agentId,messageId}`, `agent.error{agentId}`, `judge.message`, `turn.finished`.
- **Failures:** one participant failing or timing out never blocks the others. It shows an inline error chip.
- **Message actions:** "Save to vault", "Create card" (→ board Inbox, pre-filled with the message plus the user's prompt).
- **Vault mirror:** `vault/Rooms/<YYYY-MM-DD> <title>.md`.

### Acceptance (M4)
- [ ] Riff with 3 `mock` agents produces ordered replies, and each later reply can reference the earlier ones.
- [ ] Council with 3 agents runs them in parallel (wall time ≈ the slowest one), then the judge.
- [ ] `@claude` targets only Claude.
- [ ] Killing one participant mid-turn doesn't break the turn.
- [ ] Room transcripts persist and reload.

---

## 8. Feature 3 — Pipeline / Kanban

### State machine

```
Inbox ──plan──▶ Planning ──ok──▶ Planned ──human approve──▶ Approved ──queue──▶ Building
  ▲                │ fail                    │ edit/replan                         │
  │                ▼                         ▼                                     ▼
  └────────── NeedsAttention ◀──────────────────────────────── Checking ──▶ Reviewing
                    ▲                                                          │
                    │ iteration > maxIterations                 pass ──▶ Done  │ fail
                    └──────────────────────────────── Building ◀───────────────┘ (iteration+1)
Any state ──cancel──▶ Cancelled
```

- **Exactly one human checkpoint is mandatory:** `Planned → Approved`. The user can edit the plan before approving.
- Every transition is recorded in `card_events` (from, to, actor = user or agent id, note).

### Stages

1. **Capture.** A card is created from the board, ⌘K, a chat message or a room message. Fields: title, idea (free text), project.
2. **Plan.**
   - The **planner** agent (configurable, default `claude` in `readonly`) uses `prompts/planner.md` and must output JSON matching:

     ```ts
     { title: string; summary: string;
       acceptanceCriteria: string[];          // testable statements
       tasks: { title: string; detail: string }[];
       risks: string[];
       size: "S" | "M" | "L" }
     ```

   - Extract the JSON (raw, or from a fenced block) and validate it with zod. On failure, retry once with the validation error appended. A second failure → NeedsAttention.
3. **Approve.** Done by the human.
4. **Build.** Queued, with at most `board.maxParallelBuilds` running.
   - The daemon creates a worktree: `git worktree add ~/.agentos/worktrees/<project>/<cardId> -b agentos/<cardId> <defaultBranch>`.
   - The **builder** agent runs in `build` mode, with `cwd` = the worktree and `prompts/builder.md`, given plan, acceptance criteria and (on retries) the reviewer's required changes.
   - After the run, the **daemon** commits: `git add -A && git commit -m "agentos(<cardId>): <title> [iter n]"`, so it doesn't rely on the agent committing.
5. **Checks.** The project's `checks` commands run sequentially in the worktree as a `check` run (output streamed and stored). Results are stored as pass/fail per command.
6. **Review.**
   - The **reviewer** agent runs in `readonly` mode with `prompts/reviewer.md`. It **must differ from the builder** (enforced). It gets: plan, acceptance criteria, `git diff <defaultBranch>...HEAD`, and the check results.
   - Output JSON: `{ verdict: "pass" | "fail"; score: number /*0-100*/; findings: string[]; requiredChanges: string[] }`.
   - **Gate:** `verdict === "pass" && score >= passScore && all checks passed` → Done. Otherwise → Building with `requiredChanges` (iteration + 1), or NeedsAttention past `maxIterations`.
7. **Done.** The card shows the branch, diff, check output and reviewer report. Actions: **Merge** (`git merge --no-ff agentos/<cardId>` into the default branch, in the main checkout, only if that checkout is clean), **Open terminal in worktree**, **Archive** (removes the worktree, keeps the branch unless "delete branch" is ticked).

### UI
- **Columns:** Inbox · Planned · Building · Reviewing · Done · Needs attention. Transient states (Planning, Approved, Checking) appear as badges on cards.
- **Card drawer:** plan (editable before approval), live run viewer for the current stage, history of all runs, checks output, diff viewer, reviewer findings, event timeline.
- **Board live updates:** `GET /api/board/stream` (SSE) with `card.updated` events.
- **Vault mirror:** `vault/Board/<cardId> <title>.md` holds plan, status and links.

### Acceptance (M5)
- [ ] With `mock` planner/builder/reviewer and a temp git repo, a card goes Inbox → Done automatically after one approval click.
- [ ] A reviewer `fail` sends the card back to Building with requiredChanges in the next builder prompt. After `maxIterations` it goes to NeedsAttention.
- [ ] A failing check blocks Done even if the reviewer passes.
- [ ] The user's main checkout is never modified, except by an explicit Merge.
- [ ] Two cards build in parallel without conflicts (separate worktrees) when `maxParallelBuilds: 2`.
- [ ] Invalid planner JSON triggers exactly one retry.

---

## 9. Feature 4 — Terminal

- **TerminalManager** in the daemon. `POST /api/terminals {cwd, command?, title?}` → `{id}`. It creates a tmux session running the user's login shell (`$SHELL -l`, normally `zsh`). If `command` is given, it runs inside that shell (e.g. `claude`, `hermes`).
- **macOS PATH note:** a daemon started by launchd doesn't inherit your interactive shell's `PATH` (Homebrew, `~/.local/bin`, npm globals). On startup, resolve it once with `$SHELL -lic 'echo $PATH'` and use it for every spawn, and show it on the Settings page.
- **Transport:** `WS /ws/terminals/:id`.
  - Client → server: `{type:"input",data}` · `{type:"resize",cols,rows}`
  - Server → client: `{type:"output",data}` · `{type:"exit",code}`
- **Persistence:**
  - Sessions live until killed, the shell exits, or the daemon restarts.
  - Each keeps a **ring buffer** (default 2 MB; with tmux, use `capture-pane -p -S -<lines>` for history) replayed on attach, so reloads and re-opens show history.
  - Multiple viewers can attach to the same session.
  - **tmux backend (default):** sessions are `tmux new -d -s agentos-<id>`, and a `node-pty` client attaches to them. They **survive daemon restarts** and can be opened from Terminal/iTerm/Ghostty with `tmux attach -t agentos-<id>`. On daemon start, existing `agentos-*` tmux sessions are rediscovered. Fallback `terminal.backend: pty` (plain `node-pty`, no survival across restarts).
- **UI:**
  - Tabs (split panes are out of scope for v1; use tmux for splits).
  - Rename and close.
  - Quick-launch buttons for each agent (`claude`, `hermes`, `antigravity`, …), opening in the selected project or a card's worktree.
  - "Copy attach command" for tmux sessions.
  - Fit-on-resize via a ResizeObserver; clickable links.
- **Card integration:** a card's "Open terminal" opens a tab with cwd = its worktree.

### Acceptance (M3)
- [ ] Interactive programs work (`vim`/`htop`/`claude` interactive): arrows, Ctrl+C, resize.
- [ ] Reloading the page re-attaches with scrollback intact.
- [ ] Closing the tab in the UI doesn't kill the process. "Kill" does.
- [ ] A WS upgrade without the auth cookie, or with a foreign `Origin`, is rejected (automated test).
- [ ] Terminal sessions survive a daemon restart (tmux backend) and are re-listed in the UI.

---

## 10. Supporting features (minimal in v1)

- **Mission Control (home):**
  - Agent health grid (probe results, version, latency; probes every 60 s and on demand)
  - Active runs with cancel
  - Board counts per column
  - Recent chats and rooms
  - Today's estimated spend per agent
- **⌘K palette:** new chat with <agent>, new room, new terminal (with <agent>), capture card, jump to card/session/room, reload config.
- **Vault:**
  - Writer for chats, rooms and cards (markdown + YAML front-matter, filenames safe on all OSes).
  - FTS5 index of the vault directory (re-index on start and on file change via `chokidar`).
  - `GET /api/vault/search?q=`.
  - "Pin note" adds a note's content to a session's or room's context.
- **Settings page:** read-only view of the loaded config, agent probe details, data dir paths, "Reload config", "Rotate access token".

---

## 11. Data model (SQLite)

```sql
projects(id TEXT PK, name, path, default_branch, checks_json)
sessions(id TEXT PK, agent_id, project_id NULL, title, mode, native_session_id NULL, created_at, updated_at)
messages(id TEXT PK, session_id, role, content, run_id NULL, created_at)
runs(id TEXT PK, agent_id, kind /*chat|room|plan|build|check|review|judge*/, status, cwd,
     parent_type NULL, parent_id NULL, started_at, ended_at NULL, exit_code NULL,
     cost_usd NULL, cost_is_estimate INT, tokens_in NULL, tokens_out NULL, error NULL)
run_events(run_id, seq INT, ts, type, payload_json, PRIMARY KEY(run_id, seq))
rooms(id TEXT PK, title, mode, participants_json, judge_agent_id NULL, created_at)
room_messages(id TEXT PK, room_id, author /*user|agent_id*/, content, run_id NULL, turn_id, created_at)
cards(id TEXT PK, project_id, title, idea, stage, plan_json NULL, review_json NULL,
      planner_agent_id, builder_agent_id, reviewer_agent_id,
      iteration INT, branch NULL, worktree_path NULL, created_at, updated_at)
card_events(id INTEGER PK, card_id, ts, from_stage, to_stage, actor, note)
vault_fts USING fts5(path, title, body)
```

IDs are ULIDs. All timestamps are ISO-8601 UTC.

---

## 12. HTTP / WS API (daemon)

| Method & path | Purpose |
|---|---|
| `GET /auth?t=` | exchange one-time token for cookie |
| `GET /api/health` | daemon health |
| `GET /api/agents` · `POST /api/agents/:id/probe` | registry + probe |
| `GET/POST /api/projects` | projects (from config in v1; POST optional) |
| `GET/POST /api/sessions` · `GET/PATCH/DELETE /api/sessions/:id` | chat sessions |
| `POST /api/sessions/:id/messages` | send → `{runId}` |
| `GET /api/runs/:id` · `GET /api/runs/:id/events` (SSE) · `POST /api/runs/:id/cancel` | runs |
| `GET/POST /api/rooms` · `GET /api/rooms/:id` · `POST /api/rooms/:id/messages` · `GET /api/rooms/:id/stream` (SSE) | rooms |
| `GET/POST /api/cards` · `GET/PATCH /api/cards/:id` | cards |
| `POST /api/cards/:id/{plan,approve,build,retry,cancel,merge,archive}` | card actions |
| `GET /api/board/stream` (SSE) | board updates |
| `GET/POST /api/terminals` · `DELETE /api/terminals/:id` · `WS /ws/terminals/:id` | terminals |
| `GET /api/vault/search?q=` · `POST /api/vault/notes` | vault |
| `GET /api/stats/costs?from=&to=` | cost ledger |

All request and response bodies are defined as zod schemas in `packages/shared`.

---

## 13. Milestones

Each milestone ends with: all acceptance boxes ticked, `pnpm typecheck && pnpm test` green, and a short note in `CHANGELOG.md`.

| # | Milestone | Contents |
|---|---|---|
| **M0** | Scaffold + security | Monorepo, daemon + SPA skeleton, config loader, SQLite + migrations, **auth token / Host / Origin checks with tests**, mock adapter, Mission Control showing agent probes, `pnpm dev` / `pnpm start` |
| **M1** | Chat | RunManager + SSE replay, `openai-compatible` adapter (Ollama, LM Studio, OpenRouter, DeepSeek), `claude-cli` adapter with recorded fixtures, chat UI, vault chat export |
| **M2** | More agents | Investigate Hermes and Antigravity CLIs (`--help` fixtures), implement `cli-json` / `cli-text` or keep `pty`, cost ledger |
| **M3** | Terminal | TerminalManager (node-pty), WS, ring buffer, tabs, quick-launch, optional tmux backend |
| **M4** | Room | Riff + Council modes, mentions, judge, room UI, actions to vault/board |
| **M5** | Board | State machine, planner/builder/reviewer, worktrees, checks, diff view, merge/archive |
| **M6** | Polish | ⌘K palette, theming, vault search + pinning, settings page, `pnpm agentos install-service` → **launchd LaunchAgent** (`~/Library/LaunchAgents/com.agentos.daemon.plist`, `KeepAlive`, logs to `~/.agentos/logs`) |

---

## 14. Testing strategy

- **Adapters:** parser unit tests run against **recorded real outputs** in `test/fixtures/<adapter>/*.ndjson|txt`. Record them first, then write parsers. Never write parsers from memory.
- **Orchestrators** (room, board): integration tests with the `mock` adapter and a temp git repo. Deterministic, offline, token-free.
- **Security:** tests for missing or invalid token, wrong `Host`, foreign `Origin` (HTTP and WS), non-loopback bind refusal.
- **E2E:** Playwright smoke covering open app → chat with mock → room with mocks → card to Done with mocks → open terminal and run `echo ok`.

---

## 15. Explicitly out of scope for v1 (backlog)

Voice (OpenAI Realtime / WebRTC) · preview iframes on a separate origin · tool-use loop for HTTP models · Omi / NotebookLM / media studios · scheduled jobs (radar-style) · mobile PWA + Tailscale remote access · split-pane terminals without tmux · running builders inside containers / devcontainers for stronger isolation.
