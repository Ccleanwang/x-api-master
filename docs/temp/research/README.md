# 技术调研记录

最新复核：2026-09-14 的 [源码证据](source-review.md) 与 [技术路线比较](route-options.md)。本次纠正了早期根据目录名称推断能力的结论；旧笔记如有冲突，以固定版本复核为准。所有材料均非运行验证或最终选型。

本目录用于保存 x-api-master 前期技术调研的临时产出，包括搜索关键词、候选项目清单和项目粗读笔记。

## 当前阶段

当前讨论入口：[第一个模块首版范围讨论稿](module-1-first-release-scope.md)。用户已表达以 Sub2API 作为脚手架的方向，开始梳理建议复用、补充和待决定的首版能力；具体范围与方案尚待审阅，部署实验继续暂缓。

最新进展：已完成 [Sub2API 最小使用流程核对](sub2api-workflow-review.md)，沿私有模型接入、用户授权、Key 发放、调用、用量查询和停用梳理复用范围，并记录条件性的最小改动候选。当前以 Sub2API 为优先评估对象，尚未锁定选型或需求边界。

Sub2API 的[初筛笔记](notes/sub2api.md)、[源码核对](sub2api-source-review.md)及[管理员 Key 能力核对](sub2api-admin-key-review.md)保留为证据。聊天转发与用户/账号并发有实现；Simple Mode 到期与额度检查、集中代发入口存在需要注意的边界。

比较候选为 New API、LiteLLM、Sub2API；前两者的[最小改造清单](minimal-adaptation.md)继续作为对照。One API 保留历史参考。技术栈未定，仍只做静态调研。

## 文件说明

- [module-1-first-release-scope.md](module-1-first-release-scope.md)：第一个模块的首版目标、操作流程、复用建议与待讨论选择。
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
