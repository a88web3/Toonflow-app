# Toonflow 工作区既有能力审查

> 审查日期：2026-11-15。范围：只读源码追踪与公开的本地音频候选资料；没有调用 `/ai/media/generate`、没有读取用户 provider 配置/API Key、没有启动本地模型或修改服务。结论描述**当前源码能力**，不是已在本环境成功生成的证据。

## 1. 证据状态与总览

| 状态 | 定义 |
| --- | --- |
| `IMPLEMENTED` | 前端、服务端和持久化路径均在源码中存在。实际模型/账号可用性仍需另验。 |
| `PARTIAL` | 有通用机制或部分路径，但生产需求/端到端能力不完整。 |
| `UI_ONLY` | 找到界面或交互，但没有相应的执行/持久化证据。 |
| `NOT_IMPLEMENTED` | 在审查范围内未找到实现。 |
| `UNVERIFIED` | 源码无法证明当前本机配置、外部 provider、模型或运行效果。 |

**总判断**：用户观察正确。图片、视频、音频节点和右侧 AI 对话不是凭空 UI；其中图片/视频的云端生成和单任务恢复链路已实现，画布和聊天均会落盘。它们可复用为连续剧平台的创作/单镜头执行层。不过，现有实现并不等于已有连续剧生产系统：没有 Series/Episode/Shot 领域记录、跨集资产版本、审核状态机、持久 DAG、批量配额或自动成片。

## 2. 图片、视频、音频节点

### 2.1 共用实现路径

```text
画布节点（Vue）
  → useNodeGeneration：先把 mediaTask 保存到画布 JSON
  → nodeAi.generate{Image|Video|Audio}
  → POST /api/ai/media/generate
  → media generation：加载已配置 provider，调用 generate* / resumeTask
  → 下载并写入工作区 assets/<nodeId>/
  → 节点 outputs 指向相对文件路径；画布防抖保存
```

| 项目 | 状态 | 源码证据 | 实际含义与限制 |
| --- | --- | --- | --- |
| 图/视频/音频节点前端 | `IMPLEMENTED` | `packages/nodes/imageGenerationNode/src/index.vue`、`videoGenerationNode/src/index.vue`、`audioGenerationNode/src/index.vue` | 三者都有模型选择、提示词、输入引用、输出端口、上传替换和生成状态。 |
| 统一媒体 API | `IMPLEMENTED` | `apps/server/src/routes/ai/media/generate.ts` | 通过 `POST /api/ai/media/generate` 校验 `image/video/audio` 请求，支持已有 `taskId` 或同步等待结果。 |
| provider 合同 | `IMPLEMENTED` | `packages/providers/types.d.ts`：`generateImage`、`generateVideo`、`generateAudio`、`resumeTask` | 是可扩展接口；声明接口不代表每个 provider 实现三种能力。 |
| 本地模型接入 | `PARTIAL` | `apps/server/src/routes/providers/media/{add,save,validate}.ts`、`apps/server/src/utils/media/provider.ts` | 可添加/校验自定义 provider 源码并声明媒体模型，因此理论可接本地 HTTP/ComfyUI sidecar；仓库未内置 ComfyUI、MLX 图片、视频或 TTS provider。现有 Qwen 文本 provider 不会自动成为媒体 provider。 |
| 模型清单 | `PARTIAL` | `apps/server/src/utils/media/generation.ts#listMediaModels` | 运行时只列出**已安装且已配置**的 provider；本审查未读取设置，所以当前 UI 实际显示什么模型为 `UNVERIFIED`。 |

### 2.2 输入、连接与输出

| 节点 | 端口/输入 | 可配置请求 | 输出/连接 | 状态 |
| --- | --- | --- | --- |
| 图片生成 | 文本提示词；图片引用；节点可从画布的 `IMAGE`/`STRING` 引用取值 | provider、model、prompt、ratio、size、`other`；服务端最大 64 张图片引用 | 一个 `IMAGE` source，写入 `assets/<nodeId>/image<taskId><index>.<ext>` | `IMPLEMENTED` |
| 视频生成 | 文本、图片、视频、音频；可按模型选择 text、单图、首/尾帧或多参考模式 | provider、model、duration、resolution、ratio、mode、`generateAudio`、`referenceOrder` | 一个 `VIDEO` source，写入 `assets/<nodeId>/video<taskId><index>.<ext>` | `IMPLEMENTED` |
| 音频生成 | 文本；最多一个图片或音频引用，不能同时；输入允许 `STRING` | provider、model、prompt、language、sampleRate、format、`other` | 一个 `AUDIO` source，写入 `assets/<nodeId>/audio<taskId><index>.<ext>` | `IMPLEMENTED` 的抽象节点 |

证据：前端各节点的 `generationInput` 使用 `outputDirectory: assets/${id}`；请求 schema 见 `apps/server/src/utils/media/schemas.ts`；写盘规则见 `apps/server/src/utils/media/generation.ts`。视频节点按 provider 模型声明启用图片/视频/音频引用、时长、分辨率和可选原生音频，不能把通用 UI 控件当成每个模型真实支持的保证。

### 2.3 现有 provider 与模型

源码内置媒体 provider 为 `apiMart`、`metaso`、`compshare`，注册点为 `packages/providers/index.ts`。它们是**云端 API 适配器**，使用前需要在设置中安装/配置并提供有效密钥；本机是否已配置、各 API 是否仍可用、价格和商业条款均为 `UNVERIFIED`。

| provider | 源码声明能力（非运行验收） | 状态 |
| --- | --- | --- |
| `apiMart` | 视频：Seedance 2.x、Wan3.0、MiniMax-H3；图片：Seedream 5.0、GPT-Image-2、Gemini 预览模型等 | `IMPLEMENTED` 的 provider 源码；实际账号/模型为 `UNVERIFIED`。路径：`packages/providers/src/media/apiMart.ts`。 |
| `metaso` | MiniMax-H3 视频，含轮询和 `resumeTask` 记录 | `IMPLEMENTED` 的云视频适配；路径：`packages/providers/src/media/metaso.ts`。 |
| `compshare` | MiniMax/Seedance 视频；Seedream、Qwen Image、GPT Image 等图片；具体清单位于 provider `models` | `IMPLEMENTED` 的云适配；路径：`packages/providers/src/media/compshare.ts`。 |
| 内置音频 provider/model | 未发现三家内置 provider 声明 `type: "audio"` 或 `generateAudio` | `NOT_IMPLEMENTED`。音频节点会显示“请先在设置中添加音频模型”。 |
| 自定义音频 provider | 可通过 ProviderDefinition 实现 `generateAudio` 并声明 `voices`/规则 | `PARTIAL`；契约存在，未有内置样例/本机实现。 |

模型供应商代码可变化，不能将以上字符串当作长期产品目录或商业许可证明。生产应使用锁定 revision、能力合同和许可台账。

### 2.4 异步、重试、取消与文件持久化

| 能力 | 状态 | 证据 | 重要限制 |
| --- | --- | --- | --- |
| 节点先持久化任务身份 | `IMPLEMENTED` | `packages/nodeShared/src/useNodeGeneration.ts` 先将 `node.data.mediaTask` 保存，随后提交 | 画布保存失败时不会提交，避免无可追踪任务。 |
| 服务端任务检查点 | `IMPLEMENTED` | `apps/server/src/utils/media/tasks.ts` | 按工作区 hash 保存 JSON，包含 request、状态、providerData、结果 hash；大 base64 引用另存私有 blob。 |
| provider 异步恢复 | `PARTIAL` | `generation.ts` 在有 `resumeTask` 与 `providerData` 时续查；`metaso.ts` 有具体轮询/恢复 | 同步 provider、没有 provider checkpoint 或远端无恢复语义时，服务中断后会标为 `interrupted`，不能自动重提。 |
| 取消 | `PARTIAL` | `/api/ai/media/tasks/cancel`、`mediaTasks.cancel`、节点 `cancelGeneration` | 本地记录先设为 cancelled 并 abort 等待；provider 合同**没有**通用 remote-cancel 方法，不能承诺已停止远端或不计费。 |
| 自动失败重试 | `PARTIAL` | 状态查询会尝试恢复可恢复 running/interrupted 任务 | 没有跨镜头重试策略、最大尝试账本、退避/优先级/DAG。用户须复查或重新生成。 |
| 输出自动落盘/可恢复 | `IMPLEMENTED` | `generation.ts` 下载/验证媒体，原子写工作区；`useNodeGeneration` 写 outputs 并保存画布 | 支持媒体大小上限 100MB；节点输出只保存当前选择，图/视频节点历史来自同一节点目录。 |
| 图片/视频历史选择 | `IMPLEMENTED` | `packages/nodeShared/src/components/mediaHistory.vue` | 只枚举 `assets/<nodeId>` 下的图片或视频，不是全项目资产版本库。音频没有使用该 history 组件。 |

任务记录不在工作区而位于 `dirname(conf.path)/mediaTasks/workspace<sha256>/task<uuid>.json`，权限创建为私有模式；成品媒体在工作区。故仅复制画布 JSON 不等于带走成品和恢复检查点；使用“导出项目”才会递归 ZIP 工作区内文件，且当前上限 1GB（`apps/server/src/routes/workspaces/files/export.ts`）。

## 3. 右侧 AI 对话、剧本与分镜

| 问题 | 结论 | 状态 | 证据 |
| --- | --- | --- | --- |
| AI 对话是否真实调用模型 | 会话前端发送 `/api/agent`，模型/会话由 server runtime 处理；可选带当前 Canvas Context | `IMPLEMENTED`，当前 Qwen 调用成功与否仍 `UNVERIFIED` | `apps/web/src/components/agent/conversation.vue`、`apps/server/src/agent/runtime/index.ts`。 |
| 聊天记录存在哪里 | 每个工作区 `.agent/sessions/<uuid>.jsonl`，包含 session header、用户/助手消息、工具结果、分支/队列相关 custom entries | `IMPLEMENTED` | `apps/server/src/agent/runtime/sessions.ts#createAgentConversation/getAgentSession`，路由 `routes/agent/{list,get,create}.ts`。 |
| 是否自动保存、刷新恢复 | 会话写入 JSONL；列表/获取路由从磁盘重建对话，UI 会加载历史 | `IMPLEMENTED` | 同上；前端 `apps/web/src/components/agent/index.vue`。正在运行进程内的 Canvas 回调通道不跨 server 重启，见 `agent/bridge/canvas.ts`。 |
| AI 剧本/分镜保存在哪里 | 仅以聊天回复生成时：保存在会话 JSONL；若 Agent 通过 canvas/file 工具写入节点或 Markdown，另在其目标画布/工作区文件 | `PARTIAL` | 没有专用 `Script`/`Storyboard` 实体或自动提取器；`packages/tools/canvas/src/runtime.ts` 提供通用 addNode/nodeTools。 |
| 可将回复自动转为剧本/分镜节点吗 | 有 Canvas Agent 工具可在**当前画布已开放且模型主动调用工具时**新建/连接节点；未找到 UI 上“选中回复→转剧本/分镜节点”的确定性功能 | `PARTIAL` | `apps/server/src/agent/bridge/canvas.ts`、`packages/tools/canvas/src/index.ts`/`runtime.ts`。 |
| 导出 Markdown | 文档编辑器可复制 Markdown（`packages/ext/md/src/index.vue`）；**未发现聊天会话 JSONL→Markdown 的导出路由/按钮** | 聊天：`NOT_IMPLEMENTED`；工作区整体 ZIP：`IMPLEMENTED` | `conversation.vue` 只调用 clipboard copy；`workspaces/files/export.ts` 是 ZIP。 |
| 复制/分享 | 单条消息和工具参数/结果可复制到系统剪贴板；聊天有新建、重命名、删除、分支和队列编辑。未找到聊天公开分享链接或协作分享实现 | 复制/会话管理：`IMPLEMENTED`；公开分享：`NOT_IMPLEMENTED` | `conversation.vue#copyMessage`、`conversation/messageList.vue`、`agent/{branch,message,rename}.ts`。 |
| 自带“分镜团队”是否直接生产 | `storyboardTeam` 是只读、文本规划示例，README 明确不自动写工作区或执行媒体生成 | `PARTIAL` | `packages/teams/storyboardTeam/readme.md`。 |

因此，右侧对话非常适合做剧本/分镜**草案与画布操作入口**，但不应被误判为带版本/审批/连续性约束的剧集管理器。

## 4. 音频生成专项

### 4.1 当前 Toonflow 能力边界

| 需求 | 当前结论 | 状态 |
| --- | --- | --- |
| 中文 TTS | 节点提供 `language=zh-CN` 等 UI 参数，但内置 provider 没有 audio model；当前是否已安装中文 TTS provider 不读取配置，故不能确认 | `PARTIAL` / `UNVERIFIED` |
| 固定多角色音色 | Provider 合同有 `voice` 和模型 `voices` 字段，但音频节点当前 UI 未暴露独立 voice selector，内置音频样例也缺失 | `PARTIAL` |
| 情绪配音 | 合同 `AudioRequest` 具备 `emotion`，但 API schema 和当前节点 UI 没有该字段；必须自定义 provider 的 `other`/未来扩展 | `PARTIAL` |
| 旁白 | 可作为文本输入提交给将来的 TTS provider | `PARTIAL` |
| 音效/BGM | 音频节点 prompt 可抽象支持；没有内置音效/音乐 provider、授权资产库或时间轴混音工作流 | `PARTIAL` |
| ASR 转文本 | 音频节点提供转文本操作，调用共享 ASR 能力；并不是 TTS | `IMPLEMENTED` 的节点功能；本地/云端引擎仍 `UNVERIFIED` |
| 多轨混音/角色台词时间轴 | 未找到 production timeline/mixer/domain model | `NOT_IMPLEMENTED` |

音频节点支持语言、采样率、WAV/MP3/PCM/Opus 格式、一个图片/音频参考和节点输出；见 `packages/nodes/audioGenerationNode/src/{index.vue,audioSettings.ts}`。它是良好的未来 sidecar 入口，但不能替代角色声音档案、同意管理、情绪标注、台词时间轴和音画合成。

### 4.2 Apple Silicon 本地 TTS 候选（资料研究，不是安装验证）

| 候选 | 可复用价值与许可线索 | Apple Silicon / 内存结论 | Toonflow 接入结论 |
| --- | --- | --- | --- |
| [CosyVoice](https://github.com/QwenAudio/CosyVoice) | 官方仓库 Apache-2.0，面向多语言语音生成；权重、参考声音、训练数据和声纹授权仍须逐项复核 | 未见本项目对 M5 Max 的实测；可作为独立 TTS sidecar 候选，内存/中文表现/克隆权利均 `UNVERIFIED` | 实现自定义 `generateAudio` provider，返回音频文件和可选 voice capability。 |
| [Kokoro-FastAPI](https://github.com/remsky/Kokoro-FastAPI) | 仓库称 Apache-2.0、OpenAI-compatible speech endpoint、多说话人/时间戳能力；需锁定模型卡/版本 | 仓库声称原生 MPS Apple Silicon 路径；本机速度、中文表现、内存未验证 | API 形态适合独立 sidecar；先核验其模型权重与音色来源。 |
| [MeloTTS](https://github.com/myshell-ai/MeloTTS) | 源码 MIT，仓库说明支持中文；模型权重及依赖仍要逐项核验 | 以 CPU/通用 PyTorch 为候选，不等于 MPS 或 M5 实测 | 适合较轻量旁白试验；不应先承诺情绪克隆/多角色。 |
| Piper | 源码 MIT，但原仓库已归档；声音模型许可按 voice 单独变化 | 可作为轻量离线候选，但中文音质/voice 条款/Mac 部署未验证 | 不建议作为首个生产中文多角色方案。 |

**资源策略**：TTS 也使用 Studio 统一内存，不能默认和 Qwen/图片/视频并发。若选轻量 CPU 或 MPS TTS，也应被编排为 `audio-light` 资源组并先实测；高质量/克隆 TTS 则与 `media-heavy` 互斥。现有 Qwen :8000 保持不变，TTS 不得占用该端口。参考音色必须有可审计授权和撤销流程，模型代码的 Apache/MIT 许可不授予真人声纹、演员表演、音乐或训练素材权利。

## 5. 工作区、文件系统、浏览器存储与数据库

### 5.1 实际保存位置

| 数据 | 主保存位置 | 恢复机制 | 状态 |
| --- | --- | --- | --- |
| 工作区/项目目录 | 由用户选定的本机绝对目录；服务端验证并限制在许可工作区 | `resolveWorkspace` + 文件 API | `IMPLEMENTED`；实际目录取决于当前用户选择。 |
| 画布、节点、连线、节点 outputs、mediaTask | `<workspace>/<canvas>.json`，根标记 `toonflowCanvas: true` | 打开画布时读 JSON；编辑后 500ms 防抖写入，节点任务先强制保存 | `IMPLEMENTED`：`apps/web/src/pages/workspace/panels/canvas/index.vue`、`canvasFiles.ts`。 |
| 节点媒体文件 | `<workspace>/assets/<nodeId>/` | 画布 output 的相对 URL 指向文件；图片/视频可从同目录 history 重新选取 | `IMPLEMENTED`。 |
| 聊天图片下载缓存 | `<workspace>/assets/chatImages/<sha256>.<ext>` | URL hash 去重；会话附件指向工作区路径 | `IMPLEMENTED`：`apps/server/src/utils/workspace/chatImages.ts`。 |
| 聊天会话/草案 | `<workspace>/.agent/sessions/*.jsonl` | `list/get` 重建历史，支持分支/删除/重命名 | `IMPLEMENTED`。 |
| 媒体任务 checkpoint | 服务端配置目录旁 `mediaTasks/workspace<hash>/task<uuid>.json`，并可能有私有 media blob | status 时可恢复有 providerData 的任务；不属于工作区 ZIP | `IMPLEMENTED`，但不是生产级跨进程队列。 |
| 全局素材库 | 服务端 `/api/assets/*` 管理的独立资产目录 | 仅 UI 保存/选择；与工作区素材不同 | `IMPLEMENTED`，其真实根路径依赖运行配置，未读取。 |
| 多租户账户/账本 | 可选 multi-tenant SQLite/其他支持数据库 | 与工作区剧情/媒体资产不是同一生产数据库 | `PARTIAL`；不应误作剧集资产库。 |
| 浏览器存储 | localStorage 用于设置/开发者开关/新手引导；IndexedDB 用于 node clipboard | 不是画布、聊天或媒体真相源 | `IMPLEMENTED` 的辅助 UI 状态。 |

### 5.2 可复用与必须扩展

**可直接复用**：画布文件、节点引用/类型校验、`assets/<nodeId>` 成品存储、统一 media provider 合同、任务 checkpoint/resume、聊天 JSONL、canvas agent 工具、工作区 ZIP 导出、FFmpeg/音频处理基础。

**必须扩展**：

1. Series/Season/Episode/Scene/Shot、角色/地点/服装/声音的独立真相源和版本关系。
2. 从聊天草案到已批准 Script/Storyboard 的显式保存、Markdown 导出和可审计转换动作。
3. 跨节点/跨集资产 catalog，不能把 node ID 目录当长期角色库。
4. durable DAG、租约、远端取消语义、重试预算、批量并发和服务重启恢复。
5. 音色权利、角色声音档案、情绪/台词时间轴、字幕/混音/成片 render manifest。
6. 生产数据库、备份、保留/清理、成本/许可证/审核记录。

## 6. 对既有总体架构的影响

不应重写现有图片、视频、音频节点或 provider 基础。推荐让连续剧领域层生成受审核的 `ShotSpec`，再把每个批准镜头转换为当前节点可理解的请求/引用；现有节点仍承担单媒体的选择、生成、历史和手动编辑。新的 orchestration 层应调用同一媒体合同，却不能依赖 Canvas JSON 或 `.agent` JSONL 承担任务队列真相源。

本审查修正了先前“媒体能力全缺”的误解：缺的是生产领域、批量与治理，不是基础媒体 UI/API。图片/视频已有可复用实施面；音频节点已有接口但缺经批准的具体 TTS provider；本地图片/视频/音频仍需要独立 sidecar 和设备验证。

## 7. 后续验证关卡（不构成施工授权）

| 关卡 | 要验证的事实 | 当前状态 |
| --- | --- | --- |
| W-1 | 当前设置实际安装/启用的 provider、模型和 capability catalog（不暴露密钥） | `UNVERIFIED` |
| W-2 | 每个已配置图片/视频 provider 的真实提交、轮询、取消、恢复和下载 | `UNVERIFIED` |
| W-3 | 自定义 local media provider 能否仅以 sidecar API 接入，不碰 Qwen :8000 | `UNVERIFIED` |
| W-4 | 一个具体中文 TTS 的音色、情绪、时间戳、许可证、内存和恢复能力 | `UNVERIFIED` |
| W-5 | 从已批准 Script/ShotSpec 自动创建/回写节点的可审计 UX | `NOT_IMPLEMENTED` |

## 8. 对账补充（CodexQ 复核，2026-11-15）

本节补入首轮审查遗漏的既有能力，均为 `SOURCE_VERIFIED`（含本机运行时目录盘点，未读取 `data/settings.json`）。完整对账见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md)。

### 8.1 已安装的生产 playbook（最重要的遗漏）

`data/skills/` 实际安装 `canvas`、`v118workflow`、`workflow`（另有 `initialized` 标记）；`data/agents/` **目录不存在**（连 `initialized` 标记都没有，说明团队包从未安装过），故 `storyboardTeam` **未安装**。

| Skill | 已定义的生产契约 | 证据 |
| --- | --- | --- |
| `v118workflow`（v1.1.0「v1.1.8 分集制作流程」） | 目录契约 `画布/剧本.json`、`画布/资产.json`、`画布/分集/第NNN集.json`；三画布职责划分；阶段流程（项目初始化→来源与章节事件→故事骨架→改编策略→逐集剧本→基础资产→衍生资产→导演规划→分镜表→分镜图→视频→交付）；每阶段「读已确认输入→完成→保存→回读检查→展示→`askUser` 确认」；阶段记录状态枚举 `草稿/待确认/已确认/生成中/失败/待复核/已跳过`；续作与上游改动的下游影响标记 | `packages/skills/v118workflow/SKILL.md` + 16 个 `references/*` + 7 个 `references/knowledge/*` |
| `workflow`（v2.7.0「视频制作工作流」） | 内嵌 **SEEDANCE EPISODE DIRECTOR PART A（A0–A12）**：A0.1 剧情保真锁（事实来源优先级、五类绝对禁止改动、不补前史、模糊保持模糊）、A1.2.1 参考图绑定、A2 全量节拍解析与源剧情账本、A3 分段与时长预算、A4 资产与出镜状态、A5 站位、A6 表演层、A7 运镜与段间衔接、A8 跨段连续性检查、A9 分段提示词模板（2.0/2.5 路由）、A10 全片输出结构、A11 结构校验、A12 交付前优化 | `packages/skills/workflow/SKILL.md`（A0:75、A2:270、A8:714、A9:782、A11:1037、A12:1144） |
| `workflow` 状态与交付 | **制作状态台账** `productionState.md`（多集各自子目录）；四张表：成果 / 生成尝试（提示词快照、模型与完整参数、参考版本、授权范围与剩余次数、返回文件）/ 片段采用 / 交付；**确认必须绑定独立历史快照**；生成前保存提示词+参数+引用+授权快照且后续编辑不回写 | `packages/skills/workflow/references/stateAndDelivery.md` |

**结论修订**：§3 中「AI 剧本/分镜保存」判为 `PARTIAL`、「无专用 Script/Storyboard 实体」仍成立，但**必须补充**：分集结构、连续性规则、阶段确认门与生成留痕**已由 skill 契约定义**，缺的是服务端强制（schema 校验、状态机、幂等、租约），不是缺方法论。新工作应从这些文件反向抽取 schema。

### 8.2 Agent 工作区文件工具（§3 判断需修订）

`packages/tools/workspace/src/index.ts:13-35` 注册 `read`、`ls`、`grep`、`glob`、**`write`**、**`edit`**、`downloadImage`（`readOnly` 为 false 时）；本机已安装 `data/tools/workspace.tool.js`。文件上下文实现见 `apps/server/src/agent/tools/index.ts:73-124`（`readFile/writeFile/mkdir/rename/remove/copyFile/importImage`，全部经 `resolveWorkspacePath` 与 `protectWorkspaceRoot` 边界）。

因此「将 AI 回复保存为正式剧本」**已可实现**（Agent 直写工作区 Markdown/JSON），§3 表中「聊天→Markdown 导出 `NOT_IMPLEMENTED`」应限定为「**无专用 UI 按钮与导出路由**」：`conversation.vue:296` 仅有 `copyMessage`，但 Agent 写文件路径是通的。当前 `workspace` 工具是否处于 `readOnly` 为 `UNVERIFIED`（未读设置）。

### 8.3 文档工作区（§5.1 需补充）

`apps/web/src/pages/workspace/panels/document/` 提供完整文档工作区：`components/fileTree.vue`（文件树）、`components/quickOpen.vue`（快速打开）、`components/workspaceSearch.vue`（全工作区搜索）、标签页与分屏（`documentSession.ts:197` 的 `TabAction`）。保存走 `documentSession.ts:72` 的 `files.writeTextSnapshot(path, text, disk)`，**带 revision 与磁盘冲突检测**。配合 `packages/ext/{md,txt,image,media,canvasNode}`（本机已安装于 `data/ext/`），已保存的剧本可直接打开、继续编辑并在刷新后恢复。

### 8.4 `director3dNode`（§2 未覆盖的现成能力）

`packages/nodes/director3dNode/`：文字生成场景/模型动画/运镜，内置低多边形关节人偶；**关键帧可导出为 PNG 图片节点、选中动画可导出为 MP4 视频节点**（`src/renderMedia.ts:5 renderImage`、`:22 renderVideo`）。AI 局部编辑经 `ai.generate()` 与私有 `readDocument`/`editDocument`，每轮最多 40 次模型调用、10 分钟超时，工具**不注册到 `nodeTools`**。

**关键约束**：`renderVideo` 使用浏览器 `MediaRecorder`（`video/mp4;codecs=avc1.420028`、6 Mbps、~30fps），readme 明确「视频导出时保持当前窗口可见」→ **浏览器绑定，不可无人值守批量**。可用于动态漫画的运镜预演与关键帧产出。

### 8.5 FFmpeg 合成能力（§4/§5 判断需修订）

`packages/ffmpeg/src/browserTypes.ts:21-24` 的 `runMethods` 含 `concat`、`mergeToFile`、`concatenate`、`takeScreenshots`；`chainMethods`（`:3-19`）含 `complexFilter`、`filterGraph`、`audioFilter`、`videoFilter`、`inputOptions`、`outputOptions`、`withSize`、`fps` 等。Agent 侧 `ffmpeg` 与 `ffprobe` 工具已安装（`packages/tools/ffmpeg/src/index.ts:59,68`，`data/tools/ffmpeg.tool.js`），经宿主 worker 执行且**不依赖浏览器**。

因此「无 concat / 无合成能力」的表述**不成立**；准确表述是：**拼接与复杂滤镜在工具层已可达，缺 render manifest、确定性 worker、字幕产物（SRT/ASS/VTT）、响度与母版规格门禁**。本机 FFmpeg 二进制是否启用 `libass`/`subtitles` 滤镜与目标编码器为 `UNVERIFIED`。

### 8.6 无头媒体生成扩展点（§2.1「本地模型接入」需补充）

`apps/server/src/agent/tools/index.ts:146-151` 已把 `media: { listModels, generateImage, generateVideo, generateAudio }` 注入 `ToolContext`（类型 `packages/toolScaffold/src/runtime.ts:140-145 MediaContext`），其实现直接调用服务端 `generateMedia(cwd, ...)`（`utils/media/generation.ts:243`，默认落盘 `assets/generated`），**不需要浏览器**。但全仓库 grep 显示**当前无任何工具或节点消费 `context.media`**。

**含义**：让 Agent 无头批量生成媒体，只需新增一个消费 `context.media` 的薄插件工具，**不需要新建任何媒体 API**。这是批量生产的最小施工面。相对地，`node:generateImage` 路径经 `apps/server/src/agent/bridge/canvas.ts` 下发前端执行，120s 超时、要求画布打开、回传仅在进程存活期有效（`:10` 代码注释），**不适合无人值守**。

### 8.7 成本与计费（§5.1 需补充）

| 能力 | 状态 | 证据 |
| --- | --- | --- |
| 生成前价格预估、provider 余额/价格查询与 SSE | `IMPLEMENTED` | `utils/media/generation.ts:230 getMediaPrice`、`utils/media/{priceReference,billing}.ts`、`routes/ai/media/{price,balance,balanceEvents}.ts` |
| 文本 token 用量解析（三协议，含 cache read/write） | `IMPLEMENTED` | `utils/ai/billingUsage.ts:22-110` |
| 持久计费账本 | `IMPLEMENTED`（**仅多租户**） | `packages/multiTenant/db/sqlite.ts:102-118` / `mysql.ts:72`：`multiTenantBillingLedger(requestId PK, userId, currency, kind, status∈{reserved,settled,released}, requestHash, reservedMicros, amountMicros, details, usage, timestamps)` |
| 媒体任务与账单绑定 | `IMPLEMENTED`（仅多租户） | `utils/media/tasks.ts:44-45`（`chargeId`/`chargeStatus`）、`:296-305`（视频按实际秒数 `usage={requests,seconds}`）、`:346-349 reserveSnapshot`、`:414-421`（恢复时校验） |
| 个人/单机模式成本记录 | `NOT_IMPLEMENTED` | `agent/tools/index.ts:52-59`：`modelGateway.enabled()` 为 false 时直接 `generateMedia`，无 charge 记录 |

### 8.8 发布通道（§5.1 需补充）

`apps/web/src/pages/home/publishWorkDialog.vue`（作品 MP4/WebM + 封面 JPG/PNG/WebP + 项目）配合 `apps/server/src/routes/channel/community/{uploadExploreProject,getExploreProjects,getExploreProject,getExploreDownload,getExploreUploadSignature,getExploreProcess}.ts`，构成**站内（Toonflow 社区）发布**能力，`IMPLEMENTED`。外部平台（TikTok/YouTube）发布与 AI 披露留痕 `NOT_IMPLEMENTED`。

### 8.9 音频能力细化（§4.1 精修）

- HTTP 层 `audioGenerationSchema`（`apps/server/src/utils/media/schemas.ts:44-59`）**已支持** `voice`、`speed`、`volume`、`pitch`、`language`、`format`、`sampleRate`；`MediaModel.voices`（`packages/toolScaffold/src/runtime.ts:93`）与 provider 合同 `voices`（`packages/providers/types.d.ts:25`）均已定义。
- `emotion` **仅存在于 provider 合同** `AudioRequest`（`packages/providers/types.d.ts:175`），**不在 HTTP schema**，只能经 `other` 透传。
- `audioGenerationNode` 的 `data` 为 `{prompt, promptModel, model, language, sampleRate, format, other}`（`src/index.vue:102`），**无 voice 选择器**。
- 修订判断：多角色固定音色在 **API 层已可达**（直传 `voice`），缺的是节点 UI 与「已安装 provider 是否声明 audio model / voices」的配置事实（`UNVERIFIED`，未读设置）。

### 8.10 本机运行时状态与并发能力

- `data/providers/`：仅 `apiMart.ts`、`compshare.ts`、`metaso.ts`；**无任何本地/自定义媒体 provider**。
- `data/workspaces/`：`myTvProject`（**0 文件**）与 `画布1.json`；`data/assets/` 为空 → **本机无任何真实生产数据，无端到端跑通证据**。
- 全仓库 grep `inflight|concurren|semaphore|activeRequests|rateLimit` 在 `utils/ai`、`routes/ai`、`packages/multiTenant` **零命中**；唯一锁是进程内按路径的 `lockWorkspaceFiles`（`utils/workspace/files.ts:133`）。
- Toonflow **可**枚举/停止自己的 Agent 运行：`agent/runtime/sessions.ts:80-101`（`activeRuns` 请求进入即登记、`stopAgentRuns(cwd, runIds)` abort 并等待），但仅进程内有效，且**不覆盖**直接调用 `/api/ai/generate` 的文本请求。
- 结论：**批量调度、全局并发闸门、模型资源锁均为 `NOT_IMPLEMENTED`**；且无法判断其他项目是否正在使用 Qwen（详见对账报告 §8.1）。

`CAPABILITY_AUDIT_COMPLETE`

`NO_MODEL_OR_SERVICE_CHANGE_PERFORMED`

`IMPLEMENTATION_NOT_STARTED`
