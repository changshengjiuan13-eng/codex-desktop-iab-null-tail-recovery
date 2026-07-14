Additional related failure path at `rust-v0.144.0-alpha.4` / commit `049586f41571e74b44c841868bca3a2233214a71`.

This is adjacent to the silent `last_agent_message=null` behavior described here, but the discriminator is that sampling **did start** (`time_to_first_token_ms` is present) and the stream later failed before a completed assistant message existed.

### Vulnerable lifecycle

1. regular turn starts sampling;
2. reasoning/tool output may already be persisted;
3. upstream stream fails before a completed assistant message;
4. compatibility behavior lets `run_turn` return `Ok(None)`;
5. `on_task_finished` emits a normal completion with `last_agent_message=null`.

### Narrow recovery rule

Convert only this combination to an interrupted turn:

```text
TaskKind::Regular
AND task_result == Ok(None)
AND time_to_first_token_ms is present
```

Review/compact tasks and truly empty completions which never started sampling remain unchanged. A companion carry-forward change preserves the last valid assistant message across tool follow-up sampling.

The patch also normalizes legacy mid-stream error envelope variants into the same typed API errors used by `response.failed`, so a terminal upstream error does not get retried as a generic stream close.

Sanitized source patch, focused test coverage, and reproducible workflow:

https://github.com/changshengjiuan13-eng/codex-desktop-iab-null-tail-recovery

The repository contains no Desktop binary, auth material, provider configuration, rollout, or user data.
