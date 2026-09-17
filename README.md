# Multica Architecture Analysis

基于公开源码的独立全仓库静态架构分析，覆盖控制平面、远程 daemon、Agent Runtime、Git worktree、Skills、插件、VCS、安全、部署与灾备。

- [完整分析报告](analysis.md)
- [Runtime 与远程 Code Agent 专题](runtime-code-agent.md)
- [系统全景架构](diagrams/system-architecture.html)
- [运行时组件与通信关系](diagrams/runtime-components.html)（[Archify JSON](diagrams/runtime-components.json)）
- [Agent 任务执行时序](diagrams/agent-task-execution.html)
- [任务生命周期](diagrams/task-lifecycle.html)
- [恢复与重新部署](docs/disaster-recovery.md)

分析基线：[multica-ai/multica@c1a61e1](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd)，日期为 2026-08-28。

本仓库是独立技术分析，并非 Multica 官方文档。未复制或托管完整 Multica 源代码；源码权利和许可归原项目及其贡献者所有。
