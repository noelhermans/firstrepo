# AgentOS — instructions for Claude Code

A private, single-user command center for AI agents on the owner's Mac: chat with agents, multi-agent room, pipeline/kanban, terminal.

- **Source of truth:** `SPEC.md`. Build it milestone by milestone (§13). Don't start the next milestone until the current one's acceptance boxes are ticked and `pnpm typecheck && pnpm test` pass.
- **Background only:** `agentos-teardown.md` analyses a different product. Use it to understand patterns. Don't copy its branding, layout or code, and **don't** copy its security model (it had none).

## Hard rules

1. **Security baseline first** (SPEC §2): bind `127.0.0.1`, token cookie auth on every HTTP route and WS upgrade, `Host` and `Origin` checks, no CORS. These have tests. Never weaken them to make something work.
2. **Never guess CLI flags or output formats.** Before writing or changing an adapter, run `<cli> --help`, record real output into `apps/daemon/test/fixtures/<adapter>/`, and write the parser against the fixtures. If a CLI isn't installed in your environment, stop and ask the user to record the fixture.
3. **Spawning:** use `execa` with an args array. Never `shell: true` with user-controlled text. Pass prompts via stdin. Kill the whole process group on cancel. Use the login-shell `PATH` resolved at startup.
4. **Secrets** come from the macOS Keychain or `~/.agentos/.env`. Never log them, never return them from the API, and never put them in run events.
5. **No token-spending tests.** Tests use the `mock` adapter. Real agents are only exercised manually.
6. **Board builds only touch git worktrees** under `~/.agentos/worktrees/`. Never modify the user's main checkout except through the explicit Merge action.
7. **Validate at boundaries with zod** (config, HTTP bodies, adapter output, planner/reviewer JSON). Shared schemas and types live in `packages/shared`. Don't duplicate types in apps.
8. **Adapters emit only the normalized `AgentEvent` union** (SPEC §4.1). UI, room and board never special-case a specific agent.
9. **Prompts live in `prompts/*.md`** and are loaded at runtime. Don't inline long prompts in code.
10. TypeScript `strict`, no `any` without a comment explaining why. Small modules. Prefer boring, well-known libraries.

## Layout

```
apps/daemon   Fastify daemon (agentosd): runs, adapters, room, board, terminal, vault, SQLite
apps/web      Vite + React SPA (served by the daemon in prod)
packages/shared  zod schemas + types (events, DTOs, config)
prompts/      runtime system prompts (planner, builder, reviewer, room, judge)
config/       agentos.config.example.yaml
```

## Commands (create these in M0)

```
pnpm install
pnpm dev          # daemon (watch) + Vite dev server proxied through the daemon origin
pnpm start        # build web, serve it from the daemon
pnpm typecheck
pnpm test         # vitest, offline, mock adapters only
pnpm test:e2e     # playwright smoke (mock adapters)
```

## Working style

- Before each milestone, write a short plan (files, schemas, tests). Then implement, test, and update `CHANGELOG.md`.
- When the spec is ambiguous, pick the simplest option that satisfies the acceptance criteria and note the decision in `docs/decisions.md`.
- If a spec decision turns out to be wrong in practice, propose a spec change instead of silently diverging.
