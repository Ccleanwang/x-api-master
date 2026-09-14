# GitHub 候选项目

新增候选：用户提出的 [Sub2API](notes/sub2api.md) 已完成 README/依赖/部署初筛，待核对私有模型接入、并发和简化模式。当前为 New API、LiteLLM、Sub2API 三候选；第三项证据深度较浅，不直接并列判定能力已支持。

2026-09-14 范围更新：当前主候选为 New API 与 LiteLLM；One API 转为历史参考，其余项目暂缓深入。下表保留首轮筛选记录，当前优先级以此说明和 [最小改造清单](minimal-adaptation.md) 为准。

## 状态

第一轮候选清单。项目定位和技术信息来自 GitHub 仓库公开信息，尚未进行 README 深度阅读或最终选型。

## 记录表

| 项目 | GitHub 地址 | 调研路线 | 解决的问题 | README 核心能力 | 初步技术栈 | 是否继续研究 |
| --- | --- | --- | --- | --- | --- | --- |
| LiteLLM | [BerriAI/litellm](https://github.com/BerriAI/litellm) | 统一 API 和管理平台 | 统一调用多种 LLM API | OpenAI 格式、成本统计、路由、负载均衡、日志 | Python / Rust | 是 |
| One API | [songquanpeng/one-api](https://github.com/songquanpeng/one-api) | 统一 API 和管理平台 | API 管理与模型分发 | 多供应商适配、Key 管理、统一 API、Docker 部署 | JavaScript | 是 |
| New API | [QuantumNous/new-api](https://github.com/QuantumNous/new-api) | 统一 API 和管理平台 | AI 模型聚合与分发 | OpenAI/Claude/Gemini 兼容、企业模型管理 | Go | 是 |
| Portkey Gateway | [Portkey-AI/gateway](https://github.com/Portkey-AI/gateway) | 统一 API 和管理平台 | AI API 路由和治理 | 多模型路由、Guardrails | TypeScript | 待粗读 |
| Kong | [Kong/kong](https://github.com/Kong/kong) | API 网关 | 通用 API 和 AI 网关 | 网关、插件和 AI Gateway | Lua | 待粗读 |
| Apache APISIX | [apache/apisix](https://github.com/apache/apisix) | API 网关 | 云原生 API 和 AI 网关 | 动态路由、插件体系 | Lua / Nginx | 待粗读 |
| vLLM | [vllm-project/vllm](https://github.com/vllm-project/vllm) | 模型部署和运行 | 高吞吐 LLM 推理服务 | 模型推理、OpenAI 兼容服务 | Python / CUDA | 是 |
| KServe | [kserve/kserve](https://github.com/kserve/kserve) | 模型部署和运行 | Kubernetes 上的模型推理平台 | 分布式、多框架、可扩展推理部署 | Go / Kubernetes | 是 |
| Ray | [ray-project/ray](https://github.com/ray-project/ray) | 模型部署和运行 | 分布式 AI 计算和服务 | 分布式运行时、AI 服务库 | Python / C++ | 待粗读 |
| BentoML | [BentoML/BentoML](https://github.com/BentoML/BentoML) | 模型部署和运行 | AI 模型 API 服务 | 推理 API、队列、多模型流水线 | Python | 待粗读 |
| NVIDIA GPU Operator | [NVIDIA/gpu-operator](https://github.com/NVIDIA/gpu-operator) | GPU 与集群调度 | 在 Kubernetes 中配置和管理 GPU | GPU 驱动、运行时和监控管理 | Go / Kubernetes | 是 |
| Volcano | [volcano-sh/volcano](https://github.com/volcano-sh/volcano) | GPU 与集群调度 | 云原生批处理和任务调度 | Kubernetes 批处理调度 | Go / Kubernetes | 待粗读 |
| Kueue | [kubernetes-sigs/kueue](https://github.com/kubernetes-sigs/kueue) | GPU 与集群调度 | Kubernetes 原生任务排队 | Job Queueing、资源配额 | Go / Kubernetes | 待粗读 |
| Langfuse | [langfuse/langfuse](https://github.com/langfuse/langfuse) | 监控、日志与运维 | LLM 应用追踪和评估 | Trace、评估、LLM 可观测性 | TypeScript | 待粗读 |
| Helicone | [Helicone/helicone](https://github.com/Helicone/helicone) | 监控、日志与运维 | LLM 可观测性 | 请求监控、评估和实验 | TypeScript | 待粗读 |

## 调研路线

- 统一 API 和管理平台
- 模型部署和运行平台
- GPU 与 Kubernetes 集群平台
