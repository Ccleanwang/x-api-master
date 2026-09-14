# Sub2API 源码核对

后续补充见[最小使用流程核对](sub2api-workflow-review.md)：包含分组模型白名单、公开分组权限、金额额度语义、用量查询和用户停用；本文件保留前轮核对范围。

管理员能力后续核对见 [管理员 Key 能力](sub2api-admin-key-review.md)：已闭合查看用户 Key、改分组和接口重置用量；所读内置后台/路由未找到指定用户代创建、代禁用/删除或轮换入口。未定义需求或改造方案。

日期：2026-09-14。固定版本：`bdb42e22f81fcb633ff0a060961211dd2bcb515b`，与初筛一致。静态阅读，未部署、未执行测试；不承诺兼容或生产验证通过。

## 源码证据

- S1：[backend/internal/service/openai_gateway_chat_completions.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/openai_gateway_chat_completions.go)、[backend/internal/service/openai_gateway_chat_completions_raw.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/openai_gateway_chat_completions_raw.go)、[backend/internal/pkg/openai_compat/upstream_capability.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/pkg/openai_compat/upstream_capability.go)、[backend/internal/service/openai_gateway_request_body.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/openai_gateway_request_body.go)
- S2：[backend/internal/service/concurrency_service.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/concurrency_service.go)、[backend/internal/repository/concurrency_cache.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/repository/concurrency_cache.go)、[backend/internal/handler/gateway_helper.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/gateway_helper.go)、[backend/internal/handler/openai_chat_completions.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/openai_chat_completions.go)、[backend/internal/handler/openai_gateway_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/openai_gateway_handler.go)
- S3：[backend/internal/server/middleware/api_key_auth.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/middleware/api_key_auth.go)、[backend/internal/service/billing_cache_service.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/billing_cache_service.go)、[backend/internal/service/openai_gateway_usage.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/openai_gateway_usage.go)
- S4：[frontend/src/api/keys.ts](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/api/keys.ts)、[backend/internal/handler/api_key_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/api_key_handler.go)、[backend/internal/handler/admin/apikey_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/admin/apikey_handler.go)、[backend/internal/server/routes/admin.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/routes/admin.go)

## 私有模型接入：有条件复用

S1 中 OpenAI APIKey 账号可以走原生 Chat Completions 转发路径，不必把全部请求转为 Responses；代码读取 model、stream，进行模型映射并处理普通 JSON/流式 SSE。URL 构造函数支持 base URL 已含 /v1、完整聊天端点等形式。它并不只支持 OAuth 订阅。

关键配置差异：openai_responses_mode 支持 force_chat_completions、force_responses 和 auto；auto 缺少探测结果时默认按支持 Responses 处理。只有聊天端点的私有服务若未设置正确模式，不能假设自动兼容。CN 账号还使用自己的 api_protocol 分流，不应混用配置。

OpenAI validateOutboundURL 在关闭 allowlist 时按 allow_insecure_http 校验格式；开启后按 UpstreamHosts 和 AllowPrivateHosts 等检查 HTTPS 地址。内部地址是否可用取决于实际策略，不能笼统声称只支持公网或默认任意内网可达。

结论：接入标准聊天协议私有服务的基础代码存在。仍需核对具体账号创建配置、模型允许范围、凭证形式、其他端点及精确模型字段兼容；未实际连接 vLLM。映射一个账号到一个服务地址是可考虑的方式，尚不是已验证部署方案。

## 并发控制：用户和账号名额有实现

S2 的 AcquireUserSlot/AcquireAccountSlot：
- 正数上限通过 Redis 缓存申请 requestID 对应名额。
- 上限为零或负数表示不限制，不是拒绝所有请求。
- 返回 ReleaseFunc；释放用独立的 5 秒背景上下文，失败记录警告。
- Redis 申请错误向上返回。所读 OpenAI 用户名额调用链会处理错误并停止继续请求。
- 缓存 Lua 使用有序集合计数、过期清理与原子申请。辅助层提供有界等待和超时。
- OpenAI 聊天处理入口申请用户名额，defer 释放；账号名额也进入独立获取路径。断连包装和资源回收需后续运行验证。

必须区分：
- TrackAPIKeySlot 注释明确仅统计 Key 活跃请求，不实施 Key 级并发上限；统计失败可放行。
- WebSocket 入口连接租约是另一种限制，不能当成所有 HTTP 推理的行为。
- 账号级上限只管该账号；若为同一模型服务建多个账号，上限不会自然成为该服务的统一总上限。
- 过期回收、长请求、进程异常和多个网关实例尚未测试，不推导严格端到端容量保证。

相比 New API 尚未确认的并发路径，此处证据更直接；与 LiteLLM 的限制维度及故障策略不同，不能简单以“都有 Redis”判等价。

## Simple Mode：不是只关闭支付

S3 的 APIKeyAuth 在基础认证后提前进入后续处理：
- 仍检查 Key 禁用状态、用户有效状态、IP 和分组限制。
- 保留用户 Concurrency，后续处理仍可申请并发名额。
- 在该中间件跳过后续 Key 过期/额度检查。基础状态分支明确把 expired、quota_exhausted 留给后面的计费阶段处理。
- BillingCacheService.CheckBillingEligibility 在 simple 模式直接返回，跳过计费资格检查。
- OpenAI 用量记录分支仍写 usage log，但不执行后续收费流程；写入为 best effort，不能承诺永不丢失。

结论：不能推荐简化模式无损覆盖“有有效期、有配额”的内部管理需求。若要求到期失效，需要选择其他配置/模式，或进一步检查是否有可复用独立校验；不能把推理请求已在所有路径绕过过期检查作为全仓库结论，当前证据限所读中间件。

## Key 管理：自助路径已找到，集中代发待核

S4 前端 /keys 的创建、查询、修改和删除均有 API 封装。APIKeyHandler.Create 调用服务时传入 subject.UserID，更新/删除也传当前用户 ID。

管理员路由中发现：
- 可查询某用户的 API Keys。
- /admin/api-keys/:id 的 PUT 对应 UpdateGroup。
这些不能证明支持管理员为任意用户代发或任意修改 Key。集中代发继续标待核，不建议用管理员自己的 Key 改名字替代归属。

前端接口链是复用线索，实际按钮、角色可见性、全部日志页尚未逐个审阅。

## 上下文与授权

检索 service/handler 的 context_length、max_context、max_input_tokens 等命中模型描述、客户端上下文元数据等。所读原生聊天路径没有闭合“精确输入 Token + 输出预算超限拒绝”的统一机制。结论是待确认，不是确定不存在。模型元数据或报给客户端的窗口值不能代替网关拦截。

许可沿用 [初筛](notes/sub2api.md)：LICENSE 为 LGPLv3，README 写 LGPLv3 或更新版本，并另有商业运营声明。未完成具体文件的全面许可审计，不给出公司改造方案的最终授权结论。

## 最小改造判断更新

| 能力 | 当前分类 | 影响 |
| --- | --- | --- |
| 自定义私有聊天上游 | 有条件配置复用 | 要选择正确协议/地址策略，未验证目标模型 |
| 用户/账号并发 | 有实现，可配置复用候选 | 正数上限、错误处理和账号与服务映射须明确 |
| 自助 Key 管理 | 已找到操作链 | 集中代发仍待确认 |
| 简化模式下统计 | 有写入路径 | 不计费，不保证所有限制保留 |
| 简化模式下 Key 到期/额度 | 所读认证路径跳过 | 与需要到期/额度的流程不匹配，需调整模式或扩展 |
| 严格上下文限制 | 待确认 | 不能承诺无需扩展 |
| GPU 管理/部署 | 本轮未核对 | 网关账号调度不是 GPU 调度 |

Sub2API 应继续作为有实际源码依据的第三候选，不能因其订阅定位直接排除。现阶段建议将“标准模式与简化模式的差异”纳入改造成本，仍不锁定最终项目或技术栈。
