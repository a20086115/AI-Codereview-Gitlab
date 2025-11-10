# 系统架构说明

本文档概述 `AI-Codereview-Gitlab` 的核心组件、运行流程与关键技术栈，帮助开发者快速理解整体架构并高效扩展。

## 1. 总体架构

系统围绕 “Webhook 事件 → AI 审查 → 通知 & 可视化” 展开，主要由以下子系统组成：

- **API 接入层（Flask）**：`api.py` 提供统一的 HTTP 入口，接收 GitLab/GitHub Webhook、触发日报任务，并负责参数校验、调度逻辑。
- **任务调度层（Queue/多进程）**：`biz.utils.queue.handle_queue` 将 Webhook 事件异步派发至 `biz.queue.worker`，可通过多进程或 Redis+RQ 运行，避免阻塞 HTTP 请求。
- **业务处理层（Handlers + Service）**：`biz/gitlab`、`biz/github` 模块解析事件上下文，`biz/utils/code_reviewer.py` 调用大模型审查代码，`biz/service/review_service.py` 负责日志入库。
- **大模型接入层（LLM Factory）**：`biz.llm.factory.Factory` 根据 `LLM_PROVIDER` 动态选择 DeepSeek、智谱、OpenAI、通义千问、Ollama 等客户端，实现多模型切换。
- **通知与消息通道**：`biz.utils.im` 下的多种 Notifier 将审查结果推送至钉钉、企业微信、飞书或自定义 Webhook。
- **数据存储层**：使用 `sqlite3` (`data/data.db`) 存储 Merge Request/Push 审查日志，并支持数据可视化检索。
- **可视化 Dashboard（Streamlit）**：`ui.py` 读取审查数据，提供登录、统计图表、明细列表等交互界面。
- **批处理任务**：通过 APScheduler (`/review/daily_report`) 定时生成日报，并复用通知系统发送到 IM。

## 2. 关键目录结构

- `api.py`：HTTP 入口、Webhook 接收与调度、日报接口。
- `biz/queue/worker.py`：Webhook 事件实际处理逻辑，包含 GitLab/MR、Push 与 GitHub 对应处理函数。
- `biz/gitlab`、`biz/github`：针对不同平台的 Webhook Handler，负责调用平台 API 获取 diff、提交、受保护分支等。
- `biz/llm/client/*`：封装各大模型 SDK 的调用，统一向上提供 `completions()` 能力。
- `biz/utils/`：
  - `code_reviewer.py`：加载提示词、管理 Token、调用大模型生成审查结果。
  - `queue.py`：抽象队列驱动（内置 async / RQ）。
  - `im/`：钉钉、企业微信、飞书、自定义 Webhook 推送实现。
  - `reporter.py`：日报内容渲染。
  - `config_checker.py`：启动前检查关键配置。
- `biz/service/review_service.py`：SQLite 表结构、读写封装、事件去重（`last_commit_id`）。
- `conf/`：环境配置、Prompt 模板、Supervisor 配置。
- `ui.py`：Streamlit Dashboard。

## 3. 核心流程

### 3.1 Merge Request / Pull Request 审查

1. GitLab/GitHub 推送 Webhook → `api.py` 通过 `handle_queue` 异步派发。
2. `biz.queue.worker.handle_merge_request_event` / `handle_github_pull_request_event`：
   - 读取 MR 元数据、过滤 Draft/非监控分支。
   - 通过 Handler 调用平台 API 获取变更 diff、commit 列表。
   - 使用 `filter_changes()` 保留受支持的文件类型、统计增删行。
   - 组合 commit 信息与 diff，调用 `CodeReviewer().review_and_strip_code()` 发送到 LLM。
   - 将生成的 Markdown 结果写回 MR Note，并解析评分。
   - 触发 `merge_request_reviewed` 事件 → 通知 IM、写入 SQLite。

### 3.2 Push 审查

1. Webhook 异步派发至 `handle_push_event` / `handle_github_push_event`。
2. 根据 `PUSH_REVIEW_ENABLED` 决定是否执行 AI 审查。
3. 通过平台 API 对比 before/after Commit，获取 diff → 过滤、调用 LLM → 写回 Commit 评论。
4. 发送 `push_reviewed` 事件：推送通知、日志入库。

### 3.3 日报生成

1. API 启动时执行 `setup_scheduler()`，依据 `REPORT_CRONTAB_EXPRESSION` 定时触发。
2. `daily_report()` 调用 `ReviewService` 查询当天审查记录 → `Reporter` 渲染文本。
3. 通过 Notifier 推送 Markdown 报告到 IM。

## 4. 配置与运行时

- **环境变量**（摘录）：
  - `LLM_PROVIDER`：选择大模型供应商。
  - `SUPPORTED_EXTENSIONS`：审查的文件后缀白名单。
  - `PUSH_REVIEW_ENABLED` / `MERGE_REVIEW_ONLY_PROTECTED_BRANCHES_ENABLED`：控制审查策略。
  - `QUEUE_DRIVER`：`async`（默认多进程）或 `rq`（Redis 队列）。
  - `DINGTALK_ENABLED` 等：通知开关。
  - `DASHBOARD_USER/PASSWORD/SECRET_KEY`：Dashboard 登录配置。
- **依赖服务**：
  - 当 `QUEUE_DRIVER=rq` 时需 Redis。
  - Streamlit Dashboard 可通过 `streamlit run ui.py --server.port=5002` 启动。
  - Docker 部署时使用 `docker-compose.yml`，包含 API、Worker、Dashboard。

## 5. 数据与存储

- SQLite 数据库 `data/data.db`：
  - `mr_review_log`：记录每次 Merge Request 审查结果、增删行、最终评分等。
  - `push_review_log`：记录 Push 事件审查结果。
  - 初始化时自动创建表结构、索引，并兼容旧版本字段。
- 数据查询在 Dashboard、日报、米数统计中复用。

## 6. 扩展建议

- **新增大模型**：在 `biz.llm.client` 实现新的 Client，注册到 `Factory.getClient()` 中。
- **扩展通知渠道**：在 `biz.utils.im` 下新增 Notifier，并在 `notifier.send_notification` 中调用。
- **队列拓展**：如果需要更复杂的任务调度，可引入 Celery 等，引导 `handle_queue` 接入新的驱动。
- **数据库迁移**：对于高并发场景，可将 `ReviewService` 改写为云数据库或消息中间件。
- **前端展示**：`ui.py` 基于 Streamlit，可按需替换成 Web 框架或嵌入公司现有平台。

通过本架构设计，系统实现了从代码变更捕获、AI 审查、通知反馈到可视化分析的闭环，并提供丰富的扩展点以适配不同团队的工作流程。

