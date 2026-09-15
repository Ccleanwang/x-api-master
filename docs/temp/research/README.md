# 技术调研记录

最新复核：2026-09-14 的 [源码证据](source-review.md) 与 [技术路线比较](route-options.md)。本次纠正了早期根据目录名称推断能力的结论；旧笔记如有冲突，以固定版本复核为准。所有材料均非运行验证或最终选型。

本目录用于保存 x-api-master 前期技术调研的临时产出，包括搜索关键词、候选项目清单和项目粗读笔记。

## 当前阶段

当前讨论入口：[第一个模块首版范围讨论稿](module-1-first-release-scope.md)。用户已表达以 Sub2API 作为脚手架的方向，开始梳理建议复用、补充和待决定的首版能力；具体范围与方案尚待审阅，部署实验继续暂缓。

**该模块的结论已精炼至 [`docs/research/api-integration.md`](../research/api-integration.md)**，本目录保留其证据链与讨论过程，后续统一清理。

最新进展：已完成 [Sub2API 最小使用流程核对](sub2api-workflow-review.md)，沿私有模型接入、用户授权、Key 发放、调用、用量查询和停用梳理复用范围，并记录条件性的最小改动候选。当前以 Sub2API 为优先评估对象，尚未锁定选型或需求边界。

2026-09-15 更新：[配额与限流机制核对](quota-mechanism-review.md)回答首版范围讨论第 5 点的遗留问题。系统内存在两套独立配额体系；`user_platform_quotas`（用户 × 平台、自然日窗口、管理员接口已具备）在结构上可承载"每人每日 Token 额度"；计价含模型特定修正与倍率、并非按 Token 线性，**金额代理路线已放弃**。

同日的 [MVP 边界决策记录](module-1-mvp-decisions.md)已确定七项选择：研发人员不登录、管理员代创建与单 Key 停用、手工交付加说明模板、不做自动到期、做每人每日 Token 额度、做 RPM 且必须核实生效、上下文交给上游、优先支持编程 Agent。运行模式定为标准模式，额度由体系 B 承载，Token 口径为四类全计同等权重。

[首版阻塞项核对](sub2api-mvp-blockers-review.md)随后闭合两项前置：明文 Key 在 DTO 层完整返回，交付方案成立；工具调用可经原生聊天透传路径完成，但需显式配置 `openai_responses_mode=force_chat_completions`，否则 `auto` 默认走入协议转换路径。管理端确无代创建处理器，需新开发但可复用底层服务。剩余待验证项为上游模型的函数调用能力与 usage 返回行为，均需运行验证。

Sub2API 的[初筛笔记](notes/sub2api.md)、[源码核对](sub2api-source-review.md)及[管理员 Key 能力核对](sub2api-admin-key-review.md)保留为证据。聊天转发与用户/账号并发有实现；Simple Mode 到期与额度检查、集中代发入口存在需要注意的边界。

比较候选为 New API、LiteLLM、Sub2API；前两者的[最小改造清单](minimal-adaptation.md)继续作为对照。One API 保留历史参考。技术栈未定，仍只做静态调研。

## 文件说明

- [module-1-first-release-scope.md](module-1-first-release-scope.md)：第一个模块的首版目标、操作流程、复用建议与待讨论选择。
- [module-1-mvp-decisions.md](module-1-mvp-decisions.md)：七项选择的逐项结论、运行模式与配置形态、额度体系与 Token 口径。
- [quota-mechanism-review.md](quota-mechanism-review.md)：配额与限流机制核对，两套体系的分辨、"每人每日 Token 额度"的承载方案与金额代理路线的更正。
- [sub2api-mvp-blockers-review.md](sub2api-mvp-blockers-review.md)：Key 明文可见性与工具调用透传路径两项前置的核对结果。
- [sub2api-workflow-review.md](sub2api-workflow-review.md)：Sub2API 使用流程、配置条件、源码证据与条件性的最小改动范围。
- [sub2api-admin-key-review.md](sub2api-admin-key-review.md)：Sub2API 管理员与用户 Key 操作权限、接口和页面的现有能力核对。

最新操作路径核对：[operation-path-review.md](operation-path-review.md)，包括后台/API、Key 归属、并发 hook 启用、上下文失败行为与商业开关。

- `keywords.md`：需求关键词和 GitHub 搜索词
- `candidates.md`：候选项目清单及初步筛选结果
- `notes/`：候选项目 README 粗读笔记
- `source-review.md`：固定源码版本的证据与旧结论更正
- `minimal-adaptation.md`：当前两候选的闭环复用与改造清单
- `route-options.md`：基于当前人力条件的路线比较

本目录中的内容属于调研草稿。经过讨论确认的结论，后续应提炼到正式项目文档、技术设计或 Spec 中。
