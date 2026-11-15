# Toonflow 架构对账与统一修订报告

> 日期：2026-11-15（Asia/Hong_Kong）。接管方：CodexQ（复核 CodexG 的研究、架构与路线图）。
> 范围：只读源码审查 + 运行时安装状态盘点（仅目录/文件名，未读取 `data/settings.json`）+ 文档修订。
> 本轮**未**启动 OpenCode、未安装模型、未生成任何媒体、未修改 Qwen/MLX/服务、未改动业务代码、未修改 `ToDo.md`。
> 基线：分支 `master`，HEAD `0715557`，`origin https://github.com/a88web3/Toonflow-app.git`，与 `origin/master` 无本地分叉。工作区存在**无关**未提交修改 `packages/mcp/src/stdio.ts`，本轮未读写、未提交。
> 状态词：`SOURCE_VERIFIED`（本轮直接阅读源码或运行时目录确认）、`RESEARCHED`（公开资料核对）、`UNVERIFIED`（需真实配置/供应商/环境验收）、`REQUIRES_DEVICE_TEST`（需 Mac Studio 实测）；能力分级 `IMPLEMENTED` / `PARTIAL` / `UI_ONLY` / `NOT_IMPLEMENTED`。

## 0. 本轮最重要的纠正

前序报告整体方向正确（安全优先、画布不是生产数据库、本地视频未验证、Qwen 必须保护），但存在**系统性低估现有能力**的偏差。本轮源码审查发现 5 项必须纠正的事实：

| # | 前序报告的判断 | 源码事实 | 影响 |
| --- | --- | --- | --- |
| C1 | 「无剧本 schema、无集/场/镜层级、无批准流」「`storyboardTeam` 只产生可审阅文本」 | 仓库已内置 **3 个生产级 skill** 且本机已安装：`v118workflow`（分集制作流程，定义 `画布/剧本.json`+`画布/资产.json`+`画布/分集/第NNN集.json`、阶段确认关口、阶段状态枚举）、`workflow`（1406 行，内嵌 SEEDANCE EPISODE DIRECTOR **PART A：A0–A12**，含 A0.1 剧情保真锁、A2 源剧情账本、A8 跨段连续性检查、A11 结构校验）、`canvas`。证据：`packages/skills/`、`data/skills/{canvas,v118workflow,workflow,initialized}` | 连续剧集的**产物契约、分集结构、连续性规则、人工确认门已经存在**，只是以「Agent skill + 画布/Markdown 文件」形式存在，而非服务端强制的数据库 schema。新工作应是**形式化与持久化既有契约**，不是重新设计一套 |
| C2 | 「Agent 只能复制文本，不能写文件」 | `packages/tools/workspace/src/index.ts:13-35` 注册 read/ls/grep/glob/**write**/**edit**/downloadImage（`readOnly` 为 false 时）；本机已安装 `data/tools/workspace.tool.js`。文件上下文见 `apps/server/src/agent/tools/index.ts:73-124` | 「把 AI 回复保存为正式剧本」**已可实现**（Agent 写工作区 Markdown/JSON），缺的是专用 UI 按钮与版本/审批语义 |
| C3 | 「没有 timeline、concat、字幕烧录」 | `packages/ffmpeg/src/browserTypes.ts:21-24` 的 `runMethods` 含 `concat`/`mergeToFile`/`concatenate`；`chainMethods` 含 `complexFilter`/`filterGraph`/`audioFilter`/`videoFilter`/`outputOptions`；Agent 侧已安装 `ffmpeg`+`ffprobe` 工具（`packages/tools/ffmpeg/src/index.ts:59,68`，`data/tools/ffmpeg.tool.js`） | 合成能力**在工具层已可达**；真正缺的是 render manifest、确定性 worker、字幕产物与规格门禁。实际 FFmpeg 二进制是否含 `libass`/`subtitles` 滤镜为 `UNVERIFIED` |
| C4 | 未提及 `director3dNode` | `packages/nodes/director3dNode/`：文字生成场景/动画/运镜，关键帧可导出 PNG 图片节点、动画可导出 MP4 视频节点（`src/renderMedia.ts:5,22`）。**但 `renderVideo` 使用浏览器 `MediaRecorder`（MP4 avc1.420028、6 Mbps、~30fps），readme 明确要求导出时窗口可见** | 运镜预演与关键帧导出**已有现成节点**可复用；同时它证明「浏览器绑定」是本仓库自动化的硬约束，不能用于无人值守批量 |
| C5 | 未提及文档工作区与发布能力 | `apps/web/src/pages/workspace/panels/document/`（`fileTree.vue`、`quickOpen.vue`、`workspaceSearch.vue`、`documentSession.ts:72` 基于 revision 的 `writeTextSnapshot` 冲突检测、标签页/分屏）；`apps/web/src/pages/home/publishWorkDialog.vue` + `apps/server/src/routes/channel/community/uploadExploreProject.ts` | 「继续编辑已保存剧本」「刷新后恢复」「导出」「发布」都有现成载体；发布目前只面向 **Toonflow 社区**，不是 TikTok/YouTube |

**统一结论**：Toonflow 不是「只有画布和节点、缺一切生产能力」的系统，而是「**已具备完整的 Agent 驱动单集生产 playbook 与执行工具面，缺服务端强制的领域契约、无人值守批量调度、自动 QC 与成本归集**」。因此本轮把架构基调从「新建短剧生产系统」修订为「**把既有 skill 契约固化为可校验、可恢复、可调度的服务端能力**」。

## 1. Toonflow 现有能力清单（`SOURCE_VERIFIED`）

### 1.1 本机运行时安装状态

仅列目录/文件名；未读取 `data/settings.json`（含密钥），故「是否已配置可用凭据」一律 `UNVERIFIED`。

| 类别 | 已安装项 | 路径 | 判断 |
| --- | --- | --- | --- |
| 媒体 provider | `apiMart.ts`、`compshare.ts`、`metaso.ts`、`initialized` | `data/providers/` | 3 个内置**云端**适配器已就位；**无任何本地/自定义媒体 provider** |
| 节点 | 9 个全部就位（含 `director3dNode`、`frameAnalysisNode`、`audioGenerationNode`） | `data/nodes/*.umd.js` | `IMPLEMENTED` |
| Agent 工具 | 11 个全部就位：`askUser`、`assetOperator`、`browser`、`canvas`、`ffmpeg`、`pluginDeveloper`、`sandbox`、`skillOperator`、`webFetch`、`webSearch`、`workspace` | `data/tools/*.tool.js` | `IMPLEMENTED`；各工具 `enabled`/`readOnly` 配置为 `UNVERIFIED` |
| Skill | `canvas`、`v118workflow`、`workflow`、`initialized` | `data/skills/` | `IMPLEMENTED`；`conf.disabledSkills` 实际内容 `UNVERIFIED` |
| 文档编辑器 | `ext-md`、`ext-txt`、`ext-image`、`ext-media`、`ext-canvasNode` | `data/ext/` | `IMPLEMENTED` |
| 团队 | `data/agents/` **目录不存在**（无 `initialized` 标记） | — | `storyboardTeam` 仓库内存在但**本机未安装**；对比 `data/skills`、`data/tools`、`data/nodes`、`data/providers` 均有 `initialized` → 团队安装步骤从未执行 |
| 工作区 | `myTvProject`（**0 文件**）、`画布1.json` | `data/workspaces/` | 本机**无任何真实生产数据**，无端到端跑通证据 |
| 全局素材库 | 空 | `data/assets/` | 未使用 |

### 1.2 Agent 能力面（决定自动化的真实边界）

`apps/server/src/agent/tools/index.ts:30-170` 构造的 `ToolContext`（类型见 `packages/toolScaffold/src/runtime.ts:207-236`）：

| 能力 | 是否暴露给模型 | 是否依赖浏览器 | 证据 |
| --- | --- | --- | --- |
| 工作区读/列/搜索/写/编辑 | **是**（`workspace` 工具） | 否 | `packages/tools/workspace/src/index.ts:13-35`、`src/search.ts:47` |
| FFmpeg / FFprobe 执行 | **是**（`ffmpeg`、`ffprobe`） | 否（宿主 worker） | `packages/tools/ffmpeg/src/index.ts:59,68` |
| 画布增删改查、连边、调用节点函数 | **是**（`canvas` 工具 17 个操作） | **是，必须画布已打开** | `packages/tools/canvas/src/runtime.ts:28-112`；`apps/server/src/agent/bridge/canvas.ts:10,45`（120s 超时、`请确认画布仍然打开`、进程内 `pendingCalls`） |
| 媒体生成（图/视频/音频） | **否**（无内置工具消费 `context.media`） | — | `context.media` 已在服务端注入（`apps/server/src/agent/tools/index.ts:146-151`），但全仓库 grep 无任何工具/节点消费它 |
| ASR 转写 | 是（`asr.transcribe`） | **是**（浏览器 SpeechRecognition） | `apps/server/src/agent/tools/index.ts:131-143`、`packages/asr/readme.md` |
| 长期记忆（跨工作区 Markdown） | 是（`memory`） | 否 | `apps/server/src/agent/tools/memory.ts:13-15` |
| 子 Agent / 团队委派 / A2A | 是 | 否 | `apps/server/src/agent/runtime/{subAgent,delegation,agentTeam}.ts`、`agent/a2a/`、`routes/agents/a2a` |
| 技能读写 | 是（`skillOperator` + `SkillContext`） | 否 | `apps/server/src/agent/skills/index.ts:26-70` |

**两条关键约束（必须写进架构）**：

1. **画布路径 = 浏览器绑定**。`node:generateImage` 等节点函数经 `canvasCall` 事件下发前端执行，超时 120s，且回传仅在当前 server 进程存活期有效（`bridge/canvas.ts:10` 代码注释明确「多进程部署需共享请求通道」）。因此**现有节点自动化无法无人值守批量运行**。
2. **服务端媒体路径 = 已存在但未暴露**。`generateMedia(cwd, mediaType, request, signal)`（`apps/server/src/utils/media/generation.ts:243`）是纯服务端函数，默认落盘 `assets/generated`，**不需要浏览器**；`context.media` 已把它注入工具上下文。**新增一个薄插件工具即可让 Agent 无头调用统一媒体接口**——这是本对账给出的最小施工面，且不需要新建任何媒体 API。

节点函数权限受 `conf.settings.agentNodeTools`（`allowImage`/`allowVideo`/`allowAudio` + 追加名单）闸门控制：`apps/server/src/agent/tools/index.ts:40-47`、`packages/tools/canvas/src/runtime.ts:5-25`。

### 1.3 统一媒体生成面

| 项目 | 状态 | 证据 |
| --- | --- | --- |
| 统一入口 `POST /api/ai/media/generate`（image/video/audio，支持已有 `taskId` 或同步等待） | `IMPLEMENTED` | `apps/server/src/routes/ai/media/generate.ts` |
| 配套路由：`models`、`price`、`balance`、`balanceEvents`、`tasks/status`、`tasks/cancel` | `IMPLEMENTED` | `apps/server/src/routes/ai/media/` |
| Provider 合同：`generateImage`/`generateVideo`/`generateAudio`/`processMedia`/`resumeTask`/`saveTaskData` | `IMPLEMENTED` | `packages/providers/types.d.ts:238-266` |
| 自定义 provider 安装/校验/保存（可接本地 sidecar） | `IMPLEMENTED`（机制）/ 未安装任何本地 provider | `apps/server/src/routes/providers/media/{add,save,validate}.ts`、`apps/server/src/utils/media/provider.ts` |
| 请求契约 `MediaGenerationRequest` | `IMPLEMENTED` | `packages/toolScaffold/src/runtime.ts:105-130`：`providerId/modelId/prompt/other/outputDirectory/images[]/videos[]/audios[]/firstFrame/lastFrame/referenceOrder/ratio/size/resolution/duration/generateAudio/voice/speed/volume/pitch/language/format/sampleRate/mode` |
| 模型能力声明 `MediaModel` | `IMPLEMENTED` | 同上 `:80-94`：`rules/supportsPrice/type/mode/imageSizes/imageRatios/durationResolutionMap/audio/voices` |
| 任务持久化与恢复 | `IMPLEMENTED`（单任务） | `apps/server/src/utils/media/tasks.ts`：记录含 `taskId/cwd/mediaType/request/nodeId/canvasPath/ownerUserId/chargeId/chargeStatus/state/checkpoint{providerData,assets,files+hash}/blobs/result`；导出 `start/status(resume)/cancel/wait`（`:457,510,534,564`） |
| 节点侧任务快照 | `IMPLEMENTED` | `packages/nodeShared/src/useNodeGeneration.ts:9-22`（`NodeGenerationTask` 类型）、`:33`（`node.data.mediaTask` 读写）、`:47-62`（`getGenerationStatus`/`cancelGeneration`）、`:64-68`（`needsRecovery`） |
| 产物落盘 | `IMPLEMENTED` | `generation.ts:276,281,354`：`<outputDirectory>/<mediaType><taskId><index>.<ext>`；节点用 `assets/<nodeId>/` |
| 批量队列 / DAG / 租约 / 全局并发 | `NOT_IMPLEMENTED` | 全仓库 grep 无 inflight/concurrency/semaphore/rateLimit（`utils/ai`、`routes/ai`、`packages/multiTenant` 均无命中）；唯一锁是进程内按路径的 `lockWorkspaceFiles`（`utils/workspace/files.ts:133`） |

### 1.4 创作、文档与持久化面

| 能力 | 状态 | 证据 |
| --- | --- | --- |
| 无限画布 + 节点 + 防抖自动保存 | `IMPLEMENTED` | `apps/web/src/pages/workspace/panels/canvas/{index.vue,canvasFiles.ts}` |
| 文档工作区：文件树、快速打开、全工作区搜索、标签页/分屏 | `IMPLEMENTED` | `apps/web/src/pages/workspace/panels/document/{index.vue,documentSession.ts,components/fileTree.vue,components/quickOpen.vue,components/workspaceSearch.vue}` |
| 文档保存冲突检测（revision + 磁盘快照） | `IMPLEMENTED` | `documentSession.ts:72`（`writeTextSnapshot(path, text, disk)`） |
| Markdown 编辑器 + 复制 Markdown | `IMPLEMENTED` | `packages/ext/md/src/index.vue:173` |
| 工作区文件 HTTP API | `IMPLEMENTED` | `apps/server/src/routes/workspaces/files/{list,read,write,copy,rename,remove,mkdir,export,importImage,reveal}.ts` |
| 项目 ZIP 导出（1GB 上限） | `IMPLEMENTED` | `routes/workspaces/files/export.ts` |
| Agent 会话持久化 JSONL | `IMPLEMENTED` | `.agent/sessions/<uuid>.jsonl`；`apps/server/src/agent/runtime/sessions.ts:401,417,430,445`（create/rename/list/get）、`:234,334`（delete/branch）、`:219`（reportToParent） |
| @提及工作区文件/画布/节点/节点输出/素材 | `IMPLEMENTED` | `apps/server/src/agent/mentionFiles.ts:14,25,53,82,89,96` |
| 「把这条回复保存为剧本」专用按钮 | `NOT_IMPLEMENTED` | `apps/web/src/components/agent/conversation.vue` 仅有 `copyMessage`（`:296`） |
| 会话 JSONL → Markdown 导出 | `NOT_IMPLEMENTED` | 同上；仅有剪贴板复制与 ZIP 导出 |
| 发布作品（视频 MP4/WebM + 封面 + 项目） | `IMPLEMENTED`（仅 Toonflow 社区） | `apps/web/src/pages/home/publishWorkDialog.vue`、`routes/channel/community/{uploadExploreProject,getExploreProjects,getExploreProject,getExploreDownload,getExploreUploadSignature,getExploreProcess}.ts` |
| 外部平台（TikTok/YouTube）发布与 AI 披露留痕 | `NOT_IMPLEMENTED` | 未发现任何外部平台发布适配器 |

### 1.5 既有生产 playbook（前序报告完全遗漏）

| Skill | 规模 | 已定义的生产契约 | 证据 |
| --- | --- | --- | --- |
| `v118workflow`（v1.1.0「v1.1.8 分集制作流程」） | `SKILL.md` 102 行 + 16 reference + 7 knowledge | 目录契约 `画布/剧本.json`、`画布/资产.json`、`画布/分集/第NNN集.json`；三画布职责划分（剧本=项目配置/来源索引/章节事件/故事骨架/分集大纲/改编策略/各集正式剧本；资产=基础与衍生资产/父子关系/视觉设定/提示词/已选图片/使用集数；分集=导演规划/分镜表/参考素材/分镜图/视频/选用与交付记录）；阶段流程（项目初始化→来源与章节事件→故事骨架→改编策略→逐集剧本→基础资产→衍生资产→导演规划→分镜表→分镜图→视频→交付）；每阶段「读已确认输入→完成→保存→回读检查→展示→`askUser` 确认」；阶段记录节点状态枚举 `草稿/待确认/已确认/生成中/失败/待复核/已跳过`；续作与上游改动的下游影响标记 | `packages/skills/v118workflow/SKILL.md`、`references/{projectSetup,sourceEvents,storySkeleton,adaptationStrategy,episodeScript,baseAssets,deriveAssets,generateAssets,directorPlan,storyboardTable,storyboardPanel,storyboardImages,videoProduction,canvasOperations,sourceMap}.md`、`references/knowledge/{scriptCraft,narrativeGenres,assetDesign,visualStyles,shotCraft,imagePromptCraft,videoPromptCraft}.md` |
| `workflow`（v2.7.0「视频制作工作流」） | `SKILL.md` **1406 行** + 8 reference | 内嵌 **SEEDANCE EPISODE DIRECTOR PART A（A0–A12）**：A0 输入判定、**A0.1 剧情保真锁**（事实来源优先级、五类绝对禁止改动、具体化边界、不补前史、模糊保持模糊、空缺/矛盾处理）、A1 输出语言与资产命名（**A1.2.1 Seedance 2.5 参考图绑定**、A1.3 后期字幕 vs 实体文字）、**A2 全量节拍解析**（源剧情账本、五类关键标签、节拍时间预算、节拍→镜头→生成段、长剧情分批）、A3 分段与时长预算、A4 资产与出镜状态规划、A5 站位（世界坐标+画面坐标）、A6 表演层、A7 运镜与段间衔接、**A8 跨段连续性检查**、A9 分段提示词模板（9.1=2.0 / 9.2=2.5）、A10 全片输出结构、**A11 结构校验**、A12 交付前优化；另有 Seedance 路由锁存与阶段硬隔离规则 | `packages/skills/workflow/SKILL.md`（A0:75、A1:170、A2:270、A3:364、A4:461、A5:553、A6:604、A7:649、A8:714、A9:782、A10:990、A11:1037、A12:1144）、`references/{assets,canvasExecution,sourceAndScript,stateAndDelivery}.md`、`references/methods/{storyDevelopment,visualDesign,colorAndLight,screenAdaptation}.md` |
| `workflow` 的状态与交付契约 | `stateAndDelivery.md` 80 行 | **制作状态台账**（沿用旧版 `production-state.md` 或新建 `productionState.md`，多集各自子目录）；四张表：`成果`（标识/用途场次/版本/画布节点/正文或媒体路径/采用的固定版本/输入依赖/状态与确认依据）、`生成尝试`（尝试标识/资产片段节点/提示词快照/模型与完整有效参数/参考文件版本顺序用途/调用前输出/授权范围与剩余次数/返回与实际文件）、`片段采用`、`交付`；**确认版本必须绑定独立历史快照**；生成前保存提示词/参数/引用/授权快照且后续编辑不回写 | `packages/skills/workflow/references/stateAndDelivery.md` |
| `canvas` | `SKILL.md` + `references/videoProduction.md` | 画布操作手册（addCanvas/addNode/getNodeTools/nodeTools/connectNodes/fitCanvas 标准用法） | `packages/skills/canvas/` |
| `storyboardTeam` | 团队包 | director/writer/reviewer 分工的**只读**文字分镜示例；readme 明确「不执行消耗算力的媒体生成任务，不自动写入工作区」；构建产物安装到 `data/agents/storyboardTeam` | `packages/teams/storyboardTeam/readme.md`；本机 `data/agents/` 目录不存在 → **未安装** |

**含义**：前序报告提出的 `SeriesBible / SeasonPlan / EpisodeCard / ScenePlan / ShotSpec`、`StoryFact / Event / Foreshadow / CharacterState`、连续性三层检查、人工确认门、生成留痕、版本快照绑定——**在语义上已由上述 skill 定义**，只是：(a) 由模型自觉执行而非服务端强制；(b) 存于画布 JSON / Markdown 而非可查询数据库；(c) 无 schema 校验、无幂等键、无租约、无跨进程恢复。

### 1.6 成本、计费与用量（前序报告判断偏保守）

| 能力 | 状态 | 证据 |
| --- | --- | --- |
| Provider 余额/价格查询与 SSE 推送 | `IMPLEMENTED` | `apps/server/src/utils/media/billing.ts`（进程内余额快照、`accountId` HMAC、`priceSchema`/`balanceSchema`）、`routes/ai/media/{price,balance,balanceEvents}.ts` |
| 生成前价格预估 | `IMPLEMENTED` | `apps/server/src/utils/media/generation.ts:230 getMediaPrice`、`utils/media/priceReference.ts`（含 ffprobe 探测视频时长） |
| 文本 token 用量解析（OpenAI/Responses/Anthropic 三协议，含 cache read/write） | `IMPLEMENTED` | `apps/server/src/utils/ai/billingUsage.ts:22-110` |
| 持久计费账本 | `IMPLEMENTED`（**仅多租户模式**） | `packages/multiTenant/db/sqlite.ts:102-118` / `mysql.ts:72`：`multiTenantBillingLedger(requestId PK, userId, currency, kind∈{credit,model}, status∈{reserved,settled,released}, requestHash, reservedMicros, amountMicros, details, usage, createdAt, updatedAt)` + 索引；`multiTenantWallet(balanceMicros, reservedMicros)` |
| 媒体任务与账单绑定 | `IMPLEMENTED`（仅多租户） | `tasks.ts:44-45`（`chargeId`/`chargeStatus`）、`:271-275 finalizeCharge`、`:296-305`（视频按 `readPriceReference` 取秒数，`usage={requests,seconds}`）、`:346-349 reserveSnapshot`、`:414-421`（恢复时校验 charge 状态） |
| 个人/单机模式的成本记录 | `NOT_IMPLEMENTED` | `apps/server/src/agent/tools/index.ts:52-59`：`modelGateway.enabled()` 为 false 时直接 `generateMedia`，**不产生任何 charge 记录** |
| 剧集/镜头维度成本归集、预算阻断、模型加载切换耗时记账 | `NOT_IMPLEMENTED` | 账本无 series/episode/shot 维度；无预算闸门；无资源切换计时 |

## 2. 已有但尚未配置成功的能力

这些能力**代码完整**，卡在本机配置/凭据/模型，不是卡开发：

| 能力 | 阻塞点 | 状态 |
| --- | --- | --- |
| 云端图片生成 | 3 个 provider 已安装但是否填入有效 API Key、各上游是否仍可用/条款如何，本轮未读 `data/settings.json` | `UNVERIFIED` |
| 云端视频生成（含 Seedance 路由） | 同上；`workflow` skill 的 PART A 已按 Seedance 2.0/2.5 写好提示词契约，但没有对应 provider 配置证据 | `UNVERIFIED` |
| 音频生成 / 中文多角色 TTS | API schema 已支持 `voice/speed/volume/pitch/language/format/sampleRate`（`schemas.ts:44-59`），`MediaModel.voices` 已定义（`types.d.ts:25`），provider 合同还有 `emotion`（`types.d.ts:175`）；**但音频节点 UI 未暴露 voice 选择器**（`audioGenerationNode/src/index.vue:102` 的 data 无 voice 字段），且 `emotion` 未进入 HTTP schema，只能经 `other` 透传 | `PARTIAL` + `UNVERIFIED` |
| ASR 转写 | 依赖浏览器 `SpeechRecognition`（`packages/asr/readme.md`），且 `asr.transcribe` 要求画布已连接；无离线引擎 | `PARTIAL`，隐私风险已知 |
| 本地图片/视频 sidecar 接入 | 自定义 provider 机制齐备，但 `data/providers/` 无任何本地 provider | 未安装 |
| Agent 无头媒体生成 | `context.media` 已注入，但无任何工具消费它 | 缺一个薄插件工具 |
| `storyboardTeam` | 本机 `data/agents/` 目录不存在 | 未安装 |
| Qwen 文本 provider | 用户已确认接入（`USER-CONFIRMED`）；但结构化输出合规率、工具调用稳定性、并发行为无验收记录 | `UNVERIFIED` |

## 3. 真正缺失的能力（必须新增）

| # | 缺失能力 | 为什么现有机制不能替代 |
| --- | --- | --- |
| M1 | **服务端强制的领域契约**：Series/Season/Episode/Scene/Shot、Character/Location/Costume/Voice/Style 的 AssetVersion、Review、Render、Publication | skill 契约靠模型自觉；画布 JSON 无 schema 校验、无唯一约束、无事务、不可跨集查询 |
| M2 | **无人值守批量调度**：持久 DAG、幂等键、租约、优先级、并发上限、重试预算、死信、人工接管 | 现有只有「单媒体任务 checkpoint + 恢复」；画布路径还强绑定浏览器与单进程 |
| M3 | **Agent 无头媒体工具**（薄插件消费 `context.media`） | 不新增就无法脱离浏览器批量生成；这是 M2 的前置 |
| M4 | **确定性合成与字幕产物**：render manifest、timeline、字幕（SRT/ASS/VTT）生成与烧录、响度/母版规格、可复现 render worker | `ffmpeg` 工具能 concat/filterGraph，但无 manifest、无规格门禁、无字幕产物、结果不可复现校验 |
| M5 | **自动技术 QC 与一致性检查**：黑帧/静帧/静音/削波/时长漂移/编码帧率/字幕安全区、角色与场景一致性比对、音画同步 | `frameAnalysisNode` 只做本地切镜与 AI 描述；skill 的 A8/A11 是提示词级检查，非确定性度量 |
| M6 | **跨集资产目录与复用**：canonical 角色/场景/服装/声音 ID、版本冻结、引用回填、`assets/<nodeId>` 之外的长期资产库 | 现在产物按 `nodeId` 目录存放，节点删除即失去语义；全局素材库 `data/assets` 为空且与画布资产不同源 |
| M7 | **生产状态与失败恢复的服务端真相源**：任务/尝试/审批/成本/审计事件的可查询存储与重启恢复 | `productionState.md` 是 Markdown 台账，`mediaTasks/*.json` 是私有单任务文件，二者都不可聚合查询、无租约、无跨进程一致性 |
| M8 | **模型资源互斥调度与全局资源锁** | 见第 8 节：Toonflow 无任何 inflight/并发计数，且其他项目直连 `:8000` 完全绕过 Toonflow |
| M9 | **个人模式成本归集与预算闸门** | 计费账本只在多租户模式生效；单机制作无任何成本记录 |
| M10 | **外部平台发布与合规留痕** | 现有发布只到 Toonflow 社区；无 TikTok/YouTube 发布、无 AI 披露/权利检查记录 |
| M11 | **安全基线**：3000 端口 loopback 绑定、CORS allowlist、鉴权/限流/安全头/审计日志、插件与 provider 代码准入 | `apps/server/src/app.ts` 的 `app.use(cors())` 全开；`multiTenantAccess` 未启用时直接放行（沿用前序审查结论，本轮未重复验证） |

## 4. 可直接复用 vs 需要扩展

### 4.1 可直接复用（不要重写）

- 统一媒体接口与 provider 合同：`routes/ai/media/generate.ts`、`packages/providers/types.d.ts`、`utils/media/generation.ts`。
- 单媒体任务 checkpoint / resume / cancel：`utils/media/tasks.ts`、`nodeShared/useNodeGeneration.ts`。
- 自定义 provider 安装与校验通道（本地 sidecar 的唯一正确入口）：`routes/providers/media/{add,save,validate}.ts`、`utils/media/provider.ts`。
- 工作区文件边界与原子写：`utils/workspace/files.ts`（`resolveWorkspacePath`/`writeAtomic`/`lockWorkspaceFiles`/`protectWorkspaceRoot`）、`@toonflow/file`、前端 `apps/web/src/lib/workspaceFiles.ts`。
- 文档工作区（文件树/搜索/标签页/revision 冲突检测）与 ext 编辑器。
- Agent 会话 JSONL、分支、队列、@提及、子 Agent/A2A、`memory`、`askUser`。
- 三个已安装 skill 的**全部产物契约与连续性规则**（最被低估的可复用资产）。
- FFmpeg 宿主 worker 与 `ffmpeg`/`ffprobe` 工具（含 concat/complexFilter）。
- `frameAnalysisNode`（本地切镜，无 Python/视觉模型/API 依赖）用于素材拆解与 QC 取样。
- `director3dNode`（运镜预演、关键帧导出 PNG、动画导出 MP4）用于动态漫画的镜头运动。
- 多租户 `multiTenantBillingLedger` / `multiTenantWallet` 的预占-结算语义（作为成本账本实现范式，不要另造一套）。
- 发布作品通道（作为内部审看/交付载体）。

### 4.2 需要扩展（在既有机制上加薄层）

| 扩展点 | 做法（最小侵入） |
| --- | --- |
| Agent 无头生成 | 新增一个插件工具（与 `packages/tools/*` 同构），消费已注入的 `context.media`，复用 `generateMedia` 与 `tasks.ts`；**不新建媒体 API** |
| 领域契约 | 把 `v118workflow` 的画布/文件契约与 `workflow` PART A 的 A2/A4/A9/A10 字段抽成 JSON Schema，先做**校验器 + 索引**，产物仍写工作区文件；数据库化留待 ADR |
| 分镜→生产任务 | 以 ShotSpec 为输入生成 `MediaGenerationRequest`，复用现有 `providerId/modelId/prompt/images/firstFrame/referenceOrder/mode/duration/resolution/ratio` 字段，不改契约 |
| 批量调度 | 新增独立编排服务（Mini 侧），把每个镜头映射为一次 `generateMedia` 调用 + `tasks.ts` 记录；租约/幂等/重试在编排层，不侵入 provider |
| 合成 | 新增 render manifest schema + 一个确定性 worker，内部调用现有 `executeWorkspaceFfmpeg`；字幕产物与烧录作为 manifest 字段 |
| 成本 | 复用 `billingUsage.ts` + `priceReference.ts` + ledger 范式，补 series/episode/shot 维度与个人模式落盘 |
| 资源互斥 | 见第 8 节；必须在 Studio 侧新增受批准的资源控制器，Toonflow 只做租约客户端 |

## 5. 应当取消的重复开发方案

| 前序方案 | 处置 | 理由 |
| --- | --- | --- |
| 重新设计 `SeriesBible/SeasonPlan/EpisodeCard/ScenePlan/ShotSpec` 全套领域模型（`AI_SERIES_PRODUCTION_RESEARCH.md` §3.1、`ARCHITECTURE.md` §3.1） | **改为「形式化既有契约」** | `v118workflow` 已定义三画布职责与分集目录；`workflow` PART A A2/A4 已定义节拍账本与出镜状态。应从这些文件**反向抽取 schema**，而不是另起一套命名 |
| 新设计连续性规则目录与冲突/豁免格式（`CAND-SERIES-003`） | **改为「把 A0.1 + A8 + A11 固化为可执行校验」** | PART A 已有剧情保真锁（五类绝对禁止改动、事实来源优先级、模糊保持模糊、空缺/矛盾处理）、跨段连续性检查与结构校验 |
| 新设计人工审批状态机（`ARCHITECTURE.md` §3.2） | **保留状态机，但状态枚举对齐既有 skill** | skill 已用 `草稿/待确认/已确认/生成中/失败/待复核/已跳过`；另造一套英文状态机会与现场产物脱节 |
| 新设计生成留痕/版本快照机制（`CAND-SERIES-005`） | **改为「把 `stateAndDelivery.md` 四张表结构化」** | 该文件已定义成果/生成尝试/片段采用/交付四表，以及「确认绑定历史快照」「生成前保存提示词+参数+引用+授权快照」 |
| 新建一套媒体生成 API / 媒体任务模型（`P2-005` 表述含混处） | **取消** | `POST /api/ai/media/generate` + `tasks.ts` + `resumeTask` 已覆盖 submit/status/cancel/resume；sidecar 只需实现 provider 合同 |
| 新建 timeline/concat 基础能力（`P3-004` 的「当前问题」表述） | **取消「基础能力」部分，只保留 manifest/worker/字幕/规格** | concat/mergeToFile/complexFilter 已在 `runMethods`/`chainMethods` 中且工具已安装 |
| 新建分镜团队/分镜提示词方法（`P3-002` 以 `storyboardTeam` 为基准） | **降权** | `storyboardTeam` 本机未安装且 readme 自述只读；真正在用、更完整的是 `v118workflow` 的 `storyboardTable/storyboardPanel/storyboardImages/directorPlan` 与 PART A A4–A9 |
| 新建运镜/关键帧预演能力 | **取消** | `director3dNode` 已提供文字生成场景/动画/运镜、关键帧导出 PNG、动画导出 MP4 |
| 新建成本账本数据结构（`P5-002`） | **改为扩展 `multiTenantBillingLedger` 范式** | 已有 requestId 幂等、reserved/settled/released 状态、requestHash、usage、reservedMicros/amountMicros |
| 新建「AI 回复转剧本」的文件系统 | **取消** | `workspace` 工具的 write/edit + 文档工作区 + revision 冲突检测已足够；只需约定路径与可选的一键保存动作 |

## 6. 专项一（P0）：AI 生成内容的正式保存机制

### 6.1 四类保存必须分开判断

| 层次 | 真相源 | 状态 | 证据 |
| --- | --- | --- | --- |
| ① AI 聊天历史 | `<workspace>/.agent/sessions/<uuid>.jsonl`（session header、用户/助手消息、工具调用与结果、分支/队列 custom entries） | `IMPLEMENTED` | `apps/server/src/agent/runtime/sessions.ts:401,417,430,445`；路由 `routes/agent/{list,get,create,rename,branch,message}.ts` |
| ② 工作区画布 | `<workspace>/**.json`，根标记 `toonflowCanvas: true`；节点 `data`（含 `mediaTask`）、边、视口；编辑后防抖写入 | `IMPLEMENTED` | `apps/web/src/pages/workspace/panels/canvas/{index.vue,canvasFiles.ts}`；`packages/nodeShared/src/useNodeGeneration.ts:9-22,33` |
| ③ 正式剧本/分镜资产 | **无专用实体**。实际落点是两类通用文件：工作区 Markdown/JSON（Agent `write`/`edit` 或用户在文档面板创建）与画布文本节点（`v118workflow` 规定「剧本画布」为正式内容载体） | `PARTIAL` | `packages/tools/workspace/src/index.ts:30-35`；`panels/document/*`；`packages/nodes/textNode`；`packages/skills/v118workflow/SKILL.md`（三画布职责表） |
| ④ 剧集 ↔ 素材关联 | **弱关联**。媒体任务记录含 `nodeId` + `canvasPath`，产物在 `assets/<nodeId>/`；`v118workflow` 规定「资产画布登记使用集数」「分集画布的图片是同一资产文件的本地引用」，但这是**约定**而非外键 | `PARTIAL` | `apps/server/src/utils/media/tasks.ts:42`（`nodeId`/`canvasPath`）；`utils/media/generation.ts:276-281`；skill 契约同上 |

### 6.2 逐项能力回答

| 需求 | 结论 | 状态 | 依据与限制 |
| --- | --- | --- | --- |
| 将 AI 回复保存为正式剧本 | **可以，但靠 Agent 工具或人工，没有一键按钮**。对话区仅 `copyMessage`；正确路径是要求 Agent 用 `write` 写入 `剧本/第NNN集.md`，或用户在文档面板新建文件后粘贴 | `PARTIAL` | `conversation.vue:296` 只有复制；`workspace` 工具已安装（`data/tools/workspace.tool.js`）。`readOnly` 配置未读取 → `UNVERIFIED` |
| 保存 30～100 集剧情规划 | **文件层可以，结构层不行**。`v118workflow` 已规定 `画布/剧本.json` 存「故事骨架与分集大纲」、`画布/分集/第NNN集.json` 每集一画布；但没有集号唯一性、状态汇总、跨集检索或进度视图 | `PARTIAL` | skill 目录契约 + 画布/文件持久化；缺 M1/M7 |
| 继续编辑已保存的剧本 | **可以**。文档面板支持文件树打开、标签页/分屏、Markdown 富文本编辑，保存走 `writeTextSnapshot` 并带 revision 与磁盘冲突检测；Agent 亦可用 `edit` 增量修改 | `IMPLEMENTED` | `panels/document/{index.vue,documentSession.ts:72,components/fileTree.vue}`；`packages/ext/md` |
| 为每集保存独立版本 | **不能自动**。只有「手工另存/复制文件」或 `stateAndDelivery.md` 要求的「确认前保存独立历史快照」这一**约定**；无版本号、无 diff、无「批准绑定某快照」的强制机制 | `NOT_IMPLEMENTED`（机制）/ `PARTIAL`（约定） | `packages/skills/workflow/references/stateAndDelivery.md`（快照与确认绑定段落）；`routes/workspaces/files/copy.ts` 可作手工快照基础 |
| 刷新或重启后恢复 | **可以**。聊天从 JSONL 重建；画布与文档从磁盘 JSON/Markdown 重读；媒体任务可 `status(..., resume=true)` 续跑。**例外**：进行中的画布回调通道与 Agent 运行不跨 server 重启（进程内 `pendingCalls`/`activeRuns`，代码注释明确「进程退出后不恢复任务」） | `IMPLEMENTED`（有明确例外） | `agent/runtime/sessions.ts:72-73,80-81`（注释 + `activeSessions`/`activeRuns`）、`bridge/canvas.ts:10`、`utils/media/tasks.ts:510` |
| 导出 Markdown / JSON | **部分**。文件本身即 `.md`/`.json`，可直接读取、`reveal` 到 Finder、或整项目 ZIP 导出（1GB 上限）；Markdown 编辑器有「复制 Markdown」。**缺**：会话 JSONL→Markdown 导出、按集/按剧本的结构化 JSON 导出 | `PARTIAL` | `routes/workspaces/files/{read,reveal,export}.ts`；`packages/ext/md/src/index.vue:173` |
| 将剧本直接转换为分镜或生产任务 | **Agent 可以，需画布打开**。`canvas` 工具支持 `addCanvas/addNode/connectNodes/nodeTools`，`v118workflow` 与 PART A 已定义从剧本到分镜表到分镜图到视频的完整推进方式；但没有「选中剧本→生成分镜」的确定性服务端动作 | `PARTIAL` | `packages/tools/canvas/src/runtime.ts:93-112`；`bridge/canvas.ts`（浏览器依赖、120s 超时）；`packages/skills/v118workflow/references/{storyboardTable,storyboardPanel,storyboardImages}.md` |

### 6.3 最小扩展方案（不新建文件系统）

1. **约定优先**：直接采用 `v118workflow` 已定义的目录与画布契约作为正式产物布局，不另设路径规范。
2. **版本快照**：复用 `routes/workspaces/files/copy.ts` 语义，在「提交审阅」时写 `history/<集号>/<时间戳>-<hash>.md`，并把该路径写入台账（对齐 `stateAndDelivery.md` 的「确认绑定快照」）。
3. **索引而非数据库先行**：新增一个只读索引器，扫描 `画布/剧本.json`、`画布/分集/*.json` 与 `productionState.md`，产出 `集号 → 状态/产物路径/版本/依赖` 的 JSON 索引；先解决「30–100 集看不见全貌」的问题，再决定是否入库（交由 `P1-002` ADR）。
4. **一键保存**（可选、低优先）：在对话区增加「保存为工作区文件」动作，内部调用既有 `files.write`；不新增存储机制。
5. **结构化导出**：会话 JSONL→Markdown 导出可作为独立小功能，复用 `sessions.ts` 的读取逻辑。

## 7. 专项二（P0）：完整生产链路能力矩阵

链路：`AI Agent → 剧本 → 分镜 → 图片生成 → 视频生成 → 音频生成 → 自动合成 → 成片保存 → 发布`

**核心判断：已有节点功能 ≠ 已有端到端自动生产能力。** 下表逐环节区分。

### 7.1 链路矩阵

| 环节 | 状态 | 源码证据 | 输入/输出数据结构 | 需人工复制 | 支持自动调用 | 失败恢复 | 产物落盘 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| L1 Agent → 正式剧本 | `PARTIAL` | `packages/tools/workspace/src/index.ts:30-35`；`panels/document/documentSession.ts:72` | 入：对话上下文/@提及文件；出：工作区 `.md`/`.json` 或画布文本节点 | 否（Agent 直写）；无一键按钮 | **是**，且不依赖浏览器 | 文件级；无版本/审批 | `<workspace>/剧本/**.md`、`画布/剧本.json` |
| L2 剧本 → 分镜 | `PARTIAL` | `packages/tools/canvas/src/runtime.ts:93-112`；`packages/skills/v118workflow/references/{storyboardTable,storyboardPanel,directorPlan}.md`；`packages/skills/workflow/SKILL.md` A2/A4–A7 | 入：剧本文本；出：画布节点（分镜表文本节点 + 分镜图/视频节点）与节拍账本 | 否 | **是，但必须画布在浏览器中打开**（120s 超时） | 画布防抖保存；无阶段级续跑 | `画布/分集/第NNN集.json` |
| L3 分镜 → 图片生成 | `PARTIAL`（单镜头 `IMPLEMENTED`，批量 `NOT_IMPLEMENTED`） | `packages/nodes/imageGenerationNode/src/index.vue:492`；`routes/ai/media/generate.ts`；`utils/media/generation.ts:243,276-281`；`utils/media/tasks.ts:457,510,534,564` | 入：`MediaGenerationRequest{providerId,modelId,prompt,images[],ratio,size,other,outputDirectory}`；出：`GeneratedMedia[]{path,mimeType,mediaType}` | 否（引用用 `{{ref N}}` + `referenceOrder` 绑定） | 是（`node:generateImage`，受 `agentNodeTools.allowImage` 闸门）；无头路径缺 M3 | **是**（单任务 checkpoint + `resume`；节点 `needsRecovery()`） | `assets/<nodeId>/image<taskId><index>.<ext>` |
| L4 图片 → 视频生成 | `PARTIAL`（同上） | `packages/nodes/videoGenerationNode/src/index.vue:505`；`MediaGenerationRequest.firstFrame/lastFrame/videos/audios/mode/duration/resolution/generateAudio`；`MediaModel.durationResolutionMap`、`audio?:boolean\|"optional"` | 入：图片/首尾帧/参考视频音频 + 提示词；出：`VIDEO` 产物 | 否 | 是（`node:generateVideo`，`allowVideo` 闸门） | **是**（同 L3；provider 侧 `resumeTask`） | `assets/<nodeId>/video<taskId><index>.<ext>` |
| L5 音频生成（配音） | `PARTIAL`，生产可用性 `UNVERIFIED` | 节点 `packages/nodes/audioGenerationNode/src/index.vue:347`；schema `utils/media/schemas.ts:44-59`（含 `voice/speed/volume/pitch/language/format/sampleRate`）；合同 `providers/types.d.ts:167,175`（`voice`/`emotion`）、`:25`（`MediaModel.voices`） | 入：文本 + 可选一个图片或音频参考（不可同时）；出：`AUDIO` 产物 | 否 | 是（`node:generateAudio`，`allowAudio` 闸门） | 是（同 L3） | `assets/<nodeId>/audio<taskId><index>.<ext>` |
| L5' ASR / 字幕校验 | `PARTIAL`，隐私风险 | `packages/asr/readme.md`（浏览器原生识别，可能上传音频）；`agent/tools/index.ts:131-143`（要求画布连接） | 入：音频路径；出：文本 | — | 是，但**浏览器绑定** | 无 | 不落盘（返回值） |
| L6 自动合成（剪辑/字幕/混音） | `PARTIAL` | Agent `ffmpeg`/`ffprobe` 工具（`packages/tools/ffmpeg/src/index.ts:59,68`）；`packages/ffmpeg/src/browserTypes.ts:3-24`（`concat`/`mergeToFile`/`concatenate`/`complexFilter`/`filterGraph`/`audioFilter`/`outputOptions`）；`videoNode` 截取/抽音轨、`audioNode` 变速裁剪、`frameAnalysisNode` 切镜 | 入：工作区媒体相对路径 + fluent-ffmpeg 方法链；出：合成文件 | **是**（需人工/Agent 组织清单与参数） | 是（Agent 可无头调用，宿主 worker 执行） | 无（一次性命令，无 manifest 无重试） | 由调用方指定，须在工作区内 |
| L7 成片保存 | `PARTIAL` | 工作区文件 + `assets/`；ZIP 导出 `routes/workspaces/files/export.ts`（1GB）；`reveal` | 出：母版文件 + 可选 ZIP | 是 | 否（无 render manifest/版本登记） | 无 | `<workspace>/**`、ZIP |
| L8 发布 | `PARTIAL`（仅站内） | `apps/web/src/pages/home/publishWorkDialog.vue`（MP4/WebM + JPG/PNG/WebP 封面 + 项目）；`routes/channel/community/uploadExploreProject.ts` 等 | 出：Toonflow 社区作品 | 是（人工选择文件与封面） | 否 | 无 | 远端社区存储 |
| X1 批量调度 / DAG / 租约 | `NOT_IMPLEMENTED` | 全仓库无 inflight/concurrency/queue（仅 `lockWorkspaceFiles` 进程内路径锁，`utils/workspace/files.ts:133`） | — | — | 否 | 否 | — |
| X2 自动技术 QC | `PARTIAL` | `frameAnalysisNode`（本地切镜：75% RGB 差 + 25% 直方图差，输出每镜 JPG/MP4/M4A + JSON 报告）；`ffprobe`；skill 的 A8/A11 为提示词级检查 | 出：切点、镜头素材、JSON 清单 | — | 部分 | 否 | 画布节点 + 工作区素材 |
| X3 人工审核门 | `PARTIAL` | `askUser` 工具 + skill 阶段关口与状态枚举（`草稿/待确认/已确认/生成中/失败/待复核/已跳过`）；`v118workflow`「不要把检查通过、提问取消、跳过问答、超时或沉默视为批准」 | — | — | 是（流程约定） | — | 画布「阶段记录」文本节点 |
| X4 成本归集 | `PARTIAL`（仅多租户） | `multiTenantBillingLedger`；`tasks.ts:44-45,271-305,346-349`；`billingUsage.ts`；`priceReference.ts` | 出：requestId/usage/amountMicros/status | — | 是（多租户模式） | 是（reserved→settled/released 幂等） | SQLite/MySQL |

### 7.2 每环节的最小新增、验证前置与验收标准

| 环节 | 需要新增的最小功能 | 技术验证前置条件 | 未来验收标准 |
| --- | --- | --- | --- |
| L1 | 保存路径约定（沿用 skill）+ 可选「保存为文件」动作 + 只读索引器 | 确认 `workspace` 工具未处于 `readOnly`；确认 skill 未被 `disabledSkills` 关闭 | Agent 能写出指定路径剧本，文档面板可打开、可继续编辑、刷新后仍在，索引器能列出该集状态 |
| L2 | ShotSpec JSON Schema（从 PART A A2/A4/A9 反抽）+ 校验器；把「阶段记录」结构化为可索引 JSON | 确认画布工具在目标环境可用；确认 120s 超时对长分镜是否足够 | 给定剧本，产出通过 schema 校验的分镜集合，字段可追溯到剧本节拍 ID，且写入指定分集画布 |
| L3/L4/L5 | **一个薄插件工具消费 `context.media`**（无头生成），复用 `generateMedia`+`tasks.ts`；产物路径由编排层指定而非 `nodeId` | 至少一个图片/视频 provider 配置可用（`UNVERIFIED`）；音频需先选定可商用 TTS provider | 无浏览器条件下提交→轮询→取消→恢复全链路通过；产物含输入哈希、provider taskId、模型与参数快照 |
| L6 | render manifest schema（镜头顺序、源文件哈希、帧率/分辨率/响度、字幕轨）+ 确定性 render worker（内部调用 `executeWorkspaceFfmpeg`）+ 字幕产物生成 | 实测本机 FFmpeg 是否含 `libass`/`subtitles` 滤镜与目标编码器（`UNVERIFIED`）；确认 concat 对不同编码/帧率源的行为 | 同一 manifest 两次渲染产出比特可解释的一致结果；ffprobe 校验时长/编码/音轨/字幕通过 |
| L7 | 成片版本登记（manifest + 输出哈希 + 规格）与保留策略 | 存储容量基线（见 §9.5） | 任一成片可回溯到 manifest、源素材哈希、任务尝试与批准记录 |
| L8 | 外部平台发布留痕（不自动发布）：披露文案、权利检查、发布记录 | 平台规则复核（`RESEARCHED`） | 每次发布有记录，含 AI 披露与权利检查结论；默认人工触发 |
| X1 | 独立编排服务：持久 DAG、幂等键、租约、并发上限、重试预算、死信、人工接管 | L3 无头工具就绪；资源锁方案获批（§8） | 注入 worker/服务中断后恢复，不重复提交、不覆盖已批准产物 |
| X2 | 确定性 QC 检查器（空文件/编码/帧率/时长漂移/黑帧/静音/削波/字幕缺失）+ 一致性抽检工作台 | FFmpeg 滤镜与 ffprobe 字段可用性实测 | 每镜头有自动 QC 报告与阈值判定；不通过项进入人工队列 |
| X3 | 把 skill 的状态枚举落成服务端状态机 + 审计事件（批准人/时间/理由/绑定快照） | L1 索引器就绪 | 状态只能经服务端命令变更；未批准版本不得流向下游 |
| X4 | 个人模式成本落盘 + series/episode/shot 维度 + 模型加载/切换耗时记账 + 预算闸门 | 确认是否启用多租户；确认 provider `supportsPrice` 真实可用 | 每个付费任务有可审计账单证据；预算超限幂等阻断；估算与实付分列 |

## 8. 专项三（P0）：Mac Studio 模型资源安全调度

### 8.1 硬结论：当前**无法**可靠判断其他项目是否正在使用 Qwen

| 判据 | 源码/环境事实 | 状态 |
| --- | --- | --- |
| Toonflow 是否统计自己的在途文本请求 | **否**。`apps/server/src/utils/ai/`（`provider.ts`/`models.ts`/`billingUsage.ts`/`modelContextLimits.ts`）与 `apps/server/src/routes/ai/` 全文 grep `inflight|concurren|semaphore|activeRequests|rateLimit` **零命中**；`packages/multiTenant` 亦零命中 | `SOURCE_VERIFIED`（缺失） |
| Toonflow 能否看到其他客户端 | **不能**。OpenCode / codexm / codexg 及其他脚本按基础设施约定直连 `http://192.168.0.10:8000/v1`，完全绕过 Toonflow；Toonflow 既不是代理也不是网关 | `SOURCE_VERIFIED`（架构事实） |
| `/v1/models` 探测能否当作负载指标 | **不能**。`apps/server/src/utils/ai/models.ts:12-26` 的 `fetchProviderModels` 只是 30s 超时的存活性/清单探测，不返回在途请求数、KV 占用或内存状态 | `SOURCE_VERIFIED` |
| Toonflow 能否排空**自己**的任务 | **部分可以**。`apps/server/src/agent/runtime/sessions.ts:80-101`：`activeRuns` 在请求进入时即登记，`stopAgentRuns(cwd, runIds)` 可 abort 并等待结束；`activeSessions` 按工作区键索引。**但**：仅进程内有效（代码注释「多进程部署时需共享会话所有权」「进程退出后不恢复任务」），且**不覆盖**直接打 `/api/ai/generate` 的非 Agent 文本请求 | `SOURCE_VERIFIED` |
| `kill` 能否安全卸载 | **不能**。Studio `:8000` 为 system-domain LaunchDaemon，`RunAtLoad`+`KeepAlive`，kill 可能触发立即重启；`SIGSTOP` 保留进程与统一内存，不释放模型 | `RESEARCHED`（沿用 `MODEL_RESOURCE_SCHEDULING.md` §2.2，本轮未改动任何服务） |
| MLX Serve 是否支持热卸载主模型 | **未验证**。官方 `mlx_lm.server` 公开文档只记录 draft 模型卸载；第三方 `raspoli/mlx-serve` 声称有 `/v1/load-model`、`/v1/unload-model`，但不能证明本机用的就是它 | `UNVERIFIED` |

> **因此本轮确立的强制规则：在取得「Studio 侧资源控制器 + 全客户端可观测的排空协议 + 用户明确授权」之前，禁止任何自动卸载 Qwen 的行为；模型切换一律进入人工审批与维护窗口流程。** 这条规则应写入 `P0-004` 与 `MODEL_RESOURCE_SCHEDULING.md`，优先级高于任何吞吐优化目标。

### 8.2 目标状态机（`PROPOSED`，仅在获批后实施）

```
TEXT_ACTIVE ──drain请求──▶ DRAINING ──全部完成/超时──▶ TEXT_IDLE
    ▲                                                    │
    │                                        人工批准 + 资源锁获取
    │                                                    ▼
RESTORING ◀──媒体批次结束/失败/超时── MEDIA_ACTIVE ◀── QWEN_UNLOADED
    │                                        │
    └──健康检查通过──▶ TEXT_ACTIVE            └──加载图片/视频/音频模型
```

| 阶段 | 必须满足 | 失败处理 |
| --- | --- | --- |
| `DRAINING` | 拒绝新文本任务；等待已登记任务完成；**必须先解决 8.1 的可观测性缺口**（Studio 侧统计在途请求，或所有客户端改走统一 ingress） | 超时立即回 `TEXT_ACTIVE`，释放资源锁，媒体批次留在队列 |
| `QWEN_UNLOADED` | 只用**经验证**的生命周期方式（原生 load/unload API，或受批准的本机 supervisor 作为唯一进程所有者）；`QwenServiceManifest`（模型 ID/路径/量化/端口/参数/健康检查样例）hash 锁定 | 卸载失败或 hash 不匹配 → `BLOCKED` + 人工介入，**不得猜测参数恢复** |
| `MEDIA_ACTIVE` | 媒体模型最大并发 1；单一资源锁；心跳与租约；超时看门狗 | 任一异常 → 立即卸载媒体模型 → 恢复 Qwen |
| `RESTORING` | 按 manifest 原样恢复；固定文本 probe（models + 固定 prompt + 流式）通过后才回 `TEXT_ACTIVE` | 恢复失败 → 最高优先级告警，保持 Qwen 优先，媒体任务全部转队列或经批准的云端路径 |

**异常优先级铁律：任何分支失败，第一动作永远是「恢复或保持稳定 Qwen 服务」，第二动作才是处理媒体任务。**

### 8.3 必须考虑的现实约束

| 约束 | 设计应对 |
| --- | --- |
| LaunchDaemon `KeepAlive` | 不允许在 Toonflow worker 中散落 shell kill；切换只能经 Studio 本机受批准的 supervisor 或原生 API；若采用 supervisor 模式，需把 LaunchDaemon 所有权迁移给它（单独审批的基础设施改造） |
| 多项目共享 Qwen | 排空必须以 **Studio 侧真实在途计数**为准，不能以 Toonflow 自己的 `activeRuns` 为准；未建立该计数前不得自动切换 |
| 服务健康检查 | 恢复后固定 probe：`/v1/models` + 一条固定 prompt 的流式补全 + 记录首 token 延迟；probe 失败即视为恢复失败 |
| 模型加载/卸载 | 记录冷/热加载耗时、峰值统一内存、swap 与内存压力趋势；连续多轮切换后检查是否漂移 |
| 统一内存占用 | 64GB 是总量，不是并发许可；图片/视频/音频与 Qwen **不得并发**；音频若为轻量 CPU/MPS TTS，可单独归入 `audio-light` 资源组并先实测 |
| 超时与失败恢复 | 每个阶段独立超时；媒体任务保留 `tasks.ts` 的 checkpoint 语义，切换失败不得丢失已提交任务的状态 |
| 任务排队 | 排队在 Mini 侧编排服务（持久），不在 Studio；Studio 只接受带租约的单作业 |
| 资源锁 | 全局单一 `studio-media` 锁 + fencing token；崩溃后由 reconcile 回收孤儿锁 |
| 崩溃恢复 | 重启后先对账：锁状态、Qwen 是否在位、媒体进程是否残留；不一致则 `BLOCKED` 并告警 |
| 原有配置保护 | 媒体任务无权编辑 `QwenServiceManifest`；不改端口 `:8000`、不改上下文/并发/KV 参数；TTS 不得占用 `:8000` |

### 8.4 本轮不做的事

不停止、不重启、不卸载、不下载、不切换任何模型；不修改 MLX Serve、LaunchDaemon、OpenCode、端口或模型配置；不向 `:8000` 发送探测性 unload 请求。以上均为 `RESEARCH_ONLY`。

## 9. 专项四（P1）：批量生产成本与质量控制

### 9.1 成本记录

| 维度 | 现状 | 状态 | 最小新增 |
| --- | --- | --- | --- |
| 每镜头生成成本 | 多租户模式下已有：`multiTenantBillingLedger(requestId, requestHash, reservedMicros, amountMicros, usage, details, status)` + 媒体任务 `chargeId/chargeStatus`；视频按 `readPriceReference` 取实际秒数计 `usage={requests,seconds}` | `PARTIAL` | 补 `seriesId/episodeId/shotId/assetVersionId` 维度；个人模式（`modelGateway.enabled()===false`）当前**完全无记录**，需补一条本地成本流水 |
| 每集成本汇总 | 无 | `NOT_IMPLEMENTED` | 由镜头成本聚合；区分估算（`getMediaPrice`）与实付（ledger settled） |
| 文本 token 成本 | `billingUsage.ts` 已解析三协议 usage（含 cache read/write） | `IMPLEMENTED`（解析）/ `PARTIAL`（归集） | 把 usage 绑定到剧集阶段（剧本/分镜/连续性审稿） |
| 模型加载与切换耗时 | 无任何记录 | `NOT_IMPLEMENTED` | 由 Studio 资源控制器记录 load/unload/restore 墙钟与峰值内存，回传编排层入账 |
| 重试成本 | 任务级有 `state` 与 charge 状态，但无「同一镜头第 N 次尝试」的归集 | `PARTIAL` | `TaskAttempt` 概念：每次提交一条，关联同一 shotId |
| 存储成本 | 无 | `NOT_IMPLEMENTED` | 按工作区/剧集统计字节数与增长速率 |

**数据口径强制要求**：所有成本与速度数字必须标注来源——`MEASURED`（本机/本账单实测）、`PUBLIC`（官方价表/公开基准）、`ESTIMATE`（公式推算）。当前仓库内**没有任何 `MEASURED` 数据**（`data/workspaces/myTvProject` 为空，`data/assets` 为空），故本轮所有产能与成本表述均为 `ESTIMATE` 或 `UNVERIFIED`。

### 9.2 素材缓存与重复利用

| 项目 | 现状 | 结论 |
| --- | --- | --- |
| 聊天图片下载缓存 | `<workspace>/assets/chatImages/<sha256>.<ext>`，URL hash 去重 | `IMPLEMENTED`（`apps/server/src/utils/workspace/chatImages.ts`） |
| 生成结果内容哈希 | 任务 checkpoint 的 `files[].hash`（sha256） | `IMPLEMENTED`，可作为去重与幂等基础 |
| 跨镜头/跨集资产复用 | 仅 `v118workflow` 约定「资产画布登记使用集数」「分集画布引用同一资产文件」；无 canonical ID、无引用计数、无版本冻结 | `PARTIAL`（约定）/ `NOT_IMPLEMENTED`（机制） |
| 提示词/工作流缓存 | `stateAndDelivery.md` 要求保存提示词与参数快照，但无「相同输入哈希→跳过生成」的机制 | `NOT_IMPLEMENTED` |
| 最小新增 | 以 `输入哈希(providerId+modelId+prompt+参考文件哈希+关键参数)` 为键的产物索引；命中即复用，不重复计费 | — |

### 9.3 失败镜头单独重生成

- 现有基础：单任务 `cancel` / `status(resume=true)` / 节点 `needsRecovery()` / `cancelGeneration`（明确「不删除已有输出，不保证供应商撤销任务或费用」）。`SOURCE_VERIFIED`
- 缺口：无「按镜头重排队」的编排语义；重生成可能覆盖 `assets/<nodeId>/` 内既有文件（文件名含 taskId，故实际不会同名覆盖，但缺少「已批准版本不可覆盖」的强制约束）。
- 最小新增：编排层以 shotId + attemptNo 组织输出目录；已批准产物置只读，重试只产生新 attempt。

### 9.4 一致性与同步检查

| 检查 | 现有能力 | 状态 | 缺口 |
| --- | --- | --- | --- |
| 角色/场景一致性 | skill 层：PART A A0.1 剧情保真锁、A1.2.1 参考图绑定、A8 跨段连续性检查；`v118workflow` 资产画布登记识别锚点 | `PARTIAL`（提示词级） | 无确定性度量（相似度/身份检测）、无自动阻断 |
| 结构校验 | PART A A11 结构校验、A12 交付前优化 | `PARTIAL`（模型自检） | 无 schema 校验器、无可复现测试 |
| 镜头切分/素材核验 | `frameAnalysisNode` 本地检测（无 Python/视觉模型/API），输出每镜 JPG、H.264/AAC MP4、M4A、含切点的 JSON 报告 | `IMPLEMENTED` | 阈值需按项目实测标定 |
| 音频与字幕同步 | 无。ASR 为浏览器原生识别且需画布连接；无强制对齐、无字幕产物、无时码校验 | `NOT_IMPLEMENTED` | 需离线 ASR/对齐 + 字幕格式产物 + 时码比对 |
| 技术 QC | `ffprobe` 可查格式/时长/尺寸/帧率/音轨 | `PARTIAL` | 无黑帧/静音/削波/响度/字幕安全区检查，无阈值门禁 |

### 9.5 30–100 集的容量与存储（全部为 `ESTIMATE`，须实测替换）

以下为**规划公式**，不是承诺值。设：每集 `S` 个镜头、每镜头候选 `N` 次、每产物平均 `B` MB、保留 `K` 个中间版本、季 `E` 集。

```
单集产物字节 ≈ S × N × B × (1 + K)
季度总量     ≈ E × 单集产物字节 + 剧本/分镜文本(可忽略) + 成片母版(E × M)
生成尝试次数 ≈ E × S × N        （决定 API 成本与墙钟）
墙钟下限     ≈ E × S × N × T / 并发数   （T = 单次生成实测耗时，当前 UNVERIFIED）
```

参考量级（取 `S=15, N=3, B=8MB(图片)/60MB(5秒视频), K=2, E=50`）：图片与视频混合下季度素材可达 **数十至数百 GB** 量级；若每镜头另存 5 秒视频候选，量级迅速上升到 TB。**因此在 MVP-1 必须先实测 `B` 与 `T`，再决定保留策略与是否上对象存储。** 现有 ZIP 导出上限 1GB（`routes/workspaces/files/export.ts`）已明显不足以承载整季交付，需要按集分包。

### 9.6 人工审核后才允许发布

- 现有：`askUser` 工具 + skill 阶段关口 + 「不得把检查通过/提问取消/超时/沉默视为批准」的明确规则 + 阶段记录节点状态枚举。`SOURCE_VERIFIED`
- 现有发布通道（站内）无审核门：`publishWorkDialog.vue` 由用户直接选择文件发布。
- 最小新增：`Publication` 记录（终审人、时间、理由、绑定 render manifest 哈希、AI 披露与权利检查结论），默认人工触发，禁止自动发布。

## 10. 图片 / 视频 / 音频模型推荐方案与未验证问题

本节结论沿用并收敛 `LOCAL_AI_MEDIA_FEASIBILITY.md` 与 `AI_SERIES_PRODUCTION_RESEARCH.md` §12 的资料研究，**全部为 `RESEARCHED`，无一项 `VERIFIED_ON_DEVICE`**。接入方式统一为：实现 Toonflow 现有 provider 合同 → 经 `routes/providers/media/{add,save,validate}.ts` 安装为自定义媒体 provider → 由 `generateMedia`/`tasks.ts` 统一调度。**不新建媒体 API。**

### 10.1 图片

| 项目 | 结论 |
| --- | --- |
| 首选商用基线 | **FLUX.1 schnell**（官方列 Apache-2.0），经 ComfyUI sidecar（MPS/Metal）暴露 HTTP API | 
| 中文/动漫对照 | **原版 Qwen-Image**（模型卡 Apache-2.0，官方强调中文文字/编辑/动漫风格） |
| 技术对照（非默认商用） | SDXL + 经审计派生权重；LoRA/ControlNet/IP-Adapter 生态成熟但逐权重许可需审 |
| 商业排除 | FLUX.1 **dev**（非商用）、**Qwen-Image 2.1**（研究/非商用）、任何未审计 LoRA |
| 一致性方案 | 参考图 + 锁定 AssetVersion + 固定 workflow hash + 人工关键帧门；不依赖单一致性算法 |
| 接入点 | 自定义 provider 实现 `generateImage` + `resumeTask` + `saveTaskData`；请求字段用现有 `prompt/images[]/ratio/size/other/outputDirectory` |
| 未验证问题 | M5 Max 64GB 峰值统一内存、首图/后续图耗时、连续 20+ 次失败率、取消后资源释放、与 Qwen 的内存互斥实际影响、ComfyUI 队列/取消语义、角色一致性主观评分 —— 全部 `REQUIRES_DEVICE_TEST` |

### 10.2 视频

| 项目 | 结论 |
| --- | --- |
| 本地首选试验 | **LTX 2.5 Fast / LTX Desktop**（官方有 Apple Silicon 路径，公开建议 48GB，本机 64GB；仍是 beta） |
| 本地第二候选 | **Wan2.2 TI2V-5B**（官方支持 T2V/I2V、720p/24fps、ComfyUI；但官方基准与脚本以 CUDA/NVIDIA 为中心） |
| 不作为当前本地路径 | **HunyuanVideo 1.5**（官方运行要求 Linux + NVIDIA CUDA） |
| 生产兜底 | 已合同化云视频 API（`workflow` skill 的 PART A 已按 **Seedance 2.0/2.5** 写好分段与提示词契约，是天然的云端对接目标）；降级必须人工批准并留痕 |
| 镜头策略 | 动态漫画优先：对话/静景用关键帧 + 平移缩放视差 + FFmpeg；只有运动/转场/情绪关键镜头才用 I2V |
| 接入点 | provider 实现 `generateVideo` + `resumeTask`；字段用 `firstFrame/lastFrame/images/videos/audios/referenceOrder/mode/duration/resolution/ratio/generateAudio` |
| 未验证问题 | Apple 路径可维护性（安装/停用/升级回滚）、5 秒 I2V 的冷/热耗时与峰值内存、稳定 API 与状态查询、取消与断电恢复、身份/时序质量、单镜头实际成本 —— 全部 `REQUIRES_DEVICE_TEST`；云端条款与价格为 `UNVERIFIED` |

### 10.3 音频

| 项目 | 结论 |
| --- | --- |
| 现状判断 | 音频**节点与 API 已实现**，但**无任何已安装的音频 provider 模型**可直接用于中文多角色配音；`data/providers/` 只有 3 个云端聚合适配器，其是否声明 audio model 为 `UNVERIFIED` |
| 候选（资料研究） | CosyVoice（仓库 Apache-2.0，多语言）、Kokoro-FastAPI（仓库称 Apache-2.0 + OpenAI 兼容 speech endpoint，声称有 MPS 路径）、MeloTTS（源码 MIT，称支持中文）、Piper（源码 MIT，原仓库已归档） |
| 关键澄清 | 模型代码的 Apache/MIT **不授予**真人声纹、演员表演、音乐或训练素材权利；参考音色必须有可审计授权与撤销流程 |
| 接入点 | 自定义 provider 实现 `generateAudio`，并在模型声明中填 `MediaModel.voices`；请求用现有 `voice/speed/volume/pitch/language/format/sampleRate`；`emotion` **不在 HTTP schema 内**，只能经 `other` 透传或后续扩展 schema |
| 节点侧缺口 | `audioGenerationNode` UI 未暴露 voice 选择器（`index.vue:102`），多角色配音目前只能靠 Agent/API 直传 `voice`，或扩展节点 UI |
| 资源归类 | 轻量 CPU/MPS TTS 可归 `audio-light` 组并先实测；高质量/克隆 TTS 归 `media-heavy`，与图片/视频/Qwen 互斥；**不得占用 `:8000`** |
| 未验证问题 | 中文音质、多角色音色稳定性、情绪表现、时间戳/强制对齐能力、内存占用、许可与声纹授权 —— 全部 `REQUIRES_DEVICE_TEST` / 法务复核 |

### 10.4 字幕与合成

- 字幕校验候选：Whisper（官方代码与权重 MIT）或 faster-whisper（MIT），用于**校验**而非替代台词时间轴。
- 现有 `packages/asr` 是浏览器原生识别，**可能把音频外传到浏览器供应商**，生产自动转写默认禁用。
- 合成复用现有 FFmpeg 通道；发布前必须核验实际分发二进制是否启用 GPL 组件（FFmpeg 基础 LGPL，可选组件可使整体 GPL）。`RESEARCHED`

## 11. 30–100 集连续漫剧的推荐架构

### 11.1 基调修订

前序架构是「新建短剧生产域 + 外置 worker」。本轮修订为：

> **以既有 skill 契约为产物标准，以工作区文件为第一阶段真相源，以服务端校验器 + 索引器 + 编排服务逐步加固，不重写媒体层、不重写画布、不新建文件存储。**

### 11.2 分层架构

```
┌─ 创作层（完全复用）────────────────────────────────────────┐
│ 文档工作区(md/json) · 无限画布 · Agent 对话 · skills         │
│ v118workflow(分集流程) · workflow(PART A A0–A12) · canvas    │
└──────────────────────────┬──────────────────────────────────┘
                           │ 产物：画布/剧本.json、画布/资产.json、
                           │       画布/分集/第NNN集.json、productionState.md
┌─ 契约层（新增，薄）──────▼──────────────────────────────────┐
│ JSON Schema：SeriesBible/EpisodeCard/ShotSpec/AssetVersion/  │
│              StageRecord/RenderManifest（从 skill 反抽）     │
│ 校验器 + 只读索引器（集号→状态/产物/版本/依赖）              │
│ 版本快照：history/<集号>/<时间戳>-<hash>.md                  │
└──────────────────────────┬──────────────────────────────────┘
┌─ 编排层（新增，Mini 侧）─▼──────────────────────────────────┐
│ 持久 DAG · 幂等键 · 租约 · 并发/预算闸门 · 重试 · 死信       │
│ 滚动窗口：只释放已批准且在预算内的 1–3 集                    │
│ 资源锁客户端：studio-media（单一、带 fencing）               │
└───────┬──────────────────┼───────────────────┬──────────────┘
        │                  │                   │
┌───────▼───────┐  ┌───────▼────────┐  ┌───────▼──────────────┐
│ 文本：Qwen    │  │ 媒体：统一接口 │  │ 合成：FFmpeg worker  │
│ Studio :8000  │  │ generateMedia  │  │ manifest 驱动        │
│ 受保护、常驻  │  │ + tasks.ts     │  │ concat/filter/字幕   │
└───────────────┘  └───────┬────────┘  └──────────────────────┘
                           │ provider 合同
              ┌────────────┼────────────┐
        ┌─────▼─────┐ ┌────▼─────┐ ┌────▼─────┐
        │ 云图片/视频│ │ 本地图片 │ │ 本地视频 │
        │ (已安装3家)│ │ ComfyUI  │ │ LTX 试验 │
        └───────────┘ └──────────┘ └──────────┘
                     ↑ 均经 Studio 资源锁互斥
```

### 11.3 关键设计决定

| # | 决定 | 理由 |
| --- | --- | --- |
| D1 | **不重写图片/视频/音频节点与 provider 基础** | 已 `IMPLEMENTED` 且含恢复语义；重写等于放弃 checkpoint/计费/多模型能力声明 |
| D2 | **不新建媒体 API**；本地模型一律走自定义 provider 安装通道 | `routes/providers/media/{add,save,validate}.ts` + `utils/media/provider.ts` 已具备 |
| D3 | **新增无头媒体工具**（消费 `context.media`），使批量生产不依赖浏览器 | 画布路径受 `bridge/canvas.ts` 的 120s 超时与「画布须打开」限制，无法无人值守 |
| D4 | **第一阶段真相源仍是工作区文件 + 索引**，数据库化留待 `P1-002` ADR | 避免在契约未固化前引入迁移成本；`v118workflow` 的文件布局已可用 |
| D5 | **状态枚举对齐 skill**（`草稿/待确认/已确认/生成中/失败/待复核/已跳过`），不另造英文状态机 | 与现场产物一致，避免双轨 |
| D6 | **动态漫画优先，真视频按镜头选择性使用** | 降低角色漂移、成本与本地视频未验证风险；`director3dNode` 可补足运镜 |
| D7 | **滚动窗口排产**：按季纲预建计划，按集锁定输入，只释放 1–3 集 | 避免一次创建数千任务；与 skill 的逐集确认关口一致 |
| D8 | **Qwen 常驻优先**；媒体批次只在人工批准的维护窗口、且 Studio 侧确认空闲后运行 | 见 §8.1 硬结论 |
| D9 | **发布默认人工**，站内发布通道不加自动化；外部平台发布另建留痕 | 权利与平台合规责任边界 |

### 11.4 放大路径

| 阶段 | 范围 | 出口条件 |
| --- | --- | --- |
| MVP-0 契约固化 | 从 skill 反抽 ShotSpec/StageRecord/AssetVersion schema + 校验器 + 索引器 | 一个手工项目能通过校验并被索引，集号/状态/产物可查询 |
| MVP-1 单集 60 秒 | 1 集、8–20 镜头；本地文本 + 图片（云或本地）+ FFmpeg 动态漫画；云视频仅人工批准；TTS 经批准 provider | 从批准集卡到 9:16 成片，每镜头输入/版本/审核可追溯；服务重启可恢复 |
| MVP-2 连续 3 集 | 共用 Bible、角色/地点/声音版本；滚动释放 | 跨集事实/伏笔冲突可检出并裁决；资产可复用且批准版本不被覆盖 |
| MVP-3 连续 10 集 | 持久 DAG、租约、预算、看板、失败恢复 | 注入中断后恢复，不重复扣费/不覆盖批准产物；所有成片有 manifest |
| MVP-4 30–100 集 | 多季计划、滚动排产、成本账本、运营与发布流程 | 容量/备份/质量/成本指标连续稳定；可导出审计与权利清单 |

## 12. 原有报告修订记录

本轮对 6 份既有文档做了定向修订（保留原有有效结论，只纠正与源码不符处、补入遗漏能力、统一相互矛盾的判断）。每份文档末尾新增「对账修订」小节记录依据。

| 文档 | 修订内容 | 依据 |
| --- | --- | --- |
| `docs/TOONFLOW_WORKSPACE_CAPABILITY_AUDIT.md` | 新增 §8「对账补充」：(1) 三个已安装 skill 构成的分集生产 playbook；(2) Agent 的 `write`/`edit`/`grep`/`glob` 工作区工具；(3) 文档工作区（文件树/快速打开/搜索/标签页/revision 冲突检测）；(4) `director3dNode` 与其浏览器 `MediaRecorder` 约束；(5) FFmpeg `concat`/`complexFilter` 在工具层可达；(6) `multiTenantBillingLedger` 与任务 `chargeId/chargeStatus`；(7) 站内发布通道；(8) `context.media` 作为无头生成的现成扩展点；(9) 画布路径的浏览器/单进程绑定约束。并细化音频 `voice`（HTTP schema 已支持）与 `emotion`（仅 provider 合同有）的区别 | 本报告 §1.2、§1.4、§1.5、§1.6、§7.1 |
| `docs/TOONFLOW_PRODUCTION_AUDIT.md` | (1) §1 执行摘要补入音频节点、统一媒体接口、skill playbook、文档工作区，纠正「只有图片/视频节点」的印象；(2) §2 架构图补入 skills 层与文档工作区；(3) §3 流水线表：`合成与导出` 由 `MISSING` 修订为 `PARTIAL`（concat/complexFilter 可达，缺 manifest/字幕/规格），新增「生产 playbook」行；(4) §4 音频与成片：删除「无视频合成能力」的绝对表述，改为「工具层可达、缺确定性流水线」；(5) §5 补入计费账本与个人模式无成本记录的事实；(6) 新增 §14 第三阶段对账增补 | 本报告 §0 C1/C2/C3、§1.3、§1.5、§1.6、§7.1 |
| `docs/LOCAL_AI_MEDIA_FEASIBILITY.md` | (1) §5 最小集成架构：明确接入必须走 `routes/providers/media/{add,save,validate}.ts` 自定义 provider 通道并实现既有合同（`generateImage/generateVideo/generateAudio/resumeTask/saveTaskData`），**不新建媒体 API**；(2) 补入「Agent 无头媒体工具」为批量前置（画布路径浏览器绑定）；(3) 音频小节补入 schema 事实（`voice/speed/volume/pitch/language/format/sampleRate` 已在 HTTP schema，`emotion` 不在，节点 UI 无 voice 选择器）；(4) 新增 §9 对账修订记录 | 本报告 §1.2、§1.3、§2、§10.3 |
| `docs/AI_SERIES_PRODUCTION_RESEARCH.md` | (1) §3.1 分层产物模型标注为「与既有 skill 契约语义重叠」，工作性质由「设计新模型」改为「从 `v118workflow` + PART A 反向抽取并固化」；(2) §3.2 连续性三层检查标注 A0.1/A8/A11 已存在提示词级实现；(3) §4 资产合同表对齐 `v118workflow` 资产画布字段与 `stateAndDelivery.md` 快照规则；(4) §7 领域模型补入「第一阶段真相源可为工作区文件 + 索引」的选项；(5) §9 成本框架补入既有 ledger/usage/price 能力与「本机无 MEASURED 数据」的口径要求；(6) 新增 §13 对账修订记录 | 本报告 §1.5、§1.6、§4.2、§9.1 |
| `docs/AI_SERIES_PRODUCTION_ARCHITECTURE.md` | (1) §1 架构决策改为「渐进式加固既有 skill 契约」，不再是「新增独立短剧领域边界」的平地起建表述；(2) §2 现有能力表补入 skills、文档工作区、Agent 文件工具、FFmpeg 工具、计费账本、`director3dNode`、`context.media`；(3) §3.2 状态机枚举对齐 skill 的 `草稿/待确认/已确认/生成中/失败/待复核/已跳过`；(4) 新增「两条执行路径（画布路径 vs 无头路径）」与 D1–D9 设计决定；(5) 新增 §8 对账修订记录 | 本报告 §1、§7、§11.3 |
| `docs/AI_SERIES_PRODUCTION_ROADMAP.md` | (1) 新增 MVP-0「契约固化」阶段置于 MVP-1 之前；(2) `CAND-SERIES-001/002/003/005` 标注为与既有任务及 skill 契约重叠，建议合并而非新增；(3) 新增 `ADD-001～ADD-008` 候选（无头媒体工具、Studio 可观测性与排空协议、个人模式成本流水、资产哈希复用索引、版本快照等）；(4) 新增 §8 对账修订记录与验证优先顺序指引 | 本报告 §13、§14 |
| `docs/MODEL_RESOURCE_SCHEDULING.md` | (1) 新增 §10「对账增补：可观测性硬缺口与强制规则」（参考资料顺延为 §11），给出源码级证据（`utils/ai`、`routes/ai`、`packages/multiTenant` 无 inflight/并发计数；其他客户端直连 `:8000` 绕过 Toonflow；`fetchProviderModels` 只是存活探测；`stopAgentRuns` 只能排空 Toonflow 自己的进程内 Agent 运行且不覆盖 `/api/ai/generate`）；(2) 把「无法判断他方使用 → 禁止自动卸载 → 人工审批」升级为强制规则；(3) 把「Studio 侧在途计数或统一 ingress」列为 V2 排空验证的前置条件，并新增 V2.5 恢复后健康 probe | 本报告 §8.1 |
| `ToDo.md` | **本轮未修改**（含状态、顺序、文字）。调整建议见 §13，等待用户批准后统一执行 | 用户指令 |

## 13. `ToDo.md` 任务调整建议（**未写入文件**，等待批准）

现有 28 项：P0×4、P1×4、P2×7、P3×5、P4×3、P5×3、P6×2；状态总览为 5 `READY` / 23 `BLOCKED` / 0 `COMPLETED`。

### 13.1 逐项判定

| 任务 | 判定 | 建议说明 |
| --- | --- | --- |
| P0-001 服务监听、CORS 与入口隔离 | `KEEP` | 结论仍成立，与既有能力无重叠 |
| P0-002 密钥、插件与供应商代码准入基线 | `KEEP` + `REVISE` | 准入范围需显式覆盖「自定义媒体 provider 源码安装通道」，因为本地图片/视频/TTS sidecar 将从同一入口进入 |
| P0-003 数据保护、备份与恢复基线 | `KEEP` + `REVISE` | 备份范围补入 `data/skills`、`<workspace>/.agent/sessions`、画布 JSON、`productionState.md`、`data/mediaTasks`；这些是现有真实产物 |
| P0-004 Studio 媒体工作负载隔离与端口策略 | `REVISE` | 写入 §8.1 硬结论：**当前无法判断其他项目是否在用 Qwen，禁止自动卸载，一律人工审批 + 维护窗口**；把「Studio 侧在途计数或统一 ingress」列为前置 |
| P1-001 结构化日志、健康与任务可观测性 | `REVISE` | 明确必须包含**文本请求在途计数**（当前源码完全缺失），否则 P0-004 的排空无法成立 |
| P1-002 生产领域模型与迁移 ADR | `REVISE` | ADR 输入必须是既有 skill 产物契约（反向抽取 schema）；必须显式评估「工作区文件 + 只读索引」作为第一阶段真相源，不得默认新建数据库 |
| P1-003 本地图片/视频基准与验收协议 | `KEEP` | 与本报告 §10 一致 |
| P1-004 本地模型与 LoRA 商业许可台账 | `KEEP` + `MERGE` | 与 P2-004 的声纹/音色授权、P5-003 的授权审计合并为单一台账，避免三处重复登记 |
| P2-001 Qwen/MLX 受控 provider profile | `KEEP` | — |
| P2-002 ComfyUI 图片 provider 与一致性资产合同 | `REVISE` | 明确经自定义 provider 通道安装并实现既有合同；一致性资产合同字段对齐 `v118workflow` 资产画布，不另设命名 |
| P2-003 主视频供应商能力合同 | `REVISE` | 优先评估 **Seedance 2.0/2.5**，因为 `workflow` skill 的 PART A 已按其写好 A0–A12 完整契约；选其他供应商等于丢弃既有资产 |
| P2-004 TTS/ASR 隐私与角色音色方案 | `REVISE` | 补源码事实：HTTP schema 已支持 `voice/speed/volume/pitch/language/format/sampleRate`；`emotion` 仅在 provider 合同；节点 UI 无 voice 选择器；ASR 为浏览器原生且有外传风险。许可部分并入 P1-004 |
| P2-005 本地媒体 sidecar provider 契约 | `REVISE`（缩小范围） | 删除「新建 submit/status/cancel/resume 契约」表述，改为「映射到现有 provider 合同 + `tasks.ts` 语义」；并**新增子项：Agent 无头媒体工具**（见 ADD-002） |
| P2-006 商用本地图片候选隔离验收 | `KEEP` | — |
| P2-007 LTX/Wan 本地视频隔离验收与云端降级门 | `KEEP` | — |
| P3-001 项目、剧集、脚本与资产版本领域服务 | `MERGE` | 与 P1-002（ADR）和 ADD-001（契约固化）合并；「当前问题」需修订——现状不只是画布 JSON，skill 已定义产物契约与台账 |
| P3-002 结构化分镜与连续性合同 | `REVISE` | 「当前问题」不应只提 `storyboardTeam`（本机未安装且自述只读）；应以 `v118workflow` 的 `storyboardTable/storyboardPanel/storyboardImages/directorPlan` + PART A A2/A4–A9/A8/A11 为基线，工作内容改为「抽 schema + 写校验器」 |
| P3-003 镜头资产生成与人工审核闭环 | `REVISE` | 明确复用 `tasks.ts` 已有的 `nodeId/canvasPath/chargeId/chargeStatus/checkpoint` 字段；审核状态枚举对齐 skill；补「已批准产物只读、重试只产生新 attempt」 |
| P3-004 确定性成片合成与字幕导出 | `REVISE` | **纠正错误前提**：concat/mergeToFile/concatenate/complexFilter 已在 `packages/ffmpeg/src/browserTypes.ts` 且 `ffmpeg`/`ffprobe` 工具已安装。范围改为 render manifest + 确定性 worker + 字幕产物（SRT/ASS/VTT）+ 规格门禁；前置增加「实测本机 FFmpeg 是否含 libass/subtitles 与目标编码器」 |
| P3-005 60 秒单集本地优先 MVP 施工蓝图 | `REVISE` | 前置加入 ADD-001（契约固化）；明确动态漫画优先、`director3dNode` 可用于运镜与关键帧导出 |
| P4-001 持久化 DAG 队列与任务租约 | `KEEP` | 核心缺口，无重叠 |
| P4-002 多项目/多集生产看板 | `REVISE` | 先交付 ADD-001 的只读索引器，看板基于索引构建，避免重复实现查询层 |
| P4-003 并发、配额与供应商熔断策略 | `REVISE` | 增加 `studio-media` 全局资源锁与 §8.2 状态机对接；明确 Toonflow 只做租约客户端 |
| P5-001 自动技术 QC 与人工质量门 | `REVISE` | 复用 `frameAnalysisNode`（本地切镜，无 Python/视觉模型/API）与 `ffprobe`；一致性检查对齐 PART A A8/A11；人工门状态枚举对齐 skill |
| P5-002 成本账本与预算控制 | `REVISE` + `MERGE` | 扩展 `multiTenantBillingLedger` 范式（已有 requestId 幂等、reserved/settled/released、requestHash、usage、micros 金额），不新建结构；个人模式成本流水拆为 ADD-005 |
| P5-003 素材生命周期、授权与审计 | `KEEP` + `MERGE` | 许可/授权台账部分与 P1-004 合并 |
| P6-001 容量、故障与恢复演练 | `KEEP` | — |
| P6-002 发布门禁与商业上线评审 | `REVISE` | 区分**站内发布（已实现，`publishWorkDialog` + `channel/community`）**与**外部平台发布（缺失）**；补 AI 披露与权利检查留痕要求 |

### 13.2 建议取消的重复候选（来自报告，不在 `ToDo.md` 内）

| 候选 | 判定 | 理由 |
| --- | --- | --- |
| `CAND-SERIES-001` 连续剧领域 ADR | `REMOVE_CANDIDATE` → 并入 P1-002 | 与 P1-002 同一产物 |
| `CAND-SERIES-002` StoryBible/EpisodeCard/ShotSpec/AssetVersion/Review/Render schema | `REMOVE_CANDIDATE` → 并入 ADD-001 | 应从既有 skill 反抽，不是新设计 |
| `CAND-SERIES-003` 连续性规则目录与冲突/豁免格式 | `REMOVE_CANDIDATE` → 并入 P3-002 | PART A A0.1/A8/A11 已是规则目录 |
| `CAND-SERIES-005` 资产版本/参考图/声音权利元数据合同 | `REMOVE_CANDIDATE` → 并入 P1-004 + P2-004 | 权利台账重复；版本快照规则已在 `stateAndDelivery.md` |
| `CAND-SERIES-004/006/007/008/009/010/011/012` | `MERGE` | 分别映射到 P2-001、P2-003/P2-006/P2-007、P3-001、P3-003、P3-004、P4-001/P4-003、P3-002、P5-002/P5-003，不新建并行任务 |
| `CAND-RESOURCE-001～005` | `KEEP`（映射） | 001→P0-004 前置只读盘点；002→P0-004 ADR；003→P4-001；004→P1-003；005→P2-006/P2-007 |

### 13.3 建议新增任务（`ADD_CANDIDATE`）

| 建议 ID | 优先级 | 目标与独立验收 | 依赖 |
| --- | --- | --- | --- |
| ADD-001 契约固化（MVP-0） | P1 | 从 `v118workflow` + `workflow` PART A + `stateAndDelivery.md` 反向抽取 SeriesBible/EpisodeCard/ShotSpec/AssetVersion/StageRecord/RenderManifest 的 JSON Schema，实现校验器与**只读索引器**（集号→状态/产物路径/版本/依赖）。验收：一个手工项目通过校验并被索引，不写入任何数据库 | P0-003、P1-002 |
| ADD-002 Agent 无头媒体工具 | P2 | 新增插件工具消费已注入的 `context.media`（`listModels/generateImage/generateVideo/generateAudio`），复用 `generateMedia` 与 `tasks.ts`；不新建媒体 API。验收：无浏览器条件下完成提交→查询→取消→恢复，产物含输入哈希与任务 ID | P0-002、P2-005 |
| ADD-003 Studio 在途可观测性与排空协议 | P0/P1 | 建立「全客户端在途文本请求可计数」的能力（Studio 侧计数或统一 ingress），作为任何模型切换的前置。验收：能在不重启服务的前提下读出当前在途请求数与空闲判定；缺失时明确 `BLOCKED` | P1-001、P0-004 |
| ADD-004 版本快照与确认绑定 | P1/P3 | 实现 `history/<集号>/<时间戳>-<hash>.md` 快照写入与「确认绑定快照路径」的台账字段（复用 `files/copy.ts` 语义）。验收：确认后原文被修改，原确认仍指向旧快照且可展示差异 | ADD-001 |
| ADD-005 个人模式成本流水 | P2/P5 | 在 `modelGateway.enabled()===false` 时记录估算/实付、provider、模型、镜头标识与用量。验收：单机制作一集后可导出成本明细，区分估算与实付 | P5-002、ADD-001 |
| ADD-006 资产 canonical ID 与输入哈希复用索引 | P3 | 以 `providerId+modelId+prompt+参考文件哈希+关键参数` 为键建立产物索引，命中即复用不重复计费；为角色/场景/服装/声音分配稳定 ID 并登记使用集数。验收：重复请求命中缓存并跳过生成 | P3-001、ADD-001 |
| ADD-007 字幕产物与音画同步校验 | P3 | 离线 ASR/强制对齐候选评估 + SRT/ASS/VTT 产物 + 时码比对检查；明确禁用浏览器 ASR 作为生产路径。验收：一段带台词样片可产出字幕并通过时码偏差检查 | P2-004、P3-004 |
| ADD-008 一键保存与会话导出（低优先，可选） | P5 | 对话区「保存为工作区文件」动作（内部调用既有 `files.write`）+ 会话 JSONL→Markdown 导出。验收：一条回复可存为指定路径文件并在文档面板打开；一个会话可导出为 Markdown | 无（独立小功能） |

### 13.4 建议后的阶段骨架（供批准参考）

```
P0  安全与数据基线（P0-001/002/003 + P0-004 修订 + ADD-003）
P1  可观测与契约（P1-001 修订 + P1-002 修订 + ADD-001 + ADD-004 + P1-003/004）
P2  受控集成（P2-001/003/006/007 + P2-002/004/005 修订 + ADD-002 + ADD-005）
P3  单集可审核流水线（P3-001 合并 + P3-002/003/004/005 修订 + ADD-006 + ADD-007）
P4  批量与恢复（P4-001 + P4-002/003 修订）
P5  质量、成本与运营（P5-001/002/003 修订 + ADD-008）
P6  压测与上线（P6-001 + P6-002 修订）
```

## 14. 后续技术验证优先顺序

| 序 | 验证项 | 类型 | 为什么排在这里 |
| --- | --- | --- | --- |
| V1 | 读取本机实际配置：已启用的 provider/模型清单、`workspace` 工具是否 `readOnly`、`disabledSkills` 内容、`agentNodeTools` 闸门（**不导出密钥**） | `UNVERIFIED` → 只读盘点 | 决定「已有能力」中哪些真的可用；是其余一切验证的前提 |
| V2 | 用已配置的云端图片 provider 跑通一次单图生成（提交→查询→取消→恢复→产物落盘） | `REQUIRES_DEVICE_TEST` | 验证 L3 端到端与 `tasks.ts` 恢复语义；成本低、风险小 |
| V3 | 用已配置的云端视频 provider 跑通一次 5 秒 I2V（含首帧引用与 `referenceOrder`） | `REQUIRES_DEVICE_TEST` | 验证 L4；同时检验 Seedance 路由与 PART A 契约是否可落地 |
| V4 | 实测本机 FFmpeg 能力：`availableEncoders`/`availableFilters`，确认 `concat`、`subtitles`/`ass`（libass）、目标编码与响度滤镜 | `REQUIRES_DEVICE_TEST` | 决定 P3-004 修订后的可行范围；不需要任何模型 |
| V5 | 验证无头路径：临时以脚本直接调用 `POST /api/ai/media/generate`（不经浏览器画布），确认服务端可独立完成生成与落盘 | `REQUIRES_DEVICE_TEST` | 为 ADD-002 提供事实基础；证明批量不依赖浏览器 |
| V6 | 音频链路盘点：确认 3 家已安装 provider 是否声明 audio model 与 `voices`；若无，进入 TTS 候选隔离评估 | `UNVERIFIED` → `REQUIRES_DEVICE_TEST` | 纠正「音频已可用」的误判；TTS 是配音与字幕同步的前置 |
| V7 | Qwen 结构化输出验收：固定样本下 JSON Schema 合规率、修复率、长上下文与取消行为 | `REQUIRES_DEVICE_TEST` | 剧本/分镜/连续性审稿全部依赖结构化输出；不改 MLX 参数 |
| V8 | Studio 在途可观测性方案验证（ADD-003） | `RESEARCHED` → 需基础设施审批 | 未通过则**永久禁止**自动模型切换 |
| V9 | 本地图片隔离基准（FLUX.1 schnell / 原版 Qwen-Image，ComfyUI sidecar，独立端口、Qwen 空闲窗口） | `REQUIRES_DEVICE_TEST` | 只有在 V8 通过后才允许 |
| V10 | 本地视频隔离基准（LTX 2.5 Fast 首选，Wan2.2 TI2V-5B 次选） | `REQUIRES_DEVICE_TEST` | 风险最高、收益不确定，排在最后 |

## 15. 交付状态

- 现有能力清单：§1（含本机运行时安装状态）
- 已有但未配置成功：§2
- 真正缺失：§3（M1–M11）
- 可复用 / 需扩展：§4
- 应取消的重复开发：§5、§13.2
- 图片/视频/音频选型与未验证问题：§10
- Studio 模型互斥调度：§8
- 30–100 集推荐架构：§11
- 原报告修订记录：§12
- `ToDo.md` 调整建议：§13（**未写入文件**）
- 验证优先顺序：§14

`SOURCE_AUDIT_COMPLETE`

`REPORTS_RECONCILED`

`NO_MODEL_OR_SERVICE_CHANGE_PERFORMED`

`IMPLEMENTATION_NOT_STARTED`
