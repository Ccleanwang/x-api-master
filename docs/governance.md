---
title: "个人项目文档治理规范"
status: approved
version: 2.0.0
author: "ForrestWang"
created: 2026-08-25
updated: 2026-08-30
---

# 个人项目文档治理规范

## 1. 目的

本规范为 `agent-learning-lab` 下的项目提供统一的项目、Spec、设计决策、接口契约、任务跟踪和验证证据管理方式。

## 2. 文档层级与目录

项目级文档维护整体方向；每个独立开发范围建立一个 Spec，维护自己的治理文档：

```text
agent-learning-lab/
├── docs/
│   ├── governance.md
│   └── templates/
│       ├── proposal.md
│       ├── design.md
│       ├── api_schema.yaml
│       └── task.md
└── <project>/
    └── docs/
        ├── project.md
        └── specs/
            └── <spec-name>/
                ├── proposal.md
                ├── design.md
                ├── api_schema.yaml
                └── task.md
```

项目级 `project.md` 说明项目愿景、长期目标、总体范围和 Spec 路线。Spec 文档说明一个相对独立的功能、模块或开发范围。不同 Spec 的任务不得混合在同一份 `task.md` 中。

## 3. Spec 文档职责

| 文档 | 主要问题 | 内容 |
| --- | --- | --- |
| `proposal.md` | 为什么做、做什么 | 背景、目标、范围、原则、候选方案和风险 |
| `design.md` | 怎么设计、为什么这样设计 | 架构、模块、数据流、关键设计决策和验证方案 |
| `api_schema.yaml` | 对外暴露什么 | 公共接口、数据模型、参数、返回值和异常契约 |
| `task.md` | 如何执行和证明完成 | 任务、依赖、验收标准、状态和证据 |

ADR 不再作为独立文件。架构决策以 `design.md` 中的 `ADR-*` 小节记录，并在同一处维护决策状态、影响和验证结果。每个 Spec 必须提供 `api_schema.yaml`；没有稳定公共接口时保留文件并置空或声明无公共 API。

## 4. 推荐工作流

```text
project.md
    -> spec/proposal.md
    -> spec/design.md
    -> spec/api_schema.yaml
    -> spec/task.md
    -> implementation
    -> verification
```

1. 在项目级 `project.md` 中确认整体方向和 Spec 边界。
2. 为独立开发范围建立 Spec 目录，并从公共模板复制四份文档。
3. 更新 `proposal.md`，明确当前 Spec 的目标和边界。
4. 讨论并确认技术方案后，更新 `design.md`，并记录 `ADR-*`。
5. 将稳定的公共接口写入 `api_schema.yaml`；无公共接口的 Spec 也必须保留该文件。
6. 将设计拆解为 `task.md` 中的任务，执行实现并记录实际验证。

如果实现过程中发现目标、设计或接口契约需要变化，应回到对应文档更新，而不是只修改代码或另建平行文档。Spec 完成后保留其文档，通过版本、变更记录、审阅记录和 Git 提交保留历史。

## 5. 状态约定

项目和 Spec 文档：`draft`、`approved`、`active`、`completed`、`archived`。

Design 中 ADR：`proposed`、`approved`、`verified`、`failed`、`waived`、`retired`。

任务：`todo`、`in_progress`、`blocked`、`review`、`done`、`cancelled`。

状态必须反映真实进度。计划中的验证不得写成已完成；失败和阻塞必须保留原因及后续处理方式。

## 6. API Schema 规则

- `api_schema.yaml` 是当前 Spec 公共接口契约的唯一来源。
- 只记录稳定的公共类、函数、数据模型、参数、返回值、异常和兼容性要求。
- 不记录私有成员、临时实现细节或未确认的实验接口。
- 接口变更必须同步更新 `design.md`、`api_schema.yaml` 和相关任务。
- 无公共接口时使用统一空结构，并说明原因：

```yaml
version: 0.1.0
status: none
reason: "本 Spec 不对外暴露稳定公共接口"
interfaces: []
```

## 7. 证据、安全与审阅

- 每个核心公共接口至少对应一项可复现验证。
- 本地测试、静态检查、手工运行和真实外部服务验证必须区分记录。
- 真实 API 验证至少记录日期、命令、服务摘要、进程结果和响应摘要。
- 验证输出不得包含 API key、token、密码或完整敏感环境变量。
- `task.md` 只记录实际发生的验证；`design.md` 记录验证方案和 ADR 证据。
- Git 提交前必须人工确认提交范围、提交说明、验证结果和敏感信息安全。
- 未经确认不得将其他项目改动、`.env` 或生成文件纳入提交。

## 8. 治理规范变更

本规范变更时更新版本号、变更记录和审阅记录。新项目使用最新模板；既有项目仅在结构或治理行为需要调整时迁移，不要求重写无关历史。

## 9. 变更记录

| 版本 | 日期 | 变更内容 | 变更原因 |
| --- | --- | --- | --- |
| 1.0.0 | 2026-08-25 | 建立项目文档治理规范 | 统一项目核心文档管理 |
| 2.0.0 | 2026-08-27 | 引入项目/Spec 两层结构，合并 ADR 到 design，新增 API Schema | 支持大型项目按独立 Spec 演进 |

## 10. 审阅记录

| 日期 | 审阅人 | 结论 | 备注 |
| --- | --- | --- | --- |
| 2026-08-25 | ForrestWang | 通过 | 初版通用文档治理规范 |
| 2026-08-27 | ForrestWang | 通过 | 项目/Spec 两层治理、Design 内嵌 ADR 和 API Schema 规则调整 |
