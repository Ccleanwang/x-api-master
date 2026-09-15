# Sub2API 配额与限流机制核对

日期：2026-09-15。固定版本：`bdb42e22f81fcb633ff0a060961211dd2bcb515b`，与[前轮源码核对](sub2api-source-review.md)一致。仅静态阅读，未部署、未运行测试、未连接模型服务。本文用于回答[首版范围讨论稿](module-1-first-release-scope.md)第 5 点遗留的问题：能否实现"每人一个每日 Token 额度"。

## 1. 本次核对要回答的问题

首版范围讨论中，第 5 点先按"金额代理"（用虚拟定价把 Token 预算折算成 USD 阈值，复用现成金额额度字段）推进，并列为待核实的路线。本次核对围绕五项问题展开：

| 编号 | 问题 | 结论 |
| --- | --- | --- |
| V1 | 额度字段的数据类型能否承载 Token 数 | 能。`decimal(20,10)` / `decimal(20,8)`，整数范围远超需求 |
| V2 | 配额判断入口有几处、是否集中 | 两套独立体系，判断各自集中，总入口为 `CheckBillingEligibility` |
| V3 | 用量是否记录 Token 数、能否按人按日聚合 | 齐全，且有 `(user_id, created_at)` 复合索引 |
| V4 | 窗口是滚动还是自然日 | 体系 A 滚动；体系 B 为自然日，符合"每日额度" |
| V5 | 上游不返回 usage 时的行为 | 记 0、不扣量，**fail-open**，是本路线的主要风险 |

## 2. 更正：放弃金额代理路线

前次讨论曾建议首版先用"金额代理"跑通：给私有模型配虚拟定价，把每日 Token 预算折算成 USD 阈值，复用现成金额额度字段。**本次核对表明该前提不成立，建议放弃这条路线。**

理由是计价并非按 Token 线性相乘。`billing_service.go` 的 `applyModelSpecificPricingPolicyEx` 及相邻函数中存在多层与模型绑定的价格修正：

- **强制覆盖**：`forceDeepSeekRates` 为真且命中 `isDeepSeekModel` 时，忽略价目表给定价，一律替换为硬编码的 DeepSeek 官方低谷价（`deepseekProOffPeakInputPrice`、`deepseekFlashOffPeakInputPrice` 等），并按计费时点区分 pro/flash 档。
- **高峰时段倍率**：注释明确"高峰时段倍率不在本函数处理，由 `calculateTokenCost` 按 `deepseekPeakMultiplierAt` 对默认价卡另行叠加"。
- **Fast / priority 档倍率**：`openAIModelFastPricingRatio` 对 `gpt-5.6`、`gpt-6-astra`、`gpt-5.4` 返回 2.0，对 `gpt-5.5` 返回 2.5。
- **长上下文阶梯**：由 `longContextBillingEnabled` 参数显式控制，目录数据驱动（`above_XXXk` 折算或显式 `long_context_*` 字段）。
- **计费时点**：pro → Flash 的切换按 `pricingAt` 判定，历史时点与当前时点用不同价格。

此外还有 `rate_multiplier`、`account_rate_multiplier` 两层倍率快照进入成本计算。

结论：经过这些修正后，USD 数字与 Token 数之间**不存在稳定换算关系**，金额代理不能作为 Token 配额的可靠载体。这一条同时说明，即使不追求 Token 配额，把金额额度当作"能反推用量"的字段使用也不可靠。

## 3. 两套并存的配额体系

系统中存在两套相互独立、语义不同的配额机制，容易混淆，需明确区分。

| | 体系 A：Key 窗口限流 | 体系 B：用户 × 平台配额 |
| --- | --- | --- |
| 承载位置 | `api_keys` 表内嵌字段 | 独立表 `user_platform_quotas` |
| 维度 | Key × {5h, 1d, 7d} | 用户 × 平台 × {日, 周, 月} |
| 窗口类型 | 滚动窗口 | 自然日历窗口 |
| 计量单位 | USD | USD |
| 判断函数 | `evaluateRateLimits` | `checkUserPlatformQuotaEligibility` |
| 管理员接口 | 仅改分组、重置用量（前轮已核） | 查询、设置、重置窗口（本轮新增） |

两者都在 `CheckBillingEligibility` 中被串行调用，该函数是请求进入计费检查的总入口。

### 3.1 体系 A：Key 窗口限流

`billing_cache_service.go` 的 `evaluateRateLimits` 是唯一判断点，共三处比较，逻辑形态一致：

```go
if apiKey.RateLimit5h > 0 && usage5h >= apiKey.RateLimit5h { ... }
if apiKey.RateLimit1d > 0 && usage1d >= apiKey.RateLimit1d { ... }
if apiKey.RateLimit7d > 0 && usage7d >= apiKey.RateLimit7d { ... }
```

- 窗口过期判定为 `IsWindowExpired(windowStart, duration)`，实现是 `windowStart == nil || time.Since(*windowStart) >= duration`，即**滚动窗口**，不是自然日。
- 窗口过期时在内存中把 usage 清零用于本次判断，并异步触发 `ResetRateLimitWindows` 与缓存失效。判断与重置之间存在短暂不一致，但内存清零保证保守。
- 累加入口是 `QueueUpdateAPIKeyRateLimitUsage(apiKeyID, cost)`，异步写缓存；传入的是 `p.Cost.ActualCost`（金额）。
- 该体系是**按 Key** 而非按用户，一个人持有多把 Key 时额度各自独立，不满足"按人聚合"。
- 读取失败（缓存与 DB 均不可用）时 `return nil` 放行，属 fail-open。

### 3.2 体系 B：用户 × 平台配额（推荐承载）

`user_platform_quotas` 表的字段形态与"每人一个每日额度"直接对应：

```go
field.Float("daily_limit_usd").Optional().Nillable().SchemaType(... "decimal(20,10)")
field.Float("weekly_limit_usd")...
field.Float("monthly_limit_usd")...
field.Float("daily_usage_usd").Default(0)...
field.Float("weekly_usage_usd")...
field.Float("monthly_usage_usd")...
field.Time("daily_window_start")...
field.Time("weekly_window_start")...
field.Time("monthly_window_start")...
```

配合 `index.Fields("user_id", "platform").Unique()`，具备以下性质：

- **按 user_id 建唯一约束**，天然按人聚合，多把 Key 共用同一份额度，符合已确认的"按人聚合"要求。
- **自然日窗口**：`timezone.StartOfDay(now)` 判定窗口起点，非滚动 24 小时。
- **窗口自愈**：注释说明 DB 层 `IncrementUsageWithReset` 具备窗口自愈能力，持久化数据始终正确；缓存层在检测到窗口过期时用原子覆盖（而非 Delete）刷新 entry，避免并发请求的 cost 永久丢失。
- **管理员接口已存在**，位于 `admin` 路由组：

  ```go
  users.GET("/:id/platform-quotas",        h.Admin.User.GetUserPlatformQuotas)
  users.PUT("/:id/platform-quotas",        h.Admin.User.UpdateUserPlatformQuotas)
  users.POST("/:id/platform-quotas/reset", h.Admin.User.ResetUserPlatformQuotaWindow)
  ```

  该组挂载 `AdminAuth`、`AuditLog`、`AdminComplianceGuard`。
- 限额语义（schema 注释）：`nil` = 无限额放行；`0` = 完全禁用；`> 0` = 上限值。

因此，表结构、窗口语义、管理员接口、审计链均已具备，与需求形态的差距集中在**累加量与判断量使用 USD 而非 Token**。

## 4. Token 计量与聚合基础

`usage_logs` 表每次调用记录以下字段，均为 `field.Int`：

- `user_id`、`api_key_id`、`account_id`、`group_id`
- `input_tokens`、`output_tokens`
- `cache_creation_tokens`、`cache_read_tokens`、`cache_creation_5m_tokens`、`cache_creation_1h_tokens`

索引包含单列 `user_id`、`created_at`，以及复合 `index.Fields("user_id", "created_at")` 和 `index.Fields("api_key_id", "created_at")`。按人按时间范围聚合具备索引支撑。该表注释声明为只追加、不支持更新和删除。

关于累加点，`gateway_usage_billing.go` 中可见：

```go
deps.billingCacheService.QueueUpdateAPIKeyRateLimitUsage(p.APIKey.ID, p.Cost.ActualCost)
deps.billingCacheService.IncrementUserPlatformQuotaUsage(p.User.ID, p.Platform, p.Cost.ActualCost)
```

两处传入的均为 `p.Cost.ActualCost`（金额）。**同一文件相邻位置已经取得 Token 数**（`usageLog.InputTokens`、`usageLog.OutputTokens`、`CacheCreationTokens`、`CacheReadTokens`），并组装为 `UsageTokens` 用于计价。也就是说，Token 数在累加点已经存在于上下文中。

## 5. 数据类型与精度（V1）

相关字段的 Postgres 类型：

| 位置 | 类型 | 整数位 |
| --- | --- | --- |
| `user_platform_quotas.*_usd` | `decimal(20,10)` | 10 位（上限约 99.99 亿） |
| `api_keys.quota` / `rate_limit_*` / `usage_*` | `decimal(20,8)` | 12 位 |
| `usage_logs.*_cost` | `decimal(20,10)` | 10 位 |
| `usage_logs.input_tokens` / `output_tokens` | 整型 | — |

Go 侧额度字段为 `float64`，53 位有效位可精确表示整数至约 9×10¹⁵。整数 Token 数在这些字段中可精确存储，不存在此前担心的"小数类型装不下大额 Token"问题。

需要留意的是 `decimal(20,10)` 的整数位为 10 位，单值上限约 99.99 亿。若未来单日额度或用量规划到该量级附近，应重新评估字段精度。

## 6. 上游缺失用量时的行为（V5）

用量记录字段的默认值为 0。若上游响应未包含 usage 信息，对应 Token 数记 0，**该次请求不产生任何计量**。

这意味着：基于 Token 的额度判断属于 **fail-open**——上游不报用量时，请求不会被额度拦截。与读取失败放行的做法一致（缓存与 DB 不可用时 `checkAPIKeyRateLimits` 与 `checkUserPlatformQuotaEligibility` 均有放行路径）。

因此该路线的有效性**完全取决于上游推理服务是否如实返回 token 数**。该行为无法通过静态阅读确定，需在部署后以实际服务验证。

## 7. 实现时需要注意的语义与约束

1. **`0` 表示完全禁用，不是不限额。** schema 注释明确写为"`0` → 完全禁用（任何请求都会被拒绝，因为 `usage >= 0` 恒成立）"。管理员在页面上填 0 会锁死该用户，与"不限额"的直觉相反。`nil` 才是无限额。
2. **`platform` 字段受白名单约束。** `user_platform_quota.go` 的 ent 校验只接受 `anthropic`、`openai`、`gemini`、`antigravity`、`grok`、`kimi`、`zhipu`、`deepseek`、`minimax`、`opencode_go`。私有模型需借用其中一个 key。标注建议为 `deepseek`（与服务名一致、语义最近），但该映射只是权宜，会在页面上以平台名出现，需确认管理员理解成本。
3. **写入为"补齐契约"。** 设置接口写入单个平台后，读回结果必须包含全部允许平台，不是只返回被设置项。相关测试明确校验了这一契约。
4. **仅标准模式生效。** `CheckBillingEligibility` 在 `cfg.RunMode == config.RunModeSimple` 时直接 `return nil`，跳过所有计费检查。因此体系 B 的额度**在 Simple Mode 下不生效**。这与前轮 Simple Mode 证据一致，两个问题必须合并考虑。

## 8. 结论与待验证事项

**结论**：以 `user_platform_quotas` 表承载"每人每日 Token 额度"在结构上是可行的。按人聚合、自然日窗口、管理员接口与审计链均已具备，需改动的是累加与判断所用的量（由 `ActualCost` 改为 Token 数）。金额代理路线应放弃。

现阶段**不给出改造量估计**，也不构成实施任务。以下事项未闭合：

| 未闭合事项 | 性质 | 说明 |
| --- | --- | --- |
| 上游是否返回 usage token 数 | 需运行验证 | 决定该路线是否真正拦得住人。静态阅读无法确定 |
| 改为 Token 后的判定语义 | 需设计确认 | 输入+输出是否合并计入、缓存 token 是否计入、命中上限时返回什么 |
| 与体系 A 的关系 | 需设计确认 | per-Key 的 5h/1d/7d 金额窗口是否保留、是否停用，避免两套额度同时生效造成困惑 |
| `platform` 映射方案 | 需设计确认 | 借用白名单 key 的长期可维护性，以及是否改为扩展白名单 |
| 页面呈现 | 需设计确认 | 字段名仍为 `*_usd`，界面需确认是否显示为 Token 及单位说明 |
| 运行模式 | 需设计确认 | 体系 B 仅标准模式生效，与第 4 点（Key 到期）的模式取舍需一并决定 |

**2026-09-15 后续**：前三项（判定语义、与体系 A 的关系、运行模式）已在 [MVP 边界决策记录](module-1-mvp-decisions.md)中确定，本文第 8 节仅保留其余三项。该决策记录同时补充了一项必须写入设计的时序特性：额度检查在请求进入时、用量累加在响应结束后，在途请求会同时通过检查，额度允许被超出。

## 9. 证据索引

固定到本文版本，均为静态阅读。

| 编号 | 文件与说明 |
| --- | --- |
| Q1 | [billing_cache_service.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/billing_cache_service.go)：`CheckBillingEligibility`、`checkAPIKeyRateLimits`、`evaluateRateLimits`、`checkUserPlatformQuotaEligibility`、`checkRPM` |
| Q2 | [ent/schema/user_platform_quota.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/ent/schema/user_platform_quota.go)：体系 B 字段、`platform` 校验白名单、`(user_id, platform)` 唯一索引 |
| Q3 | [ent/schema/api_key.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/ent/schema/api_key.go)：体系 A 字段与 `decimal` 类型 |
| Q4 | [ent/schema/usage_log.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/ent/schema/usage_log.go)：Token 计数字段、`billing_mode` 注释、`(user_id, created_at)` 索引 |
| Q5 | [billing_service.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/billing_service.go)：`applyModelSpecificPricingPolicyEx`、`openAIModelFastPricingRatio`、`calculateCostInternalWithPolicy` |
| Q6 | [gateway_usage_billing.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/gateway_usage_billing.go)：用量累加调用点与 `UsageTokens` 组装 |
| Q7 | [routes/admin.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/routes/admin.go)：`platform-quotas` 路由与 `admin` 组中间件 |
| Q8 | [service/api_key.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/api_key.go)：`IsWindowExpired` 实现 |

本文不新增源码能力结论以外的判断；上述静态证据不等于最新版能力，也不代表生产验证结果。
