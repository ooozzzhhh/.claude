---
name: code-implementation-research
description: 产品经理调研代码实现现状与数据库表结构的标准化流程。在产品 AI 工作流中执行现状调研时调用本 skill；覆盖模块结构、技术栈、代码查找路径、完成标准、调研结论结构与数据库调研约定。调研结论直接写入当前需求的 PRD 现状章节或问题分析报告，不再单独维护跨需求的产品知识库。
---

# 实现现状调研 Skill

产品经理在执行「现状调研」时，按本 skill 执行，确保调研充分、报告清晰、无信息缺失。

调研结论的最终落点是当前需求的 PRD 现状章节或问题分析报告，本流程**不再单独维护跨需求的产品知识库**。

---

## 1. 产品模块结构（SCP Foundation）

SCP Foundation 是供应链计划平台，采用微服务架构。模块与端口对应关系：

| 模块 | 服务名 | 端口 | 说明 |
|------|--------|------|------|
| IPS | scp-ips-service | 8761 | Integration Platform Service |
| MDS | scp-mds-service | 8762 | Master Data Service |
| SDS | scp-sds-service | 8763 | Supply-Demand Data Service |
| DFP | scp-dfp-service | 8764 | Demand Forecasting & Planning |
| SOP | scp-sop-service | 8765 | Sales & Operations Planning |
| MPS | scp-mps-service | 8766 | Master Production Scheduling |
| MRP | scp-mrp-service | 8767 | Material Requirements Planning |
| ODS | scp-ods-service | 8768 | Order Delivery Scheduling |
| AMS/SCH | scp-ams-service | 8769 | Advanced Manufacturing Scheduling |
| WFP | scp-wfp-service | 8770 | Workforce Planning |
| CTS | scp-cts-service | 8771 | Control Tower Service |
| DCP | scp-dcp-service | 8772 | Data Exchange Service |
| UBA | scp-uba-service | 8773 | User Behavior Analytics |
| DAS | scp-das-service | 8788 | Deloris Algorithm Service |

**模块命名约定**：S&OP 统一用 SOP；AMS 模块统一用 SCH（禁止用 AMS 作为模块目录名）。

---

## 2. 技术栈与代码项目结构

### 2.1 代码根目录

- **工作区根**：本仓库（UHA）根目录；绝对路径随本机克隆位置而定
- **代码仓库根**：`scp-foundation/`（相对工作区根，Maven 多模块项目根目录）
- 若主会话或需求文档指定其他路径/分支，以指定为准。

### 2.2 Maven 模块分层（scp-{service}-{layer}）

每个业务模块通常包含以下子模块：

| 层级 | 模块名模式 | 职责 | 关键内容 |
|------|------------|------|----------|
| api | scp-{service}-api | 接口定义、DTO、常量 | 接口、枚举、请求/响应对象 |
| domain | scp-{service}-domain | 领域模型（部分模块有） | 实体、聚合、值对象 |
| sdk | scp-{service}-sdk | 业务逻辑、控制器、服务实现 | Controller、Service、Mapper、批任务 |
| service | scp-{service}-service | Spring Boot 应用入口 | 启动类、配置 |

**说明**：IPS、DCP 等部分模块无 domain 层，业务逻辑集中在 sdk。

### 2.3 代码查找路径与上下游关联

| 调研目标 | 查找路径 | 关联上下游 |
|----------|----------|------------|
| REST 接口 | `scp-{service}-sdk/.../controller/` | → Service → Mapper/Repository |
| 应用服务 | `scp-{service}-sdk/.../service/impl/` | ← Controller；→ Domain/Mapper |
| 领域服务 | `scp-{service}-domain/.../domain/service/` | ← 应用服务 |
| 实体/DO | `scp-{service}-domain/.../entity/` 或 sdk 内 | ← Mapper；→ 表结构 |
| Mapper/DAO | `scp-{service}-sdk/.../mapper/`、`*Dao.java`、`*Dao.xml` | ← Service；→ 表 |
| 批任务/定时任务 | 搜索 `@Scheduled`、`Job`、`Task` | → 调用的 Service |
| 枚举 | `scp-{service}-api/.../enums/` 或 sdk 内 | 被 Controller/Service 引用 |

**查找策略**（在 Claude Code 中使用 `Glob` 和 `Grep` 工具）：
1. 从需求关键词（如「接口日志」「接口配置」）出发，用 `Grep` 在对应模块 sdk 中搜索 Controller、Service 类名或路径。
2. 从 Controller 的 `@RequestMapping`、方法入参/出参，顺藤摸瓜到 Service、Mapper（用 `Read` 阅读相关文件）。
3. 从实体/DO 的 `@Table`、字段注解，反推表名与字段；结合 Mapper XML 中的 SQL 确认表结构。
4. 若涉及跨模块（如 IPS 调 DCP），在 api 层查找 feign 或 http 客户端调用。

### 2.4 前端代码（若需求涉及）

- 前端工程可能位于 `scp-front/` 或项目内其他目录；路由、页面组件、API 调用通常在 `src/router/`、`src/views/`、`src/api/` 下。
- 需求文档或主会话会指明前端工程路径；未指明时在回答中说明「前端未纳入本次调研范围」。

---

## 3. 调研完成标准与结束条件

**禁止在以下任一条件未满足时结束调研**：

| 序号 | 条件 | 说明 |
|------|------|------|
| 1 | 入口已定位 | 已找到与需求主题直接相关的 Controller/API 或批任务入口 |
| 2 | 调用链已梳理 | 从入口到数据落库/出库的主流程调用链已理清（含跨模块调用） |
| 3 | 数据模型已明确 | 核心实体/DO、表名、主键及关键字段已识别；若无法直连 DB，须基于 Mapper/实体做合理推断并标注「待实库确认」 |
| 4 | 业务规则已归纳 | 状态机、校验规则、幂等/去重、归档策略等与需求相关的规则已总结 |
| 5 | 缺口已标注 | 代码不可见、分支不可用、DB 不可连等限制已显式写出，并说明对结论的影响 |
| 6 | 上下游已覆盖 | 若需求涉及数据流入/流出，上游数据来源与下游消费方已识别 |

**自检清单**（结束前逐项确认）：
- [ ] 需求文档中的核心概念（如「接口日志」「中间表」「场景库」）是否都在代码/表中找到对应实现？
- [ ] 是否存在「推测」「可能」「未找到」等模糊表述？若有，是否已标注为待确认？
- [ ] 数据库表结构是否已调研（直连或基于 Mapper/实体推断）？若无法获取，是否已说明原因？

---

## 4. 调研结论输出结构

调研结论须**清晰、结构化**，直接写入当前需求的 PRD 现状章节或问题分析报告。建议按以下要点组织（可根据实际简略，但不删核心要素）：

### 4.1 现状概述
- 用若干要点说明当前系统在该主题下已具备的能力、主要流程或核心表/服务。

### 4.2 代码实现
- **入口与路由**：Controller 路径、主要接口、批任务入口。
- **关键类/接口/方法**：职责说明，调用关系或关键时序。
- **数据流**：从请求/触发到落库/出库的路径。

### 4.3 数据库表结构
- **核心表**：表名、用途、主键、关键字段（类型、含义、约束）。
- **主外键关系**：表间关联。
- **重要索引与约束**：影响查询与唯一性的设计。
- **若无法直连 DB**：基于 Mapper/实体推断，并标注「基于代码推断，待实库确认」。

### 4.4 业务规则与约束
- 校验规则、状态机、计算逻辑、幂等/去重策略、数据保留/归档规则等。

### 4.5 与需求相关的要点
- 本次调研支持的需求名称。
- 当前实现中**有助于实现需求**的点。
- 当前实现中**构成限制或需改造**的点。

### 4.6 当前实现的缺点与问题（可选但建议包含）
- 功能完整性：缺失的能力、未覆盖的边界场景。
- 可观测性：日志粒度不足、错误信息不结构化、难以定位阶段/批次。
- 数据与性能：表设计、索引、分区/归档策略的潜在问题。
- 可维护性：耦合度高、扩展点不清晰等。
- 与平台规范的偏差：如与统一报错、日志规范的脱节。

### 4.7 潜在改动影响
- 可能受影响的模块、表、接口、批任务。
- 数据量、性能、历史数据兼容性等风险点。

---

## 5. 数据库调研约定

- **默认环境**：测试环境 `scp-test`；仅在上层或用户明确指定时使用 `scp-prod`。
- **库命名**：`scp_XX`，如 `scp_ips`、`scp_dfp`、`scp_sop`、`scp_mps`、`scp_ams` 等。
- **工具**：优先使用 MCP 数据库工具连接并查询表结构；若 MCP 调用失败，可尝试在 `temp-work/` 下创建临时 Python 脚本连接；若仍无法连接，须明确告知主会话和用户，并在调研结论中写明无法进行数据库调研。
