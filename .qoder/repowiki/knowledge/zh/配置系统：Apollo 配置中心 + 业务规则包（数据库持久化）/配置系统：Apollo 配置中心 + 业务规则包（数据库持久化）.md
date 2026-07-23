---
kind: configuration_system
name: 配置系统：Apollo 配置中心 + 业务规则包（数据库持久化）
category: configuration_system
scope:
    - '**'
source_files:
    - CLAUDE.md
    - docs/design/目标测算与拆解.md
    - docs/design/00-契约基线-接口清单.md
    - docs/design/数据接入与达成计算.md
    - docs/design/目标协同管理.md
---

## 1. 系统/方案概览
本仓库为**纯文档与规范仓库**，不包含后端或前端工程源码，因此不存在运行时配置加载代码。但通过 `CLAUDE.md` 编码规范与多份设计/需求文档，可明确该项目的配置体系由两层组成：
- **基础设施层**：使用公司统一配置中心 **Apollo** 管理环境配置、连接串、密钥等外部化配置；本地开发允许临时写 `application.yml`，但禁止提交真实密钥。
- **业务配置层**：将国家规则包、测算公式、预警阈值、审批模板等“随业务变更的配置”持久化到 MySQL，并通过 REST 后台接口（如 `/goal/config/calc-rules`、`/goal/config/alert-rules`、`/goal/config/approval-templates`）在线维护，具备 `config_version` 版本递增与审计日志能力。

## 2. 关键文件与位置
- `CLAUDE.md` — 项目宪法，第 3.1 节规定“配置外部化(配置中心 Apollo)，严禁把环境配置、连接串、密钥写进代码”，并给出 Hulk 框架 starter 引入约定。
- `docs/design/目标测算与拆解.md` — 定义 `goal_calc_rule_pack` 表结构、`config_version` 字段、`algorithm_params` JSON 存储策略、规则包匹配器与配置后台入口。
- `docs/design/00-契约基线-接口清单.md` — 列出所有配置类 admin REST 资源路径及归属模块。
- `docs/design/数据接入与达成计算.md` — 说明实时/T+1 模式开关、指标口径 `MetricDefinition` 配置化。
- `docs/design/目标协同管理.md` — 定义 `goal_wb_approval_template` 审批模板配置表、逾期提前量 `overdueAdvanceDays` 等配置项。

## 3. 架构与约定
- **分层原则**：Apollo 管“部署期不变的环境配置”，MySQL 管“业务期频繁变更的规则配置”。二者互不替代。
- **版本与审计**：业务配置每次变更递增 `config_version`，写操作记录操作人/时间/维度/前后值；仅对变更后新触发生效，已生成结果保留当时的 `rule_pack_version` 用于回溯。
- **参数形态**：算法参数集合采用 JSON 列（`algorithm_params`），避免随国家数量膨胀出大量列，校验在入库时做 schema 校验。
- **实现约束**：业务逻辑从 `EstimateContext` / `SplitContext` 读取规则包参数，**不读配置文件、不写死常量**；新增国家 = 加一个实现类 + 加一条规则包配置行。

## 4. 开发者应遵守的规则
- 环境配置一律走 Apollo，禁止硬编码密钥/连接串；本地调试可用 `application.yml` 但不许提交生产配置。
- 业务规则配置必须通过设计文档中定义的 REST 后台接口维护，不得绕过接口直接改库。
- 任何新增配置项需同步更新对应表的 `config_version` 语义与审计日志。
- 配置变更遵循 Gate #1“契约先行”：跨模块/对外部系统的配置改动先由 architect 在 `docs/design/` 确认契约后再实现。
