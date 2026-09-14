# Sub2API 最小使用流程核对

日期：2026-09-14；状态：调研草稿，待用户审阅。

固定源码版本：`bdb42e22f81fcb633ff0a060961211dd2bcb515b`。本轮只阅读源码，不部署、不执行上游测试、不调用模型服务。以下流程用于检查复用程度，不作为已批准的需求边界或技术选型。

## 1. 当前结论

Sub2API 的接入、用户管理、分组授权、用户/上游账号并发、用量查询和用户停用具备可复用代码与管理入口。以用户自助创建 Key 为前提，主要环节存在静态实现依据；管理员集中代发流程仍缺内置入口。因此可以优先评估，但不能表述为公司流程已经开箱即用或经过运行验证。

主要新增发现：

- 分组模型白名单有管理页面和转发前准入检查，不只是模型列表展示。
- 普通用户可由管理员停用，认证检查用户状态；不能等同于管理员单独停用他人的某一把 Key，也不保证中断已开始的生成。
- Key 总额度及 5h/1d/7d 窗口限制按 USD 金额计算，不是 Token 数量、请求次数或并发数。
- 公开分组默认可绑定；必须结合专属分组或用户的公开分组限制，才能正确理解允许分组配置。

## 2. 用日常操作串起流程

```text
管理员登记已运行的私有模型服务（地址、凭证、模型名称）
  → 配置访问分组并关联上游账号
  → 创建研发人员用户，分配分组权限及用户限制
  → 为该用户发放 Key【管理员代发缺内置入口；用户自助创建已有】
  → 研发人员在支持相应协议的工具中配置入口、Key、模型名
  → Sub2API 认证、检查分组/模型权限、申请并发名额并转发
  → 记录用量，管理员查询
  → 需要时停用普通用户，阻止其后续访问
```

这里的“上游账号”是 Sub2API 对模型服务访问配置的称呼，不是研发人员账号。“分组”用于访问与路由配置，不预先等同于公司部门。登记服务不包含安装模型、启动 vLLM 或分配 H200。

## 3. 逐环节复用核对

| 操作 | 分类 | 已有入口及调用链 | 条件和边界 |
| --- | --- | --- | --- |
| 登记私有模型服务 | 配置后复用候选 | CreateAccountModal → 管理员账号创建接口 → CreateAccount；有 base_url、凭证、group_ids、concurrency、模型映射与协议配置 [E1] | OpenAI APIKey 路径存在；仅有聊天协议的上游应检查 force_chat_completions；内网/HTTP 地址受出站策略影响，未连接实际服务 |
| 创建研发人员用户 | 现成页面及接口 | UserCreateModal → 管理员创建用户服务 [E2] | 创建用户不自动发模型 Key；可设置角色、初始余额、并发与 RPM |
| 分配可访问分组 | 配置后复用 | UserAllowedGroupsModal 提交 allowed_groups、restrict_public_groups；CanBindGroup 与认证时的分组校验 [E3] | 公开分组默认允许；专属标准分组需授权；订阅分组另有订阅逻辑，不能只看允许分组数组 |
| 限定能调用哪些模型 | 配置后复用 | GroupsView 创建/编辑 model_allowlist → GroupModelAllowlist 中间件 → Allows [E4] | 聊天请求按客户端模型名检查，未命中返回 404；未绑定分组或未启用白名单时放行。不是精确上下文限制 |
| 配置用户及上游并发 | 已有实现，可配置复用 | 用户编辑页、账号配置页 → 用户/账号 Concurrency → Redis 名额申请与释放 [E2][E5] | 正数生效，非正数不限；Key 活跃请求统计不等于 Key 并发限制；多个账号指向同一服务不自动合并总上限 |
| 配置 Key 到期与额度 | 自助管理已有；管理员配置能力不完整 | 普通 Key 创建/更新服务、APIKeyAuth、计费资格服务 [E6] | 普通更新校验归属；Simple Mode 跳过所读到期/额度检查。标准模式还关联余额/订阅及定价，不能只当成开关 |
| 管理员集中代发 Key | 需要扩展内置流程 | 底层 APIKeyService.Create 可接收用户 ID；已读管理路由无对应代发入口 [E7] | 需补管理端授权入口和页面；不能使用管理员自己的 Key 改名代替用户归属 |
| 研发工具调用 | 协议层可复用候选 | /v1/chat/completions、/v1/models、Bearer Key 认证及原生聊天转发 [E8] | 支持自定义地址且使用相应聊天协议的客户端才有直接接入基础；Responses、工具调用、流式细节须按实际客户端核对，未承诺所有编程工具兼容 |
| 查看使用记录 | 现成页面及接口 | UsageView → admin usage API → UsageHandler.List → UsageService → repository [E9] | 可按用户、Key、上游账号、分组、模型、时间过滤；页面使用输入/输出 Token、费用、耗时等字段。数据完整性依赖上游返回与记录链，不等同于保存全部对话或 GPU 监控 |
| 停止某个人继续使用 | 现成页面及接口 | UsersView 状态切换 → UpdateUser → 用户认证缓存失效 → APIKeyAuth 检查 User.IsActive [E10] | 普通用户可停用；服务保护管理员账号不被禁用。影响用户整体访问，不是单 Key 停用；跨实例传播和在途请求未运行验证 |

## 4. 配置中容易混淆的地方

### 分组权限与模型映射

用户是否能绑定分组、请求是否命中分组模型白名单、上游账号是否支持该模型，是三层不同检查。普通 APIKey 上游的空模型映射通常允许所有模型；开启 OpenAI 透传时 IsModelSupported 还会直接放行，不能把账号模型映射当成所有模式下统一的用户授权边界。

已读 /v1 路由先做 Key 认证，再挂分组模型白名单，之后进入合成路由和处理器。白名单检查客户端填写的公开模型名，而不是最终映射后的上游名称。WebSocket 有独立检查路径，本轮不将 HTTP 证据扩大为所有协议已核验。

### 金额额度、请求频率、并发、上下文

- Key Quota 与 RateLimit5h/1d/7d：按金额累计，字段注释明确 USD；这些窗口名称中的 RateLimit 不能解释为每秒请求数。
- RPM：每分钟请求数。本轮确认管理字段和更新路径；不同入口及运行模式的完整执行链仍需专项核对。
- Concurrency：同时占用的请求名额；用户和上游账号级已有路径。
- 上下文限制：一次请求能容纳的输入和输出 Token 总量；前轮未闭合统一严格校验，本轮不把它标为已支持。

私有模型可以有内部计量规则，但目前尚未确定。若沿用金额额度，需要继续核对自定义模型定价、计量来源与额度扣减行为；不能假设不收费、价格为零与资源配额控制等价，也不能承诺并发请求绝不超用余额。

Simple Mode 仍检查禁用状态、用户状态和分组等，但所读认证路径跳过 Key 有效期与额度。标准模式保留相关检查，同时带入余额/订阅条件。暂不推荐具体运行模式。

## 5. 最小改动候选清单

这是复用成本的初步结构评估，不是实施任务或工期承诺。只在后续确认相应需求时成立。

| 候选事项 | 初步修改位置 | 页面 / 数据库影响 | 上游升级维护关注 |
| --- | --- | --- | --- |
| 管理员给指定用户创建 Key | admin 路由、管理员处理器，复用 Key 创建服务 | 增加用户 Key 弹窗的创建/展示操作；现有 UserID/Key 字段可承载基本归属，暂未发现基本代发必须迁移表的理由 | 接入现有管理员认证、审计和创建校验；服务签名、权限与弹窗是需要跟踪的接触点 |
| 管理员停用/修改指定用户的 Key（若需要） | 管理员 Key 服务与处理器、弹窗 | 现有 Key 状态/配额/到期字段可复用；普通 Update 有 owner 检查，不能直接删除该检查 | 要保持普通用户只能操作自己的 Key；缓存失效与审计链需一并确认，单独停用不是轮换 |
| 私有上游、分组、并发及普通用户停用 | 优先使用现有配置与页面 | 当前没有足够证据要求新建页面或表 | 跟踪协议配置、公开分组默认值、模式及限额语义变化 |
| 自定义模型计量与额度 | 先继续核对定价配置/计费链 | 是否需要改代码与数据结构尚不确定 | 不应为省页面工作而直接改成 Simple Mode；统计 Token 与扣减金额须区分 |
| 指定编程工具兼容、严格上下文限制 | 尚未完成专项证据核对 | 待定，不能预估为小改动 | 可能涉及协议转换、模型分词及流式处理，不宜与新增管理按钮按同等成本估算 |

基本代发底层已有生成与持久化能力，有利于复用；但密钥如何呈现和交付、管理员允许设置哪些字段、是否保留用户自助入口，都留待需求边界讨论。没有据此预先增加批量导入、组织表、审批流或自动消息发送。

## 6. 固定版本证据

以下链接均固定到本轮版本；此前已经闭合的路径引用现有报告，不重复宣称本轮重新全面审计。

- E1：[CreateAccountModal.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/components/account/CreateAccountModal.vue)、[account_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/admin/account_handler.go)；协议与 URL 条件见[前轮源码核对](sub2api-source-review.md)。
- E2：[UserCreateModal.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/components/admin/user/UserCreateModal.vue)、[UserEditModal.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/components/admin/user/UserEditModal.vue)、[admin_user.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/admin_user.go)。
- E3：[UserAllowedGroupsModal.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/components/admin/user/UserAllowedGroupsModal.vue)、[user.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/user.go)、[api_key_auth.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/middleware/api_key_auth.go)。
- E4：[GroupsView.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/views/admin/GroupsView.vue)、[group_model_allowlist.go 中间件](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/middleware/group_model_allowlist.go)、[白名单服务](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/group_model_allowlist.go)、[account.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/account.go)。
- E5：[前轮用户/账号并发证据 S2](sub2api-source-review.md)。
- E6：[api_key.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/api_key.go)、[模式与认证证据 S3](sub2api-source-review.md)。
- E7：[管理员 Key 权限与调用链核对](sub2api-admin-key-review.md)。
- E8：[gateway.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/routes/gateway.go)、[聊天转发证据 S1](sub2api-source-review.md)。
- E9：[UsageView.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/views/admin/UsageView.vue)、[usage_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/admin/usage_handler.go)、[usage_service.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/usage_service.go)。
- E10：[UsersView.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/views/admin/UsersView.vue) 的状态切换，配合 E2 的 UpdateUser 与 E3 的 APIKeyAuth。

## 7. 后续讨论依据

本轮已足以把“复用成熟后台”细化为主要配置工作、集中代发的明确入口缺口、计量和客户端兼容的待核项。下一步可据此讨论实际需要哪些管理动作，再决定是否继续对计量或指定客户端做专项源码阅读。具体需求、运行模式、选型与 Spec 均未定稿。
