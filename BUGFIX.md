# BUGFIX — 时间线订阅中枢在批次派发期间不校验退订，订阅被「复活」并泄漏会话轮询

## Bug 标题

`SessionTimelineSubscriptionHub` 派发一批时间线更新时，订阅者在其他订阅者的 `onUpdates` 回调中途退订后仍收到该批次；其游标还会被写回订阅表使订阅「复活」，导致该会话的轮询定时器与 Core 尾巴订阅（`releaseChatSessionTimelineTail`）永不释放。

上游 open issue 中无此报告（#7/#6 为 DSH 版本兼容通知、#3 为使用提问、#2 为功能帖、#1 为收录通知），本 bug 由静态审查定位 + 最小复现实锤。

## 证据链（修复前）

用与上游测试相同的方式（`typescript` 转译 `src/runtime/session-timeline-stream.ts` 后以 data-URL 导入）驱动真实 hub 类，场景：同一 session 两个订阅者 A、B 共享一次轮询；A 的 `onUpdates` 回调里调用 B 的退订函数（对应真实路径：`src/runtime/http.ts` 中 SSE 订阅者的退订由 `req aborted` / `res close` 触发，可在一个批次向其他订阅者写响应的期间发生）；随后 A 也退订。基线（上游 HEAD `7bbc62a`）结果，复现脚本 exit 1，四项断言全中：

- `B unsubscribed during dispatch but still received ["call-1"]`（退订后仍被投递）
- `session leaked: activeSessionCount=1 after last unsubscribe`（最后一个真实订阅者退订后会话仍在册）
- `releaseTimeline should be called once, got 0`（Core 尾巴从未释放）
- `polling continued after release (9 -> 13)`（轮询定时器持续运行、持续调用后端）

`git stash` 只暂存修复、复跑同一脚本：exit 1、输出逐字一致，证明缺陷为上游既有、非本次引入；恢复修复后 exit 0。

## 根因

`src/runtime/session-timeline-stream.ts` 的 `#poll()` 派发段迭代的是订阅表的**快照** `[...state.subscribers]`，退订只删除实时表中的条目；循环对快照中的每个条目不复查成员资格就执行 `state.subscribers.set(subscriber, batch.cursorSequence)` 写回游标并调用 `onUpdates`。于是：

1. 已在本次派发中途被退订的订阅者照样收到批次（向已关闭的 SSE 响应写数据）；
2. 写回把已删除的条目重新塞回订阅表，订阅被复活——此后无人持有其退订句柄，`state.subscribers.size` 永不归零，`#release()` 永不触发：轮询定时器无限续期、Core 侧 `releaseChatSessionTimelineTail` 永不调用，会话资源永久泄漏。

## 改动文件

仅一处逻辑改动：

- `src/runtime/session-timeline-stream.ts`：派发循环开头新增成员资格校验——快照条目的游标与实时订阅表不一致（已退订、或已按另一游标重新订阅）时跳过该条目，不投递、不写回。附英文注释说明原因。

其余为发版同步（非逻辑改动）：`package.json` 版本 0.1.5 → 0.1.6、`repository.url` 改指修复版仓库；`README.md` 顶部追加中文 fork 头（上游原文一字未删）；新增 `UPSTREAM.md`、本文件。`LICENSE` 与上游逐字一致。上游代码中无其他版本戳常量、无 lock 文件需同步（`0.1.5` 字面量仅存在于 `package.json`；其 `dependencies` 中 `@beforewave/agent-helm: 0.1.5` 的 pin 按原样保留，不在本版改动范围）。

## 复验结果（修复后）

- 同一复现脚本 exit 0：B 退订后未收到批次；A 退订后 `activeSessionCount=0`、`releaseTimeline` 恰调用 1 次、轮询停止。
- 上游可独立运行的测试全绿、无回归（13/13，与修复前基线一致）：`tests/session-timeline-stream.test.mjs`（含共享轮询、迟到订阅者回退游标、轮询中回退、稳态游标推进、history-to-live upsert 交接五个场景）、`tests/work-history-order.test.mjs`、`tests/timeline-adapter.test.mjs`、`tests/protocol-contract.test.mjs`。

## 验证边界（如实说明）

- 上游 HEAD（v0.1.5）在 `package.json` 中 pin 了 `@beforewave/agent-helm@0.1.5`，该版本**尚未发布到 npm**（npm 上最新为 0.1.4；GitHub 上仅有同名 prerelease tag，其公开仓库不包含该 npm 包的源码），故在本环境中无法按上游 pin 完成 `npm install`，`npm run build`、`npm test`（其 `pretest` 含全量构建）、`npm run typecheck` 无法以真实 pin 跑通。本次验证临时以 agent-helm 0.1.4 替代安装依赖，仅借其工具链（typescript/esbuild）；被改模块对 agent-helm 只有类型导入（type-only import，转译后无运行时依赖），上述行为复现与测试结果不受替代影响。
- 受同一限制，依赖 `lib/` 构建产物的 `tests/core-unavailable.test.mjs`、`tests/runtime-status.test.mjs`、`tests/dsh-0-2-compatibility.test.mjs` 未能运行；替代 pin 下的 typecheck 有 8 处预存错误，全部位于 `src/hosts/dsh.ts`、`src/runtime/runtime.ts` 对 agent-helm 0.1.5 新 API 的引用，与本次改动文件无关（修复前后一致）。未调用任何真实外部服务（ChatGPT 等），复现与测试均为离线 mock/本地逻辑。
