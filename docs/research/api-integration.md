# 统一 API 接入与业务管理：调研结果

日期：2026-09-15；状态：调研结论，待审阅。

本文是第一模块（[项目说明](../project.md) 第 3.1 节）的调研结论汇编，面向四项大模块之一。

**本文是精炼后的调研结论**；支撑它的讨论过程与逐项证据属于临时产出，保留在 `docs/temp/research/`，后续统一清理。本目录只保存精炼后的内容。

所有结论基于 Sub2API 固定版本 `bdb42e22f81fcb633ff0a060961211dd2bcb515b` 的**静态阅读**，未部署、未运行测试、未连接模型服务、未经验证上游推理服务的实际行为。本文不代替正式 Spec，也不代表已进入编码阶段。

## 1. 结论

以 Sub2API 作为本模块的脚手架。其现成的多用户、分组授权、上游账号、Key 管理、并发控制与用量统计可覆盖首版的主要环节；需要新增的主要是**管理端集中代发 Key**，需要改造的主要是**把额度计量从金额改为 Token**。

首版目标是：管理员能够把公司已运行的私有模型开放给指定研发人员，集中发放访问 Key，按人设置每日 Token 额度与并发，查看使用情况，并能停用某个人。

**自建推理服务是既定前提**：出于数据安全考虑，模型须部署在内网自有硬件上，不存在采购官方云服务的选项。因此本模块的上游是自建服务，官方服务不在可比范围内。

这一前提的代价落在第 3.5 与 3.6 节：官方服务（`api.deepseek.com/anthropic`）已替调用方完成"思考内容与工具调用分离"和"缓存字段映射"，自建环境下这两件事无人代劳，需在部署或网关层自行解决。

### 1.1 与最初计划的两处修正

| 原计划 | 修正 | 原因 |
| --- | --- | --- |
| 首版用金额代理（虚拟定价折算 USD 阈值）实现 Token 额度 | **放弃**，改为按 Token 计量 | 计价含模型特定强制价格、高峰倍率、Fast 档倍率与长上下文阶梯，USD 与 Token 不存在稳定换算关系 |
| 依赖 Simple Mode 简化部署 | **改为标准模式** | 额度检查在 Simple Mode 下被整体跳过；标准模式的余额前提可通过配置满足 |

## 2. 能力盘点

### 2.1 可直接复用

| 能力 | 现状 | 说明 |
| --- | --- | --- |
| 用户管理 | 已有页面与服务 | 创建、编辑、状态管理；角色与分组 |
| 分组与模型白名单 | 已有转发前准入检查 | 非仅展示；检查客户端填写的公开模型名 |
| 私有模型接入 | 有条件复用 | OpenAI APIKey 账号路径，支持自定义地址与模型映射 |
| 上游账号配置 | 已有页面 | 地址、凭证、并发、模型映射、协议模式 |
| 用户与账号并发 | 已实现 | 正数上限生效，非正数不限；Redis 名额申请与释放 |
| RPM 限流 | 字段与管理路径已确认 | 完整执行链待核（见第 6 节） |
| 用量查询 | 已有页面与接口 | 可按用户、Key、账号、分组、模型、时间筛选；输入输出 Token、耗时等 |
| 用户停用 | 已有操作 | 影响该用户整体访问；认证检查用户状态 |
| 配额承载 | 表结构与接口已具备 | 用户 × 平台的日/周/月窗口，自然日历窗口，管理员接口已存在 |
| 原生聊天转发 | 透传路径 | 请求体与 SSE 均不重构，仅替换模型名 |

### 2.2 需要新增或改造

| 事项 | 类型 | 说明 |
| --- | --- | --- |
| 管理端集中代发 Key | **新增** | 底层 `APIKeyService.Create` 已接收 userID，缺管理端授权入口与处理器 |
| 管理端单 Key 停用 | **新增** | 管理端现仅有改分组处理器；需保持普通接口的 owner 校验不被破坏 |
| 额度计量改为 Token | **改造** | 累加点与判断点均已定位，改造量取决于字段类型与入口集中度 |
| Key 明文暴露的取舍 | **待设计** | 列表接口响应亦含完整明文 Key（见第 3.3 节） |

## 3. 关键技术结论

### 3.1 计价不可用于反推用量

`billing_service.go` 的定价策略包含多层与模型绑定的修正：`forceDeepSeekRates` 命中时强制替换为硬编码官方价、高峰时段倍率、Fast/priority 档倍率（`gpt-5.6`/`gpt-6-astra`/`gpt-5.4` 为 2x，`gpt-5.5` 为 2.5x）、长上下文阶梯，以及 `rate_multiplier`、`account_rate_multiplier` 两层倍率快照。

**USD 数字与 Token 数之间不存在稳定换算关系。** 金额代理路线因此放弃，同时说明不能把金额额度当作"能反推用量"的字段使用。

### 3.2 存在两套配额体系，首版只用其中一套

| | 体系 A：Key 窗口限流 | 体系 B：用户 × 平台配额 |
| --- | --- | --- |
| 维度 | Key × {5h, 1d, 7d} | 用户 × 平台 × {日, 周, 月} |
| 窗口 | 滚动 | **自然日历** |
| 判断点 | `evaluateRateLimits`，3 处 | `checkUserPlatformQuotaEligibility` |
| 首版 | **停用**（字段留空即可，有 `RateLimitXx > 0` 前置条件，零代码改动） | **采用** |

体系 B 具备按人唯一约束（多把 Key 共用一份额度）、自然日窗口、窗口自愈与原子覆盖防丢计数。

### 3.3 Key 与凭证

- **明文可得**：`APIKeyFromService` 完整映射 `Key` 字段，DTO 层不脱敏，交付方案成立。
- **附带发现**：同一映射用于列表展示，**查询 Key 列表的响应同样含完整明文 Key**；前轮记录的"页面截断"只是前端行为。设计阶段需决定是否在响应层截断。
- **管理端无代发能力**：管理端 Key 处理器只有一个改分组函数，可改字段仅 `group_id` 与 `reset_rate_limit_usage`。

### 3.4 工具调用依赖显式配置

原生聊天透传路径**不剥离任何请求字段**（`upstreamBody := body`，仅替换模型名），`tools`/`tool_choice` 原样到上游，SSE 响应亦透传。这是对编程 Agent 最有利的路径。透传路径还会**强制给上游加 `stream_options.include_usage = true`**（`raw.go:146`），以保证流式响应也返回 usage。

但走不走这条路由账号模式决定，且**默认不走**：

```go
shouldForwardOpenAIResponsesViaRawChatCompletions(account)
  └─ return !openai_compat.ShouldUseResponsesAPI(account.Extra)
       └─ return ResolveResponsesSupport(extra) != ResponsesSupportNo
```

`ResolveResponsesSupport` 在探测标记缺失时返回 `Unknown`，而 `Unknown` 判定为"使用 Responses"，即**默认走协议转换路径**。转换路径同样支持工具调用（`convertChatToolsToResponses` 有完整映射），但其返回方向需把 Responses 事件流反向映射为 `tool_calls` 增量，环节更多。**配置上应确保走透传。**

### 3.5 自建推理服务的响应结构约束

**这项约束在官方云服务上不存在，是自建独有。**

DeepSeek 的思考模式要求：若助手回合发生工具调用，后续请求须回传上一轮的思考内容，否则报 400：

```
The `content[].thinking` in the thinking mode must be passed back to the API.
```

已有多个第三方代理专门处理该问题，触发场景为"含 tool call 的多轮对话，第二轮以后失败"。

**但实测 DeepSeek 官方服务（`api.deepseek.com/anthropic`）不会踩到它。** 对 2026-09-15 一次 Claude Code 会话（451 条响应）的原始响应统计：

| content 块组合 | 条数 |
| --- | --- |
| 仅 `tool_use` | 179 |
| 仅 `thinking` | 169 |
| 仅 `text` | 109 |
| 组合出现 | **0** |

`thinking` 与 `tool_use` **严格互斥，从不共存**；`thinking → tool_use` 的连续转换出现 60 次，是稳定交替模式。客户端（Claude Code）全程不回传 thinking 块，仍正常工作——因为回传约束的触发前提是二者同现，而官方服务在协议层把它们分到了不同响应里。

**对自建的含义**：官方服务替我们做了"思考内容与工具调用分离"。自建 vLLM 时这件事没有保证——若 `--reasoning-parser` 使 `reasoning_content` 与 `tool_calls` 出现在同一响应中，而客户端不回传该字段，即落回上述 400。

由于透传路径不重构响应，网关层无法补救。可行的做法是让思考内容不出现在响应里，二者其一：

- vLLM 启动参数默认关闭思考模式（`--default-chat-template-kwargs`）
- 不加载 reasoning parser，不产出 `reasoning_content`

**待验证**：vLLM 上 `reasoning_content` 与 `tool_calls` 是否同现。这是目前对本模块影响最大的单项未知。

### 3.6 usage 字段与缓存缺口

DeepSeek 用顶层 `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` 报告缓存，**整个 Sub2API 代码库无任何一处引用这些字段名**；它只认 Anthropic 风格的 `cache_creation_input_tokens` / `cache_read_input_tokens`。

实测 DeepSeek 官方服务的 usage（同一次会话）：

| 字段 | 实测 |
| --- | --- |
| `input_tokens` / `output_tokens` | 全部有值 |
| `cache_read_input_tokens` | 450/454 条非零，最大 161792 |
| `cache_creation_input_tokens` | **恒为 0**（DeepSeek 无"缓存写入"概念） |
| `output_tokens_details.thinking_tokens` | **恒为 0**，尽管确有 thinking 块 |

两点结论：

1. **上游确实报告缓存命中**，官方服务也完成了到 Anthropic 字段名的映射。Sub2API **拿得到数据但没接**，因此缓存命中会全部计入 `input_tokens`。方向上偏保守（额度算多不算少），但读数高于官方账单口径。
2. **`thinking_tokens` 不可用作思考模式的判据**——字段存在但恒为 0。判断思考模式是否开启，应看 `reasoning_content` / thinking 块是否实际出现。

### 3.7 额度允许被超出

额度检查在**请求进入时**、用量累加在**响应结束后**，两者不在同一时刻。同一时刻在途的请求会同时通过检查，超出幅度由并发上限决定。

该能力的准确定位是**防滥用的配额**，不是**精确计费**。此定位需在设计与验收标准中明确，避免把超额误判为缺陷。

## 4. 首版边界

### 4.1 七项选择

| # | 事项 | 结论 |
| --- | --- | --- |
| 1 | 研发人员登录 | 不登录，仅用 Key 调用；用户记录仍建立，作为归属与统计容器 |
| 2 | 管理员 Key 操作 | 代创建 + 单 Key 停用；不做原子轮换 |
| 3 | Key 交付 | 手工交付 + 内置说明模板 |
| 4 | Key 自动到期 | 不做，收回靠停用 |
| 5 | 用量硬配额 | 做：每人一个每日 Token 额度 |
| 6 | 频率与上下文 | 做 RPM 且必须核实生效；上下文不设网关层限制，依赖上游模型拦截 |
| 7 | 优先工具 | 编程 Agent |

### 4.2 配置清单

| 配置项 | 值 | 作用 |
| --- | --- | --- |
| `run_mode` | `standard` | 保证额度检查生效；Simple Mode 下整体跳过 |
| `default_user_balance` | `999999999` | 占位值，越过 `balance <= 0` 的资格闸 |
| 私有模型定价 | `0` | `ActualCost` 恒为 0，余额永不消耗 |
| `openai_responses_mode` | `force_chat_completions` | 保证工具调用走透传路径 |
| 额度计量口径 | input + output + cache_creation + cache_read，**四类同比重** | 缓存命中亦消耗预填充算力 |
| 体系 A 字段 | 全部留空 | 靠 `RateLimitXx > 0` 前置条件自然失效 |
| `user_platform_quotas.platform` | 借用白名单内的 key | 白名单仅接受 10 个固定平台值 |

### 4.3 操作流程

一次性接入（管理员）：

1. **配置上游账号**：新建 OpenAI 类型 APIKey 账号，填私有模型服务地址与凭证，设置该账号并发上限，配好模型映射（对外模型名 → 上游模型名），并将协议模式显式设为 `force_chat_completions`。
2. **建分组并配模型白名单**：新建"研发"分组，加入对外模型名（客户端实际填写的名字）。
3. **建用户**：为每位研发人员建独立用户，余额填占位值；作为额度与统计的归属容器，本人不登录。
4. **设每日额度**：通过 `PUT /api/v1/admin/users/:id/platform-quotas` 为该用户设置 `platform` 与每日 Token 上限（周/月留空）。
5. **发 Key**：管理端代创建 Key 并指定归属用户与分组；Key 明文在创建响应中取得，按模板手工交付。

日常运行：

- **看用量**：用量页面按用户、Key、模型、时间筛选查看输入/输出 Token 与耗时。
- **调额度**：改某人的每日上限，同接口重设；需清零当日计数则调 `POST /api/v1/admin/users/:id/platform-quotas/reset`。
- **停某人**：停用其用户记录（该人整体失效），或单独停用其某把 Key（需新增的管理端接口）。
- **停某把 Key**：首版依赖用户停用；单 Key 停用能力需新开发。

## 5. 易错点

以下三项的共同特征是**配置错误不会报错，只会静默降级或锁死**：

1. **额度填 `0` 表示完全禁用，不是不限额。** schema 注释明确：`nil` 为无限额，`0` 为完全禁用（任何请求都被拒，因 `usage >= 0` 恒成立）。填 `0` 会锁死该用户。
2. **`openai_responses_mode` 不显式设置会走协议转换。** `auto` 或缺失时，未探测即按"支持 Responses"处理，与私有模型只有聊天端点的实际情况相反。
3. **余额为 0 会拒绝所有请求。** 标准模式下 `balance <= 0` 即拒，建用户时余额必须为正；本方案中用占位值满足该前提。

## 6. 未闭合事项

| 事项 | 性质 | 说明 |
| --- | --- | --- |
| **`reasoning_content` 与 `tool_calls` 是否同现** | **需运行验证（最高优先）** | 决定思考模式能否开启；同现则客户端第二轮可能 400。见第 3.5 节 |
| 上游是否返回 usage token 数 | 需运行验证 | 透传路径已强制 `include_usage`，待确认上游配合程度 |
| 透传路径是否真被走到 | 需运行验证 | 层级 1 可验，不依赖真实上游 |
| RPM 完整执行链 | 需静态核对 | 已确认字段与更新路径，未确认哪个入口、哪种模式、超限返回什么 |
| 缓存字段是否需补映射 | 需设计确认 | Sub2API 不解析 DeepSeek 缓存字段；是否改造取决于对额度口径的要求 |
| 额度计量改造的具体范围 | 需设计确认 | 累加点与判断点已定位，改造量取决于字段类型与入口集中度 |
| `platform` 映射方案 | 需设计确认 | 借用白名单 key 的长期可维护性 |
| `*_usd` 字段的页面呈现 | 需设计确认 | 字段名仍为 `*_usd`，界面需确认如何显示为 Token |
| 列表接口明文暴露的取舍 | 需设计确认 | 见第 3.3 节 |
| vLLM 部署要求清单 | 需决策 | 模块一依赖的推理服务配置（工具调用解析、思考模式、模型名、接口协议），需与模块二对齐 |
| 源码纳入仓库方案 | 需决策 | 固定版本、代码位置、许可证保留与后续同步方式 |

## 7. 证据索引

支撑证据均为临时产出，位于 `docs/temp/research/`，后续统一清理。

| 主题 | 文档 |
| --- | --- |
| 配额与限流机制、两套体系、计价非线性 | `temp/research/quota-mechanism-review.md` |
| Key 明文、管理端能力、工具调用透传 | `temp/research/sub2api-mvp-blockers-review.md` |
| 逐环节操作链、用量与停用证据 | `temp/research/sub2api-workflow-review.md` |
| 私有模型接入、并发、Simple Mode | `temp/research/sub2api-source-review.md` |
| 管理员 Key 能力边界 | `temp/research/sub2api-admin-key-review.md` |
| 七项选择的讨论过程 | `temp/research/module-1-first-release-scope.md`、`temp/research/module-1-mvp-decisions.md` |
| 候选比较与技术路线 | `temp/research/source-review.md`、`temp/research/route-options.md` |

所有 Sub2API 证据固定到 `bdb42e22f81fcb633ff0a060961211dd2bcb515b`，均为静态阅读结果，不等于最新版能力或生产验证结论。

**第 3.5、3.6 节另有一类证据**：2026-09-15 对一次 Claude Code 会话（后端为 DeepSeek 官方服务）的原始响应统计，共 451 条响应。这类证据是**实测**而非源码阅读，但**实测对象是官方云服务，与自建的 vLLM 环境不同**，不可直接外推。外部来源：

- [DeepSeek Anthropic API Endpoint 路由风险](https://therouter.ai/news/deepseek-anthropic-api-format-routing/)
- [cc-switch Issue #3263：thinking block round-trip 导致 400](https://github.com/farion1231/cc-switch/issues/3263)
- [vLLM Reasoning Outputs 文档](https://docs.vllm.ai/en/v0.19.0/features/reasoning_outputs/)

证据文件清理后，本文第 1 至 6 节仍应自洽可读；如需长期留存某条证据，应在清理前将其结论并入本文。
