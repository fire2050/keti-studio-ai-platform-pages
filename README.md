# 课题智慧坊-AI智能平台 · 静态方案总览

本仓库是 **课题智慧坊-AI智能平台** 的公开静态展示页（GitHub Pages）。

- 在线预览：https://fire2050.github.io/keti-studio-ai-platform-pages/
- 完整架构设计方案与系统源码位于私有仓库 `keti-studio-ai-platform`（不公开）。
- 版本：v3.3.2（四角色体系 + 课题申报私有化知识库 5.8 + 课题中心与公告栏 5.9 + 前端界面与账号体系收口）。
- 动态交互演示版（FastAPI 服务）已上线：https://keti-workshop-ai-2026.app.workbuddy.host/

## Agent 参考文档

- [REVIEW-SUGGESTIONS-v3.3.2.md](./REVIEW-SUGGESTIONS-v3.3.2.md) — 架构方案 v3.3.2 评审意见与修改建议（含成本/ARPU 模型、技术栈评估、v3.4 落地顺序）。
- [WEKNORA-INTEGRATION-v3.4.md](./WEKNORA-INTEGRATION-v3.4.md) — 腾讯开源 WeKnora（v0.8.0，MIT）与知识库体系的结合修订方案，落地评审文档的技术栈收敛建议。

## 方案演进史

| 版本 | 时间 | 核心变更 |
|------|------|----------|
| v2.5 | 2026-08 | 轻量化分期路线（阶段0-3 · 量化验收门禁）；知识库与文档撰写 MCP 对接架构图（rag-server 只读先行 / doc-server 读写守门） |
| v2.6 | 2026-09 | 统一知识平台 · 多源知识联邦：自建 / ima / 钉钉 / 腾讯文档四源六层架构，镜像同步与检索代理双形态 |
| v3.0 | 2026-09 | 用户体系重构：四角色（ADMIN / EXPERT / PREMIUM / BASIC）× 三端工作台，RBAC→ABAC→额度三层判定，表驱动权限 |
| v3.1 | 2026-09 | 课题申报私有化知识库（5.8）：MinerU + BGE-M3 + Milvus + Ollama + LangGraph，Grant-Master 五阶段写作流水线 + 引用验证 |
| v3.2 | 2026-09 | 课题中心与公告栏信息架构重构（5.9）：公告栏独立一级栏目、课题申报六步动线、撰写中心三栏七区、新增 policy / doc_template / doc_draft 三表 |
| v3.3.2 | 2026-09-18 | 前端界面与账号体系收口（对应线上 v12/v13）：登录分栏重构、公开注册页、管理员「账号与角色」、全站 UI 一致性规范 |
| v3.3.2+ | 2026-09-23 | 本展示页 P2 收口：AI 审计第七项术语统一为「学术语体规范化」、章节编号连续化（01–13）、页脚版本对齐 v3.3.2；发布评审与 WeKnora 结合两份 Agent 参考文档 |
| v3.4（规划） | — | 技术栈收敛（三运行时→一主、三检索轨→两轨）+ WeKnora 替换 Dify + 自研检索层，详见两份 Agent 参考文档 |
