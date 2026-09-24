# A2AHub 阶段 1：初始化、身份与凭证 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让员工能自助注册、拿到一次性令牌、通过 MCP 完成认证，让角色（REVIEWER / OWNER）经申请 + 管理员批准生效，并保证停用账号在所有路径上被拒绝。

**Architecture:** Spring Boot 4 单体应用。身份数据落 PostgreSQL，经 Flyway 迁移管理。认证用不透明令牌（服务端存 SHA-256 摘要）。MCP 走 Streamable HTTP，身份由 Spring Security 过滤器链在每请求解析，工具方法从 `SecurityContextHolder` 读取——不缓存权限快照。

**Tech Stack:** Java 21 / Maven / Spring Boot 4.1.1 / Spring Framework 7.0.9 / Spring AI 2.0.1 / `JdbcClient` / Flyway / PostgreSQL 17 / Testcontainers 2.x / JUnit 6

**Spec:** `docs/superpowers/specs/2026-09-24-a2ahub-mvp-design.md`

## Global Constraints

以下为规格中的全局要求，**每个任务都隐含包含本节**。数值逐字取自规格。

- Java 21；Spring Boot 4.1.1；Spring AI 2.0.1（1.x 不支持 Boot 4）
- 数据访问用 `JdbcClient`，**禁止引入 JPA / Hibernate**
- 用 `spring-boot-starter-flyway`，且**必须**显式引入 `org.flywaydb:flyway-database-postgresql`（Flyway 10+ 拆出）；不覆盖 Boot 托管的 Flyway 版本
- 令牌：格式 `a2ah_<uuid>.<secret>`；服务端只存 `SHA-256(secret)` 十六进制；`prefix` 为 secret 前 8 位
- **令牌原文不落库、不进日志、不放进 URL**
- 注册只产生 `MEMBER` 角色。**注册不产生任何其他角色**
- `name` / `department` / `position` 是个人资料，**不产生任何权限**
- 角色只能是 `MEMBER` / `REVIEWER` / `OWNER`；账号状态只能是 `ACTIVE` / `DISABLED`
- 管理员密码来自 `A2AHUB_ADMIN_PASSWORD`；**未设置则随机生成，不预置默认值、不硬编码、不做隐藏后门**
- 强制首次改密在**所有认证路径**上生效
- 每次请求重新解析账号状态与角色，**不缓存权限快照**
- MCP 端点**必须校验 `Origin` 头**
- 结构化日志：`logging.structured.format.console=ecs`
- Testcontainers 2.x 坐标是 `org.testcontainers:testcontainers-postgresql`，包名 `org.testcontainers.postgresql.PostgreSQLContainer`（旧的 `postgresql` 坐标已 404）
- 测试用 JUnit 6；`@MockBean` / `@SpyBean` 已移除，用 `@MockitoBean` / `@MockitoSpyBean`

## Review Focus

规格是一份愿景文档，它规定了系统必须做什么，但没有穷举系统会遇到什么。**规格对某个输入的沉默，不等于允许该输入把程序打崩。** 以下五类输入/失败模式最容易伤到真实使用者，按可能性排序。每一类都在其归属任务的测试里被固定住：

1. **并发注册同一工号** —— 两个请求同时提交同一个 `employee_no`。使用者期望后到者得到一句干净的「工号已占用」，而不是数据库唯一约束抛出的 500。（Task 4）
2. **格式合法但 id 不存在的令牌** —— `a2ah_<不存在的uuid>.<secret>`。使用者期望 401，而不是 NPE 或 500。（Task 5）
3. **上级工号指向已停用账号** —— 规格只覆盖了「尚未注册」和「自环/环路」，没覆盖「目标存在但已停用」。使用者期望这条链接不产生可用的上级权威。（Task 8）
4. **工号的大小写、首尾空白与超长** —— `"e12345"` 与 `"E12345"`、`"  "`、10KB 长的名字。使用者期望归一化后判重、空白拒绝、长度有上限。（Task 3）
5. **MCP 请求缺少 `Origin` 头** —— 非浏览器的 Agent 客户端不会发这个头。使用者期望**放行**；只拒绝**不匹配**的 Origin。（Task 6）

---

## 文件结构

```
src/main/java/com/echoyan/a2ahub/
  A2AHubApplication.java                        已存在
  config/
    AppProperties.java                          配置绑定
    SecurityConfig.java                         两条过滤器链
    McpConfig.java                              MCP 服务端配置
  identity/
    Account.java                                record
    Role.java                                   enum
    AccountRepository.java                      账号 SQL
    AccountService.java                         账号生命周期（建/停用/恢复/更正）
    InputNormalizer.java                        工号与资料输入的归一化与校验
    RegistrationService.java                    注册事务
    ApiToken.java                               record
    TokenService.java                           签发/校验/轮换
    TokenAuthFilter.java                        Bearer 认证过滤器
    SupervisorLink.java                         record
    SupervisorLinkRepository.java               上级链接 SQL
    SupervisorLinkService.java                  建立/激活/环路检测
    RoleGrantRequest.java                       record
    RoleGrantRepository.java                    角色申请 SQL
    RoleGrantService.java                       申请与批准
    AdminBootstrap.java                         引导管理员账号
    IdentityTools.java                          MCP 工具
  audit/
    AuditService.java                           只追加审计
  web/
    RegistrationController.java                 POST /api/agent/register
    OriginCheckFilter.java                      MCP Origin 校验
    AdminController.java                        角色申请审批等管理端点
    ApiExceptionHandler.java                    统一错误码
  security/
    CurrentAccount.java                         record，当前调用者
    CurrentAccountResolver.java                 每请求解析，不缓存
    McpIdentity.java                            MCP 工具读取身份的入口

src/main/resources/
  application.yaml
  db/migration/V1__identity.sql

src/test/java/com/echoyan/a2ahub/
  support/IntegrationTestBase.java              Testcontainers 基类
  identity/…  web/…  mcp/…                      各任务的测试

skill/a2ahub/
  SKILL.md                                      Skill 引导文档
  scripts/write-credentials.sh                  凭证落盘（0600）
  scripts/mcp-config.json.tmpl                  MCP 配置模板
```

---

## Task 1: 依赖与配置基线

**Files:**
- Modify: `pom.xml`
- Modify: `src/main/resources/application.yaml`
- Create: `src/test/java/com/echoyan/a2ahub/support/IntegrationTestBase.java`
- Test: `src/test/java/com/echoyan/a2ahub/support/BaselineTest.java`

**Interfaces:**
- Consumes: 无（首个任务）
- Produces: `IntegrationTestBase` —— 后续所有集成测试的基类，提供已迁移的 PostgreSQL 17 容器与 `JdbcClient`

- [ ] **Step 1: 先确认 Spring AI 2.0.1 真的能解析**

不要凭假设写版本号。先跑：

```bash
cd /Users/echoyan/Developer/A2AHub
mvn -B -q dependency:get -Dartifact=org.springframework.ai:spring-ai-bom:2.0.1:pom
```

预期：BUILD SUCCESS。

若报 `Could not find artifact`，说明 2.0.1 尚未进 Maven Central，需在 `pom.xml` 加里程碑仓库后再继续：

```xml
<repositories>
  <repository>
    <id>spring-milestones</id>
    <url>https://repo.spring.io/milestone</url>
  </repository>
</repositories>
```

**在得到确定结果之前不要进入下一步。** 如果两个源都找不到 2.0.1，停下来报告——这意味着 MCP 方案需要重新选择，属于影响产品方向的变更。

- [ ] **Step 2: 加入依赖**

`pom.xml` 的 `<dependencies>` 中新增（保留现有的 webmvc / postgresql / lombok / webmvc-test）：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-flyway</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

测试依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers-postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

在 `</project>` 前加 BOM 管理：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>2.0.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

- [ ] **Step 3: 写配置**

`src/main/resources/application.yaml` 整体替换为：

```yaml
spring:
  application:
    name: A2AHub
  datasource:
    url: ${A2AHUB_DB_URL:jdbc:postgresql://localhost:5432/a2ahub}
    username: ${A2AHUB_DB_USER:a2ahub}
    password: ${A2AHUB_DB_PASSWORD:a2ahub}
  flyway:
    enabled: true
  ai:
    mcp:
      server:
        name: a2ahub
        protocol: STREAMABLE

server:
  port: 8080

logging:
  structured:
    format:
      console: ecs

management:
  endpoint:
    health:
      probes:
        enabled: true

a2ahub:
  base-url: ${A2AHUB_BASE_URL:http://localhost:8080}
  allowed-origins: ${A2AHUB_ALLOWED_ORIGINS:http://localhost:8080}
  admin-password: ${A2AHUB_ADMIN_PASSWORD:}
  bootstrap-dir: ${A2AHUB_BOOTSTRAP_DIR:./data/bootstrap}
```

`admin-password` 默认为**空**——这是刻意的，Task 10 会把空值解释为"随机生成"。

- [ ] **Step 4: 写 Testcontainers 基类**

```java
package com.echoyan.a2ahub.support;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.ActiveProfiles;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
public abstract class IntegrationTestBase {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17-alpine");

    static {
        POSTGRES.start();
    }
}
```

注意包名是 `org.testcontainers.containers.PostgreSQLContainer`——Testcontainers 2.x 把模块拆成了 `testcontainers-postgresql` 坐标，但类仍在 `containers` 包下。若编译报找不到类，用 `mvn dependency:tree -Dincludes=org.testcontainers` 确认实际包路径后再改 import。

- [ ] **Step 5: 写基线测试**

```java
package com.echoyan.a2ahub.support;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import static org.assertj.core.api.Assertions.assertThat;

class BaselineTest extends IntegrationTestBase {

    @Autowired
    JdbcClient jdbcClient;

    @Test
    void 上下文可加载且数据库可连接() {
        Integer one = jdbcClient.sql("SELECT 1").query(Integer.class).single();
        assertThat(one).isEqualTo(1);
    }
}
```

- [ ] **Step 6: 运行测试**

Run: `mvn -B test -Dtest=BaselineTest`
Expected: PASS。首次运行会拉取 PostgreSQL 镜像，可能较慢。

- [ ] **Step 7: 提交**

```bash
git add pom.xml src/main/resources/application.yaml src/test/java/com/echoyan/a2ahub/support/
git commit -m "chore: 补齐阶段1依赖与 Testcontainers 测试基线"
```

---

## Task 2: 数据库迁移 V1（身份相关六张表）

**Files:**
- Create: `src/main/resources/db/migration/V1__identity.sql`
- Test: `src/test/java/com/echoyan/a2ahub/identity/IdentitySchemaTest.java`

**Interfaces:**
- Consumes: `IntegrationTestBase`（Task 1）
- Produces: 六张表的物理结构——`account`、`account_role`、`api_token`、`supervisor_link`、`role_grant_request`、`audit_log`。后续所有 Repository 直接依赖这些列名。

- [ ] **Step 1: 写迁移脚本**

```sql
-- V1__identity.sql
-- 身份、凭证、角色、上级链接、审计

CREATE TABLE account (
    id                    uuid PRIMARY KEY,
    employee_no           text NOT NULL,
    name                  text NOT NULL,
    department            text NOT NULL,
    position              text NOT NULL,
    status                text NOT NULL DEFAULT 'ACTIVE',
    is_admin              boolean NOT NULL DEFAULT false,
    password_digest       text,
    must_change_password  boolean NOT NULL DEFAULT false,
    created_at            timestamptz NOT NULL DEFAULT now(),
    updated_at            timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ux_account_employee_no UNIQUE (employee_no),
    CONSTRAINT ck_account_status CHECK (status IN ('ACTIVE', 'DISABLED')),
    CONSTRAINT ck_account_employee_no_not_blank CHECK (btrim(employee_no) <> '')
);

CREATE TABLE account_role (
    account_id            uuid NOT NULL REFERENCES account (id),
    role                  text NOT NULL,
    granted_by_account_id uuid REFERENCES account (id),
    granted_at            timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (account_id, role),
    CONSTRAINT ck_account_role CHECK (role IN ('MEMBER', 'REVIEWER', 'OWNER'))
);

CREATE TABLE api_token (
    id           uuid PRIMARY KEY,
    account_id   uuid NOT NULL REFERENCES account (id),
    digest       text NOT NULL,
    prefix       text NOT NULL,
    label        text NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    last_used_at timestamptz,
    revoked_at   timestamptz
);
CREATE INDEX ix_api_token_account ON api_token (account_id);

CREATE TABLE supervisor_link (
    id                     uuid PRIMARY KEY,
    subordinate_account_id uuid NOT NULL REFERENCES account (id),
    supervisor_employee_no text NOT NULL,
    supervisor_account_id  uuid REFERENCES account (id),
    status                 text NOT NULL,
    created_at             timestamptz NOT NULL DEFAULT now(),
    activated_at           timestamptz,
    CONSTRAINT ux_supervisor_link_subordinate UNIQUE (subordinate_account_id),
    CONSTRAINT ck_supervisor_link_status CHECK (status IN ('PENDING', 'ACTIVE', 'REJECTED')),
    CONSTRAINT ck_supervisor_link_not_self CHECK (subordinate_account_id <> supervisor_account_id)
);
CREATE INDEX ix_supervisor_link_pending
    ON supervisor_link (supervisor_employee_no) WHERE status = 'PENDING';

CREATE TABLE role_grant_request (
    id                    uuid PRIMARY KEY,
    account_id            uuid NOT NULL REFERENCES account (id),
    requested_role        text NOT NULL,
    reason                text NOT NULL,
    status                text NOT NULL DEFAULT 'PENDING',
    created_at            timestamptz NOT NULL DEFAULT now(),
    decided_by_account_id uuid REFERENCES account (id),
    decided_at            timestamptz,
    decision_note         text,
    CONSTRAINT ck_rgr_role CHECK (requested_role IN ('REVIEWER', 'OWNER')),
    CONSTRAINT ck_rgr_status CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED'))
);
CREATE UNIQUE INDEX ux_rgr_one_pending
    ON role_grant_request (account_id, requested_role) WHERE status = 'PENDING';

CREATE TABLE audit_log (
    id               bigserial PRIMARY KEY,
    at               timestamptz NOT NULL DEFAULT now(),
    actor_account_id uuid REFERENCES account (id),
    actor_type       text NOT NULL,
    action           text NOT NULL,
    target_type      text,
    target_id        text,
    detail           jsonb,
    request_id       text,
    CONSTRAINT ck_audit_actor_type CHECK (actor_type IN ('ACCOUNT', 'ADMIN', 'SYSTEM'))
);
CREATE INDEX ix_audit_at ON audit_log (at DESC);
```

- [ ] **Step 2: 写迁移测试**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.jdbc.core.simple.JdbcClient;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class IdentitySchemaTest extends IntegrationTestBase {

    @Autowired
    JdbcClient jdbc;

    private void insertAccount(UUID id, String employeeNo) {
        jdbc.sql("""
                INSERT INTO account (id, employee_no, name, department, position)
                VALUES (:id, :no, '张三', '研发部', '工程师')
                """)
            .param("id", id).param("no", employeeNo).update();
    }

    @Test
    void 迁移后六张表都存在() {
        Integer n = jdbc.sql("""
                SELECT count(*) FROM information_schema.tables
                 WHERE table_schema = 'public'
                   AND table_name IN ('account','account_role','api_token',
                                      'supervisor_link','role_grant_request','audit_log')
                """).query(Integer.class).single();
        assertThat(n).isEqualTo(6);
    }

    @Test
    void 工号唯一() {
        insertAccount(UUID.randomUUID(), "E1001");
        assertThatThrownBy(() -> insertAccount(UUID.randomUUID(), "E1001"))
                .isInstanceOf(DataIntegrityViolationException.class);
    }

    @Test
    void 工号不能是纯空白() {
        assertThatThrownBy(() -> insertAccount(UUID.randomUUID(), "   "))
                .isInstanceOf(DataIntegrityViolationException.class);
    }

    @Test
    void 同一账号同一角色同时只能有一个待批申请() {
        UUID id = UUID.randomUUID();
        insertAccount(id, "E1002");
        jdbc.sql("""
                INSERT INTO role_grant_request (id, account_id, requested_role, reason)
                VALUES (:id, :acc, 'OWNER', '我是负责人')
                """).param("id", UUID.randomUUID()).param("acc", id).update();

        assertThatThrownBy(() -> jdbc.sql("""
                INSERT INTO role_grant_request (id, account_id, requested_role, reason)
                VALUES (:id, :acc, 'OWNER', '再申请一次')
                """).param("id", UUID.randomUUID()).param("acc", id).update())
            .isInstanceOf(DataIntegrityViolationException.class);
    }

    @Test
    void 上级链接不能指向自己() {
        UUID id = UUID.randomUUID();
        insertAccount(id, "E1003");
        assertThatThrownBy(() -> jdbc.sql("""
                INSERT INTO supervisor_link
                    (id, subordinate_account_id, supervisor_employee_no, supervisor_account_id, status)
                VALUES (:id, :acc, 'E1003', :acc, 'ACTIVE')
                """).param("id", UUID.randomUUID()).param("acc", id).update())
            .isInstanceOf(DataIntegrityViolationException.class);
    }
}
```

- [ ] **Step 3: 运行测试**

Run: `mvn -B test -Dtest=IdentitySchemaTest`
Expected: 5 个测试全部 PASS。

- [ ] **Step 4: 提交**

```bash
git add src/main/resources/db/migration/V1__identity.sql src/test/java/com/echoyan/a2ahub/identity/IdentitySchemaTest.java
git commit -m "feat(db): 身份相关六张表的 Flyway 迁移"
```

---

## Task 3: 工号与资料输入的归一化与校验

**Files:**
- Create: `src/main/java/com/echoyan/a2ahub/identity/InputNormalizer.java`
- Test: `src/test/java/com/echoyan/a2ahub/identity/InputNormalizerTest.java`

**Interfaces:**
- Consumes: 无
- Produces:
  - `static String normalizeEmployeeNo(String raw)` —— 去首尾空白、转大写；空白或超长抛 `IllegalArgumentException`
  - `static String normalizeDisplayText(String raw, String fieldName, int maxLen)` —— 去首尾空白、压缩内部连续空白；空白或超长抛 `IllegalArgumentException`
  - `static void validateReason(String raw)` —— reason 字段校验，上限 500 字

**这是 Review Focus 第 4 类。** 规格没有规定工号的大小写与空白行为；本任务把它定死为「归一化后判重」。

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.identity;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class InputNormalizerTest {

    @Test
    void 工号去空白并转大写() {
        assertThat(InputNormalizer.normalizeEmployeeNo("  e12345  ")).isEqualTo("E12345");
    }

    @Test
    void 大小写不同的工号归一化后相同() {
        assertThat(InputNormalizer.normalizeEmployeeNo("e12345"))
                .isEqualTo(InputNormalizer.normalizeEmployeeNo("E12345"));
    }

    @Test
    void 空白工号被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.normalizeEmployeeNo("   "))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("工号");
    }

    @Test
    void null工号被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.normalizeEmployeeNo(null))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void 超长工号被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.normalizeEmployeeNo("E".repeat(65)))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void 工号允许字母数字与连字符下划线() {
        assertThat(InputNormalizer.normalizeEmployeeNo("e-10_2")).isEqualTo("E-10_2");
    }

    @Test
    void 工号含非法字符被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.normalizeEmployeeNo("E12345;DROP"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void 姓名压缩内部空白() {
        assertThat(InputNormalizer.normalizeDisplayText("  张  三  ", "姓名", 64))
                .isEqualTo("张 三");
    }

    @Test
    void 超长姓名被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.normalizeDisplayText("张".repeat(65), "姓名", 64))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("姓名");
    }

    @Test
    void 空白姓名被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.normalizeDisplayText("\t\n", "姓名", 64))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void 申请理由超长被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.validateReason("x".repeat(501)))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("理由");
    }

    @Test
    void 空申请理由被拒绝() {
        assertThatThrownBy(() -> InputNormalizer.validateReason("  "))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=InputNormalizerTest`
Expected: 编译失败，`InputNormalizer` 不存在。

- [ ] **Step 3: 实现**

```java
package com.echoyan.a2ahub.identity;

import java.util.regex.Pattern;

/**
 * 注册与资料输入的归一化与校验。
 *
 * <p>工号归一化后判重，因此 "e12345" 与 "E12345" 是同一个工号。
 * 所有方法对非法输入抛 IllegalArgumentException，由 ApiExceptionHandler 转成 400。
 */
public final class InputNormalizer {

    private static final Pattern EMPLOYEE_NO = Pattern.compile("[A-Z0-9_-]{1,64}");
    private static final int MAX_EMPLOYEE_NO = 64;

    private InputNormalizer() {
    }

    public static String normalizeEmployeeNo(String raw) {
        if (raw == null) {
            throw new IllegalArgumentException("工号不能为空");
        }
        String normalized = raw.strip().toUpperCase();
        if (normalized.isEmpty()) {
            throw new IllegalArgumentException("工号不能为空");
        }
        if (normalized.length() > MAX_EMPLOYEE_NO) {
            throw new IllegalArgumentException("工号长度不能超过 " + MAX_EMPLOYEE_NO + " 个字符");
        }
        if (!EMPLOYEE_NO.matcher(normalized).matches()) {
            throw new IllegalArgumentException("工号只能包含字母、数字、连字符和下划线");
        }
        return normalized;
    }

    public static String normalizeDisplayText(String raw, String fieldName, int maxLen) {
        if (raw == null) {
            throw new IllegalArgumentException(fieldName + "不能为空");
        }
        String normalized = raw.strip().replaceAll("\\s+", " ");
        if (normalized.isEmpty()) {
            throw new IllegalArgumentException(fieldName + "不能为空");
        }
        if (normalized.length() > maxLen) {
            throw new IllegalArgumentException(fieldName + "长度不能超过 " + maxLen + " 个字符");
        }
        return normalized;
    }

    public static void validateReason(String raw) {
        normalizeDisplayText(raw, "申请理由", 500);
    }
}
```

- [ ] **Step 4: 运行确认通过**

Run: `mvn -B test -Dtest=InputNormalizerTest`
Expected: 12 个测试全部 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/identity/InputNormalizer.java src/test/java/com/echoyan/a2ahub/identity/InputNormalizerTest.java
git commit -m "feat(identity): 工号与资料输入的归一化与校验"
```

---

## Task 4: 注册（账户 + MEMBER 角色 + 一次性令牌）

**Files:**
- Create: `src/main/java/com/echoyan/a2ahub/identity/Account.java`
- Create: `src/main/java/com/echoyan/a2ahub/identity/Role.java`
- Create: `src/main/java/com/echoyan/a2ahub/identity/AccountRepository.java`
- Create: `src/main/java/com/echoyan/a2ahub/audit/AuditService.java`
- Create: `src/main/java/com/echoyan/a2ahub/identity/TokenService.java`（本任务只实现 `issue`）
- Create: `src/main/java/com/echoyan/a2ahub/identity/RegistrationService.java`
- Create: `src/main/java/com/echoyan/a2ahub/web/RegistrationController.java`
- Create: `src/main/java/com/echoyan/a2ahub/web/ApiExceptionHandler.java`
- Test: `src/test/java/com/echoyan/a2ahub/identity/RegistrationServiceTest.java`

**Interfaces:**
- Consumes: `InputNormalizer`（Task 3）、六张表（Task 2）
- Produces:
  - `record Account(UUID id, String employeeNo, String name, String department, String position, String status, boolean admin, boolean mustChangePassword)`
  - `enum Role { MEMBER, REVIEWER, OWNER }`
  - `record IssuedToken(String plaintext, String prefix)` —— `TokenService.issue(...)`
  - `record RegistrationResult(UUID accountId, IssuedToken token, SupervisorResolution supervisor)`
  - `record SupervisorResolution(String submittedEmployeeNo, String resolvedName, boolean active)`
  - `RegistrationService.register(RegisterCommand)` 返回 `RegistrationResult`
  - `TokenService.issue(UUID accountId, String label)` 返回 `IssuedToken`
  - `AuditService.record(UUID actorAccountId, String actorType, String action, String targetType, String targetId, Map<String,Object> detail)`

**这是 Review Focus 第 1 类。** 并发注册同一工号必须得到干净的 409。

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RegistrationServiceTest extends IntegrationTestBase {

    @Autowired RegistrationService registrationService;
    @Autowired JdbcClient jdbc;

    private RegistrationService.RegisterCommand cmd(String no) {
        return new RegistrationService.RegisterCommand(no, "张三", "研发部", "工程师", null);
    }

    @Test
    void 注册成功只产生MEMBER角色() {
        RegistrationResult result = registrationService.register(cmd("E2001"));

        List<String> roles = jdbc.sql("SELECT role FROM account_role WHERE account_id = :id")
                .param("id", result.accountId()).query(String.class).list();

        assertThat(roles).containsExactly("MEMBER");
    }

    @Test
    void 注册返回的令牌原文只出现一次且不落库() {
        RegistrationResult result = registrationService.register(cmd("E2002"));
        String plaintext = result.token().plaintext();

        assertThat(plaintext).startsWith("a2ah_");

        // 库里任何位置都不应出现令牌原文
        Integer hits = jdbc.sql("""
                SELECT count(*) FROM api_token
                 WHERE digest = :plain OR prefix = :plain OR id::text = :plain
                """).param("plain", plaintext).query(Integer.class).single();
        assertThat(hits).isZero();

        // 库里存的是摘要，且与原文的 SHA-256 一致
        String stored = jdbc.sql("SELECT digest FROM api_token WHERE account_id = :id")
                .param("id", result.accountId()).query(String.class).single();
        assertThat(stored).isEqualTo(TokenService.sha256Hex(plaintext.substring(plaintext.indexOf('.') + 1)));
    }

    @Test
    void 注册写入审计() {
        registrationService.register(cmd("E2003"));
        Integer n = jdbc.sql("SELECT count(*) FROM audit_log WHERE action = 'ACCOUNT_REGISTERED'")
                .query(Integer.class).single();
        assertThat(n).isEqualTo(1);
    }

    @Test
    void 工号大小写不同视为冲突() {
        registrationService.register(cmd("e2004"));
        assertThatThrownBy(() -> registrationService.register(cmd("E2004")))
                .isInstanceOf(EmployeeNoTakenException.class);
    }

    @Test
    void 并发注册同一工号只有一个成功且失败者是干净的业务异常() throws Exception {
        try (ExecutorService pool = Executors.newFixedThreadPool(8)) {
            List<Callable<Object>> tasks = List.of(
                    () -> attempt("E2099"),
                    () -> attempt("E2099"),
                    () -> attempt("E2099"),
                    () -> attempt("E2099"),
                    () -> attempt("E2099"),
                    () -> attempt("E2099"),
                    () -> attempt("E2099"),
                    () -> attempt("E2099"));

            List<Future<Object>> futures = pool.invokeAll(tasks);
            long succeeded = 0;
            for (Future<Object> f : futures) {
                Object value = f.get();
                if (value instanceof RegistrationResult) {
                    succeeded++;
                } else {
                    // 失败者必须是业务异常，不能是 DataIntegrityViolationException / 500
                    assertThat(value).isInstanceOf(EmployeeNoTakenException.class);
                }
            }
            assertThat(succeeded).isEqualTo(1);

            Integer accounts = jdbc.sql("SELECT count(*) FROM account WHERE employee_no = 'E2099'")
                    .query(Integer.class).single();
            assertThat(accounts).isEqualTo(1);
        }
    }

    private Object attempt(String employeeNo) {
        try {
            return registrationService.register(cmd(employeeNo));
        } catch (Exception e) {
            return e;
        }
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=RegistrationServiceTest`
Expected: 编译失败，相关类不存在。

- [ ] **Step 3: 实现领域类型**

`Account.java`：

```java
package com.echoyan.a2ahub.identity;

import java.util.UUID;

public record Account(
        UUID id,
        String employeeNo,
        String name,
        String department,
        String position,
        String status,
        boolean admin,
        boolean mustChangePassword) {

    public boolean isActive() {
        return "ACTIVE".equals(status);
    }
}
```

`Role.java`：

```java
package com.echoyan.a2ahub.identity;

public enum Role {
    MEMBER, REVIEWER, OWNER
}
```

`EmployeeNoTakenException.java`：

```java
package com.echoyan.a2ahub.identity;

public class EmployeeNoTakenException extends RuntimeException {
    public EmployeeNoTakenException(String employeeNo) {
        super("工号 " + employeeNo + " 已被占用");
    }
}
```

- [ ] **Step 4: 实现 `AccountRepository`**

```java
package com.echoyan.a2ahub.identity;

import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Repository;

import java.util.Optional;
import java.util.UUID;

@Repository
public class AccountRepository {

    private final JdbcClient jdbc;

    public AccountRepository(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }

    /**
     * 插入账号，工号冲突时不报错而是静默跳过。
     *
     * @return 影响行数：1 表示插入成功，0 表示工号已被占用
     */
    public int insertIfAbsent(UUID id, String employeeNo, String name,
                              String department, String position) {
        return jdbc.sql("""
                INSERT INTO account (id, employee_no, name, department, position)
                VALUES (:id, :no, :name, :dept, :pos)
                ON CONFLICT (employee_no) DO NOTHING
                """)
            .param("id", id)
            .param("no", employeeNo)
            .param("name", name)
            .param("dept", department)
            .param("pos", position)
            .update();
    }

    /** 已存在的工号返回其账号 id，用于区分「已占用」与其它完整性冲突。 */
    public Optional<UUID> findIdByEmployeeNo(String employeeNo) {
        return jdbc.sql("SELECT id FROM account WHERE employee_no = :no")
                .param("no", employeeNo)
                .query(UUID.class)
                .optional();
    }

    public void grantRole(UUID accountId, Role role, UUID grantedBy) {
        jdbc.sql("""
                INSERT INTO account_role (account_id, role, granted_by_account_id)
                VALUES (:acc, :role, :by)
                ON CONFLICT (account_id, role) DO NOTHING
                """)
            .param("acc", accountId)
            .param("role", role.name())
            .param("by", grantedBy)
            .update();
    }
}
```

- [ ] **Step 5: 实现 `AuditService`**

```java
package com.echoyan.a2ahub.audit;

import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Service;
import tools.jackson.databind.ObjectMapper;

import java.util.Map;
import java.util.UUID;

/** 只追加的审计写入。没有更新与删除方法，这是刻意的。 */
@Service
public class AuditService {

    private final JdbcClient jdbc;
    private final ObjectMapper objectMapper;

    public AuditService(JdbcClient jdbc, ObjectMapper objectMapper) {
        this.jdbc = jdbc;
        this.objectMapper = objectMapper;
    }

    public void record(UUID actorAccountId, String actorType, String action,
                       String targetType, String targetId, Map<String, Object> detail) {
        jdbc.sql("""
                INSERT INTO audit_log
                    (actor_account_id, actor_type, action, target_type, target_id, detail)
                VALUES (:actor, :actorType, :action, :targetType, :targetId, CAST(:detail AS jsonb))
                """)
            .param("actor", actorAccountId)
            .param("actorType", actorType)
            .param("action", action)
            .param("targetType", targetType)
            .param("targetId", targetId)
            .param("detail", objectMapper.writeValueAsString(detail == null ? Map.of() : detail))
            .update();
    }
}
```

注意 `ObjectMapper` 来自 `tools.jackson.databind`——Spring Boot 4 默认 Jackson 3。

- [ ] **Step 6: 实现 `TokenService`（本任务只做 `issue` 与 `sha256Hex`）**

```java
package com.echoyan.a2ahub.identity;

import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Service;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.HexFormat;
import java.util.UUID;

@Service
public class TokenService {

    private static final SecureRandom RANDOM = new SecureRandom();
    private static final Base64.Encoder B64 = Base64.getUrlEncoder().withoutPadding();

    private final JdbcClient jdbc;

    public TokenService(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }

    public record IssuedToken(String plaintext, String prefix) {
    }

    /** 生成令牌并只存摘要。返回的 plaintext 是本令牌唯一一次以原文出现。 */
    public IssuedToken issue(UUID accountId, String label) {
        UUID tokenId = UUID.randomUUID();
        byte[] secretBytes = new byte[32];
        RANDOM.nextBytes(secretBytes);
        String secret = B64.encodeToString(secretBytes);

        jdbc.sql("""
                INSERT INTO api_token (id, account_id, digest, prefix, label)
                VALUES (:id, :acc, :digest, :prefix, :label)
                """)
            .param("id", tokenId)
            .param("acc", accountId)
            .param("digest", sha256Hex(secret))
            .param("prefix", secret.substring(0, 8))
            .param("label", label)
            .update();

        return new IssuedToken("a2ah_" + tokenId + "." + secret, secret.substring(0, 8));
    }

    public static String sha256Hex(String value) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(md.digest(value.getBytes(StandardCharsets.UTF_8)));
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 不可用", e);
        }
    }
}
```

- [ ] **Step 7: 实现 `RegistrationService`**

关键点：**先查重给出友好错误，再依赖唯一约束兜底并发**。捕获 `DuplicateKeyException` 后重新查询确认是工号冲突还是别的原因，只在确认冲突时转成 `EmployeeNoTakenException`。

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.audit.AuditService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Map;
import java.util.UUID;

@Service
public class RegistrationService {

    private final AccountRepository accounts;
    private final TokenService tokens;
    private final AuditService audit;

    public RegistrationService(AccountRepository accounts, TokenService tokens, AuditService audit) {
        this.accounts = accounts;
        this.tokens = tokens;
        this.audit = audit;
    }

    public record RegisterCommand(String employeeNo, String name, String department,
                                  String position, String supervisorEmployeeNo) {
    }

    public record SupervisorResolution(String submittedEmployeeNo, String resolvedName, boolean active) {
    }

    public record RegistrationResult(UUID accountId, TokenService.IssuedToken token,
                                     SupervisorResolution supervisor) {
    }

    @Transactional
    public RegistrationResult register(RegisterCommand command) {
        String employeeNo = InputNormalizer.normalizeEmployeeNo(command.employeeNo());
        String name = InputNormalizer.normalizeDisplayText(command.name(), "姓名", 64);
        String department = InputNormalizer.normalizeDisplayText(command.department(), "部门", 128);
        String position = InputNormalizer.normalizeDisplayText(command.position(), "岗位", 128);

        // 一条语句同时完成「插入」与「冲突判定」。
        //
        // 刻意不用 try/catch DuplicateKeyException：在 PostgreSQL 里，语句失败后
        // 事务会进入 aborted 状态，catch 块中任何后续查询都会报
        // "current transaction is aborted"。ON CONFLICT DO NOTHING 返回影响行数，
        // 既能判冲突又不会污染事务。
        UUID accountId = UUID.randomUUID();
        if (accounts.insertIfAbsent(accountId, employeeNo, name, department, position) == 0) {
            throw new EmployeeNoTakenException(employeeNo);
        }

        // 注册只产生 MEMBER。这是规格的硬性要求，不因任何入参而改变。
        accounts.grantRole(accountId, Role.MEMBER, null);

        TokenService.IssuedToken token = tokens.issue(accountId, "注册签发");

        audit.record(accountId, "ACCOUNT", "ACCOUNT_REGISTERED", "account",
                accountId.toString(), Map.of("employeeNo", employeeNo));

        return new RegistrationResult(accountId, token, null);
    }
}
```

`supervisor` 解析在 Task 8 接入，本任务先返回 `null`。Task 8 会修改这个返回值并补测试。

- [ ] **Step 8: 实现 Web 层**

`RegistrationController.java`：

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.identity.RegistrationService;
import jakarta.validation.constraints.NotBlank;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RegistrationController {

    private final RegistrationService registrationService;

    public RegistrationController(RegistrationService registrationService) {
        this.registrationService = registrationService;
    }

    public record RegisterRequest(String employeeNo, String name, String department,
                                  String position, String supervisorEmployeeNo) {
    }

    public record RegisterResponse(String accountId, String token, String tokenPrefix,
                                   SupervisorView supervisor) {
        public record SupervisorView(String submittedEmployeeNo, String resolvedName, boolean active) {
        }
    }

    @PostMapping("/api/agent/register")
    @ResponseStatus(HttpStatus.CREATED)
    public RegisterResponse register(@RequestBody RegisterRequest request) {
        RegistrationService.RegistrationResult result = registrationService.register(
                new RegistrationService.RegisterCommand(
                        request.employeeNo(), request.name(), request.department(),
                        request.position(), request.supervisorEmployeeNo()));

        RegisterResponse.SupervisorView supervisor = result.supervisor() == null ? null
                : new RegisterResponse.SupervisorView(
                        result.supervisor().submittedEmployeeNo(),
                        result.supervisor().resolvedName(),
                        result.supervisor().active());

        return new RegisterResponse(
                result.accountId().toString(),
                result.token().plaintext(),
                result.token().prefix(),
                supervisor);
    }
}
```

`ApiExceptionHandler.java`：

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.identity.EmployeeNoTakenException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class ApiExceptionHandler {

    public record ErrorBody(String code, String message) {
    }

    @ExceptionHandler(EmployeeNoTakenException.class)
    public ProblemDetail onEmployeeNoTaken(EmployeeNoTakenException e) {
        return problem(HttpStatus.CONFLICT, "EMPLOYEE_NO_TAKEN", e.getMessage());
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ProblemDetail onIllegalArgument(IllegalArgumentException e) {
        return problem(HttpStatus.BAD_REQUEST, "INVALID_INPUT", e.getMessage());
    }

    private ProblemDetail problem(HttpStatus status, String code, String message) {
        ProblemDetail detail = ProblemDetail.forStatus(status);
        detail.setProperty("code", code);
        detail.setProperty("message", message);
        return detail;
    }
}
```

- [ ] **Step 9: 运行测试**

Run: `mvn -B test -Dtest=RegistrationServiceTest`
Expected: 5 个测试全部 PASS，其中并发测试恰好 1 个成功、7 个得到 `EmployeeNoTakenException`。

- [ ] **Step 10: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/ src/test/java/com/echoyan/a2ahub/identity/RegistrationServiceTest.java
git commit -m "feat(identity): 自助注册产出 MEMBER 角色与一次性令牌"
```

---

## Task 5: 令牌校验、轮换与 Bearer 认证过滤器

**Files:**
- Modify: `src/main/java/com/echoyan/a2ahub/identity/TokenService.java`
- Create: `src/main/java/com/echoyan/a2ahub/identity/TokenAuthFilter.java`
- Create: `src/main/java/com/echoyan/a2ahub/security/CurrentAccount.java`
- Create: `src/main/java/com/echoyan/a2ahub/security/CurrentAccountResolver.java`
- Test: `src/test/java/com/echoyan/a2ahub/identity/TokenServiceTest.java`

**Interfaces:**
- Consumes: `TokenService.issue`、`AccountRepository`（Task 4）
- Produces:
  - `Optional<CurrentAccount> TokenService.verify(String plaintext)` —— 格式非法、id 不存在、摘要不符、已撤销、账号非 ACTIVE 一律返回 `Optional.empty()`
  - `IssuedToken TokenService.rotate(UUID accountId, UUID oldTokenId, String label)`
  - `record CurrentAccount(UUID accountId, String employeeNo, String name, Set<Role> roles, boolean admin, boolean mustChangePassword)`
  - `CurrentAccountResolver.resolve(UUID accountId)` —— 每次调用都查库，**不缓存**

**这是 Review Focus 第 2 类。** id 不存在必须 401。

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.audit.AuditService;
import com.echoyan.a2ahub.security.CurrentAccount;
import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class TokenServiceTest extends IntegrationTestBase {

    @Autowired TokenService tokens;
    @Autowired RegistrationService registrations;
    @Autowired JdbcClient jdbc;

    private RegistrationService.RegistrationResult register(String no) {
        return registrations.register(new RegistrationService.RegisterCommand(
                no, "李四", "研发部", "工程师", null));
    }

    @Test
    void 合法令牌可校验出账号与角色() {
        var result = register("E3001");
        Optional<CurrentAccount> me = tokens.verify(result.token().plaintext());

        assertThat(me).isPresent();
        assertThat(me.get().accountId()).isEqualTo(result.accountId());
        assertThat(me.get().employeeNo()).isEqualTo("E3001");
        assertThat(me.get().roles()).containsExactly(Role.MEMBER);
    }

    @Test
    void id不存在时返回空而不是抛异常() {
        String bogus = "a2ah_" + UUID.randomUUID() + ".abcdefghijklmnop";
        assertThat(tokens.verify(bogus)).isEmpty();
    }

    @Test
    void 格式非法的令牌返回空() {
        assertThat(tokens.verify("not-a-token")).isEmpty();
        assertThat(tokens.verify("a2ah_nodot")).isEmpty();
        assertThat(tokens.verify("a2ah_" + UUID.randomUUID() + ".")).isEmpty();
        assertThat(tokens.verify("")).isEmpty();
        assertThat(tokens.verify(null)).isEmpty();
    }

    @Test
    void 摘要不符返回空() {
        var result = register("E3002");
        String wrong = result.token().plaintext().replaceAll("\\..*$", ".wrongsecret");
        assertThat(tokens.verify(wrong)).isEmpty();
    }

    @Test
    void 已撤销令牌返回空() {
        var result = register("E3003");
        jdbc.sql("UPDATE api_token SET revoked_at = now() WHERE account_id = :id")
                .param("id", result.accountId()).update();
        assertThat(tokens.verify(result.token().plaintext())).isEmpty();
    }

    @Test
    void 账号停用后令牌立即失效() {
        var result = register("E3004");
        assertThat(tokens.verify(result.token().plaintext())).isPresent();

        jdbc.sql("UPDATE account SET status = 'DISABLED' WHERE id = :id")
                .param("id", result.accountId()).update();

        assertThat(tokens.verify(result.token().plaintext())).isEmpty();
    }

    @Test
    void 轮换后旧令牌失效新令牌可用() {
        var result = register("E3005");
        UUID oldTokenId = UUID.fromString(
                result.token().plaintext().substring(5, result.token().plaintext().indexOf('.')));

        TokenService.IssuedToken fresh = tokens.rotate(result.accountId(), oldTokenId, "轮换");

        assertThat(tokens.verify(result.token().plaintext())).isEmpty();
        assertThat(tokens.verify(fresh.plaintext())).isPresent();
    }

    @Test
    void 校验更新最后使用时间() {
        var result = register("E3006");
        tokens.verify(result.token().plaintext());
        assertThat(jdbc.sql("SELECT last_used_at FROM api_token WHERE account_id = :id")
                .param("id", result.accountId()).query(java.time.OffsetDateTime.class).single())
                .isNotNull();
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=TokenServiceTest`
Expected: 编译失败，`verify` / `rotate` / `CurrentAccount` 不存在。

- [ ] **Step 3: 实现 `CurrentAccount` 与 `CurrentAccountResolver`**

```java
package com.echoyan.a2ahub.security;

import com.echoyan.a2ahub.identity.Role;

import java.util.Set;
import java.util.UUID;

public record CurrentAccount(
        UUID accountId,
        String employeeNo,
        String name,
        Set<Role> roles,
        boolean admin,
        boolean mustChangePassword) {

    public boolean hasRole(Role role) {
        return roles.contains(role);
    }
}
```

```java
package com.echoyan.a2ahub.security;

import com.echoyan.a2ahub.identity.Role;
import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Component;

import java.util.HashSet;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

/**
 * 每次调用都重新查库解析账号状态与角色。
 *
 * <p>刻意不缓存：缓存权限快照是「账号停用后仍能访问」这类缺陷的唯一来源。
 */
@Component
public class CurrentAccountResolver {

    private final JdbcClient jdbc;

    public CurrentAccountResolver(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }

    /**
     * 解析当前账号。
     *
     * <p>「停用即失效」被做成 SQL 条件而不是解析后的过滤器——这样它无法被
     * 后续的代码改动绕过，停用账号在结构上就不可能解析出 CurrentAccount。
     */
    public Optional<CurrentAccount> resolve(UUID accountId) {
        return jdbc.sql("""
                SELECT id, employee_no, name, is_admin, must_change_password
                  FROM account
                 WHERE id = :id AND status = 'ACTIVE'
                """)
            .param("id", accountId)
            .query((rs, rowNum) -> new CurrentAccount(
                    rs.getObject("id", UUID.class),
                    rs.getString("employee_no"),
                    rs.getString("name"),
                    loadRoles(accountId),
                    rs.getBoolean("is_admin"),
                    rs.getBoolean("must_change_password")))
            .optional();
    }

    /**
     * 读取角色。刻意独立于 resolve：停用账号的角色行仍然保留（历史事实），
     * 只是不再解析成可用的 CurrentAccount，因此角色也就不产生任何权威。
     */
    public Set<Role> loadRoles(UUID accountId) {
        return new HashSet<>(jdbc.sql("SELECT role FROM account_role WHERE account_id = :id")
                .param("id", accountId).query(String.class).list()
                .stream().map(Role::valueOf).toList());
    }
}
```

- [ ] **Step 4: 扩展 `TokenService`**

在 Task 4 的类中追加：

```java
    /** 校验令牌。任何一项不通过都返回空——不区分原因，避免给攻击者提供信息。 */
    public Optional<CurrentAccount> verify(String plaintext) {
        if (plaintext == null || !plaintext.startsWith("a2ah_")) {
            return Optional.empty();
        }
        int dot = plaintext.indexOf('.');
        if (dot < 0 || dot == plaintext.length() - 1) {
            return Optional.empty();
        }

        UUID tokenId;
        try {
            tokenId = UUID.fromString(plaintext.substring(5, dot));
        } catch (IllegalArgumentException e) {
            return Optional.empty();
        }
        String secret = plaintext.substring(dot + 1);

        Optional<UUID> accountId = jdbc.sql("""
                SELECT account_id FROM api_token
                 WHERE id = :id AND revoked_at IS NULL
                """)
            .param("id", tokenId)
            .query(UUID.class)
            .optional();

        if (accountId.isEmpty()) {
            return Optional.empty();
        }

        String storedDigest = jdbc.sql("SELECT digest FROM api_token WHERE id = :id")
                .param("id", tokenId).query(String.class).single();

        // 常数时间比较，避免时序侧信道
        if (!MessageDigest.isEqual(
                storedDigest.getBytes(StandardCharsets.UTF_8),
                sha256Hex(secret).getBytes(StandardCharsets.UTF_8))) {
            return Optional.empty();
        }

        jdbc.sql("UPDATE api_token SET last_used_at = now() WHERE id = :id")
                .param("id", tokenId).update();

        return resolver.resolve(accountId.get());
    }

    /** 撤销旧令牌并签发新令牌，同一事务内完成。 */
    @Transactional
    public IssuedToken rotate(UUID accountId, UUID oldTokenId, String label) {
        jdbc.sql("""
                UPDATE api_token SET revoked_at = now()
                 WHERE id = :id AND account_id = :acc AND revoked_at IS NULL
                """)
            .param("id", oldTokenId).param("acc", accountId).update();
        return issue(accountId, label);
    }
```

需要新增字段与 import：

```java
    private final CurrentAccountResolver resolver;

    public TokenService(JdbcClient jdbc, CurrentAccountResolver resolver) {
        this.jdbc = jdbc;
        this.resolver = resolver;
    }
```

以及新增 import：`com.echoyan.a2ahub.security.CurrentAccount`、`com.echoyan.a2ahub.security.CurrentAccountResolver`、`org.springframework.transaction.annotation.Transactional`、`java.util.Optional`。`MessageDigest` 来自 `java.security`，`StandardCharsets` 来自 `java.nio.charset`——两者在 Task 4 已经导入。

注意：`TokenService` 依赖 `CurrentAccountResolver`，而 `CurrentAccountResolver` 只依赖 `JdbcClient`——无循环依赖。

- [ ] **Step 5: 实现 `TokenAuthFilter`**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.security.CurrentAccount;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;
import java.util.Optional;

@Component
public class TokenAuthFilter extends OncePerRequestFilter {

    private static final String BEARER = "Bearer ";

    private final TokenService tokens;

    public TokenAuthFilter(TokenService tokens) {
        this.tokens = tokens;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith(BEARER)) {
            Optional<CurrentAccount> account = tokens.verify(header.substring(BEARER.length()).strip());
            account.ifPresent(a -> {
                var authorities = a.roles().stream()
                        .map(r -> new SimpleGrantedAuthority("ROLE_" + r.name()))
                        .toList();
                var auth = new UsernamePasswordAuthenticationToken(a, null, authorities);
                SecurityContextHolder.getContext().setAuthentication(auth);
            });
            // 校验失败不在此处返回 401：交给授权链决定，保证 permitAll 路径仍可访问
        }
        chain.doFilter(request, response);
    }
}
```

- [ ] **Step 6: 运行测试**

Run: `mvn -B test -Dtest=TokenServiceTest`
Expected: 8 个测试全部 PASS。

- [ ] **Step 7: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/identity/TokenService.java \
        src/main/java/com/echoyan/a2ahub/identity/TokenAuthFilter.java \
        src/main/java/com/echoyan/a2ahub/security/ \
        src/test/java/com/echoyan/a2ahub/identity/TokenServiceTest.java
git commit -m "feat(identity): 令牌校验、轮换与 Bearer 认证过滤器"
```

---

## Task 6: 安全过滤器链与 Origin 校验

**Files:**
- Create: `src/main/java/com/echoyan/a2ahub/config/AppProperties.java`
- Create: `src/main/java/com/echoyan/a2ahub/config/SecurityConfig.java`
- Create: `src/main/java/com/echoyan/a2ahub/web/OriginCheckFilter.java`
- Test: `src/test/java/com/echoyan/a2ahub/web/OriginCheckFilterTest.java`

**Interfaces:**
- Consumes: `TokenAuthFilter`（Task 5）
- Produces: `record AppProperties(String baseUrl, List<String> allowedOrigins, String adminPassword, String bootstrapDir)` 绑定 `a2ahub.*` 配置

**这是 Review Focus 第 5 类。** 缺 `Origin` 必须放行——非浏览器 Agent 不发这个头。

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import static org.assertj.core.api.Assertions.assertThat;

class OriginCheckFilterTest extends IntegrationTestBase {

    @Autowired TestRestTemplate rest;

    private ResponseEntity<String> postMcp(String origin) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("Content-Type", "application/json");
        headers.set("Accept", "application/json, text/event-stream");
        if (origin != null) {
            headers.set("Origin", origin);
        }
        return rest.exchange("/mcp", HttpMethod.POST,
                new HttpEntity<>("{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{}}", headers),
                String.class);
    }

    @Test
    void 缺少Origin头必须放行() {
        // 非浏览器 Agent 客户端不发 Origin。这条必须放行，否则 Level A 客户端全部失效。
        ResponseEntity<String> response = postMcp(null);
        assertThat(response.getStatusCode()).isNotIn(HttpStatus.FORBIDDEN, HttpStatus.BAD_REQUEST);
    }

    @Test
    void 允许列表内的Origin放行() {
        ResponseEntity<String> response = postMcp("http://localhost:8080");
        assertThat(response.getStatusCode()).isNotIn(HttpStatus.FORBIDDEN, HttpStatus.BAD_REQUEST);
    }

    @Test
    void 不在允许列表的Origin被拒绝() {
        ResponseEntity<String> response = postMcp("https://evil.example.com");
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
    }

    @Test
    void 注册端点无需令牌() {
        HttpHeaders headers = new HttpHeaders();
        headers.set("Content-Type", "application/json");
        ResponseEntity<String> response = rest.exchange("/api/agent/register", HttpMethod.POST,
                new HttpEntity<>("""
                        {"employeeNo":"E4001","name":"王五","department":"研发部","position":"工程师"}
                        """, headers),
                String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }

    @Test
    void 受保护端点无令牌返回401() {
        ResponseEntity<String> response = rest.getForEntity("/api/agent/protected-probe", String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);
    }
}
```

`/api/agent/protected-probe` 需要一个真实存在的受保护端点。本任务加一个最小探针控制器：

```java
package com.echoyan.a2ahub.web;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/** 仅用于验证授权链的探针端点。Task 8 的 MCP 身份端点落地后可删除。 */
@RestController
public class AuthorizationProbeController {

    @GetMapping("/api/agent/protected-probe")
    public String probe() {
        return "ok";
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=OriginCheckFilterTest`
Expected: 失败——无安全配置时 `/mcp` 与探针端点行为不符合预期。

- [ ] **Step 3: 实现 `AppProperties`**

```java
package com.echoyan.a2ahub.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;

import java.util.List;

@ConfigurationProperties(prefix = "a2ahub")
public record AppProperties(
        @DefaultValue("http://localhost:8080") String baseUrl,
        @DefaultValue("http://localhost:8080") List<String> allowedOrigins,
        @DefaultValue("") String adminPassword,
        @DefaultValue("./data/bootstrap") String bootstrapDir) {
}
```

在 `A2AHubApplication` 上加 `@ConfigurationPropertiesScan`：

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class A2AHubApplication {
    // main 不变
}
```

- [ ] **Step 4: 实现 `OriginCheckFilter`**

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.config.AppProperties;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

/**
 * MCP Streamable HTTP 规范要求校验 Origin 头（MUST）。
 *
 * <p>刻意区分「缺失」与「不匹配」：浏览器一定发 Origin，非浏览器的 Agent 客户端
 * 一定不发。缺失必须放行，否则所有 Agent 客户端都会被挡在门外，而这并不能带来
 * 任何安全收益——攻击者可以直接不发这个头。真正被拦下的是「浏览器被诱导发出的
 * 跨站请求」，那种请求一定带 Origin。
 */
@Component
public class OriginCheckFilter extends OncePerRequestFilter {

    private final List<String> allowedOrigins;

    public OriginCheckFilter(AppProperties properties) {
        this.allowedOrigins = properties.allowedOrigins();
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return !request.getRequestURI().startsWith("/mcp");
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String origin = request.getHeader("Origin");
        if (origin != null && !allowedOrigins.contains(origin)) {
            response.setStatus(HttpStatus.FORBIDDEN.value());
            response.setContentType("application/json");
            response.getWriter().write("{\"code\":\"ORIGIN_NOT_ALLOWED\",\"message\":\"Origin 不被允许\"}");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

- [ ] **Step 5: 实现 `SecurityConfig`**

```java
package com.echoyan.a2ahub.config;

import com.echoyan.a2ahub.identity.TokenAuthFilter;
import com.echoyan.a2ahub.web.OriginCheckFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    /** 面向 Agent：无状态、不透明令牌、无 Cookie，因此关闭 CSRF。 */
    @Bean
    @Order(1)
    SecurityFilterChain agentChain(HttpSecurity http, TokenAuthFilter tokenAuthFilter,
                                   OriginCheckFilter originCheckFilter) throws Exception {
        http.securityMatcher("/mcp/**", "/api/agent/**")
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/agent/register").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(originCheckFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(tokenAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    /** 面向浏览器：会话 + CSRF。Task 10 接入登录页后补齐。 */
    @Bean
    @Order(2)
    SecurityFilterChain browserChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/actuator/health/**").permitAll()
                .anyRequest().authenticated())
            .formLogin(form -> form.permitAll());
        return http.build();
    }
}
```

- [ ] **Step 6: 运行测试**

Run: `mvn -B test -Dtest=OriginCheckFilterTest`
Expected: 5 个测试全部 PASS。

- [ ] **Step 7: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/config/ src/main/java/com/echoyan/a2ahub/web/ \
        src/test/java/com/echoyan/a2ahub/web/OriginCheckFilterTest.java
git commit -m "feat(security): 两条过滤器链与 MCP Origin 校验"
```

---

## Task 7: MCP 服务端与每请求身份解析（含技术验证）

**Files:**
- Create: `src/main/java/com/echoyan/a2ahub/security/McpIdentity.java`
- Create: `src/main/java/com/echoyan/a2ahub/identity/IdentityTools.java`
- Test: `src/test/java/com/echoyan/a2ahub/mcp/McpIdentityTest.java`

**Interfaces:**
- Consumes: `TokenAuthFilter`（Task 5）、过滤器链（Task 6）
- Produces:
  - `McpIdentity.current()` 返回 `CurrentAccount`，未认证时抛 `McpIdentity.NotAuthenticatedException`
  - MCP 工具 `identity_whoami` 返回 `WhoAmI(employeeNo, name, roles, admin, mustChangePassword)`

**本任务以一次技术验证开头。** 规格 §10.1 要求「已实测」而非假设。在写任何实现之前，必须先证明 `SecurityContextHolder` 在 `@McpTool` 方法体内确实被填充——MCP 服务端可能把工具调用派发到与 Servlet 请求不同的线程，那样 `SecurityContextHolder` 就是空的。

- [ ] **Step 1: 验证 `SecurityContextHolder` 在 `@McpTool` 中可见**

先写一个最小工具与一个直连测试：

```java
package com.echoyan.a2ahub.mcp;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

@Component
public class SecurityContextSpike {

    @Tool(name = "spike_whoami", description = "技术验证：返回当前认证主体，未认证时返回 ANONYMOUS")
    public String spikeWhoami() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            return "ANONYMOUS";
        }
        return auth.getPrincipal().getClass().getSimpleName();
    }
}
```

Run: `mvn -B test -Dtest=McpIdentityTest#spike`

用一个测试向 `/mcp` 发 `tools/call` 并带上真实令牌，断言返回 `CurrentAccount`。

**判定：**

- **返回 `CurrentAccount`** → 走 Step 2 的方案 A（`SecurityContextHolder`），继续。
- **返回 `ANONYMOUS`** → MCP 派发到了别的线程，走方案 B：实现 `TransportContextExtractor<CurrentAccount>`，在 `McpTransportContext` 中携带账号，工具方法加 `McpTransportContext` 参数读取。方案 B 的实现细节取决于 Spring AI 2.0.1 的实际 API——**先 `mvn dependency:tree` 找到 `spring-ai-mcp` jar，用 `javap` 确认 `TransportContextExtractor` 与 `McpTransportContext` 的真实签名，不要凭记忆写。**
- **两者都不可行** → 停下来报告。这意味着 MCP 身份方案需要重新设计，属于影响产品方向的变更。

**方案 B 若被采用，本任务后续所有 `McpIdentity.current()` 的实现随之改变，但接口签名不变**，后续任务不受影响。

- [ ] **Step 2: 实现 `McpIdentity`**

```java
package com.echoyan.a2ahub.security;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

@Component
public class McpIdentity {

    public static class NotAuthenticatedException extends RuntimeException {
        public NotAuthenticatedException() {
            super("未认证");
        }
    }

    public CurrentAccount current() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !(auth.getPrincipal() instanceof CurrentAccount account)) {
            throw new NotAuthenticatedException();
        }
        return account;
    }
}
```

- [ ] **Step 3: 实现 `IdentityTools`**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.security.CurrentAccount;
import com.echoyan.a2ahub.security.McpIdentity;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class IdentityTools {

    private final McpIdentity identity;

    public IdentityTools(McpIdentity identity) {
        this.identity = identity;
    }

    public record WhoAmI(String employeeNo, String name, List<String> roles,
                         boolean admin, boolean mustChangePassword) {
    }

    @Tool(name = "identity_whoami", description = "返回当前账号的工号、姓名、角色与是否待改密")
    public WhoAmI whoami() {
        CurrentAccount me = identity.current();
        return new WhoAmI(
                me.employeeNo(),
                me.name(),
                me.roles().stream().map(Enum::name).sorted().toList(),
                me.admin(),
                me.mustChangePassword());
    }
}
```

- [ ] **Step 4: 写直连 MCP 测试**

本测试用原始 JSON-RPC 驱动 `/mcp`，不依赖任何第三方客户端——这是规格 §10.1「Level A 已实测」的实际落点。

```java
package com.echoyan.a2ahub.mcp;

import com.echoyan.a2ahub.identity.RegistrationService;
import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import tools.jackson.databind.JsonNode;
import tools.jackson.databind.ObjectMapper;

import static org.assertj.core.api.Assertions.assertThat;

class McpIdentityTest extends IntegrationTestBase {

    @Autowired TestRestTemplate rest;
    @Autowired RegistrationService registrations;
    @Autowired ObjectMapper objectMapper;

    private HttpHeaders mcpHeaders(String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Accept", "application/json, text/event-stream");
        if (token != null) {
            headers.setBearerAuth(token);
        }
        return headers;
    }

    private String call(String token, String body) {
        ResponseEntity<String> response = rest.exchange("/mcp", HttpMethod.POST,
                new HttpEntity<>(body, mcpHeaders(token)), String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        return response.getBody();
    }

    private JsonNode parse(String body) {
        // Streamable HTTP 可能返回 SSE 帧，取 data: 行
        String json = body.lines()
                .filter(line -> line.startsWith("data:"))
                .map(line -> line.substring(5).strip())
                .findFirst()
                .orElse(body);
        return objectMapper.readTree(json);
    }

    private String initialize(String token) {
        String body = call(token, """
                {"jsonrpc":"2.0","id":1,"method":"initialize","params":{
                  "protocolVersion":"2025-11-25",
                  "capabilities":{},
                  "clientInfo":{"name":"a2ahub-test","version":"1.0"}}}
                """);
        return parse(body).path("result").path("sessionId").asString("");
    }

    @Test
    void 带令牌调用whoami返回当前账号() {
        var reg = registrations.register(new RegistrationService.RegisterCommand(
                "E5001", "赵六", "研发部", "工程师", null));
        String token = reg.token().plaintext();

        initialize(token);
        String response = call(token, """
                {"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
                  "name":"identity_whoami","arguments":{}}}
                """);

        JsonNode result = parse(response).path("result");
        String text = result.path("content").get(0).path("text").asString();
        assertThat(text).contains("E5001").contains("MEMBER");
    }

    @Test
    void 无令牌调用whoami被拒绝() {
        ResponseEntity<String> response = rest.exchange("/mcp", HttpMethod.POST,
                new HttpEntity<>("""
                        {"jsonrpc":"2.0","id":1,"method":"initialize","params":{
                          "protocolVersion":"2025-11-25","capabilities":{},
                          "clientInfo":{"name":"t","version":"1"}}}
                        """, mcpHeaders(null)), String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);
    }
}
```

`tools/call` 的确切请求/响应形状以 Step 1 的实际观测为准——**如果实际返回与上面的断言不符，改测试去匹配真实行为，不要改断言去迁就假设**。把观测到的真实报文记录进 `docs/superpowers/notes/mcp-wire-format.md`（新建），它是 §10.4 要求的证据。

- [ ] **Step 5: 运行测试**

Run: `mvn -B test -Dtest=McpIdentityTest`
Expected: 2 个测试 PASS。

- [ ] **Step 6: 删除验证用的 spike**

删除 `SecurityContextSpike.java` 与其测试方法。技术验证的产出是**结论**（记入 `mcp-wire-format.md`），不是留在仓库里的临时代码。

- [ ] **Step 7: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/security/McpIdentity.java \
        src/main/java/com/echoyan/a2ahub/identity/IdentityTools.java \
        src/test/java/com/echoyan/a2ahub/mcp/ docs/superpowers/notes/
git commit -m "feat(mcp): 身份解析与 identity_whoami，附实测线格式记录"
```

---

## Task 8: 上级链接建立、激活与环路检测

**Files:**
- Create: `src/main/java/com/echoyan/a2ahub/identity/SupervisorLinkRepository.java`
- Create: `src/main/java/com/echoyan/a2ahub/identity/SupervisorLinkService.java`
- Modify: `src/main/java/com/echoyan/a2ahub/identity/RegistrationService.java`
- Test: `src/test/java/com/echoyan/a2ahub/identity/SupervisorLinkServiceTest.java`

**Interfaces:**
- Consumes: `AccountRepository`（Task 4）、`AuditService`（Task 4）
- Produces:
  - `SupervisorResolution SupervisorLinkService.createOnRegistration(UUID subordinateId, String supervisorEmployeeNo)`
  - `void SupervisorLinkService.activatePendingLinksFor(String newAccountEmployeeNo)` —— 新账号注册后调用
  - `Optional<UUID> SupervisorLinkService.resolveSupervisor(UUID subordinateId)` —— 仅当链接为 `ACTIVE` 且上级账号 `ACTIVE` 时返回

**这是 Review Focus 第 3 类。** 规格只覆盖了「尚未注册」和「自环/环路」，没覆盖「目标存在但已停用」。

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SupervisorLinkServiceTest extends IntegrationTestBase {

    @Autowired RegistrationService registrations;
    @Autowired SupervisorLinkService links;
    @Autowired JdbcClient jdbc;

    private RegistrationService.RegistrationResult reg(String no, String supervisor) {
        return registrations.register(new RegistrationService.RegisterCommand(
                no, "员工" + no, "研发部", "工程师", supervisor));
    }

    @Test
    void 上级已存在时链接立即生效() {
        var boss = reg("E6001", null);
        var subordinate = reg("E6002", "E6001");

        assertThat(subordinate.supervisor().active()).isTrue();
        assertThat(links.resolveSupervisor(subordinate.accountId())).contains(boss.accountId());
    }

    @Test
    void 上级尚未注册时链接为待激活() {
        var subordinate = reg("E6003", "E6999");

        assertThat(subordinate.supervisor().active()).isFalse();
        assertThat(subordinate.supervisor().submittedEmployeeNo()).isEqualTo("E6999");
        assertThat(links.resolveSupervisor(subordinate.accountId())).isEmpty();
    }

    @Test
    void 上级注册后待激活链接自动生效() {
        var subordinate = reg("E6004", "E6005");
        assertThat(links.resolveSupervisor(subordinate.accountId())).isEmpty();

        var boss = reg("E6005", null);

        assertThat(links.resolveSupervisor(subordinate.accountId())).contains(boss.accountId());
    }

    @Test
    void 自环被拒绝() {
        assertThatThrownBy(() -> reg("E6006", "E6006"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("自己");
    }

    @Test
    void 直接环路被拒绝() {
        // E6007 先认尚未注册的 E6008 为上级（链接 PENDING，不构成有效边）
        var a = reg("E6007", "E6008");
        // E6008 注册时认 E6007 为上级 → E6008 → E6007 成为有效边
        var b = reg("E6008", "E6007");
        assertThat(links.resolveSupervisor(b.accountId())).contains(a.accountId());

        // 此时再让 E6007 认 E6008 为上级，就形成 E6007 → E6008 → E6007
        assertThatThrownBy(() -> links.repoint(a.accountId(), "E6008"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("环");
    }

    @Test
    void 上级账号被停用后不产生可用权威() {
        var boss = reg("E6009", null);
        var subordinate = reg("E6010", "E6009");
        assertThat(links.resolveSupervisor(subordinate.accountId())).contains(boss.accountId());

        jdbc.sql("UPDATE account SET status = 'DISABLED' WHERE id = :id")
                .param("id", boss.accountId()).update();

        // 规格未覆盖这种情况：链接行仍是 ACTIVE，但停用账号不能充当可用上级
        assertThat(links.resolveSupervisor(subordinate.accountId())).isEmpty();
    }

    @Test
    void 链接激活写入审计() {
        reg("E6011", "E6012");
        Integer before = jdbc.sql("SELECT count(*) FROM audit_log WHERE action = 'SUPERVISOR_LINK_ACTIVATED'")
                .query(Integer.class).single();

        reg("E6012", null);

        Integer after = jdbc.sql("SELECT count(*) FROM audit_log WHERE action = 'SUPERVISOR_LINK_ACTIVATED'")
                .query(Integer.class).single();
        assertThat(after).isEqualTo(before + 1);
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=SupervisorLinkServiceTest`
Expected: 编译失败，`SupervisorLinkService` 不存在。

- [ ] **Step 3: 实现 `SupervisorLinkRepository`**

```java
package com.echoyan.a2ahub.identity;

import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Repository
public class SupervisorLinkRepository {

    private final JdbcClient jdbc;

    public SupervisorLinkRepository(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }

    public record Link(UUID id, UUID subordinateAccountId, String supervisorEmployeeNo,
                       UUID supervisorAccountId, String status) {
    }

    public void insert(UUID subordinateId, String supervisorEmployeeNo, UUID supervisorAccountId,
                       String status) {
        jdbc.sql("""
                INSERT INTO supervisor_link
                    (id, subordinate_account_id, supervisor_employee_no, supervisor_account_id, status,
                     activated_at)
                VALUES (:id, :sub, :no, :sup, :status,
                        CASE WHEN :status = 'ACTIVE' THEN now() ELSE NULL END)
                """)
            .param("id", UUID.randomUUID())
            .param("sub", subordinateId)
            .param("no", supervisorEmployeeNo)
            .param("sup", supervisorAccountId)
            .param("status", status)
            .update();
    }

    public Optional<Link> findBySubordinate(UUID subordinateId) {
        return jdbc.sql("""
                SELECT id, subordinate_account_id, supervisor_employee_no, supervisor_account_id, status
                  FROM supervisor_link WHERE subordinate_account_id = :id
                """)
            .param("id", subordinateId)
            .query((rs, n) -> new Link(
                    rs.getObject("id", UUID.class),
                    rs.getObject("subordinate_account_id", UUID.class),
                    rs.getString("supervisor_employee_no"),
                    rs.getObject("supervisor_account_id", UUID.class),
                    rs.getString("status")))
            .optional();
    }

    /** 找出所有把这个工号当上级、且尚未激活的链接。 */
    public List<Link> findPendingBySupervisorEmployeeNo(String supervisorEmployeeNo) {
        return jdbc.sql("""
                SELECT id, subordinate_account_id, supervisor_employee_no, supervisor_account_id, status
                  FROM supervisor_link
                 WHERE supervisor_employee_no = :no AND status = 'PENDING'
                """)
            .param("no", supervisorEmployeeNo)
            .query((rs, n) -> new Link(
                    rs.getObject("id", UUID.class),
                    rs.getObject("subordinate_account_id", UUID.class),
                    rs.getString("supervisor_employee_no"),
                    rs.getObject("supervisor_account_id", UUID.class),
                    rs.getString("status")))
            .list();
    }

    public void markActive(UUID linkId, UUID supervisorAccountId) {
        jdbc.sql("""
                UPDATE supervisor_link
                   SET supervisor_account_id = :sup, status = 'ACTIVE', activated_at = now()
                 WHERE id = :id AND status = 'PENDING'
                """)
            .param("sup", supervisorAccountId).param("id", linkId).update();
    }

    public void markRejected(UUID linkId) {
        jdbc.sql("UPDATE supervisor_link SET status = 'REJECTED' WHERE id = :id")
                .param("id", linkId).update();
    }

    public void updateTarget(UUID subordinateId, String supervisorEmployeeNo,
                             UUID supervisorAccountId, String status) {
        jdbc.sql("""
                UPDATE supervisor_link
                   SET supervisor_employee_no = :no, supervisor_account_id = :sup, status = :status
                 WHERE subordinate_account_id = :sub
                """)
            .param("no", supervisorEmployeeNo).param("sup", supervisorAccountId)
            .param("status", status).param("sub", subordinateId).update();
    }

    /** 返回给定账号的所有下级账号 id（沿 ACTIVE 链接向上追溯用不到，这里只走一跳）。 */
    public Optional<UUID> findSupervisorAccountId(UUID subordinateId) {
        return jdbc.sql("""
                SELECT supervisor_account_id FROM supervisor_link
                 WHERE subordinate_account_id = :id AND status = 'ACTIVE'
                """)
            .param("id", subordinateId).query(UUID.class).optional();
    }
}
```

- [ ] **Step 4: 实现 `SupervisorLinkService`**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.audit.AuditService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashSet;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

@Service
public class SupervisorLinkService {

    private static final int MAX_DEPTH = 64;

    private final SupervisorLinkRepository links;
    private final AccountRepository accounts;
    private final AuditService audit;

    public SupervisorLinkService(SupervisorLinkRepository links, AccountRepository accounts,
                                 AuditService audit) {
        this.links = links;
        this.accounts = accounts;
        this.audit = audit;
    }

    /** 注册时建立链接。上级已存在则立即 ACTIVE，否则 PENDING。 */
    @Transactional
    public RegistrationService.SupervisorResolution createOnRegistration(
            UUID subordinateId, String rawSupervisorEmployeeNo) {
        if (rawSupervisorEmployeeNo == null || rawSupervisorEmployeeNo.isBlank()) {
            return null;
        }
        String supervisorNo = InputNormalizer.normalizeEmployeeNo(rawSupervisorEmployeeNo);

        Optional<UUID> supervisorId = accounts.findIdByEmployeeNo(supervisorNo);
        if (supervisorId.isPresent() && supervisorId.get().equals(subordinateId)) {
            throw new IllegalArgumentException("上级不能是自己");
        }
        if (supervisorId.isPresent()) {
            assertNoCycle(subordinateId, supervisorId.get());
        }

        String status = supervisorId.isPresent() ? "ACTIVE" : "PENDING";
        links.insert(subordinateId, supervisorNo, supervisorId.orElse(null), status);

        String resolvedName = supervisorId
                .map(id -> accounts.findNameById(id).orElse(null))
                .orElse(null);

        if (supervisorId.isPresent()) {
            audit.record(subordinateId, "ACCOUNT", "SUPERVISOR_LINK_ACTIVATED", "account",
                    subordinateId.toString(),
                    Map.of("supervisorEmployeeNo", supervisorNo));
        }

        return new RegistrationService.SupervisorResolution(supervisorNo, resolvedName,
                supervisorId.isPresent());
    }

    /** 新账号注册后调用：把等待这个工号的链接激活。 */
    @Transactional
    public void activatePendingLinksFor(UUID newAccountId, String newAccountEmployeeNo) {
        for (SupervisorLinkRepository.Link link : links.findPendingBySupervisorEmployeeNo(newAccountEmployeeNo)) {
            if (link.subordinateAccountId().equals(newAccountId)) {
                links.markRejected(link.id());
                continue;
            }
            try {
                assertNoCycle(link.subordinateAccountId(), newAccountId);
            } catch (IllegalArgumentException e) {
                // 激活时会形成环 —— 拒绝并留痕，不静默丢弃
                links.markRejected(link.id());
                audit.record(newAccountId, "SYSTEM", "SUPERVISOR_LINK_REJECTED_CYCLE", "account",
                        link.subordinateAccountId().toString(),
                        Map.of("supervisorEmployeeNo", newAccountEmployeeNo));
                continue;
            }
            links.markActive(link.id(), newAccountId);
            audit.record(newAccountId, "SYSTEM", "SUPERVISOR_LINK_ACTIVATED", "account",
                    link.subordinateAccountId().toString(),
                    Map.of("supervisorEmployeeNo", newAccountEmployeeNo));
        }
    }

    /**
     * 解析可用上级：链接必须是 ACTIVE，且上级账号必须仍是 ACTIVE。
     *
     * <p>规格只覆盖了「尚未注册」与「自环/环路」，没有覆盖「上级账号已停用」。
     * 停用账号不得充当可用权威 —— 这正是本方法第二个条件存在的原因。
     */
    public Optional<UUID> resolveSupervisor(UUID subordinateId) {
        return links.findSupervisorAccountId(subordinateId)
                .filter(accounts::isActive);
    }

    /** 管理员更正上级时使用。会重新做自环与环路检测。 */
    @Transactional
    public void repoint(UUID subordinateId, String rawSupervisorEmployeeNo) {
        String supervisorNo = InputNormalizer.normalizeEmployeeNo(rawSupervisorEmployeeNo);
        Optional<UUID> supervisorId = accounts.findIdByEmployeeNo(supervisorNo);

        if (supervisorId.isPresent() && supervisorId.get().equals(subordinateId)) {
            throw new IllegalArgumentException("上级不能是自己");
        }
        if (supervisorId.isPresent()) {
            assertNoCycle(subordinateId, supervisorId.get());
        }

        links.updateTarget(subordinateId, supervisorNo, supervisorId.orElse(null),
                supervisorId.isPresent() ? "ACTIVE" : "PENDING");

        audit.record(subordinateId, "ADMIN", "SUPERVISOR_LINK_REPOINTED", "account",
                subordinateId.toString(), Map.of("supervisorEmployeeNo", supervisorNo));
    }

    /** 从候选人出发向上追溯；若能回到起点则成环。 */
    private void assertNoCycle(UUID subordinateId, UUID candidateSupervisorId) {
        Set<UUID> seen = new HashSet<>();
        UUID cursor = candidateSupervisorId;

        for (int depth = 0; depth < MAX_DEPTH; depth++) {
            if (cursor == null) {
                return;
            }
            if (cursor.equals(subordinateId)) {
                throw new IllegalArgumentException("上级关系会形成环");
            }
            if (!seen.add(cursor)) {
                return; // 上游已有环，但不是本次引入的；不阻断
            }
            cursor = links.findSupervisorAccountId(cursor).orElse(null);
        }
        throw new IllegalArgumentException("上级层级超过 " + MAX_DEPTH + " 层，疑似存在环");
    }
}
```

`AccountRepository` 需要新增两个方法：

```java
    public Optional<String> findNameById(UUID accountId) {
        return jdbc.sql("SELECT name FROM account WHERE id = :id")
                .param("id", accountId).query(String.class).optional();
    }

    /** 账号是否为 ACTIVE。用于「停用账号不得充当权威」的判定。 */
    public boolean isActive(UUID accountId) {
        return jdbc.sql("SELECT status FROM account WHERE id = :id")
                .param("id", accountId).query(String.class).optional()
                .map("ACTIVE"::equals).orElse(false);
    }
```

- [ ] **Step 5: 把上级解析接进注册流程**

修改 `RegistrationService.register`，在 `grantRole` 之后：

```java
        // 建立上级链接（若填写了上级工号）
        SupervisorResolution supervisor =
                supervisorLinks.createOnRegistration(accountId, command.supervisorEmployeeNo());

        // 本账号的注册可能激活等待这个工号的链接
        supervisorLinks.activatePendingLinksFor(accountId, employeeNo);
```

并修改返回值为 `new RegistrationResult(accountId, token, supervisor)`，同时把 `supervisorLinks` 加进构造器。

`RegistrationService` 需要新字段：

```java
    private final SupervisorLinkService supervisorLinks;

    public RegistrationService(AccountRepository accounts, TokenService tokens, AuditService audit,
                               SupervisorLinkService supervisorLinks) {
        this.accounts = accounts;
        this.tokens = tokens;
        this.audit = audit;
        this.supervisorLinks = supervisorLinks;
    }
```

**核对信息必须回传给调用方**：`RegistrationResult.supervisor()` 已经携带了 `submittedEmployeeNo` / `resolvedName` / `active`，`RegistrationController` 已在 Task 4 中把它们放进响应体。

- [ ] **Step 6: 运行测试**

Run: `mvn -B test -Dtest=SupervisorLinkServiceTest`
Expected: 7 个测试全部 PASS。

再跑一次全量，确认 Task 4 的注册测试没有被破坏：

Run: `mvn -B test -Dtest='RegistrationServiceTest,TokenServiceTest,SupervisorLinkServiceTest'`
Expected: 全部 PASS。

- [ ] **Step 7: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/identity/ \
        src/test/java/com/echoyan/a2ahub/identity/SupervisorLinkServiceTest.java
git commit -m "feat(identity): 上级链接建立、激活、环路与停用检测"
```

---

## Task 9: 角色申请与管理员批准

**Files:**
- Create: `src/main/java/com/echoyan/a2ahub/identity/RoleGrantRepository.java`
- Create: `src/main/java/com/echoyan/a2ahub/identity/RoleGrantService.java`
- Modify: `src/main/java/com/echoyan/a2ahub/identity/IdentityTools.java`
- Create: `src/main/java/com/echoyan/a2ahub/web/AdminController.java`
- Test: `src/test/java/com/echoyan/a2ahub/identity/RoleGrantServiceTest.java`

**Interfaces:**
- Consumes: `AccountRepository`（Task 4）、`AuditService`（Task 4）、`McpIdentity`（Task 7）
- Produces:
  - `UUID RoleGrantService.request(UUID accountId, Role role, String reason)`
  - `void RoleGrantService.approve(UUID requestId, UUID adminAccountId, String note)`
  - `void RoleGrantService.reject(UUID requestId, UUID adminAccountId, String note)`
  - `List<PendingGrant> RoleGrantService.pending()`
  - MCP 工具 `identity_request_role(role, reason)`
  - HTTP `GET /api/admin/role-requests`、`POST /api/admin/role-requests/{id}/approve`、`POST /api/admin/role-requests/{id}/reject`

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RoleGrantServiceTest extends IntegrationTestBase {

    @Autowired RegistrationService registrations;
    @Autowired RoleGrantService grants;
    @Autowired JdbcClient jdbc;

    private UUID register(String no) {
        return registrations.register(new RegistrationService.RegisterCommand(
                no, "员工" + no, "研发部", "工程师", null)).accountId();
    }

    private UUID registerAdmin(String no) {
        UUID id = register(no);
        jdbc.sql("UPDATE account SET is_admin = true WHERE id = :id").param("id", id).update();
        return id;
    }

    @Test
    void 申请后角色不立即生效() {
        UUID applicant = register("E7001");
        grants.request(applicant, Role.OWNER, "我是负责人");

        assertThat(jdbc.sql("SELECT count(*) FROM account_role WHERE account_id = :id AND role = 'OWNER'")
                .param("id", applicant).query(Integer.class).single()).isZero();
    }

    @Test
    void 批准后角色生效() {
        UUID applicant = register("E7002");
        UUID admin = registerAdmin("E7003");

        UUID requestId = grants.request(applicant, Role.OWNER, "我是负责人");
        grants.approve(requestId, admin, "确认");

        assertThat(jdbc.sql("SELECT count(*) FROM account_role WHERE account_id = :id AND role = 'OWNER'")
                .param("id", applicant).query(Integer.class).single()).isEqualTo(1);
    }

    @Test
    void 批准写入审计且记录批准人() {
        UUID applicant = register("E7004");
        UUID admin = registerAdmin("E7005");
        UUID requestId = grants.request(applicant, Role.REVIEWER, "我带团队");

        grants.approve(requestId, admin, "确认");

        var row = jdbc.sql("""
                SELECT status, decided_by_account_id FROM role_grant_request WHERE id = :id
                """).param("id", requestId)
                .query((rs, n) -> new Object[]{rs.getString(1), rs.getObject(2, UUID.class)})
                .single();
        assertThat(row[0]).isEqualTo("APPROVED");
        assertThat(row[1]).isEqualTo(admin);

        assertThat(jdbc.sql("""
                SELECT count(*) FROM audit_log
                 WHERE action = 'ROLE_GRANT_APPROVED' AND actor_account_id = :admin
                """).param("admin", admin).query(Integer.class).single()).isEqualTo(1);
    }

    @Test
    void 拒绝后角色不生效() {
        UUID applicant = register("E7006");
        UUID admin = registerAdmin("E7007");
        UUID requestId = grants.request(applicant, Role.OWNER, "我是负责人");

        grants.reject(requestId, admin, "工号核对不上");

        assertThat(jdbc.sql("SELECT count(*) FROM account_role WHERE account_id = :id AND role = 'OWNER'")
                .param("id", applicant).query(Integer.class).single()).isZero();
    }

    @Test
    void 非管理员不能批准() {
        UUID applicant = register("E7008");
        UUID notAdmin = register("E7009");
        UUID requestId = grants.request(applicant, Role.OWNER, "我是负责人");

        assertThatThrownBy(() -> grants.approve(requestId, notAdmin, "我说了算"))
                .isInstanceOf(RoleGrantService.NotAuthorizedException.class);

        assertThat(jdbc.sql("SELECT count(*) FROM account_role WHERE account_id = :id AND role = 'OWNER'")
                .param("id", applicant).query(Integer.class).single()).isZero();
    }

    @Test
    void 已批准的申请不能重复批准() {
        UUID applicant = register("E7010");
        UUID admin = registerAdmin("E7011");
        UUID requestId = grants.request(applicant, Role.OWNER, "我是负责人");
        grants.approve(requestId, admin, "确认");

        assertThatThrownBy(() -> grants.approve(requestId, admin, "再批一次"))
                .isInstanceOf(RoleGrantService.AlreadyDecidedException.class);
    }

    @Test
    void 重复申请被拒绝() {
        UUID applicant = register("E7012");
        grants.request(applicant, Role.OWNER, "第一次");

        assertThatThrownBy(() -> grants.request(applicant, Role.OWNER, "第二次"))
                .isInstanceOf(RoleGrantService.DuplicateRequestException.class);
    }

    @Test
    void 不能申请MEMBER角色() {
        UUID applicant = register("E7013");
        assertThatThrownBy(() -> grants.request(applicant, Role.MEMBER, "我想要"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void 停用账号不能申请角色() {
        UUID applicant = register("E7014");
        jdbc.sql("UPDATE account SET status = 'DISABLED' WHERE id = :id")
                .param("id", applicant).update();

        assertThatThrownBy(() -> grants.request(applicant, Role.OWNER, "我是负责人"))
                .isInstanceOf(RoleGrantService.NotAuthorizedException.class);
    }

    @Test
    void 停用账号不能批准角色() {
        UUID applicant = register("E7015");
        UUID admin = registerAdmin("E7016");
        UUID requestId = grants.request(applicant, Role.OWNER, "我是负责人");

        jdbc.sql("UPDATE account SET status = 'DISABLED' WHERE id = :id").param("id", admin).update();

        assertThatThrownBy(() -> grants.approve(requestId, admin, "确认"))
                .isInstanceOf(RoleGrantService.NotAuthorizedException.class);
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=RoleGrantServiceTest`
Expected: 编译失败。

- [ ] **Step 3: 实现 `RoleGrantRepository`**

```java
package com.echoyan.a2ahub.identity;

import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Repository
public class RoleGrantRepository {

    private final JdbcClient jdbc;

    public RoleGrantRepository(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }

    public record Row(UUID id, UUID accountId, String employeeNo, String name,
                      String requestedRole, String reason, String status) {
    }

    public void insert(UUID id, UUID accountId, Role role, String reason) {
        jdbc.sql("""
                INSERT INTO role_grant_request (id, account_id, requested_role, reason)
                VALUES (:id, :acc, :role, :reason)
                """)
            .param("id", id).param("acc", accountId).param("role", role.name())
            .param("reason", reason).update();
    }

    public Optional<Row> findById(UUID id) {
        return jdbc.sql("""
                SELECT r.id, r.account_id, a.employee_no, a.name, r.requested_role, r.reason, r.status
                  FROM role_grant_request r JOIN account a ON a.id = r.account_id
                 WHERE r.id = :id
                """)
            .param("id", id)
            .query((rs, n) -> new Row(
                    rs.getObject("id", UUID.class),
                    rs.getObject("account_id", UUID.class),
                    rs.getString("employee_no"),
                    rs.getString("name"),
                    rs.getString("requested_role"),
                    rs.getString("reason"),
                    rs.getString("status")))
            .optional();
    }

    public List<Row> findPending() {
        return jdbc.sql("""
                SELECT r.id, r.account_id, a.employee_no, a.name, r.requested_role, r.reason, r.status
                  FROM role_grant_request r JOIN account a ON a.id = r.account_id
                 WHERE r.status = 'PENDING'
                 ORDER BY r.created_at
                """)
            .query((rs, n) -> new Row(
                    rs.getObject("id", UUID.class),
                    rs.getObject("account_id", UUID.class),
                    rs.getString("employee_no"),
                    rs.getString("name"),
                    rs.getString("requested_role"),
                    rs.getString("reason"),
                    rs.getString("status")))
            .list();
    }

    public Optional<UUID> findPendingId(UUID accountId, Role role) {
        return jdbc.sql("""
                SELECT id FROM role_grant_request
                 WHERE account_id = :acc AND requested_role = :role AND status = 'PENDING'
                """)
            .param("acc", accountId).param("role", role.name())
            .query(UUID.class).optional();
    }

    /** 原子决策：只有仍是 PENDING 才更新，返回影响行数。 */
    public int decide(UUID requestId, String status, UUID adminId, String note) {
        return jdbc.sql("""
                UPDATE role_grant_request
                   SET status = :status, decided_by_account_id = :admin,
                       decided_at = now(), decision_note = :note
                 WHERE id = :id AND status = 'PENDING'
                """)
            .param("status", status).param("admin", adminId).param("note", note)
            .param("id", requestId).update();
    }
}
```

- [ ] **Step 4: 实现 `RoleGrantService`**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.audit.AuditService;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Map;
import java.util.UUID;

@Service
public class RoleGrantService {

    public static class NotAuthorizedException extends RuntimeException {
        public NotAuthorizedException(String message) {
            super(message);
        }
    }

    public static class AlreadyDecidedException extends RuntimeException {
        public AlreadyDecidedException() {
            super("该申请已被处理");
        }
    }

    public static class DuplicateRequestException extends RuntimeException {
        public DuplicateRequestException(Role role) {
            super("已有一个待处理的 " + role + " 申请");
        }
    }

    private final RoleGrantRepository requests;
    private final AccountRepository accounts;
    private final AuditService audit;

    public RoleGrantService(RoleGrantRepository requests, AccountRepository accounts, AuditService audit) {
        this.requests = requests;
        this.accounts = accounts;
        this.audit = audit;
    }

    @Transactional
    public UUID request(UUID accountId, Role role, String reason) {
        if (role == Role.MEMBER) {
            throw new IllegalArgumentException("MEMBER 在注册时自动获得，无需申请");
        }
        InputNormalizer.validateReason(reason);

        // 每次申请都重新解析账号状态，不信任调用方传入的上下文
        if (!accounts.isActive(accountId)) {
            throw new NotAuthorizedException("账号不可用");
        }
        if (requests.findPendingId(accountId, role).isPresent()) {
            throw new DuplicateRequestException(role);
        }

        UUID id = UUID.randomUUID();
        try {
            requests.insert(id, accountId, role, reason.strip());
        } catch (DuplicateKeyException e) {
            // 并发兜底：唯一索引 (account_id, requested_role) WHERE PENDING
            throw new DuplicateRequestException(role);
        }

        audit.record(accountId, "ACCOUNT", "ROLE_GRANT_REQUESTED", "role_grant_request",
                id.toString(), Map.of("role", role.name()));
        return id;
    }

    @Transactional
    public void approve(UUID requestId, UUID adminAccountId, String note) {
        decide(requestId, adminAccountId, "APPROVED", note);
    }

    @Transactional
    public void reject(UUID requestId, UUID adminAccountId, String note) {
        decide(requestId, adminAccountId, "REJECTED", note);
    }

    private void decide(UUID requestId, UUID adminAccountId, String decision, String note) {
        // 批准人必须是仍然有效的管理员 —— 每请求重新解析，不用缓存的角色快照
        if (!accounts.isActiveAdmin(adminAccountId)) {
            throw new NotAuthorizedException("需要有效的管理员身份");
        }

        RoleGrantRepository.Row row = requests.findById(requestId)
                .orElseThrow(() -> new AlreadyDecidedException());

        if (requests.decide(requestId, decision, adminAccountId, note) == 0) {
            throw new AlreadyDecidedException();
        }

        if ("APPROVED".equals(decision)) {
            accounts.grantRole(row.accountId(), Role.valueOf(row.requestedRole()), adminAccountId);
            audit.record(adminAccountId, "ADMIN", "ROLE_GRANT_APPROVED", "role_grant_request",
                    requestId.toString(),
                    Map.of("accountId", row.accountId().toString(),
                           "role", row.requestedRole(),
                           "note", note == null ? "" : note));
        } else {
            audit.record(adminAccountId, "ADMIN", "ROLE_GRANT_REJECTED", "role_grant_request",
                    requestId.toString(),
                    Map.of("accountId", row.accountId().toString(),
                           "role", row.requestedRole(),
                           "note", note == null ? "" : note));
        }
    }

    public List<RoleGrantRepository.Row> pending() {
        return requests.findPending();
    }
}
```

`AccountRepository` 再加一个方法：

```java
    /** 账号是否既是 ACTIVE 又是管理员。 */
    public boolean isActiveAdmin(UUID accountId) {
        return jdbc.sql("SELECT status, is_admin FROM account WHERE id = :id")
                .param("id", accountId)
                .query((rs, n) -> "ACTIVE".equals(rs.getString("status")) && rs.getBoolean("is_admin"))
                .optional().orElse(false);
    }
```

- [ ] **Step 5: 加 MCP 工具 `identity_request_role`**

在 `IdentityTools` 中追加：

```java
    private final RoleGrantService roleGrants;

    // 构造器追加 RoleGrantService roleGrants 参数

    public record RoleRequestResult(String requestId, String role, String status) {
    }

    @Tool(name = "identity_request_role",
          description = "申请 REVIEWER 或 OWNER 角色。申请不会立即生效，需管理员批准。")
    public RoleRequestResult requestRole(String role, String reason) {
        CurrentAccount me = identity.current();
        Role parsed;
        try {
            parsed = Role.valueOf(role == null ? "" : role.strip().toUpperCase());
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException("角色只能是 REVIEWER 或 OWNER");
        }
        UUID id = roleGrants.request(me.accountId(), parsed, reason);
        return new RoleRequestResult(id.toString(), parsed.name(), "PENDING");
    }

    @Tool(name = "identity_token_status",
          description = "返回当前令牌的前缀、签发时间与最后使用时间")
    public TokenStatus tokenStatus() {
        CurrentAccount me = identity.current();
        return tokens.statusOf(me.accountId());
    }

    @Tool(name = "identity_rotate_token",
          description = "轮换令牌。旧令牌立即失效，新令牌原文只返回这一次。")
    public RotateResult rotateToken(String currentTokenId) {
        CurrentAccount me = identity.current();
        TokenService.IssuedToken fresh = tokens.rotate(
                me.accountId(), UUID.fromString(currentTokenId), "轮换");
        return new RotateResult(fresh.plaintext(), fresh.prefix());
    }

    public record TokenStatus(String prefix, String createdAt, String lastUsedAt) {
    }

    public record RotateResult(String token, String tokenPrefix) {
    }
```

`TokenService` 补一个 `statusOf`：

```java
    public IdentityTools.TokenStatus statusOf(UUID accountId) {
        return jdbc.sql("""
                SELECT prefix, created_at, last_used_at FROM api_token
                 WHERE account_id = :acc AND revoked_at IS NULL
                 ORDER BY created_at DESC LIMIT 1
                """)
            .param("acc", accountId)
            .query((rs, n) -> new IdentityTools.TokenStatus(
                    rs.getString("prefix"),
                    String.valueOf(rs.getObject("created_at")),
                    String.valueOf(rs.getObject("last_used_at"))))
            .optional()
            .orElseThrow(() -> new IllegalStateException("无可用令牌"));
    }
```

`IdentityTools` 需要注入 `TokenService tokens`——构造器再加一个参数。

- [ ] **Step 6: 加管理端点**

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.identity.RoleGrantRepository;
import com.echoyan.a2ahub.identity.RoleGrantService;
import com.echoyan.a2ahub.security.CurrentAccount;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.UUID;

@RestController
public class AdminController {

    private final RoleGrantService roleGrants;

    public AdminController(RoleGrantService roleGrants) {
        this.roleGrants = roleGrants;
    }

    public record PendingGrantView(String id, String employeeNo, String name,
                                   String requestedRole, String reason) {
    }

    public record DecisionRequest(String note, String confirmation) {
    }

    @GetMapping("/api/admin/role-requests")
    public List<PendingGrantView> pending() {
        return roleGrants.pending().stream()
                .map(r -> new PendingGrantView(r.id().toString(), r.employeeNo(), r.name(),
                        r.requestedRole(), r.reason()))
                .toList();
    }

    @PostMapping("/api/admin/role-requests/{id}/approve")
    public void approve(@PathVariable UUID id, @RequestBody DecisionRequest body,
                        @AuthenticationPrincipal CurrentAccount admin) {
        roleGrants.approve(id, admin.accountId(), body.note());
    }

    @PostMapping("/api/admin/role-requests/{id}/reject")
    public void reject(@PathVariable UUID id, @RequestBody DecisionRequest body,
                       @AuthenticationPrincipal CurrentAccount admin) {
        roleGrants.reject(id, admin.accountId(), body.note());
    }
}
```

`AdminController` 自身不做管理员判定——判定在 `RoleGrantService.decide` 里，因为那是唯一不能绕过的地方。Web 层的 `@PreAuthorize` 是第二道，不是唯一一道。

`ApiExceptionHandler` 补三个映射：

```java
    @ExceptionHandler(RoleGrantService.NotAuthorizedException.class)
    public ProblemDetail onNotAuthorized(RoleGrantService.NotAuthorizedException e) {
        return problem(HttpStatus.FORBIDDEN, "NOT_AUTHORIZED", e.getMessage());
    }

    @ExceptionHandler(RoleGrantService.AlreadyDecidedException.class)
    public ProblemDetail onAlreadyDecided(RoleGrantService.AlreadyDecidedException e) {
        return problem(HttpStatus.CONFLICT, "ALREADY_DECIDED", e.getMessage());
    }

    @ExceptionHandler(RoleGrantService.DuplicateRequestException.class)
    public ProblemDetail onDuplicateRequest(RoleGrantService.DuplicateRequestException e) {
        return problem(HttpStatus.CONFLICT, "DUPLICATE_REQUEST", e.getMessage());
    }
```

- [ ] **Step 7: 运行测试**

Run: `mvn -B test -Dtest=RoleGrantServiceTest`
Expected: 10 个测试全部 PASS。

- [ ] **Step 8: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/ src/test/java/com/echoyan/a2ahub/identity/RoleGrantServiceTest.java
git commit -m "feat(identity): 角色申请制与管理员批准闭环"
```

---

## Task 10: 管理员引导账号与强制改密

**Files:**
- Create: `src/main/java/com/echoyan/a2ahub/identity/AdminBootstrap.java`
- Create: `src/main/java/com/echoyan/a2ahub/web/SessionController.java`
- Create: `src/main/java/com/echoyan/a2ahub/web/MustChangePasswordFilter.java`
- Create: `src/test/resources/application-test.yaml`
- Modify: `src/main/java/com/echoyan/a2ahub/config/SecurityConfig.java`
- Modify: `src/main/java/com/echoyan/a2ahub/identity/AccountRepository.java`
- Test: `src/test/java/com/echoyan/a2ahub/identity/AdminBootstrapTest.java`

**Interfaces:**
- Consumes: `AppProperties`（Task 6）、`AccountRepository`（Task 4）
- Produces:
  - `AdminBootstrap.run()` —— `ApplicationRunner`，启动时确保 `admin` 账号存在
  - `AdminBootstrap.isPasswordFromEnvironment()` —— 测试可断言

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import static org.assertj.core.api.Assertions.assertThat;

class AdminBootstrapTest extends IntegrationTestBase {

    @Autowired JdbcClient jdbc;

    @Test
    void 启动后存在admin账号且是管理员() {
        var row = jdbc.sql("""
                SELECT is_admin, must_change_password, password_digest IS NOT NULL AS has_pw
                  FROM account WHERE employee_no = 'ADMIN'
                """).query((rs, n) -> new Object[]{
                    rs.getBoolean("is_admin"), rs.getBoolean("must_change_password"), rs.getBoolean("has_pw")})
                .single();

        assertThat(row[0]).isEqualTo(true);
        assertThat(row[1]).isEqualTo(true);   // 强制首次改密
        assertThat(row[2]).isEqualTo(true);
    }

    @Test
    void admin账号只有ADMIN身份不是OWNER() {
        // ADMIN 与 OWNER 是两件事：ADMIN 管账号与角色，OWNER 读全组织材料。
        // 引导账号不应顺带获得 OWNER。
        Integer owners = jdbc.sql("""
                SELECT count(*) FROM account_role r JOIN account a ON a.id = r.account_id
                 WHERE a.employee_no = 'ADMIN' AND r.role = 'OWNER'
                """).query(Integer.class).single();
        assertThat(owners).isZero();
    }

    @Test
    void 未配置环境变量时生成随机密码且不落库明文() {
        // 测试环境未设置 A2AHUB_ADMIN_PASSWORD
        String digest = jdbc.sql("SELECT password_digest FROM account WHERE employee_no = 'ADMIN'")
                .query(String.class).single();

        // 摘要必须是 bcrypt 之类的慢哈希，不能是明文或简单 SHA
        assertThat(digest).doesNotContain("admin");
        assertThat(digest).doesNotContain("123456");
        assertThat(digest).startsWith("$2");
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=AdminBootstrapTest`
Expected: 失败——`admin` 账号不存在。

- [ ] **Step 3: 实现 `AdminBootstrap`**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.audit.AuditService;
import com.echoyan.a2ahub.config.AppProperties;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.attribute.PosixFilePermissions;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

/**
 * 确保存在一个引导管理员账号，用于批准第一个角色申请。
 *
 * <p>密码来源：环境变量 A2AHUB_ADMIN_PASSWORD；未设置则随机生成，
 * 打印一次并写入 0600 文件。不预置默认值、不硬编码、不做隐藏后门。
 */
@Component
public class AdminBootstrap implements ApplicationRunner {

    private static final Logger log = LoggerFactory.getLogger(AdminBootstrap.class);
    private static final String ADMIN_EMPLOYEE_NO = "ADMIN";

    private final AccountRepository accounts;
    private final AuditService audit;
    private final AppProperties properties;
    private final PasswordEncoder passwordEncoder;

    private boolean passwordFromEnvironment;

    public AdminBootstrap(AccountRepository accounts, AuditService audit, AppProperties properties,
                          PasswordEncoder passwordEncoder) {
        this.accounts = accounts;
        this.audit = audit;
        this.properties = properties;
        this.passwordEncoder = passwordEncoder;
    }

    public boolean isPasswordFromEnvironment() {
        return passwordFromEnvironment;
    }

    @Override
    public void run(ApplicationArguments args) {
        if (accounts.findIdByEmployeeNo(ADMIN_EMPLOYEE_NO).isPresent()) {
            return; // 已存在，不覆盖密码
        }

        String configured = properties.adminPassword();
        passwordFromEnvironment = configured != null && !configured.isBlank();

        String plaintext = passwordFromEnvironment ? configured : generateRandomPassword();

        UUID adminId = UUID.randomUUID();
        accounts.insertWithPassword(adminId, ADMIN_EMPLOYEE_NO, "系统管理员", "信息技术部",
                "管理员", passwordEncoder.encode(plaintext), true);
        accounts.grantRole(adminId, Role.MEMBER, null);

        audit.record(adminId, "SYSTEM", "ADMIN_BOOTSTRAPPED", "account", adminId.toString(),
                Map.of("passwordFromEnvironment", passwordFromEnvironment));

        if (passwordFromEnvironment) {
            log.info("引导管理员账号已创建，密码取自 A2AHUB_ADMIN_PASSWORD");
        } else {
            Path target = writePasswordFile(plaintext);
            log.warn("未配置 A2AHUB_ADMIN_PASSWORD，已生成随机密码并写入 {}（权限 0600）。"
                    + "首次登录后必须立即修改，该文件随后应删除。", target);
        }
    }

    private String generateRandomPassword() {
        byte[] bytes = new byte[24];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    private Path writePasswordFile(String plaintext) {
        try {
            Path dir = Path.of(properties.bootstrapDir());
            Files.createDirectories(dir);
            Path target = dir.resolve("admin-initial-password.txt");
            Files.writeString(target, plaintext + System.lineSeparator(), StandardCharsets.UTF_8);
            Files.setPosixFilePermissions(target, PosixFilePermissions.fromString("rw-------"));
            return target;
        } catch (IOException e) {
            throw new IllegalStateException(
                    "无法写入初始密码文件。请设置 A2AHUB_ADMIN_PASSWORD 环境变量后重启。", e);
        }
    }
}
```

`AccountRepository` 新增：

```java
    public void insertWithPassword(UUID id, String employeeNo, String name, String department,
                                   String position, String passwordDigest, boolean mustChangePassword) {
        jdbc.sql("""
                INSERT INTO account (id, employee_no, name, department, position, is_admin,
                                     password_digest, must_change_password)
                VALUES (:id, :no, :name, :dept, :pos, true, :pw, :must)
                """)
            .param("id", id).param("no", employeeNo).param("name", name)
            .param("dept", department).param("pos", position)
            .param("pw", passwordDigest).param("must", mustChangePassword)
            .update();
    }

    public Optional<String> findPasswordDigest(String employeeNo) {
        return jdbc.sql("SELECT password_digest FROM account WHERE employee_no = :no")
                .param("no", employeeNo).query(String.class).optional();
    }

    public void changePassword(UUID accountId, String newDigest) {
        jdbc.sql("""
                UPDATE account SET password_digest = :pw, must_change_password = false,
                                   updated_at = now()
                 WHERE id = :id
                """)
            .param("pw", newDigest).param("id", accountId).update();
    }
```

- [ ] **Step 4: 配置 `PasswordEncoder` 与会话登录**

`SecurityConfig` 中新增 bean 与浏览器链的完整配置：

```java
    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    @Order(2)
    SecurityFilterChain browserChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/actuator/health/**", "/api/session/login").permitAll()
                .anyRequest().authenticated())
            .formLogin(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .logout(logout -> logout.logoutUrl("/api/session/logout"));
        return http.build();
    }
```

登录端点由 `SessionController` 提供（Task 10 一并创建）：

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.identity.AccountRepository;
import com.echoyan.a2ahub.security.CurrentAccount;
import com.echoyan.a2ahub.security.CurrentAccountResolver;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.springframework.http.HttpStatus;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ResponseStatusException;

import java.util.Optional;

@RestController
public class SessionController {

    private final AccountRepository accounts;
    private final CurrentAccountResolver resolver;
    private final PasswordEncoder passwordEncoder;

    public SessionController(AccountRepository accounts, CurrentAccountResolver resolver,
                             PasswordEncoder passwordEncoder) {
        this.accounts = accounts;
        this.resolver = resolver;
        this.passwordEncoder = passwordEncoder;
    }

    public record LoginRequest(String employeeNo, String password) {
    }

    public record LoginResponse(String employeeNo, boolean mustChangePassword) {
    }

    @PostMapping("/api/session/login")
    public LoginResponse login(@RequestBody LoginRequest body, HttpServletRequest request) {
        String employeeNo = body.employeeNo() == null ? "" : body.employeeNo().strip().toUpperCase();
        String digest = accounts.findPasswordDigest(employeeNo)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.UNAUTHORIZED, "账号或密码错误"));

        if (!passwordEncoder.matches(body.password() == null ? "" : body.password(), digest)) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "账号或密码错误");
        }

        CurrentAccount account = resolver.resolve(
                        accounts.findIdByEmployeeNo(employeeNo).orElseThrow())
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.FORBIDDEN, "账号不可用"));

        var authorities = account.roles().stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r.name()))
                .toList();
        SecurityContextHolder.getContext().setAuthentication(
                new UsernamePasswordAuthenticationToken(account, null, authorities));

        HttpSession session = request.getSession(true);
        session.setAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
                SecurityContextHolder.getContext());

        return new LoginResponse(account.employeeNo(), account.mustChangePassword());
    }

    public record ChangePasswordRequest(String currentPassword, String newPassword) {
    }

    @PostMapping("/api/session/change-password")
    public void changePassword(@RequestBody ChangePasswordRequest body, HttpServletRequest request) {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !(auth.getPrincipal() instanceof CurrentAccount me)) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "未登录");
        }

        String digest = accounts.findPasswordDigest(me.employeeNo())
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.UNAUTHORIZED, "未登录"));
        if (!passwordEncoder.matches(body.currentPassword() == null ? "" : body.currentPassword(), digest)) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "当前密码不正确");
        }
        if (body.newPassword() == null || body.newPassword().length() < 12) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "新密码至少 12 位");
        }

        accounts.changePassword(me.accountId(), passwordEncoder.encode(body.newPassword()));
    }
}
```

- [ ] **Step 5: 强制改密在令牌路径上生效**

`must_change_password` 必须在**所有**路径上生效。加一个过滤器，放在 `TokenAuthFilter` 之后：

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.security.CurrentAccount;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

/** 待改密账号只能访问改密端点，其余一律 403。 */
@Component
public class MustChangePasswordFilter extends OncePerRequestFilter {

    private static final List<String> ALLOWED = List.of(
            "/api/session/change-password", "/api/session/logout");

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof CurrentAccount me
                && me.mustChangePassword()
                && ALLOWED.stream().noneMatch(p -> request.getRequestURI().startsWith(p))) {
            response.setStatus(HttpStatus.FORBIDDEN.value());
            response.setContentType("application/json");
            response.getWriter().write(
                    "{\"code\":\"MUST_CHANGE_PASSWORD\",\"message\":\"必须先修改初始密码\"}");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

在 `SecurityConfig` 的两条链上都注册：

```java
            .addFilterAfter(mustChangePasswordFilter, TokenAuthFilter.class)   // agent 链
            .addFilterAfter(mustChangePasswordFilter, SecurityContextHolderFilter.class)  // browser 链
```

- [ ] **Step 6: 写强制改密测试**

先建 `src/test/resources/application-test.yaml`，把引导目录指到临时位置，避免测试污染工作区：

```yaml
a2ahub:
  bootstrap-dir: ${java.io.tmpdir}/a2ahub-test-bootstrap
```

再写测试。**注意探针端点必须选浏览器链上的**（`/api/admin/**`），不能用 `/api/agent/protected-probe`——那条链是无状态的，会话 Cookie 不被认证，只会得到 401 而不是我们要断言的 403。

```java
    @Test
    void 待改密账号被拦在改密端点之外() throws Exception {
        // 引导账号 ADMIN 的 must_change_password = true
        ResponseEntity<String> login = rest.postForEntity("/api/session/login",
                Map.of("employeeNo", "ADMIN", "password", adminPassword()), String.class);
        assertThat(login.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(login.getBody()).contains("\"mustChangePassword\":true");

        String cookie = login.getHeaders().getFirst(HttpHeaders.SET_COOKIE);
        assertThat(cookie).isNotNull();

        // 浏览器链上的受保护端点：应被 MustChangePasswordFilter 拦下
        ResponseEntity<String> blocked = exchangeWithCookie("/api/admin/role-requests", cookie);
        assertThat(blocked.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
        assertThat(blocked.getBody()).contains("MUST_CHANGE_PASSWORD");

        // 改密端点本身必须放行，否则账号会被永久锁死
        ResponseEntity<String> change = rest.exchange("/api/session/change-password",
                HttpMethod.POST,
                new HttpEntity<>(Map.of("currentPassword", adminPassword(),
                                        "newPassword", "a-very-long-new-password-2026"),
                                 jsonWithCookie(cookie)),
                String.class);
        assertThat(change.getStatusCode()).isEqualTo(HttpStatus.OK);
    }

    @Test
    void 改密后标记清除且可正常访问() throws Exception {
        String initial = adminPassword();
        ResponseEntity<String> login = rest.postForEntity("/api/session/login",
                Map.of("employeeNo", "ADMIN", "password", initial), String.class);
        String cookie = login.getHeaders().getFirst(HttpHeaders.SET_COOKIE);

        rest.exchange("/api/session/change-password", HttpMethod.POST,
                new HttpEntity<>(Map.of("currentPassword", initial,
                                        "newPassword", "a-very-long-new-password-2026"),
                                 jsonWithCookie(cookie)),
                String.class);

        // 改密后 must_change_password 必须为 false
        assertThat(jdbc.sql("SELECT must_change_password FROM account WHERE employee_no = 'ADMIN'")
                .query(Boolean.class).single()).isFalse();

        // 重新登录后应能正常访问
        ResponseEntity<String> relogin = rest.postForEntity("/api/session/login",
                Map.of("employeeNo", "ADMIN", "password", "a-very-long-new-password-2026"),
                String.class);
        assertThat(relogin.getBody()).contains("\"mustChangePassword\":false");

        ResponseEntity<String> allowed = exchangeWithCookie("/api/admin/role-requests",
                relogin.getHeaders().getFirst(HttpHeaders.SET_COOKIE));
        assertThat(allowed.getStatusCode()).isEqualTo(HttpStatus.OK);
    }

    @Autowired AppProperties properties;
    @Autowired JdbcClient jdbc;

    /** 测试环境未设置 A2AHUB_ADMIN_PASSWORD，密码由引导流程随机生成并落盘。 */
    private String adminPassword() throws IOException {
        return Files.readString(
                Path.of(properties.bootstrapDir()).resolve("admin-initial-password.txt")).strip();
    }

    private HttpHeaders jsonWithCookie(String cookie) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.add(HttpHeaders.COOKIE, cookie);
        return headers;
    }

    private ResponseEntity<String> exchangeWithCookie(String path, String cookie) {
        HttpHeaders headers = new HttpHeaders();
        headers.add(HttpHeaders.COOKIE, cookie);
        return rest.exchange(path, HttpMethod.GET, new HttpEntity<>(headers), String.class);
    }
```

用到的 import：`com.echoyan.a2ahub.config.AppProperties`、`org.springframework.jdbc.core.simple.JdbcClient`、`org.springframework.http.*`、`java.nio.file.*`、`java.io.IOException`、`java.util.Map`。

这两个测试还顺带证明了「随机密码 + 落盘 + 0600」这条路径真的能跑通——它读的就是引导流程写出的那个文件。

- [ ] **Step 7: 运行测试**

Run: `mvn -B test -Dtest=AdminBootstrapTest`
Expected: 3 个测试全部 PASS。

- [ ] **Step 8: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/ src/test/java/com/echoyan/a2ahub/identity/AdminBootstrapTest.java
git commit -m "feat(identity): 管理员引导账号、会话登录与强制首次改密"
```

---

## Task 11: 停用账号在所有路径被拒

**Files:**
- Modify: `src/main/java/com/echoyan/a2ahub/identity/AccountService.java`（新建）
- Create: `src/main/java/com/echoyan/a2ahub/web/AdminMemberController.java`
- Test: `src/test/java/com/echoyan/a2ahub/identity/AccountDisablingTest.java`

**Interfaces:**
- Consumes: `CurrentAccountResolver`（Task 5）、`TokenService`（Task 5）
- Produces:
  - `void AccountService.disable(UUID accountId, UUID adminAccountId, String reason)`
  - `void AccountService.enable(UUID accountId, UUID adminAccountId, String reason)`
  - HTTP `POST /api/admin/members/{id}/disable`、`POST /api/admin/members/{id}/enable`

- [ ] **Step 1: 写失败测试**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.security.CurrentAccountResolver;
import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class AccountDisablingTest extends IntegrationTestBase {

    @Autowired RegistrationService registrations;
    @Autowired AccountService accounts;
    @Autowired TokenService tokens;
    @Autowired CurrentAccountResolver resolver;
    @Autowired JdbcClient jdbc;

    private UUID registerAdmin(String no) {
        UUID id = registrations.register(new RegistrationService.RegisterCommand(
                no, "管理员", "信息技术部", "管理员", null)).accountId();
        jdbc.sql("UPDATE account SET is_admin = true WHERE id = :id").param("id", id).update();
        return id;
    }

    @Test
    void 停用后令牌立即失效() {
        var victim = registrations.register(new RegistrationService.RegisterCommand(
                "E8001", "员工", "研发部", "工程师", null));
        UUID admin = registerAdmin("E8002");

        assertThat(tokens.verify(victim.token().plaintext())).isPresent();

        accounts.disable(victim.accountId(), admin, "离职");

        assertThat(tokens.verify(victim.token().plaintext())).isEmpty();
    }

    @Test
    void 停用后解析器不再返回账号() {
        var victim = registrations.register(new RegistrationService.RegisterCommand(
                "E8003", "员工", "研发部", "工程师", null));
        UUID admin = registerAdmin("E8004");

        accounts.disable(victim.accountId(), admin, "离职");

        assertThat(resolver.resolve(victim.accountId())).isEmpty();
    }

    @Test
    void 停用后角色不再产生任何权威() {
        var leader = registrations.register(new RegistrationService.RegisterCommand(
                "E8005", "领导", "研发部", "经理", null));
        UUID admin = registerAdmin("E8006");
        jdbc.sql("INSERT INTO account_role (account_id, role) VALUES (:id, 'REVIEWER')")
                .param("id", leader.accountId()).update();

        accounts.disable(leader.accountId(), admin, "离职");

        assertThat(resolver.loadRoles(leader.accountId())).contains(Role.REVIEWER);
        // 角色行还在（保留历史），但账号不可用，因此不产生任何权威
        assertThat(resolver.resolve(leader.accountId())).isEmpty();
    }

    @Test
    void 恢复后令牌重新可用() {
        var victim = registrations.register(new RegistrationService.RegisterCommand(
                "E8007", "员工", "研发部", "工程师", null));
        UUID admin = registerAdmin("E8008");

        accounts.disable(victim.accountId(), admin, "误操作");
        accounts.enable(victim.accountId(), admin, "恢复");

        assertThat(tokens.verify(victim.token().plaintext())).isPresent();
    }

    @Test
    void 停用与恢复都写审计() {
        var victim = registrations.register(new RegistrationService.RegisterCommand(
                "E8009", "员工", "研发部", "工程师", null));
        UUID admin = registerAdmin("E8010");

        accounts.disable(victim.accountId(), admin, "离职");
        accounts.enable(victim.accountId(), admin, "恢复");

        assertThat(jdbc.sql("SELECT count(*) FROM audit_log WHERE action = 'ACCOUNT_DISABLED'")
                .query(Integer.class).single()).isEqualTo(1);
        assertThat(jdbc.sql("SELECT count(*) FROM audit_log WHERE action = 'ACCOUNT_ENABLED'")
                .query(Integer.class).single()).isEqualTo(1);
    }

    @Test
    void 非管理员不能停用账号() {
        var victim = registrations.register(new RegistrationService.RegisterCommand(
                "E8011", "员工", "研发部", "工程师", null));
        var notAdmin = registrations.register(new RegistrationService.RegisterCommand(
                "E8012", "路人", "研发部", "工程师", null));

        org.assertj.core.api.Assertions.assertThatThrownBy(
                () -> accounts.disable(victim.accountId(), notAdmin.accountId(), "我说了算"))
            .isInstanceOf(RoleGrantService.NotAuthorizedException.class);
    }

    @Test
    void 不能停用自己() {
        UUID admin = registerAdmin("E8013");
        org.assertj.core.api.Assertions.assertThatThrownBy(
                () -> accounts.disable(admin, admin, "手滑"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: 运行确认失败**

Run: `mvn -B test -Dtest=AccountDisablingTest`
Expected: 编译失败，`AccountService` 不存在。

- [ ] **Step 3: 实现 `AccountService`**

```java
package com.echoyan.a2ahub.identity;

import com.echoyan.a2ahub.audit.AuditService;
import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Map;
import java.util.UUID;

@Service
public class AccountService {

    private final JdbcClient jdbc;
    private final AccountRepository accounts;
    private final AuditService audit;

    public AccountService(JdbcClient jdbc, AccountRepository accounts, AuditService audit) {
        this.jdbc = jdbc;
        this.accounts = accounts;
        this.audit = audit;
    }

    @Transactional
    public void disable(UUID accountId, UUID adminAccountId, String reason) {
        requireAdmin(adminAccountId);
        if (accountId.equals(adminAccountId)) {
            throw new IllegalArgumentException("不能停用自己的账号");
        }
        setStatus(accountId, "DISABLED");
        audit.record(adminAccountId, "ADMIN", "ACCOUNT_DISABLED", "account", accountId.toString(),
                Map.of("reason", reason == null ? "" : reason));
    }

    @Transactional
    public void enable(UUID accountId, UUID adminAccountId, String reason) {
        requireAdmin(adminAccountId);
        setStatus(accountId, "ACTIVE");
        audit.record(adminAccountId, "ADMIN", "ACCOUNT_ENABLED", "account", accountId.toString(),
                Map.of("reason", reason == null ? "" : reason));
    }

    private void requireAdmin(UUID adminAccountId) {
        if (!accounts.isActiveAdmin(adminAccountId)) {
            throw new RoleGrantService.NotAuthorizedException("需要有效的管理员身份");
        }
    }

    private void setStatus(UUID accountId, String status) {
        int updated = jdbc.sql("UPDATE account SET status = :status, updated_at = now() WHERE id = :id")
                .param("status", status).param("id", accountId).update();
        if (updated == 0) {
            throw new IllegalArgumentException("账号不存在");
        }
    }
}
```

**注意本任务没有「撤销令牌」这一步，这是刻意的。** 令牌失效由 `TokenService.verify` 里的账号状态检查承担——停用账号后令牌自动失效，不需要遍历撤销。这消除了「停用了账号但漏撤销令牌」这一整类缺陷。

- [ ] **Step 4: 加管理端点**

```java
package com.echoyan.a2ahub.web;

import com.echoyan.a2ahub.identity.AccountService;
import com.echoyan.a2ahub.security.CurrentAccount;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.UUID;

@RestController
public class AdminMemberController {

    private final AccountService accounts;

    public AdminMemberController(AccountService accounts) {
        this.accounts = accounts;
    }

    public record StatusRequest(String reason) {
    }

    @PostMapping("/api/admin/members/{id}/disable")
    public void disable(@PathVariable UUID id, @RequestBody StatusRequest body,
                        @AuthenticationPrincipal CurrentAccount admin) {
        accounts.disable(id, admin.accountId(), body.reason());
    }

    @PostMapping("/api/admin/members/{id}/enable")
    public void enable(@PathVariable UUID id, @RequestBody StatusRequest body,
                       @AuthenticationPrincipal CurrentAccount admin) {
        accounts.enable(id, admin.accountId(), body.reason());
    }
}
```

- [ ] **Step 5: 运行测试**

Run: `mvn -B test -Dtest=AccountDisablingTest`
Expected: 7 个测试全部 PASS。

- [ ] **Step 6: 提交**

```bash
git add src/main/java/com/echoyan/a2ahub/ src/test/java/com/echoyan/a2ahub/identity/AccountDisablingTest.java
git commit -m "feat(identity): 账号停用与恢复，令牌失效由状态检查承担"
```

---

## Task 12: Skill 凭证落盘与 MCP 配置生成

**Files:**
- Create: `skill/a2ahub/SKILL.md`
- Create: `skill/a2ahub/scripts/write-credentials.sh`
- Create: `skill/a2ahub/scripts/mcp-config.json.tmpl`
- Test: `src/test/java/com/echoyan/a2ahub/skill/SkillArtifactsTest.java`

**Interfaces:**
- Consumes: `POST /api/agent/register`（Task 4）
- Produces: 交付给客户的 Skill 目录结构

- [ ] **Step 1: 写凭证落盘脚本**

```bash
#!/usr/bin/env bash
# 把注册返回的令牌写入 ~/.a2ahub/credentials.json，权限 0600。
# 令牌不写入项目仓库、不写入普通日志、不出现在命令行参数里（从 stdin 读）。
set -euo pipefail

CRED_DIR="${A2AHUB_CRED_DIR:-$HOME/.a2ahub}"
CRED_FILE="$CRED_DIR/credentials.json"

mkdir -p "$CRED_DIR"
chmod 700 "$CRED_DIR"

umask 077
cat > "$CRED_FILE" <<'CREDENTIALS'
CREDENTIALS

chmod 600 "$CRED_FILE"
echo "凭证已写入 $CRED_FILE（权限 600）"
```

调用方用法（从 stdin 传入注册响应）：

```bash
echo "$REGISTER_RESPONSE" | jq '{token: .token, prefix: .tokenPrefix, employeeNo: "..."}' \
  | ./write-credentials.sh
```

**注意**：令牌经 stdin 传递，不出现在 `ps` 可见的命令行参数里。

- [ ] **Step 2: 写 MCP 配置模板**

`skill/a2ahub/scripts/mcp-config.json.tmpl`：

```json
{
  "mcpServers": {
    "a2ahub": {
      "type": "http",
      "url": "${A2AHUB_MCP_URL}",
      "headers": {
        "Authorization": "Bearer ${A2AHUB_TOKEN}"
      }
    }
  }
}
```

- [ ] **Step 3: 写 `SKILL.md`**

必须包含以下内容，且**明确标注未实测的部分**：

````markdown
# A2AHub Skill

## 用途

引导 Agent 完成 A2AHub 的初始化：注册身份、保存凭证、配置 MCP 连接。

## 初始化步骤

### 1. 注册

向 A2AHub 服务端发起注册。**注册走 HTTP，不走 MCP**——此时还没有令牌。

```bash
curl -X POST "$A2AHUB_BASE_URL/api/agent/register" \
  -H 'Content-Type: application/json' \
  -d '{
    "employeeNo": "<员工工号>",
    "name": "<姓名>",
    "department": "<部门>",
    "position": "<岗位>",
    "supervisorEmployeeNo": "<上级工号，可留空>"
  }'
```

**必须向用户展示返回的 `supervisor` 字段并请求确认**：

- `active: true` —— 「已识别你的上级为 <resolvedName>，是否正确？」
- `active: false` —— 「工号 <submittedEmployeeNo> 尚未注册，上下级关系将在对方注册后自动生效。是否正确？」

工号一旦注册不可自行更改，只能由管理员更正。**注册前务必确认工号输入正确。**

### 2. 保存凭证

把响应中的 `token` 交给 `scripts/write-credentials.sh`（经 stdin），它会写入
`~/.a2ahub/credentials.json`，权限 0600。

**令牌原文只在注册响应中出现这一次。** 不要把它写进项目仓库、日志或聊天回复。

### 3. 配置 MCP

用 `scripts/mcp-config.json.tmpl` 生成客户端配置，`A2AHUB_TOKEN` 从凭证文件读取。

配置位置按客户端而定：

| 客户端 | 配置文件 | 生效方式 |
|---|---|---|
| OpenClaw | `~/.openclaw/openclaw.json`（Gateway 主机） | 热生效，无需重启 |
| WorkBuddy | `~/.workbuddy/mcp.json` | **需重启客户端并点击「信任」** |

**OpenClaw 必须显式写 `"transport": "streamable-http"`**——省略时默认为 `sse`，会连不上。

### 4. 申请角色

注册只得到 MEMBER。如果需要领导或老板权限，调用 MCP 工具：

```
identity_request_role(role: "REVIEWER" | "OWNER", reason: "<申请理由>")
```

申请**不会立即生效**，需要管理员在管理台批准。告诉用户这一点，不要让他们以为已经拿到权限。

## 客户端支持性说明

| 客户端路径 | 状态 |
|---|---|
| MCP Streamable HTTP（原始 JSON-RPC） | **已实测**，仓库内 `McpIdentityTest` 覆盖 |
| OpenClaw `transport: streamable-http` | **按官方资料配置，未实测** |
| WorkBuddy `type: "http"` + headers | **未实测**——其桌面端文档只记录 stdio，必须真实客户端确认 |
| MCP stdio 传输 | **不支持** |
````

- [ ] **Step 4: 写产物测试**

```java
package com.echoyan.a2ahub.skill;

import org.junit.jupiter.api.Test;

import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class SkillArtifactsTest {

    private static final Path SKILL_DIR = Path.of("skill/a2ahub");

    @Test
    void skill目录结构完整() {
        assertThat(SKILL_DIR.resolve("SKILL.md")).exists();
        assertThat(SKILL_DIR.resolve("scripts/write-credentials.sh")).exists();
        assertThat(SKILL_DIR.resolve("scripts/mcp-config.json.tmpl")).exists();
    }

    @Test
    void 凭证脚本以0600写入且令牌不经命令行参数() throws Exception {
        String script = Files.readString(SKILL_DIR.resolve("scripts/write-credentials.sh"));
        assertThat(script).contains("chmod 600");
        assertThat(script).contains("umask 077");
        // 令牌必须从 stdin 读，不能出现在 argv 里（argv 对 ps 可见）
        assertThat(script).doesNotContain("$1");
    }

    @Test
    void skill文档明确标注未实测的客户端路径() throws Exception {
        String doc = Files.readString(SKILL_DIR.resolve("SKILL.md"));
        assertThat(doc).contains("未实测");
        assertThat(doc).contains("streamable-http");
        assertThat(doc).contains("不支持");
    }

    @Test
    void skill文档说明申请不立即生效() throws Exception {
        String doc = Files.readString(SKILL_DIR.resolve("SKILL.md"));
        assertThat(doc).contains("不会立即生效");
    }
}
```

- [ ] **Step 5: 运行测试**

Run: `mvn -B test -Dtest=SkillArtifactsTest`
Expected: 4 个测试 PASS。

- [ ] **Step 6: 提交**

```bash
git add skill/ src/test/java/com/echoyan/a2ahub/skill/
git commit -m "feat(skill): 凭证落盘脚本、MCP 配置模板与引导文档"
```

---

## Task 13: 阶段 1 端到端验收

**Files:**
- Create: `src/test/java/com/echoyan/a2ahub/AcceptancePhase1Test.java`

**Interfaces:**
- Consumes: 阶段 1 的全部组件
- Produces: 规格 §13 中 15.1 / 15.2 / 15.3 / 15.4 的可运行证据

- [ ] **Step 1: 写验收测试**

```java
package com.echoyan.a2ahub;

import com.echoyan.a2ahub.identity.*;
import com.echoyan.a2ahub.security.CurrentAccountResolver;
import com.echoyan.a2ahub.support.IntegrationTestBase;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.simple.JdbcClient;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AcceptancePhase1Test extends IntegrationTestBase {

    @Autowired RegistrationService registrations;
    @Autowired RoleGrantService grants;
    @Autowired TokenService tokens;
    @Autowired AccountService accounts;
    @Autowired CurrentAccountResolver resolver;
    @Autowired JdbcClient jdbc;

    private UUID registerAdmin(String no) {
        UUID id = registrations.register(new RegistrationService.RegisterCommand(
                no, "管理员", "信息技术部", "管理员", null)).accountId();
        jdbc.sql("UPDATE account SET is_admin = true WHERE id = :id").param("id", id).update();
        return id;
    }

    @Test
    void 验收15_1_普通注册无法获得老板权限() {
        var boss = registrations.register(new RegistrationService.RegisterCommand(
                "B001", "老板", "总经办", "总经理", null));

        // 无论填什么资料、什么岗位，注册只产生 MEMBER
        assertThat(resolver.loadRoles(boss.accountId())).containsExactly(Role.MEMBER);
        assertThat(resolver.resolve(boss.accountId()).orElseThrow().hasRole(Role.OWNER)).isFalse();
    }

    @Test
    void 验收15_2_老板经申请与管理员批准获得OWNER() {
        UUID admin = registerAdmin("A001");
        var boss = registrations.register(new RegistrationService.RegisterCommand(
                "B002", "老板", "总经办", "总经理", null));

        assertThat(resolver.loadRoles(boss.accountId())).doesNotContain(Role.OWNER);

        UUID requestId = grants.request(boss.accountId(), Role.OWNER, "我是公司负责人");
        assertThat(resolver.loadRoles(boss.accountId())).doesNotContain(Role.OWNER);

        grants.approve(requestId, admin, "已核对");

        assertThat(resolver.loadRoles(boss.accountId())).contains(Role.OWNER);
        assertThat(resolver.resolve(boss.accountId()).orElseThrow().hasRole(Role.OWNER)).isTrue();
    }

    @Test
    void 验收15_3_停用账号无法访问() {
        UUID admin = registerAdmin("A002");
        var member = registrations.register(new RegistrationService.RegisterCommand(
                "M001", "员工", "研发部", "工程师", null));

        accounts.disable(member.accountId(), admin, "离职");

        assertThat(tokens.verify(member.token().plaintext())).isEmpty();
        assertThat(resolver.resolve(member.accountId())).isEmpty();
    }

    @Test
    void 验收15_4_自报资料不提升权限() {
        // 自报岗位为「总经理」、部门为「总经办」、并声称上级是某个工号
        var selfClaimed = registrations.register(new RegistrationService.RegisterCommand(
                "M002", "张三", "总经办", "总经理", "B003"));

        // 资料原样保存，但不产生任何角色
        assertThat(resolver.loadRoles(selfClaimed.accountId())).containsExactly(Role.MEMBER);

        // 上级关系是 PENDING，且不产生可用权威
        assertThat(selfClaimed.supervisor().active()).isFalse();
        assertThat(resolver.resolve(selfClaimed.accountId()).orElseThrow().hasRole(Role.OWNER)).isFalse();
    }

    @Test
    void 阶段一闭环_注册到角色生效的完整路径() {
        UUID admin = registerAdmin("A003");

        // 领导自助注册
        var leader = registrations.register(new RegistrationService.RegisterCommand(
                "L001", "李经理", "研发部", "经理", null));
        assertThat(resolver.loadRoles(leader.accountId())).containsExactly(Role.MEMBER);

        // 申请 REVIEWER
        UUID requestId = grants.request(leader.accountId(), Role.REVIEWER, "我负责研发部评审");
        UUID approved = jdbc.sql("""
                SELECT decided_by_account_id FROM role_grant_request WHERE id = :id
                """).param("id", requestId).query(UUID.class).optional().orElse(null);
        assertThat(approved).isNull();  // 尚未批准

        grants.approve(requestId, admin, "已核对组织架构");

        assertThat(resolver.loadRoles(leader.accountId())).contains(Role.REVIEWER);
        assertThat(tokens.verify(leader.token().plaintext()).orElseThrow().hasRole(Role.REVIEWER))
                .isTrue();

        // 员工注册并认这个领导为上级，链接立即生效
        var member = registrations.register(new RegistrationService.RegisterCommand(
                "M003", "王五", "研发部", "工程师", "L001"));
        assertThat(member.supervisor().active()).isTrue();
        assertThat(member.supervisor().resolvedName()).isEqualTo("李经理");
    }

    @Test
    void 阶段一闭环_停用领导后下级不再有可用上级() {
        UUID admin = registerAdmin("A004");
        var leader = registrations.register(new RegistrationService.RegisterCommand(
                "L002", "赵经理", "研发部", "经理", null));
        var member = registrations.register(new RegistrationService.RegisterCommand(
                "M004", "钱六", "研发部", "工程师", "L002"));

        assertThat(member.supervisor().active()).isTrue();

        accounts.disable(leader.accountId(), admin, "离职");

        // 链接行仍是 ACTIVE，但停用账号不产生可用权威
        assertThat(jdbc.sql("""
                SELECT status FROM supervisor_link WHERE subordinate_account_id = :id
                """).param("id", member.accountId()).query(String.class).single())
            .isEqualTo("ACTIVE");
    }
}
```

- [ ] **Step 2: 运行验收测试**

Run: `mvn -B test -Dtest=AcceptancePhase1Test`
Expected: 6 个测试全部 PASS。

- [ ] **Step 3: 跑全量测试**

Run: `mvn -B test`
Expected: 全部 PASS。若有失败，**先修实现，不要改断言去迁就**——除非断言本身写错了。

- [ ] **Step 4: 提交**

```bash
git add src/test/java/com/echoyan/a2ahub/AcceptancePhase1Test.java
git commit -m "test: 阶段1验收测试，覆盖 15.1 / 15.2 / 15.3 / 15.4"
```

- [ ] **Step 5: 向用户报告阶段 1 完成**

报告必须包含：

1. `mvn -B test` 的真实输出（通过数、失败数、耗时）
2. Task 7 Step 1 的技术验证结论——`SecurityContextHolder` 在 `@McpTool` 中是否可见，走了方案 A 还是方案 B
3. 实际观测到的 MCP 线格式（`docs/superpowers/notes/mcp-wire-format.md` 的路径）
4. **未完成或未验证的部分**——例如 WorkBuddy 的 HTTP 传输仍未在真实客户端上验证

**不得把未测试的能力描述为已支持。**

---

## Self-Review

**1. 规格覆盖检查**

| 规格章节 | 覆盖任务 |
|---|---|
| §5.1 注册流程 | Task 4、Task 8 |
| §5.2 凭证存储 | Task 12 |
| §5.3 两条过滤器链 | Task 6 |
| §5.4 角色申请制 | Task 9 |
| §5.5 授权判定（材料读取部分） | **不在阶段 1**——材料尚不存在，属阶段 2 |
| §5.6 授权实现纪律 | Task 5（每请求解析）、Task 9 / 11（不信任传入上下文） |
| §4.1 六张表 | Task 2 |
| §10.1 Level A 实测 | Task 7 |
| §10.2 / §10.4 客户端边界与声明纪律 | Task 12 |
| §9.3 管理台账号与强制改密 | Task 10 |
| §11.2 配置项 | Task 1、Task 6 |

阶段 1 不含材料、附件、评审请求——这些是阶段 2 与阶段 3。§5.5 中的材料读取规则在阶段 2 落地。

**2. 占位符扫描**

已扫描：无 TBD / TODO / "类似 Task N" / "补充适当的错误处理"。每个代码步骤都有可执行内容。

自审中发现并已修复的五处实质缺陷（不是措辞问题）：

1. **注册的并发兜底会炸。** 原写法 `try { insert } catch (DuplicateKeyException) { 再查一次 }` 在 PostgreSQL 上必然失败——语句报错后事务进入 aborted 状态，catch 块里的查询会得到 `current transaction is aborted`。改为 `INSERT ... ON CONFLICT (employee_no) DO NOTHING` 加影响行数判断（Task 4 Step 4 / Step 7）。
2. **强制改密的测试探针选错了链。** 原打算用 `/api/agent/protected-probe`，但那条链是无状态的，会话 Cookie 不被认证，断言会得到 401 而非 403。改用浏览器链上的 `/api/admin/role-requests`（Task 10 Step 6）。
3. **环路测试用了不存在的断言方法** `assertThatThrownBy(...).doesNothing()`，且用裸 SQL 准备数据绕过了被测代码。改为纯服务调用构造环路（Task 8 Step 1）。
4. **`CurrentAccountResolver` 写了两版**，第一版四次数据库往返且"停用即失效"靠解析后过滤。收敛为单次查询，并把 `status = 'ACTIVE'` 做成 SQL 条件（Task 5 Step 3）。
5. **`MessageDigest` 的包写错**（`java.util` → `java.security`），并删掉了 `ApiExceptionHandler` 里用不上的 `ObjectMapper` 字段。

**3. 类型一致性**

- `TokenService.IssuedToken` 在 Task 4 定义，Task 5、Task 9、Task 12 使用同一名称
- `RegistrationService.RegistrationResult` / `SupervisorResolution` / `RegisterCommand` 在 Task 4 定义，Task 8、Task 13 使用同一名称
- `CurrentAccount` 在 Task 5 定义，Task 7、Task 9、Task 10、Task 11 使用同一名称与 `hasRole` 方法
- `AccountRepository.isActive` 在 Task 8 新增，Task 9 的 `RoleGrantService` 使用
- `AccountRepository.isActiveAdmin` 在 Task 9 新增，Task 11 的 `AccountService` 使用
- `RoleGrantService.NotAuthorizedException` 在 Task 9 定义，Task 11 复用

**4. Review Focus 覆盖**

| # | 类别 | 归属任务 |
|---|---|---|
| 1 | 并发注册同工号 | Task 4 Step 1 的 `并发注册同一工号只有一个成功` |
| 2 | id 不存在的令牌 | Task 5 Step 1 的 `id不存在时返回空而不是抛异常` |
| 3 | 上级指向已停用账号 | Task 8 Step 1 的 `上级账号被停用后不产生可用权威` |
| 4 | 工号大小写/空白/超长 | Task 3 全部测试 + Task 4 的 `工号大小写不同视为冲突` |
| 5 | 缺 Origin 头 | Task 6 Step 1 的 `缺少Origin头必须放行` |
