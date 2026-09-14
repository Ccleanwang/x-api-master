# 三个候选项目：源码复核

日期：2026-09-14。状态：调研草稿，未选型。仅静态阅读，未部署、未运行测试。目录名只能作为线索，以下判断基于具体结构、函数和调用位置；没有找到不等于不支持。

## 固定版本与证据

以下链接固定到本轮获取的提交，不随默认分支变化。它们不是经验证的生产推荐版本。

| 编号 | 项目与提交 | 已阅读证据 |
| --- | --- | --- |
| O1 | One API `8df4a2670b98266bd287c698243fff327d9748cf` | [Token 结构与校验](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/model/token.go)、[认证与模型授权](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/middleware/auth.go) |
| O2 | 同上 | [限流](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/middleware/rate-limit.go)、[实际入口](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/router/relay.go)、[文本转发](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/relay/controller/text.go) |
| O3 | 同上 | [日志](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/model/log.go)、[缓存](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/model/cache.go)、[许可证](https://github.com/songquanpeng/one-api/blob/8df4a2670b98266bd287c698243fff327d9748cf/LICENSE) |
| N1 | New API `04c64734cc7c2e58a9efd0b247182330f3f52cce` | [Token](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/model/token.go)、[认证](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/middleware/auth.go)、[路由注册](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/router/relay-router.go) |
| N2 | 同上 | [模型请求限流](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/middleware/model-rate-limit.go)、[Redis 限流器](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/common/limiter/limiter.go)、[上游地址适配](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/relay/channel/openai/adaptor.go) |
| N3 | 同上 | [日志](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/model/log.go)、[渠道缓存](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/model/channel_cache.go)、[许可证第 13 节](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/LICENSE#L539) |
| L1 | LiteLLM `30f33a949b8a2bb890a2baee18e2ab7ab015a4f7` | [数据结构](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/schema.prisma)、[认证](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/proxy/auth/user_api_key_auth.py) |
| L2 | 同上 | [并发名额申请与释放](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/proxy/hooks/parallel_request_limiter_v3.py#L1451)、[Router](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/router.py) |
| L3 | 同上 | [部署配置](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/docker-compose.yml)、[根许可证](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/LICENSE)、[企业许可证](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/enterprise/LICENSE.md) |

## 六个场景的核对结果

### 1. 给同事发放 Key

- One API：O1 的 Token 包含所属用户、状态、有效期、额度、允许模型和网段；ValidateUserToken 检查状态、过期和余额，TokenAuth 检查允许模型。具备具体实现证据。
- New API：N1 包含状态、过期、ModelLimits、AllowIps、Group 和跨分组重试配置；认证层传递模型限制供后续处理。这里的 Group 不能直接当作公司的部门/团队成员关系。
- LiteLLM：L1 的 VerificationToken 与 User、Team、Budget 分开建模，含过期、禁用、模型范围、用户/团队关联及速率参数；认证有过期检查和团队缓存刷新逻辑。有业务管理能力，不是单纯转发器；完整权限操作及商业开关仍需逐项核对。

### 2. 接入私有模型

- One API：O2 注册聊天接口，文本转发设置流式标记、模型映射，调用适配器处理请求与响应；渠道分配设置 BaseURL。
- New API：N1 注册聊天/Responses 等接口，N2 使用 ChannelBaseUrl 构造上游 URL。
- 两者均有接入自定义地址的代码路径，但不代表任意 vLLM 版本、工具调用或多模态字段均兼容。
- LiteLLM：已有 README 中 vLLM 适配声明，L2 有模型部署路由；本轮未完整追踪 vLLM 适配器，因此保留为“文档支持、具体适配待核”。

### 3. 保护 GPU 资源

请求频率（每分钟多少次）、并发（同时多少个请求）、上下文（一次请求的输入及输出长度预算）是三个独立问题。

- One API：O2 的通用限流以客户端 IP 为键，用时间窗口计数。仅看到此函数不能保证它绑定到每个推理接口；所读 Relay 路由直接挂载的是认证和渠道分配。尚未确认推理用户级并发限制。
- New API：N1 明确挂载 ModelRequestRateLimit；N2 用用户 ID 构造键，总请求用令牌桶、成功请求用窗口计数。函数名中的 Model 不代表按每个模型独立计数。没有发现这条路径上占用和释放并发名额的机制，撤回此前“并发已支持”的判断。渠道测试并发参数也不是用户推理并发。
- LiteLLM：L2 的 v3 limiter 有按 slot_id 申请/释放名额、Redis Lua 原子申请、过期名额回收和本地降级实现。属于真实并发控制代码，但启用哪版 limiter、流式断连和多维度参数需在后续固定配置下核对。Redis 故障时本机降级不能保证集群严格上限。
- 上下文：One/New 尚未确认统一的输入加预留输出预算限制；Token 计数、HTTP 请求体字节限制和输出 max_tokens 都不足以单独证明该能力。LiteLLM Router 的 enable_pre_call_checks 默认 false，开启后可按 max_input_tokens 筛掉超长部署；它依赖模型元数据与计数方式，不能直接等同于准确的 DeepSeek/GLM 总上下文硬限制。
- 额度不是 GPU 容量：充值余额或美元预算能限制累计消费，但不能代替显存、并发和上下文控制。

### 4. 查看使用情况

- O3 的日志有用户、Token 名、模型、输入/输出 Token、耗时和渠道字段；N3 还包含 Token ID、分组、请求 ID、上游请求 ID、错误记录函数。
- L1 的 SpendLogs 包含用户/团队、模型、Token、耗时、状态，也有 messages/response 字段。字段存在不说明默认保存全文，需另外核对写入和脱敏配置。
- One API 文本转发的 debug 日志可记录转换后的请求体。因此不能把“消费表只存统计”写成“系统绝不记录代码内容”。

### 5. 扩展为多个网关实例

- One API：O3 从 Redis 读取 Token、未命中则查数据库并写 TTL 缓存；能共享部分状态，但不等于禁用立即在所有实例生效。
- New API：N3 的 SyncChannelCache 定期刷新本机渠道缓存；N2 有 Redis 和内存两种限流路径。成功数检查/记录分开进行，不能由此承诺严格原子上限。
- LiteLLM：L3 示例包含 PostgreSQL，L2 可用 Redis 共享限额状态；示例 Compose 本身未配 Redis。数据库配置变更的传播时延和 Redis 失联策略尚未全面核对。
- 三者均不能仅凭存在 Docker/Redis 就标记为“多节点一致性已验证”。

### 6. 内部修改与长期维护

- One API：MIT，允许修改与使用，保留相应版权/许可声明；作为精简二次开发候选，尚缺的治理能力需要补齐。
- New API：AGPLv3 第 13 节要求修改后的网络服务向交互用户提供相应源码获取机会。“内网使用”不能自动推导为没有相关义务，也不等于必须向全世界公开公司全部代码。公司应确认拟修改/集成边界是否接受该条件。
- LiteLLM：根许可证明确 enterprise 目录单独授权；企业许可证要求生产使用满足相应协议。认证源码中的 JWT Auth 有 premium_user 检查，不能将代码公开等同免费生产可用。普通 Key/Team 数据结构存在不证明所有企业权限和审计免费。
- 维护成本尚无工时测量；本轮只按改动范围、依赖数量、升级耦合评估，不以 Star 或文件数量下最终结论。

## 更正旧结论

撤回 New API“完整团队权限已确认”“并发已确认”的标记；上下文字节限制与 Token 限制分开；LiteLLM 业务管理能力不再简单归为缺失。此前 notes、数据模型/流程笔记是早期线索，若有冲突以本次固定版本复核为准。
