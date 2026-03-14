## GitNexus 团队培训手册（DICT 权限场景版）

> 面向：DICT 系统后端开发、架构师、代码审核人员  
> 场景：DICT 模块权限回收、查询权限稽核等存量系统改造  
> 版本：2026-03

---

### 第一章 GitNexus 介绍与快速上手

#### 1.1 GitNexus 是什么

- **GitNexus 定位**：面向 AI 代码助手的**代码知识图谱引擎**，在本地为代码仓库建立图数据库，记录类、方法、调用关系、执行流程、功能聚类等结构化信息，通过 MCP 工具暴露给 AI 使用。
- **解决的问题**：
  - **改代码不知道会影响哪里**：GitNexus 用 `impact` / `detect_changes` 把改动映射到调用链和执行流。
  - **接手老项目难以上手**：通过 `query` / `context` 直接从“业务概念 → 执行流程 → 关键类方法”。
  - **重构/重命名风险大**：用 `rename` 基于调用图批量重命名，避免遗漏引用。

#### 1.2 与 DICT 项目的关系

- 当前已对 **simbest-cloud-dict-agent / simbest-boot-dict** 等仓库建立 GitNexus 索引，支持：
  - 查询权限申请/审批/生效/失效/清理 全链路
  - 查询权限稽核任务、稽核日志记录与监控告警
  - 菜单权限、操作日志（`UserClickLogSaveDto` 等）
- 在 Cursor / Claude Code 中，AI 已内置 GitNexus MCP 工具和相关技能，可以在对话中直接用自然语言触发。

#### 1.3 安装 GitNexus CLI

- **前置条件**：
  - Node.js ≥ 18
  - npm（随 Node 一起安装）

- **全局安装（只需一次）**：

```bash
npm install -g gitnexus

# 验证版本
gitnexus --version
```

#### 1.4 为仓库建立索引（初始化仓库）

- 在 **项目根目录**（例如 `simbest-cloud-dict-agent`）执行：

```bash
cd /path/to/simbest-cloud-dict-agent

# 首次索引（构建知识图谱）
gitnexus analyze
```

- 这一步会完成：
  - 扫描文件结构，解析 Java / TypeScript 等代码。
  - 建立函数/类/方法/接口等符号表和调用关系。
  - 自动生成或更新 `AGENTS.md`、`CLAUDE.md`、`.claude/skills/`。
  - 在项目内创建 `.gitnexus/` 目录（索引数据库，仅本地使用，**不要提交 Git**）。

> **建议**：将 `.gitnexus/` 添加到 `.gitignore`，而把 `.claude/skills/`、`AGENTS.md`、`CLAUDE.md` 提交到仓库，方便团队共享。

#### 1.5 将 GitNexus 配置到 AI / CLI 工具中

**1）配置到 Claude Code / Cursor（MCP）**

- 自动配置（推荐）：

```bash
# 在任意项目目录执行一次
gitnexus setup

# 或者手动配置（效果相同）
claude mcp add gitnexus -- npx -y gitnexus@latest mcp
```

- 作用：
  - 注册 GitNexus MCP 服务器，使 AI 可以调用 `query` / `impact` / `detect_changes` / `rename` / `cypher` / `context` 等工具。
  - 后续在 Cursor / Claude Code 里，只需用自然语言对话即可触发这些工具。

**2）多仓库场景下的 repo 参数**

- 在 DICT 体系下通常会索引多个仓库（如 `simbest-boot-dict`、`simbest-cloud-dict-agent` 等），调用工具时**必须带上 `repo` 参数**：

```text
impact(target="UserClickLogSaveDto", direction="upstream", repo="simbest-boot-dict")
detect_changes(scope="staged", repo="simbest-cloud-dict-agent")
query(query="查询权限申请 BPM 回调", repo="simbest-boot-dict")
```

否则当 MCP 发现多个已索引仓库时，会返回“Multiple repositories indexed” 之类错误。

#### 1.6 快速验证是否工作正常

- 在 Cursor / Claude Code 中打开 DICT 项目，对 AI 发送类似问题：

```text
“列出当前已索引的代码仓库，并说明 simbest-cloud-dict-agent 中有哪些业务模块。”
```

- 预期：
  - AI 能正确列出至少一个已索引仓库（如 `simbest-cloud-dict-agent`、`simbest-boot-dict`）。
  - AI 的回答中会引用 `AGENTS.md` / `CLAUDE.md` 中的结构描述，并能说出模块划分和关键包路径。

---

### 第二章 开始使用：DICT 权限回收与查询稽核实战

本章选取 DICT 项目中**三个真实需求模块**，展示如何从原始需求出发，逐步优化“输入提示词 → AI 利用 GitNexus 工具 → 输出结果”，形成可复用的工作模板。

- **M01 离职人员权限处理**
- **M02 应用统计-查询权限稽核日志**
- **M03 应用统计-应用日志清单**

#### 2.1 模块 M01 离职人员权限处理

##### 2.1.1 原始业务需求

每日凌晨设置定时任务，扫描DICT系统生效的账号清单，针对不在生效清单的中的账号，保留“我的工作台（包括：待办、已办、待阅、已阅、草稿）”菜单权限，失效其他菜单权限，并将人员信息库的账号状态置为“失效”

##### 2.1.2 优化后的输入提示词

**改写思路**：将业务需求内容转换成对应的开发语言，明确详细细节。

```text
SELECT
    p.id
    FROM
        sys_user_permission       p
        LEFT JOIN v_user_org_only u
            ON p.username = u.username
    WHERE
          p.enabled = 1
      AND u.username IS NULL
      AND p.username != 'sys_epm';

请生成一个xxljob定时任务，每日凌晨3点执行。逻辑：查询目前系统内需要失效的人员菜单权限数据进行删除。使用批量处理，内存安全，线程安全。
```

##### 2.1.3 AI 结合 GitNexus 的回答示例（摘要）

- 通过输入以上内容，claude-code回答信息如下（节选整理）：

```text
完成。以下是本次改动说明：

---

变更内容

**`SysUserPermissionRepository.java`** — 新增2个方法

| 方法 | 说明 |
|------|------|
| `findOrphanedPermissionIds()` | 原始 SQL 查询孤儿权限 ID（用户不在 `v_user_org_only`） |
| `batchDeleteByIds(ids, modifier)` | 按 ID 批量逻辑删除，带 `@Modifying + @Transactional` |

**`MenuPermissionCleanupTask.java`** — 新建任务类

| 设计点 | 实现方式 |
|--------|----------|
| **内存安全** | 只查 ID 列表（String），不加载实体对象；`Lists.partition` 切成每批 200 条 |
| **线程安全** | 无共享可变状态，每次执行独立局部变量；Repository 方法为数据库原子操作 |
| **批次容错** | 单批次失败只记录 `failedBatches`，不中断后续批次；全部完成后按是否有失败批次决定 `handleSuccess/handleFail` |
| **操作日志** | 任务结束后写入一条 `operationType=清理` 的 `UserClickLog`，与现有日志体系对齐 |
| **任务标识** | Handler 名 `menuPermissionCleanupTask`，XXL-Job 控制台 Cron 配置为 `0 0 3 * * ?` |
```
可以看到很准确的完成了需求的实现


---

#### 2.2 模块 M02 应用统计-查询权限稽核日志

##### 2.2.1 原始业务需求

```text
当前系统已实现针对通过【DICT系统-系统管理-查询权限申请】功能申请的自定义报表权限每日稽核用户的在职状态、查询权限失效时间（包括是否存在权限申请记录，查询配置人员除外）、部门是否存在变动等，为确保稽核机制的有效性，系统需对稽核定时任务本身进行再次稽核
（1）查询权限日志：“应用统计”下新增菜单“查询权限稽核日志”，记录稽核任务执行情况，每次稽核任务执行完，无论成功或失败，都插入一条记录。字段包括“日志唯一ID、稽核任务名称、任务实际开始时间、任务实际结束时间、执行状态（成功/失败）、本次处理数据条数、失败原因、日志写入时间”，时间格式【YYYY/MM/DD HH24:MI:SS】。操作日志信息在线保存不少于1年。
（2）新增定时任务，每日检查稽核任务的执行日志，获取当前日期内该稽核任务的执行日志，判断是否有执行记录和执行状态，若没有当天的执行记录或执行状态失败，触发短信提醒给系统维护人员。
```


##### 2.2.2 优化后的输入提示词（带 GitNexus 背景）

- 指定仓库，直接将原始需求输入：

```text
repo simbest-boot-dict  当前系统已实现针对通过【DICT系统-系统管理-查询权限申请】功能申请的自定义报表权限每日稽核用户的在职状态、查询权限失效时间（包括是否存在权限申请记录，查询配置人员除外）、部门是否存在变动等，为确保稽核机制的有效性，系统需对稽核定时任务本身进行再次稽核
（1）查询权限日志：“应用统计”下新增菜单“查询权限稽核日志”，记录稽核任务执行情况，每次稽核任务执行完，无论成功或失败，都插入一条记录。字段包括“日志唯一ID、稽核任务名称、任务实际开始时间、任务实际结束时间、执行状态（成功/失败）、本次处理数据条数、失败原因、日志写入时间”，时间格式【YYYY/MM/DD HH24:MI:SS】。操作日志信息在线保存不少于1年。
（2）新增xxljob定时任务，每日9点检查稽核任务的执行日志，获取当前日期内该稽核任务的执行日志，判断是否有执行记录和执行状态，若没有当天的执行记录或执行状态失败，触发短信提醒给系统维护人员。
```

- GitNexus搜索代码后在理解结构后，将原始需求转换为开发语言，并开始实现需求：

```text
1. 系统已有两个定时稽核任务：
   - `QueryPermissionTask`（23:00）：检查查询权限失效状态。
   - `QueryPermissionCheckTask`（00:01）：检查 OA 账号变更导致的权限失效。
2. 新增需求：
   - 建表 `US_QUERY_AUDIT_TASK_LOG`，记录每次稽核任务执行情况：
     - 任务名称、开始/结束时间、执行状态（成功/失败）、处理数据条数、失败原因、日志写入时间等。
   - 增加前端菜单“查询权限稽核日志”，支持按任务/状态查询。
   - 新增 XXL-Job 监控任务：
     - 每日 9:00 执行，回溯过去 24 小时的稽核执行记录。
     - 若无执行记录或存在失败记录，发送短信给运维人员。
```

##### 2.2.3 AI 结合 GitNexus 的回答示例（实现情况说明）

所有文件已创建完毕，以下是完整汇总：

---
```text
## 实现说明

### 新增文件

| 文件 | 说明 |
|------|------|
| `task/queryPermission/model/QueryAuditTaskLog.java` | 稽核日志实体，表 `US_QUERY_AUDIT_TASK_LOG`，ID前缀 `QATL` |
| `task/queryPermission/repository/QueryAuditTaskLogRepository.java` | Repository，含按任务名+时间查询方法 |
| `task/queryPermission/service/IQueryAuditTaskLogService.java` | Service 接口 |
| `task/queryPermission/service/impl/QueryAuditTaskLogServiceImpl.java` | Service 实现 |
| `task/queryPermission/web/QueryAuditTaskLogController.java` | Controller，路径 `/action/queryAuditTaskLog`，供"应用统计→查询权限稽核日志"菜单调用 |
| `task/queryPermission/QueryAuditMonitorTask.java` | XXL-Job 监控任务，Handler名 `QueryAuditMonitorTask` |
| `src/main/resources/sql/query_audit_task_log.sql` | 建表DDL + 索引 + 菜单配置参考SQL |

### 修改文件

**`QueryPermissionTask.java`**（查询权限失效稽核，23:00）
- 注入 `IQueryAuditTaskLogService`
- `execute()` 重构：try-catch-finally 包裹，finally 中写入 `taskName="查询权限失效稽核"` 的日志

**`QueryPermissionCheckTask.java`**（OA变动稽核，00:01）
- 注入 `IQueryAuditTaskLogService`
- `execute()` 重构：原业务逻辑提取为 `processOAChangedUsers()`，finally 中写入 `taskName="查询权限OA变动稽核"` 的日志

### 监控任务逻辑（QueryAuditMonitorTask）

- XXL-Job Cron 配置：`0 0 9 * * ?`（每日9:00）
- 向前查 **24小时**内的执行日志，覆盖23:00和00:01两个执行窗口
- 对每个受监控任务：无记录 → 告警；有"失败"状态 → 告警（附失败原因）
- 告警通过 `ISmsService.sendMessageToManager()` 发送，非生产环境自动跳过

### XXL-Job 后台配置

在 XXL-Job 控制台新增任务：
- **执行器**：当前应用执行器
- **JobHandler**：`QueryAuditMonitorTask`
- **Cron**：`0 0 9 * * ?`
- **描述**：查询权限稽核任务执行状态监控（每日9点检查，失败/缺失则短信告警）
```

#### 2.3 模块 M03 应用统计-应用日志清单

##### 2.3.1 原始需求输入给 AI 的内容

这次没有调整输入内容，测试直接输入需求原文测试实现需求准确率

```text
在应用日志清单中增加“查询权限申请、菜单权限申请”2类权限申请的操作日志数据，
记录权限“申请、审批、生效、失效、清理”等操作日志：
1）操作人OA账号/姓名（若是系统执行的，记录系统账号）；
2）操作类型（新增枚举值“申请、审批、生效、失效、清理”，记录从提交申请到权限生效、失效的所有动作）；
3）菜单ID（新增字段，若是菜单，记录菜单ID）；
4）查询编号（新增字段，若是查询权限，记录查询编号）。
```

##### 2.3.3 AI 第一次响应的实现摘要（申请 / 失效 / 清理 + 前端展示）
```text
  - DTO 扩展：
    - 在 `UserClickLogSaveDto.java` 中新增字段：
      - `menuId`（菜单ID）— 菜单权限申请日志使用；
      - `queryCode`（查询编号）— 查询权限申请日志使用；
    - 将数据库列 `menu_id` / `query_code` 映射到前端 VO（`convertMapToVO` 更新）。
  - 控制器覆盖：
    - `QueryPermissionController.create()`：
      - 在创建查询权限记录后，调用 `userClickLogService.saveLog(dto)` 记录 **申请** 日志；
      - `queryCode` 取新建记录的 ID 或配置编号。
    - `QueryPermissionController.deleteById()`：
      - 在删除查询权限记录前/后记录 **清理** 日志；
      - `queryCode` 取被删除记录的 ID。
  - 定时任务注入：
    - `QueryPermissionTask.execute()`：
      - 在已有 `QueryPermissionBusinessLog` 写入之后，额外调用 `pimUserLogService.saveUserClickLog()` 记录 **失效** 日志；
      - `username = "system"`、`truename = "系统自动处理"`；
      - `queryCode` 取配置编号 `configNumber`。
  - 前端页面：
    - `sysUserClickLogQuery.html`：
      - `operationType` 格式化函数增加新枚举：申请 / 审批 / 生效 / 失效 / 清理；
      - 新增两列展示：
        - **菜单ID**（`menuId`）
        - **查询编号**（`queryCode`）。
  - 数据库侧：
    - `us_user_click_log` 表新增两列 `menu_id`、`query_code`（例如 VARCHAR2 类型），以便 PIM 侧写入日志、DICT 侧查询展示。

##### 2.3.4 补充追问：审批 / 生效 由流程驱动，需要 BPM 回调注入
```
实现准确度很准确，继续提问，完善审批、生效节点
```text
审批 和 生效 两个操作节点是由流程驱动的，通过查找对应的流程表单处理方法补充完整。
```

##### 2.3.5 AI 第二次响应的实现摘要（审批 / 生效 注入点）
```text
  - 关键回调方法：`ApplicationFormEnhanceServiceImpl.saveBusiness_Q_X_Y()`
    - 负责处理 QX（市级查询权限）和 QY（省级查询权限）两类查询权限流程的所有 BPM 回调。
  - **生效日志**：
    - 触发条件：`outcome == "dict.toStart"`（应用管理员审批通过，权限真正授予用户）；
    - 注入位置：授权用户写入子表循环之后，`dict.toStart` 分支块末尾；
    - 日志参数：`queryCode = queryPermission.getId()`，`operationType = 生效`，其他字段按统一规范填写。
  - **审批日志**：
    - 触发条件：`outcome` 不为 `"return"`（退回）且不为 `"dict.toStart"`（生效已单独记录）；
    - 覆盖的审批节点（示例）：
      - `dict.startToDepartmentDeputyManager/Leader/Manager` — 部门领导审批；
      - `dict.innovateManagerToIctAdmin` — ICT 管理员审批；
      - `dict.managerAdminToInnovateManager` — 创新部经理审批；
      - `end` — 审批流程结束；
    - 注入位置：`iQueryPermissionService.update/insert` 之后，结合当前 `outcome` 写入 **审批** 日志。

##### 2.3.6 M03 模块下“权限申请全生命周期日志”的最终覆盖矩阵

| 操作类型 | 触发点 | 说明 |
|----------|--------|------|
| 申请     | `QueryPermissionController.create()` | 用户提交查询/菜单权限申请时记录 |
| 审批     | `ApplicationFormEnhanceServiceImpl.saveBusiness_Q_X_Y()` 中 `outcome ≠ return/toStart` | 每个审批节点回调时记录 |
| 生效     | 同上，`outcome == "dict.toStart"` | 权限正式授予用户时记录 |
| 失效     | `QueryPermissionTask.execute()` | 定时稽核任务将权限置为失效时记录 |
| 清理     | `QueryPermissionController.deleteById()` | 管理员删除权限记录时记录 |
```
##### 2.3.7 使用 GitNexus 的价值点总结（针对 M03）
  - 可以看到通过原始的需求描述，还是可以很精准的找到对应的代码，可以完成需求实现。最后建议完成需求实现后输入：*目前的代码改动会影响哪里，给出影响分析* 这样可以清晰的看到全部修改，以及影响范围。



````md
现在有了完整的分析数据，给出系统化的影响分析报告。

---

## 影响分析报告

### 一、`UserClickLogSaveDto` — 新增 `menuId` / `queryCode` 字段

**调用方总数：14 个文件**，分以下三类：

| 类型 | 文件 | 影响 |
|------|------|------|
| **写入方**（saveLog） | `ApplicationFormEnhanceServiceImpl`、`ApplicationFormServiceImpl`、`ApplicationSignedFormServiceImpl`、`ApplicationFormGatherServiceImpl`、`IApplicationFormCommonServiceImpl`、`ApplicationFormProjectServiceImpl`、`QueryPermissionController`、`QueryPermissionTask` 等 | 新字段默认值为 `null`，Lombok `@Data` 自动生成 setter，**现有调用不受影响** |
| **读取方**（查询/导出） | `IUserClickLogServiceImpl.findAllList()` → `SELECT * FROM us_user_click_log` → `convertMapToVO` | `getStringIgnoreCase` 查不到列时返回 `null`，**不抛异常**。但若数据库表无 `menu_id`/`query_code` 列，新字段永远为空 |
| **Feign 序列化** | `PimLogClient.saveLog(UserClickLogSaveDto)` | Jackson 将新字段序列化进 JSON body。**PIM 接收端若无对应字段，取决于其是否配置 `@JsonIgnoreProperties`**；若未配置且严格校验则报错 |

**Excel 导出副作用**：`@ExcelProperty(value = "菜单ID")` 和 `@ExcelProperty(value = "查询编号")` 会在导出文件中自动追加两列。无固定模板映射时，**新增两列展示**，不破坏现有列。

**结论：本地代码侧无破坏性影响。主要外部依赖风险在 PIM 服务是否接受新字段，以及 `us_user_click_log` 数据库表是否已有对应列。**

---

### 二、`QueryPermissionController.create` — 覆盖父类 `create`

```
调用链：前端表单提交 → POST /action/queryPermission/create → create() [覆盖] → super.create() → service.insert()
```

- 字节码确认父类 `create(T)` 签名一致，`@Override` **无编译风险**
- 执行顺序：`super.create(o)` 先完成，日志写入在其之后，**接口响应结果不受影响**
- `saveLog` 为 `@Async`，异步执行，**不增加接口响应时延**
- 整个日志逻辑在 `try-catch` 内，日志失败不会向前端抛错

**结论：无破坏性影响，申请接口行为与原来完全一致。**

---

### 三、`QueryPermissionController.deleteById` — 覆盖父类 `deleteById`

字节码分析结果：

```
RuntimeVisibleParameterAnnotations:
  parameter 0: @RequestParam   ← 父类原本就有 @RequestParam
```

父类签名为 `public JsonResponse deleteById(@RequestParam PK)`，与我们的覆盖 `deleteById(@RequestParam String id)` **完全一致**，`@PostMapping({"/deleteById", "/sso/deleteById", "/api/deleteById"})` 映射也继承自父类，不需要重复声明。

- 执行顺序：先记录日志，**再调用 `super.deleteById(id)`**（即使日志失败，删除仍执行）
- 日志在 `try-catch` 内，**不阻断删除操作**

**结论：签名匹配正确，无任何风险。**

---

### 四、`QueryPermissionTask.execute` — 定时任务新增日志

```
调用链：Spring Scheduler (cron: dayQian) → checkAndExecute() → super.checkAndExecute(true) → execute()
```

- `PimUserLogService` 已在多处 Bean 中注入，**无循环依赖风险**
- 日志调用时 `username="system"` 非空，`PimUserLogService.saveUserClickLog()` 不会调用 `SecurityUtils.getCurrentUser()`，**无 Spring Security 上下文问题**
- 日志写入为**同步调用**（定时任务已在独立线程中，同步合理），在 try-catch 内保护
- 原有 `QueryPermissionBusinessLog` 的业务日志逻辑**完全不变**

**结论：定时任务行为不变，新增日志失败不影响权限失效处理的核心逻辑。**

---

### 五、`ApplicationFormEnhanceServiceImpl.saveBusiness_Q_X_Y` — BPM 回调注入日志

```
调用链：POST /action/applicationFormEnhance/startSubmitProcess
           → startSubmitProcess()
               → switch(processType):
                   case PROCESS_Q_X → saveBusiness_Q_X_Y(..., "QX")
                   case PROCESS_Q_Y → saveBusiness_Q_X_Y(..., "QY")
```

两处注入位置：

| 注入点 | outcome 条件 | 风险 |
|--------|-------------|------|
| **生效日志**（L6743） | `dict.toStart` 块末尾，授权用户写入之后 | `iUserClickLogService` 已注入（L1031），try-catch 保护 |
| **审批日志**（L6777） | `!return && !dict.toStart`，update/insert 之后 | 同上，且仅在非退回/非生效时触发，覆盖所有中间审批节点 |

- `saveLog` 为 `@Async`，**不阻塞 BPM 回调响应**
- `saveBusiness_Q_X_Y` 同时服务 QX（市级）和 QY（省级）两条流程，**两条流程均生效**
- BPM 回调方法本身 `try-catch` 包裹整个 `saveBusiness_Q_X_Y` 调用，双重保护

**结论：BPM 流程核心逻辑完全不变，日志注入对回调结果无任何影响。**

---

### 六、`sysUserClickLogQuery.html` — 前端展示变更

- 表格新增 **"菜单ID"** 和 **"查询编号"** 两列（字段为 `menuId`、`queryCode`）
- `operationType` 格式化函数扩展，原有 `CLICK/EXPORT/QUERY/PROCESS-START` 映射**不变**，新增 `申请/审批/生效/失效/清理`
- 旧数据 `operationType` 无法匹配新枚举时走 `return value || ""`，**直接展示原始值，不报错**

**结论：纯前端变更，无任何后端影响，向下兼容旧数据。**

---

### 汇总

| 改动 | 风险等级 | 核心结论 |
|------|----------|---------|
| `UserClickLogSaveDto` 加字段 | **中**（外部依赖） | PIM 表结构与服务端需配合新增列；本地逻辑无破坏 |
| `QueryPermissionController.create` 覆盖 | **低** | try-catch + async，不影响接口行为 |
| `QueryPermissionController.deleteById` 覆盖 | **低** | 父类注解确认一致，签名正确 |
| `QueryPermissionTask` 新增日志 | **低** | 无 Security 上下文问题，try-catch 保护 |
| `saveBusiness_Q_X_Y` 注入日志 | **低** | 双重 try-catch，async 不阻塞 BPM |
| `sysUserClickLogQuery.html` | **低** | 纯展示，旧数据兼容 |

**唯一需要在部署前确认的外部动作**：联系 PIM 侧在 `us_user_click_log` 表中新增 `menu_id VARCHAR2(256)` 和 `query_code VARCHAR2(256)` 两列，并确认 PIM 的 `saveLog` 接口能接受 JSON 中的未知字段（通常 Spring Boot 默认忽略，无需改动）。
````




---

### 第三章 最佳实践与常见问题（FAQ）

#### 3.1 DICT 项目使用 GitNexus 的最佳实践

- **改动前：先看影响半径**
  - 使用 GitNexus MCP 工具前，建议 AI 先执行：
    - `impact(target="目标类或方法", direction="upstream", repo="...")`
    - 或请 AI 总结“d=1/d=2/d=3 调用方及影响的执行流”。
  - 在权限相关改动（如查询/菜单权限、审计日志、稽核任务）中，**必看 d=1 调用方**。

- **提交前：确认改动范围**

```text
“请在 repo=simbest-boot-dict 中运行 detect_changes(scope='staged')，
帮我列出这次提交涉及的文件、受影响的执行流，以及可能需要额外回归测试的功能点。”
```

- **索引新鲜度检查**
  - 当出现以下信号之一时，应优先怀疑索引已过期：
    - 明显改动了核心业务类，但 `impact` 显示“无调用方”。
    - 明显影响流程的改动，但 `detect_changes` 报告中 `affected_processes` 为空。
  - 建议命令：

```bash
gitnexus status         # 查看当前项目索引状态
gitnexus analyze        # 或 gitnexus analyze --embeddings
```

#### 3.2 提示词撰写经验（DICT 权限场景）

- **尽量带上 repo 信息和业务关键字**：
  - ✓ “repo=simbest-boot-dict，查询权限申请 BPM 回调 审批流程”
  - ✗ “帮我找审批流程在哪”

- **分步提问，先结构后改造**：
  - 第一步先问“有哪些执行流 / 类 / 方法”，借助 `query` / `context`。
  - 第二步再问“在这些节点上如何注入日志 / 审计 / 告警”。

- **让 AI 明确使用 GitNexus 工具**：
  - 明确要求“先用 GitNexus 的 query/context/impact 工具给出结构，再基于结果设计方案”，可以显著提高回答准确度。

#### 3.3 GitNexus 常见问题（结合 DICT 场景）

- **问题 1：`detect_changes` / `impact` 结果看起来不对？**
  - 排查步骤：
    - 先检查 `gitnexus status`，确认索引是否过期。
    - 必要时执行 `gitnexus analyze --force` 或 `gitnexus analyze --embeddings` 重新索引。
    - 对明显关键改动，可让 AI 同时结合 `grep`/IDE 全局搜索做一次人工交叉验证。

- **问题 2：`rename` 能否重命名 REST URL 或前端 JS 中的字符串？**
  - 目前 `rename` 主要针对“代码符号”（类/方法名等），字符串字面量中的 URL、菜单编码等仍需人工复核。
  - 在 DICT 系统中，**权限编码 / 菜单编码** 相关改动尤其要谨慎，建议：
    - 先用 `query` 找出与该编码相关的前后端调用，再做人工替换。

- **问题 3：定时任务中访问 Security 上下文导致 NPE？**
  - 定时任务一般在独立线程中执行，通常没有 HTTP 请求上下文。
  - 建议：
    - 在日志 DTO 中显式设置 `username`（例如 `"system"`），避免依赖 ThreadLocal 中的用户信息。
    - 如确需区分“任务来源用户”，则通过业务参数显式传入。


---
### 第四章 团队协作

> 目标：把 GitNexus 从“个人增强插件”升级为“DICT 团队统一的开发基础设施”，在多人、多仓库、多环境下保持一致的使用方式和质量标准。

> 🎯 **核心问题**：GitNexus 会在项目中生成哪些文件？哪些该提交到 Git，哪些不该提交？

#### 4.0 生成文件清单

| 文件/目录 | 位置 | 大小 | 作用 | 是否提交 Git？ |
|----------|------|------|------|--------------|
| **`.gitnexus/`** | 项目根目录 | 50MB - 500MB | 索引数据库（KuzuDB） | ❌ **不提交** |
| **`.claude/skills/`** | 项目根目录 | ~20KB | AI 技能文件（4个） | ✅ **建议提交** |
| **`AGENTS.md`** | 项目根目录 | 5KB - 50KB | 项目结构说明 | ✅ **建议提交** |
| **`CLAUDE.md`** | 项目根目录 | 1KB - 5KB | AI 上下文配置 | ✅ **建议提交** |

---

##### 4.0.1 详细说明

###### ❌ `.gitnexus/` — 不提交（本地索引数据库）

**为什么不提交？**
1. **体积大**：根据项目规模，索引文件可能有 50MB - 500MB
2. **个人化**：每个开发者的索引可能因分支、本地修改而不同
3. **易重建**：运行 `gitnexus analyze` 即可重建，无需共享

**如何忽略？**
```bash
# 在 .gitignore 中添加（GitNexus 会自动添加）
.gitnexus/
```

**团队协作建议**：
- 每个开发者在本地运行 `gitnexus analyze` 建立自己的索引
- 首次索引需要 2-5 分钟，后续增量更新只需几秒



#### 4.1 团队引入 GitNexus 的统一流程

##### 4.1.1 新成员加入流程（Onboarding）

- **步骤 1：环境准备与全局安装**
  - 安装 Node.js（推荐 LTS，≥ 18）。
  - 全局安装 GitNexus 并验证版本：

  ```bash
  npm install -g gitnexus
  gitnexus --version
  ```

- **步骤 2：配置编辑器集成（Cursor / Claude Code / Claude Desktop）**

  ```bash
  gitnexus setup        # 自动注册 GitNexus MCP 服务器
  ```

  - 成功后，AI 助手可以调用 `query` / `impact` / `detect_changes` / `rename` / `cypher` / `context` 等工具。

- **步骤 3：克隆 DICT 相关仓库并建立索引**
  - 根据岗位克隆必要仓库，例如：
    - `simbest-boot-dict`
    - `simbest-cloud-dict-agent`
    - 其它与权限/稽核相关的 PIM / UUMS 仓库。
  - 在每个仓库根目录执行：

  ```bash
  gitnexus analyze
  ```

  - 首次分析会创建：
    - `.gitnexus/`：索引数据库，**不提交 Git**（需加入 `.gitignore`）
    - `.claude/skills/`：GitNexus 技能文件（建议提交）
    - `AGENTS.md`、`CLAUDE.md`：项目结构与 AI 上下文说明（建议提交）

- **步骤 4：验证 GitNexus 是否生效**
  - 在 Cursor / Claude Code 中打开项目，提问示例：

  ```text
  repo=simbest-boot-dict，
  帮我梳理“查询权限申请”的完整执行流和关键类。
  ```

  - 预期：
    - 回答里能看到按执行流分组的调用链；
    - 会引用真实类/方法（`QueryPermissionController`、`saveBusiness_Q_X_Y` 等）；
    - 工具调用记录中出现 `gitnexus_query` / `gitnexus_context` / `gitnexus_impact` 等。

##### 4.1.2 新项目 / 新模块接入流程

- 在新建 DICT 子模块或新仓库时，骨架和基础依赖就绪后，执行一次：

  ```bash
  gitnexus analyze
  ```

- 然后对生成内容进行团队化整理：
  - **`AGENTS.md`**：
    - 补充模块职责、目录结构、关键执行流；
    - 标出核心入口（Controller / Service / Job）。
  - **`CLAUDE.md`**：
    - 增加“可用 GitNexus 工具”“推荐提示词”等说明；
    - 链接到本培训手册或内部 Wiki。
  - 将 `.claude/skills/`、`AGENTS.md`、`CLAUDE.md` 纳入版本控制，供团队共享。
  - 确认 `.gitnexus/` 已在 `.gitignore` 中，遵循原 GitNexus 文档的建议。

##### 4.1.3 索引更新与团队广播

- **适合广播的场景**：
  - 大型重构（包结构调整、模块拆分、类迁移）合入主干；
  - 新增重要业务模块（权限域、稽核、日志等）；
  - 更新了 `AGENTS.md` / `CLAUDE.md` 中的整体结构说明。

- **推荐广播模板**：

  ```text
  【GitNexus 配置更新通知】
  仓库：simbest-boot-dict
  
  变更内容：
  1）更新 AGENTS.md，补充“查询权限申请 / 稽核 / 日志”模块说明；
  2）在 CLAUDE.md 中新增“DICT 权限场景推荐提示词”章节；
  3）完成 gitnexus analyze --embeddings，索引已更新到最新代码。
  
  建议操作（开发同学）：
  1）在本地执行 git pull；
  2）在本地仓库执行 gitnexus analyze；
  3）在 Cursor 中尝试使用 repo=simbest-boot-dict + 业务关键字提问，确认效果。
  ```

---

#### 4.2 角色分工与责任界面

> 结合 GitNexus 官方文档中的“索引维护”“生成文件管理”等建议，对 DICT 团队常见角色进行职责约定。

##### 4.2.1 架构师 / 模块负责人

- **职责**：
  - 维护“必须被 GitNexus 索引”的仓库清单；
  - 维护 `AGENTS.md` 的高层架构描述（模块边界、关键执行流）；
  - 为高风险领域（权限、稽核、日志、任务）制定 GitNexus 使用规范。
- **关键动作**：
  - 在大规模重构前后统一执行：

  ```bash
  gitnexus analyze --embeddings
  gitnexus status
  ```

  - 对涉及 M01/M02/M03 等关键模块的 MR，要求提供 GitNexus 影响分析摘要。

##### 4.2.2 普通开发

- **日常要求**：
  - **改动前**：用 `impact` / `context` 了解关键类/方法的影响范围；
  - **改动中**：对不清楚的调用链使用 `query` / `context` 梳理执行流；
  - **提交前**：用 `detect_changes(scope='staged')` 确认改动范围是否合理。
- **建议在 MR 中增加简要摘要**（特别是 M01/M02/M03）：

  ```text
  【GitNexus 分析摘要】
  repo=simbest-boot-dict
  impact(target="QueryPermissionTask.execute", direction="upstream"):
  - d=1：仅 QueryPermissionTask 自身及 QueryAuditMonitorTask 依赖，均在本次 MR 中变更；
  - d=2：影响“查询权限稽核日志”菜单的数据来源，无其他执行流被标记。
  detect_changes(scope="staged"):
  - changed_files：4 个，均为 M02 模块相关文件。
  ```

##### 4.2.3 代码审核人（Reviewer）

- **关注点**：
  - `impact`：
    - d=1 调用方是否都在 MR 涉及范围；
    - 是否存在 HIGH / CRITICAL 风险且缺少说明。
  - `detect_changes`：
    - 是否有与本次需求无关的模块被修改；
    - `affected_processes` 是否与需求描述的执行流一致。
- **典型提问**：

  ```text
  “你这次改了 QueryPermissionController 和 saveBusiness_Q_X_Y，
  impact 报告里的 d=1/d=2 调用方是否都已经处理？
  detect_changes 报告里是否存在与权限模块无关的意外改动文件？”
  ```

##### 4.2.4 运维 / SRE

- **使用场景**：
  - 评估监控点是否覆盖关键执行流（M01/M02/M03）；
  - 故障/告警时，利用 GitNexus 追溯上游/下游依赖。
- **示例提示词**：

  ```text
  repo=simbest-boot-dict，
  帮我用 GitNexus 查一下：
  1）QueryAuditMonitorTask.execute() 的所有上游调用方；
  2）它参与了哪些执行流；
  3）如果这个任务失败，会影响哪些“应用统计”相关页面。
  ```

---

#### 4.3 Code Review 与 GitNexus 的结合规范

##### 4.3.1 开发自检清单（提交前）

- 以下改动类型，建议“提交前必须用 GitNexus 自检一次”：
  - 权限（查询/菜单/角色）相关逻辑；
  - 日志/审计/稽核（M02、M03）相关实体和服务；
  - 定时任务 / XXL-Job；
  - 跨模块 DTO / Entity 字段变更。
- **推荐自检步骤**：

  ```text
  1）impact(target="关键类/方法", direction="upstream", repo="...")：
     - 确认 d=1/d=2 调用方列表是否符合预期；
  2）detect_changes(scope="staged", repo="...")：
     - 确认 changed_files / affected_processes 与需求一致；
  3）若分析明显不准确，执行 gitnexus analyze 后重新获取报告。
  ```

##### 4.3.2 Reviewer 关注的报告字段

- **`impact` 报告**：
  - d=1 是否全部覆盖在 MR 修改范围；
  - HIGH / CRITICAL 风险是否有详细说明与测试计划。
- **`detect_changes` 报告**：
  - 是否存在“顺手改了但没说明”的文件；
  - `affected_processes` 是否覆盖了需求相关流程，是否意外影响其他流程。

##### 4.3.3 高风险改动（HIGH / CRITICAL）的额外流程

- 当 `impact` 显示 HIGH / CRITICAL 风险时：
  - MR 描述中增加“小节：GitNexus 高风险分析与测试计划”；
  - 明确列出：
    - 受影响执行流列表；
    - 已做/计划做的测试项（单测/集成/回归）；
    - 风险缓解措施（灰度、监控、回滚方案）。
- 对生产关键路径（M01 离职权限回收、M02 稽核任务、M03 应用日志清单）：
  - 建议架构师/模块负责人参与 Review；
  - 合入后配合运维设置短期观察窗口和重点监控。

---

#### 4.4 跨仓库协作与多项目索引管理

##### 4.4.1 统一 repo 命名与使用规范

- GitNexus 多仓库模式下，DICT 团队约定统一的仓库名，如：
  - `simbest-boot-dict`
  - `simbest-cloud-dict-agent`
  - `simbest-cloud-dict-pim`
- **所有提示词必须显式带上 repo**，避免“Multiple repositories indexed” 错误：

  ```text
  repo=simbest-boot-dict，查询权限申请 BPM 回调 审批流程
  repo=simbest-cloud-dict-pim，菜单权限申请 执行流 和 权限回收
  ```

##### 4.4.2 典型跨仓库分析流程

- 典型场景：菜单权限在 PIM，查询/稽核/日志在 DICT，希望有“端到端视角”。
- 推荐提示词流程：

  ```text
  第一步：
  repo=simbest-cloud-dict-pim，
  梳理菜单权限申请/审批/生效/失效的执行流（入口 Controller / Service / Job）。
  
  第二步：
  repo=simbest-boot-dict，
  梳理查询权限申请/稽核/日志记录的执行流。
  
  第三步：
  基于上面两部分结果，帮我用文字串联出从 OA/用户操作到权限最终生效/失效的跨系统链路，
  并标出可以加监控/审计日志的关键节点。
  ```

##### 4.4.3 索引生命周期与清理

- 日常通过 `gitnexus analyze` 做增量更新；
- 当仓库不再维护/需要重置时，可使用：

  ```bash
  gitnexus clean          # 删除当前项目索引
  gitnexus clean --all    # 删除所有项目索引（谨慎使用）
  ```

- 团队层面，可定期执行：

  ```bash
  gitnexus list
  gitnexus status
  ```

  检查索引健康，清理无用索引。

---

#### 4.5 培训与知识沉淀

##### 4.5.1 提示词与案例库建设

- 在内部 Wiki / 知识库维护“GitNexus 提示词与案例库”，建议结构：
  - **通用提示词**：探索、影响分析、重构、调试等；
  - **DICT 专用提示词**：按模块（M01/M02/M03、schemeMatch、lostbid 等）分类；
  - **案例条目**包含：
    - 原始需求；
    - 初始提问与问题点；
    - 优化后提示词；
    - GitNexus 分析结果摘要；
    - 最终改动与上线效果。

##### 4.5.2 制度化落地

- 在开发规范 / Code Review 指南 / PR 模板中加入 GitNexus 相关条款：
  - 对高风险改动（权限、稽核、日志、任务）建议或要求附带 GitNexus 分析摘要；
  - 对关键仓库，要求本地必须保持 GitNexus 索引可用（`gitnexus status` 正常）。
- 在各关键项目 `README` / `CLAUDE.md` 中：
  - 明确标注“已集成 GitNexus，建议优先通过其进行结构理解与影响分析”；
  - 链接到本培训手册和内部案例库，作为新成员必读材料。