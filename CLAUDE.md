# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

琅嬛（Langhuan）是 RAG 工程中的知识处理与检索层：把 pdf/docx/markdown/txt/csv/xlsx 及 FAQ、网页、飞书文档转为可检索、可向量化、可追溯的结构，通过 REST 与 MCP over HTTP 对外服务。**不生成 LLM 答案、不编排 Chat/Agent**。单一二进制同时承载 REST、MCP、asynq worker 和内嵌 Web Console（`web_embed` tag）。

完整开发规范见 `AGENTS.md`；涉及数据库的任务必读 `docs/DATABASE_GUIDELINES.md`；架构细节见 `docs/ARCHITECTURE.md`；版本范围见 `ROADMAP.md`。

## 常用命令

```bash
# 运行（零配置 SQLite 单机模式，数据落 ~/.langhuan-data/）
make standalone            # = go run ./cmd/langhuan
make dev                   # 用 ./config.local.yaml 运行
go run ./cmd/langhuan migrate   # 单独执行迁移

# 测试
go test ./...              # 单元测试（无 DSN 时 DB 测试自动 skip）
make test-integration      # 集成测试：自动构建测试镜像 + 临时 pgvector 容器 + 跑完销毁
make test-image            # 手动跑单个 integration package 前必须先执行一次
go test -tags=integration ./internal/infrastructure/db -count=1   # 单 package 集成测试（需先有临时容器并注入 LANGHUAN_TEST_DATABASE_DSN）
make test-sqlite           # SQLite 相关单元测试（不依赖 Docker）
go test ./internal/application/service/ -run TestXxx -count=1      # 单个测试

# 格式化与静态检查
gofmt -w .
go vet ./...

# 检索评测（改 chunker/RRF/检索默认参数/embedding 链路时必须跑并附对比）
make eval-smoke            # 离线冒烟，mock embedding，无网络依赖
make eval                  # 真实评测（首次下载约 730MB 数据集；需 eval.config.yaml）

# Web 前端（web/ 目录内）
pnpm dev / pnpm build      # build 含 tsc -b 类型检查
pnpm check / check:fix     # Biome lint + format（不要引入 ESLint/Prettier）
pnpm test                  # vitest（browser 模式）
make e2e-web               # 全栈冒烟：前端构建 + 内嵌 SPA 后端 + Playwright

# 发布构建
make linux                 # 构建内嵌前端的 linux/amd64 二进制到 bin/
```

提交遵循 Conventional Commits，主题与正文以中文为主。

## 架构

### DDD 分层（依赖方向不可逆）

```
interfaces/  (http / mcp / worker)   只做协议转换、身份提取、错误映射
    ↓
application/ (service / pipeline / dto)  业务编排、事务边界、RRF 融合
    ↓
domain/      (model / value / errors)    纯 struct，零外部依赖（无 GORM/HTTP tag）
ports/       使用方（application）定义接口
adapters/    实现 ports（MinerU、embedding、storage、parser、queue）
infrastructure/db   Repository 实现（GORM 薄封装）+ Row 模型 + 迁移
```

- 领域层与 Application 层**不得持有 `*gorm.DB`**；领域模型与 GORM Row 双层分离（`toRow`/`fromRow`）。
- Repository 内 `gorm.ErrRecordNotFound` → `domainerrors.ErrNotFound`；HTTP 层做错误→状态码映射。
- DDD 适度：不为臆想的未来预留 port；简单查询允许简单实现（见 AGENTS.md 5.7）。

### 核心数据流

```
导入（File/FAQ/Web/飞书）
  → 事务：Document + 不可变 Revision + Job
  → asynq 异步链：解析（MinerU Cloud for PDF）→ 分块 → embed + stage RetrievalEntry
  → 原子发布到 KB 唯一 active Generation（双缓冲，切换指针）
检索：pgvector topK + FTS(zhparser) topK → application 层确定性 RRF → 按父块聚合出 evidence
```

关键概念：

- **Document/Revision**：`kind`（file/faq/web）不可变；文件内容、hash、解析产物全在 Revision 上，重传=新 Revision。
- **Chunk 角色**：父子模式（默认）`parent` 只作返回上下文、`child` 是召回单元；关闭时只有 `flat`。父块永不产生 RetrievalEntry。
- **Generation**：KB 的不可变索引快照，单活双缓冲；构建在 inactive 代完成，激活时校验后仅切指针。
- **SearchRun**：每次检索落稳定 `search_id` + 引用血缘（CitationRef），绝不落 query 明文；支持管理员回放。
- **Workspace**：租户与数据隔离边界，所有业务表带 `workspace_id`，查询必须显式过滤。

### 双数据库与队列

- 生产：PostgreSQL 17 + pgvector（halfvec/HNSW）+ zhparser FTS + Redis/asynq。
- standalone：SQLite（modernc.org/sqlite + sqlite-vec + FTS5 + gse），进程内队列；方言分流只在 Repository SQL 层（`db.Dialect`），domain/application 不感知。
- 异步任务必须幂等：执行前检查 ready/published/terminal 状态；worker handler 只解码并调用 application service。

### Web 前端（web/）

React 19 + TypeScript + Vite + TanStack Router/Query + Tailwind v4 + shadcn/ui + RHF + Zod + Biome + Zustand。服务端状态一律 TanStack Query（不用 useEffect 手写请求）；表单 RHF+Zod；禁止 `any`；`src/routes/routeTree.gen.ts` 是生成文件禁止手改；`components/ui/` 是 shadcn 生成物尽量不手改。

## 铁律

1. **测试数据库隔离**：任何自动化测试需要数据库时，只能用运行期临时拉起的 docker pgvector 容器（`LANGHUAN_TEST_DATABASE_DSN` 注入），**严禁连 `config.yaml` 的库或本机 `localhost:5432/langhuan`**。干净机器（仅 docker）上 `go test ./...` 必须可直过。
2. **迁移**：`internal/infrastructure/migrate/migrations/` 下成对 up/down SQL，版本号严格递增，优先幂等。
3. **向量索引**：`chunk_embeddings.embedding` 为 halfvec；查询侧 cast/distance/维度谓词必须与 `000001_init` 预建的 HNSW 部分索引完全一致，否则退化为全表扫描；维度仅允许 798/1024/2048/3584。
4. **评测回归**：修改 chunker、RRF、默认检索参数或 embedding 链路的 PR，须附 `make eval` 新旧 `metrics.json` 对比或说明为何不适用。
5. **安全**：远程图片下载必须用 SSRF-safe HTTP client；日志不得记录 API key、文档正文、向量、凭证。
6. **Context 贯穿**：所有 I/O 透传 `context.Context`；错误用 `errors.Is/As` 判断，每层只包一次上下文。
7. **配置集中**：超时/重试/批量大小等不硬编码，进 `config.yaml`（`config.example.yaml` 为样例）或具名常量；业务代码不直接 `os.Getenv`。
