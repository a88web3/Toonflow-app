# AI 连续短剧生产路线图（候选，未进入施工）

> 状态：仅研究/规划。**本文件不修改 `ToDo.md` 的 28 项任务、不改变其状态，也不授权 OpenCode 施工。** 候选任务须经用户批准、与既有任务去重并映射后，才能写入唯一施工入口 `ToDo.md`。

## 1. 前提和决策门

现有 28 项仍是前置基线，尤其是 P0-001～P0-004、P1-001～P1-004、P2 provider/许可/基准项、P3 流水线项和 P4 durable DAG。当前没有一项可因本报告自动提前施工。

| 决策门 | 必须有的证据 | 不通过时 |
| --- | --- | --- |
| 安全与数据 | 既有 P0 完成、恢复演练和外部入口策略 | 不进入多项目/云端/批量。 |
| 领域 ADR | Series/Shot/Asset/Task/Review 真相源、迁移和保留策略获批 | 不新建生产表/API。 |
| 媒体准入 | 模型/权重/节点/音色许可证台账，隔离 benchmark | 默认动态漫画+人工工作流；不宣称本地视频可生产。 |
| 成本闸门 | 试产实际镜头、重试、人工时间、云账单数据 | 不估算 10/50/100 集商业报价。 |
| 发布门 | 终审、权利、AI 披露、平台规则复核 | 只导出内部 render，禁止自动发布。 |

媒体状态必须按 [专项选型结论](./AI_SERIES_PRODUCTION_RESEARCH.md#12-图片与视频模型专项选型结论研究不是设备验收) 标记为 `RESEARCHED`、`COMPATIBLE_ON_PAPER`、`VERIFIED_ON_DEVICE` 或 `PRODUCTION_READY`。当前所有图片/视频候选最高只到前两级；不得以报告中的推荐顺序绕过设备和生产验收。

若选择复用 Mac Studio 统一内存，资源切换的候选工作和 Go/No-Go 门另见 [模型资源调度方案](./MODEL_RESOURCE_SCHEDULING.md)。它是对 P0-004、P1-003、P2-005～P2-007 和 P4-001 的补充设计，不是新施工授权。

## 2. 分阶段 MVP

| 阶段 | 范围 | 验收标准 | 质量指标（先记录，后定阈值） | 人工门与主要风险 |
| --- | --- | --- | --- | --- |
| **MVP-0：契约固化（对账新增，前置于 MVP-1）** | 从 `v118workflow`、`workflow` PART A（`A0–A12`）与 `stateAndDelivery.md` **反向抽取** SeriesBible/EpisodeCard/ShotSpec/AssetVersion/StageRecord/RenderManifest 的 JSON Schema；实现校验器与**只读索引器**（集号→状态/产物路径/版本/依赖）；产物仍写工作区文件，**不建数据库** | 一个手工项目能通过 schema 校验并被索引；30–100 集的状态与产物可一次查询出全貌；不写入任何生产数据库 | schema 覆盖率、校验失败率、索引与真实文件的一致性 | 人工确认字段口径与既有 skill 命名一致；风险是与现场产物脱节 |
| MVP-1：单集 60 秒动态漫剧 | 1 项目、1 集、约 8–20 个 ShotSpec；本地文本/图片候选、FFmpeg 动画、TTS/字幕；云视频仅经批准用于少数镜头 | 从批准集卡到 9:16 render，可追溯每镜头输入/版本/审核；服务重启后可恢复；无自动发布 | 时长偏差、技术 QC 通过率、人工返工数、镜头重试率、实际耗时/成本 | 集卡、关键帧、粗剪、终审；角色漂移、视频失败、版权。 |
| MVP-2：连续 3 集 | 共用 SeriesBible、角色/地点/声音版本；滚动释放三集 | 跨集事实/伏笔检查发现并处理冲突；同一资产可复用且版本不被覆盖 | 连续性缺陷数、复用率、每集人工工时 | 季/集计划、每集分镜、首集风格锁定；上下文膨胀。 |
| MVP-3：连续 10 集 | durable DAG、租约、预算、审查看板、失败恢复 | 注入 worker/provider 中断后恢复，不重复扣费/覆盖批准资产；所有 render 有 manifest | 恢复成功率、重复任务率、队列等待、成本偏差 | 每集/每批预算批准；并发与存储增长。 |
| MVP-4：30–100 集 | 多季计划、滚动排产、成本账本、运营/发布流程 | 仅在容量、备份、质量和成本指标连续稳定后扩大；可导出审计与权利清单 | 吞吐、SLO、容量、人工审核负荷、平台合规事件 | 季度路线/预算/权利审查；不可把试产结果直接外推。 |

时间、成本和质量阈值目前均为 `UNVERIFIED`，需以 MVP-1 的真实试产数据制定。镜头数不是 KPI；通过率和可复现性更重要。

## 3. 候选施工任务（待批准，禁止直接合并 ToDo）

| 候选 ID | 优先级 | 目标和可独立验收标准 | 依赖 | OpenCode 适配性 / 人工决策 |
| --- | --- | --- | --- | --- |
| CAND-SERIES-001 | P1 | 形成并批准连续剧领域 ADR：实体、真相源、迁移、保留、权限、状态机。验收：架构评审通过且不改生产数据。 | P0-003、P1-002 | 适合 OpenCode 起草；**人工**批准领域边界和数据库选择。 |
| CAND-SERIES-002 | P1 | 定义/测试 StoryBible、EpisodeCard、ShotSpec、AssetVersion、Review、Render manifest JSON Schema。验收：正负样例及 schema 校验测试。 | CAND-SERIES-001 | 适合 OpenCode；人工确认创作字段。 |
| CAND-SERIES-003 | P1 | 定义连续性规则目录及冲突/豁免审计格式。验收：死亡、服装、场景、伏笔、事件顺序的确定性测试均可复现。 | CAND-SERIES-002 | 适合 OpenCode；人工定义叙事豁免政策。 |
| CAND-SERIES-004 | P2 | 文本结构化输出评测协议：固定样本、schema 合规、修复、上下文预算和人工评分。验收：不调用未批准生产任务的可重复基准报告。 | P2-001、CAND-SERIES-002 | 适合 OpenCode；**人工**批准 Qwen 调用窗口。 |
| CAND-SERIES-005 | P2 | 资产版本/参考图/声音权利元数据合同。验收：角色、地点、服装、style、voice 的样例可表达批准、停用和来源。 | P1-004、CAND-SERIES-002 | 适合 OpenCode；**人工/法务**确认权利字段。 |
| CAND-SERIES-006 | P2 | 动态漫画镜头与真视频选择规则、QC 样例和成本预算接口。验收：每种镜头有明确路由和失败回退。 | P1-003、P2-003、P2-006、P2-007 | 适合 OpenCode 文档/测试；**人工**选择供应商和预算。 |
| CAND-SERIES-007 | P3 | 在获批领域边界实现最小 Series→Episode→Shot→Review 数据/API 垂直切片。验收：一个手工创建的集能流转至批准状态并有审计。 | P3-001、CAND-SERIES-001、002 | 适合 OpenCode；不得替换既有画布。 |
| CAND-SERIES-008 | P3 | 关键帧审批后的镜头资产任务桥接与 manifest。验收：任务输入固定，失败生成新 attempt，不覆盖批准资产。 | P3-003、P4-001、CAND-SERIES-005 | 适合 OpenCode。 |
| CAND-SERIES-009 | P3 | 确定性动态漫画 render + 台词时间轴/字幕/技术 QC。验收：从批准 manifest 生成相同规格 render，并报告时长/音频/字幕检查。 | P3-004、CAND-SERIES-006、008 | 适合 OpenCode；音色与音乐使用需人工批准。 |
| CAND-SERIES-010 | P4 | Series 级滚动排产、依赖图、lease 恢复、资源组（Qwen/视频互斥）。验收：故障注入后不重复执行已完成任务。 | P4-001、P4-003、CAND-SERIES-007、008 | 适合 OpenCode；人工设并发和预算阈值。 |
| CAND-SERIES-011 | P4 | 连续性审查工作台：冲突证据、豁免、影响范围和重新排队。验收：已批准事实被冲突时阻断下游并可人工裁决。 | CAND-SERIES-003、007、010 | 适合 OpenCode；人工负责实际裁决。 |
| CAND-SERIES-012 | P5 | 成本/权利/发布就绪账本。验收：每个 render 可回溯来源资产、模型/许可证、任务尝试、成本和人工批准。 | P5-002、P5-003、CAND-SERIES-009 | 适合 OpenCode；**人工/法务**确认发布与授权。 |

候选数量为 **12**。它们和既有 28 项可能需要合并（例如 CAND-SERIES-001 与 P1-002、CAND-SERIES-007 与 P3-001），不能机械相加成 40 项。获批后应逐项映射、去重、给出文件范围/测试/commit 证据/恢复办法，再更新 `ToDo.md`。

## 4. 推荐实施顺序

1. 先完成既有 P0/P1 安全、备份、日志、领域迁移 ADR 和媒体隔离基线。
2. 批准 CAND-SERIES-001～003；先冻结数据/连续性合同，再写 UI 或生成工作流。
3. 在模型/许可证准入后做 CAND-SERIES-004～006，并用手工小样验证 MVP-1 假设。
4. 仅在 P3/P4 基础完成后进行 CAND-SERIES-007～011，实现单集→三集→十集的可恢复放大。
5. CAND-SERIES-012 与既有 P5/P6 通过后，才考虑 30–100 集和发布自动化。

## 5. 当前 28 项如何调整

**本轮不调整 `ToDo.md` 文件本身**（含状态、顺序与文字）。当前任务表的优先级逻辑依然正确：P0/P1 是任何模型/批量能力的门槛；P2 是受控集成；P3 是可审核单集；P4 才是持久批量；P5/P6 才能宣称运营和生产验收。

> **第三轮对账修订**：本轮源码审查发现仓库已内置且本机已安装三个生产级 skill（`v118workflow`、`workflow` 含 PART A `A0–A12`、`canvas`），以及 Agent 工作区 `write`/`edit` 工具、文档工作区、FFmpeg `concat`/`complexFilter`、`context.media` 无头扩展点、`director3dNode` 与多租户计费账本。因此下列候选任务中，**凡与既有 skill 契约或既有能力重叠者，应由「新建」改为「形式化/扩展」**。逐项判定（`KEEP` / `MERGE` / `REVISE` / `REMOVE_CANDIDATE` / `ADD_CANDIDATE`）见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md) §13。两处必须纠正的前提：`P3-004` 的「没有 concat」已被源码证伪（`packages/ffmpeg/src/browserTypes.ts:21-24`）；`P3-002` 的基线应由 `storyboardTeam`（本机 `data/agents/` 目录不存在、未安装且自述只读）改为 `v118workflow` 的分镜 reference + PART A `A4–A9`。

需要在用户批准后做的仅是：

- 将 CAND-SERIES-001～003 与 P1-002 的领域 ADR 范围合并/拆分，防止重复。
- 将 CAND-SERIES-004～006 分别映射到 P2-001、P2-004、P2-006/P2-007，而不是新建并行模型项目。
- 将 CAND-SERIES-007～009 映射到 P3-001～P3-005；将 CAND-SERIES-010/011 映射到 P4-001～P4-003。
- 将 CAND-SERIES-012 映射到 P5-002/P5-003 和 P6-002。

这是候选 crosswalk，不是任务状态或施工指令。

## 6. Go/No-Go 指标与商业边界

| 扩大阶段 | Go 的最低证据 | No-Go 信号 |
| --- | --- | --- |
| 单集→3 集 | 角色/地点版本已锁定，连续性缺陷有证据和裁决，试产成本可解释 | 关键帧/角色漂移反复、无资产来源或审核记录。 |
| 3 集→10 集 | 恢复演练无重复扣费/无数据丢失，队列和存储在预算内 | 任务长期卡死、人工审核成为瓶颈、许可证不清。 |
| 10 集→30–100 集 | 批量 SLO、备份恢复、权利/成本账本、发布门均稳定 | 把未验证本地视频性能或单例质量样片外推。 |

渠道变现、实际单集成本、实际本机视频吞吐和模型输出权利均需按具体供应商/版本/地区核验，当前为 `UNVERIFIED`。平台对合成内容披露与重复内容的规则见 [TikTok](https://support.tiktok.com/en/using-tiktok/creating-videos/ai-generated-content) 与 [YouTube](https://support.google.com/youtube/answer/14328491)；发布前重新检查。

## 7. 本轮交付状态

- 研究：[AI 连续短剧生产研究](./AI_SERIES_PRODUCTION_RESEARCH.md)
- 架构：[推荐架构](./AI_SERIES_PRODUCTION_ARCHITECTURE.md)
- 前置证据：[生产审查](./TOONFLOW_PRODUCTION_AUDIT.md)、[本地媒体可行性](./LOCAL_AI_MEDIA_FEASIBILITY.md)

`RESEARCH_COMPLETE`（文档研究完成）

`ARCHITECTURE_PROPOSED`

`IMPLEMENTATION_NOT_STARTED`

## 8. 对账修订记录与新增候选（2026-11-15，CodexQ 复核）

完整对账见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md)。本路线图的决策门、MVP 分级与 Go/No-Go 逻辑**继续有效**。

### 8.1 修订要点

| 修订项 | 内容 |
| --- | --- |
| 新增 MVP-0 | 「契约固化」置于 MVP-1 之前：从既有 skill 反向抽取 schema + 校验器 + 只读索引器，产物仍写工作区文件，不建数据库 |
| 决策门补强 | 「媒体准入」门新增前置：**必须先有 Agent 无头媒体工具**（消费 `context.media`），否则批量生产被浏览器画布绑定，任何 MVP 的「服务重启后可恢复」都无法成立 |
| 候选任务去重 | `CAND-SERIES-001/002/003/005` 标为 `REMOVE_CANDIDATE`（分别并入 `P1-002`、ADD-001、`P3-002`、`P1-004`+`P2-004`）；其余 `CAND-SERIES-004/006～012` 标为 `MERGE`，映射到既有任务，不新建并行项 |
| 资源切换门 | 新增强制规则：**当前无法判断其他项目是否在用 Qwen → 禁止自动卸载 → 人工审批 + 维护窗口**；把「Studio 侧在途计数或统一 ingress」（ADD-003）列为任何切换演练的前置 |
| 数据口径 | 所有时间/成本/质量数字必须标注 `MEASURED` / `PUBLIC` / `ESTIMATE`；本机当前**无任何 `MEASURED` 数据**（`data/workspaces/myTvProject` 为 0 文件、`data/assets/` 为空） |

### 8.2 新增候选任务（`ADD_CANDIDATE`，**未写入 `ToDo.md`**）

| 建议 ID | 优先级 | 目标与独立验收 | 依赖 |
| --- | --- | --- | --- |
| ADD-001 契约固化（MVP-0） | P1 | 从 `v118workflow` + PART A + `stateAndDelivery.md` 反向抽取 JSON Schema，实现校验器与只读索引器。验收：手工项目通过校验并被索引，不写数据库 | P0-003、P1-002 |
| ADD-002 Agent 无头媒体工具 | P2 | 新增插件工具消费 `context.media`，复用 `generateMedia` 与 `tasks.ts`；不新建媒体 API。验收：无浏览器完成提交→查询→取消→恢复，产物含输入哈希与任务 ID | P0-002、P2-005 |
| ADD-003 Studio 在途可观测性与排空协议 | P0/P1 | 建立「全客户端在途文本请求可计数」能力（Studio 侧计数或统一 ingress）。验收：不重启服务即可读出在途数与空闲判定；缺失时明确 `BLOCKED` | P1-001、P0-004 |
| ADD-004 版本快照与确认绑定 | P1/P3 | 实现 `history/<集号>/<时间戳>-<hash>.md` 快照与「确认绑定快照路径」台账字段（复用 `files/copy.ts` 语义）。验收：确认后原文被改，原确认仍指向旧快照并可展示差异 | ADD-001 |
| ADD-005 个人模式成本流水 | P2/P5 | 在 `modelGateway.enabled()===false` 时记录估算/实付、provider、模型、镜头标识与用量。验收：单机制作一集后可导出成本明细，估算与实付分列 | P5-002、ADD-001 |
| ADD-006 资产 canonical ID 与输入哈希复用索引 | P3 | 以 `providerId+modelId+prompt+参考文件哈希+关键参数` 为键建立产物索引，命中即复用不重复计费；为角色/场景/服装/声音分配稳定 ID 并登记使用集数。验收：重复请求命中缓存并跳过生成 | P3-001、ADD-001 |
| ADD-007 字幕产物与音画同步校验 | P3 | 离线 ASR/强制对齐候选评估 + SRT/ASS/VTT 产物 + 时码比对；明确禁用浏览器 ASR 作为生产路径。验收：带台词样片可产出字幕并通过时码偏差检查 | P2-004、P3-004 |
| ADD-008 一键保存与会话导出（低优先，可选） | P5 | 对话区「保存为工作区文件」动作（内部调用既有 `files.write`）+ 会话 JSONL→Markdown 导出。验收：一条回复可存为指定路径并在文档面板打开；一个会话可导出为 Markdown | 无 |

### 8.3 建议后的阶段骨架（供批准参考）

```
P0  安全与数据基线   ：P0-001 / P0-002(范围补 sidecar 准入) / P0-003(备份范围补 skills、.agent、画布、台账)
                      + P0-004(写入禁止自动卸载) + ADD-003
P1  可观测与契约     ：P1-001(补在途计数) + P1-002(输入改为 skill 契约) + ADD-001 + ADD-004 + P1-003 / P1-004
P2  受控集成         ：P2-001 / P2-003(优先 Seedance) / P2-006 / P2-007
                      + P2-002(对齐资产画布字段) / P2-004(补 schema 事实) / P2-005(缩小范围) + ADD-002 + ADD-005
P3  单集可审核流水线 ：P3-001(并入 P1-002+ADD-001) + P3-002/003/004/005(修订) + ADD-006 + ADD-007
P4  批量与恢复       ：P4-001 + P4-002(基于索引器) / P4-003(对接资源锁与状态机)
P5  质量、成本与运营 ：P5-001(复用 frameAnalysisNode/ffprobe) / P5-002(扩展既有 ledger) / P5-003(与 P1-004 合并台账) + ADD-008
P6  压测与上线       ：P6-001 + P6-002(区分站内/外部平台发布)
```

### 8.4 后续技术验证优先顺序（摘自对账报告 §14）

`V1` 只读盘点本机实际配置 → `V2` 云端单图端到端 → `V3` 云端 5 秒 I2V → `V4` 本机 FFmpeg 能力（concat/libass/编码器）→ `V5` 无头路径直调 `/api/ai/media/generate` → `V6` 音频 provider 与 TTS 候选盘点 → `V7` Qwen 结构化输出验收 → `V8` Studio 在途可观测性（未通过则永久禁止自动切换）→ `V9` 本地图片隔离基准 → `V10` 本地视频隔离基准。

`ROADMAP_RECONCILED`
`TODO_NOT_MODIFIED`
`IMPLEMENTATION_NOT_STARTED`
