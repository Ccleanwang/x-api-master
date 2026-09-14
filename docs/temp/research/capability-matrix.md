# 统一 API 与管理平台能力核对矩阵

本矩阵基于 GitHub 仓库源码目录、关键文件和部署配置的静态阅读。它是调研草稿，不代表最终选型；“已发现”表示代码中有明确线索，仍需后续阅读具体实现和测试。

标记：早期 ✅ 仅表示线索存在，不等于运行验证。本轮采用“实现证据 / 有条件 / 待确认”，详见 [固定版本源码复核](source-review.md)。

| 能力 | One API | New API | LiteLLM | 源码/配置依据与备注 |
| --- | --- | --- | --- | --- |
| OpenAI 兼容接口 | ✅ | ✅ | ✅ | One/New 的 `relay/` 适配层；LiteLLM README 与 Proxy 入口 |
| vLLM / OpenAI 兼容上游 | 自定义地址转发；具体版本待核 | 自定义地址转发；具体版本待核 | 文档支持 vLLM；适配器待完整核对 | O2/N2；不承诺全部端点兼容 |
| 模型/渠道配置 | ✅ | ✅ | ✅ | One/New `controller/channel.go`、`model/channel.go`；LiteLLM Router/配置 |
| API Key 管理 | ✅ | ✅ | ✅ | One/New `controller/user.go`、`model/user.go`；LiteLLM Virtual Key |
| 用户管理 | ✅ | ✅ | ⚠️ | One/New 用户控制器和模型明确存在；LiteLLM 需核对开源版边界 |
| 团队/角色/细粒度权限 | 基础角色；团队待核 | 基础角色/分组；完整团队待核 | Team/User 表；商业边界待核 | O1/N1/L1：渠道授权不等于团队权限 |
| 速率限制 | IP 时间窗口，推理接口覆盖待核 | 用户 ID 令牌桶/时间窗口 | RPM/TPM 实现，配置路径待核 | O2/N2/L2；速率不等于并发 |
| 并发限制 | 待确认 | 待确认（撤回此前支持结论） | 有条件：v3 名额申请/释放 | L2；New API 的 N2 是速率限制 |
| 上下文长度限制 | 待确认 | 待确认 | 有条件：启用预检查及正确模型信息 | L2 默认关闭预检查；HTTP 字节上限不等于 Token 上限 |
| Token/额度配额 | ✅ | ✅ | ✅ | One/New `model/token.go`、计费/日志控制器；LiteLLM spend/budget 能力 |
| 模型路由/负载均衡 | ⚠️ | ✅ | ✅ | New API 有 `channel_affinity_cache.go`、自动分组/渠道选择测试；LiteLLM Router 是核心能力 |
| 重试/故障转移 | ⚠️ | ✅ | ✅ | New API 有 `channel_pin_retry.go` 等测试；LiteLLM README 明确有 load balancing/fallback |
| 流式响应 | ✅ | ✅ | ✅ | One/New relay 及 README/API；LiteLLM 支持统一 endpoint |
| 调用日志与 Token 统计 | ✅ | ✅ | ✅ | One/New `model/log.go`、`controller/log.go`；LiteLLM logging/callbacks |
| 管理操作审计 | ⚠️ | ✅ | ⚠️ | New API 有 `model/audit_log.go` 和 `access_token_audit_test.go`；One API/LiteLLM 待确认 |
| 管理后台 | ✅ | ✅ | ✅ | One/New 为完整应用；LiteLLM README 提到 admin dashboard |
| Redis 共享状态/缓存 | ✅ | ✅ | ⚠️ | One/New Compose 和 `common/redis.go`；LiteLLM 需确认部署依赖 |
| 关系数据库 | ✅ | ✅ | ✅ | One API 默认 MySQL/SQLite；New API PostgreSQL/MySQL；LiteLLM 有数据库配置 |
| 日志独立存储 | ❓ | ✅ | ⚠️ | New API 支持独立 PostgreSQL/ClickHouse 日志；其余待确认 |
| Docker 私有化部署 | ✅ | ✅ | ✅ | 三者均有 Docker/Compose 线索 |
| 多节点部署 | TTL 缓存，一致性待核 | 定期渠道同步，一致性待核 | Redis 共享及本机降级，一致性待核 | O3/N3/L2；均未运行验证 |
| Kubernetes 部署 | ⚠️ | ⚠️ | ⚠️ | LiteLLM 仓库含 Helm 线索；One/New 本轮仅确认 Docker Compose |

## 当前结论

- One API 和 New API 的源码形态都接近“Go 单体管理与转发平台”，适合重点阅读业务数据模型、权限、配额和管理接口。
- LiteLLM 的核心优势集中在模型协议统一、路由、负载均衡、Fallback、Virtual Key 和成本/日志治理。
- 目前没有证据表明任何一个项目能完整覆盖 x-api-master 的四个大模块；GPU 集群和模型生命周期管理仍需其他组件或自研管理层。
- 下一步应针对标记为 `⚠️`/`❓` 的能力定位具体实现，再更新判断和源码证据。
