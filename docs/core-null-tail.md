# Core null-tail completion defect

## Symptom

A turn can contain real reasoning, tool output, or file changes and still end
with:

```json
{
  "type": "task_complete",
  "last_agent_message": null
}
```

Desktop can then present the task as `systemError`, empty, or irrecoverable
even though useful work was already persisted.

## Important distinction

The patch does not promise that a network connection, WebSocket, proxy, or
upstream model stream will never fail. It changes how a failed sampled turn is
classified and finalized.

## Root cause at the pinned source

At `rust-v0.144.0-alpha.4`:

1. a sampling request starts and time-to-first-token is recorded;
2. the response later ends through an error path before a completed assistant
   message exists;
3. compatibility behavior lets `run_turn` return `Ok(None)`;
4. `on_task_finished` treats that result as an ordinary completion;
5. `TurnComplete`/`task_complete` is emitted with a null final message.

There is a second loss mode during tool follow-ups: a valid assistant message
from an earlier sampling pass can be overwritten by a later pass that returns
no new final message.

## Patch behavior

The included [`core006.patch`](../patches/core006.patch):

1. carries the last valid assistant message forward across follow-up sampling;
2. recognizes several mid-stream error-envelope shapes and maps them to the
   same typed API errors as `response.failed`;
3. captures task kind and turn timing before completion is emitted;
4. converts only this combination to interruption:

```text
TaskKind::Regular
AND task_result == Ok(None)
AND time_to_first_token_ms is present
```

5. emits the normal abort lifecycle with `TurnAbortReason::Interrupted` rather
   than a normal null completion.

## Preserved behavior

- Review and compact tasks are not changed by the new classification.
- Truly empty responses that never began sampling remain legitimate empty
  completions.
- Tool-only and reasoning-only interrupted turns keep their existing persisted
  items.
- A previously completed assistant message is retained across a tool
  follow-up.

## Regression coverage

The patch adds focused coverage for:

- carry-forward of the last valid assistant message;
- tool-only and reasoning-only sampled failures;
- partial assistant output followed by an upstream stream error;
- legacy upstream error envelopes;
- a legitimate empty completion that must remain completed.

The public workflow checks the exact upstream commit, applies the patch, runs
format/diff checks, executes focused Linux tests, and builds a Windows x64
artifact. Publishing the binary is not required to review or reproduce the
source change.
