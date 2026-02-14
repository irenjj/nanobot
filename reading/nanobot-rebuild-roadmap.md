# nanobot 复刻实现顺序与模块功能

## 1. 先理解运行主链路

- `nanobot agent`：CLI -> 加载配置 -> 初始化 Provider/Tool/Context/Session -> `AgentLoop.process_direct()` -> 输出回复
- `nanobot gateway`：CLI -> 加载配置 -> 初始化 `MessageBus` + `AgentLoop` + `ChannelManager` + `CronService` + `HeartbeatService` -> 并发运行
- `nanobot channels login`：CLI -> 启动 `bridge/` (Node.js WhatsApp 桥接) -> Python `WhatsAppChannel` 通过 WebSocket 通信

---

## 2. 推荐复刻顺序（全量版）

## Step 1: 配置与路径基础

- 模块：
  - `nanobot/config/schema.py`
  - `nanobot/config/loader.py`
  - `nanobot/utils/helpers.py`
- 功能：
  - 定义全局配置结构（agents/providers/channels/tools/gateway）
  - 支持配置读写、默认值、旧字段迁移
  - 提供 workspace/data/sessions/skills 路径工具
- 为什么先做：后面所有模块都依赖配置和路径。

## Step 2: Provider 抽象层

- 模块：
  - `nanobot/providers/base.py`
  - `nanobot/providers/registry.py`
  - `nanobot/providers/litellm_provider.py`
  - `nanobot/providers/transcription.py`（可后补）
- 功能：
  - 统一 LLM 接口（输入消息、输出文本 + tool calls）
  - 用 provider registry 管理模型前缀、env 注入、网关识别
  - 用 LiteLLM 作为默认推理后端
- 完成标志：给定 model + message，能稳定返回 `LLMResponse`。

## Step 3: 消息总线与事件结构

- 模块：
  - `nanobot/bus/events.py`
  - `nanobot/bus/queue.py`
- 功能：
  - 定义 `InboundMessage` / `OutboundMessage`
  - 异步队列发布与消费，支持 channel <-> agent 解耦
- 完成标志：能 publish/consume inbound/outbound。

## Step 4: 会话持久化

- 模块：
  - `nanobot/session/manager.py`
- 功能：
  - 维护每个 session 的消息历史（JSONL）
  - 读写会话、裁剪上下文、供 memory consolidation 使用
- 完成标志：同一 session 连续对话可恢复上下文。

## Step 5: 上下文构建与记忆层

- 模块：
  - `nanobot/agent/context.py`
  - `nanobot/agent/memory.py`
  - `nanobot/agent/skills.py`
- 功能：
  - 组装系统提示词（AGENTS/SOUL/USER + memory + skills + 最近会话）
  - 管理长期记忆 `memory/MEMORY.md` 与历史日志 `memory/HISTORY.md`
  - 发现并读取 `skills/` 中可用技能
- 完成标志：可以输出完整 prompt messages，且 memory 文件可增量更新。

## Step 6: Tool 框架

- 模块：
  - `nanobot/agent/tools/base.py`
  - `nanobot/agent/tools/registry.py`
  - `nanobot/agent/tools/filesystem.py`
  - `nanobot/agent/tools/shell.py`
  - `nanobot/agent/tools/web.py`
  - `nanobot/agent/tools/message.py`
- 功能：
  - 工具参数 schema 校验
  - 工具注册与执行
  - 提供最核心的文件/命令/Web/回消息能力
- 完成标志：LLM 触发 tool call 后，能正确执行并把结果回填对话。

## Step 7: Agent 核心循环

- 模块：
  - `nanobot/agent/loop.py`
- 功能：
  - 处理单轮与多轮 agent 执行
  - 调用 provider -> 解析 tool calls -> 执行工具 -> 再次推理
  - 写 session、发布 outbound 消息
- 完成标志：`process_direct("你好")` 能输出最终答案并落盘历史。

## Step 8: 子代理能力

- 模块：
  - `nanobot/agent/subagent.py`
  - `nanobot/agent/tools/spawn.py`
- 功能：
  - 支持后台任务（spawn 子代理执行）
  - 子代理可复用工具体系，完成后回主会话/消息总线
- 完成标志：主代理可创建子任务并收回结果。

## Step 9: 定时任务

- 模块：
  - `nanobot/cron/types.py`
  - `nanobot/cron/service.py`
  - `nanobot/agent/tools/cron.py`
- 功能：
  - 维护 cron 任务（add/list/remove/enable/run）
  - 触发时回调 agent 执行消息
- 完成标志：任务可持久化并按 cron 或 every 秒触发。

## Step 10: 心跳机制

- 模块：
  - `nanobot/heartbeat/service.py`
- 功能：
  - 周期触发 heartbeat prompt，驱动“主动检查/自唤醒”
  - 识别空响应（`HEARTBEAT_OK`）
- 完成标志：服务可周期运行且不干扰正常会话。

## Step 11: Channel 抽象与管理器

- 模块：
  - `nanobot/channels/base.py`
  - `nanobot/channels/manager.py`
- 功能：
  - 统一 channel 生命周期（start/stop/send）
  - channel inbound -> bus，bus outbound -> channel send
  - 统一 `allowFrom` 权限控制
- 完成标志：接入 1 个具体 channel 后可双向通信。

## Step 12: 各聊天渠道接入

- 模块：
  - `nanobot/channels/telegram.py`
  - `nanobot/channels/discord.py`
  - `nanobot/channels/feishu.py`
  - `nanobot/channels/mochat.py`
  - `nanobot/channels/dingtalk.py`
  - `nanobot/channels/email.py`
  - `nanobot/channels/slack.py`
  - `nanobot/channels/qq.py`
  - `nanobot/channels/whatsapp.py`
- 功能：
  - 对接各平台 SDK/API，把平台事件标准化为 `InboundMessage`
  - 发送 agent 回复到平台
- 建议顺序：
  - 先做 Telegram（最简单）
  - 再做 Discord/Slack（事件模型清晰）
  - 最后做 WhatsApp（需要 Node bridge）

## Step 13: CLI 与整合入口

- 模块：
  - `nanobot/cli/commands.py`
  - `nanobot/__main__.py`
- 功能：
  - `onboard` 初始化配置和 workspace
  - `agent` 本地交互模式
  - `gateway` 启动全服务
  - `channels login` 启动 WhatsApp 登录桥接
  - `cron` 子命令与 `status` 诊断
- 完成标志：通过 CLI 即可完成初始化、聊天、网关运行、任务管理。

## Step 14: WhatsApp Bridge（Node 侧）

- 模块：
  - `bridge/src/index.ts`
  - `bridge/src/server.ts`
  - `bridge/src/whatsapp.ts`
- 功能：
  - 封装 Baileys 客户端
  - WebSocket 暴露给 Python 端（带 token 握手）
  - 处理 QR、连接状态、消息上行下行
- 完成标志：`nanobot channels login` 可扫码登录，`gateway` 可收发消息。

## Step 15: 测试补齐

- 模块：
  - `tests/test_commands.py`
  - `tests/test_tool_validation.py`
  - `tests/test_email_channel.py`
  - 其余测试文件
- 功能：
  - 保证 CLI、工具参数验证、邮件通道等关键路径稳定
- 建议新增：
  - `AgentLoop` 的 tool-call 循环测试
  - `ChannelManager` 的生命周期与路由测试
  - WhatsApp bridge 通讯契约测试

---

## 3. MVP 复刻顺序（最短路径）

如果你要“先跑起来再完善”，按这个顺序：

1. `config + loader + helpers`
2. `providers(base + registry + litellm_provider)`
3. `bus + session`
4. `context + memory`
5. `tools(base + registry + filesystem + shell + message)`
6. `agent/loop.py`
7. `cli agent`（先只做本地终端）
8. `cron`（可选）
9. `channels(base + manager + telegram)`（先单通道）
10. `gateway`
11. 再扩展其他 channels、spawn、heartbeat、whatsapp bridge

---

## 4. 依赖关系速记

- 最底层：`config` / `utils`
- 中间层：`providers` / `bus` / `session`
- 能力层：`context` / `memory` / `skills` / `tools`
- 编排层：`agent loop` / `subagent` / `cron` / `heartbeat`
- 接口层：`channels` / `bridge` / `cli`

一句话记忆：先把“会思考的内核”搭起来，再接“会说话的外壳”。
