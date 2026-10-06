# 课题智慧坊 · 多源接入详细设计（DES）

| 项 | 内容 |
|---|---|
| 文档编号 | KBS-DES-2026-001 |
| 文档名称 | 多源接入详细设计（ima / 钉钉 / 腾讯 / 自建库四现场） |
| 文档版本 | DES v1.0 |
| 编制日期 | 2026-10-03 |
| 上游依据 | 《课题智慧坊-AI智能平台·架构设计方案 v3.13》（2026-09-30，页面 v29，远端逐字节校验 blob c72cf648）；《知识库建设立项申请书 PR v1.3》附录A（本文档将其 A.1–A.4 展开为可施工细则） |
| 约束红线 | PR v1.3 §4.0「底座外借 · 壁垒自研 · 出口收敛」三红线；本文档不引入任何未拍板的底座假设 |
| 读者 | 后端/检索/集成开发、知识库运营、平台管理员 |

> **一句话总原则**（承接 PR 附录A）：**钉钉=对内现场，腾讯=对客现场，ima=个人沉淀现场，自建库=资产真值**。平台只做连接、索引与受控升级——「不搬迁只连接」。

---

## 0. 范围与阅读指引

### 0.1 本文档覆盖

- CanonicalDoc 统一数据模型的字段级定义与建表口径（§1）
- Knowledge Connector 适配器五接口的签名、语义与错误契约（§2）
- 三种接入模式（A 原生托管 / B 镜像同步 / C 检索代理）的判据与降级条件（§3）
- 镜像同步时序、三通道调度参数与幂等/冲突消解算法（§4）
- sediment 沉淀准入管线各节点判据（§5）
- 四角色 × 镜像内容权限映射的程序化判定算法（§6）
- 各源 API 调用面、配额与限流设计（§7）
- fail-closed 默认值、受控动作（holds）与 killswitch 接入（§8）
- 验收门禁与可观测指标（§9）

### 0.2 本文档不覆盖（属于其它工单）

- 底座二选一（MaxKB / Dify）的内部实现——本文档所有接口只依赖平台侧适配器抽象，底座可替换（PR §4.0 红线3）。
- 向量库/重排模型选型细节（v3.13 既有口径：bge-m3 + BGE-Reranker + Milvus/pgvector）。
- 微信生态（v3.8）、小程序前端形态。

### 0.3 关键名词对齐

| 名词 | 口径来源 | 含义 |
|---|---|---|
| 四角色 | v3.13 §02 | BASIC / PREMIUM / EXPERT / ADMIN，角色恒 4 |
| 四级可见域 | v3.13 §15 | public / premium / expert / internal |
| kbs/v1 层级梯度 | v3.5.1 kb_adapter | title→summary→teaser→content，未达层连键不存在，服务端裁剪，teaser 硬顶 120 |
| MCP 广场三表 | v3.13 阶段A | mcp_catalog / mcp_binding / mcp_cred，均无 content 列（R6 结构保证） |
| SoD | v3.11 阶段A′ | 确认者≠发起者；双人=两账号；≥2admin 自批返回 400；admin_action_hold 为真值 |

---

## 1. CanonicalDoc 数据模型

### 1.1 设计目标

CanonicalDoc 是四类源归一化后的**唯一中间表示**：适配器把各平台异构对象翻译为 CanonicalDoc，索引层、融合层、权限层、沉淀层**只认 CanonicalDoc**，不认识任何源平台字段。这样第三方差异全部吸收在接入层（v3.13 §07「上层只见一个检索接口」）。

### 1.2 字段级定义

| 字段 | 类型 | 必填 | 语义 | 取值/生成规则 |
|---|---|---|---|---|
| `doc_id` | TEXT PK | ✅ | 平台内稳定标识 | `uq(source, source_ref)` 生成，跨同步稳定 |
| `source` | TEXT | ✅ | 源枚举 | `self` / `dingtalk` / `tencent` / `ima`（+ `feishu` 第五源登记位，仅登记不启用） |
| `source_ref` | TEXT | ✅ | 源侧原生标识 | 钉钉 nodeId / 腾讯 fileId / ima 条目 ID；自建=主库 PK |
| `origin_url` | TEXT | ✅ | 回链到源现场 | 镜像**不承载正文真值**，点击回原文仍在源平台协作（§2.4 模式B铁律） |
| `title` | TEXT | ✅ | kbs/v1 第1层 | 直接映射 |
| `summary` | TEXT | ⚠️ | kbs/v1 第2层 | 无则**该键不存在**（不得置空串占位） |
| `teaser` | TEXT | ⚠️ | kbs/v1 第3层 | 服务端硬顶 120 字符，超长截断，不由客户端裁剪 |
| `content` | TEXT | ⚠️ | kbs/v1 第4层 | 仅服务端按可见域决定是否返回；**镜像原文 content 默认仅内部可见** |
| `acl` | JSONB | ✅ | 权限载体 | `{level, org_scope, roles_allowed[]}`，落值见 §6 |
| `acl.level` | TEXT | ✅ | 可见域档位 | 镜像入库**恒先取 internal**（§6.1 收敛不映射） |
| `content_hash` | TEXT | ✅ | 幂等键 | 规范化正文后的摘要（见 §1.3），非文件名 |
| `quality_score` | INT | ⚠️ | 排序加权 + 准入门禁 | 沉淀时计算（§5.3），纯镜像条目可为空、排序按缺省下权 |
| `stage` | TEXT | ⚠️ | 六阶段轴标签 | 仅当命中课题交付物；非法 stage 名一律 400（v3.3.4 口径） |
| `domain` | TEXT | ✅ | 分域过滤键 | policy / template / achievement / methodology / process，检索前置过滤 |
| `lang` | TEXT | ✅ | 语言 | 缺省 `zh` |
| `source_updated_at` | TIMESTAMPTZ | ✅ | 源侧更新时间 | 增量与对账判据 |
| `indexed_at` | TIMESTAMPTZ | ✅ | 平台索引时间 | 写入即记 |
| `sync_state` | TEXT | ✅ | 同步状态机 | `pending`/`mirrored`/`代理only`/`withdrawn`（§6.1 单向阀撤出即置 withdrawn） |
| `dedup_of` | TEXT FK | ⚠️ | SimHash 去重指向 | 命中重复时指向保留条目，不物理删除 |
| `version` | INT | ✅ | 版本化回滚 | 每次正文变更 +1，历史版本留表（可回滚，v3.13 缺 status/teaser/version 列的工单在 PR §4.4 登记） |

> **对齐 PR §4.4 在册缺陷**：v3.13 口径下 `knowledge` 表仍缺 `status / teaser / version` 三列、`sediment` 仍缺权限校验——本文档把这些字段列为**阶段0建表工单**必须先补齐的列，未补齐前 §5 沉淀管线不得对外写。

### 1.3 content_hash 规范化算法

幂等不能靠文件名（腾讯同名文件可重复）。规范化顺序固定：

1. 取正文文本，去 HTML/富文本标记，统一为 Markdown 纯文本；
2. Unicode NFC 归一化；行尾统一 `\n`；连续空白折叠为单空格；
3. 去源平台注入的水印行与动态时间戳行（正则白名单，避免把正文误删）；
4. `SHA-256(规范化文本)` → `content_hash`。

同步层比对：`content_hash` 不变 → 只刷 `source_updated_at` 等元数据，**不动 content、不改 acl**（§6.1 对账不自动放开）。

### 1.4 建表口径（PostgreSQL 16）

```sql
CREATE TABLE canonical_doc (
  doc_id             TEXT PRIMARY KEY,
  source             TEXT NOT NULL CHECK (source IN ('self','dingtalk','tencent','ima','feishu')),
  source_ref         TEXT NOT NULL,
  origin_url         TEXT NOT NULL,
  title              TEXT NOT NULL,
  summary            TEXT,
  teaser             TEXT,          -- 服务端生成，应用层保证 ≤120
  content            TEXT,          -- 镜像原文，默认 internal
  acl                JSONB NOT NULL,
  content_hash       TEXT NOT NULL,
  quality_score      INT,
  stage              TEXT,
  domain             TEXT NOT NULL,
  lang               TEXT NOT NULL DEFAULT 'zh',
  source_updated_at  TIMESTAMPTZ NOT NULL,
  indexed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  sync_state         TEXT NOT NULL DEFAULT 'pending',
  dedup_of           TEXT REFERENCES canonical_doc(doc_id),
  version            INT NOT NULL DEFAULT 1,
  UNIQUE (source, source_ref)
);
CREATE INDEX idx_cd_domain ON canonical_doc (domain);
CREATE INDEX idx_cd_acl_level ON canonical_doc ((acl->>'level'));
CREATE INDEX idx_cd_hash ON canonical_doc (content_hash);
CREATE INDEX idx_cd_updated ON canonical_doc (source, source_updated_at);
```

- 生产主库本地 PostgreSQL 16（v3.13 §02 数据库规划），demo 阶段 Supabase 需 **RLS 必开**、`service_role` 不下发前端、Storage 私有 + 签名 URL。
- 向量索引与全文索引另表（pgvector + OpenSearch/全文），通过 `doc_id` 关联；向量列不进本表，避免主表膨胀。

---

## 2. 适配器接口契约（Knowledge Connector）

### 2.1 五接口总览

每个源实现一个适配器，统一接口（v3.13 §07 接入层）。上层对四源调用完全同形：

```ts
interface KnowledgeConnector {
  list(cursor?: string): Promise<{ items: SourceRef[]; next_cursor?: string }>;
  fetch(ref: SourceRef): Promise<CanonicalDoc>;
  search(query: string, opts: SearchOpts): Promise<Hit[]>;
  write_back(doc: DraftDoc, target: string): Promise<{ ref: string; status: 'draft' }>;
  watch(cb: (evt: SourceEvent) => void): Subscription;
}
```

| 接口 | 职责 | 关键语义 |
|---|---|---|
| `list(cursor)` | 遍历白名单范围 | **必须游标分页**，禁止一次性全量拉取；cursor 不透明、源侧自定 |
| `fetch(ref)` | 单篇取正文并归一化 | 返回 CanonicalDoc；导出失败/无权限→抛 typed error（§2.5），不返回半篇 |
| `search(query,opts)` | 模式C检索代理 | 结果归一化为 `Hit{ref,title,snippet,score_raw,source}`；`score_raw` 保留源侧原分，融合层负责归一化 |
| `write_back(doc,target)` | 沉淀回写 | **只建草稿**（status 恒 `draft`），人工在源侧评审定稿；平台不代理用户身份改文档（§6.3 身份线3） |
| `watch(cb)` | 事件订阅 | Webhook 增量入口；源无 Webhook 时该适配器 `watch` 返回空订阅，靠定时增量兜底 |

### 2.2 各源实现差异（吸收在适配器内，不外泄）

| 源 | list | fetch | search | write_back | watch |
|---|---|---|---|---|---|
| 自建 self | 原生表游标 | 原生读 | **原生语义检索**（唯一具备） | 直写主库（走 sediment） | 内置事件总线 |
| 钉钉 | 节点树遍历 | markdown 导出（**定向开通**，未批则无此能力） | **无原生语义检索**→由平台补齐；适配器 `search` 仅结构遍历 | 创建文档/目录 | 部分事件，缺则定时兜底 |
| 腾讯 | drive 遍历 | 导出 | 原生 `GET /openapi/drive/v2/search`（关键字） | 全能力（新建/导入/编辑） | 支持 |
| ima | 库/条目订阅范围 | add_knowledge/import_urls 侧写 | `search_knowledge`/`search_knowledge_base` | add_knowledge（保持用户端行为一致） | 混合模式，见 §3.4 |

### 2.3 层级梯度约束（kbs/v1）

适配器 `fetch` 产出的 CanonicalDoc 在**返回给任何消费方之前**必须过 kb_adapter 服务端裁剪：

- 消费方可见域未达 `content` 层 → 响应 JSON 中 `content` **键不存在**（不是 null、不是空串）；同理 `teaser`、`summary`。
- 客户端**不得**传入"要哪一层"来突破——可见域由服务端 key/session 预置决定（对齐 v3.7 MCP：客户端传入一律 400）。
- `teaser` 超 120 由服务端截断，不由客户端处理。

### 2.4 三接入模式判据（承接 v3.13 §07）

| 模式 | 适用 | 数据落点 | content 是否入平台 |
|---|---|---|---|
| A 原生托管 | 自建库 | 直接写统一索引 | ✅ |
| **B 镜像同步（优先）** | 钉钉、腾讯、ima(混合) | 导出→归一化→入自建索引，保留 origin_url 回链 | ✅（默认 internal） |
| C 检索代理（兜底） | 导出受限/成本过高 | **不搬原文**，实时转发检索，结果归一化为 Hit 参与融合 | ❌（连 content 键都不产生） |

**决策规则**：能镜像则镜像，不能镜像才代理。镜像保证检索质量可控（统一分块/向量/重排），是"单一答案"前提；代理只补长尾。

### 2.5 适配器错误契约（typed，不返回半态）

`fetch`/`search` 失败必须映射为下列稳定错误码，供 §3 降级逻辑消费：

| 错误码 | 触发 | 后果 |
|---|---|---|
| `E_EXPORT_UNAVAILABLE` | 钉钉导出未开通/无权限 | 该篇→候选降级为模式C代理 |
| `E_QUOTA_EXCEEDED` | 腾讯免费额度打满 | 触发同步层限流错峰；检索标记"该源未参与本次" |
| `E_PERMISSION_REVOKED` | 源侧撤权 | doc 置 `sync_state=withdrawn`，即时生效（单向阀） |
| `E_TIMEOUT` | 单源超时 | 不阻塞整体返回，融合层记 degraded |
| `E_SCHEMA_DRIFT` | 源侧字段变更致归一化失败 | 隔离该条入死信队列，告警，不污染索引 |

---

## 3. 接入模式落地细则

### 3.1 钉钉（模式B为主）

- 无原生语义检索 → 镜像入自建索引后由平台 bge-m3 补齐检索能力。
- 三层权限继承（库/夹/单文档）落 `CanonicalDoc.acl` 时**只向下收敛**（子级不得比父级更宽，§6）。
- markdown 导出为**定向开放能力**：阶段0第1月初提交开通申请；**不通过则该源整体降级为模式C检索代理**。
- 镜像范围走白名单（§6.4），客户经营/财务/原始稿件类不入范围。

### 3.2 腾讯（模式B）

- 读写全能力；`search` 用 `GET /openapi/drive/v2/search`（关键字，非语义）。
- 两件事必须管住：①应用级 API 免费额度需监控（同步层限流错峰、跨源检索按实际调用源数计费）；②同名文件可重复 → 幂等靠 `content_hash` + 显式路径去重，**不信文件名**。
- 新文档默认私有 → 镜像准入 `acl.level` 一律先按 `internal`。
- `write_back` 留给人工评审回稿；**对外 MCP 不因写入能力开放写入面**（F4：对外恒只读四工具）。

### 3.3 ima（混合模式）

- 定位：个人/团队资料沉淀、文章素材、网页剪藏；**不是主库**。
- 权限粒度仅知识库级——敏感资料天然不适合放 ima（分级沉淀第一道过滤器）。
- 平台侧维护**索引镜像**参与融合排序；写入侧直调 `add_knowledge` 保持用户在 ima 端行为一致。
- 单文件上限：TXT/MD/Excel ≤10MB、图片 ≤30MB、PDF/Word/PPT ≤200MB；大文件先拆分或走 MinerU 解析通道。
- 接入用 `scope_subscribe` = 平台应用级凭证 + **白名单范围订阅**（逐个指定订阅哪几个 ima 库，首批只放"公开可复用"类库）。

### 3.4 ima 候选池 ≠ 自动同步（边界铁律）

ima 好内容是**候选池不是自动同步**：入主库唯一走 `POST /api/kb/sediment`（脱敏→评分→准入→审计）。绑定面 R1 恒 `private`——基础用户不会因平台接了 ima 多看到任何内容。

---

## 4. 镜像同步机制

### 4.1 流水线（两源共用骨架）

```
遍历 list(cursor)
  → 逐篇 fetch(ref) 导出 markdown
  → 归一化 CanonicalDoc（source/acl/content_hash/quality_score/origin_url）
  → 分块 + bge-m3 向量化 + 全文索引（入自建主库索引）
  → 参与联邦融合排序
```

每次检索结果带**来源徽标 + 更新时间 + origin_url 回链**。平台不自造正文真值。

### 4.2 三通道调度

| 通道 | 触发 | 频率/参数 | 用途 |
|---|---|---|---|
| Webhook 事件推送 | `watch(cb)` | 近实时 | 增量：新建/更新/撤权即时感知 |
| 定时增量拉取 | `list(cursor)` 从上次游标 | 每 30min（可配，错峰） | 补 Webhook 漏事件；带宽/配额友好 |
| 周级全量对账 | 全 `list` + hash 比对 | 每周低峰 | 幂等纠偏、检测漂移、撤出清理 |

### 4.3 增量同步算法（伪代码）

```
for ref in list(cursor):
  cur = fetch_content_hash(ref)
  if cur == db.content_hash(ref):
     update metadata only (source_updated_at)   # 不动 content、不改 acl
  elif cur is null (源侧已删/撤权):
     sync_state = 'withdrawn'                    # 单向阀，即时生效
  else:
     doc = normalize(fetch(ref))                  # content_hash 变=内容真变了
     doc.acl.level = max(db.acl.level, 'internal') # 入库不自动升档
     reindex(doc); doc.version += 1
  commit cursor
```

### 4.4 冲突消解（主源优先级）

多源指向同一知识时按主源优先级：**自建 > 钉钉 > 腾讯 > ima**。规则是"**高优先级源为准并登记**"，**不是后写者赢**，**禁止静默覆盖**。冲突写入 `sync_conflict` 审计表（doc_id、两源 ref、两 hash、胜出方、人工可复核），供回滚。

### 4.5 幂等与去重

- **写入幂等**：`uq(source, source_ref)` + `content_hash` 双键；重复事件不产生重复条目。
- **跨源去重**：SimHash（索引层）+ `dedup_of` 指针；同一条内容多源命中，融合层只呈现保留条目，其余标 `dedup_of`，不物理删除（可回溯）。

### 4.6 单源故障隔离

`E_TIMEOUT` / `E_QUOTA_EXCEEDED` 时标记"**该源未参与本次检索**"并在结果提示，不阻塞整体返回（对齐 v3.13 §07 降级熔断）。融合层对缺源结果为部分集合，来源徽标明示覆盖度。

### 4.7 防双轨漂移门禁

镜像与直连检索共用**同一检索核心实现**（共享 RAG 底座），A/B 对比 50 题差异率 < 5% 才允许上线（v3.7/v3.13 既有门禁）。禁止为某源另写一套排序逻辑。

---

## 5. 沉淀准入管线（sediment）

沉淀是"产出即沉淀"的执行体，也是**镜像内容升档的唯一出口**（§6.1）。管线节点：

```
定稿事件 → 结构化抽取 → 脱敏 → 质量评分 → 准入判定 → 去重合并 → 回写目标库 → 索引更新 + 审计
```

### 5.1 各节点判据

| 节点 | 判据 | 不通过处理 |
|---|---|---|
| 结构化抽取 | 抽出 title/summary/teaser/domain/引用清单 | 抽取失败入"待完善"队列 |
| 脱敏 | 三级内容规则（§5.2）；客户信息/档案必须脱敏或拒入 | 含不可脱敏敏感项→硬拒 |
| 质量评分 | 引用率：≥1 条可溯源依据；重复率：<30%；质量分 ≥85 | 未达标进"待完善"队列，不入库 |
| 准入判定 | 目标库 + 目标可见域档位（受控升档） | **缺权限校验=P0 缺陷**（PR §4.4），补齐前不得对外写 |
| 去重合并 | content_hash + SimHash；命中已有→版本 +1 而非新建 | 冲突登记，可回滚 |
| 回写 | 目标库默认自建主库；跨库同步需显式指定；`write_back` 只建 draft | — |

### 5.2 沉淀三级与对外可见（对齐 v3.13 §07）

| 分级 | 内容范围 | 处理 | 对外 MCP 可见 |
|---|---|---|---|
| ① 公开可复用 | 政策解读/模板/方法论/通用文章 | 入全库 | ✅ |
| ② 含客户信息 | 项目交付物/案例 | 脱敏后入模板库·成果库，原文留项目库仅内部可见 | ⚠️ 仅脱敏版 |
| ③ 敏感档案 | 客户档案/财务数据 | **不沉淀**，仅项目内可见 | ❌ 否 |

### 5.3 质量分构成（准入线 ≥85）

`quality_score = 引用可溯源(0-30) + 结构完整(0-25) + 去重后信息增量(0-25) + 人工采纳率加权(0-20)`；三项硬门禁（引用率≥1、重复率<30%、总分≥85）任一不过即拒入。**反哺写作**：沉淀条目下次撰写同类文档时自动进入 RAG 上下文，撰写耗时随沉淀量递减。

### 5.4 sediment 是唯一升档通道（关键约束）

镜像内容 `acl.level` 从 `internal` 升到 `premium/expert/public` **只能**经 §5 管线并写审计；**任何同步、对账、Webhook 事件都不得直接修改 acl.level 使其变宽**。这是"单向阀"的实现落点：内部看到的多、对外放出的少。

---

## 6. 四角色 × 镜像内容权限映射

### 6.1 核心规则：收敛，不映射

钉钉/腾讯的 owner/可管理/可查看等权限模型**不逐条翻译**为平台四角色（必然出洞）。规则：

1. **入库默认 internal**：所有镜像入口统一降档为 `internal`。
2. **升档只走 sediment**（§5.4）：分级出口受控升档。
3. **源侧权限变更不自动跟随**：周级对账只刷新元数据，不自动放开。
4. **撤出即时生效（单向阀）**：`E_PERMISSION_REVOKED` → `sync_state=withdrawn`，即时收。

### 6.2 角色可见矩阵

| 角色 | internal 镜像原文 | premium 域（sediment 后） | public 域 | 对外 MCP |
|---|---|---|---|---|
| 基础用户 BASIC | ❌ | ❌ | ✅ | —（无 MCP 权） |
| 高级用户 PREMIUM | ⚠️ 仅 origin_url 回链（能否打开由源平台权限决定，平台不担保） | ✅ | ✅ | 仅 public 分级 |
| 课题专家 EXPERT | ✅ 工作协同类可读（建议专家账号同为钉钉组织成员，现场协作在钉钉） | ✅ | ✅ | — |
| 管理员 ADMIN | ✅ 全部；查私有稿走 v3.11 SoD（双人=两账号、自批400、留痕） | ✅ | ✅ | key 预置可见域裁剪 |

### 6.3 三条身份线坚决不混

1. **登录**：平台登录走 v3.10 三通道；源平台回调**永不签发平台令牌**（R2）。
2. **镜像凭证**：一律企业应用级凭证 + 白名单范围（类 `scope_subscribe`），**不用员工个人令牌**（防离职/换岗带走链路，R4/R5）。
3. **回写身份**：`write_back` 用应用身份建草稿，人工在源侧评审定稿，平台不代理用户身份改文档。

### 6.4 钉钉侧最小改造三件事

1. **镜像范围白名单**：客户经营/财务/原始稿件类不入范围（源侧不暴露是第一道闸）。
2. **目录约定承载"建议"分级**：如 `/公开-政策解读`、`/高级-模板库`，同步层 `fetch` 时写入元数据（`acl.suggested_level`）；**准入判定仍在平台**，防止目录名"洗权限"。
3. **导出权限定向申请**。

### 6.5 叠加判据：取交集，禁并集

`§15 权限矩阵` 管自建主库条目；镜像原文是候选池、不在矩阵库列——**先吃 internal 全局规则，沉淀后才入矩阵管辖**。两层规则叠加时**取交集（更严格者生效）**，实现禁止做成并集。

```ts
// 判定：某 role 能否读某 CanonicalDoc 的某一层
function canRead(role: Role, doc: CanonicalDoc, level: VisLevel): boolean {
  // 交集 = min(平台角色可见域, doc.acl.level, suggested 建议档经准入后的终档)
  const docFloor = doc.sync_state === 'mirrored'
      ? 'internal'                      // 镜像原文未沉淀 → 恒 internal
      : doc.acl.level;                  // 已沉淀 → 矩阵管辖终档
  const effective = stricterOf(roleMaxView(role), docFloor); // 取更严格
  return tierReached(level, effective); // kbs/v1 层级梯度服务端裁剪
}
```

`stricterOf` 严格序：`public < premium < expert < internal`（internal 最严，仅 EXPERT/ADMIN 按矩阵可读，BASIC/PREMIUM 读不到镜像原文）。

---

## 7. 各源调用面 · 配额 · 限流

### 7.1 调用面清单

| 源 | 主要调用 | 频率约束 | 监控点 |
|---|---|---|---|
| 钉钉 | 节点树遍历、markdown 导出（需开通）、成员/权限管理 | 遍历按 cursor 分批、导出逐篇 | 导出能力是否已批；无权限条目占比 |
| 腾讯 | drive 遍历/导出、`GET /openapi/drive/v2/search`、写回 | **应用级免费额度按应用计数** | 额度余量告警阈值（如 80%）；同名重复计数 |
| ima | `search_knowledge`/`search_knowledge_base`、`add_knowledge`、`import_urls` | 单文件上限约束 | 超限拒入计数；混合模式索引/写入一致性 |
| 自建 | 原生 SQL / 向量检索 | — | Recall@5、来源召回率 |

### 7.2 限流与错峰设计

- **同步层**：定时增量错峰执行（避开业务高峰）；`E_QUOTA_EXCEEDED` 指数退避重试，重试上限后转下轮对账。
- **检索层**：跨源检索按**实际调用源数**计费/限流，防止一次查询打满多源配额；全量审计调用链（问题 → 命中源 → 回写目标）。
- **对外 MCP**：每客户端独立 API key + 独立配额（60 次/分基线，v3.11 落表 `mcp_quota_usage`）。

### 7.3 域名与传输

生产固定域名 + HTTPS 反代（APISIX/Nginx），**临时隧道仅限联调、生产禁用**。客户档案与案例永不出内网（PR 红线）。

---

## 8. fail-closed · 受控动作 · killswitch

### 8.1 fail-closed 默认值（对齐 v3.13 广场）

任何外部源新增绑定，默认：

| 项 | fail-closed 默认 |
|---|---|
| `enabled` | `false`（不配即不通） |
| `endpoint_ref` | 只存 env 变量名，不存明文 endpoint |
| `acl_ceiling` | `internal`（放开须走审批） |
| 三表 content 列 | mcp_catalog/mcp_binding/mcp_cred **均无 content 列**（R6 结构保证：绑定面拿不到正文真值） |

### 8.2 受控动作（holds）流程

源启用、`acl_ceiling` 放开、key 轮换/吊销等是**管理面受控动作**，走 v3.11 SoD 状态机 `admin_action_hold`：

```
发起(账号A) → hold 登记 → 确认(账号B≠A) → 生效 + 双留痕
```

- 双人 = 两账号；≥2admin 时自批返回 **400**；单 admin 场景走冷却 + 显式自曝降级。
- SoD 审批记录可回写钉钉审批/OA 作治理证据副本，但 **admin_action_hold 为真值**（PR 附录A 场景6）。

### 8.3 绑定六红线 R1–R6（本文档执行面）

- R1：ima/外部源绑定恒 `private`（§3.4）。
- R2：源平台回调永不签发平台令牌（§6.3）。
- R4/R5：镜像只用企业应用级凭证 + 白名单，禁个人令牌（§6.3）。
- R6：三表无 content 列，绑定面不得缓存/落正文（§8.1）。

### 8.4 killswitch

对外 MCP 出口与各源适配器提供 killswitch：命中安全事件（凭据泄露、异常爬取、脱敏失效）时**即时全局关停**，关停走 SoD 但生效不等确认（安全优先），事后补审计。撤出镜像范围（§6.1 单向阀）是数据面细粒度关停。

---

## 9. 验收门禁与可观测

### 9.1 阶段0 建表/契约工单（前置，未完成不得对外写）

1. `knowledge` 表补 `status / teaser / version` 三列（PR §4.4 缺陷）。
2. `sediment` 补权限校验（P0）。
3. CanonicalDoc 建表（§1.4）+ 向量/全文关联表。
4. 钉钉 markdown 导出开通申请提交并获批（未批→该源模式C）。

### 9.2 功能门禁

| 门禁 | 判据 |
|---|---|
| 层级梯度 | 未达层响应中键不存在；客户端传层请求 → 400 |
| 幂等 | 重复事件不产生重复条目；content_hash 不变只刷元数据 |
| 单向阀 | 同步/对账/Webhook 均不改宽 acl；撤权即时 withdrawn |
| 冲突 | 主源优先级正确，禁止静默覆盖，冲突入审计表 |
| 交集判定 | 叠加取交集；单测覆盖 role×level×sync_state 全组合 |
| fail-closed | 新绑定默认 enabled=false / acl_ceiling=internal |
| SoD | 自批 400、双账号留痕、killswitch 即时生效 |

### 9.3 检索质量指标（对齐 PR §2.3）

| 指标 | 目标 |
|---|---|
| 知识条目总量 | ≥ 3,000 条 |
| 50 问评测准确率 | ≥ 85%（含来源引用） |
| Recall@5 | 97.5% |
| 来源召回率 | 98% |
| 无依据回答率 | < 5% |
| 双轨一致性 | A/B 50 题差异率 < 5% |

### 9.4 可观测

- 每源同步健康度（`list/fetch` 成功率、`E_*` 错误码分布、degraded 次数）。
- 配额水位（腾讯免费额度、MCP key 用量）。
- 沉淀管线各节点通过/拒入计数与"待完善"队列积压。
- 调用链审计：问题 → 命中源 → 归一化 → 权限判定 → 返回层级 → 是否回写。

---

## 10. 决策与未决点（承接 PR §10）

| # | 事项 | 本文档立场 | 需谁拍板 |
|---|---|---|---|
| 1 | MaxKB vs Dify 底座二选一 | 本文档接口只依赖适配器抽象，底座可替换，不阻塞 §1–§9 施工 | 管理层（阶段0第1周） |
| 2 | 推理算力路线 | 自建 GPU（数据不出内网，推荐）vs 商用 API | 管理层 |
| 3 | 钉钉导出开通 | 已按"未批→模式C"设计降级路径，不阻塞 | 对外协调 |
| 4 | 权限矩阵终版 | §6.2 矩阵为设计口径，以业务方最终确认为准 | 业务方 |
| 5 | 外部源启用批次 | 五源登记仅 self 启用；每源启用即受控动作（acl_ceiling 默认 internal） | 管理层确认顺序 |

---

*本设计文档锚定《架构设计方案 v3.13》（2026-09-30，页面 v29，远端逐字节校验 blob c72cf648）与 PR v1.3 附录A 口径；未引入任何未拍板的底座假设。字段级定义、接口签名与算法为可施工细则，若与 v3.13 冲突以 v3.13 为准并回本文档修订。*
