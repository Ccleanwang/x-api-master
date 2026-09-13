# LiteLLM

- GitHub：[BerriAI/litellm](https://github.com/BerriAI/litellm)
- 定位：开源 AI Gateway，同时提供 Python SDK；用统一的 OpenAI 格式访问大量模型。
- README 核心能力：统一接口、虚拟 Key、成本统计、Guardrails、负载均衡、日志、管理后台；支持 vLLM 和自定义 OpenAI 兼容服务。
- 技术线索：Python SDK/Proxy，README 也标注 Rust core；可通过 Docker/命令行运行 Proxy。
- 与本项目关系：与“统一 API 接入与业务管理”重合度最高，适合作为重点研究对象或底层能力候选。
- 待确认：开源版具体用户/团队权限、限流、上下文限制、数据存储和企业功能边界。

## 目录与部署线索

- 仓库同时包含 Python 代码、Rust 相关配置、UI、Helm 和大量 CI/E2E 测试，规模明显大于 One API/New API。
- README 将 LiteLLM 分为 Python SDK 和 Proxy Server 两种使用方式；Proxy 是面向团队的集中式网关。
- 重点能力围绕 provider 适配、统一 endpoint、Router、虚拟 Key、预算/成本、Guardrails、负载均衡和日志。
- 初步架构判断：以网关/代理为核心，外加数据库配置、管理 UI 和多种部署方式；适合作为模型调用层研究对象，业务后台是否完全匹配需单独验证。
