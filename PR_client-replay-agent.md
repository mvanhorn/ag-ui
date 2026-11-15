Title: feat(client): replay recorded event streams with ReplayAgent

## Summary

`@ag-ui/client` currently ships one concrete agent, `HttpAgent`. Testing a
frontend, reproducing a captured run, or driving a UI while the original
server is down all require a live backend. This adds `ReplayAgent` and
`recordEvents` / `createEventRecorder` so a recorded event list can play back
through the normal `AbstractAgent` pipeline (enforce, verify, apply,
subscribers, middleware) with no network.

The recording is a flat array of events. `JSON.stringify(recorder)` writes
that array; `JSON.parse` of the file is a valid `ReplayAgent` constructor
argument. `recordEvents(agent)` stays subscribed until `recorder.stop()`.
`stop()` is idempotent and is a no-op for `createEventRecorder()`.

## Behavior

- `new ReplayAgent(events, { delayMs })` takes the recorded events plus an
  optional delay. `delayMs` defaults to `0` (as fast as the pipeline will take
  events). A number waits that many milliseconds before each event; a function
  receives the event and its index within the current run.
- A recording may hold several runs. Each `runAgent()` call plays exactly one
  run, in order (first call → first run, next call → next run).
- After the last recorded run, the next `runAgent()` emits a `RUN_ERROR` whose
  message names how many runs were played. It does not hang or complete
  silently.
- `RUN_STARTED`, `RUN_FINISHED`, and `RUN_ERROR` events that already carry
  `threadId` / `runId` are rewritten to the current `RunAgentInput`. Every
  other field is left as recorded.
- `abortRun()` stops playback promptly, including while waiting on `delayMs`.
  An in-flight abort emits `RUN_ERROR` with `code: "abort"`.
- Events still go through verify and apply, so a malformed recording fails the
  same way a malformed server stream does.
- `clone()` copies the recording, the delay, and the playback cursor. The clone
  has its own `AbortController`.

## Testing

- `pnpm --filter @ag-ui/client test` — 108 files, 1363 tests passed (includes
  14 new tests in `src/agent/__tests__/replay-agent.test.ts` and an extra clone
  field-coverage case).
- `pnpm --filter @ag-ui/client typecheck` — passed.
- `pnpm --filter @ag-ui/client lint` — passed.
- `pnpm --filter @ag-ui/client build` — passed.
- `pnpm --filter @ag-ui/client test:exports` — passed (`publint --strict` and
  `attw --pack`; existing package.json suggestions only).
- Live capture: `uvicorn example_server:app` from
  `integrations/server-starter/python/examples` on `127.0.0.1:8765`.
  `recordEvents` on `HttpAgent` streamed `Hello world!` (5 events), then
  `recorder.stop()`. After the server was stopped, `curl` failed with exit 7.
  `ReplayAgent` with `delayMs=40` streamed the same text. A further
  `runAgent()` emitted `ReplayAgent has no more recorded runs (played 1 of 1).`

## Not covered

- Protobuf / binary HTTP recordings.
- `connectAgent()`, resume, and interrupt flows.
- Browser-only `fetch` receiver binding (this run used Node).
- Changing the shared conformance runner or its fixtures.

## Demo

![demo](https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1115/client-replay-agent/preview.gif)

[out.mp4](https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1115/client-replay-agent/out.mp4)
