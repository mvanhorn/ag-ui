Title: feat(client): add an idle timeout to HttpAgent streams

## Summary

`HttpAgent` can hang forever when a server accepts the request and then stops
writing. This change adds an opt-in `idleTimeoutMs` on `HttpAgentConfig`. When
set, the agent aborts the in-flight fetch if response headers never arrive
within that window, or if the response body goes silent for that long. Unset,
behavior is unchanged: the stream waits indefinitely.

## Behavior

- One option covers both phases (time-to-headers and idle-on-body).
- The timer lives on the HTTP byte stream, not on parsed AG-UI events. Any
  body chunk resets it, including SSE comment keepalives (`: ping`).
- Expiry throws `HttpAgentIdleTimeoutError` with `code: "IDLE_TIMEOUT"` and
  the configured milliseconds in the message. Subscribers see it on
  `onRunFailed` / the Observable error path once.
- The existing `AbortController` is aborted with that error so the fetch is
  cancelled. Timers are cleared on completion, error, unsubscribe, and
  `abortRun()`.
- A caller `abortRun()` during a stall is still an abort (`code: "abort"`),
  not a timeout.
- Applies to SSE and protobuf, because both share the byte-stream timer.
- `clone()` copies `idleTimeoutMs`.
- Construction rejects `0`, negative numbers, `NaN`, and non-finite values
  with `AGUIError`.

## Testing

```
pnpm --filter @ag-ui/client typecheck   # exit 0
pnpm --filter @ag-ui/client lint        # exit 0
pnpm --filter @ag-ui/client build       # exit 0
pnpm --filter @ag-ui/client test        # 108 files, 1358 passed
pnpm exec prettier --check <touched files>  # exit 0
pnpm exec vitest run src/agent/__tests__/http-idle-timeout.test.ts  # 10 passed
```

Vitest coverage (fake timers, fixture `fetch`, controllable `ReadableStream`):
headers never arrive; mid-stream stall after events; SSE `: ping` keepalives;
normal completion clears the timer; `abortRun()` vs timeout; invalid values;
unset never times out; protobuf stall; Observable errors once.

Live demo against a local SSE server: `/stall` without the option still
waiting at 10 s then script abort; `/stall` with 2000 ms times out at ~2009 ms
after partial text; `/keepalive` with 1.5 s gaps completes; `/ok` completes
with no leftover `Timeout` handles.

## Not covered

- No default timeout (callers must opt in).
- No separate `responseTimeoutMs`; one idle window covers headers and body.
- No change to the stream parser buffer limit or to `HttpAgent.connect`.
- Docs update is the JS `HttpAgent` page only.
- Not run against a hosted production agent endpoint.

## Demo

![demo](https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1115/http-agent-idle-timeout/preview.gif)

<video src="https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1115/http-agent-idle-timeout/out.mp4" controls width="720"></video>
