# Toonflow 生产级架构审查报告

> 审查日期：2026-11-15（Asia/Hong_Kong）
> 审查范围：本地源码快照 `/Users/azhong/Toonflow/Toonflow-app`；未启动服务、未调用媒体 API、未修改数据库或业务代码。
> 证据口径：`VERIFIED` 为本次直接阅读源码确认；`PARTIAL` 为有实现但不满足生产目标；`MISSING` 为在源码范围内未发现实现；`UNVERIFIED` 为需要真实环境/供应商验收；`PROPOSED` 为建议设计，非现有能力。

## 1. 执行摘要

结论：**Toonflow 适合作为个人创作工具和半自动工作室的交互底座；不适合在当前状态直接作为商业级批量自动短剧生产系统。**

它已经具备成熟度较高的无限画布、插件节点、工作区文件边界、**文档工作区（文件树/搜索/标签页/revision 冲突检测）**、文本模型供应商设置、**统一媒体生成接口（图片/视频/音频三合一）**、媒体任务检查点恢复、FFmpeg 基础处理（**含 concat 与复杂滤镜**）和素材库；并且**已内置三个生产级 Agent skill**（`v118workflow` 分集制作流程、`workflow` 含 SEEDANCE EPISODE DIRECTOR PART A `A0–A12`、`canvas`），它们已经定义了分集目录契约、阶段确认关口、连续性规则、生成留痕与版本快照约定。它没有**服务端强制的**短剧领域模型、剧集/镜头资产谱系、服务端工作流编排队列、跨项目调度、可审计的人工审批状态机、确定性成片渲染流水线、自动质量门禁、**剧集维度**成本归集、备份/保留策略或生产可观测性。

> **口径修订（第三轮对账）**：首轮把「缺领域模型/缺分镜契约/缺审批门」表述为完全缺失，属**低估**。准确表述是：这些能力**以 Agent skill 契约 + 工作区文件/画布 JSON 的形式已存在**，缺的是 schema 校验、服务端状态机、幂等键、租约与跨进程恢复。详见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md) §0、§1.5。

在修复 P0 安全暴露并完成 P1 的基础架构后，建议以“**半自动、人工审核、云端视频为主**”作为首个可运营目标；本地 Mac Studio 保持专注 Qwen 文本/规划，不和高显存视频模型并发争用。批量自动化应在可恢复的持久化编排服务建立后再启动。

### 本次基线与审查限制

| 项目 | 结果 |
| --- | --- |
| 实际目录 | `VERIFIED`：`/Users/azhong/Toonflow/Toonflow-app` |
| Git 分支/提交 | `VERIFIED`（首轮审查基线）：`master` / `6086cdffceb704a632eca143dd16cd461a78cdac`（2026-11-05） |
| 远端 | `VERIFIED`：`origin https://github.com/a88web3/Toonflow-app.git`；后续审查文档以独立文档提交/推送 |
| 工作区状态 | `VERIFIED`：已有用户修改 `packages/mcp/src/stdio.ts`；本审查未读写该文件 |
| 许可证 | `VERIFIED`：仓库 `LICENSE` 为 MIT（Copyright 2026 HBAI-Ltd） |
| 运行验证 | `UNVERIFIED`：本次不启动/重启 3000、4096、8000，不发送模型或媒体请求 |

## 2. 实际系统架构

```text
Vue 3 Web / Desktop / Mobile
  └─ 画布、节点、Pinia 工作区状态、素材选择
       │ HTTP / SSE
       ▼
Bun + Express (`apps/server`，默认 3000)
  ├─ 配置 conf/settings.json (0600)
  ├─ 工作区与全局素材文件系统
  ├─ AI 文本网关 ── OpenAI-compatible / Responses / Anthropic
  ├─ 媒体供应商加载器 ── apiMart / compshare / metaso / 自定义 provider
  ├─ 媒体任务 JSON 检查点 ── `data/mediaTasks/.../task*.json`
  ├─ FFmpeg 远程执行/下载管理
  ├─ MCP、Agent、A2A、插件安装
  └─ 可选多租户网关（SQLite 或 MySQL：用户、会话、钱包、路由）
       │
       ├─ 当前 Mac Studio：Qwen OpenAI-compatible API（后续用户确认已接入；稳定性矩阵仍待验收）
       ├─ 云端图片/视频供应商（内置三家适配器）
       └─ 本机 FFmpeg（按需安装）

缺失：短剧领域项目数据库、工作流编排器/队列、持久化依赖图、成片渲染器、QC 服务、成本账本聚合、观测/备份服务。
```

### 模块与耦合判断

| 模块 | 状态与证据 | 评估 |
| --- | --- | --- |
| Web/画布 | `VERIFIED`：`apps/web/src/pages/workspace/panels/canvas/index.vue` 将节点、边、视口直接保存为 `toonflowCanvas` JSON | 创作交互可复用；领域状态混在画布 JSON，不适合作为生产事实源 |
| Server | `VERIFIED`：`apps/server/src/app.ts` 装配 CORS、JSON、路由、MCP、A2A、静态站点 | 单进程 Express 是桌面/个人工具合理起点，不是多 worker 控制平面 |
| 插件/节点 | `VERIFIED`：`packages/nodeScaffold` + `packages/nodes/*` + `apps/server/src/utils/plugins/*` | 强可扩展性；插件可执行代码是信任边界，需签名/准入治理 |
| Agent | `VERIFIED`：`apps/server/src/agent/runtime/*`，另有 `packages/teams/storyboardTeam` | 通用创作 Agent 已有；示例团队只读、只交付文字分镜，非短剧自动生产控制器 |
| 多租户 | `PARTIAL`：`packages/multiTenant/*` 有用户/会话/钱包/路由，`apps/server/src/utils/multiTenant/runtime.ts` | 商业账户和计费基础存在；没有项目生产数据模型，且部署/安全需先验收 |

## 3. 实际功能清单与短剧流水线核对

| 阶段 | 状态 | 源码证据与实际能力 | 生产缺口 |
| --- | --- | --- | --- |
| 故事创意/剧本 | `PARTIAL` | 文本节点、Agent 与文本模型调用：`packages/nodes/textNode`、`apps/server/src/routes/ai/generate.ts` | 无剧本 schema、版本、集/场/镜层级、批准流或约束校验 |
| 人物/场景设定 | `PARTIAL` | 图片生成节点支持参考图：`packages/nodes/imageGenerationNode/src/index.vue` | 无角色 bible、服装/场景/道具 canonical ID、参考锁定与跨镜检验 |
| 分镜 | `PARTIAL` | `packages/teams/storyboardTeam` 明确生成可审阅文字分镜；`frameAnalysisNode` 可拆已有视频 | 无结构化 shot list、镜头版本、镜头依赖/状态机 |
| 图片生成 | `PARTIAL` | `imageGenerationNode`，三类内置云适配器位于 `packages/providers/src/media/*` | 无 ComfyUI 实现、无批量队列和一致性 QA；单节点输出为主 |
| 视频片段 | `PARTIAL` | `videoGenerationNode` 支持文生、图生、首尾帧/混合参考；任务可恢复 | 无供应商无关的作业 SLA、配额并发、镜头批量编排、自动重试策略 |
| 配音/TTS | `PARTIAL` | `audioGenerationNode` 抽象支持 text、voice、参考音频；内置适配器实际模型能力须逐家验证 | 没有角色声音库、台词切分、时轴对齐、语音一致性评分 |
| 字幕/ASR | `PARTIAL` | `packages/asr` 依赖浏览器 SpeechRecognition；其 README 明确可能上传音频到浏览器供应商 | 无离线 ASR、SRT/ASS/VTT 产物、断句/校时/审校流 |
| 音效/BGM | `PARTIAL` | 可生成/处理音频，FFmpeg 工具可处理媒体 | 无音乐授权库、cue sheet、混音/响度规范/ducking 工作流 |
| 合成与导出 | `PARTIAL`（**由 `MISSING` 修订**） | `packages/ffmpeg/src/browserTypes.ts:21-24` 的 `runMethods` 含 `concat`/`mergeToFile`/`concatenate`，`chainMethods` 含 `complexFilter`/`filterGraph`/`audioFilter`/`videoFilter`；Agent 侧 `ffmpeg`+`ffprobe` 工具已安装（`packages/tools/ffmpeg/src/index.ts:59,68`，`data/tools/ffmpeg.tool.js`），经宿主 worker 执行且**不依赖浏览器**；另有 `videoNode` 截取/抽音轨、`audioNode` 变速裁剪、`frameAnalysisNode` 本地切镜 | 仍缺确定性 render manifest、字幕产物（SRT/ASS/VTT）与烧录、响度/母版规格、可复现 render worker 与渲染重试；本机 FFmpeg 是否含 `libass`/`subtitles` 为 `UNVERIFIED` |
| 生产 playbook（分集流程/连续性/留痕） | `PARTIAL` | `packages/skills/v118workflow/`（三画布契约 + 阶段关口 + 状态枚举）、`packages/skills/workflow/SKILL.md`（PART A `A0–A12`，含 A0.1 剧情保真锁、A2 源剧情账本、A8 跨段连续性检查、A11 结构校验）、`references/stateAndDelivery.md`（制作状态台账四表 + 确认绑定历史快照）；三者**本机均已安装**（`data/skills/`） | 由模型自觉执行而非服务端强制；存于画布 JSON/Markdown，无 schema 校验、无幂等键、无租约、无跨进程恢复 |
| 质量检查 | `PARTIAL` | `frameAnalysisNode` 可做镜头切分/AI 分析 | 无自动质量门禁、角色相似度、黑帧/静音/码率/字幕安全区/版权 QC |

### 中间产物、重试和人工审核

`VERIFIED`：媒体任务在 `apps/server/src/utils/media/tasks.ts` 以每工作区 SHA-256 路径保存 JSON；包含 `providerData`、结果文件哈希、状态并可经 `status(..., resume=true)` 恢复。节点侧 `packages/nodeShared/src/useNodeGeneration.ts` 也把任务状态写回画布。

`PARTIAL`：该恢复是“单一媒体生成请求”的恢复，不是故事至成片的 DAG 恢复。没有依赖级别的幂等键、全局队列、死信、指数退避、操作员接管界面、跨进程租约或审计记录。人工审核目前是用户在节点/画布上手动判断，并非审批状态机。

## 4. 模型与媒体集成审查

### 文本模型：Mac Studio Qwen

`VERIFIED`：`apps/server/src/utils/ai/provider.ts` 接受 `openai-completions`、`openai-responses`、`anthropic-messages`；`apps/server/src/utils/ai/models.ts` 会读 `/v1/models`，30 秒超时；`routes/ai/generate.ts` 使用 SSE、20 秒 keepalive、远程计费路线 5 分钟上限。

`PARTIAL`：可把现有 Qwen 填为 OpenAI-compatible provider，模型发现、SSE 文本流和工具调用上下文均有代码路径。没有针对 MLX/Qwen 的 capability profile、结构化输出契约回归、模型级并发闸门、token 预算策略或请求级可观测性。因此“可配置”不等于“已稳定使用”。

`PROPOSED`：建立 `qwen-local` 受控 provider profile：固定 Studio URL、模型 ID、上下文/输出上限、并发=1、30/120/300 秒分级超时、幂等 `requestId`、JSON schema 失败重试一次、trace ID。不得改 MLX 参数；先用非付费探针验收模型列表、流式、工具/JSON、长上下文、取消和并发拒绝。

### 图片：ComfyUI 与本地生成

`MISSING`：在源码中搜索 `ComfyUI/comfyui`，仅 README/翻译文案提及，未发现 provider、API client、workflow schema、队列轮询或角色 LoRA/ControlNet/IP-Adapter 管线。因此不能把 README 声称当成已实现集成。

`PARTIAL`：图片 provider 抽象支持 text、singleImage、multiReference；`packages/providers/types.d.ts` 定义了 image 请求、媒体预处理与恢复接口。内置适配器是 `apiMart.ts`、`compshare.ts`、`metaso.ts`，均以云端 API 为主。

对 Mac Studio 64GB 的判断为 `UNVERIFIED`：静态内存数字不足以证明可与当前 Qwen 共存。生产建议是**工作负载互斥**：Qwen 常驻时不启动重型扩散/视频模型；图片本地化前，用独立维护窗口测量峰值统一内存、首图时间、连续 20 张失败率和热恢复，再决定是否切换模型或独立主机。

### 视频

`PARTIAL`：视频节点、参考图/首尾帧/视频/音频输入、轮询和续作业均可见。`apiMart.ts` 与 `compshare.ts` 对异步任务保存 taskId/deadline，`metaso.ts` 有 MiniMax H3 轮询。`packages/providers/types.d.ts` 的接口契约明确支持 resumeTask。

`MISSING`：本地视频生成服务适配器、供应商无关任务队列、全局并发/预算、片段拼接、质量评测和供应商降级规则。

建议：MVP 选择一个已合同化的云视频供应商作为主路径，一个人工切换的备选路径；按镜头而非整集提交；限制单项目和全局并发；把供应商 task ID、输入哈希、费用快照、输出校验写入生产数据库。Mac Studio 不应假设同时承载 27B 文本模型和大型视频扩散。

### 音频与成片

`VERIFIED`：音频节点能生成/处理；`packages/nodes/audioNode/src/audioProcessing.ts` 支持变速、裁剪并验证输出；`packages/ffmpeg` 封装本地 FFmpeg。`packages/asr/readme.md` 明说浏览器原生识别可能将音频发送到浏览器供应商。

`MISSING`：角色声纹资产、离线/可控 ASR、字幕格式生成、台词时码、混音总线、响度标准、确定性合成 worker 与导出规格。

`PARTIAL`（**修订**：首轮将「视频合成」整体判为 `MISSING`，不准确）：`concat`/`mergeToFile`/`concatenate` 与 `complexFilter`/`filterGraph` 已在 FFmpeg 能力表中，且 `ffmpeg`/`ffprobe` 已作为 Agent 工具安装，可无头调用；缺的是 manifest 驱动的确定性流水线与字幕产物。

音频契约补充（`SOURCE_VERIFIED`）：HTTP 层 `audioGenerationSchema`（`apps/server/src/utils/media/schemas.ts:44-59`）**已支持** `voice`/`speed`/`volume`/`pitch`/`language`/`format`/`sampleRate`，`MediaModel.voices` 亦已定义；`emotion` **仅在** provider 合同 `AudioRequest`（`packages/providers/types.d.ts:175`），不在 HTTP schema；`audioGenerationNode` UI **未暴露** voice 选择器（`src/index.vue:102`）。故多角色音色在 API 层可达，缺节点 UI 与「已安装 provider 是否声明 audio model」的配置事实（`UNVERIFIED`）。

## 5. 数据、任务与生产记录

### 当前实现

- `VERIFIED`：画布是工作目录 JSON，前端防抖串行保存（`apps/web/src/pages/workspace/panels/canvas/index.vue`）；服务端 `writeAtomic` 与路径边界由 `apps/server/src/utils/workspace/files.ts` 保护。
- `VERIFIED`：全局素材库存于 data 目录的 `assets`，支持分页查询；单文件上限 100MB（`apps/server/src/utils/assets/index.ts`）。
- `VERIFIED`：媒体任务记录是 data 目录下私有 JSON，并保存检查点/结果哈希（`apps/server/src/utils/media/tasks.ts`）。
- `PARTIAL`：可选多租户 SQLite/MySQL 有用户、会话、钱包、模型路由和 source mapping（`packages/multiTenant/db/*`）。SQLite 由 `bun:sqlite` 打开，MySQL 路径的账本使用 InnoDB 事务。
- `VERIFIED`（**第三轮补充**）：多租户侧已有**持久计费账本** `multiTenantBillingLedger(requestId PK, userId, currency, kind∈{credit,model}, status∈{reserved,settled,released}, requestHash, reservedMicros, amountMicros, details, usage, createdAt, updatedAt)` 与 `multiTenantWallet(balanceMicros, reservedMicros)`（`packages/multiTenant/db/sqlite.ts:95-118`、`mysql.ts:63-72`）；媒体任务记录携带 `chargeId`/`chargeStatus` 并在成功时按实际秒数结算（`apps/server/src/utils/media/tasks.ts:44-45,271-305,346-349,414-421`）；文本 token 用量解析见 `apps/server/src/utils/ai/billingUsage.ts`。**但个人/单机模式（`modelGateway.enabled()===false`）直接调用 `generateMedia`，不产生任何 charge 记录**（`apps/server/src/agent/tools/index.ts:52-59`），且账本无 series/episode/shot 维度。

### 生产适配性

SQLite 对**单机、单进程、单人或小量受控用户**可以作为控制数据的起点；不应承载不受限的多 worker 并发编排、对象元数据、审计/分析和大规模账务。现有多租户表也不是短剧项目模型。

`MISSING`：版本化 schema migration runner、备份校验/演练、保留策略、存储配额、对象引用计数、项目/集/镜头/产物版本表、删除审批、灾难恢复 RPO/RTO。服务重启后单媒体任务可尝试恢复，但业务流水线的状态一致性未成立。

`PROPOSED`：生产数据面应使用 Postgres（或先受控 MySQL）保存项目、剧集、镜头、资产、任务、尝试、审批、成本、质量报告和审计事件；媒体存对象存储或受控本地卷，DB 只存内容哈希/URI/保留状态。SQLite 仅保留为桌面离线单项目模式。

## 6. 安全、隐私与商业风险

| 优先级 | 发现 | 状态/证据 | 影响与建议 |
| --- | --- | --- | --- |
| P0 | 默认服务 CORS 全开且 `app.listen(port)` 未限定 host | `VERIFIED`：`apps/server/src/app.ts` 的 `app.use(cors())`；`apps/server/src/index.ts` | 若 3000 可被局域网/反向代理访问，可能暴露无认证 API、工作区及本机能力。先绑定 loopback 或明确受控网关、严格 allowlist CORS、加鉴权/CSRF/速率限制。 |
| P0 | 未启用多租户时，多数 API 不经用户认证 | `VERIFIED`：`multiTenantAccess` 在 `isEnabled()` false 时直接 next；工作区允许本地来源 | 3000 绝不能直接公网暴露。为个人模式保持 loopback；外部访问必须走独立认证入口。 |
| P0 | 任意插件/供应商源码是高权限执行面 | `VERIFIED`：插件初始化和 provider 安装路径位于 `apps/server/src/utils/plugins/*`、`routes/providers/media/*` | 恶意插件可读配置/调用网络/处理文件。生产必须有签名、来源 allowlist、审核、隔离与撤销。 |
| P1 | API Key 进入 settings/provider 配置 | `VERIFIED`：`providerSchema` 有 apiKey，`conf` 文件 mode 0600 | 本机权限保护已有基础，但无 OS Keychain/KMS、密钥轮换/掩码审计/租户隔离证明。迁移为 secret reference。 |
| P1 | 浏览器 ASR 有外传风险 | `VERIFIED`：`packages/asr/readme.md` | 用户音频/剧本可能离开设备。默认禁用生产自动转写，改用经合同批准的 ASR，并展示数据流告知。 |
| P1 | 供应商媒体和文本均会出站 | `VERIFIED`：provider fetch 与云视频适配器 | 按供应商建立 DPA、数据地域、保留期、训练使用、版权/肖像条款的准入台账。 |
| P1 | 无明确 API 限流/安全头/审计日志实现 | `MISSING`：未发现 rate limiter/helmet/audit logging 入口 | 易被滥用，无法追责。加入网关控制、结构化审计与告警。 |

MIT 仅覆盖 Toonflow 源码；不自动覆盖第三方依赖、模型权重、训练数据、字体、音乐、图片、角色肖像、供应商 API 输出或分发权。每项商业上线前必须由法务核对具体模型/素材/云服务当期条款；本报告不是法律意见。

## 7. 三个商业目标成熟度

| 目标 | 当前成熟度 | 可复用模块 | 缺失与主要风险 | 改造量（假设 1 名全职工程师） |
| --- | --- | --- | --- | --- |
| 个人创作工具 | 中等，约 55% | 画布、节点、素材、文本/图/视频接入、FFmpeg 基础 | 网络暴露、供应商可用性、无一键合成/字幕、资产连续性主要靠人工 | 4–8 周（P0–P2 + 手动导出流程） |
| 半自动工作室 | 初始，约 30% | 上述 + 可恢复媒体任务 + 分镜团队示例 | 无项目/镜头模型、审批、版本、成片、可追溯成本/QC | 3–5 人月（P0–P3，取决于一个供应商与明确交付规格） |
| 批量自动生产系统 | 很低，约 10% | provider 抽象、任务检查点、可选多租户账本 | 无持久 DAG/队列、租约、并发、幂等、恢复、指标、质量门禁、数据治理 | 9–15 人月（P0–P6；不含模型训练/法务/供应商认证） |

估算假设：既有工程可持续维护；只覆盖中文短剧、一个主视频供应商、一种 1080p/25fps 成片规格、单区域部署；不含训练自有模型、移动端重构、云资源采购和人工内容审核成本。成本估算不使用 README 宣传价作为生产报价，应由后续供应商实测账单确定。

## 8. 分阶段改造路线图与关键决策

| 阶段 | 目标 | 出口条件 |
| --- | --- | --- |
| P0 | 防止未授权访问/数据泄露，冻结运行边界 | 3000 不再直接暴露；密钥与插件准入策略可验证；备份基线完成 |
| P1 | 建立可观测、可迁移、可恢复的基础 | 结构化日志、健康/指标、迁移和恢复演练、架构回归清单 |
| P2 | 受控文本/媒体集成 | Qwen profile 验收；一个图片、一个视频、一个音频路径有 capability contract 与成本记录 |
| P3 | 交付可审核短剧流水线 | 项目/剧集/镜头/资产/审批/成片 manifest 可追溯；可输出一集 |
| P4 | 持久任务编排 | 队列、依赖 DAG、租约、并发/优先级、幂等、断点恢复、人工接管 |
| P5 | 质量与运营 | 自动技术 QC、人工质量门、成本仪表盘、保留/清理、审计 |
| P6 | 压测和生产验收 | 故障演练、容量/成本基线、SLO、备份恢复和发布门禁通过 |

关键决策：

1. **画布不是生产数据库。** 保留它为创作 UI；新增领域服务为唯一生产事实源。
2. **先云视频、后本地视频。** Mac Studio 先稳定承担 Qwen；本地视频只有独立容量测试通过才接入。
3. **一个媒体任务不等于一条生产工作流。** 复用现有 checkpoint，但新增持久 DAG、租约与幂等。
4. **把人工审批设计成数据状态，不是 UI 按钮。** 每个 script/asset/shot/render 均需要版本、批准人、时间和原因。
5. **不建议修改的稳定模块：** 当前 MLX/OpenCode 基础设施；`@toonflow/file` 的原子写入/路径边界；现有媒体任务检查点机制；多租户钱包的事务原则。改造应在适配层与新生产服务完成。

## 9. 推荐部署：Mac mini + Mac Studio

`PROPOSED`：

- Mac mini：Toonflow UI/API、生产控制平面、Postgres/MySQL（或独立受管实例）、队列 worker、FFmpeg 渲染 worker、监控与备份代理。3000 只 loopback，由认证反向代理或 VPN 访问。
- Mac Studio：仅 MLX Qwen 的现有稳定 API；以受控 OpenAI-compatible profile 接入。单并发文本基线，不重启/调参。
- 图片：初期云端供应商；若以后本地 ComfyUI，单独服务、明确 API、队列和资源互斥，不与 Qwen 同时抢占统一内存。
- 视频：初期云端异步任务；本地视频必须为单独容量池/机器，不能假设与 27B Qwen 同机并行。
- 存储：本地生产前至少采用加密卷 + 快照 + 异机备份；规模化转对象存储，按项目隔离、内容哈希、生命周期策略和恢复演练治理。

## 10. 验收前禁止做出的声明

- 不得称“已接入 ComfyUI”或“本地视频生成已可用”。
- 不得称“Qwen 稳定支持工具调用/结构化输出”，仅因能返回文本。
- 不得称“可自动完成短剧”，现有代码没有端到端编排或成片导出。
- 不得把当前 JSON 检查点表述为批量任务灾难恢复方案。
- 不得用 README 价格作为成本承诺；成本必须由实际模型、时长、重试率、存储和人工审核测量。

## 11. 第一批推荐施工任务

施工入口与依赖关系以根目录 `ToDo.md` 为准。首批 READY：`P0-001`、`P0-002`、`P0-003`、`P1-001`、`P1-002`。它们只建立安全/证据/数据基线，不改变 MLX/OpenCode 稳定服务。

## 12. 审查结论

在 P0 与 P1 完成前，**不建议**将 Toonflow 的 3000 服务暴露给真实用户，也不建议开始批量生成。完成 P0、P1 并通过 ToDo 所列验收后，可以启动 OpenCode 进行 P2/P3 的受控施工；每个任务必须先更新真实证据，不能按计划自动跳阶段。

`AUDIT_COMPLETE`
`NOT_READY_FOR_OPENCODE`（当前不具备启动生产功能施工的安全前置条件；P0/P1 基线任务完成并验收后重新判定）

## 13. 第二阶段本地媒体研究增补（2026-11-15）

本轮没有改变首轮审查结论。新增的 [本地媒体可行性研究](LOCAL_AI_MEDIA_FEASIBILITY.md) 确认：Qwen 已由用户确认正常接入；ComfyUI 和 LTX Desktop 有官方 Apple Silicon 路径，LTX 是本地视频首个隔离验收候选；Wan2.2 5B 是第二候选但官方证据仍是 CUDA/NVIDIA；HunyuanVideo 1.5 官方要求 Linux/CUDA，不适合作为当前 Studio 路径。所有 Mac Studio M5 Max 64GB 的生成速度、内存峰值、稳定性和画质均仍为 `UNVERIFIED`，禁止以 NVIDIA 基准替代。

建议保持“Mac mini 编排、Studio 单媒体 job、与 Qwen 错峰、云视频人工批准兜底”的架构。新增计划任务位于唯一施工入口 `ToDo.md`；模型安装/实测前仍不得进入生产施工。

## 14. 第三阶段对账增补（2026-11-15，CodexQ 复核）

本轮以源码与本机运行时安装状态为准，对首轮审查做了定向纠正；首轮的安全结论、P0/P1 优先级与「画布不是生产数据库」判断**继续有效**。完整对账见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md)。

### 14.1 纠正的判断

| 首轮判断 | 修订 | 依据 |
| --- | --- | --- |
| 合成与导出 `MISSING`（无 timeline/concat） | 改为 `PARTIAL`：concat/mergeToFile/concatenate/complexFilter 已在能力表且 `ffmpeg`/`ffprobe` 工具已安装、可无头执行 | `packages/ffmpeg/src/browserTypes.ts:3-24`、`packages/tools/ffmpeg/src/index.ts:59,68`、`data/tools/ffmpeg.tool.js` |
| 无剧本 schema / 无集场镜层级 / 无批准流 | 改为「**契约已存在于 skill，缺服务端强制**」 | `packages/skills/v118workflow/`、`packages/skills/workflow/SKILL.md` PART A `A0–A12`、`references/stateAndDelivery.md`；本机 `data/skills/` 已安装三者 |
| Agent 输出只能复制 | 改为「Agent 具备工作区 `write`/`edit`/`grep`/`glob`，可直写正式剧本文件」 | `packages/tools/workspace/src/index.ts:13-35`、`apps/server/src/agent/tools/index.ts:73-124` |
| 成本账本 `MISSING` | 改为 `PARTIAL`：多租户侧已有持久 ledger 与任务级 charge 绑定；缺个人模式记录与剧集维度 | `packages/multiTenant/db/{sqlite,mysql}.ts`、`apps/server/src/utils/media/tasks.ts` |
| 未提及 `director3dNode` | 补入：文字生成场景/动画/运镜，关键帧导出 PNG、动画导出 MP4；但 `renderVideo` 用浏览器 `MediaRecorder`，**不可无人值守** | `packages/nodes/director3dNode/{readme.md,src/renderMedia.ts:5,22}` |
| 未提及文档工作区 | 补入：文件树、快速打开、全工作区搜索、标签页/分屏、revision 冲突检测 | `apps/web/src/pages/workspace/panels/document/*` |
| 未提及站内发布 | 补入：作品（MP4/WebM）+ 封面 + 项目发布至 Toonflow 社区；外部平台发布仍缺失 | `apps/web/src/pages/home/publishWorkDialog.vue`、`apps/server/src/routes/channel/community/*` |

### 14.2 新增的关键架构事实

1. **两条执行路径必须分开设计**：
   - **画布路径**（现有）：Agent → `canvas` 工具 → `canvasCall` 事件 → 前端执行节点函数。要求画布在浏览器中打开，120s 超时，回传仅在当前 server 进程存活期有效（`apps/server/src/agent/bridge/canvas.ts:10,45`）。适合人工在环的单镜头创作，**不适合无人值守批量**。
   - **无头路径**（缺一个薄工具）：`context.media` 已注入 `listModels/generateImage/generateVideo/generateAudio`，实现直连服务端 `generateMedia`（`apps/server/src/utils/media/generation.ts:243`，默认落盘 `assets/generated`），**不需要浏览器**；但**当前无任何工具消费它**。新增一个插件工具即可打通批量生产，且**不需要新建媒体 API**。
2. **并发与资源锁完全缺失**：`utils/ai`、`routes/ai`、`packages/multiTenant` 全文 grep `inflight|concurren|semaphore|rateLimit` 零命中；唯一锁是进程内按路径的 `lockWorkspaceFiles`（`apps/server/src/utils/workspace/files.ts:133`）。因此 P4-001/P4-003 是真实缺口，且是模型互斥切换的前置。
3. **无法判断其他项目是否在用 Qwen**：其他客户端（OpenCode/codexm/codexg 等）直连 `http://192.168.0.10:8000/v1`，完全绕过 Toonflow；`fetchProviderModels` 只是 30s 超时的存活探测（`apps/server/src/utils/ai/models.ts:12-26`）。Toonflow 只能排空**自己进程内**的 Agent 运行（`sessions.ts:80-101 stopAgentRuns`），且不覆盖直接打 `/api/ai/generate` 的请求。**结论：禁止自动卸载 Qwen，模型切换一律人工审批 + 维护窗口**（详见 [模型资源调度方案](./MODEL_RESOURCE_SCHEDULING.md) §10）。
4. **本机无生产数据**：`data/workspaces/myTvProject` 为 0 文件、`data/assets/` 为空 → 任何「已跑通端到端」的表述都不成立；所有产能/成本数字当前只能是 `ESTIMATE` 或 `UNVERIFIED`。

### 14.3 对施工入口的影响

`ToDo.md` 的 28 项**本轮未修改**。建议的调整（含 P3-004 前提纠正、P3-002 基线更换、P1-002 输入更换、以及 ADD-001～ADD-008 新增候选）见对账报告 §13，等待用户批准后统一写入。

`AUDIT_RECONCILED`
`IMPLEMENTATION_NOT_STARTED`
