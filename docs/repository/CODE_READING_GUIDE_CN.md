# OpenViking 代码阅读指南

面向第一次阅读 OpenViking 源码的开发者。本文不重复产品文档，而是提供一张可用的代码地图：每块代码负责什么、从哪里进入、推荐按什么顺序读。

> 本文按 2026-11-02 的 `main` 分支核对。目录或入口调整后，请同步更新本文。

## 先建立整体认识

OpenViking 是一个多语言仓库，不要把它当成单一 Python 项目来读：

- **Python 后端**在 `openviking/`，提供 HTTP Server、业务服务、解析、检索、会话和存储抽象。
- **Python 启动层**在 `openviking_cli/`，负责 `openviking-server` 启动、`init`/`doctor`、配置加载，以及 `ov` 命令的 Python 包装。
- **Rust CLI** 在 `crates/ov_cli/`，是 `ov` 命令的实际实现。
- **Rust 存储引擎** 在 `crates/ragfs/`，通过 `crates/ragfs-python/` 嵌入 Python 进程。
- **C++ 向量引擎**在 `src/`，编译成 Python abi3 扩展，为向量库提供索引和存储能力。
- **客户端 SDK** 在 `sdk/`，包含 Python、TypeScript 和 Go 三个版本。
- **Web Studio** 在 `web-studio/`，是 React + TanStack Router 的前端 SPA。
- **VikingBot** 在 `bot/`，是独立的多渠道 Agent 运行时，不是 Server 内部模块。

一条主请求的路径可以简化成：

```text
ov / SDK / Web Studio / 直接 HTTP
              │
              ▼
    OpenViking Server（FastAPI）
    openviking/server/app.py
              │
              ▼
      OpenVikingService
    openviking/service/core.py
     ├─ FSService（文件系统）
     ├─ ResourceService（导入、解析、入库）
     ├─ SearchService（检索）
     ├─ SessionService（会话、归档、记忆）
     └─ ...
              │
       ┌──────┴──────┐
       ▼             ▼
   VikingFS        VectorDB
 （RAGFS/Rust）   （C++ 引擎 / Python 适配）
```

Gateway 和 VikingBot 是旁边两条独立的链路：Gateway 作为模型 API 代理，通过公开 HTTP API 读写记忆；VikingBot 作为独立进程运行 Agent 对话，也通过 OpenViking API 使用上下文。两者都不直接复用 `OpenVikingService` 的内部对象。

## 先读哪些产品与架构文档

读代码前先读下面几篇，能避免把实现细节和产品概念混在一起：

| 文档 | 用途 |
|---|---|
| [README_CN.md](README_CN.md) | 了解 OpenViking 是什么、如何快速跑起来 |
| [架构概述](../zh/concepts/01-architecture.md) | 建立系统分层、核心模块和数据流的整体印象 |
| [上下文类型](../zh/concepts/02-context-types.md) | 理解 Resource、Memory、Skill 的区别 |
| [上下文分层](../zh/concepts/03-context-layers.md) | 理解 L0/L1/L2 与摘要、概览、原文的关系 |
| [Viking URI](../zh/concepts/04-viking-uri.md) | 理解 `viking://` 路径和用户空间 |
| [存储架构](../zh/concepts/05-storage.md) | 理解内容存储与向量索引的双层结构 |
| [上下文提取](../zh/concepts/06-extraction.md) | 跟住资源导入的解析、写入、语义生成链路 |
| [检索机制](../zh/concepts/07-retrieval.md) | 跟住 `find()` 与 `search()` 的差异 |
| [会话管理](../zh/concepts/08-session.md) | 跟住提交、归档和记忆提取 |

## 主要目录职责

### 顶层目录

| 目录 | 职责 |
|---|---|
| [`openviking/`](../../openviking/) | Python 主包，HTTP Server、业务服务、解析、检索、会话、存储抽象都在这里 |
| [`openviking_cli/`](../../openviking_cli/) | Python 启动与兼容层：`openviking-server` 入口、配置模型与加载、`init`/`doctor`、`ov` 的 Python 包装、旧 HTTP client 兼容入口 |
| [`openviking_gateway/`](../../openviking_gateway/) | 模型 API 网关（Beta）：为客户端补充记忆召回、OpenViking 工具、会话保存与压缩；可嵌入 Server，也可独立运行 |
| [`crates/`](../../crates/) | Rust workspace：`ov_cli`（CLI）、`ragfs`（RAGFS 内容文件系统）、`ragfs-python`（PyO3 绑定） |
| [`src/`](../../src/) | C++ 向量索引与存储引擎，编译成 Python abi3 扩展，不是独立服务 |
| [`sdk/`](../../sdk/) | Python、TypeScript、Go 的轻量 HTTP SDK，只负责请求、响应和错误映射 |
| [`web-studio/`](../../web-studio/) | Web Studio 前端源码；构建产物复制到 `openviking/web_studio/dist/` |
| [`bot/`](../../bot/) | VikingBot：CLI、Gateway、多渠道接入、Agent loop、工具和沙箱 |
| [`agent-plugins/`](../../agent-plugins/) | Agent Plugins 1.0 包，包含 MCP 代理和技能定义 |
| [`integrations/`](../../integrations/) | LangChain/LangGraph 与 Microsoft Agent Framework 适配器 |
| [`examples/`](../../examples/) | 可运行示例、客户端插件和配置样例，不属于核心运行时 |
| [`tests/`](../../tests/) | 按领域拆分的 pytest 测试，以及 C++ 引擎测试；入口见 [tests/README.md](../../tests/README.md) |
| [`docs/`](../../docs/) | VitePress 文档站（`en/`、`zh/`）和仓库型文档（`repository/`） |
| [`benchmark/`](../../benchmark/) | Memory、RAG、检索和向量库性能评测 |
| [`deploy/`](../../deploy/) | Docker 与 Helm 部署物料 |
| [`scripts/`](../../scripts/) | 构建辅助、网关验收和发布脚本 |
| [`third_party/`](../../third_party/) | vendored 三方依赖（LevelDB、spdlog、rapidjson 等），通常不需要读 |

### `openviking/` 内部

| 子目录 | 职责 | 先看什么 |
|---|---|---|
| [`server/`](../../openviking/server/) | FastAPI 应用、路由、认证、依赖注入、MCP 端点、内嵌 Gateway | [`app.py`](../../openviking/server/app.py)、[`routers/`](../../openviking/server/routers/) |
| [`service/`](../../openviking/service/) | 业务服务层，与传输层解耦 | [`core.py`](../../openviking/service/core.py)、各 `*_service.py` |
| [`core/`](../../openviking/core/) | 领域模型、URI/命名空间、目录定义、技能加载 | [`context.py`](../../openviking/core/context.py)、[`namespace.py`](../../openviking/core/namespace.py) |
| [`retrieve/`](../../openviking/retrieve/) | 意图分析、全局检索、Rerank、上下文组装 | [`intent_analyzer.py`](../../openviking/retrieve/intent_analyzer.py)、[`hierarchical_retriever.py`](../../openviking/retrieve/hierarchical_retriever.py) |
| [`session/`](../../openviking/session/) | 会话、归档、记忆抽取、工作记忆 | [`session.py`](../../openviking/session/session.py)、[`compressor_v3.py`](../../openviking/session/compressor_v3.py) |
| [`parse/`](../../openviking/parse/) | 文档解析、来源接入、目录树构建、VLM 处理 | [`parser_router.py`](../../openviking/parse/parser_router.py)、[`registry.py`](../../openviking/parse/registry.py)、[`tree_builder.py`](../../openviking/parse/tree_builder.py) |
| [`storage/`](../../openviking/storage/) | VikingFS、RAGFS 客户端、向量库管理、队列、ACL、写入协调 | [`viking_fs/`](../../openviking/storage/viking_fs/)、[`queuefs/`](../../openviking/storage/queuefs/)、[`vectordb/`](../../openviking/storage/vectordb/) |
| [`resource/`](../../openviking/resource/) | 资源监听、调度、Watch 存储与来源管理 | [`watch_manager.py`](../../openviking/resource/watch_manager.py)、[`watch_scheduler.py`](../../openviking/resource/watch_scheduler.py) |
| [`ingest/`](../../openviking/ingest/) | 把 Claude Code、Codex 等本地对话日志回放进 OpenViking 会话 | [`orchestrator.py`](../../openviking/ingest/orchestrator.py) |
| [`connector/`](../../openviking/connector/) | 外部 Connector 服务客户端和来源路由 | [`client.py`](../../openviking/connector/client.py) |
| [`config/`](../../openviking/config/) | 运行时配置管理：配置源、集群/账号作用域、合并与校验 | [`manager.py`](../../openviking/config/manager.py) |
| [`models/`](../../openviking/models/) | Embedding、Rerank、VLM 和网络访问封装 | [`embedder/`](../../openviking/models/embedder/) |
| [`prompts/`](../../openviking/prompts/) | Prompt 管理和 YAML 模板 | [`manager.py`](../../openviking/prompts/manager.py) |
| [`crypto/`](../../openviking/crypto/) | 文件加密配置、Provider 和加密器 | [`encryptor.py`](../../openviking/crypto/encryptor.py) |
| [`privacy/`](../../openviking/privacy/) | 隐私规则、扫描、脱敏、恢复 | [`sanitizer.py`](../../openviking/privacy/sanitizer.py) |
| [`observability/`](../../openviking/observability/)、[`telemetry/`](../../openviking/telemetry/)、[`metrics/`](../../openviking/metrics/) | 请求上下文、日志追踪、指标采集与导出 | [`observability/context.py`](../../openviking/observability/context.py) |
| [`system_assets/`](../../openviking/system_assets/) | 启动时同步打包的系统资源 | [`sync.py`](../../openviking/system_assets/sync.py) |
| [`utils/`](../../openviking/utils/) | 资源处理、Embedding 工具、Git 认证等共用逻辑 | [`resource_processor.py`](../../openviking/utils/resource_processor.py) |
| [`pyagfs/`](../../openviking/pyagfs/) | Python 侧 AGFS 客户端，负责加载原生 RAGFS 绑定 | [`__init__.py`](../../openviking/pyagfs/__init__.py) |
| [`ttl/`](../../openviking/ttl/)、[`scheduling/`](../../openviking/scheduling/) | Redis TTL 与过期任务运行时 | [`ttl/runtime.py`](../../openviking/ttl/runtime.py) |
| [`eval/`](../../openviking/eval/)、[`usage_reporter/`](../../openviking/usage_reporter/) | 评测与使用量上报扩展点 | [`eval/ragas/`](../../openviking/eval/ragas/) |
| [`client/`](../../openviking/client/) | 兼容导出；实际 HTTP 客户端实现见 `sdk/python/` | [`__init__.py`](../../openviking/client/__init__.py) |

## 关键入口文件

### 服务端启动与组装

| 文件 | 作用 |
|---|---|
| [`pyproject.toml`](../../pyproject.toml) | `[project.scripts]` 定义 `ov`、`openviking`、`openviking-server`、`openviking-gateway`、`vikingbot` 的入口 |
| [`openviking_cli/server_bootstrap.py`](../../openviking_cli/server_bootstrap.py) | `openviking-server` 的轻量入口：先处理 `--config`、`init`、`doctor`、`ingest`，再交给真正的 bootstrap |
| [`openviking/server/bootstrap.py`](../../openviking/server/bootstrap.py) | 解析启动参数、启动 uvicorn、管理 Bot/Expiry 子进程和信号 |
| [`openviking/server/app.py`](../../openviking/server/app.py) | `create_app()`：构造 FastAPI、生命周期、认证插件、路由注册、可选内嵌 Gateway |
| [`openviking/server/routers/__init__.py`](../../openviking/server/routers/__init__.py) | 所有 HTTP Router 的导出总表；新增独立 Router 时还要在 `app.py` 中注册 |
| [`openviking/server/dependencies.py`](../../openviking/server/dependencies.py) | 把 `OpenVikingService` 注入路由的模块级单例 |
| [`openviking/server/auth/__init__.py`](../../openviking/server/auth/__init__.py) | `get_request_context()`：把 API Key、身份请求头解析成 `RequestContext` |
| [`openviking/service/core.py`](../../openviking/service/core.py) | `OpenVikingService`：初始化存储、队列、VikingFS、各子服务和后台调度器 |

### CLI 与 SDK

| 文件 | 作用 |
|---|---|
| [`openviking_cli/rust_cli.py`](../../openviking_cli/rust_cli.py) | `ov` 的 Python 包装：查找并 `exec` Rust 二进制，`doctor` 是 Python 原生子命令 |
| [`crates/ov_cli/src/main.rs`](../../crates/ov_cli/src/main.rs) | Rust CLI 主入口：参数预处理、`clap` 解析、命令分发 |
| [`crates/ov_cli/src/commands/mod.rs`](../../crates/ov_cli/src/commands/mod.rs) | CLI 子命令模块列表，按领域拆分 |
| [`crates/ov_cli/src/handlers.rs`](../../crates/ov_cli/src/handlers.rs) | 命令处理层：本地校验、调用 client、渲染结果 |
| [`crates/ov_cli/src/client.rs`](../../crates/ov_cli/src/client.rs) | CLI 使用的 HTTP client |
| [`sdk/python/openviking_sdk/client.py`](../../sdk/python/openviking_sdk/client.py) | Python SDK 的 `AsyncHTTPClient` / `SyncHTTPClient` 实现 |
| [`sdk/typescript/src/client.ts`](../../sdk/typescript/src/client.ts) | TypeScript SDK 的 `OpenVikingClient` |
| [`sdk/go/client.go`](../../sdk/go/client.go) | Go SDK 的 `Client` 和 `NewClient()` |

补充：`openviking_cli/client/` 是旧导入路径的兼容层，实际实现通过 `_sdk_import` 指向 `sdk/python/openviking_sdk/`；读客户端逻辑时不要停在兼容 shim。

### Gateway、Web Studio 与 VikingBot

| 文件 | 作用 |
|---|---|
| [`openviking_gateway/cli.py`](../../openviking_gateway/cli.py) | `openviking-gateway` 独立进程入口：加载配置、`init`、启动 ASGI app |
| [`openviking_gateway/app.py`](../../openviking_gateway/app.py) | Gateway ASGI 应用：代理、召回、保存、管理 API |
| [`openviking_gateway/proxy.py`](../../openviking_gateway/proxy.py) | 上游选择、请求头处理、绑定解析 |
| [`openviking_gateway/kernel.py`](../../openviking_gateway/kernel.py) | 记忆召回、上下文拼装和对话内核 |
| [`openviking/server/gateway_host.py`](../../openviking/server/gateway_host.py) | 构造内嵌 Gateway 子应用；由 `app.py` 挂载到 `/gateway` |
| [`web-studio/src/main.tsx`](../../web-studio/src/main.tsx) | Web Studio 启动入口：QueryClient、Router、Theme、国际化初始化 |
| [`web-studio/src/routes/__root.tsx`](../../web-studio/src/routes/__root.tsx) | TanStack Router 根路由 |
| [`web-studio/src/app/router/root-layout.tsx`](../../web-studio/src/app/router/root-layout.tsx) | 全局布局与路由边界 |
| [`web-studio/docs/frontend-structure.md`](../../web-studio/docs/frontend-structure.md) | 前端目录职责、依赖方向和验证方式，读前端前先看这篇 |
| [`bot/vikingbot/cli/commands.py`](../../bot/vikingbot/cli/commands.py) | `vikingbot` Typer 命令入口：chat、gateway、渠道管理等 |
| [`bot/vikingbot/agent/loop.py`](../../bot/vikingbot/agent/loop.py) | Agent 主循环 |
| [`bot/vikingbot/channels/manager.py`](../../bot/vikingbot/channels/manager.py) | 多渠道接入管理 |

### 原生层

| 文件 | 作用 |
|---|---|
| [`src/CMakeLists.txt`](../../src/CMakeLists.txt) | C++ 向量引擎的构建入口 |
| [`src/index/index_engine.cpp`](../../src/index/index_engine.cpp) | 向量索引实现 |
| [`src/store/persist_store.cpp`](../../src/store/persist_store.cpp) | 持久化 KV 存储实现 |
| [`src/abi3_engine_backend.cpp`](../../src/abi3_engine_backend.cpp) | Python abi3 扩展的模块入口 |
| [`crates/ragfs/src/lib.rs`](../../crates/ragfs/src/lib.rs) | RAGFS 库入口，导出核心文件系统抽象 |
| [`crates/ragfs-python/src/lib.rs`](../../crates/ragfs-python/src/lib.rs) | RAGFS 的 PyO3 绑定入口 |
| [`openviking/pyagfs/__init__.py`](../../openviking/pyagfs/__init__.py) | Python 侧加载原生绑定并提供 `AsyncAGFSClient` |
| [`openviking/storage/vectordb/engine/_python_api.py`](../../openviking/storage/vectordb/engine/_python_api.py) | Python 侧对 C++ 向量引擎的字段与编码约定 |

## 推荐阅读顺序

下面的顺序按“从简单到复杂、从入口到核心”排列，适合第一次通读。每一步都给出读完应能回答的问题。

### 1. 产品与架构

读 [README_CN.md](README_CN.md) 和 [架构概述](../zh/concepts/01-architecture.md)，再扫一遍 [上下文类型](../zh/concepts/02-context-types.md)、[上下文分层](../zh/concepts/03-context-layers.md) 和 [Viking URI](../zh/concepts/04-viking-uri.md)。

读完应能回答：OpenViking 管哪些上下文、L0/L1/L2 分别是什么、`viking://` 路径长什么样。

### 2. 启动链

按顺序读：

1. [`pyproject.toml`](../../pyproject.toml) 的 `[project.scripts]`
2. [`openviking_cli/server_bootstrap.py`](../../openviking_cli/server_bootstrap.py)
3. [`openviking/server/bootstrap.py`](../../openviking/server/bootstrap.py)
4. [`openviking/server/app.py`](../../openviking/server/app.py)
5. [`openviking/service/core.py`](../../openviking/service/core.py) 的 `OpenVikingService.initialize()`

读完应能回答：`openviking-server` 如何加载配置、创建 FastAPI、初始化存储和队列、启动后台任务。

### 3. 一条简单的 HTTP 路由

从文件系统接口入手，它比资源导入和检索都短：

1. [`openviking/server/routers/filesystem.py`](../../openviking/server/routers/filesystem.py)
2. [`openviking/service/fs_service.py`](../../openviking/service/fs_service.py)
3. [`openviking/storage/viking_fs/`](../../openviking/storage/viking_fs/) 的 `VikingFS`

读完应能回答：HTTP 参数如何变成 `RequestContext` 和 `FSService` 调用、`VikingFS` 如何把 URI 映射到底层文件系统。

### 4. 领域模型和 URI

读：

1. [`openviking/core/context.py`](../../openviking/core/context.py)：`ContextType`、`ContextLevel`、`Context`、`Vectorize`
2. [`openviking/core/namespace.py`](../../openviking/core/namespace.py)：URI 解析、用户空间、可见性
3. [`openviking/core/uri_validation.py`](../../openviking/core/uri_validation.py)：请求 URI 校验
4. [`openviking/core/directories.py`](../../openviking/core/directories.py)：预设目录初始化

读完应能回答：`viking://resources/...`、`viking://~/memories/...` 如何被解析，权限和租户边界在哪些函数里进入。

### 5. 存储抽象

读 [存储架构](../zh/concepts/05-storage.md)，然后：

1. [`openviking/storage/viking_fs/__init__.py`](../../openviking/storage/viking_fs/__init__.py)：`VikingFS` 由多个 mixin 组合
2. [`openviking/storage/viking_fs/_ops.py`](../../openviking/storage/viking_fs/_ops.py)：基础文件操作
3. [`openviking/storage/viking_fs/_semantic.py`](../../openviking/storage/viking_fs/_semantic.py)：`find` / `search` / 摘要读取
4. [`openviking/storage/vikingdb_manager.py`](../../openviking/storage/vikingdb_manager.py)：向量库管理器
5. [`openviking/storage/queuefs/queue_manager.py`](../../openviking/storage/queuefs/queue_manager.py)：命名队列和 worker
6. [`openviking/storage/vectordb/`](../../openviking/storage/vectordb/)：Collection、索引和本地/远端实现

读完应能回答：内容存在哪里、向量索引存在哪里、删除/移动时如何同步两边、异步任务由哪些队列消费。

### 6. 检索链路

读 [检索机制](../zh/concepts/07-retrieval.md)，然后：

1. [`openviking/server/routers/search.py`](../../openviking/server/routers/search.py)
2. [`openviking/service/search_service.py`](../../openviking/service/search_service.py)
3. [`openviking/retrieve/intent_analyzer.py`](../../openviking/retrieve/intent_analyzer.py)
4. [`openviking/retrieve/hierarchical_retriever.py`](../../openviking/retrieve/hierarchical_retriever.py)
5. [`openviking/retrieve/rerank_scoring.py`](../../openviking/retrieve/rerank_scoring.py)
6. [`openviking/retrieve/context_assembler/`](../../openviking/retrieve/context_assembler/)

读完应能回答：`find` 和 `search` 的差异、意图分析何时启用、向量召回和 Rerank 在哪里发生、上下文如何组装。

### 7. 会话与记忆链路

读 [会话管理](../zh/concepts/08-session.md)，然后：

1. [`openviking/server/routers/sessions.py`](../../openviking/server/routers/sessions.py)
2. [`openviking/service/session_service.py`](../../openviking/service/session_service.py)
3. [`openviking/session/session.py`](../../openviking/session/session.py)：`Session`、归档、提交
4. [`openviking/session/compressor_v3.py`](../../openviking/session/compressor_v3.py)：`SessionCompressorV3`
5. [`openviking/session/memory/`](../../openviking/session/memory/)：记忆抽取、合并、更新
6. [`openviking/storage/queuefs/session_commit_processor.py`](../../openviking/storage/queuefs/session_commit_processor.py)：后台提交消费者

读完应能回答：`commit` 为什么分同步和异步两段、归档何时完成、记忆抽取由谁触发、记忆文件如何写回。

### 8. 资源导入链路

读 [上下文提取](../zh/concepts/06-extraction.md)，然后：

1. [`openviking/server/routers/resources.py`](../../openviking/server/routers/resources.py)
2. [`openviking/server/resource_ingest.py`](../../openviking/server/resource_ingest.py)
3. [`openviking/service/resource_service.py`](../../openviking/service/resource_service.py)
4. [`openviking/storage/queuefs/add_resource_processor.py`](../../openviking/storage/queuefs/add_resource_processor.py)
5. [`openviking/utils/resource_processor.py`](../../openviking/utils/resource_processor.py)
6. [`openviking/parse/parser_router.py`](../../openviking/parse/parser_router.py) 和 [`openviking/parse/registry.py`](../../openviking/parse/registry.py)
7. [`openviking/parse/tree_builder.py`](../../openviking/parse/tree_builder.py)
8. [`openviking/storage/queuefs/semantic_processor.py`](../../openviking/storage/queuefs/semantic_processor.py)

读完应能回答：一个 PDF、网页或代码仓库如何被解析、确定目标 URI、写入 RAGFS、生成 L0/L1 并进入向量索引。

### 9. 客户端与 CLI

读：

1. [`sdk/python/openviking_sdk/client.py`](../../sdk/python/openviking_sdk/client.py)
2. [`openviking_cli/rust_cli.py`](../../openviking_cli/rust_cli.py)
3. [`crates/ov_cli/src/main.rs`](../../crates/ov_cli/src/main.rs)
4. [`crates/ov_cli/src/commands/mod.rs`](../../crates/ov_cli/src/commands/mod.rs)
5. [`crates/ov_cli/src/handlers.rs`](../../crates/ov_cli/src/handlers.rs)

读完应能回答：`ov add-resource` 或 `ov find` 如何解析参数、发 HTTP 请求、渲染输出；SDK 与 CLI 的职责边界在哪里。

### 10. 按兴趣继续

| 目标 | 继续读 |
|---|---|
| 修 HTTP API、认证、错误映射 | [`openviking/server/routers/`](../../openviking/server/routers/)、[`openviking/server/auth/`](../../openviking/server/auth/)、[`openviking/server/error_mapping.py`](../../openviking/server/error_mapping.py) |
| 修导入、解析、格式支持 | [`openviking/parse/`](../../openviking/parse/)、[`openviking/utils/resource_processor.py`](../../openviking/utils/resource_processor.py) |
| 修检索、Rerank、上下文预算 | [`openviking/retrieve/`](../../openviking/retrieve/)、[`openviking/storage/viking_fs/_semantic.py`](../../openviking/storage/viking_fs/_semantic.py) |
| 修会话、记忆、归档 | [`openviking/session/`](../../openviking/session/)、[`openviking/service/session_service.py`](../../openviking/service/session_service.py) |
| 修存储、队列、加密 | [`openviking/storage/`](../../openviking/storage/)、[`openviking/crypto/`](../../openviking/crypto/)、[`crates/ragfs/`](../../crates/ragfs/) |
| 修 CLI 或 SDK | [`crates/ov_cli/`](../../crates/ov_cli/)、[`sdk/`](../../sdk/) |
| 修 Gateway | [`openviking_gateway/`](../../openviking_gateway/)、[网关文档](../zh/guides/15-gateway.md) |
| 修 Web Studio | [`web-studio/src/`](../../web-studio/src/)、[前端工程结构](../../web-studio/docs/frontend-structure.md) |
| 修 VikingBot | [`bot/vikingbot/`](../../bot/vikingbot/)、[VikingBot 文档](../../bot/README_CN.md) |
| 修向量引擎或性能 | [`src/`](../../src/)、[`openviking/storage/vectordb/`](../../openviking/storage/vectordb/)、[`benchmark/`](../../benchmark/) |

## 跟真实链路走一遍

只看目录容易迷路，建议至少完整跟两条链路。

### 链路一：资源写入

```text
POST /api/v1/resources
  → resources.py:add_resource
  → resource_ingest.py:ingest_resource_request
  → ResourceService.add_resource / _submit_resource_ingestion
  → QueueManager.ADD_RESOURCE
  → AddResourceProcessor
  → ResourceService 任务处理
  → ResourceProcessor.process_resource
  → ParserRouter / ParserRegistry
  → TreeBuilder.finalize_from_temp
  → VikingFS 写入 RAGFS
  → SemanticQueue / EmbeddingQueue
  → VectorDB
```

关键点：

- 默认情况下请求不等待完整语义处理；响应里的 `task_id` 要用任务接口查询，见[后台任务 API](../zh/api/17-tasks.md)。
- 解析和语义生成是分开的：解析器可能调用模型或外部解析服务，L0/L1 由 SemanticQueue 异步生成。
- 写入完成后才进入向量化；不要把 `add_resource` 返回理解成“已经可检索”。

### 链路二：检索

```text
POST /api/v1/search/find
  → search.py:find
  → SearchService.find
  → VikingFS.find
  → HierarchicalRetriever.retrieve
  → VikingDBManagerProxy（按 RequestContext 限定权限和范围）
  → 向量召回
  → 可选 Rerank
  → FindResult
```

`POST /api/v1/search/search` 会在前面加一段 `IntentAnalyzer`，把查询拆成 0 到多个 `TypedQuery`，再逐条召回。

关键点：

- `target_uri` 和 `RequestContext` 共同限定检索范围，权限不是路由层额外加的装饰。
- `find` 不依赖会话上下文；`search` 可以使用 `session_id` 的摘要和最近消息。
- Rerank 只对召回候选执行一次，不是对每个目录逐层执行。

### 链路三：会话提交（可选）

```text
POST /api/v1/sessions/{session_id}/commit
  → SessionService.commit
  → Session.commit_async
  → 同步归档 + 入队 SessionCommit
  → SessionCommitProcessor
  → SessionCompressorV3.extract_long_term_memories
  → 记忆文件写回 VikingFS + 向量索引
```

关键点：归档阶段在返回前完成，记忆抽取在后台任务中执行；`task_id` 只表示后台任务已创建，不代表已经完成。

## 读代码的实用方法

- **从路由开始，不从仓库根目录开始。** 先找到 `@router.post(...)`，再顺着 service 方法往下追，比从大文件第一行读起有效。
- **先看 `RequestContext` 怎么来。** 绝大多数 service 方法都带 `ctx`；不理解身份、账号、用户和权限，就读不懂为什么查询范围被限制。
- **遇到 mixin 组合的类，拆开读。** 例如 `VikingFS` 由 `_ops.py`、`_semantic.py`、`_vector.py` 等 mixin 组成，先看 `__init__.py` 的组合关系，再进入具体 mixin。
- **异步链路看队列名。** 看到 `QueueManager.ADD_RESOURCE`、`SEMANTIC`、`SESSION_COMMIT` 时，去 [`openviking/storage/queuefs/`](../../openviking/storage/queuefs/) 找对应的 processor。
- **用测试确认行为。** [`tests/server/test_api_resources.py`](../../tests/server/test_api_resources.py)、[`tests/server/test_api_search.py`](../../tests/server/test_api_search.py)、[`tests/server/test_http_client_sdk.py`](../../tests/server/test_http_client_sdk.py) 比注释更接近真实契约。
- **分清生成代码。** `web-studio/src/routeTree.gen.ts`、`web-studio/src/gen/ov-client/` 和构建产物不要手工修改；改源文件或生成脚本。
- **分清兼容层和实现。** `openviking/client/`、`openviking_cli/client/` 是旧路径兼容；新实现优先看 `sdk/python/openviking_sdk/`。

## 配置从哪里读

配置相关代码容易散，按下面顺序找：

1. [`openviking_cli/utils/config/open_viking_config.py`](../../openviking_cli/utils/config/open_viking_config.py)：`OpenVikingConfig` 模型和加载逻辑
2. [`openviking_cli/utils/config/config_loader.py`](../../openviking_cli/utils/config/config_loader.py)：`ov.conf` 路径解析
3. [`openviking/server/config.py`](../../openviking/server/config.py)：`ServerConfig`
4. [`openviking/config/manager.py`](../../openviking/config/manager.py)：运行时配置管理、账号级配置
5. [`examples/ov.conf.example`](../../examples/ov.conf.example)：配置样例

## 本地验证

第一次读完启动链后，建议实际跑一遍：

```bash
uv sync --all-extras
uv run python -c "import openviking; print(openviking.__version__)"
uv run openviking-server init
uv run openviking-server doctor
```

跑聚焦测试：

```bash
uv run pytest tests/server/test_api_resources.py
uv run pytest tests/server/test_api_search.py
uv run pytest tests/server/test_http_client_sdk.py
```

构建相关命令见 [CONTRIBUTING_CN.md](CONTRIBUTING_CN.md) 和 [`Makefile`](../../Makefile)。

## 与其他文档的关系

- 本文回答“代码在哪、从哪读起、调用链怎么走”。
- [架构概述](../zh/concepts/01-architecture.md) 回答“系统由哪些层组成、为什么这样分层”。
- [上下文提取](../zh/concepts/06-extraction.md)、[检索机制](../zh/concepts/07-retrieval.md)、[会话管理](../zh/concepts/08-session.md) 回答“每条链路的产品语义和边界”。
- [CONTRIBUTING_CN.md](CONTRIBUTING_CN.md) 回答“怎么搭环境、怎么改、怎么提交 PR”。
- [前端工程结构](../../web-studio/docs/frontend-structure.md) 回答 Web Studio 的目录职责和依赖规则。
- [tests/README.md](../../tests/README.md) 回答测试目录和常用测试命令。

如果发现目录职责、入口文件或调用链已经变化，请直接更新本文对应小节，并在 PR 中说明核对范围。
