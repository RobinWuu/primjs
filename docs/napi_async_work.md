# N-API Async Work Behavior

PrimJS implements the Node-API async work lifecycle for work that is created
once, queued once, and deleted after its completion callback. Calling
`napi_queue_async_work` again after it has already succeeded is undefined by
Node-API and is outside this contract.

The behavior described here follows the Node-API documentation for
[`napi_create_async_work`](https://nodejs.org/api/n-api.html#napi_create_async_work),
[`napi_queue_async_work`](https://nodejs.org/api/n-api.html#napi_queue_async_work),
and
[`napi_cancel_async_work`](https://nodejs.org/api/n-api.html#napi_cancel_async_work).

## Lifecycle

Async work moves through the following internal states:

```text
Created -> Queued -> Running -> Completed
              |
              +---------> Cancelled
```

The worker and the caller of `napi_cancel_async_work` race on the transition
out of `Queued`. An atomic compare-and-exchange operation ensures that exactly
one side wins:

- `Queued -> Running`: the execute callback runs and cancellation fails.
- `Queued -> Cancelled`: the execute callback is skipped and cancellation
  succeeds.

The complete callback runs on the configured foreground task runner after
either path:

- `Completed` is reported as `napi_ok`.
- `Cancelled` is reported as `napi_cancelled`.

Successfully cancelled work must remain valid until its complete callback is
invoked. It can then be released with `napi_delete_async_work`.

## Cancellation Results

| Work state when cancellation wins or is attempted | Return status | Execute callback | Complete callback status |
| --- | --- | --- | --- |
| Queued and not started | `napi_ok` | Skipped | `napi_cancelled` |
| Running | `napi_generic_failure` | Continues | `napi_ok` |
| Completed | `napi_generic_failure` | Already ran | `napi_ok` |
| Already cancelled | `napi_generic_failure` | Skipped | `napi_cancelled` |

`napi_cancelled` is a completion status. It is not returned by
`napi_cancel_async_work` to report a successful cancellation.

## Behavior Change

Before this fix:

- The Windows implementation did not create a worker thread. Queued work,
  including its completion callback, could remain pending indefinitely.
- `CancelWork()` returned the previous value of its cancellation flag. The
  caller interpreted that value as the result of the current attempt, so the
  first successful cancellation returned `napi_cancelled`, while a repeated
  cancellation returned `napi_ok`.
- A cancellation flag and a completion flag did not reliably distinguish
  queued, running, completed, and cancelled work.

After this fix:

- Windows starts the worker with `_beginthreadex` and joins it with
  `WaitForSingleObject` during runtime shutdown.
- The return value from `napi_cancel_async_work` represents the current
  cancellation attempt: only a successful `Queued -> Cancelled` transition
  returns `napi_ok`; work that can no longer be cancelled returns
  `napi_generic_failure`.
- An atomic lifecycle state makes worker startup and cancellation mutually
  exclusive and gives the complete callback an unambiguous status.

## Regression Test

Build and run the focused PrimJS test:

```bash
ninja -C out/Default napi_unittest
out/Default/napi_unittest
```

The test holds one work item in `Running` so that later items remain queued. It
then verifies worker-thread execution, successful queued cancellation, failed
running/completed/repeated cancellation, skipped execution for cancelled work,
and both completion statuses.
