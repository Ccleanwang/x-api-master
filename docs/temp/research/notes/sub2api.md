# Sub2API：新增候选初筛

后续进展：[源码核对](../sub2api-source-review.md)已追踪私有聊天转发、用户/账号并发、Simple Mode 与自助 Key 管理。初筛待确认项以该文更新为准。

日期：2026-09-14。阶段：README、依赖清单与部署配置初筛；尚未达到 New API/LiteLLM 的源码核对深度。未部署、未运行测试。

## 来源

固定提交：`bdb42e22f81fcb633ff0a060961211dd2bcb515b`，本次 GitHub API 返回 archived=false；未核查长期维护趋势，不由此判断生产成熟度。

- [README](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/README.md)：Overview、Features、Tech Stack、Simple Mode、Project Structure。
- [后端依赖](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/backend/go.mod)与[前端依赖](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/frontend/package.json)。
- [部署配置](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/deploy/docker-compose.yml)与[许可证](https://github.com/Wei-Shaw/sub2api/blob/bdb42e22f81fcb633ff0a060961211dd2bcb515b/LICENSE)。

README 的赞助商广告不作为项目功能证据。未执行 README 中的安装脚本。

## 项目定位与相关性

项目自称面向订阅额度分发的 AI API 网关：用户通过平台 Key 调用上游账号，平台管理认证、计费、账号选择和转发。

与公司需求有交集，但上游主要抽象为云端账号/订阅，而公司核心资源是自建模型服务及 GPU。需要确认标准 API Key 账号能否配置任意内部地址和模型，而不能由“支持 OpenAI”直接推出“已支持私有 DeepSeek/GLM”。

## README 明确声明的能力（未验证实现）

- 多上游账号（OAuth、API Key）、平台 Key 分发。
- 用户级和账号级并发限制、请求与 Token 速率限制。
- Token 用量计费、管理后台、保持会话使用同一账号的路由策略。
- Composite Groups：将模型请求解析到具体提供方。
- 可通过 iframe 嵌入外部系统；只是页面嵌入，不等于统一权限/数据集成。
- Simple Mode：面向个人/内部团队，声明隐藏 SaaS 功能并跳过计费；RUN_MODE=simple，生产还要求 SIMPLE_MODE_CONFIRM=true。需核对它是否同时影响我们需要的额度和用量限制，不能直接视为可无损关闭商业功能。
- README 描述部分 Responses WebSocket 连接上限及 Redis 租约；它与逐请求用户/账号并发是不同限制，均待源码追踪。

## 初步架构和技术栈

请求路径的概念理解：调用方 → 平台 Key 认证/管理 → 账号选择与转发 → 上游服务。并非 GPU 调度平台。

| 部分 | 已读取线索 | 通俗说明 |
| --- | --- | --- |
| 后端 | Go、Gin、Ent；go.mod 声明 Go 1.27.0 | Gin 处理 HTTP，Ent 管理数据库对象；版本为仓库声明，未核验构建环境 |
| 前端 | Vue 3、Vite、TypeScript 工具链、Pinia；README 列 TailwindCSS | 现成 Web 后台，Pinia 管理页面共享状态 |
| 数据/共享状态 | PostgreSQL、Redis | 保存业务数据与共享计数等，具体一致性待查 |
| 部署 | Docker Compose、二进制/服务安装文档 | 可私有化部署网关，不等于能直接接入任意私有模型 |

## 许可证边界

LICENSE 为 LGPLv3，README 写 LGPLv3 或更新版本；README 另有“未向任何主体授权商业运营”的声明。不能只据该声明判定 LGPL 禁止商业使用，也不能忽略两处措辞差异。若考虑公司修改/部署，需结合具体文件许可及使用方式明确边界；当前不作授权可行性的最终结论。

## 纳入方式与优先核对

加入第三候选，New API/LiteLLM 不因此被淘汰。暂不与已有源码证据同等级打勾。

1. 私有上游：API Key 模式的 base_url、内网 HTTP/HTTPS、任意模型名和聊天/流式路径；是否依赖云端账号刷新或固定提供方。
2. 并发：用户和账号名额在哪里申请/释放；Redis 故障、断连、多个网关实例如何处理；一个账号能否自然映射一个私有模型服务。
3. 后台：账号、用户、Key、用量页面能否完成我们的闭环，是否支持集中代发或仅自助。
4. Simple Mode：隐藏/跳过的业务与仍保留的统计、权限和限额，避免为了省开发关闭必要治理。
5. 上下文：是否有按私有模型 Token 校验，不能把 WebSocket 超时或请求体大小当成上下文限制。
6. 许可与必要功能边界。

初步判断：值得深入；吸引点是现成后台、文档明确的并发能力与内部团队模式。决定性前提是私有模型适配，尚不能据 README 推荐替代 New API/LiteLLM。
