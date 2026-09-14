# 后台操作与限制机制核对

日期：2026-09-14。仅静态阅读，未部署/测试。沿用 [源码复核](source-review.md) 固定提交；本轮没有更新上游版本。本文细化 [最小改造清单](minimal-adaptation.md)，不代表最终选型。

## 证据索引

- N4：[web/src/features/keys/api.ts](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/web/src/features/keys/api.ts)、[controller/token.go](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/controller/token.go)、[router/api-router.go](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/router/api-router.go)
- N5：[web/src/features/users/api.ts](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/web/src/features/users/api.ts)、[web/src/features/channels/api.ts](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/web/src/features/channels/api.ts)、[web/src/features/usage-logs/api.ts](https://github.com/QuantumNous/new-api/blob/04c64734cc7c2e58a9efd0b247182330f3f52cce/web/src/features/usage-logs/api.ts)
- L4：[ui/litellm-dashboard/src/components/networking.tsx](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/ui/litellm-dashboard/src/components/networking.tsx)、[litellm/proxy/management_endpoints/key_management_endpoints.py](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/proxy/management_endpoints/key_management_endpoints.py)
- L5：[litellm/proxy/hooks/__init__.py](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/proxy/hooks/__init__.py)、[litellm/proxy/utils.py](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/proxy/utils.py)、[litellm/proxy/common_request_processing.py](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/proxy/common_request_processing.py)
- L6：[litellm/router.py](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/router.py)、[litellm/proxy/_types.py](https://github.com/BerriAI/litellm/blob/30f33a949b8a2bb890a2baee18e2ab7ab015a4f7/litellm/proxy/_types.py)

## 1. 后台到接口：可以复用，但须区分操作人

### New API

N4 的前端 API 封装明确调用：
- 创建 Key：POST /api/token/ → AddToken。
- 修改 Key：PUT /api/token/ → UpdateToken。
- 禁用/启用：PUT /api/token/?status_only=true → UpdateToken 的状态分支。
- 删除：DELETE /api/token/:id → DeleteToken。
- 取回完整 Key：POST /api/token/:id/key；创建接口本身返回成功状态，不能假设同时返回明文 Key。

路由对这些接口挂载 UserAuth 和 TokenOperationAudit。AddToken 用当前身份的 id 填充 UserId；UpdateToken/DeleteToken 按 Token ID 与当前用户 ID 查询归属。因此已闭合的是“登录用户管理自己的 Key”，不是“管理员为任意同事代发和管理 Key”。

N5 已找到用户增删改及管理接口调用、渠道配置接口调用、管理员与个人日志/统计查询。现有业务前端有可复用基础，不应新写一套相同页面。但前端 API 封装存在不等于每种角色都能看到全部按钮；本轮未逐个追踪组件显示条件。

最省改造的候选流程是管理员管理账号与渠道，同事登录后自助创建自己的 Key。若要求管理员集中代发且记录为每位同事所有，需要再查管理入口是否已有该能力。不能用管理员自有 Key 改名字冒充真实用户归属。

### LiteLLM

L4 的 networking.tsx 有 /user/new、/key/generate、/key/delete、/model/new、/spend/logs/ui 等调用。后端 /key/generate 注册认证依赖，请求含 user_id、team_id、duration 和 models，进入权限校验函数，而非无条件允许给任何人发 Key。

可以继续按复用现成后台推进；尚未完整闭合所有角色下的代发、禁用页面与模型创建授权路径。当前证据不足以承诺所有操作均免费、零改造可用。

## 2. 并发：LiteLLM 默认 hook 指向 v3，New API 暂未闭合

L5 的 PROXY_HOOKS 将 parallel_request_limiter 映射到 v3；LEGACY_MULTI_INSTANCE_RATE_LIMITING=true 会改为旧实现。_add_proxy_hooks 遍历映射、实例化 hook 并注册 callback，不能仅因 utils 中另有旧 limiter 对象就认定实际使用旧版本。

结合上一轮 L2：
- 配置 max_parallel_requests 才能形成有意义的名额上限；hook 注册本身不等于已配置限额。
- v3 有名额申请/释放及 Redis 原子操作。
- common_request_processing.py 的流式断连清理调用释放路径；这里是代码证据，未实际模拟断连。
- Redis 失效会降级本机控制；严格集群上限、进程崩溃回收和超长请求仍需专项验证。
- 本轮没有形成经过运行验证的可直接部署配置。

New API 检索 middleware、relay、setting 中 concurrency、semaphore、max_parallel 等非测试 Go 代码，命中主要为渠道测试并发设置；结合既有请求入口和用户速率限制，仍未闭合推理请求并发名额机制。结论保持“待确认”，不是全仓库不支持的证明。

## 3. 上下文：必须明确失败行为

LiteLLM Router 的 enable_pre_call_checks 默认 false。开启后，_pre_call_checks 在部署声明 max_input_tokens 且输入可计数时统计 Token，超限则排除该部署。

但所读代码还表明：
- 没有输入上限元数据时可跳过检查。
- 计数失败时返回原候选部署列表，不一定拒绝请求。
- 本路径比较输入 Token，不足以单独证明“输入 + 预留输出”总预算限制。
- 私有模型元数据、计数准确性和 fallback 选择仍影响结果。

因此可复用它作路由预检查，不能把它直接视为严格的公司上下文保护规则。若公司要求计数失败也拒绝、按模型精确总预算，可能需要其他现有机制或扩展，尚未决定实现方案。

New API 的已读路由与限流路径尚未确认类似统一 Token 上下文校验。字节上限、输出长度参数和计费预扣不能替代这个结论。

## 4. 授权边界

New API 基础操作文件属于已读 AGPLv3 范围，前端还注明可联系商业授权。未因这些基础路由发现必须付费的依据，但公司修改使用仍需满足相应许可条件。

LiteLLM 已读根 LICENSE 将 enterprise 目录单列授权。L6 的 Premium 元数据列表包含 tags、guardrails、policies、logging 等；L4 修改 Key 的路径按字段调用 _premium_user_check，生成路径也有 tags 的商业限制。此前认证中的 JWT Auth 检查仍成立。

这里的 logging 是特定 Key 元数据配置字段，不应扩大解释成“所有调用日志均收费”。同样，普通模型名授权与模型访问组也不能混为一谈。基本 Key 数据字段存在不能替代逐端点授权核对。

## 本轮对改造量判断的影响

| 项目 | 已减少的不确定性 | 仍影响选择的事项 |
| --- | --- | --- |
| New API | 自助 Key 创建/修改/禁用/删除的前端调用与后端路径已闭合；用户/渠道/统计有前端接口封装 | 集中代发、推理并发、严格上下文机制仍待确认 |
| LiteLLM | 四类后台操作有 API 调用；v3 hook 选择和注册机制更明确；流式断连有释放路径 | 完整后台角色/授权条件、模型配置链、严格上下文和多节点降级 |

建议仍以两者现成后台作候选。若接受同事自助创建 Key，New API 的已读流程可直接作为配置复用基础；若硬性要求平台并发治理，LiteLLM 当前实现证据更直接。两者均不能据此判定生产闭环已通过。

剩余工作应围绕决定性缺口推进，而不是重新泛读仓库：集中代发是否必要、并发控制要求的维度、上下文是否要求计数失败即拒绝，以及必要页面的免费功能边界。

