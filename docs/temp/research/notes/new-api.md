# New API

- GitHub：[QuantumNous/new-api](https://github.com/QuantumNous/new-api)
- 定位：统一 AI 模型聚合与分发平台，面向个人和企业模型管理。
- README 核心能力：OpenAI、Claude、Gemini 等格式互转，集中式模型管理和 API 分发。
- 技术线索：Go；支持私有化部署（具体部署方式待进一步阅读）。
- 与本项目关系：重点参考统一协议、多供应商/多模型管理、Key 和渠道抽象。
- 待确认：与 One API 的差异、管理后台功能、权限和配额模型、vLLM 私有服务接入方式。

## 目录与部署线索

- 后端同样是 Go 单体项目，目录包含 `controller/`、`middleware/`、`model/`、`relay/`、`monitor/` 和 `common/`。
- 相比 One API，README 和目录显示其增加了会话安全、请求体限制、系统监控、审计日志、限流器和可选日志数据库等治理能力。
- Docker Compose 默认使用 PostgreSQL 和 Redis，也可切换 MySQL；日志可选独立 PostgreSQL 或 ClickHouse。
- 初步架构判断：Go 单体管理/转发服务 + PostgreSQL/MySQL + Redis，日志和业务数据可以分离；支持多节点部署配置。
