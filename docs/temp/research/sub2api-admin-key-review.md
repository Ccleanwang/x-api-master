# Sub2API 管理员 Key 能力核对

日期：2026-09-14。版本：`bdb42e22f81fcb633ff0a060961211dd2bcb515b`。仅静态阅读，未运行测试或部署。按用户要求只核对现有能力，不确定需求边界，不设计改造方案。

## 结论

所读内置后台及注册路由尚未形成“管理员指定用户并创建 Key”的完整功能。已确认的是管理员管理用户、查看用户 Key、调整 Key 分组和通过接口重置限速用量。普通 Key 创建/修改/删除由当前登录身份决定归属。

这个结论针对本次固定版本的内置路由和页面，不代表所有插件、外部分支或未来版本绝无代发能力。

## 能力表

| 操作 | 本次核对结果 | 具体边界 |
| --- | --- | --- |
| 管理员创建用户 | 有实现 | CreateUser 创建用户、设置角色/并发/RPM/允许分组等，未在该函数发现自动生成模型 Key |
| 查看指定用户已有 Key | 有页面和接口 | UsersView 打开 UserApiKeysModal，调用 GET /admin/users/:id/api-keys |
| 给已有 Key 改分组 | 有页面和接口 | 弹窗分组选择器 → PUT /admin/api-keys/:id → AdminUpdateAPIKeyGroupID |
| 重置 Key 的用量窗口 | 有管理员接口 | reset_rate_limit_usage 重置 5h/1d/7d 用量与窗口起点；不等于修改限额值，未在所读弹窗找到按钮 |
| 为指定用户新建 Key | 所读管理员入口未发现 | /admin/api-keys 仅注册 PUT；用户管理路由未见代创建；普通 POST /keys 使用当前 subject.UserID |
| 任意修改他人 Key 名称、状态、到期或配额 | 所读管理员接口未提供 | 管理员更新请求只接收 group_id、reset_rate_limit_usage；普通更新服务检查 owner |
| 删除他人 Key / 转移归属 / 一键轮换 | 所读路由与页面未发现 | 删除自己 Key 的接口不能算管理员代删；新建再删除也不等于有原子轮换能力 |
| 管理操作审计 | 路由有挂载 | /admin 在 AdminAuth 后挂 AuditLog，再经过 AdminComplianceGuard；未验证每种事件的最终落库与脱敏 |
| 分组变更后的缓存失效 | 有调用 | 管理服务调用 InvalidateAuthCacheByKey；未测跨实例传播时延 |

## 为什么“管理员能看见”不等于“管理员能代发”

管理员用户弹窗展示已有 Key、状态、创建时间等，并提供改分组动作；Key 文本在页面上截取显示。不能由可见性推出创建或编辑权限，也不能由页面截断推出后端脱敏。

普通接口：
- POST /api/v1/keys → APIKeyHandler.Create → APIKeyService.Create(ctx, subject.UserID, request)。
- PUT /api/v1/keys/:id → Update(ctx, keyID, subject.UserID, request)；服务比较 apiKey.UserID 与 userID。
- DELETE /api/v1/keys/:id 同样传当前用户 ID。
- 管理员登录后调用这些普通接口，不能据其角色推导可给请求体中的任意 user_id 发 Key。

底层 Create 函数接收 userID，检查用户存在、分组可绑定、IP 规则等，并按传入 userID 落库。说明生成能力可接受一个用户标识，但这只是服务函数结构，不是已有授权的管理员代发接口。是否扩展及如何扩展留待需求讨论。

## 管理员改分组的细节

AdminUpdateAPIKeyGroupID：
- group_id 缺省不改；0 解绑；正数绑定目标活跃分组。
- 对订阅分组检查用户有效订阅。
- 对专属标准分组可同时授予用户分组访问权限；有事务路径，并返回 auto_granted_group_access。
- 更新分组并使认证缓存失效，不修改 Key 的 UserID。
- reset_rate_limit_usage 是另一步更新；不应把两个动作推定为整体单一事务。

因此，分组调整可能影响用户访问资格，并非只改一段标签。这里的分组是产品访问/路由概念，尚不映射为公司的团队需求。

## 核对范围与证据

本轮查看 admin/user 路由注册、管理员用户与 Key 处理器、对应服务、普通 Key 服务、管理员前端 API 和用户 Key 弹窗；搜索相关创建/代登录入口。未运行 UI，不全面审计插件。

- [backend/internal/server/routes/admin.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/routes/admin.go)
- [backend/internal/server/routes/user.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/server/routes/user.go)
- [frontend/src/components/admin/user/UserApiKeysModal.vue](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/components/admin/user/UserApiKeysModal.vue)
- [frontend/src/api/admin/apiKeys.ts](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/src/api/admin/apiKeys.ts)
- [backend/internal/handler/admin/apikey_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/admin/apikey_handler.go)
- [backend/internal/handler/admin/user_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/admin/user_handler.go)
- [backend/internal/service/admin_group.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/admin_group.go)
- [backend/internal/service/admin_user.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/admin_user.go)
- [backend/internal/handler/api_key_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/api_key_handler.go)
- [backend/internal/service/api_key_service.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/api_key_service.go)

现阶段应将“管理员集中代发”从笼统待确认，细化为“本轮核对的内置路由/后台未找到完整入口，底层存在按用户创建的服务函数”。不据此自动排除 Sub2API，也不预先决定需要多少改造。

