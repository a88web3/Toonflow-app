# AI 连续短剧生产平台：推荐架构

> 状态：`ARCHITECTURE_PROPOSED`。此文是设计建议，不代表 Toonflow 已实现这些模块；不改变现有部署或基础设施。

## 1. 产品定位与架构决策

Toonflow 应演进为“**本地 AI 优先、云端为受控补充的连续短剧生产工作室**”：它管理创作事实、生产资产、任务、审核与可发布成片；模型/provider 是可替换的生产资源，而不是产品数据库。

**推荐决策（第三轮对账修订）：渐进式加固既有 skill 契约 + 薄契约层 + 外置可恢复编排/媒体 sidecar。** 保留 Toonflow 画布、provider、统一媒体接口、媒体任务检查点、FFmpeg 与文档工作区基础，**并以仓库已内置且本机已安装的三个 skill（`v118workflow`、`workflow` 含 PART A `A0–A12`、`canvas`）所定义的产物契约为领域模型的抽取来源**。不要在 MVP 阶段拆微服务，不要把生产状态只留在 Canvas JSON 或 provider 任务文件里，也**不要另起一套与既有 skill 命名冲突的领域实体**。

**两条执行路径必须分开设计（关键约束）**：

| 路径 | 机制 | 适用 | 约束 |
| --- | --- | --- | --- |
| 画布路径（现有） | Agent → `canvas` 工具 → `canvasCall` 事件 → 前端执行节点函数（`node:generateImage` 等） | 人工在环的单镜头创作、分镜搭建、`director3dNode` 运镜与导出 | **必须画布在浏览器中打开**；120s 超时；回传仅在当前 server 进程存活期有效（`apps/server/src/agent/bridge/canvas.ts:10,45`）→ **不可无人值守** |
| 无头路径（缺一个薄工具） | `context.media` 已注入 `listModels/generateImage/generateVideo/generateAudio`，底层直连服务端 `generateMedia`（`apps/server/src/utils/media/generation.ts:243`） | 批量生产、编排服务调用 | **当前无任何工具消费 `context.media`**；需新增一个插件工具即可打通，**不需要新建媒体 API** |

```
创作/审查 UI ──────┐
Canvas / Storyboard ├── Toonflow API / Auth boundary
运营看板 ──────────┘             │
                                  ▼
                     Series Production Domain API
      ┌──────── story bible / episode / shot / review / render / publication ────────┐
      │               Production DB + migration + audit trail                         │
      └──────────────────────────────┬────────────────────────────────────────────────┘
                                     ▼
                       Durable orchestration / lease / retry
                         │            │              │
                         ▼            ▼              ▼
                  Qwen text gateway  Media adapters  Render worker
                  (Studio, protected) (local/cloud)  (FFmpeg)
                         │            │              │
                         └────── Asset catalog + object/file storage ──────┘
                                      │
                              immutable manifest/version/hash
```

## 2. 现有能力与边界

| Toonflow 现有模块 | 证据 | 在目标架构中的位置 | 缺口 |
| --- | --- | --- | --- |
| Provider 定义 | `packages/providers/types.d.ts` 包含图/视频/音频与 `resumeTask` | 保留为 AI gateway/media adapter 合同 | 缺能力声明、预算、任务幂等与生产审计。 |
| 媒体任务恢复 | `apps/server/src/utils/media/tasks.ts`、`generation.ts` | 可作为单个 provider 任务适配基础 | JSON 文件不是持久生产 DAG/租约。 |
| 画布/素材/节点 | 既有前端与 workspace | 创作入口、人工编辑和可视化 | 不能承担 Series/Shot/Review 真相源。 |
| storyboard team | `packages/teams/storyboardTeam` | 作为提示词/文本样例 | 缺锁定 schema、连续性、审批和持久化。 |
| FFmpeg wrapper | `packages/ffmpeg` | 可作为受控 render runner 基础 | 缺时间轴、manifest、可重复 render/QC。 |
| **生产 playbook（三个 skill）** | `packages/skills/v118workflow/`（三画布契约、阶段关口、状态枚举）、`packages/skills/workflow/SKILL.md`（PART A `A0–A12`：A0.1 剧情保真锁、A2 源剧情账本、A4 出镜状态、A8 跨段连续性、A9 分段模板、A11 结构校验）、`references/stateAndDelivery.md`（台账四表 + 快照绑定）；本机 `data/skills/` **已安装三者** | **领域模型与连续性规则的抽取来源**；第一阶段产物标准 | 由模型自觉执行，无 schema 校验、无服务端状态机、无幂等/租约。 |
| **Agent 工作区文件工具** | `packages/tools/workspace/src/index.ts:13-35`（read/ls/grep/glob/**write**/**edit**/downloadImage）、`apps/server/src/agent/tools/index.ts:73-124`；本机 `data/tools/workspace.tool.js` 已安装 | 剧本/分镜/台账的正式落盘通道 | 无版本、无审批语义；`readOnly` 配置未核实。 |
| **文档工作区** | `apps/web/src/pages/workspace/panels/document/`（`fileTree.vue`、`quickOpen.vue`、`workspaceSearch.vue`、标签页/分屏、`documentSession.ts:72` revision 冲突检测）、`packages/ext/{md,txt,image,media,canvasNode}` | 剧本/台账的人工编辑与审阅界面 | 无按集版本视图、无审批绑定。 |
| **无头媒体扩展点** | `apps/server/src/agent/tools/index.ts:146-151` 注入 `context.media`；类型 `packages/toolScaffold/src/runtime.ts:140-145` | 批量生产的正确入口 | **无工具消费它**；需新增薄插件工具。 |
| **3D 导演台** | `packages/nodes/director3dNode/`（关键帧导出 PNG、动画导出 MP4，`src/renderMedia.ts:5,22`） | 动态漫画运镜预演与关键帧产出 | `renderVideo` 用浏览器 `MediaRecorder`，**浏览器绑定**。 |
| **FFmpeg 工具（concat/complexFilter）** | `packages/ffmpeg/src/browserTypes.ts:3-24`、`packages/tools/ffmpeg/src/index.ts:59,68` | 合成 worker 的执行底座，**可无头调用** | 缺 manifest、字幕产物、规格门禁；本机滤镜/编码器能力未实测。 |
| **计费账本与用量** | `packages/multiTenant/db/{sqlite,mysql}.ts`（`multiTenantBillingLedger`、`multiTenantWallet`）、`apps/server/src/utils/media/tasks.ts:44-45,271-305`、`apps/server/src/utils/ai/billingUsage.ts`、`utils/media/{billing,priceReference}.ts` | 成本账本的实现范式（预占-结算-释放、requestHash 幂等） | 仅多租户模式生效；无剧集/镜头维度；无预算闸门。 |
| **站内发布** | `apps/web/src/pages/home/publishWorkDialog.vue`、`apps/server/src/routes/channel/community/*` | 内部审看/交付载体 | 非外部平台发布；无 AI 披露与权利留痕。 |

结论：继续使用 Toonflow 作为交互、provider 与媒体执行底座是合理的；把它直接当作多季生产系统则不合理。

## 3. 领域边界与数据模型（`PROPOSED`）

### 3.1 真相源

| 聚合 | 责任 | 不可省略的字段 |
| --- | --- | --- |
| `Series` / `Season` / `Episode` | 目标、状态、剧情计划、发布策略 | owner、状态、版本、审核门、预算。 |
| `StoryBible` / `StoryFact` / `Event` / `Foreshadow` | 剧情连续性 | 来源、有效版本、冲突/豁免、证据 episode。 |
| `Character` / `Location` / `AssetVersion` | 可复用视觉、声音、地点资产 | 权利、引用输入哈希、style/workflow/model、批准状态。 |
| `Scene` / `Shot` | 生产最小单元 | 输入资产版本、时长、台词、依赖、输出规范。 |
| `GenerationTask` / `TaskAttempt` | 可恢复执行 | 幂等键、输入快照、租约、provider 引用、重试/成本。 |
| `Review` / `Render` / `Publication` | 审核、成片、渠道责任 | 决定、评语、签名者、manifest、披露/权利检查。 |

生产数据库需支持迁移、事务、索引和备份恢复。SQLite 能用于个人试产，但几十集、多 worker、租约和并发审查需要经 `P1-002` ADR 决定是否迁移至受管关系数据库；本报告不擅自指定或迁移数据库。

### 3.2 状态机

```
Shot: DRAFT → PLANNED → ASSET_LOCKED → QUEUED → RUNNING
      → QC_PENDING → REVIEW_PENDING → APPROVED → RENDERED
      ↘ BLOCKED / FAILED_RETRYABLE / FAILED_FINAL / SUPERSEDED

GenerationTask: PENDING → LEASED → SUBMITTED → POLLING → SUCCEEDED
                ↘ RETRY_WAIT / CANCELLED / FAILED_FINAL

Publication: DRAFT → RIGHTS_CHECKED → FINAL_REVIEWED → READY_TO_PUBLISH
             → PUBLISHED / WITHDRAWN
```

> **对账修订（状态枚举对齐）**：`v118workflow` 已在使用中文阶段状态 `草稿 / 待确认 / 已确认 / 生成中 / 失败 / 待复核 / 已跳过`，并写入画布「阶段记录」文本节点。为避免双轨，服务端状态机应**以此枚举为对外语义**，上面的英文状态作为内部实现映射：`DRAFT=草稿`、`PLANNED/ASSET_LOCKED=待确认`、`QUEUED/RUNNING=生成中`、`QC_PENDING/REVIEW_PENDING=待复核`、`APPROVED/RENDERED=已确认`、`FAILED_*=失败`、`SUPERSEDED/BLOCKED=已跳过或阻塞`。`GenerationTask` 的 `SUBMITTED/POLLING/SUCCEEDED/CANCELLED` 可直接对齐 `apps/server/src/utils/media/tasks.ts` 现有的 `state ∈ {running, interrupted, succeeded, failed, cancelled}`，不新造枚举。

状态转换由服务端命令处理并写审计事件；UI 不直接改状态。每次任务输入固定为 snapshot，重跑产生新 attempt 或新 AssetVersion，绝不覆盖已批准产物。

## 4. 模块职责和 API 轮廓

| 模块 | 责任 | 不应承担 |
| --- | --- | --- |
| Series Domain | 计划、事实、资产引用、审核规则 | 直接调用模型或执行 FFmpeg。 |
| Orchestrator | DAG、优先级、租约、并发/预算、恢复 | 存储剧本真相或做 UI 业务判断。 |
| AI Gateway | Qwen/provider profile、结构化输出、超时、审计 | 保存跨集记忆。 |
| Media Adapter | ComfyUI/本地 sidecar/云 API 的能力映射、轮询、取消 | 决定剧情或审核通过。 |
| Asset Catalog | 文件位置、hash、版本、权利、保留策略 | 隐藏任务状态。 |
| Render/QC | 时间轴→manifest→FFmpeg、技术检查 | 自动发布。 |
| Review Workbench | 对比、标注、批准/驳回、人工接管 | 绕过状态机。 |

API 采用资源命令而非“一个超级生成接口”：`POST /episodes/:id/plan`、`POST /shots/:id/queue`、`POST /reviews/:id/decision`、`POST /renders/:id/queue`、`POST /publications/:id/approve`；所有命令要求 idempotency key、actor、输入版本和预期状态。具体路径将在获批的 ADR 中确定。

## 5. 连续性与一致性服务

### 剧情

Story Memory Service 在每次集卡/剧本/分镜前构建有限 `ContextPack`：系列规则、角色状态、当前集约束、相关事实/事件/伏笔和人工确认摘要。它不传递无限对话。生成后运行确定性约束、检索式 AI 审稿和人工豁免流程；发现冲突时阻止下游镜头排队。

### 视觉/声音

Asset Catalog 提供 `CharacterBible`、`CostumeLook`、`LocationBible`、`StylePack`、`VoiceProfile` 的批准版本。每个 ShotSpec 只能指向具体版本。每个生成结果记录模型/权重/工作流/LoRA/seed（适用时）、参考图 hash、prompt 模板和输入版权状态。这样才可定位跨集漂移、复现或有选择地重做。

## 6. 调度、恢复和批量生产

1. 季/集计划可以预建，但只在滚动窗口内（建议 1–3 集）释放媒体任务。
2. 任务依赖显式化：剧本批准 → 分镜批准 → 关键帧批准 → 视频/动画 → 音频/字幕 → render → QC → 发布审核。
3. worker 取得有过期时间的 lease；崩溃后新 worker 检查输入 hash、provider remote task 和输出 manifest，安全续跑或创建新 attempt。
4. 每个 provider 有并发上限、指数退避、熔断、单集/单季预算；本地 Qwen 与视频模型为互斥资源组。
5. `FAILED_FINAL` 必须显示错误、输入版本、下一步（重试、换 provider、人工修改、取消）；不可无限重试。

这比当前单媒体任务 JSON 恢复严格得多，是 P4 的核心，而不是 UI 增强。

## 7. Mac mini + Mac Studio 部署

| 节点 | 职责 | 保护条件 |
| --- | --- | --- |
| Mac mini | Toonflow UI/API、生产 DB、调度、审查、轻量 render/元数据 | 不将 3000 直接暴露公网；完成既有 P0 安全项。 |
| Mac Studio | 现有 Qwen 文本服务（稳定资源） | 不改配置、不重启；媒体实验仅独立服务/端口/维护窗口，且与 Qwen 错峰。 |
| 独立媒体 sidecar（未来） | ComfyUI、TTS/ASR 或受控视频候选 | 只通过适配层访问；资源限额、健康检查、无共享密钥。 |
| 云 provider（可选） | 高质量/弹性视频或备用生成 | 能力合同、预算闸门、数据流/保留期/条款准入。 |

推荐生产策略：文本规划优先使用受控本地 Qwen；图片优先本地候选的隔离验证结果；视频采取“动态漫画优先、少数高价值镜头用经批准云端或验证通过的本地 sidecar”。不得假设 Studio 64GB 可与 Qwen 同时稳定运行大型视频模型。

若将来要复用 Studio 统一内存，模型切换必须由受控的 Studio Resource Controller 完成：排空 Qwen → 经验证卸载 → 单一媒体批次 → 经验证卸载媒体 → 用不可变 Qwen manifest 恢复。不得以杀进程或暂停信号代替生命周期管理；现有 MLX 服务是否有热卸载能力尚未确认。完整状态机、KeepAlive 风险、恢复策略和 Go/No-Go 门见 [模型资源调度方案](./MODEL_RESOURCE_SCHEDULING.md)。

## 8. 方案比较

| 方案 | 优点 | 失败模式 | 结论 |
| --- | --- | --- | --- |
| 直接在 Canvas/节点图中拼流程 | 早期演示快 | 没有跨集真相、事务、审批、恢复边界 | 不采纳为生产核心。 |
| 一开始微服务/消息集群 | 理论可扩展 | 过早运维复杂度，MVP 难验收 | 当前不采纳。 |
| 模块化单体 + DB + durable worker | 可复用 Toonflow，边界清晰，可逐步拆分 | 需先做 ADR、迁移和状态机 | 推荐。 |
| 仅靠云端“全自动” | 初期视频质量/弹性可能高 | 成本、隐私、供应商锁定、版权 | 仅作受控备选。 |

## 9. 安全、合规与不应改动的稳定模块

- 先完成 `ToDo.md` 的 P0 安全/备份和 P1 可观测性；未完成时不得多项目/外部访问。
- provider/插件为高权限代码入口，只允许审核过的来源、固定版本和最小网络/文件权限。
- 密钥仅以 secret reference 存储；资产、音色、参考图、提示词及云端传输纳入审计和数据保留策略。
- 保留现有稳定 Qwen/MLX 与 OpenCode 基础设施，不为了媒体试验升级、重启或改参数。
- 发布必须保留真人声音同意、角色/IP/音乐/字体权利、模型与权重许可证、平台 AI 披露证据。FFmpeg 实际二进制的 LGPL/GPL 组成需要单独复核。

## 10. 验收架构的最小条件

MVP-1 至少能从一个经过审批的 EpisodeCard 产生可追溯的 ShotSpec、镜头资产、音频/字幕、render manifest 和人工终审记录；重启后不重复计费或丢失已完成镜头。没有这些证据，不应宣称“自动短剧生产平台”。具体阶段和候选施工项见 [路线图](./AI_SERIES_PRODUCTION_ROADMAP.md)。

## 8. 对账修订记录与设计决定（2026-11-15，CodexQ 复核）

完整对账见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md)。本架构的「画布不是生产数据库」「人工审批是数据状态而非 UI 按钮」「不建议修改的稳定模块」等原则**继续有效**。

### 8.1 修订要点

| 修订项 | 内容 |
| --- | --- |
| 架构基调 | 由「新增独立的短剧领域边界」修订为「**以既有 skill 契约为抽取来源，渐进加固**」，避免与现场产物命名双轨 |
| 现有能力表 | 补入 7 项此前遗漏的可复用模块：三个已安装 skill、Agent 工作区文件工具、文档工作区、`context.media` 无头扩展点、`director3dNode`、FFmpeg `concat`/`complexFilter` 工具、计费账本与用量、站内发布 |
| 执行路径 | 新增「画布路径（浏览器绑定）vs 无头路径（缺薄工具）」的强制区分，这是批量生产可行性的决定性约束 |
| 状态机 | 状态枚举对齐 `v118workflow` 的中文阶段状态，并把 `GenerationTask` 状态对齐 `utils/media/tasks.ts` 现有枚举，不新造 |
| storyboard team | 降权：本机 `data/agents/` 目录不存在、未安装，且 readme 自述只读不写工作区；真正在用且更完整的是 `v118workflow` 的 `storyboardTable/storyboardPanel/storyboardImages/directorPlan` 与 PART A `A4–A9` |
| 真相源策略 | 补入分阶段策略：**第一阶段 = 工作区文件（沿用 skill 目录契约）+ 只读索引器**；第二阶段才由 `P1-002` ADR 决定是否入库。避免在契约未固化前承担迁移成本 |

### 8.2 九项设计决定（D1–D9）

| # | 决定 | 理由 |
| --- | --- | --- |
| D1 | 不重写图片/视频/音频节点与 provider 基础 | 已 `IMPLEMENTED` 且含 checkpoint/resume/cancel 与模型能力声明；重写等于放弃既有计费与恢复语义 |
| D2 | 不新建媒体 API；本地模型一律走自定义 provider 安装通道 | `POST /api/ai/media/generate` + `routes/providers/media/{add,save,validate}.ts` + `utils/media/provider.ts` 已覆盖 |
| D3 | 新增无头媒体工具（消费 `context.media`），使批量不依赖浏览器 | 画布路径受 120s 超时与「画布须打开」限制，且回传仅进程内有效 |
| D4 | 第一阶段真相源为工作区文件 + 索引，数据库化留待 ADR | 降低契约未稳定时的迁移风险；`v118workflow` 目录契约已可用 |
| D5 | 状态枚举对齐既有 skill | 避免与现场产物脱节的双轨状态 |
| D6 | 动态漫画优先，真视频按镜头选择性使用 | 降低角色漂移、成本与本地视频未验证风险；`director3dNode` 可补足运镜 |
| D7 | 滚动窗口排产：按季纲预建计划、按集锁定输入、只释放 1–3 集 | 避免一次创建数千任务；与 skill 的逐集确认关口一致 |
| D8 | Qwen 常驻优先；媒体批次只在人工批准窗口且 Studio 侧确认空闲后运行 | 当前**无法判断**其他项目是否在用 Qwen（其他客户端直连 `:8000`；Toonflow 无在途计数）→ **禁止自动卸载** |
| D9 | 发布默认人工；站内通道不加自动化，外部平台发布另建留痕 | 权利、AI 披露与平台合规的责任边界 |

### 8.3 应当取消的重复设计

| 原设计 | 处置 |
| --- | --- |
| 全新 `SeriesBible/SeasonPlan/EpisodeCard/ScenePlan/ShotSpec` 命名体系 | 改为从 `v118workflow` 三画布契约 + PART A `A2/A4/A9/A10` **反向抽取** schema |
| 全新连续性规则目录与冲突/豁免格式 | 改为把 PART A `A0.1 / A8 / A11` 固化为可执行校验与测试 |
| 全新生成留痕/版本快照机制 | 改为把 `stateAndDelivery.md` 的四张表（成果/生成尝试/片段采用/交付）与「确认绑定历史快照」结构化 |
| 全新 timeline/concat 基础能力 | 取消；`concat`/`mergeToFile`/`complexFilter` 已在能力表且工具已安装，只需 manifest + 确定性 worker + 字幕产物 |
| 全新运镜/关键帧预演能力 | 取消；`director3dNode` 已提供 |
| 全新成本账本数据结构 | 改为扩展 `multiTenantBillingLedger` 范式（requestId 幂等、reserved/settled/released、requestHash、usage、micros 金额） |
| 全新「AI 回复转剧本」的文件系统 | 取消；`workspace` 工具的 write/edit + 文档工作区 + revision 冲突检测已足够 |

`ARCHITECTURE_RECONCILED`
`IMPLEMENTATION_NOT_STARTED`
