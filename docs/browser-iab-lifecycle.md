# Browser/IAB lifecycle defect

## User-visible symptoms

- A Browser tab header exists but the right panel is blank.
- A page appears briefly and disappears.
- The page is still live in the guest renderer, but the current task has no
  visible host attachment.
- `tabs.list()`/`user.openTabs()` can disagree with the visible UI.
- A visibility request can return successfully while the Browser pane remains
  hidden.
- Switching away from a task and back can lose the presentation binding.

These symptoms must not be conflated with website login, Browser control-plane
OAuth, DNS/TLS failures, or a guest navigation error. In the reproduced mount
failure the target page had valid DOM and guest pixels; the whole Codex window
did not contain those pixels in the right panel.

## Confirmed affected official builds

Local Store package artifacts:

| Package | State inspected | ASAR SHA-256 |
|---|---|---|
| `26.707.6957.0` | registered package and reproduction base | `A728370E715DFA2E8AAA01D1E67C4C27087608CD8FA527D1B3D46CED78A3A8EB` |
| `26.707.8479.0` | locally staged official update | `8DDC04D44985CA64D59097A76E5871C1010EEB94D7305FD61C5B3F111D452DFF` |

The following packed assets are byte-identical in both builds:

| Asset | Size | SHA-256 |
|---|---:|---|
| `webview/assets/app-main-CnX2qEsn.js` | 862,724 | `2B30220B986AF70B4D24386A54FF4CCB6D21162615FC4723B5CFDF4FDB68BC49` |
| `webview/assets/browser-sidebar-manager-BKDVnejf.js` | 52,476 | `F284A94D9ECEDA6A49E00EA9C0F2F81C3CD7044E45E5154C9D49FB02BB9201CF` |

The main bundle name/hash changed between the packages, but both copies retain
the same two agent-origin-only handoff branches. Neither contains the bounded
registration recovery or all-origin handoff behavior described below.

This demonstrates that the relevant lifecycle fix was not incorporated into
the newer staged build. It does not claim that every Browser crash has the same
root cause.

## Reproduction

The highest-yield pattern is concurrent or rapid cross-thread Browser use:

1. Open a working IAB page in task A.
2. Finalize or transfer the tab for user visibility.
3. Switch to task B and open another IAB page.
4. Rapidly return to task A, or resume several Browser-using tasks close
   together.
5. Observe a tab header with a blank right panel, or an empty runtime tab list
   while a Browser surface is still shown.
6. Confirm the target page independently produced DOM and guest pixels.
7. Confirm the whole Codex-window capture lacks the page pixels in the expected
   right-panel bounds.

This is a race and may require repeated task switching. Do not use page title,
URL, or offscreen guest screenshots alone as proof of visible success.

## Root-cause model

The shared right-panel host has multiple state owners:

- task/thread state;
- user/agent tab origin;
- claim/release/finalize lifecycle;
- background visibility requests;
- BrowserUse page registration;
- the renderer webview host and its current bounds.

The failing implementation can preserve one part of this state while dropping
another. A tab can therefore remain alive without a valid presentation owner,
or a new host can be created without consuming the visibility intent that was
queued before registration.

## Recovery design

### Claim and handoff

- Both claim branches consume pending capability/visibility intents.
- `handoff` preserves the tab regardless of whether its origin is `agent` or
  `user`.
- Turn completion removes transient marks but does not release a handed-off
  user tab.

### Visibility ownership

- A background request records pending visibility.
- Presentation occurs only after the requesting task becomes the authoritative
  visible owner.
- Task switching hydrates the current owner before applying pending intent.

### Registration recovery

- Retry state is keyed by persistent Browser page identity rather than only by
  a transient object reference.
- Missing bridge, missing session, rejected session, release, and target
  conflict paths converge on one bounded recovery mechanism.
- Retries have an attempt cap and cannot spin indefinitely.

### Zero-viewport recovery

1. Detect an exact current visible host whose bounds remain `0x0` for two
   animation frames.
2. Pulse the existing host non-destructively and restore visibility.
3. Recheck bounds and ownership.
4. If still `0x0`, invalidate and recreate only that host once.
5. Abort recovery on task/owner/document visibility changes.
6. Apply in-flight, stale-owner, cooldown, and loop guards.

## Validation used locally

- static anchor and syntax checks;
- synthetic lifecycle/race cases for claim, handoff, background visibility,
  missing bridge/session, release, target conflict, and bounded retry;
- ASAR path/order/packed-state round trip;
- injected write-failure rollback checks;
- hidden startup process checks;
- guest DOM/pixel checks;
- whole-window right-panel pixel checks;
- task A -> B -> A reattachment checks;
- no repeated host-rebuild marker.

The patched local build passed those checks. This report presents the design
and evidence, not a redistributed Desktop binary.
