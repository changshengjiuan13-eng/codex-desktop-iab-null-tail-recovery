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

## What users actually see

### Problem 1: the Browser tab opens, but the page is blank

Typical symptoms:

- a tab is visible on the right, but no page content is displayed;
- the page flashes once and disappears;
- a page opened by one task appears under another task;
- the page has loaded successfully in the guest renderer, but is not attached
  to the visible right panel;
- refreshing or closing/reopening the tab does not always recover it.

In plain language: **the page is still alive, but Codex has lost the connection
between that page and the current task's visible panel.** The failure is easier
to trigger when several tasks use Browser at the same time or the user switches
tasks frequently.

### Problem 2: a stream interruption leaves the conversation broken

Typical symptoms:

- Codex repeatedly shows “Reconnecting”;
- the UI reports `stream disconnected before completion`;
- commands, tool output, or file changes already exist, but no final reply is
  shown;
- the task becomes a red `systemError`;
- the stored turn ends with `last_agent_message: null`.

In plain language: **a network or upstream stream can fail, but the client
should not record that failure as a normal completion with an empty reply and
leave the whole conversation unusable.**

The local Beta7 change addresses Browser ownership/mount recovery. The Core006
change preserves existing progress and classifies a sampled null-tail turn as
interrupted instead of normally completed.

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

## 中文说明：用户实际看到的故障

### 故障一：内置网页打开了，但右侧看不到内容

常见现象：

- 右边能看到网页标签，但下面是一片白；
- 网页刚出现一下，马上又消失；
- 在一个任务里打开的网页，切换后跑到了另一个任务；
- 网页实际上已经加载成功，但没有显示在右侧面板；
- 刷新或关闭重开也不一定恢复。

大白话：**网页还活着，但 Codex 把网页与当前任务右侧面板之间的连接弄丢了。**
多个任务同时使用 Browser、频繁切换任务时更容易触发。

### 故障二：任务断了一下，整个对话却被写坏

常见现象：

- Codex 不断显示“正在重新连接”；
- 报错 `stream disconnected before completion`；
- 命令已经执行、文件已经修改，但最后没有回复；
- 对话出现红色 `systemError`；
- 底层记录以 `last_agent_message: null` 结束。

大白话：**网络或上游可以断，但客户端不应该把断流写成“正常完成、回复为空”，更不应该因此让整个对话无法继续。**

### 本地版本修改了什么

- **Beta7**：修复内置网页与任务之间的归属、切换、保留和重新挂载。
- **Core006**：保留断流前已经产生的进度，把缺少最终回复的采样任务标记为中断，而不是正常空完成。

本仓库公开复现、版本对比、修复设计、Core 源码补丁和测试流程，不公开完整官方客户端、登录信息、Cookie、聊天记录或其他用户隐私数据。
