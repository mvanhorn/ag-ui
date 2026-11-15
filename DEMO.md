# ReplayAgent demo

Captured against `integrations/server-starter` (Python FastAPI example on
`127.0.0.1:8765`). That server emits a fixed `Hello world!` text stream and
does not call a model.

## What was run

1. `pnpm --filter @ag-ui/client test` — 108 files, 1363 tests passed, including
   `replay-agent.test.ts` (14) and `agent-clone.test.ts` (7).
2. `pnpm --filter @ag-ui/client typecheck` — exit 0.
3. `pnpm --filter @ag-ui/client lint` — exit 0.
4. `pnpm --filter @ag-ui/client build` — exit 0.
5. `pnpm --filter @ag-ui/client test:exports` — exit 0.
6. Started `uvicorn example_server:app --host 127.0.0.1 --port 8765`.
7. `recordEvents` on `HttpAgent` recorded five events and streamed
   `Hello world!`. `recorder.stop()` unsubscribed so later runs are not
   recorded.
8. Stopped the server. `curl` to the same port failed (`exit: 7`,
   `Couldn't connect to server`).
9. `ReplayAgent` with `delayMs=40` replayed the saved JSON and streamed the
   same `Hello world!` text with the server down.
10. A further `runAgent()` emitted
    `ReplayAgent has no more recorded runs (played 1 of 1).`

## Media

- `out.mp4`: 1920x1080, 30 fps, 720 frames, 24.000 s, H.264 yuv420p.
- `preview.gif`: 960x540, 8 fps, 24.01 s, 16-color palette, 1.5 MB.
  The final frame holds the kill, curl `exit: 7` (`Couldn't connect to
  server`), the offline replay of `Hello world!`, and the past-the-end
  `RUN_ERROR`: `ReplayAgent has no more recorded runs (played 1 of 1).`
