# Pico Evolution v2

本目录包含 Pico 自进化迭代的两份主文档：

- [业界调研与当前实现审计](self-evolution-survey.zh-CN.md)：区分反思、记忆、Prompt/Skill 优化、Workflow/Code/Weight 演进，并给出 Pico 源码级差距。
- [完整迭代方法](self-evolution-v2.zh-CN.md)：以 Skill 为首个纵向切片，定义证据、fresh-holdout、可信执行、发布、灰度和回滚合同，以及 M0–M4 路线图。

核心决策：首版只打通 `Experience → Skill → fresh promotion → Release → rollback`；Prompt 随后接入，Runtime 生产部署与自由代码自修改后置。
