# WeKnora × 课题智慧坊：v3.4 架构修订建议（知识库结合方案）

> 本文档供其他 Agent 参考。配套阅读：[REVIEW-SUGGESTIONS-v3.3.2.md](./REVIEW-SUGGESTIONS-v3.3.2.md)（总评审）。本文对应其 §3.2 技术栈规模修正的落地方案。情报核实时间：2026-09-22。

## 0. 事实基础（2026-09 实测/核实）

腾讯开源 **WeKnora**（github.com/Tencent/WeKnora）当前状态：

- 版本 **v0.8.0**（未到 1.0，迭代快），社区约 2.7 万星
- **MIT 许可证**；Go 后端 + Vue 前端
- 核心能力：多格式解析（docreader）、自适应三层分块+版本控制、**BM25+稠密向量混合检索**、Rerank 集成、GraphRAG（可选 Neo4j）、ReAct Agent、**官方自带 mcp-server**、Wiki 自治生成互链页面（行级 Diff/快照/回滚）、四级空间 RBAC + 细粒度 API Key
- 存储后端：PostgreSQL/pgvector、Elasticsearch、Milvus 均可切换
- 模型兼容：OpenAI/DeepSeek/Qwen/Ollama 等 20+；提供 Docker/Helm/离线私有化部署

## 1. 定位判断

**WeKnora 不是「再加一个组件」，而是能替换方案中两大块自研**：它的能力与方案 07（统一知识平台索引/检索层）+ 5.8（课题私有化知识库数据摄入与检索层）+ 06（rag-server MCP）几乎 1:1 对应，且 MIT 许可顺带缓解 MinIO/OnlyOffice 的 AGPL 传导隐患（见评审文档 §3.2-5）。

### 能力对照表

| 方案原设计 | WeKnora 对应 | 结论 |
|---|---|---|
| 自研 rag-server MCP（search_knowledge 等） | 官方 `mcp-server` | ✅ 直接部署，自研缩为权限过滤薄包装 |
| pgvector + bge-m3 自维护检索服务 | BM25+稠密混合检索、Rerank | ✅ 替换 |
| 数据摄入层（解析→分块→入库） | docreader + 三层分块 + 版本控制 | ⚠️ 部分替换：公式/表格保真需与 MinerU 实测对比 |
| 权限矩阵「检索前置过滤」 | 四级 RBAC + KB 归属 + API Key | ⚠️ 需 PoC 验证：public/premium/expert/internal 能否按库分域表达 |
| 沉淀层「待完善队列 + 人工审核」 | Wiki 自治生成 + Diff + 回滚 | ✅ 可复用为人工审核前台 |
| Milvus（>50万条目触发） | 原生支持切换 Milvus/ES | ✅ 触发条件不变，迁移成本由 WeKnora 吸收 |
| Dify（阶段0 底座） | 能力重合 | **二选一：选 WeKnora，Dify 删除或降为备选** |

### 保留自研的三块（WeKnora 不覆盖）

1. LangGraph 五阶段写作流水线 + `verify_citations`（它是知识库平台，不是写作编排引擎）；
2. doc-server 的 session_token 两段式鉴权与「AI初稿」水印；
3. ima/钉钉/腾讯文档三个 Knowledge Connector 镜像同步管道。

## 2. v3.4 具体修订（按章节）

### 2.1 阶段0 激活清单重写（分期路线 + 07 章）

```
原：Dify + 自研检索服务 + pgvector + MinIO + Vue3 自研前端（约10件）
改：WeKnora v0.8.0(pin) + Qwen3.5-9B(vLLM/Ollama) + bge-m3
    + PostgreSQL 16 + Nginx + Docker Compose（6件）
```

WeKnora 自带管理 UI——阶段0 连 Vue3 自研都可推迟。「零自研代码起步」从口号变事实，且避免 Dify 路线「后期替换知识库引擎」的回头路。

### 2.2 5.8 课题申报私有化知识库

- 检索层主选改为 **WeKnora（pgvector 后端）**；Milvus 保持为条目>50万触发项（由 WeKnora 原生切换）；
- 数据摄入：docreader 为主，**MinerU 保留为公式密集型学科降级位**；新增验收门禁：「50 份课题样本实测 docreader vs MinerU 表格结构保真率，差距>10% 则双解析器按学科路由」；
- 生成层/服务层（LangGraph 五阶段、引用强制、拒答机制）不变。

### 2.3 06 章 MCP 输出

- rag-server 上线时点从「阶段0 末」提前到**阶段0 周 2**（WeKnora MCP 开箱即用，对外演示增量最大）；
- 自研聚焦薄网关：domain 分域过滤 + 脱敏层 + 每客户端独立 Key 配额（APISIX/Nginx 层，原设计不变）。

### 2.4 版本治理（本结合方案的硬代价）

v0.8.0 未到 1.0，接口 break 风险真实存在。复用方案已有的「A/B 50 题差异率<5%」防双轨门禁，改造为 **WeKnora 升级门禁**：每次升版本跑 50 题基线（Recall@5 基线 97.5% 口径），过门禁才升。三条纪律：

1. **pin 小版本**，升级窗口 = 阶段验收点；
2. 自研代码只走 WeKnora 公开 API / MCP，**不 fork、不改源码**；
3. 模型换型槽位制同样适用于 WeKnora 自身（评测集不回退才换）。

### 2.5 成本模型联动（对应评审文档 §1.1/§1.2）

- 阶段0-1 人力从约 3.5 人月降至 **≈2 人月**（省检索服务自研 + 管理 UI）；
- WeKnora Go 服务资源占用低，9B 推理 + WeKnora 同机可跑，GPU 采购可压到入门级卡起步，与「GPU 投入与验收门禁绑定」建议叠加后阶段2 前硬件投入可≈0。

## 3. 风险清单

1. **成熟度**：公开测评有「检索找对、模型答错」案例——评测门禁不可省；
2. **GraphRAG 克制启用**：Neo4j 是又一个有状态组件，**阶段0-2 明确关闭**，触发条件示例：「跨库实体关联查询月均>500 次」；
3. **RBAC 表达能力未验证**：四级空间矩阵能否承载「角色×库×条」三轴，是结合能否成立的唯一硬前提——**阶段0 第一周完成映射 PoC**；若仅支持库级，则用「按可见域分库」兼容；
4. **对外 SaaS/私有化交付时复查依赖**：MIT 主体安全，但需例行审计其 `third_party/` 与 `licenses/` 目录的依赖许可传导。

## 4. 一句话结论

**WeKnora 是 v3.4 该做的最大一处减法**：替换 Dify+自研检索层后，阶段0 运行件 10→6、人力砍约 40%、rag-server MCP 提前一个月上线，MIT 许可顺带解掉 AGPL 传导隐患；代价是 v0.8 版本管理纪律与一次 RBAC 映射 PoC。

---

*核实来源：Tencent/WeKnora 仓库（VERSION=0.8.0、README_CN、目录含 mcp-server/docreader/miniprogram/helm）、2026-09 多篇中文技术报道与实测视频。引用前请重新核实版本状态。*
