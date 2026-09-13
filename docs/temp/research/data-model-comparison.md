# 数据模型对比（源码阅读草稿）

## One API

- `model/user.go`：用户实体。
- `model/token.go`：访问 Token/API Key 实体。
- `model/channel.go`：上游渠道/模型服务实体及配置。
- `model/log.go`：调用日志和统计结构。
- `controller/`：用户、Token、渠道、日志等管理接口。
- 初步关系：用户持有 Token，Token 触发对渠道的调用，调用生成日志和 Token/额度统计。

## New API

- 保留 User、Token、Channel、Log 等核心实体，并增加渠道约束、渠道亲和缓存、配额预留和审计日志。
- `model/audit_log.go`：管理操作和访问 Token 审计记录，支持独立日志数据库线索。
- `model/quota_reserve.go`、`model/user_quota_adjustment.go`：额度预留和调整，说明其配额处理比简单计数更完整。
- `dto/channel_constraints.go`：渠道筛选、固定渠道和重试约束的数据结构。
- 初步关系：用户/访问 Token → 授权和限流 → 渠道筛选/亲和 → Relay → 调用日志、配额和审计日志。

## LiteLLM

- README 和源码目录显示有 Proxy 管理端点、Router、成本计算、缓存、日志集成和数据库配置。
- 其核心对象更偏 Gateway：模型部署配置、Router 路由组、Virtual Key/User/Team/Budget，以及调用日志和成本记录。
- 具体表结构和开源版权限边界仍需进一步定位，不能直接按 One/New 的实体模型类推。

## 对 x-api-master 的启发

1. “模型”与“上游服务实例/渠道”应区分：一个逻辑模型可能对应多个实际服务地址。
2. API Key 不应只存一个字符串，还应有状态、所属用户/团队、允许模型、限额和过期信息。
3. 调用日志、额度统计和审计日志可能需要不同保留周期，后续可考虑分离存储。
