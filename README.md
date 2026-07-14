# Codex Desktop IAB lifecycle and null-tail recovery notes

This repository documents two independent reliability defects observed in
Codex Desktop and the bundled open-source Codex core:

1. the in-app Browser (IAB) can keep a live tab while losing the visible
   right-panel/thread binding; and
2. a sampled turn can be reported as normally completed with
   `last_agent_message: null` after an upstream stream failure.

The Browser findings were reproduced on Windows with Microsoft Store package
`26.707.6957.0`. A byte-level comparison with the locally staged official
package `26.707.8479.0` confirms that the two relevant Browser bundles are
unchanged and that the old lifecycle branches remain present.

The core patch is based on the public upstream tag
`rust-v0.144.0-alpha.4`, commit
`049586f41571e74b44c841868bca3a2233214a71`.

## Executive summary

### Browser/IAB

The guest page can remain navigable and render valid DOM pixels while the
Desktop host no longer presents it in the correct task's right panel. Rapid
task switching, cross-thread Browser activity, claim/release/finalize, and
background visibility requests make the race easier to trigger.

The local recovery design:

- consumes pending capability intents in both claim paths;
- preserves `handoff` tabs for both agent-origin and user-origin tabs;
- defers background visibility until the owning task is synchronized;
- gives BrowserUse pages a stable persistence identity;
- retries failed host registration with bounded, keyed recovery;
- detects a persistent `0x0` host after two frames, pulses the existing host,
  and rebuilds only the exact visible host once if the pulse fails;
- guards all recovery with owner, visibility, in-flight, cooldown, and loop
  checks.

### Core/null tail

The upstream sampling error is not always the defect. The defect is that a
regular turn which started sampling can flow through `Ok(None)` and be emitted
as a normal completion. The patch:

- carries forward the last valid assistant message across tool follow-ups;
- decodes legacy mid-stream error envelope variants as real API errors;
- converts only `TaskKind::Regular + Ok(None) + TTFT present` into
  `TurnAborted(Interrupted)`;
- leaves review/compact tasks and legitimate empty completions unchanged.

## Contents

- [`docs/browser-iab-lifecycle.md`](docs/browser-iab-lifecycle.md)
- [`docs/core-null-tail.md`](docs/core-null-tail.md)
- [`docs/reproduction.md`](docs/reproduction.md)
- [`evidence/official-build-comparison.json`](evidence/official-build-comparison.json)
- [`patches/core006.patch`](patches/core006.patch)
- [reproducible Core build workflow](.github/workflows/build-core006.yml)

## Related upstream reports

- [openai/codex#20743](https://github.com/openai/codex/issues/20743)
- [openai/codex#32040](https://github.com/openai/codex/issues/32040)
- [openai/codex#32798](https://github.com/openai/codex/issues/32798)
- [openai/codex#25619](https://github.com/openai/codex/issues/25619)
- [openai/codex#28751](https://github.com/openai/codex/issues/28751)
- [openai/codex#32366](https://github.com/openai/codex/issues/32366)

## Distribution boundary

No Microsoft Store package, OpenAI Desktop binary, patched ASAR, Browser
profile, authentication material, cookies, conversation rollout, or private
user data is included. The Rust patch applies only to the public Apache-2.0
Codex source at the pinned tag.

## 中文摘要

本仓库记录两个不同问题：内置 Browser 标签仍存活但与当前任务右侧面板失去绑定；以及流式请求失败后仍被写成
`last_agent_message: null` 的正常完成。这里公开复现、版本对比、修复设计、Core
源码补丁和测试流程，不公开完整官方客户端或任何用户隐私数据。
