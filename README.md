# DocGraph

# AI PDF 聊天机器人与智能体 - 基于 LangChain 和 LangGraph

这个单体仓库是一个可定制的 AI 聊天机器人智能体模板示例，它可以「摄取」PDF 文档，将嵌入向量存储在向量数据库（Supabase）中，然后使用 OpenAI（或其他 LLM 提供商）回答用户查询，整个过程由 LangChain 和 LangGraph 作为编排框架。

这个模板也是书籍《学习 LangChain（O'Reilly）》的配套示例：使用 LangChain 和 LangGraph 构建 AI 和 LLM 应用程序。

**聊天机器人界面预览：**


## 目录

1. [功能特性](#功能特性)
2. [架构概述](#架构概述)
3. [前置要求](#前置要求)
4. [安装指南](#安装指南)
5. [环境变量](#环境变量)
   - [前端变量](#前端变量)
   - [后端变量](#后端变量)
6. [本地开发](#本地开发)
   - [运行后端](#运行后端)
   - [运行前端](#运行前端)
7. [使用说明](#使用说明)
   - [上传/摄取 PDF](#上传摄取-pdf)
   - [提问](#提问)
   - [查看聊天历史](#查看聊天历史)
8. [生产构建与部署](#生产构建与部署)
9. [自定义智能体](#自定义智能体)
10. [故障排除](#故障排除)
11. [后续步骤](#后续步骤)

---

## 功能特性

- **文档摄取图**：上传并解析 PDF 为 `Document` 对象，然后将向量嵌入存储到向量数据库中（本示例使用 Supabase）。
- **检索图**：处理用户问题，决定是检索文档还是直接回答，然后生成简洁的响应并引用检索到的文档。
- **流式响应**：从服务器到客户端 UI 的部分响应实时流式传输。
- **LangGraph 集成**：使用 LangGraph 的状态机方法编排摄取和检索过程，可视化智能体工作流程，并调试图中的每个步骤。
- **Next.js 前端**：支持文件上传、实时聊天，并通过 React 组件和 Tailwind 轻松扩展。

---

## 架构概述

```ascii
┌─────────────────────┐    1. 上传 PDF    ┌───────────────────────────┐
│前端 (Next.js)       │ ────────────────────> │后端 (LangGraph)          │
│ - React 聊天 UI     │                      │ - 摄取图                  │
│ - 上传 .pdf 文件    │ <────────────────────┤   + 通过 SupabaseVectorStore 进行向量嵌入 │
└─────────────────────┘    2. 确认信息      └───────────────────────────┘
(在数据库中存储嵌入向量)

┌─────────────────────┐    3. 提问         ┌───────────────────────────┐
│前端 (Next.js)       │ ────────────────────> │后端 (LangGraph)          │
│ - 聊天 + SSE 流     │                      │ - 检索图                  │
│ - 显示来源          │ <────────────────────┤   + 聊天模型 (OpenAI)    │
└─────────────────────┘  4. 流式回答       └───────────────────────────┘

```
- **Supabase** 用作向量存储，用于在查询时存储和检索相关文档。
- **OpenAI**（或其他 LLM 提供商）用于语言建模。
- **LangGraph** 编排摄取、路由和生成响应的「图」步骤。
- **Next.js**（React）为上传 PDF 和实时聊天提供用户界面。

系统由以下部分组成：
- **后端**：Node.js/TypeScript 服务，包含 LangGraph 智能体「图」：
  - **摄取**（`src/ingestion_graph.ts`）- 处理文档索引/摄取
  - **检索**（`src/retrieval_graph.ts`）- 对摄取的文档进行问答
  - **配置**（`src/shared/configuration.ts`）- 处理后端 API 的配置，包括模型提供商和向量存储
- **前端**：Next.js/React 应用程序，为用户提供上传 PDF 和与 AI 聊天的 Web UI。
---

## 前置要求

1. **Node.js v18+**（我们推荐 Node v20）。
2. **Yarn**（或 npm，但此单体仓库已使用 Yarn 预配置）。
3. **Supabase 项目**（如果计划在 Supabase 中存储嵌入向量；请参阅[设置 Supabase](https://supabase.com/docs/guides/getting-started)）。
   - 您需要：
     - `SUPABASE_URL`
     - `SUPABASE_SERVICE_ROLE_KEY`
     - 名为 `documents` 的表和名为 `match_documents` 的函数，用于向量相似性搜索（请参阅[LangChain 文档中关于设置表的指导](https://js.langchain.com/docs/integrations/vectorstores/supabase/)）。
4. **OpenAI API 密钥**（或 LangChain 支持的其他 LLM 提供商的密钥）。
5. **LangChain API 密钥**（免费且可选，但强烈推荐用于调试和跟踪您的 LangChain 和 LangGraph 应用程序）。了解更多[此处](https://docs.smith.langchain.com/administration/how_to_guides/organization_management/create_account_api_key)

---

## 安装指南

1. **克隆** 仓库：

   ```bash
   git clone https://github.com/mayooear/ai-pdf-chatbot-langchain.git
   cd ai-pdf-chatbot-langchain
   ```

2. 在单体仓库根目录安装依赖：

   ```bash
   yarn install
   ```

3. 在后端和前端配置环境变量。详情请参见 `.env.example` 文件。

## 环境变量

项目依赖环境变量来配置密钥和端点。每个子项目（后端和前端）都有自己的 .env.example 文件。将这些文件复制到 .env 并填写您的详细信息。

### 前端变量

在 frontend 中创建 .env 文件：

`cp frontend/.env.example frontend/.env`


**环境变量说明：**

-   `NEXT_PUBLIC_LANGGRAPH_API_URL`：LangGraph 后端服务器运行的 URL。对于本地开发，默认为 `http://localhost:2024`。
-   `LANGCHAIN_API_KEY`：您的 LangSmith API 密钥。这是可选的，但强烈推荐用于调试和跟踪您的 LangChain 和 LangGraph 应用程序。
-   `LANGGRAPH_INGESTION_ASSISTANT_ID`：用于文档摄取的 LangGraph 智能体 ID。默认为 `ingestion_graph`。
-   `LANGGRAPH_RETRIEVAL_ASSISTANT_ID`：用于问答的 LangGraph 智能体 ID。默认为 `retrieval_graph`。
-   `LANGCHAIN_TRACING_V2`：启用跟踪以在 LangSmith 平台上调试您的应用程序。设置为 `true` 启用。
-   `LANGCHAIN_PROJECT`：您的 LangSmith 项目名称。
-   `OPENAI_API_KEY`：您的 OpenAI API 密钥。
-   `SUPABASE_URL`：您的 Supabase URL。
-   `SUPABASE_SERVICE_ROLE_KEY`：您的 Supabase 服务角色密钥。


## 本地开发

这个单体仓库使用 Turborepo 管理后端和前端项目。您可以分别运行它们进行开发。

### 运行后端

1. 导航到后端：

```bash
cd backend
```

2. 安装依赖（如果您在根目录运行了 yarn install，则此步骤已完成）。

3. 以开发模式启动 LangGraph：

```bash
yarn langgraph:dev
```

这将在默认端口 2024 上启动本地 LangGraph 服务器。它应该会重定向您到一个用于与 LangGraph 服务器交互的 UI。[Langgraph studio 指南](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/)

### 运行前端

1. 导航到前端：

```bash
cd frontend
```

2. 启动 Next.js 开发服务器：

```bash
yarn dev
```

这将启动本地 Next.js 开发服务器（默认在端口 3000 上）。

在浏览器中访问 UI：http://localhost:3000。

## 使用说明

当两个服务都运行后：

1. 使用 langgraph studio UI 与 LangGraph 服务器交互，确保工作流程按预期运行。

2. 导航到 http://localhost:3000 使用聊天机器人 UI。

3. 通过页面底部的文件上传按钮上传一个小的 PDF 文档。这将触发摄取图提取文本并通过前端 `app/api/ingest` 路由将嵌入向量存储在 Supabase 中。
    
4. 摄取完成后，在聊天输入中提问。

5. 聊天机器人将通过 `app/api/chat` 路由触发检索图，从向量数据库中检索最相关的文档，并使用相关的 PDF 上下文（如果需要）来回答。


### 上传/摄取 PDF

点击聊天输入区域中的回形针图标。

选择一个或多个 PDF 文件上传，确保总数不超过 5 个，每个文件不超过 10MB（您可以在 `app/api/ingest` 路由中更改这些阈值）。

后端处理 PDF，提取文本，并将嵌入向量存储在 Supabase（或您选择的向量存储）中。

### 提问

- 在聊天输入字段中输入您的问题。
- 响应实时流式传输。如果系统检索了文档，您将看到一个"查看来源"的链接，用于回答中使用的每个文本块。

### 查看聊天历史

- 系统为每个用户会话（前端）创建一个唯一的线程。所有消息都保存在该会话的状态中。
- 出于演示目的，当前示例 UI 不会将整个对话存储在本地线程状态之外，并且不会在会话之间持久化。您可以扩展它以在数据库中持久化线程。但是，"摄取的文档"在会话之间是持久的，因为它们存储在向量数据库中。


## 部署后端

要将您的 LangGraph 智能体部署到云服务，您可以按照此[指南](https://langchain-ai.github.io/langgraph/cloud/quick_start/?h=studio#deploy-to-langgraph-cloud)使用 LangGraph 的云服务，或按照此[指南](https://langchain-ai.github.io/langgraph/how-tos/deploy-self-hosted/)自托管它。

## 部署前端
前端可以部署到任何支持 Next.js 的托管服务（Vercel、Netlify 等）。

确保在部署环境中设置相关的环境变量。特别是，确保 `NEXT_PUBLIC_LANGGRAPH_API_URL` 指向您部署的后端 URL。

## 自定义智能体

您可以在后端和前端自定义智能体。

### 后端

- 在配置文件 `src/shared/configuration.ts` 中，您可以更改默认配置，即向量存储、k 值和过滤参数，在摄取和检索图之间共享。在后端，配置可以在图工作流的每个节点中使用，或者从前端，您可以将配置对象传递到图的客户端中。
- 您可以在 `src/retrieval_graph/prompts.ts` 文件中调整提示词。
- 如果您想更改检索模型，可以在 `src/shared/retrieval.ts` 文件中添加另一个检索器函数，该函数封装了向量存储的所需客户端，然后更新 `makeRetriever` 函数以返回新的检索器。


### 前端

- 您可以在 `app/api/ingest` 路由中修改文件上传限制。
- 在 `constants/graphConfigs.ts` 中，您可以更改发送到摄取和检索图的默认配置对象。这些包括模型提供商、k 值（要检索的源文档数量）和检索器提供商（即向量存储）。


## 故障排除
1. .env 未加载
   - 确保您在后端和前端都将 .env.example 复制到 .env。
   - 检查您的环境变量是否正确，并重新启动开发服务器。

2. Supabase 向量存储
   - 确保您已使用 documents 表和 match_documents 函数配置了 Supabase 实例。请查看 LangChain 关于 Supabase 集成的官方文档。

3. OpenAI 错误
   - 仔细检查您的 OPENAI_API_KEY。确保您有足够的积分/配额。

4. LangGraph 未运行
   - 如果 yarn langgraph:dev 失败，请确认您的 Node 版本 >= 18 并且您已安装所有依赖项。

5. 网络错误
   - 前端必须指向正确的 NEXT_PUBLIC_LANGGRAPH_API_URL。默认情况下，它是 http://localhost:2024。

## 后续步骤

如果您想为此项目做出贡献，请随时提交拉取请求。确保它有良好的文档记录，并在测试文件中包含测试。

如果您想了解更多关于使用 LangChain 和 LangGraph 构建 AI 聊天机器人和智能体的信息，请查看书籍《学习 LangChain（O'Reilly）》。