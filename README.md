# Codex Desktop IAB lifecycle and null-tail recovery notes

## 先用大白话说明：到底坏在哪里

这个仓库记录的是 Codex Desktop 的两个实际故障。

### 故障一：内置网页打开了，但看不见

常见现象：

- 右边能看到网页标签，但下面是一片白；
- 网页刚出现一下，马上又消失；
- 在一个对话里打开的网页，切换对话后跑到了别的对话；
- 网页实际上已经加载成功，但没有显示在右侧面板；
- 关闭重开、刷新网页也不一定恢复。

简单说：**网页还活着，但 Codex 把网页和当前对话之间的连接弄丢了。**

这个问题在同时使用多个对话、频繁切换对话、让多个对话打开内置 Browser 时更容易出现。

### 故障二：任务断了一下，整个对话却被写坏

常见现象：

- Codex 显示“正在重新连接”；
- 报错 `stream disconnected before completion`；
- 任务已经执行了命令、修改了文件，但最后没有回复；
- 对话出现红色 `systemError`；
- 重新发送消息后，旧对话仍可能没有正常回复。

底层记录经常以这一项结束：

```json
"last_agent_message": null
```

简单说：**网络或上游可以断，但官方程序不应该把一次断流写成“正常完成但回复为空”，更不应该因此把整个对话弄坏。**

### 我们修改了什么

- **Beta7**：修复内置网页与当前对话的绑定、切换、保留和重新挂载。
- **Core006**：断流后保留已有进度，不再把缺少最终回复的任务写成正常空完成。

这不是修改网站登录，也不是修改 API 账号。修复对象是 Codex Desktop 自己的网页挂载和任务收尾逻辑。

下面内容是给开发者和官方维护人员看的技术分析。

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
