# Reproduction and evidence checklist

## Browser/IAB

### Minimal user-visible reproduction

1. Use Codex Desktop on Windows with the in-app Browser enabled.
2. In task A, ask Codex to open a simple page such as `https://example.com/`.
3. Keep/transfer that page for user visibility.
4. Switch to task B and open another Browser page.
5. Switch A -> B -> A several times, or resume multiple Browser-using tasks in
   close succession.
6. Look for a visible tab header with no page content, a tab that flashes and
   disappears, or a mismatch between UI state and the Browser runtime.

### Evidence required to classify a mount failure

- Browser control-plane access succeeds.
- Target-site navigation succeeds or the page DOM is meaningful.
- Guest screenshot contains real page pixels.
- The correct task is foreground.
- The expected right-panel bounds are non-empty in the host layout.
- Whole-window pixels do not contain the guest page in those bounds.

URL/title/tab metadata alone is insufficient.

## Core/null tail

1. Start a regular turn which produces at least one streamed event or tool
   action.
2. Cause the upstream stream to end before a completed assistant message. A
   deterministic test transport is preferable to a live network failure.
3. Inspect app-server events and rollout JSONL.

### Vulnerable behavior

```text
sampling started
useful reasoning/tool items persisted
no completed assistant message
turn/completed
last_agent_message = null
```

### Expected behavior

```text
sampling started
useful items persisted
no completed assistant message
turn/aborted(reason = interrupted)
```

If a valid assistant message was completed before a follow-up tool pass, that
message should remain the final message.

## Redaction checklist before sharing logs

- remove auth headers, cookies, API keys, JWTs, and query strings;
- replace local usernames and absolute paths;
- replace thread/turn IDs unless needed for event correlation;
- omit private page contents and screenshots containing account data;
- include package version, asset hash, timestamps, and only the relevant log
  markers.
