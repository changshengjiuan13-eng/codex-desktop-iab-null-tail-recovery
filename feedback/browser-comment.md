Additional Windows reproduction and code-level comparison for the active-pane / shared-right-panel lifecycle failure.

### Environment

- Windows 10 x64
- registered Codex Desktop package: `26.707.6957.0`
- locally staged official update also inspected: `26.707.8479.0`
- backend: Codex in-app Browser (`iab`)

### Observed failure

After cross-thread Browser use and rapid task switching, a tab could remain alive while the visible right-panel host was blank or bound to another task. In affected states, guest DOM/navigation evidence remained healthy, but the whole Codex window did not contain the guest pixels in the expected right panel. This is consistent with the UI/runtime disagreement already documented in this issue.

### Version comparison

The official ASARs were inspected read-only:

- `26.707.6957.0`: `A728370E715DFA2E8AAA01D1E67C4C27087608CD8FA527D1B3D46CED78A3A8EB`
- `26.707.8479.0`: `8DDC04D44985CA64D59097A76E5871C1010EEB94D7305FD61C5B3F111D452DFF`

Two relevant packed assets are byte-identical in both packages:

- `app-main-CnX2qEsn.js`: `2B30220B986AF70B4D24386A54FF4CCB6D21162615FC4723B5CFDF4FDB68BC49`
- `browser-sidebar-manager-BKDVnejf.js`: `F284A94D9ECEDA6A49E00EA9C0F2F81C3CD7044E45E5154C9D49FB02BB9201CF`

Both main bundles also retain the same two agent-origin-only handoff branches. The newer staged package does not contain all-origin handoff preservation or bounded keyed registration recovery. This shows that the relevant lifecycle fix is not present in the newer package; it does not claim that every IAB crash has this single cause.

### Fix design validated in a self-owned test image

- consume pending capability/visibility intents in both claim paths;
- preserve `handoff` tabs for both user and agent origins;
- defer background visibility until the owning task is authoritative;
- use stable BrowserUse persistence identity;
- use bounded keyed registration recovery for missing bridge/session, rejection, release, and target conflict paths;
- detect a two-frame persistent `0x0` visible host, pulse it non-destructively, then rebuild only that exact host once if needed;
- abort recovery on owner/visibility changes and enforce in-flight/cooldown/loop guards.

Validation included synthetic lifecycle/race cases, ASAR round-trip checks, injected rollback failures, guest and whole-window pixel checks, and task A -> B -> A reattachment.

Sanitized report, reproduction checklist, version evidence, and fix design:

https://github.com/changshengjiuan13-eng/codex-desktop-iab-null-tail-recovery

No official Desktop binaries, Browser profile, authentication material, or private rollout data are included.
