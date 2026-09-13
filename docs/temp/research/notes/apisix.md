# Apache APISIX

- GitHub：[apache/apisix](https://github.com/apache/apisix)
- 定位：动态、高性能、云原生 API Gateway，也支持 AI Gateway。
- README 核心能力：动态路由、负载均衡、重试、熔断、认证、Token 限流、可观测性和 Kubernetes Ingress。
- 技术线索：基于 NGINX 和 etcd；Lua 插件体系；支持 Docker、Kubernetes 和多种外部服务发现。
- 与本项目关系：可作为统一 API 入口和通用流量治理基础；业务管理、模型目录和用户体系仍需平台层实现。
- 初步判断：基础设施网关候选，适合与 LiteLLM/自研管理层进行组合评估。
