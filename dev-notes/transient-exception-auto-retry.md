# Transient-exception auto-retry (Shape 2 only)

## Problem

Long runs die when a transient transport failure escapes the provider adapters
as a **raised exception** (our "Shape 2"): the traceback shows raw
`ssl.SSLError` (e.g. `SSLV3_ALERT_BAD_RECORD_MAC`, `UNEXPECTED_EOF_WHILE_READING`)
escaping `openai_compatible.py`'s `except httpx.HTTPError`, unwinding through
`canonicalize_provider_stream` → `_assistant_events` → `run_agent_loop` →
harness, and out of `CodingSession.prompt()`/`continue_()` as
`kind: "exception"` in `/Users/<user>/.tau/logs/agent-calls.jsonl`. Nothing is
committed for the failed turn, but the whole run blows up.

Scope decision: **only Shape 2 changes.** Shape 1 — failures the adapters
convert into an empty `stop_reason == "error"` `AssistantMessage` — keeps its
current behavior (manual "continue" recovery). No adapter changes; shapes are
not converted.

## Why a retry is safe here

A Shape-2 failure raises inside the provider-stream consumption, before
`run_agent_loop` appends any assistant message (`loop.py` only appends after
`_assistant_events` completes). Therefore the transcript has no partial turn and
re-running the loop (`_harness.continue_()`) re-prompts exactly the clean
context a human "continue" would use — matching the user's experience that
"continue" always fixes these.

## Classification

Auto-retry only when the escaped exception is a recognized **transient
transport failure** (`_is_transient_transport_exception`):

- `ssl.SSLError` / `ssl.SSLEOFError` — **except** `ssl.SSLCertVerificationError`
  (that is a persistent trust/config problem, not a blip);
- `ConnectionError` family (`ConnectionResetError`, `BrokenPipeError`);
- `TimeoutError` / `socket.timeout`;
- `httpx.TransportError` family (defensive net; normally the adapters catch
  these, but if one ever escapes it is transient by the same rule);
- `asyncio.CancelledError` / user cancellation is `BaseException` and never
  reaches `except Exception` — retries never fight cancellation.

Anything else (our own bugs, `KeyError`, provider-induced non-transport
exceptions) logs and re-raises exactly as today.

## Budget, backoff, cancellation

Identical to the previous draft:

- `_GENERIC_RETRY_MAX_ATTEMPTS = 3` total requests (initial failure + **two**
  retries). Hard cap prevents token burn and loops.
- Exponential backoff: base `_GENERIC_RETRY_BASE_DELAY_MS = 2_000`, cap
  `_GENERIC_RETRY_MAX_DELAY_MS = 16_000` → delays 2s, 4s.
- `_retry_interrupt` (`SimpleCancellationToken`) on `CodingSession`, cancelled
  by `cancel()`, reset at the top of each `prompt()` / `continue_()`; backoff
  sleeps in 50ms polls so user cancellation aborts promptly and a cancelled
  retry reports failure, not success.

## Hook points (tau_coding/session.py) — as implemented

- `prompt()` and `continue_()` `except Exception` blocks now delegate to
  `_run_transient_exception_auto_retry(context, exc, phase)`. The non-transient
  path runs the same log-and-raise as before inside the helper, so
  `kind == "exception"` (today's failure record) is preserved for terminal
  failures; a *recovered* blip records only
  `kind == "transient_auto_retry"` so "kind = exception ⇒ the run died" stays
  meaningful for log mining.
- The retry helper: `AutoRetryStartEvent` per attempt → cancellable backoff
  via `tau_ai.retry.wait_for_retry` (50ms poll, `_retry_interrupt` as signal) →
  `_harness.continue_()` with events passed through (`AgentEndEvent` maps to
  `SessionAgentEndEvent(will_retry=False)`). A nested transient failure inside a
  retried run is retried again within the same budget; non-transient or
  exhausted budget → `AutoRetryEndEvent(success=False)` + log + re-raise.
- The helper owns and acloses its own `continue_()` iterator (the
  route-failover convention); the outer `finally` still reconciles the
  original raised iterator exactly as it did before this change.
- `cancel()` also cancels `_retry_interrupt`; `prompt()`/`continue_()` replace
  it with a fresh token at run start (`SimpleCancellationToken` has no reset).
- Residual edge, accepted as bounded: the `except Exception` covers whole
  method bodies, so a transport-class exception from an extension
  `before_tool_call`/`after_tool_call` hook (unguarded inside `run_agent_loop`)
  would also be retried once — bounded to an extra turn, and rare because the
  gate requires a raw transport exception. Auxiliary session-side provider
  calls (auto-name, compaction) are already self-guarded and never reach the
  hook.
- Nesting semantics (intentional): a transient exception escaping the
  overflow-retry or route-failover continuation gets its own fresh two-retry
  budget — worst case overflow(1) + failover(1) + 2 retries = 4 extra
  requests, bounded.

## Events and diagnostics — as implemented

Reuses `AutoRetryStartEvent`/`AutoRetryEndEvent`; TUI/print renderers already
handle them.
`AgentCallDiagnosticLogger.log_transient_auto_retry(...)` mirrors
`log_huggingface_route_failover` but records **per-attempt** error type +
message and uses kind `"transient_auto_retry"` (never `"exception"`), so live
``kind == "exception"`` entries keep meaning "the run died".

## Tests (tests/test_coding_session.py)

1. Provider raises `ssl.SSLError("bad record mac")` on its first `stream_response`,
   succeeds on the next → `prompt()` does **not** raise; sequence is
   `agent_end(will_retry=True)`, `auto_retry_start`, continuation,
   `auto_retry_end(success=True)`, `agent_settled`; first-call messages are
   exactly the same set as the retry-call messages (nothing committed in
   between).
2. Persistent transport exceptions → two retries, then the terminal exception
   is re-raised; `auto_retry_end(success=False)`; no further provider calls.
3. Non-transport exception (e.g. `ValueError` from session-side code) → no
   retry, re-raised, logged as `kind: "exception"` (same as today).
4. `ssl.SSLCertVerificationError` → no retry (persistent class excluded).
5. Cancel during backoff (small real delay) → prompt ends promptly, no retry
   provider call.

## Open questions

- Is two retries / 3 attempts plus the 2s/4s backoff the right budget for long
  runs, or should it follow the provider config (`max_retries`) instead?
- Should the backoff base/cap be configurable (env or providers.json) or stay
  module constants?
- Should a **successful** retry append any marker to the transcript (e.g. a
  small notice entry) so sessions are transparent about auto-recovery, or keep
  the transcript identical to a clean run?
- Exceptions inside auxiliary session-side provider calls (`auto-name`,
  pre-prompt compaction) that happen to be transient-transport by type will now
  be retried too — is that desirable, or should the retry be scoped to the
  main agent-loop stream only?