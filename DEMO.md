# Demo: HttpAgent idle timeout

This branch holds recorded evidence for `feat/http-agent-idle-timeout`.
The feature itself lives on that branch only. This orphan branch has no
product source.

## What the recording shows

A local Node HTTP server exposes three AG-UI SSE routes:

- `/ok` — a complete run (`RUN_STARTED` … `RUN_FINISHED`)
- `/stall` — `RUN_STARTED` plus a partial text message, then silence
- `/keepalive` — the same run shape, with `: ping` comments every 1.5 s

A script drives `HttpAgent` from `@ag-ui/client` against those routes:

1. Invalid `idleTimeoutMs` values (`0`, `-1`, `NaN`, `Infinity`) are rejected
   at construction.
2. `(a)` `/stall` with no timeout is still waiting after 10 s. The script then
   calls `abortRun()`. That abort is reported as `code=abort`, not a timeout.
3. `(b)` `/stall` with `idleTimeoutMs: 2000` delivers the partial text, then
   fails with `HttpAgentIdleTimeoutError` / `IDLE_TIMEOUT` after ~2009 ms.
4. `(c)` `/keepalive` with `idleTimeoutMs: 2000` and 1.5 s gaps completes.
5. `(d)` `/ok` completes in a few milliseconds. `Timeout` handles remaining: 0.

`commands.log` is the exact captured stdout. `out.mp4` / `preview.gif` replay
those lines. Full typecheck, lint, build, and test logs are under `logs/`.

## Reproduce the live run

From a checkout of `feat/http-agent-idle-timeout` with `@ag-ui/client` built:

```bash
node /tmp/idle-timeout-demo/run.mjs
```

The helper used here is not part of the feature branch. It POSTs to the local
server with `HttpAgent` and prints timestamps for each event and error.

## Package checks recorded in `logs/`

```bash
pnpm --filter @ag-ui/client typecheck
pnpm --filter @ag-ui/client lint
pnpm --filter @ag-ui/client build
pnpm --filter @ag-ui/client test
pnpm exec prettier --check <touched files>
pnpm exec vitest run src/agent/__tests__/http-idle-timeout.test.ts
```

Results: typecheck, lint, build, and Prettier all exit 0. Client tests:
108 files, 1358 passed. Idle-timeout file: 10 passed.
