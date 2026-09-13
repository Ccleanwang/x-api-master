# One API

- GitHub：[songquanpeng/one-api](https://github.com/songquanpeng/one-api)
- 定位：LLM API 管理与分发系统，统一多个供应商 API，并提供 Key 管理和二次分发。
- README 核心能力：多模型供应商适配、统一 API、Key 管理、Docker 部署、开箱即用。
- 技术线索：GitHub 语言统计为 JavaScript；单可执行文件并提供 Docker 镜像。
- 与本项目关系：重点参考用户、Key、渠道/模型配置和管理后台的完整度，尤其适合中文团队理解业务平台形态。
- 待确认：项目实际后端组成、权限粒度、限流/并发/上下文限制，以及许可证和社区维护风险。

## 目录与部署线索

- 后端按 Go 项目组织，入口为 `main.go`，主要目录包括 `controller/`、`middleware/`、`model/`、`relay/` 和 `monitor/`。
- `controller/` 覆盖用户、渠道、模型、日志、计费等管理接口；`relay/` 按供应商适配上游模型。
- `common/rate-limit.go`、`common/redis.go` 和 `middleware/rate-limit.go` 表明使用 Redis 支持限流/共享状态。
- Docker Compose 默认包含 One API、Redis 和 MySQL；支持使用 SQLite，并提供多节点配置线索。
- 初步架构判断：单体 Go 服务 + 内嵌前端资源/管理台 + 关系数据库 + Redis + 多供应商适配层。
