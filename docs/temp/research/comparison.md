# 统一 API 与管理平台候选项目对比（深入粗读）

本表是基于 GitHub README、仓库目录和公开部署文件形成的调研草稿，不代表最终选型。

| 项目 | 主要定位 | 后端/架构线索 | 数据与部署 | 与 x-api-master 的关系 |
| --- | --- | --- | --- | --- |
| LiteLLM | LLM 统一网关和 SDK | Python Proxy + SDK，仓库含 Rust、UI、Helm、E2E | Docker、Kubernetes 等；数据库配置和日志能力丰富 | 最接近统一模型调用层，重点研究网关、路由、Key、预算和可观测性 |
| One API | LLM API 管理与分发 | Go 单体，controller/middleware/model/relay/monitor | SQLite/MySQL、Redis、Docker Compose，支持多节点线索 | 最接近管理后台和 Key/渠道管理的完整产品形态 |
| New API | AI 模型聚合与企业管理 | Go 单体，治理和安全目录更丰富 | PostgreSQL/MySQL、Redis，可选 ClickHouse 日志、Docker Compose | 重点比较其在审计、安全、日志分离和多节点方面的演进 |
| Portkey Gateway | AI 请求路由与 Guardrails | TypeScript Gateway | 具体部署和业务管理待深入 | 作为路由和安全治理参考 |
| Kong | 通用 API/AI Gateway | Lua 插件化网关 | 适合独立基础设施部署 | 可作为通用入口和流量治理组件，但业务管理需自建 |
| Apache APISIX | 云原生 API/AI Gateway | NGINX + etcd，Lua 插件 | Docker/Kubernetes，配置动态更新 | 可作为高性能流量入口，需评估 LLM 业务能力与管理层组合方式 |

## 当前阶段观察

1. LiteLLM 更像“模型调用和路由层”，One API/New API 更像“带管理后台的 API 分发产品”。
2. One API 和 New API 的代码结构相近，适合做同一类产品的演进对比；New API 在安全、审计和日志存储方面出现更多企业化线索。
3. Kong/APISIX 解决的是通用网关问题，不能直接等同于用户、模型和额度管理平台。
4. 当前尚不能判断采用单一项目还是组合方案，需要继续阅读三者的配置、权限、限流、数据模型和部署文档。
