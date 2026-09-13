# 请求流程对比（源码阅读草稿）

## One API

从目录和 `controller/relay.go`、`middleware/distributor.go` 可推断：

```text
客户端请求
→ 路由进入 Relay
→ 认证/中间件处理
→ 根据模型和分组选择渠道
→ 调用 relay/adaptor 上游适配器
→ 失败时按重试次数选择其他渠道
→ 写入调用日志和统计
→ 返回普通或流式响应
```

## New API

源码线索显示流程更细：

```text
客户端请求
→ 用户/访问 Token 认证
→ 请求体大小、用户/模型限流
→ 渠道授权和约束检查
→ 渠道亲和/分组选择
→ Relay 上游调用
→ 按渠道错误和重试策略切换
→ 记录调用、配额、错误和审计数据
→ 返回响应
```

`middleware/audit.go` 的注释明确审计中间件位于用户认证之后、端点限流之前；这说明中间件顺序会影响权限、限流和审计结果。

## LiteLLM

基于 README、Proxy、Router 和相关目录，流程可概括为：

```text
客户端 OpenAI 格式请求
→ Proxy 认证/Virtual Key
→ 模型配置和 Router 选择部署
→ 负载均衡、重试或 Fallback
→ 调用具体 Provider（包括 vLLM/自定义 OpenAI 服务）
→ 计算 Token/成本并写入日志回调
→ 返回统一格式响应
```

## 待继续确认

- One API/New API 的上下文 Token 计算是否在网关层完成。
- 并发限制是否在请求进入 Relay 前获得可释放的租约。
- 多网关实例下 Redis、数据库和缓存分别承担什么一致性职责。
- LiteLLM 开源版的 Team/Role/预算和审计功能边界。
