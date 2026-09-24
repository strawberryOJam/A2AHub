# workhub → Java 端口设计规格

日期：2026-09-24
状态：待评审
目标仓库：`A2AHub`（`/Users/echoyan/Developer/A2AHub`）
行为来源：`workhub`（`/Users/echoyan/Developer/workhub`，Python）

---

## 1. 背景与目标

### 1.1 动机

`workhub` 是一个已经跑通的 Python 服务：自助身份 + 逐级审批。员工 agent 提交工作或日报，直属领导初审、老板终审，MCP 是主要接口。

换语言的原因只有一条：**Python 难以阅读，长期维护困难**。这是一次**换实现语言**，不是重新设计。

### 1.2 目标

1. 把 `workhub` 的服务端**保行为地**用 Java 重写，落在 `A2AHub` 仓库。
2. 配套的两个 skill（`workhub-workflow`、`workhub-mcp-connect`）**重新编写**——原版初次引导实测失败，用户卡了很久都没能真正开始。

### 1.3 成功标准

- Java 服务能顶替 workhub 现有职责：`tests/test_sequential.py` 的 35 个场景中实际端口 31 个（取舍见 9.2），全部通过。
- 现场部署脚本、备份恢复流程照旧可用。
- 客户端（员工机器上那份 `mcpServers` 配置）**不用改**就能连上。
- 代码结构对**不熟悉 Python 的维护者**可读——这是本次的实质目标，也是选择"按领域重新分层"而非"逐模块直译"的原因。

### 1.4 非目标

- 不改领域模型。任何偏离 workhub 行为的地方，都在第 11 节逐条列出并已获确认。
- 不做前端。workhub 没有 UI；旧规格里的 React+Vite 属于另一套设计，已随旧文档删除。
- 本次不做 skill 重写（见第 13 节阶段划分）。

### 1.5 前置决策（已确认）

| 决策 | 结论 |
|---|---|
| 数据库 | **可以从空库重来**，不接存量 workhub 库 |
| Python 版 | **退役**，不需要并行运行、不需要数据迁移 |
| 落点 | `A2AHub` 仓库；`workhub` 的业务逻辑是唯一事实来源 |
| 旧文档 | 用户已自行删除（`3a5dd39 delete(doc): 不需要`），本规格不恢复、不引用 |

---

## 2. 技术选型

| 项 | 选择 | 理由 |
|---|---|---|
| Java | 21 | 仓库既有 |
| 构建 | Maven | 仓库既有 |
| 框架 | Spring Boot 4.1.1 | 仓库既有 |
| MCP | Spring AI 2.0.1 `spring-ai-starter-mcp-server-webmvc` | 见下 |
| 数据库访问 | `JdbcClient` + Java record，**不用 JPA/Hibernate** | 见 4.3 |
| 迁移 | Flyway（`flyway-core` + `flyway-database-postgresql`） | 取代自研迁移 |
| 数据库 | PostgreSQL 17 | 与现网一致 |
| 测试 | JUnit 6 + Testcontainers（`postgres:17-alpine`） | 与 compose 同版本 |
| 序列化 | Jackson 3（`tools.jackson.databind`） | Spring Boot 4 默认 |
| Lombok | **移除** | record 覆盖绝大部分场景，留着只会混入两种风格 |

### 2.1 MCP 可行性（已验证）

**Spring AI 2.0.1 的 starter 直接依赖 `spring-boot-starter-web` 4.1.1 —— 与仓库现有 Spring Boot 版本完全相同。** 不存在版本兼容风险。

- 协议必须用 `STREAMABLE`，**不能用 `STATELESS`**：无状态传输对 GET 返回 405，而 Claude Code 2.1.84+ 会先发 GET 再 POST，会直接连不上。
- 端点路径通过 `spring.ai.mcp.server.streamable-http.mcp-endpoint` 配置（默认 `/mcp`）。
- 传输层默认**不鉴权**，与 workhub 现有行为一致（认证在工具层）。这是有意保持，不是疏漏。
- 工具声明用 `@Tool` + `MethodToolCallbackProvider`。

参考：
- <https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html>
- <https://docs.spring.io/spring-ai/reference/api/mcp/mcp-streamable-http-server-boot-starter-docs.html>
- <https://github.com/anthropics/claude-code/issues/39790>

---

## 3. 现状盘点（端口面）

来源 `workhub` 实测数据：

| 面 | 规模 |
|---|---|
| 服务端 Python | `app/` 共 2,985 行。其中九个主文件 2,380 行：`service` 628 / `api` 282 / `identity` 249 / `config` 246 / `models` 232 / `auth` 228 / `notify` 225 / `mcp_server` 166 / `storage` 124；余下 605 行为 `legacy` 160 / `migrate` 129 / `cli`、`main`、`healthcheck`、`maintenance`、`idempotency` 52 等 |
| 数据库 | 14 张表、1 个迁移版本 |
| REST | `api.py` 中 29 条路由 |
| MCP | 20 个工具（18 个实体 + 2 个兼容别名 `whoami`、`read_text_file`） |
| 状态机 | 2 套（`sequential_v1` 与 `legacy_single_stage`） |
| 契约测试 | `tests/test_sequential.py` 669 行、35 个测试方法 |
| 部署 | 多阶段 Dockerfile、compose、`deploy.sh`（11 个子命令）、备份脚本与 systemd 定时器 |
| skill | 2 个 SKILL.md + 双语 metadata + references + `mcp_connect.py`（927 行） |

### 3.1 legacy 状态机的处置

`legacy_single_stage` 存在的唯一理由是让**迁移之前的老数据**仍可读取：

- `app/legacy.py`（160 行）
- `app/service.py` 中 11 处 `if workflow_mode == "legacy_single_stage"` 分支
- `app/migrate.py`（129 行）的版本闸门
- `tests/test_sequential.py` 中 15 行涉及 legacy / 迁移

数据库可清空重来 ⇒ **整套不端口**。

---

## 4. 架构与模块划分

### 4.1 目录结构

```
com.echoyan.a2ahub
├── A2AHubApplication
├── config/          配置属性 + Spring 装配
├── common/          跨领域的横切机制
│   ├── error/       错误码枚举 + 异常体系 + 全局处理
│   ├── audit/       操作审计
│   └── idempotency/ 幂等闸门
├── identity/        身份：资料、凭据、登记、鉴权
├── work/            工作项领域（核心）
├── file/            附件存储
├── notify/          通知投递
└── api/
    ├── rest/        REST 适配器
    └── mcp/         MCP 适配器
```

### 4.2 分层规则

`api` → 领域模块 → `common`。领域模块之间不直接查对方的表，需要数据就调对方的 service。

`api` 下的适配器**只做三件事**：取身份、转参数、包错误。任何适配器里出现业务判断即为设计违规。

### 4.3 三个关键取舍

**① `work` 与 `approval` 不拆成两个包。**

`review_work` 一次审批同时修改三张表：`approval_steps.status`、`work_submissions.status`、`work_items.status`（`service.py:418-431`），且推进靠"把 `waiting` 中最小的一步改为 `pending`"。这三张表在业务上是**一个聚合**；拆开会产生环依赖（work 要问 approval 算状态，approval 要问 work 拿上下文）。

`work` 包内部分四块：

```
work/
├── WorkItem, WorkStatus, WorkRepository        工作项与其状态
├── Submission, ApprovalStep, Review            提交快照与审批链
├── WorkflowStateMachine                        纯函数 · 无 DB
└── ApprovalChainPlanner                        纯函数 · 无 DB
```

后两个无数据库依赖，是整个系统最应被一眼读懂的部分。

**② 不用 JPA / Hibernate。**

三条硬理由：

- `uq_current_step`、`uq_daily_active` 是**部分唯一索引**，JPA 无法表达。
- 幂等依赖 `pg_advisory_xact_lock`。
- `query_works` 需精确控制语句数（`test_list_query_count_is_flat`：查 50 条必须少于 20 条 SQL）。

且对"可读"这一目标而言 JPA 是负分：SQL 藏进注解与派生方法名，维护者更难看懂。SQL 全部写在明面上。

**③ 领域对象用 Java 21 record。**

数据库行映射为不可变 record，不需要 Lombok 的 getter/setter 噪音。

---

## 5. 核心领域

### 5.1 状态机

`work_items.status` 取值：`draft` / `in_review` / `approved` / `rejected` / `withdrawn` / `recorded` / `closed`。

以下转移表由 `service.py` 中全部 7 处 `_state()` 守卫逐条核实得出（行号 `295 / 320 / 368 / 410 / 461 / 469 / 478`）：

| 状态 | 允许的动作 |
|---|---|
| `draft` | 编辑、传附件、提交、作废 |
| `in_review` | 审批、撤回 |
| `rejected` | 返工、作废 |
| `withdrawn` | 返工、作废 |
| `approved` / `recorded` / `closed` | 无（终态） |

```java
public enum WorkStatus { DRAFT, IN_REVIEW, APPROVED, REJECTED, WITHDRAWN, RECORDED, CLOSED }
public enum WorkAction { EDIT, ATTACH, SUBMIT, REVIEW, WITHDRAW, REWORK, CLOSE }

/** 唯一的真相：哪个状态允许哪个动作。整个状态机就这七行。 */
static final Map<WorkStatus, Set<WorkAction>> ALLOWED = Map.of(
    DRAFT,     EnumSet.of(EDIT, ATTACH, SUBMIT, CLOSE),
    IN_REVIEW, EnumSet.of(REVIEW, WITHDRAW),
    REJECTED,  EnumSet.of(REWORK, CLOSE),
    WITHDRAWN, EnumSet.of(REWORK, CLOSE),
    APPROVED,  EnumSet.noneOf(WorkAction.class),
    RECORDED,  EnumSet.noneOf(WorkAction.class),
    CLOSED,    EnumSet.noneOf(WorkAction.class));
```

两处**照搬不改**的行为：

- 老板提交时无审批人，直接进 `recorded`，**不经过 `in_review`**（`service.py:377`）。
- `approved` **没有出边**——已通过的工作不能作废。看似遗漏，但这是现状。

`approval_steps.status` 取值：`waiting` / `pending` / `approved` / `rejected` / `canceled`（已有 `ck_step_status` 约束）。

### 5.2 审批链

规则（`identity.py:91`）：

| 作者角色 | 审批链 |
|---|---|
| `staff` | 直属 `manager` → `boss` |
| `manager` | `boss` |
| `boss` | 空 |

实现为沿 `supervisor_profile_id` 向上遍历，逐级校验角色（员工要 manager，领导要 boss），带 `seen` 集合防环。链上每级的 `stage` 为 `manager_review` / `boss_review`，`step_no` 从 1 开始，**第一步 `pending`、其余 `waiting`**——这是审批链串行推进的全部机制。

**由抛异常改为返回值**：

```java
public sealed interface ChainPlan {
    record Ready(List<AgentProfile> approvers) implements ChainPlan {}
    record NotReady(String reason) implements ChainPlan {}   // 上级未登记 / 角色不符 / 成环
}
```

理由：「链没建好」是**正常业务状态**而非异常——`get_my_context` 把它翻译成"缺谁"报给用户，`submit_work` 把它变成拒绝。原实现靠 `try/except` 区分两条路。改为返回值后控制流显式，不靠异常传递。

### 5.3 幂等

契约（`app/idempotency.py`）：

> **一个事务里同时完成：业务改动 + 审计 + outbox + 可重放的结果。**

```java
public <T> T execute(Principal p, String operation, String requestId, Object params, Supplier<T> handler)
```

四件必做事：

1. `request_id` 必须是 1–128 字符，否则 `INVALID_REQUEST`。
2. 计算 `sha256(canonical(params))`。
3. 取 `pg_advisory_xact_lock`：**取 sha256 前 8 字节当作有符号整数**（此细节必须照搬，否则锁的不是同一条）。
4. 命中旧记录时：哈希不同 → `IDEMPOTENCY_CONFLICT`；哈希相同 → **原样返回存下的 response**。

键：`(actor_scope, operation, request_id)`；`actor_scope` 为 `profile_id`，登记场景（尚无 profile）为 `"register:" + token_fingerprint`。

`handler` 必须运行在同一事务内（`TransactionTemplate`）。

一处自然简化：原实现在重读 profile 时用了 `populate_existing=True` 以打穿 SQLAlchemy 的 identity map 缓存（`idempotency.py:31`）。`JdbcClient` 无 identity map，该问题自动消失。

### 5.4 并发控制

**行锁与版本号两者都要，职责不同：**

| 机制 | 挡什么 |
|---|---|
| `SELECT ... FOR UPDATE` | 两个请求同时进来 |
| `lock_version` 比对 | 客户端拿陈旧版本号来写 |

原实现已经在用行锁（`service.py:25` 的 `with_for_update()`），**照搬 `SELECT ... FOR UPDATE`，不改**。

`update_profile` 使用独立计数器 `profile_revision`，机制相同。

### 5.5 一处需要注释说明的镜像

`work_submissions.status` 与 `work_items.status` 由同一次状态转移一起写入（`service.py:426`、`431`、`466`）。两个字段存同一份真相。

**不修改该设计**（改了即偏离现状），但在 `Submission` 上注明：**其 status 是 work 状态的镜像，随同一次转移写入**，避免后来者误以为可单独修改。

---

## 6. 数据与迁移

### 6.1 删除的列与表

每一列均已追溯"谁在读它"：

| 删除 | 依据 |
|---|---|
| `agent_profiles.legacy_role`、`legacy_group_id` | 仅被 `service.py:33` 的 `_legacy_principal` 与 `migrate.py:65` 读写，两者均 legacy 专属 |
| `agent_profiles.claim_token_fingerprint` | **全仓库零个读取方**；声明为 `NOT NULL` 但无任何使用 |
| `agent_profiles.token_fingerprint` + 其唯一索引 | 仅在 `auth.py:148-150` 的兜底分支被读，该行注释自述为"Legacy fingerprints remain usable during the migration window" |
| `work_items.workflow_mode` | legacy 状态机判别字段 |
| `work_items.archived` | 仅被 `legacy.py:143` 读；注释为"预留：WeKnora 归档"，从未落地 |
| 表 `schema_versions` | 由 Flyway 的 `flyway_schema_history` 取代 |

删除 `token_fingerprint` 后的收益：**`agent_credentials` 成为唯一凭据存储**。原 `auth.py:141-150` 是"先查 `agent_credentials`，查不到再拿 fingerprint 兜底"的双路径，此后只剩一条。

### 6.1.1 连同删除的配置面

删 legacy 会连带让两处配置失效，须一并清理，否则留下"配了但没人读"的死键：

| 删除 | 依据 |
|---|---|
| `features.archive_to_weknora` 配置键 | 全仓库只有 `config.py:213-214` 读它，而那段代码**只做类型校验**（非 bool 则置 False），没有任何功能消费它。它是"终态归档到 WeKnora"这个从未落地的功能留下的占位，与 `work_items.archived` 是同一件事的两半——列已删，键不应独留 |
| 环境变量 `WORKHUB_LEGACY_TIMEZONE` | 全仓库唯一读取点是 `app/migrate.py:46`，即迁移过程解释老时间戳用的。`migrate.py` 整体不端口，该变量随之消失。`deploy/` 中无人设置它，故删除不影响部署脚本 |

剩余 13 张表：`agent_profiles`、`agent_credentials`、`work_items`、`work_files`、`work_submissions`、`submission_files`、`draft_files`、`approval_steps`、`reviews`、`audit_logs`、`operation_requests`、`webhook_subs`、`delivery_logs`。

### 6.2 保留不动的三样

**`setup_complete` + `pending_supervisor_employee_id`** —— "上级尚未登记"的整套机制。后者存的是**上级的工号**（上级尚未登记故拿不到 profile_id），`daily_status` 靠它反查"谁的下属已上线"（`service.py:619-622`）。审批链能否建立依赖这两个字段。

**`verified` + `source`** —— `models.py` 注释自述为"后续收紧的挂钩：要改成『领导需管理员确认』时，只需把校验挂到 verified 上，不必改数据结构"。这是**有意的扩展点**，非残留。

**`config.yaml` 中的 `tokens`** —— 部署方身份的路径，与自助登记并行。全局 webhook 仅认"直接配置的 boss 且 `group_id: "*"`"。且 `auth.py:116-120` 有一段**故意报错**的逻辑：配置中残留 `claimable` / `__unclaimed__` 旧条目时，宁可在认证这一跳失败，也不让它"当普通令牌凑合放行"。该行为（废旧配置要吵，不要静默降级）原样保留。

### 6.3 Flyway 接管

现状的结构有三个来源：`db/schema.sql`（自述 "Generated reference schema v1"）、`models.py` 的 `Base.metadata.create_all`、`migrate.py` 手工补索引与约束。Java 版收敛为一个：

```
src/main/resources/db/migration/
└── V1__baseline.sql        由 db/schema.sql 改写，去掉 6.1 所列列与表
```

`db/schema.sql` 是可靠底稿：部分唯一索引（`uq_daily_active`、`uq_current_step`、`uq_delivery_event`）与 5 个既有 CHECK 均已包含。**改写而非直接拷贝**，因为需要删列、补约束、整理格式。

依赖：`flyway-core` + `flyway-database-postgresql`（Flyway 10 起 PostgreSQL 支持拆为独立模块，缺后者会在启动时报找不到方言）。

### 6.4 补充的 CHECK 约束

现有 5 个 CHECK：`ck_work_kind`、`ck_report_date`、`ck_work_versions`、`ck_step_status`、`ck_no_self_supervisor`。

补充 9 个（当前均为裸 VARCHAR，合法取值只存在于 Python 注释中）：

| 列 | 取值 |
|---|---|
| `work_items.status`、`work_submissions.status` | `draft` / `in_review` / `approved` / `rejected` / `withdrawn` / `recorded` / `closed` |
| `agent_profiles.status` | `active` / `inactive` |
| `agent_profiles.claimed_role` | `staff` / `manager` / `boss` |
| `agent_profiles.source` | `self_claim` / `admin` |
| `agent_credentials.status` | `active` / `revoked` |
| `delivery_logs.status` | `pending` / `sending` / `ok` / `failed` / `dead` |
| `reviews.verdict` | `pass` / `fail` / `comment` |
| `webhook_subs.type` | `generic` / `dingtalk` |

**为何现在可以补**：原库有历史数据，加约束可能导致旧行无法读出。空库重来后 Java 版是唯一写入方，约束只会在代码写错时当场失败。

**风险**：若 Java 代码试图写入范围外的值，行为由"静默写入"变为"抛约束异常"。这是**故意的**。9 条约束与对应 Java enum 一一对应、可互相对照。

**注意 `delivery_logs.status` 一列的教训**：`models.py:144` 的注释写作 `# pending | ok | failed`，**该注释是错的**——实际写入的是五个值，代码中的 `sending`（`notify.py:158`）与 `dead`（`notify.py:207`）均未出现在注释里。若按注释建约束，上线首次投递失败即会失败。这正是本次把取值从注释搬进 enum 与约束的直接动因。

**一处不补**：`delivery_logs.event_id` 为 nullable 却出现在唯一索引 `uq_delivery_event (event_id, sub_id)` 中。PostgreSQL 中 NULL 不参与唯一性比较，故 `event_id` 为 NULL 的行可重复插入。分析见 8.2。

---

## 7. 两个入口

### 7.1 形状

```
api/
├── rest/    ProfileController、WorkController、FileController、
│            WebhookController、HealthController
└── mcp/     ProfileTools、WorkTools、FileTools
        ↓  两者调用同一批
   identity/ · work/ · file/ · notify/ 的 service
```

### 7.2 工具名与参数名是契约

Spring AI 从 Java 方法签名**自动生成 JSON Schema**。因此方法名、参数名、类型、默认值必须与 `mcp_server.py` 中的定义**逐字对应**：

```python
def review_work(work_id: str, submission_id: str, step_id: str,
                verdict: Literal["pass","fail"],          # → Java enum Verdict
                expected_lock_version: int, request_id: str,
                comment: str = "") -> dict
```

`Literal[...]` → Java enum；`str | None = None` → 可空参数。20 个签名一比一还原，不改名、不改顺序、不加字段。

### 7.3 错误的呈现

**MCP 侧**：`ServiceError` 被捕获后 `raise RuntimeError(canonical(exc.as_dict()))`（`mcp_server.py:22-35`）——错误以 **JSON 字符串**形式钻出，客户端读到 `{"code": "...", "message": "..."}`。`AuthError` / `StorageError` 走同一路径。Java 版抛携带同样 JSON 的 `ToolException`；**JSON 形状不可变**，它是客户端判断"该重读还是该重试"的唯一依据。

**REST 侧**：现状有三种形状——

| 来源 | 形状 |
|---|---|
| `ServiceError` | `{"code": "INVALID_STATE", "message": "..."}` |
| `HTTPException(413, "...")` | `{"detail": "..."}` |
| FastAPI 参数校验 | 422 + 原始数组 |

**统一为 `{code, message}` 一种**，由 `GlobalExceptionHandler` 一处收口。这是**客户端可见的改动**；唯一消费方 `mcp_connect.py` 本来就要重写，故成本为零。

### 7.4 一处有意保留的设计

`mcp_server.py:54-56` 的注释：

> 自助登记没有 MCP 工具，这是有意的：要调 MCP 工具，得先把密钥写进客户端配置并被请求带出来，而新密钥在登记成功前认不出任何身份 —— 先有鸡还是先有蛋。所以登记固定走 REST。

该判断正确且易被后来者"顺手补全"。Java 版将这段原样写成注释留在 `McpTools` 上，并且**不添加任何登记类工具**。

### 7.5 Bearer 如何进入工具层（已查证定案）

原实现用 contextvar（`principal_from_context()`），由中间件每请求填充。Java 无 contextvar。以下依据 **Spring AI 2.0.1 源码**，非文档推测。

**第一段：传输层注入。必须自己声明 transport provider bean。**

```java
@Bean
WebMvcStreamableServerTransportProvider mcpTransport(
        @Qualifier("mcpServerJsonMapper") JsonMapper jsonMapper,
        ServerTransportSecurityValidator security) {
    return WebMvcStreamableServerTransportProvider.builder()
        .jsonMapper(new JacksonMcpJsonMapper(jsonMapper))
        .mcpEndpoint("/mcp")
        .securityValidator(security)                    // 见 7.6
        .contextExtractor(req -> {
            String auth = req.headers().firstHeader("Authorization");
            Map<String, Object> md = new HashMap<>();   // Map.of 遇 null 会 NPE
            if (auth != null) md.put("authorization", auth);
            return McpTransportContext.create(md);
        })
        .build();
}
```

**这个 bean 不能省。** `McpServerStreamableHttpWebMvcAutoConfiguration` 从不设置 `contextExtractor`，而 builder 的默认值是 `serverRequest -> McpTransportContext.EMPTY`——开箱即用时每个工具拿到的都是空 context（spring-ai issue #4603）。该 bean 标了 `@ConditionalOnMissingBean`，自己声明即可覆盖；但**必须把自动配置的 `.jsonMapper(...)` 与 `.mcpEndpoint(...)` 一并抄来**，否则 MCP 消息序列化会坏。`req` 的类型是 `org.springframework.web.servlet.function.ServerRequest`。

**第二段：工具层读取。用一个末尾的 `ToolContext` 参数。**

```java
@Tool(description = "…")
public String reviewWork(@ToolParam(description = "…") String workId, /* …业务参数… */
                         ToolContext ctx) {
    McpSyncServerExchange ex = McpToolUtils.getMcpExchange(ctx).orElseThrow();
    String authorization = (String) ex.transportContext().get("authorization");
    …
}
```

**参数类型只能是 `ToolContext`，绝不能用 `McpSyncRequestContext`。** 已查证 `JsonSchemaGenerator.generateForMethodInput` 只跳过 `ToolContext` 一种类型（源码注释即"not included in the JSON Schema generation"）；`McpSyncRequestContext` 与 `McpTransportContext` **既不会被排除出对客户端可见的 schema，也不会被注入**——它们会作为普通业务参数出现在 `tools/list` 里，并被拿去模型的 JSON 参数里取值。

因此原拟的"兜底方案"不是丑陋，而是**错的**。`ToolContext` 对客户端完全不可见，故 20 个工具的签名仍然只有业务参数 + 这个尾参，与 Python 契约一一对应。

**不用 `RequestContextHolder`。** 无官方文档或测试支持；社区 issue（#2757 报告在 `@Tool` 内取到 null、#5374 问同一件事且未获维护者答复）都指向不可靠。即便在当前"同步 + streamable + WebMVC"组合下大概率能跑，它也会在切到 `ASYNC`、WebFlux 或任何线程切换时静默失效。**上面那条不是首选，是唯一可行路径。**

### 7.6 两个兼容点（已查证）

**① 尾斜杠必须自己补。** `/mcp` 可用，`/mcp/` 返回 **404**（不是 405）。传输层以精确 PathPattern 注册 GET/POST/DELETE，而自 Spring Framework 6.0 起 `matchOptionalTrailingSeparator` 默认为 false，**到 Spring Framework 7.0（Boot 4.1.1 所用）连 `setMatchOptionalTrailingSeparator` 与 `setUseTrailingSlashMatch` 两个方法都已删除**；MCP 与 Spring AI 都没有对应属性。

README 承诺两种写法都能连，故用 Spring Framework 的 `UrlHandlerFilter` 补上：

```java
@Bean
FilterRegistrationBean<UrlHandlerFilter> mcpTrailingSlash() {
    var filter = UrlHandlerFilter.trailingSlashHandler("/mcp")
            .wrapRequest()               // 工具是 POST，不能用 redirect
            .build();
    var reg = new FilterRegistrationBean<>(filter);
    reg.setOrder(Ordered.HIGHEST_PRECEDENCE);
    return reg;                          // 只挂 /mcp，不要 /**
}
```

**② DNS 重绑定防护必须手工接，且默认是关的。** MCP Java SDK 2.0.0（Spring AI 2.0.1 锁定）里有与 Python `TransportSecuritySettings` 对应的实现，但**传输层 builder 的默认值是 `ServerTransportSecurityValidator.NOOP`——即全部放行**，自动配置也不设置它。必须显式声明：

```java
@Bean
ServerTransportSecurityValidator mcpSecurity(McpSecurityProperties props) {
    var b = DefaultServerTransportSecurityValidator.builder();
    props.allowedHosts().forEach(b::allowedHost);   // 精确主机名，或 主机名:*
    return b.build();
}
```

它校验 `Host` 头，不在白名单则 **421**（与 Python 行为一致），`Origin` 非法则 403。

白名单仍来自 `mcp.allowed_hosts`，**并且必须保留原 `_normalize_mcp_hosts` 的那条校验**（`config.py:159-164`）：任何 `*` 只要不是 `:*` 结尾就直接报错。不要因为换了语言就放宽成接受全局 `*`——那会让白名单失去意义。

---

## 8. 附件与通知

### 8.1 附件存储

布局不变：`data/files/<work_id>/v<version>/<file_id>__<安全文件名>`

**四个必须照搬的安全细节：**

**① 路径包含检查用 `is_relative_to`，不用 `startswith`。** `storage.py:6-8` 的注释点名：`startswith` 挡不住 `/data/files_evil` 这类同前缀目录。Java 对应 `Path.normalize().startsWith(root)`——`java.nio.file.Path.startsWith` 按**路径分量**比较，语义正确。

**② 文件名清洗**：剥路径 → 控制字符与 `/\:*?"<>|` 换 `_` → 去尾部空格与点（Windows 下会写失败）→ 按 **UTF-8 字节**截到 200 并保留扩展名 → 全空则 `unnamed`。

**③ 原子写**：在**目标同目录**下 `mkstemp` → 写入 → `flush` → **`fsync`** → `os.replace`。同目录是关键（跨设备 rename 非原子）。Java：`Files.createTempFile(dir, ...)` → 写 → `FileChannel.force(true)` → `Files.move(..., ATOMIC_MOVE)`。

**④ 三种错误三种语义**：`StorageInputError`(422，调用方改参数可重试) / `StorageMissingError`(404，库里有记录磁盘无文件) / `StorageError`(500)。MCP 侧再映射为 `FILE_NOT_FOUND` 与 `ACCESS_OR_FILE_ERROR`（`mcp_server.py:31`）。

**扩展名白名单来自配置，content-type 从不存储也不信任。** 不得因"Spring 提供 `MultipartFile.getContentType()`"而改用它做判断。

### 8.2 通知投递

`deliver_next()` 的自我描述即设计：**"Claim in a short transaction, send without a DB connection, finish by lease ID."**

```
事务一（短）  SELECT ... FOR UPDATE SKIP LOCKED ORDER BY id LIMIT 1
             标记 status=sending, worker_id=<随机>, lease_until=now+max(120, timeout*4+60)
             attempts += 1；取出 url/type/payload 后立即 commit
─────────────────────────────────────────────────────────
无事务        实际发送 HTTP（慢，可能超时）
─────────────────────────────────────────────────────────
事务二（短）  UPDATE ... WHERE id=? AND worker_id=?      ← 按租约身份归还
             成功 → ok；失败 → failed + 下次时间；超过 5 次 → dead
```

**重试梯度 `(10, 30, 120, 600, 1800)` 秒，第 6 次直接 `dead`。**

Java 实现：两个 `TransactionTemplate` 块夹一段不持连接的发送。**不得用 `@Transactional` 包住整个方法**——那会把数据库连接攥在手里发 HTTP，正好破坏该设计。

**`error` 字段只写"异常类名 + 数字状态码"，绝不写异常原文**（`notify.py:194-197`）。理由见注释：异常文本可能含 webhook URL 与凭据。运维看到的 `HTTP 401` 是**故意**这么短的。`DeliveryRejected` 的消息由本模块构造（只含状态码或渠道名），故可安全写入。

**`delivery_logs.event_id` 可空的结论**：不会出问题，**但挡住它的不是那条唯一索引**。

`emit()` 每次现场生成新的 `event_id`（`notify.py:129` 的 `secrets.token_hex(16)`），故 `uq_delivery_event (event_id, sub_id)` **几乎永不触发**——它是一个**哑掉的守卫**。真正在干活的是另外两条：

- **接收方去重**靠 payload 中的 `event_id` **被原样重放**：重试使用 `delivery_logs.payload` 里存的那份，`event_id` 不变，接收方得以识别重复投递。这才是 README 中"至少一次投递，接收方按 event_id 去重"的实现方式。
- **防止同一业务事件被 emit 两遍**的是幂等闸门：重放请求直接返回存下的 response，不会再跑一遍 `emit`。

**处置**：索引保留（无害，且表达了"同一事件不该对同一订阅入队两次"的意图），但规格中记录其**当前不生效**，避免后来者误以为有它在兜底。

**可删除**：`notify.py:171` 的 `payload.setdefault("event_id", f"legacy-delivery-{lid}")` 是为引入 `event_id` 之前写下的旧记录兜底，空库重来后无服务对象。

### 8.3 SSRF 防护（两段式）

webhook 的 url 由调用方提供，服务端主动请求它——这是一个 SSRF 面。

- **注册时**（`check_webhook_url`）：仅校验协议为 http/https、具备主机名。**故意不解析 DNS**——注释写明"避免注册时把请求变成探测"。
- **投递前**（`assert_outbound_allowed`）：解析主机名，逐个 IP 检查。链路本地 / 未指定 / 组播 / 保留地址**一律拒绝**，与配置无关；环回与私网由 `notify.block_private_hosts` 控制（内网自托管可能确需回调私网）。
- **`follow_redirects=False`**（`notify.py:180`）：挡住用 302 跳入内网的打法。Java 的 `HttpClient` **默认跟随重定向，必须显式关闭**，否则等于拆掉这道防线。

### 8.4 其他必须保留的行为

- `mask_url`：列表接口返回 webhook 时，对 query 中 `access_token` / `token` / `key` / `secret` / `sign` / `password` 等键脱敏。
- `parse_events`：订阅的事件过滤为损坏 JSON 时返回 None，跳过该订阅并打 warning，**不中断投递循环**。
- 钉钉渠道：`{"msgtype":"text","text":{"content":...}}`，且**要求 `errcode == 0`**，非 0 视为确定性拒绝。

---

## 9. 测试策略

### 9.1 隔离方式原样保留

`test_sequential.py` 开头的护栏必须保留，一字不改：

```python
if url.host not in ("127.0.0.1","localhost") or not url.database.startswith("workhub_test"):
    raise RuntimeError("Tests require a loopback database whose name begins workhub_test")
```

这是防止误连生产库的安全护栏。

每个测试方法 `CREATE SCHEMA accept_<随机>` + `search_path` 指向该 schema，测毕删除。**不用"事务内跑完回滚"**：`test_concurrent_conflicting_reviews`、`test_withdraw_review_race`、`test_concurrent_same_request` 测的是**真实并发提交**，回滚隔离会使其失效。schema 隔离才能让事务真正提交。

Java 侧：Testcontainers 起 `postgres:17-alpine`，每测试建 schema，DataSource 的 JDBC URL 携带 `?currentSchema=accept_xxx`。

### 9.2 场景取舍：35 个中端口 31

判据不是测试名，而是**它 import 谁**。文件里只有两种外部依赖：

- `skills/workhub-mcp-connect/scripts/mcp_connect.py` —— 接入程序，属 skill 侧
- `tests/fixtures/legacy_models.py` —— 迁移前的老表结构，已无对应数据

| 处置 | 场景 | 依据 |
|---|---|---|
| 砍 | `test_legacy_null_employee_id_can_complete_profile`（500 行） | import `fixtures/legacy_models.py`，测老档案空工号补填 |
| 砍 | `test_migrate_legacy_preserves_data`（628 行） | 同上，测老数据迁移 |
| **移 skill 阶段** | `test_bootstrap_recovers_lost_registration_response`（450 行） | import `mcp_connect.py`，测的是**接入程序**的"响应丢失后重跑"与待处理文件 `0o600`。它依赖的服务端性质（同工号重复登记同 token）已被 `test_registration_replay_and_duplicate`（118 行）与 `test_parallel_registration_does_not_reissue`（158 行）单独覆盖，故服务端侧无损失 |
| **移 skill 阶段** | `test_claim_reuses_legacy_server_name`（476 行） | import `mcp_connect.py`，断言的是客户端 `mcp.json` 里 `mcpServers` 的**键名**（应沿用 `workhub-staff` 而非新建 `workhub`）。服务端根本没有"server 名"这个概念——那是客户端配置的键。此测试在 Java 服务端套件里**无法表达** |
| **保留** | `test_legacy_claim_entry_token_is_rejected`（150 行） | 名字含 legacy 但**不** import 任何 fixture，测的是 `auth.py:116-120` 那段活的行为（配置残留废弃条目须明确报错），是纯服务端场景 |

由此文件可切成三段：服务端场景 98–448 行、接入/引导 450–498 行、legacy 与部署 500–668 行。**服务端端口 31 个场景**（35 − 2 砍 − 2 移）。

移走的那 2 个不丢：它们属第 13.1 节的 skill 阶段，届时连同新写的引导流程一起重做。

### 9.3 不可省略的场景

这几条是整套机制存在的理由，必须端口：

| 场景 | 守护的性质 |
|---|---|
| `test_list_query_count_is_flat` | N+1 守卫：查 50 条必须少于 20 条 SQL |
| `test_outbox_write_failure_rolls_back_business` | 通知写失败必须回滚业务 |
| `test_outbox_atomic_and_nonblocking` | MCP 不等待外部 HTTP |
| `test_real_mcp_http_and_process_restart` | 真 MCP HTTP + 进程重启 |
| `test_database_and_file_backup_restore` | 数据库与附件必须成套备份 |
| `test_duplicate_success_precedes_version_check` | 幂等优先于版本检查，顺序反了即 bug |
| `test_notification_retry_and_expired_lease` | 租约过期后由他人接管 |
| `test_concurrent_conflicting_reviews` | 真实并发审批 |
| `test_identity_inactive_cannot_replay_writes` | 停用身份不得重放写入 |

---

## 10. 部署与运维

### 10.1 `docker-compose.yml` 几乎不动

文件内那些看似啰嗦的注释均由实际故障换来，**一条都不改**：

- `POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"`——缺 locale 时会落到 `SQL_ASCII`，中文标题与点评以字节存入，检索排序全部失真；`locale=C` 避免 glibc/musl 版本差异导致索引排序不一致。
- 数据库端口只绑 `127.0.0.1`。
- 命名卷而非 bind mount（macOS 上 PG 数据目录 bind mount 有属主与 fsync 语义问题）。
- healthcheck 指向 `127.0.0.1` 而非 unix socket。
- 日志 `max-size: 10m` 轮转（不轮转会写满宿主磁盘，而磁盘满时 PG 直接写不进去）。
- `stop_grace_period: 30s`。原注释写的理由是"上传上限 200MB，默认 10s 可能截断在途请求"——**200MB 是代码里的兜底默认值（`DEFAULT_MAX_SIZE_MB`），而实际投放的 `config.yaml` 把它收紧为 50MB**。两个数都对，只是分属两层：宽限期要按"最坏情况"给，所以引用默认值。**端口时两层都要保留**：常量仍是 200，`config.yaml` 仍写 50（见 10.5）。
- `user: "${WORKHUB_UID:-10001}:${WORKHUB_GID:-10001}"`。

**唯一变更是 `workhub` 服务的 `build`** 段。

### 10.2 `Dockerfile` 重写，保留三条实质约束

1. **非 root，uid 10001**——与 `./data` 属主对齐。
2. **安装 `tzdata` 并设 `TZ=Asia/Shanghai`**。此条最易被当作装饰删除。原注释说明：不装 tzdata，`TZ` 会被**静默忽略**、时间退化为 UTC。本系统的 webhook payload 中 `timestamp` 使用 `datetime.now()`（**本地时间、不带时区**），而数据库存的是 UTC——时区错了，通知中的时间与审计中的时间会**各错各的**。
3. **HEALTHCHECK 走 `/api/ready`**，数据库不可用返回 503 即判不健康。Java 版不使用 curl（基础镜像未必有），改为 `main` 在启动 Spring 之前判断参数：`java -jar app.jar --healthcheck` 直接探测并返回退出码。理由同原 `healthcheck.py`——**不用一行式命令**，否则数据库维护期间每 30 秒往容器日志打一段堆栈。

构建阶段的 Maven 依赖分层缓存对应原 uv 的两层同步：先拷 `pom.xml` 装依赖，再拷源码编译。

### 10.3 `deploy.sh` 只有一行要改

`deploy.sh:644` 有一行 `info`，用途是**打印**给运维去远端执行的迁移命令。Flyway 启动时自动迁移，该行整个删除。

其余 11 个子命令（`check` / `deploy` / `update` / `load` / `verify` / `status` / `logs` / `doctor` / `backup` / `start` / `down`）与硬性规则（不碰非 workhub 资源、不覆盖 `.env`、不以镜像名判构建结果）**全部不变**。

### 10.4 运维命令的参数面不变

```bash
# 现状
uv run python -m app.maintenance inspect-files --verify-hashes
uv run python -m app.maintenance deactivate E101 --reason '离职'

# 之后
java -jar app.jar maintenance inspect-files --verify-hashes
java -jar app.jar maintenance deactivate E101 --reason '离职'
```

参数完全一致，运维文档与肌肉记忆无需改变。`inspect-files --verify-hashes` 只报告不删除；`deactivate` 明确停用并保留档案与历史。

### 10.5 配置文件

**保留 `config.yaml` 及其键名不变**，Java 侧绑定到类型化的 `@ConfigurationProperties`。

理由——它是部署契约而非普通配置文件：

- compose 只读挂载（`./config.yaml:/app/config.yaml:ro`）
- `deploy.sh` 的 `setkv` 会写它
- 运维文档与 README 指向它（"`registration.enabled` 是唯一开关"）
- **文件内的注释本身是文档**——例如解释"MCP 白名单没配好会返回 421，但 REST 不受影响，很容易误判成服务没问题"

**五条环境变量名不变**（不改成 `A2AHUB_*`——它们是部署契约，compose 与运维文档都按这些名字设值）：`WORKHUB_TOKENS`、`WORKHUB_DATABASE_URL`、`WORKHUB_STORAGE_DIR`、`WORKHUB_BLOCK_PRIVATE_HOSTS`、`WORKHUB_MCP_ALLOWED_HOSTS`。

`upload.max_size_mb` 按现状保留 **50**，同时 Java 侧仍须保留 `200` 这个兜底默认值（见 10.1）。`features.archive_to_weknora` 与 `WORKHUB_LEGACY_TIMEZONE` 删除（见 6.1.1）。

**`database_url` 必须翻译，不能直读。** 投放的 `config.yaml` 与 compose 注入的都是 SQLAlchemy 写法（`postgresql+psycopg://workhub:workhub@127.0.0.1:5432/workhub`），而 Java 数据源要 `jdbc:postgresql://…`。原 `config.py` 本就有 `_DIALECT_ALIASES` 在做归一化，接受 `postgres://`（Heroku 风格）、`postgresql://`、`postgresql+psycopg://` 三种。既然配置契约不变，**Java 侧就接下这段翻译**：入参接受这三种写法，转成 JDBC 形式再交给数据源。

不要把 `config.yaml` 里的值直接改成 `jdbc:` 形式了事——那会破坏"同一份配置在本地与内网共用"的现状，也和环境变量覆盖路径冲突（compose 注入的同样是 SQLAlchemy 写法）。

**`server.port` 是僵尸键，不要实现。** 原 `load_config` 显式做了 `server.pop("port", None)`，注释写明它"看着像能用、改了不生效"——绑定地址由启动命令决定，该键只曾喂给一条提示性告警。Java 侧同样不得读它，否则会凭空造出一个新的僵尸配置项。

**布尔环境变量的取值集合要照搬。** `WORKHUB_BLOCK_PRIVATE_HOSTS` 认 `1/true/yes/on` 与 `0/false/no/off`（`config.py` 的 `_BOOL_TRUE` / `_BOOL_FALSE`），不是只认 `true`/`false`。非法值的行为也要一致。

**配置加载失败要快速失败并指出文件与键。** 原 `config.py` 把 `load_config()` 包在 try/except 里（`config.py:243-246`），失败时给的是"哪个文件、哪个键"，而不是一段无上下文的启动堆栈。Java 侧对应为启动即失败 + 明确报出键名。

待实现细节：Spring 的配置绑定默认偏好 `workhub:` 前缀，而本文件使用顶层键。读取机制（`spring.config.additional-location` 或专用加载器）在实现早期确定，**约束是文件形状与上述五个环境变量名不得改变**。

### 10.6 时间戳的不一致（记录在案）

同一份数据里存在两种时间：

- **数据库**：`utcnow()` → 带时区的 UTC
- **webhook payload 的 `timestamp`**：`datetime.now().isoformat()` → **本地时间，不带时区**

接收方拿到该时间戳无法判断时区。看似疏漏，但它是现状，且是给人看的字段。

**决定：按现状端口，不改。** 两条理由：

1. 改它等于改 webhook 的对外契约，而 webhook 可能指向第三方（钉钉或自建接收端）。这属于"顺手改进"，与本项目"与 workhub 一致优先"的取舍原则相冲突。
2. 实际风险有限：部署恒定在单一时区（第 10.2 节的 `TZ=Asia/Shanghai`），故该时间戳恒为东八区本地时间，歧义被部署固定住了。

仅在此记录，以免后来者误以为它是 UTC。**注意这与第 10.2 节的 tzdata 是同一条链**：若哪天有人删掉 Dockerfile 里的 tzdata，这个时间戳会静默退化为 UTC，而数据库里的时间不变——两边对不上，且没有任何报错。

---

## 11. 偏离清单

本次端口**全部**偏离原实现的地方，共 7 处，均已获确认：

| # | 偏离 | 性质 | 风险 |
|---|---|---|---|
| 1 | 状态机由 7 处散落的 `_state()` 守卫收拢为显式转移表 | 可读性重构，行为不变（转移表逐条核实自原守卫） | 低 |
| 2 | 审批链由抛异常改为 `ChainPlan` 返回值 | 控制流显式化，两条调用路径的语义不变 | 低 |
| 3 | 删除 6 列、1 表、2 处配置（第 6.1、6.1.1 节） | 均为已死代码，逐列逐键追溯过读取方 | 删列不可逆，已在评审中确认 |
| 4 | 补 9 条 CHECK 约束（第 6.4 节） | 把注释中的取值搬进数据库 | 代码写越界值将由静默转为报错（故意） |
| 5 | REST 错误信封统一为 `{code, message}` | 客户端可见 | 唯一消费方 `mcp_connect.py` 待重写 |
| 6 | `delivery_logs.status` 由 3 值更正为 5 值 | **更正既有错误**，非设计改动 | 无（原注释是错的） |
| 7 | `config.yaml` 保留不变（第 10.5 节） | 保守选择 | 读取机制需绕开 Spring 前缀偏好 |

**其余全部原样端口。**

明确**不做**的"顺手改进"：

- 不修 `delivery_logs.event_id` 可空 + 唯一索引的组合（第 8.2 节有结论）
- 不修 `work_submissions.status` 与 `work_items.status` 的镜像关系（第 5.5 节）
- 不修 `approved` 状态无出边（第 5.1 节）
- 不修 webhook payload 的本地无时区时间戳（第 10.6 节）
- 不给 `work_items.current_submission_id` 加外键（现状无 FK）

---

## 12. 风险与待验证点

### 12.1 已定案（查 Spring AI 2.0.1 源码得出，非文档推测）

| # | 项 | 结论 |
|---|---|---|
| 1 | MCP 工具如何取得 Bearer | `contextExtractor` 注入 + 工具尾部 `ToolContext` 参数读回（7.5） |
| 3 | 附加参数是否泄漏进客户端可见 schema | **会泄漏。** 只有 `ToolContext` 被 `JsonSchemaGenerator` 跳过；`McpSyncRequestContext` / `McpTransportContext` 会进 `tools/list` 并被当业务参数取值——故**不能用**（7.5） |
| 4 | DNS 重绑定防护 | `DefaultServerTransportSecurityValidator`；**builder 默认是 NOOP（全放行）**，必须手工接（7.6） |
| 5 | `/mcp/` 尾斜杠 | Spring Framework 7.0 已彻底删除尾斜杠匹配，用 `UrlHandlerFilter.wrapRequest()` 补（7.6） |

### 12.2 未决（实现早期验证）

| # | 风险 | 验证方式 | 退路 |
|---|---|---|---|
| 2 | 测试的 schema-per-test 与 Spring 测试上下文缓存冲突（9.1） | 写第一个集成测试时验证 | 改为顺序执行 + 测试间 `TRUNCATE` |
| 6 | `config.yaml` 顶层键绑定（10.5） | 写配置绑定类时验证 | 专用加载器（如原 `config.py` 的做法） |

### 12.3 本轮查证带来的一处新脆弱点

自己声明 transport provider 会**覆盖**自动配置（`@ConditionalOnMissingBean`），所以 `.jsonMapper(...)` 与 `.mcpEndpoint(...)` 必须手动抄全。抄漏任何一处，MCP 消息序列化或端点路径就会坏，而症状不会指向根因——这正是 spring-ai issue #4603 那类问题的形态。

**对策**：把 transport provider 的装配放进第 0 阶段，用一个真发 HTTP 的 `initialize` 测试盖住它（对齐原 `test_real_mcp_http_and_process_restart` 的做法），不要等到第 5 阶段接工具时才发现。

---

## 13. 分阶段实施

端口面积大（`app/` 2,985 行 Python，去掉不端口的 legacy 后约 2,700 行），但模块间有强依赖顺序。**本规格对应一个实施计划，计划内部按下列阶段推进，每阶段可独立验证。**

| 阶段 | 内容 | 完成标志 |
|---|---|---|
| 0 | 骨架：pom 依赖、Flyway `V1__baseline.sql`、配置绑定、错误体系、`/api/health` 与 `/api/ready`；**MCP transport provider 装配**（7.5/7.6 的 bean 与安全校验），并用真发 HTTP 的 `initialize` 测试盖住它 | 空库启动建表成功，健康检查通过；`/mcp` 与 `/mcp/` 都能完成 initialize，Host 头不在白名单时返回 421 |
| 1 | 身份：`identity` + 凭据 + 登记 + 鉴权 | 登记幂等、并发登记、角色与上级校验的测试通过 |
| 2 | 工作项与审批：`work` 核心、状态机、审批链、幂等闸门 | 三级流转、并发审批、撤回返工测试通过 |
| 3 | 附件：存储、上传、读取、下载 | 原子写、扩展名白名单、路径逃逸防护测试通过 |
| 4 | 通知：outbox、租约、重试、SSRF 防护、两个渠道 | 原子性、租约过期接管、重试梯度测试通过 |
| 5 | 两个入口：REST 控制器 + 20 个 MCP 工具类（transport 已在阶段 0 装好） | 29 条路由与 20 个工具行为一致 |
| 6 | 部署收口：Dockerfile、compose、`deploy.sh`、运维命令 | 备份恢复演练、进程重启测试通过 |
| 7 | **（独立规格）** skill 重写 | 见下 |

### 13.1 阶段 7 的边界

skill 重写**不在本规格范围内**，将另起一份规格。本规格只约束一件事：

**服务端的登记接口必须原样提供**（REST，无 MCP 工具），使新接入程序可以替换旧程序而不改动服务端契约。

接入程序（`mcp_connect.py`，927 行，纯标准库 Python）的最终形态——是重写为 Java、还是保留 Python、还是改为 shell——属阶段 7 的决策，本规格不预设。

已知的输入（供阶段 7 使用）：

- 用户实测原版 skill 时，**初次引导卡了很久都未能真正开始**。根因分析需要用户回忆当时卡在哪一步，届时单独采集。
- 第 9.2 节移出的 2 个场景——`test_bootstrap_recovers_lost_registration_response`（接入程序响应丢失后重跑，及待处理文件 `0o600`）与 `test_claim_reuses_legacy_server_name`（客户端 `mcp.json` 键名沿用）——归属这一阶段。它们全程不需要服务端改动。

---

## 14. 附：本规格的依据

- 行为来源：`/Users/echoyan/Developer/workhub`（Python，已退役）
- 全部关键结论均通过阅读源码逐条核实，涉及的原始位置已在正文中标注行号
- 用户已删除的旧文档（`docs/superpowers/specs/2026-09-24-a2ahub-mvp-design.md`、`docs/superpowers/plans/2026-09-24-phase1-identity-credentials.md`）描述的是**另一套领域模型**（工作归档 + 评审），与 workhub 的"自助身份 + 逐级审批"不构成同一系统的两个设计版本。本规格不引用、不恢复。
