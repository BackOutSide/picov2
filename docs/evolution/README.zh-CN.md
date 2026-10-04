# Pico Evolution

本目录包含 Pico 自进化迭代的三份主文档：

- [业界调研与当前实现审计](self-evolution-survey.zh-CN.md)：区分反思、记忆、Prompt/Skill 优化、Workflow/Code/Weight 演进，并给出 Pico 源码级差距。
- [Local-First 改造计划](self-evolution-local-first-plan.zh-CN.md)：当前 canonical 计划；分别定义 Skill、Prompt、Runtime 的副作用、审批、实验评测、代码改动和 M0–M5 路线。
- [Evolution v2 历史方案](self-evolution-v2.zh-CN.md)：包含生产发布、灰度和回滚设计，当前项目尚无生产环境，因此仅作为远期参考。

核心决策：近期只建设 `Development → Candidate → Fresh Validation → Type-aware Approval → Local Baseline`。先完成 Skill，再接入一个结构化 Prompt slot，最后加固 Runtime；不建设生产部署系统。
