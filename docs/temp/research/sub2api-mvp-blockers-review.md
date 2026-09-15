# Sub2API 首版阻塞项核对：Key 明文与工具调用

日期：2026-09-15。固定版本：`bdb42e22f81fcb633ff0a060961211dd2bcb515b`，与前轮一致。仅静态阅读，未部署、未运行测试。本文回答 [MVP 边界决策记录](module-1-mvp-decisions.md) 第 8 节列出的两项前置问题，两项均涉及"若不成立则方案需重做"。

## 1. 结论摘要

| 问题 | 结论 | 影响 |
| --- | --- | --- |
| 交付方案依赖的明文 Key 是否可获得 | **可获得**。DTO 层完整映射 `Key` 字段，不脱敏 | 第 3 点交付方案成立 |
| 管理端是否有代创建设施 | **无**。管理端只有一个改分组函数 | 第 2 点需新开发，但可复用底层服务 |
| 编程 Agent 的工具调用能否透传 | **能**，但依赖账号模式配置 | 第 7 点成立，且**必须显式配置** `force_chat_completions` |

## 2. 明文 Key 与管理员能力边界

### 2.1 管理端只有一个函数

`backend/internal/handler/admin/apikey_handler.go` 全文 76 行，仅包含一个处理器：

```go
// UpdateGroup handles updating an API key's admin-managed fields.
// PUT /api/v1/admin/api-keys/:id
func (h *AdminAPIKeyHandler) UpdateGroup(c *gin.Context) { ... }
```

文件的请求结构体注释明确列出该接口可改的字段：

```go
type AdminUpdateAPIKeyGroupRequest struct {
	GroupID             *int64 `json:"group_id"`               // nil=不修改, 0=解绑, >0=绑定到目标分组
	ResetRateLimitUsage *bool  `json:"reset_rate_limit_usage"` // true=重置 5h/1d/7d 限速用量
}
```

管理端**没有创建、启用或停用 API Key 的处理器**。这与前轮 [管理员 Key 能力核对](sub2api-admin-key-review.md) 的结论一致，本轮从处理器文件层面再次确认。

### 2.2 明文 Key 在 DTO 层不脱敏

`backend/internal/handler/dto/mappers.go` 的 `APIKeyFromService` 完整映射 Key 字段：

```go
out := &APIKey{
	ID:     k.ID,
	UserID: k.UserID,
	Key:    k.Key,          // 完整明文，未截断、未掩码
	Name:   k.Name,
	...
}
```

同包的 `credentials_redact.go` 提供 `RedactCredentials`，但其注释说明适用范围是 `service.SensitiveCredentialKeys`——即**上游账号凭证**（账号的 API Key、OAuth token 等），与用户自己的 API Key 无关。

结论：管理员代创建接口若复用 `APIKeyFromService`，响应中即为完整明文 Key，手工交付方案可行。

### 2.3 附带发现：列表接口也返回完整明文

由于同一个映射函数同时用于列表展示，**查询某用户 Key 列表的接口响应中同样包含完整明文 Key**。前轮记录的"页面显示截断"只是前端展示行为，不代表后端脱敏。

这不是本次核对要解决的问题，但需在设计中明确取舍：若为安全而让列表接口截断，则创建后的明文展示与交付方案一并受影响；若保留现状，则应确认列表接口的访问权限边界（管理端有 `AdminAuth`、`AuditLog`、`AdminComplianceGuard` 三层中间件）。

### 2.4 代创建与单 Key 停用的实现路径

- **代创建**：底层 `APIKeyService.Create(ctx, userID, request)` 已接收 userID 并按该 ID 落库（前轮 [源码核对](sub2api-source-review.md) 证据 S4）。需新增的是**管理端授权入口与处理器**，复用该服务函数即可保证归属正确。
- **单 Key 停用**：管理端无对应处理器，需新增。同时须保持普通接口 `Update(ctx, keyID, subject.UserID, request)` 的 owner 校验不被移除，并一并确认认证缓存失效链。

两项新增功能与现有改分组处理器同属管理端 Key 操作，改造位置相近。

## 3. 工具调用与透传路径

### 3.1 透传路径不剥离任何请求字段

`backend/internal/service/openai_gateway_chat_completions_raw.go` 的 `forwardAsRawChatCompletions` 中：

```go
// 3. Rewrite model in body (no protocol conversion)
upstreamBody := body
if upstreamModel != originalModel {
	upstreamBody = ReplaceModelInBody(body, upstreamModel)
}
```

请求体以客户端原始字节为基础，**只做模型名替换**。函数中不存在字段剥离逻辑；平台特定的清理（Grok、Ollama Cloud 等）不适用于私有 OpenAI APIKey 账号。响应侧 `streamRawChatCompletions` 注释写明"透传上游 CC SSE 流到客户端"。

因此 `tools`、`tool_choice`、`parallel_tool_calls` 等字段原样到达上游，上游返回的 `tool_calls`（含流式 delta 分片）原样到达客户端。这是对工具调用最有利的路径。

### 3.2 走哪条路径由账号模式决定

`backend/internal/service/openai_gateway_forward.go`：

```go
func shouldForwardOpenAIResponsesViaRawChatCompletions(account *Account) bool {
	if account == nil || account.Type != AccountTypeAPIKey { return false }
	if account.IsOpenCodeGo() { return false }
	if account.IsCNProvider() {
		switch account.GetAPIProtocol() {
		case APIProtocolChatCompletions: return true
		case APIProtocolAdaptive:        return !account.SupportsNativeCNResponses()
		default:                         return false
		}
	}
	return !openai_compat.ShouldUseResponsesAPI(account.Extra)
}
```

对普通 OpenAI APIKey 账号，判定落在最后一行，其实现为：

```go
func ShouldUseResponsesAPI(extra map[string]any) bool {
	return ResolveResponsesSupport(extra) != ResponsesSupportNo
}
```

`ResolveResponsesSupport` 的返回规则：

| `openai_responses_mode` | 探测标记 | 返回值 | 是否走透传 |
| --- | --- | --- | --- |
| `force_chat_completions` | 任意 | `No` | **是** |
| `force_responses` | 任意 | `Yes` | 否 |
| `auto` 或缺失 | `openai_responses_supported = true` | `Yes` | 否 |
| `auto` 或缺失 | `openai_responses_supported = false` | `No` | 是 |
| `auto` 或缺失 | 标记缺失 | `Unknown` | 否 |

**关键点**：标记缺失时返回 `Unknown`，而 `ShouldUseResponsesAPI` 判定为 true，即**默认走协议转换路径，不是透传**。对私有模型服务而言，若未显式设置模式且探测未运行，请求会进入 CC → Responses 转换链。

### 3.3 转换路径也支持工具调用，但风险高于透传

`backend/internal/pkg/apicompat/chatcompletions_to_responses.go` 对工具字段有显式转换：

```go
// tools[] and legacy functions[] → ResponsesTool[]
if len(req.Tools) > 0 || len(req.Functions) > 0 {
	out.Tools = convertChatToolsToResponses(req.Tools, req.Functions)
}
// tool_choice: already compatible format — pass through directly.
if len(req.ToolChoice) > 0 { out.ToolChoice = req.ToolChoice }
```

且 `chatFunctionToResponses`、`convertChatToolsToResponses` 等函数处理函数调用与返回值。因此"转换路径不支持工具调用"的说法不成立。

但转换路径的**返回方向**（`responses_to_chatcompletions.go`）需把 Responses 事件流反向映射为 Chat Completions 的 `tool_calls` 增量，环节更多、出错面更大。透传路径没有这一层。**结论是两条都可用，透传更安全**，配置上应优先保证走透传。

### 3.4 配置结论

首版应为私有模型账号显式设置：

```
openai_responses_mode = force_chat_completions
```

理由：`auto` 模式的默认行为是"未探测即按支持 Responses 处理"，与私有模型（通常只有聊天端点）的实际能力相反，会静默走入转换路径。该设置正是前轮 [源码核对](sub2api-source-review.md) 提到"仅有聊天端点的私有服务若未设置正确模式，不能假设自动兼容"的具体机制。

另需注意：CN 供应商账号走的是 `api_protocol` 分支而非上述模式字段，两类配置不应混用（前轮已记录）。

### 3.5 无法由网关解决的依赖

透传只保证**网关不破坏**工具调用。上游推理服务本身是否支持函数调用，仍取决于该模型的部署与服务实现，属于运行验证范围，静态阅读无法确定。

## 4. 对首版边界的影响

| 决策项 | 状态变化 |
| --- | --- |
| 第 2 点：管理员代创建与单 Key 停用 | 代创建的底层能力已确认可复用，管理端入口需新开发；实现位置与现有改分组处理器相近 |
| 第 3 点：手工交付加说明模板 | **前置已闭合**，明文 Key 在响应中可用 |
| 第 7 点：优先支持编程 Agent | 网关侧成立，但**需显式配置 `force_chat_completions`**；上游模型能力仍待运行验证 |

## 5. 仍需处理

1. **列表接口明文暴露的取舍**：设计阶段需决定是否让列表响应截断，以及创建后的明文如何呈现。
2. **上游模型的工具调用能力**：需在跑通流程时以实际模型验证，不能由网关透传推导。
3. **`platform` 映射与页面呈现**：`user_platform_quotas.platform` 借用白名单 key 的方案、`*_usd` 字段显示为 Token 的处理，仍待设计确认（见 [MVP 边界决策记录](module-1-mvp-decisions.md) 第 7 节）。

## 6. 证据索引

固定到本文版本，均为静态阅读。

| 编号 | 文件与说明 |
| --- | --- |
| B1 | [handler/admin/apikey_handler.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/admin/apikey_handler.go)：管理端唯一 Key 处理器与可改字段 |
| B2 | [handler/dto/mappers.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/dto/mappers.go)：`APIKeyFromService` 完整映射 `Key` |
| B3 | [handler/dto/credentials_redact.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/handler/dto/credentials_redact.go)：脱敏适用范围为上游账号凭证 |
| B4 | [service/openai_gateway_chat_completions_raw.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/openai_gateway_chat_completions_raw.go)：请求体透传与 SSE 透传 |
| B5 | [service/openai_gateway_forward.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/openai_gateway_forward.go)：`shouldForwardOpenAIResponsesViaRawChatCompletions` |
| B6 | [pkg/openai_compat/upstream_capability.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/pkg/openai_compat/upstream_capability.go)：`ResolveResponsesSupport`、`ShouldUseResponsesAPI` 与模式常量 |
| B7 | [pkg/apicompat/chatcompletions_to_responses.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/pkg/apicompat/chatcompletions_to_responses.go)：转换路径的 tools 映射 |
| B8 | [service/openai_gateway_chat_completions.go](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/internal/service/openai_gateway_chat_completions.go)：入站路由分流顺序 |

本文不新增运行结论；固定版本的静态证据不等于最新版能力或生产验证结果。
