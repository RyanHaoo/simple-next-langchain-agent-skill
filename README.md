# create-next-langchain-agent

一个用于指导 AI Coding Agent 快速在可一键部署的 next.js 项目中实现 LangChain Agent 的技能

```txt
skills/
  simple-next-langchain-agent/
    SKILL.md
```

## 这个 skill 做什么

这个 skill 定义了一条固定实现路径，用来在 **Next.js App Router** 中搭建单体 Agent 应用，支持：

- 服务端固定 tools
- MCP tools
- HITL（Human in the Loop）
- Supabase 持久化聊天历史

**支持随所在的 next 项目使用 vercel 等一键部署上线，不用维护 Agent Server**

## 技术路线

技术栈全都给定，主打快速复现，不是通用的生产级 Agent 开发框架

```txt
AI Elements
-> @ai-sdk/react useChat
-> DefaultChatTransport
-> @ai-sdk/langchain
-> LangChain createAgent().stream()
-> server-side tools / MCP / HITL
```

## 使用方式

1. 在支持技能读取的 Agent 环境中加载 `skills/simple-next-langchain-agent/SKILL.md`。
2. 按照 skill 中定义的技术边界实现项目。
