---
name: simple-next-langchain-agent
description: 在 Next.js App Router 中搭建无持久运行环境的单体 Agent 应用，支持消息持久化、固定服务端tool、MCP、HITL。
---

# Next.js + LangChain Agent

## 固定技术路径

用这条路径连接前端 AI UI 与后端 LangChain Agent：

```txt
AI Elements
-> @ai-sdk/react useChat
-> DefaultChatTransport 提交完整 UIMessage[]
-> @ai-sdk/langchain adapter
-> LangChain createAgent().stream()
-> fixed server-side tools / MCP tools / HITL-gated tools
```

不要引入 `@langchain/react`、`FetchStreamTransport`、`HttpAgentServerAdapter`、LangGraph Agent Server protocol、手写 LangGraph-compatible SSE、或多 endpoint command/state/event bridge。

## 先查文档

实现前检查当前官方文档或本地类型：

- AI SDK：`useChat`、`DefaultChatTransport`、`UIMessage`、`createUIMessageStream`、`createUIMessageStreamResponse`、`UIMessageStreamWriter`。
- `@ai-sdk/langchain`：`toBaseMessages`、`toUIMessageStream`。
- AI Elements：`Conversation`、`Message`、`PromptInput`、tool rendering 组件和安装命令。
- LangChain JS：`createAgent`、`tool`、`agent.stream`、`humanInTheLoopMiddleware`。
- LangChain MCP：`MultiServerMCPClient`、`getTools()`、目标 transport。
- LangGraph：checkpointer、`Command({ resume })`、稳定 `thread_id`。
- Supabase：Auth SSR、RLS、服务端数据访问、migration。

## 依赖

核心依赖：

- UI stream：`ai`、`@ai-sdk/react`、`@ai-sdk/langchain`。
- Agent：`langchain`、`@langchain/core`、`@langchain/openai`、`zod`。
- MCP：`@langchain/mcp-adapters`；自建 MCP server 时再加 `@modelcontextprotocol/sdk`。
- HITL：`@langchain/langgraph`。
- Auth/storage：`@supabase/supabase-js`、`@supabase/ssr`。
- UI：AI Elements 和项目需要的 shadcn 组件。

AI Elements / shadcn 生成目录可加入 lint 忽略或降级规则；业务容器代码仍要正常 lint/typecheck。

## 数据边界

前端、API route、持久化之间统一使用 AI SDK `UIMessage[]` 作为聊天历史边界。

要求：

- 页面初始 hydrate 从 Supabase 读取业务会话历史，并传给 `useChat({ messages })`。
- `useChat` 使用 `DefaultChatTransport`，每次提交完整 `UIMessage[]` 到 `/api/chat`。
- `/api/chat` 直接把本次提交的 `UIMessage[]` 作为 Agent 上下文，不做 DB 历史回捞和 prefix 对比。
- Agent 上下文由 `toBaseMessages(uiMessages)` 转成 LangChain messages。
- LangChain stream 由 `toUIMessageStream(langchainStream)` 转回 AI SDK UIMessage stream。
- `createUIMessageStream` 的 `onEnd` 保存最终 `UIMessage[]` 快照；刷新页面时再从 Supabase 恢复。
- LangGraph checkpoint 只用于 HITL 暂停/恢复运行状态；UI 历史仍以 `UIMessage[]` 为边界。

## 后端 Agent

在服务端集中创建 Agent：

- 使用 `createAgent()`。
- 使用 `tool(fn, { name, description, schema })` 定义固定工具，schema 用 `zod`。
- 工具访问业务数据时，只使用服务端 runtime context；不要从消息文本或客户端字段读取 owner/user/thread。
- MCP tools 与本地 tools 一起传入 `createAgent({ tools })`。
- 启用 HITL 时必须配置 checkpointer，并为 `agent.stream` / resume 提供服务端派生的稳定 `thread_id`。

## MCP 工具

MCP 只作为服务端固定工具来源加入 LangChain Agent：

- 使用 `MultiServerMCPClient` 按项目固定配置连接 MCP server。
- 调用 `client.getTools()`，把返回的 tools 与本地 `tool(...)` 合并。
- MCP server、transport、命令、URL、凭据都只能在服务端配置；浏览器不得选择或注入 MCP server。
- local stdio MCP 示例使用 `McpServer.registerTool` 暴露普通工具。
- 在 Vercel 单体部署中确认目标 transport 适合 serverless 生命周期；不要依赖跨请求常驻子进程。

## HITL

HITL 保持在 LangChain Agent 层实现，不引入 Agent Server protocol：

- 使用 `humanInTheLoopMiddleware` 给指定工具配置审批策略。
- 首次运行遇到审批会暂停；API 把 LangGraph interrupt 转成 UI 可渲染的 data part。
- 用户 approve / reject 后，服务端使用 `Command({ resume })` 恢复同一个 `thread_id`。
- 将 HITL resume、interrupt 解析、approval data 写入逻辑放到独立 helper 文件，避免污染 `/api/chat` 主流程。
- 不在 `interrupt` 或审批点之前执行不可重复副作用；必须执行的副作用要幂等。
- 不要把 AI SDK `needsApproval` 当作 LangChain `createAgent` 的 HITL 机制。

## API Route 数据流

`POST /api/chat` 只做这条流水线：

```txt
authenticate
-> ensure owned session
-> build server runtime context
-> toBaseMessages(body.messages)
-> getAgent().stream(..., { streamMode: ["values", "messages", "tools"] })
-> toUIMessageStream()
-> createUIMessageStreamResponse()
-> onEnd save final UIMessage[]
```

要求：

- 使用 `DefaultChatTransport({ api: "/api/chat" })` 对接 `useChat`；仅按需附加 `sessionId` 和 `resume`。
- 使用 `createUIMessageStream` 写入 session data、合并 LangChain stream，并用 `createUIMessageStreamResponse` 返回。
- 请求级鉴权失败返回 HTTP 401/403；Agent 执行错误交给 AI SDK stream error 机制显示。
- 不手写 `event:` / `data:` SSE 帧。
- 不自己拼 token、不自己合并 tool call/tool result、不自己维护 tool UI 状态机。
- 不用 `toTextStreamResponse()` 服务浏览器聊天 UI。

## 前端 UI

Client component 使用：

- `useChat` 管理消息、提交、流式状态和错误。
- `DefaultChatTransport` 指向 `/api/chat`，直接提交完整 `messages[]`。
- AI Elements 渲染 `UIMessage`，优先使用 `<Message message={message} />`。
- 输入组件用 AI Elements `PromptInput`。

要求：

- 刷新页面时，历史来自服务端初始数据。
- streaming、submitted、error 状态下保留已有消息，不闪空。
- 错误态可见展示，但不清空 conversation。
- tool call、tool result、markdown 渲染交给 AI Elements / AI SDK UIMessage parts。
- HITL 审批 UI 只提交 resume 决策；客户端不直接执行工具。

## Supabase

- Supabase 负责 Auth、业务会话列表和 `UIMessage[]` 历史快照。
- 所有会话读写都从服务端 session 派生 owner/权限。
- `/api/chat` 创建或确认当前用户拥有的业务 session。
- 浏览器 bundle 不得包含 provider key、Supabase secret/service role、数据库 URL。
- 需要浏览器直连的业务表必须启用 RLS。

## 可选：图片上传

如需支持对话中上传图片，使用 Supabase Storage 保存图片文件，消息历史中只保存图片 URL。

固定路径：

- 创建公开 Storage bucket，限制为图片 MIME 和合理大小。
- 浏览器选择图片后，先提交到项目自己的上传 API，再使用服务端 Supabase secret/service role client 写入 Storage。
- 上传成功后返回 AI SDK `FileUIPart[]`，包含 `type: "file"`、`mediaType`、`filename`、`url`，前端调用 `chat.sendMessage({ text, files })`。
- 消息渲染遇到 image file part 时显示图片。
- `onEnd` 继续保存完整 `UIMessage[]`；其中只保存 public URL，不保存图片二进制。
- `/api/chat` 数据流不因图片上传改变，仍消费完整 `UIMessage[]` 并使用 `toBaseMessages(body.messages)`。

注意：

- Storage migration 可能需要 secret/service role 权限
- 浏览器直传 Storage 会额外依赖 `storage.objects` RLS policy；为保持主路径简单，直接使用服务端上传 API。
- 图片 bucket 公开读取，避免签名 URL 过期或模型服务无法访问。
- 简化实现时可忽略图片清理等细节路径

## 可选：会话监控

需要显式询问用户是否开启自动 Agent 会话监控。若启用，则指导用户注册 [LangSmith](https://smith.langchain.com/) 帐号，新建项目，在新项目初始界面中依次点击 “.env”、“generate API key”，然后把 LANGSMITH_ 开头的几行环境变量复制到本地 env 文件中即可自动生效，无需更改代码

## 环境变量管理

严禁硬编码敏感凭据，必须使用 .env* 文件管理环境变量，包括 key、project id、llm id 等易变的外部内容，但不要在此管理超时等技术性常量。当需要新增环境变量时，先把 `<VAR_NAME>=` 输出到文件中，然后指导用户手动输入并保存，你确认存在值后才能继续进入开发。

## 验收

- 登录后能从服务端历史 hydrate 出 `UIMessage[]`。
- `useChat` 提交后有流式 assistant 回复。
- AI Elements 正确渲染 markdown、tool call 和 tool result。
- 后端固定工具能被 Agent 调用，且 UI 中可辨认工具结果。
- MCP 工具能作为服务端 tools 被 Agent 调用，浏览器不能修改 MCP 配置。
- HITL 工具能暂停、显示审批、恢复同一运行，并保持 UI 历史一致。
- `onEnd` 后历史被保存，刷新后完整恢复。
- 鉴权、RLS、服务端 secret 边界正确。
