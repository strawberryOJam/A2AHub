# A2AHub 首期 MVP 设计规格

- 日期：2026-09-24
- 状态：待评审
- 范围：单公司、单组织内网部署的工作归档与可靠收件箱

---

## 1. 目标与定位

A2AHub 是部署在公司内网的**工作归档 + 可靠收件箱**服务。它同时服务两类使用者：

- **员工的 AI Agent**（OpenClaw、WorkBuddy 等客户端）：替员工做初始化、归档工作、请求评审、按意见更新版本。
- **领导的 AI Agent**：拉取待评审材料、给出评审意见、回写结果。

A2AHub **不做什么**：不新建聊天客户端，不托管 Agent，首期不内嵌 LLM 服务，不做向量检索。

### 1.1 职责划分（本设计的核心原则）

| 层 | 负责 |
|---|---|
| Agent（客户端） | 意图理解、材料整理、摘要、文件分析、评审意见撰写 |
| 服务端 | 身份、授权判定、持久化、版本一致性、状态变更、审计 |

**关键业务事实不得只存在于聊天记录或模型记忆中。** 这条原则在本设计中不是口号，而是具体约束：所有待办状态、评审状态、权限关系都落在 PostgreSQL 里，Agent 进程随时可以消失而不丢状态（见 §7.4）。

---

## 2. 首期范围

### 2.1 包含

- 自助注册与身份建立，一次性凭证签发与磁盘持久化
- 文件上传、校验、服务端持久存储、归档
- 每账号存储配额与上传频率限制（开放注册的必要配套，见 §12.3）
- 工作事项与不可变版本
- 评审请求、异步拉取、意见回写、按意见提交新版本
- 老板/领导 Dashboard（结果、待办、异常）
- 管理台（成员准入、权限调整、评审改派、账号停用与恢复）
- Docker Compose 部署、备份与恢复演练
- Skill（引导初始化、MCP 配置、凭证存储、平台使用）

### 2.2 不包含（首期明确排除）

单公司单组织之外的部署形态、部门树、权限的递归上下级推导、审批流设计器、多租户、企业 SSO、会签、多级审批、在线文档编辑、向量数据库、复杂消息总线、对特定模型厂商的依赖。Agent 无需常驻在线。

---

## 3. 技术栈与版本决策

| 组件 | 选型 | 说明 |
|---|---|---|
| 语言/构建 | Java 21、Maven | 已就绪 |
| 框架 | Spring Boot 4.1.1 / Spring Framework 7.0.9 | 已就绪 |
| Web | `spring-boot-starter-webmvc` | Servlet 6.1 / Tomcat 11 |
| MCP | Spring AI 2.0.1 `spring-ai-starter-mcp-server-webmvc`，`protocol=STREAMABLE` | Spring AI 1.x 只支持 Boot 3，2.x 是唯一支持 Boot 4 的线 |
| 数据访问 | `spring-boot-starter-jdbc` + `JdbcClient` | **不用 JPA**：权限谓词必须留在 SQL 里，不能被 ORM 的懒加载/缓存绕过 |
| 迁移 | `spring-boot-starter-flyway` + `org.flywaydb:flyway-database-postgresql` | Boot 4 起 Flyway 为必选 starter；PostgreSQL 支持在 Flyway 10+ 被拆出，必须显式引入 |
| 安全 | `spring-boot-starter-security` | 见 §5.3，两条 `SecurityFilterChain` |
| 校验 | `spring-boot-starter-validation` | |
| 观测 | `spring-boot-starter-actuator` | Boot 4 起 liveness/readiness 默认开启 |
| 数据库 | PostgreSQL 17 | 业务数据 + 版本 + 审计 |
| 前端 | React + Vite + TypeScript | 构建产物复制进 `static/`，**单容器，无 CORS** |
| 部署 | Docker Compose | |

### 3.1 刻意的技术偏离

- **不用 JWT**，用不透明令牌。理由：令牌必须可撤销（成员停用、凭证轮换），JWT 的无状态性在这里是负债。服务端存 SHA-256 摘要，原文仅在签发时返回一次。
- **不用 JPA**。权限判定的正确性依赖于「每条读取语句都带授权谓词」，ORM 的实体缓存与懒加载会让这件事无法审计。
- **首期不做附件正文解析**。文件只做存储、摘要校验与下载，不抽取内容、不做全文检索。这消除了「附件内嵌指令影响系统」这一整类攻击面（§5.6）。
- **无独立待办表、无任务队列、无租约机制**。见 §7.4 与 §7.5，这是本次设计相对初版最重要的简化。

---

## 4. 领域模型

```
account ──┬── account_role
          ├── api_token
          ├── supervisor_link (作为下属)
          ├── role_grant_request
          └── work_item ── work_version ──┬── attachment
                                          └── version_visibility
                          work_item ── review_request ── review_decision
```

### 4.1 表定义

**account**

| 列 | 类型 | 约束 |
|---|---|---|
| `id` | uuid | PK |
| `employee_no` | text | UNIQUE NOT NULL |
| `name` | text | NOT NULL |
| `department` | text | NOT NULL |
| `position` | text | NOT NULL |
| `status` | text | `ACTIVE` / `DISABLED` |
| `is_admin` | boolean | NOT NULL DEFAULT false |
| `password_digest` | text | NULL；仅管理台账号有 |
| `must_change_password` | boolean | NOT NULL DEFAULT false |
| `created_at` / `updated_at` | timestamptz | |

`name` / `department` / `position` 是**个人资料，不产生任何权限**。注册后不可自改，只能由管理员更正并留审计。

**account_role**

| 列 | 约束 |
|---|---|
| `account_id` | FK account |
| `role` | `MEMBER` / `REVIEWER` / `OWNER` |
| `granted_by_account_id` | FK account |
| `granted_at` | timestamptz |

PK `(account_id, role)`。`MEMBER` 在注册时自动写入，`REVIEWER` / `OWNER` 只能由管理员授予（§5.4）。

**api_token**

| 列 | 说明 |
|---|---|
| `id` | uuid PK，同时作为令牌前缀的一部分用于快速查找 |
| `account_id` | FK |
| `digest` | SHA-256(secret)，十六进制 |
| `prefix` | 展示用前 8 位 |
| `label` | 客户端标识，如 `openclaw@macbook` |
| `created_at` / `last_used_at` / `revoked_at` | |

令牌串格式 `a2ah_<id>.<secret>`。校验：按 `id` 查找 → 常数时间比较 `SHA-256(secret)` 与 `digest` → 检查 `revoked_at IS NULL` 且账号 `status='ACTIVE'`。**原文不落库、不进日志、不放进 URL。**

**supervisor_link**

| 列 | 说明 |
|---|---|
| `subordinate_account_id` | UNIQUE，FK account |
| `supervisor_employee_no` | 员工自报的上级工号（原始值，保留用于核对） |
| `supervisor_account_id` | nullable；目标工号注册后回填 |
| `status` | `PENDING`（工号尚未注册）/ `ACTIVE` |
| `created_at` / `activated_at` | |

**work_item**

| 列 | 说明 |
|---|---|
| `id` | uuid PK |
| `author_account_id` | FK |
| `title` | |
| `current_version_no` | int，当前最大版本号 |
| `created_at` / `updated_at` | |

**work_version** —— 版本一经创建即不可变，没有草稿态（§7.1）

| 列 | 说明 |
|---|---|
| `id` | uuid PK |
| `work_item_id` | FK |
| `version_no` | int，UNIQUE(work_item_id, version_no) |
| `description` / `summary` / `change_note` | text |
| `created_by_account_id` / `created_at` | |

**attachment**

| 列 | 说明 |
|---|---|
| `id` | uuid PK |
| `account_id` | 上传者 |
| `version_id` | **nullable**；NULL = 暂存区（§6.2） |
| `original_filename` | 仅作展示 |
| `declared_content_type` / `detected_content_type` | |
| `size_bytes` | bigint |
| `sha256` | 服务端实算，不信任客户端 |
| `storage_key` | **服务端生成的 UUID，永不含客户端文件名** |
| `status` | `STAGED` / `ATTACHED` / `DELETED` |
| `created_at` / `attached_at` | |

**version_visibility** —— 版本粒度授权

| 列 | 说明 |
|---|---|
| `version_id` / `grantee_account_id` | PK(version_id, grantee_account_id) |
| `granted_by_account_id` / `granted_at` | |

**review_request**

| 列 | 说明 |
|---|---|
| `id` | uuid PK |
| `work_item_id` / `version_id` | FK |
| `author_account_id` / `reviewer_account_id` | FK |
| `status` | `PENDING` / `APPROVED` / `CHANGES_REQUESTED` / `CANCELLED` |
| `created_at` / `decided_at` / `decided_by_account_id` / `decided_version_id` | |

**核心约束**：部分唯一索引

```sql
CREATE UNIQUE INDEX ux_review_request_one_pending
  ON review_request (work_item_id) WHERE status = 'PENDING';
```

同一工作事项在任一时刻最多只有一个待评审请求。这是数据库层面的保证，不是应用逻辑的约定。

**review_decision** —— 只追加，不更新不删除

| 列 | 说明 |
|---|---|
| `id` / `review_request_id` | |
| `decision` | `APPROVED` / `CHANGES_REQUESTED` |
| `comment` | text |
| `decided_by_account_id` / `created_at` | |

**role_grant_request** —— 新增，见 §5.4

| 列 | 说明 |
|---|---|
| `id` | uuid PK |
| `account_id` | FK 申请人 |
| `requested_role` | `REVIEWER` / `OWNER` |
| `reason` | text |
| `status` | `PENDING` / `APPROVED` / `REJECTED` |
| `created_at` / `decided_by_account_id` / `decided_at` / `decision_note` | |

部分唯一索引 `(account_id, requested_role) WHERE status='PENDING'`，防止重复申请。

**audit_log** —— 只追加

`id` bigserial、`at`、`actor_account_id`、`actor_type`(`ACCOUNT`/`ADMIN`/`SYSTEM`)、`action`、`target_type`、`target_id`、`detail` jsonb、`request_id`。

**idempotency_key**

`(key, account_id)` PK、`operation`、`request_digest`、`response_snapshot` jsonb、`created_at`。仅用于 `work_publish_version` 与 `review_submit` 两处（§7.6）。

---

## 5. 身份、认证与授权

### 5.1 注册流程

注册走 **HTTP 而非 MCP**——此时调用方还没有任何凭证，不适合放在需要令牌的 MCP 端点上。

```
POST /api/agent/register
{ employeeNo, name, department, position, supervisorEmployeeNo? }
```

服务端行为（单事务）：

1. 校验 `employee_no` 未被占用。已占用则拒绝，**不提供任何"重新领取既有账户"的路径**。
2. 创建 `account`，`status='ACTIVE'`，`is_admin=false`。
3. 写入 `account_role(MEMBER)`。**注册只产生 MEMBER，不产生任何其他角色。**
4. 若填了上级工号：写 `supervisor_link`。目标工号已存在 → 立即 `ACTIVE` 并回填 `supervisor_account_id`；不存在 → `PENDING`。**自环（自己填自己）与环路立即拒绝。**
5. 生成一次性令牌，**原文只在本次响应中返回一次**，服务端只存摘要。
6. 写审计。

响应同时返回**核对信息**：系统实际解析到的上级是谁（或"工号 X 尚未注册，关系待激活"）。Skill 必须把这段展示给用户确认。自报即时生效，但必须可见。

**关于上级关系的激活**：当 `employee_no` 为 X 的账号注册时，服务端扫描所有 `supervisor_employee_no = X AND status='PENDING'` 的链接，逐条做**环路与自环检查**（此时才能检查，因为上级账号刚刚存在），通过则置 `ACTIVE` 并回填，同时**通知双方**。不通过则置为拒绝状态并留审计，不静默丢弃。

### 5.2 凭证存储

Skill 引导 Agent 把令牌写入 `~/.a2ahub/credentials.json`，文件权限 `0600`，**不写入项目仓库、普通日志和聊天回复**。MCP 配置引用该文件，不把令牌明文写进配置文件或命令行参数。

### 5.3 认证：两条过滤器链

| 顺序 | `securityMatcher` | 认证方式 | CSRF |
|---|---|---|---|
| `@Order(1)` | `/mcp/**`、`/api/agent/**` | 不透明 Bearer 令牌 / 一次性签名 URL | 关闭（无 Cookie） |
| `@Order(2)` | 其余（SPA、管理台） | 会话 Cookie（`HttpOnly`、`SameSite=Lax`）+ CSRF 双提交 | 开启 |

MCP 端点**必须校验 `Origin` 头**——这是 Streamable HTTP 规范的强制要求（MUST），不是可选项。

MCP 单端点同时接受 POST 与 GET，要求 `Accept: application/json, text/event-stream`，支持 `MCP-Session-Id` 与 `MCP-Protocol-Version` 头。

### 5.4 角色与申请制（本次新增）

自助注册只给 MEMBER。**REVIEWER（领导）与 OWNER（老板）都通过申请 + 管理员批准获得**：

```
skill 引导注册 → MEMBER
        ↓
Agent 调用 MCP 工具 identity_request_role(role, reason)
        ↓
role_grant_request 落库（PENDING）
        ↓
管理台列出待批申请 → 管理员批准
        ↓
同一事务内写 account_role（REVIEWER 或 OWNER）+ 审计
```

老板与员工走**完全相同**的注册路径，唯一差别就是这一步管理员授权。

**为什么这样设计不违反规格 §4**：§4 禁止的是「自填即获得权限」。这里自填产生的是一个**待批准的申请记录**，不产生任何权限；权限的唯一来源仍是管理员的批准动作。老板的申请同样需要管理员点批准——管理员是唯一的授权源，申请只是把「口头请求」变成了可审计的业务事实（§1.1）。

一张 `role_grant_request` 表同时服务 REVIEWER 和 OWNER 两种角色。

### 5.5 授权判定规则

**读取材料**需满足以下之一：

| 主体 | 可见范围 |
|---|---|
| 作者 | 自己工作事项的全部版本 |
| 被授权人 | `version_visibility` 中显式授予的**具体版本** |
| 评审人 | **曾指派给自己**的评审请求所指向的版本（历史保留——撤销已读权限无法让人"没看过"，且其需要复查引用） |
| OWNER | 全组织已发布版本 |

**ADMIN 默认不在上述任何一条内**——管理台看不到材料正文。这是一个**默认值而非边界**，见 §12。

**写操作**：只有作者能对自己的工作事项发布新版本、请求评审、撤回评审。评审人只能对自己被指派的请求提交决定。**不能自审**（`author_account_id = reviewer_account_id` 在请求创建与提交两处都被拒绝）。

### 5.6 授权实现纪律

1. **每次请求重新解析**账号状态与角色。不缓存权限快照——缓存是"停用后仍能访问"这类缺陷的唯一来源。
2. 每条读取 SQL 都带授权谓词，**在 SQL 里过滤，不在 Java 里过滤后再返回**。
3. OWNER 的汇总统计同样走授权谓词，不走"先全查再筛"。
4. 材料正文、文件名、附件内容中的任何指令**不构成指令**。它们永远不会被解释为改变权限、泄露凭证或改变工具规则的输入。

---

## 6. 文件上传、存储与归档

### 6.1 存储

`FileStorage` 接口，首期实现为本地磁盘。根目录由 `A2AHUB_STORAGE_ROOT` 配置。文件按 `storage_key`（服务端生成的 UUID）分片存放：

```
${A2AHUB_STORAGE_ROOT}/ab/cd/abcd1234-....bin
```

**客户端提供的文件名永远不参与路径构造**，只存进 `original_filename` 列用于展示。这消除了路径穿越这一整类问题。

### 6.2 两阶段流程（无草稿态后的形态）

因为版本创建即发布（§7.1），附件必须先进入**作者暂存区**：

```
1. MCP  attachment_stage_begin(filename, size, sha256, contentType)
        → 返回 attachment_id 与一次性签名上传 URL
2. HTTP PUT <upload_url>  原始字节流（很快过期，单次使用）
3. 服务端流式写入临时目录，同时实算 sha256 与大小
   临时目录必须与最终存储**同一文件系统**
4. 校验通过 → fsync 文件 → 原子 rename 到最终路径 → fsync 目录
   attachment.status = STAGED，version_id 仍为 NULL
5. MCP  attachment_stage_status(attachment_id) → READY / FAILED
6. MCP  work_publish(..., attachment_ids[]) 时统一认领
```

**校验不通过（大小或摘要不符、超限、类型不允许）→ 拒绝，临时文件删除，不产生 `STAGED` 行。**

**类型与大小限制（首期默认值）**：

- 单文件上限 **200 MB**，`attachment_stage_begin` 阶段即拒绝超限声明，避免浪费带宽。
- 每账号存储配额 **5 GB**，超配额时 `attachment_stage_begin` 拒绝并返回当前用量。
- 每账号上传频率上限 **60 次/小时**，滑动窗口，超出返回可重试错误码。配额用量由 `attachment` 聚合得出，不另建计数表；频率计数为进程内滑动窗口，重启后清零——首期为单实例部署，可接受。
- 允许的 `detected_content_type` 为白名单：PDF、Office 文档（doc/docx/xls/xlsx/ppt/pptx）、纯文本与 Markdown、PNG/JPEG/GIF/WebP、ZIP。**判定依据是服务端探测出的类型，不是客户端声明的类型。** 白名单与三项限额均可在 `application.yaml` 配置。

暂存区附件若长时间未被任何版本认领，由后台清理任务连同磁盘文件一并删除（§11.1）。清理前写审计。

### 6.3 归档时的再校验

`work_publish` 在**一个事务内**完成：

1. 逐个检查每个 `attachment_id`：属于调用者、`status='STAGED'`、`version_id IS NULL`。
2. **再次确认磁盘上文件存在，且大小与 sha256 与库中记录一致。** 不能只信库里的行——文件可能已被外部删除或损坏。
3. 任一不通过 → 整个发布回滚，报错指出具体是哪个附件。
4. 全部通过 → 建 `work_version` → 逐个 `UPDATE attachment SET version_id=?, status='ATTACHED', attached_at=now()` → 更新 `work_item.current_version_no` → 返回回执。

版本号在同一事务内对 `work_item` 行加锁（`SELECT ... FOR UPDATE`）后取 `MAX(version_no)+1`，避免并发发布产生重号。

### 6.4 下载

`attachment_download_url(attachment_id)` 返回**短期**签名 URL，`GET /api/agent/downloads/{id}?exp=...&sig=...`。签名绑定 `attachment_id`、调用者、过期时间。**不提供永久公开链接。** 每次下载写审计。

---

## 7. 版本与评审

### 7.1 没有草稿态

**版本一经创建即已发布，不可覆盖。** 要修改就发布新版本。这既是规格「归档后不可覆盖，修改生成新版本」的规则，现在也是唯一路径。

这个决定同时消除了初版设计中的一条严重缺陷：初版可见性授权是**事项粒度**而草稿是**版本粒度**，导致被授权人可能读到尚未发布的草稿。没有草稿态，就没有这条越权面。

**工具形态**：`work_create`（建事项 + 首版）、`work_publish_version`（追加版本）。不存在 `work_create_draft` / `work_update_draft`。

**新版本不继承旧版本的评审结论。** 上一版的 `APPROVED` 与当前版本无关，必须重新请求评审。

### 7.2 评审请求

```
work_publish → review_request(work_item_id, version_no, reviewer_employee_no?)
```

- 未指定评审人 → 解析**默认评审人**：作者 `supervisor_link` 指向的上级。
- 显式指定的评审人**必须已持有 REVIEWER 角色**，否则拒绝。
- 以下三种情况一律**拒绝并明确告知原因，不静默排队**：作者未填上级工号、链接为 `PENDING`（上级尚未注册）、上级尚未获得 REVIEWER 角色。此时作者可显式指定另一位已具备 REVIEWER 角色的评审人。
- 部分唯一索引保证同一事项最多一个 `PENDING`。
- 数据库拒绝自审。

### 7.3 评审与回写

评审人侧：

```
review_inbox()                    → 待办列表（见 §7.4，派生）
review_get(review_request_id)     → 请求 + 该版本材料（写审计）
review_submit(review_request_id, decision, comment)
```

`review_submit` 的落点是**一次原子的比较并交换**：

```sql
UPDATE review_request
   SET status = ?, decided_at = now(), decided_by_account_id = ?, decided_version_id = ?
 WHERE id = ? AND status = 'PENDING'
```

取影响行数；为 0 即失败。同一事务内**重新校验**：

- 当前 `reviewer_account_id` 仍是调用者（防止改派后旧评审人写入）
- 决定的版本仍是请求指向的版本
- 调用者账号 `status='ACTIVE'`（防止停用账号写入）
- 调用者 ≠ 作者（防止自审）

四条全过才提交，同时追加一条 `review_decision`（只追加）。

**改派 / 撤回 / 撤销如何阻止旧请求写入**：

| 动作 | 实现 | 效果 |
|---|---|---|
| 改派 | 管理员改 `reviewer_account_id`，保持 `PENDING` | 旧评审人回写时第一项校验失败 |
| 撤回 | 作者把请求置 `CANCELLED` | CAS 失败，`status` 已不是 `PENDING` |
| 撤销 | 管理员停用账号 | 调用者账号校验失败 |

三者都不需要额外的失效机制——安全性来自回写时的重校验，而不是来自事先的锁。

### 7.4 没有独立待办表（简化）

**待办列表完全派生，不落独立表**：

- 评审人待办 = `review_request WHERE reviewer_account_id = ? AND status = 'PENDING'`
- 作者待办 = `review_request WHERE author_account_id = ? AND status = 'CHANGES_REQUESTED'`

规格里「评审请求与待办记录应一致写入，避免出现『已提交但待办丢失』」这一风险从此**在结构上不可能发生**，不需要任何补偿逻辑、对账任务或双写事务。

这也直接满足「Agent 中途退出后任务可以恢复」：状态从未存放在 Agent 里。Agent 进程死了就是"还没提交"，重启后重新读取即可，**没有锁需要等待过期**。

### 7.5 没有租约机制（简化）

初版设计的「领取 / 租约 / 超时失效」被移除。租约提供的是「互斥占用」这一**便利**，而不是安全边界——安全边界由 §7.3 的回写重校验承担。

**如实说明的代价**：没有租约意味着两个 Agent 可以同时读到同一请求并各自提交，**先到者成功，后到者失败（不是覆盖）**。这浪费一次无用功，但不会产生错误结果。我认为这个代价可以接受，但它是真实存在的，不是零成本的。

### 7.6 幂等

只保留两处必要的地方：

- **`work_publish_version`**：带幂等键。重试返回首次的响应快照，不产生重复版本。
- **`review_submit`**：同一评审人对同一请求提交**相同**决定 → 返回既有结果（成功）；提交**不同**决定 → 拒绝。

两处都先**重新鉴权**再回放响应快照——不能凭幂等键跳过授权检查。

### 7.7 状态机

```
review_request:
  PENDING ──review_submit(APPROVED)────────→ APPROVED
          ──review_submit(CHANGES_REQUESTED)→ CHANGES_REQUESTED
          ──review_withdraw() / 管理员撤销──→ CANCELLED

CHANGES_REQUESTED ──作者 work_publish_version()──→ 新版本（旧请求保持 CHANGES_REQUESTED）
                                                    新版本可再次 review_request（新 PENDING）
```

---

## 8. MCP 工具契约

所有工具在 `McpTransportContext` 中解析调用者身份（每请求解析，不缓存）。每个失败响应都带**稳定的错误码**，便于 Skill 与 Agent 判断可重试性。

### 8.1 身份

| 工具 | 说明 |
|---|---|
| `identity_whoami` | 返回当前账号、角色、上级链接状态 |
| `identity_request_role(role, reason)` | 申请 REVIEWER / OWNER，落 `role_grant_request` |
| `identity_token_status` | 返回令牌前缀、签发时间、最后使用时间 |
| `identity_rotate_token()` | 轮换：旧令牌立即失效，返回新令牌一次 |

注册不在 MCP 上（§5.1），走 `POST /api/agent/register`。

### 8.2 工作与版本

| 工具 | 说明 |
|---|---|
| `work_create(title, description, summary)` | 建事项 + 首版 |
| `work_publish_version(work_item_id, description, summary, change_note, attachment_ids[])` | 追加版本，带幂等键 |
| `work_list_mine(cursor?, limit?)` | 自己的工作事项 |
| `work_get(work_item_id)` | 事项元数据与版本列表，**不含正文** |
| `work_get_version(work_item_id, version_no)` | 版本正文 + 附件元数据（写审计） |
| `work_summary(scope)` | 按角色返回汇总：OWNER 全组织，REVIEWER 自己队列，MEMBER 自己 |

### 8.3 附件

| 工具 | 说明 |
|---|---|
| `attachment_stage_begin(filename, size_bytes, sha256, content_type)` | 返回 `attachment_id` + 一次性上传 URL |
| `attachment_stage_status(attachment_id)` | `VERIFYING` / `READY` / `FAILED` |
| `attachment_download_url(attachment_id)` | 短期签名下载 URL |

上传与下载本身走 HTTP（`PUT` / `GET`），不走 MCP——二进制流塞进 JSON-RPC 需要 base64，浪费且不利于大文件。

### 8.4 评审

| 工具 | 说明 |
|---|---|
| `review_request(work_item_id, version_no, reviewer_employee_no?)` | 创建评审请求 |
| `review_inbox(cursor?, limit?)` | 评审人待办（派生） |
| `review_get(review_request_id)` | 请求 + 材料（写审计） |
| `review_submit(review_request_id, decision, comment)` | 提交决定，原子 CAS |
| `review_withdraw(review_request_id)` | 作者撤回 |
| `review_my_author_inbox(cursor?, limit?)` | 作者侧：待处理的修改意见（派生） |

---

## 9. Dashboard 与管理台

前端为 React SPA（Vite + TypeScript），构建产物进 `static/`，与后端同容器，无 CORS。UI 使用 `/design-taste-frontend` skill 设计。

### 9.1 Dashboard

| 角色 | 首屏 |
|---|---|
| 员工 | 我的工作事项、当前版本、待处理的修改意见、我的评审请求状态 |
| 领导 | 待评审队列、我已提交的意见、我评审过的事项 |
| 老板 | 全组织汇总：已归档数量、评审中、待处理修改、超期未评审的异常项 |

「异常」定义为可计算的事实，不是主观判断：长期 `PENDING` 未决的评审请求、上级链接长期 `PENDING` 的账号、暂存区长期未认领的附件。

### 9.2 管理台

| 功能 | 说明 |
|---|---|
| 成员列表与准入 | 查看成员、停用、恢复 |
| **角色申请审批** | 列出 `PENDING` 的 `role_grant_request`，批准 / 拒绝并留审计 |
| 权限调整 | 直接授予 / 撤销 REVIEWER、OWNER（经审计） |
| 评审改派 | 改 `reviewer_account_id`，保持 `PENDING` |
| 身份更正 | 更正 `employee_no`、姓名、部门、岗位；工号更正需**级联重指关系**并把旧工号标记为已失效 |
| 请求撤销 | 撤销任一 `PENDING` 评审请求 |
| 审计查询 | 按操作者、动作、目标、时间检索 |

管理台**默认看不到材料正文**。

### 9.3 管理台账号

`admin` 断链账号，用于首次授权（否则第一个角色申请无人可批）：

- 密码来自 `A2AHUB_ADMIN_PASSWORD` 环境变量；未设置则**随机生成**，打印一次并写入权限 `0600` 的文件。
- **不预置默认密码，不硬编码，不做隐藏后门。**
- **首次登录强制改密**，且该强制在**所有认证路径**上生效（包括一次性票据路径）。

---

## 10. 客户端接入

**本节区分「已实测」「按官方资料配置但未实测」「暂不支持」三类，不得混同。**

### 10.1 Level A —— 已实测

自动化测试直接以**原始 MCP JSON-RPC over Streamable HTTP** 驱动运行中的服务端，覆盖：

- 单端点 POST + GET
- `Accept: application/json, text/event-stream`
- `MCP-Session-Id`、`MCP-Protocol-Version` 头
- `Origin` 校验（含拒绝非法 Origin 的反向用例）
- 无令牌 / 过期令牌 / 已撤销令牌的拒绝
- 完整业务闭环（注册 → 上传 → 发布 → 请求评审 → 拉取 → 回写 → 新版本）

这是仓库内真实可跑的测试，不依赖任何第三方客户端。

### 10.2 Level B —— 按官方资料配置，**未实测**

以下配置依据官方文档与源码，**在本环境无法实测**，需在真实客户端上确认：

**OpenClaw** — 配置文件 `~/.openclaw/openclaw.json`（JSON5），位于 **Gateway 主机**；`OPENCLAW_CONFIG_PATH` 可覆盖。

```json5
{
  mcp: {
    servers: {
      a2ahub: {
        url: "https://<host>/mcp",
        transport: "streamable-http",   // 必须显式写：省略时默认为 sse
        headers: { Authorization: "Bearer ${A2AHUB_TOKEN}" }
      }
    }
  }
}
```

Skill 放在 `~/.openclaw/skills/a2ahub/SKILL.md`。`mcp` 与 `skills` 配置是热生效的，无需重启。

**WorkBuddy** — 配置文件 `~/.workbuddy/mcp.json`：

```json
{
  "mcpServers": {
    "a2ahub": {
      "type": "http",
      "url": "https://<host>/mcp",
      "headers": { "Authorization": "Bearer ${A2AHUB_TOKEN}" }
    }
  }
}
```

Skill 放在 `~/.workbuddy/skills/a2ahub/`。**需要重启客户端并点击「信任」。**

**两个必须如实说明的未实测点**：

1. **WorkBuddy 的 `type: "http"` + `headers` 缺乏桌面端文档支撑。** 其桌面客户端自身文档只记录了 `stdio`；`http` / `streamable-http` 与 `headers` 认证方式仅有 CLI 文档与第三方厂商资料佐证。**必须在一台真实客户端上确认后才能声称支持。**
2. **OpenClaw 的 `exec` 工具受审批策略约束。** 官方文档建议把上传能力暴露为 MCP 工具而不是 `exec curl`。本设计正是如此（上传走 MCP 签发的签名 URL），因此不依赖 `exec`；但如果 Skill 里写了任何 `exec` 辅助脚本，它会被策略网关拦截。

### 10.3 Level C —— 暂不支持

- 任何 MCP `stdio` 传输（服务端只提供 Streamable HTTP）
- 无 Linux 构建的 WorkBuddy 版本
- 未在 §10.2 列出的其他客户端路径

### 10.4 客户端支持声明纪律

验收标准 §15.19 要求**客户端支持性主张必须有证据**。因此：

- Level A 的每一条都有仓库内可运行的测试作为证据。
- Level B 的每一条都必须标注「未实测」直到有人在真实客户端上跑通并留下记录。
- **不得把未测试能力描述为已支持。**

---

## 11. 可靠性与运维

### 11.1 后台任务

首期只有一个后台任务：**暂存区孤儿清理**。

因为待办完全派生（§7.4），不存在需要后台推进的待办。实现为 `@Scheduled` + `pg_try_advisory_lock`（单实例部署，避免重复执行）。清理超出保留期且 `version_id IS NULL` 的 `STAGED` 附件，删除前写审计。

**这是相对初版的又一处简化**：初版计划中的 `job_queue` 表与 Spring Batch 持久化仓库都被移除——在没有独立待办与租约之后，它们没有承载任何东西。

### 11.2 部署

Docker Compose：`app` 容器（Spring Boot + 内嵌 React 构建产物）+ `postgres` 容器 + 持久化卷（数据库卷与文件存储卷分开）。配置经环境变量注入：

| 变量 | 说明 |
|---|---|
| `A2AHUB_ADMIN_PASSWORD` | 管理台密码；缺省则随机生成并写 0600 文件 |
| `A2AHUB_STORAGE_ROOT` | 文件存储根目录 |
| `SPRING_DATASOURCE_*` | 数据库连接 |
| `A2AHUB_BASE_URL` | 对外地址，用于生成签名 URL |

Actuator liveness/readiness 探针（Boot 4 默认开启）作为容器健康检查。结构化日志用 Boot 4 原生 `logging.structured.format.console=ecs`。

### 11.3 备份与恢复

**数据库与文件必须联合备份、联合恢复**——只恢复其中一个会导致附件记录与磁盘文件不一致。

`scripts/backup.sh`：`pg_dump` + 文件存储目录打包，产出带时间戳的一对文件。
`scripts/backup-verify.sh`：**真实验证**，不是打印成功：

1. 恢复到临时数据库与临时目录
2. 逐条重算 `attachment` 中每个文件的 sha256，与库中记录比对
3. 比对行数、检查外键完整性
4. 输出可核对的差异报告；任一不符则非零退出

这直接满足验收标准 §15.18「数据库与文件备份可联合恢复，并给出真实验证结果」。

---

## 12. 已知残余风险

以下问题**在本设计中没有被解决**，如实列出，不做粉饰：

1. **ADMIN 事实上是超级管理员。** 管理台能授予 OWNER，因此 ADMIN 可以给自己授 OWNER 从而读取全组织材料。「管理台默认看不到材料正文」是一条**默认行为，不是安全边界**。首期单公司内网部署下接受此风险；若要真正隔离，需要把角色授予拆成需要两方确认的流程。

2. **工号抢注是开放注册 + 自报上级模型的固有代价。** 任何人只要先注册了某个工号，就把该工号占了；被冒用工号的真实员工无法再注册。缓解手段是注册时的展示核对、上级关系激活时通知双方、以及管理员的级联更正——但**抢注无法在发生前被阻止**，且已经被看过的内容无法"没看过"。

3. **开放注册意味着内网任何人都能注册并上传。** 因此**每账号存储配额、上传频率限制、孤儿附件清理是必需的配套**，不是可选优化项。

4. **无租约导致并发评审时后提交者失败**（§7.5）。不产生错误结果，但会浪费一次无用功。

5. **WorkBuddy 的 HTTP 传输 + headers 认证未实测**（§10.2）。在此之前不能声称支持。

6. **Spring AI 2.0.1 实现的是 MCP 规范 `2025-11-25`**，落后当前修订一个版本。若客户端按更新修订校验，可能出现兼容性问题。

---

## 13. 验收标准

规格 §15 的 20 条，在本设计下的状态：

| 编号 | 内容 | 状态 |
|---|---|---|
| 15.1 | 普通注册无法获得老板权限 | **不变**（注册只给 MEMBER） |
| 15.2 | 邀请注册 + 接受 | **改写**：自助注册得 MEMBER；REVIEWER / OWNER 经申请 + 管理员批准 |
| 15.3 | pending / disabled 账号无法访问组织材料 | **改写**：无 pending 态；`DISABLED` 账号无法访问任何材料 |
| 15.4 | 自报岗位/部门/上级不能提升权限 | **不变**，语义变化：自报产生**申请记录**，权限仍只来自管理员批准 |
| 15.5 | 归档版本不可覆盖，更新生成新版本 | 不变（措辞去掉"草稿"） |
| 15.6 | 草稿仅作者可见，已归档材料按明确授权访问 | **改写**：不存在草稿态；所有已发布版本按明确授权访问 |
| 15.7 – 15.11 | — | 不变 |
| 15.12 | Agent 中途退出后任务可恢复，过期尝试不能覆盖新结果 | **不变**（无租约后由 §7.3 回写重校验保证） |
| 15.13 | 改派 / 撤回 / 撤销能阻止旧请求写入 | **不变** |
| 15.14 – 15.17 | — | 措辞将「草稿」替换为「已发布版本」，语义不变 |
| 15.18 | 数据库与文件备份可联合恢复并给出真实验证结果 | 不变（§11.3） |
| 15.19 | 客户端支持性主张需有证据而非假设 | 不变（§10.4） |
| 15.20 | 员工关机、领导换客户端、重试，仍然可用 | 不变 |

---

## 14. 交付计划

逐阶段实现，每阶段验收。

| 阶段 | 内容 | 验收 |
|---|---|---|
| 1 | 初始化、身份、凭证 | 注册 → 得令牌 → 落盘 → MCP 可认证；角色申请与批准闭环；停用账号被拒 |
| 2 | 文件归档、查询、不可变版本 | 上传 → 校验 → 暂存 → 发布 → 下载；摘要不符被拒；版本不可覆盖 |
| 3 | 评审请求、异步拉取、结果回写 | 请求 → 派生待办 → 拉取 → 回写；改派/撤回/撤销后旧写入失败；自审被拒 |
| 4 | Dashboard 与管理台 | 三种角色首屏正确；管理台全部操作留审计 |
| 5 | 失败恢复、部署、验收 | Compose 起停；备份恢复真实验证；Level A 客户端测试；验收标准逐条过 |

交付物：可运行的后端 + 前端、Skill 与辅助脚本、`docker-compose.yml`、配置样例、Flyway 迁移、部署/初始化/凭证恢复/备份恢复文档、客户端接入文档（含实测与未实测边界）、核心业务约束测试、端到端验收脚本。

**不得留下 TODO 形式的核心上传、鉴权、存储与评审实现；不得产出无法持久化的演示。**

---

## 15. 参考

- MCP Streamable HTTP 传输规范（`Origin` 校验为 MUST）
- Spring AI 2.0.1 MCP Server Boot Starter 文档
- Spring Boot 4.1 迁移说明（Jackson 3、Flyway starter、Actuator 权限模型、Testcontainers 坐标变更）
- OpenClaw 官方配置文档（`~/.openclaw/openclaw.json`、`mcp.servers`、Skills 目录）
- WorkBuddy 官方配置文档（`~/.workbuddy/mcp.json`、Skills 目录）
