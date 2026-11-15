# AI 连续短剧生产研究

> 研究日期：2026-11-15。范围：公开资料、既有 Toonflow 源码与前两轮审查；未安装模型、未运行媒体生成、未调用付费 API。本报告不是对模型效果、吞吐、许可证完整性的实测或法律意见。

## 1. 结论与证据标记

**产品目标**：不是“文本生成视频”按钮，而是能保存创作决策、复用资产、恢复中断、接受人工审核的连续剧生产系统。首个现实目标应是 60 秒竖屏动态漫剧；30–100 集只能在连续性、资产版本、可恢复调度和质量门都通过后逐级放大。

| 标记 | 含义 |
| --- | --- |
| `VERIFIED` | 源码、官方仓库/条款或可复核公开页面直接支持。 |
| `PARTIAL` | 已有基础能力，但不能满足生产合同。 |
| `UNVERIFIED` | 没有在本机或可信独立条件下验证；不能当作承诺。 |
| `PROPOSED` | 本报告提出的设计，不是现有功能。 |
| `CASE` | 单一公开案例/项目自报；不能外推为行业平均。 |

前两轮结论仍有效：Toonflow 已有画布、provider 抽象、媒体任务 JSON 检查点和单任务续跑；没有**服务端强制的**连续剧领域模型、持久 DAG、资产/版本账本、审查状态机和可审计成片流水线。详见 [生产审查](./TOONFLOW_PRODUCTION_AUDIT.md) 与 [本地媒体可行性](./LOCAL_AI_MEDIA_FEASIBILITY.md)。

> **第三轮对账修订（重要）**：本轮源码审查发现仓库**已内置且本机已安装**三个生产级 Agent skill——`v118workflow`（分集制作流程）、`workflow`（1406 行，内嵌 SEEDANCE EPISODE DIRECTOR **PART A：A0–A12**）、`canvas`。它们已经定义了分集目录契约、阶段确认关口、剧情保真锁、跨段连续性检查、结构校验、制作状态台账与版本快照绑定。因此本报告中「分层产物模型」「连续性三层检查」「资产合同」「生成留痕」等设计**与既有契约语义高度重叠**，其工作性质应由「设计新模型」修订为「**从既有 skill 反向抽取 schema 并固化为服务端校验**」。证据与逐项映射见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md) §1.5、§5、§12。

## 2. 真实生产方式：应自动化什么、保留什么人工决定

公开研究和行业案例的共同信号是：短剧链路会把剧本、分镜、关键帧、镜头视频和成片相连；上游错误会传递到下游，而不是由“再生成一次视频”自动消失。[DramaChain Bench](https://arxiv.org/abs/2609.00646) 把这条链显式建模；它是研究论文，结论仍应视作 `UNVERIFIED` 的生产性能主张。公开工作室案例通常由人工读剧本、制作资产、粗剪清除错误、精剪与终剪，而非无人值守生成。[浙江新闻案例](https://zjnews.zjol.com.cn/zjnews/202607/t20260726_31808771.shtml?v=1.0) (`CASE`)。

`PROPOSED` 的单集工作包如下：

1. 编剧/主策确认 episode card、情绪弧、时长与禁区；AI 只能起草。
2. 导演审查结构化分镜与镜头预算；锁定角色、服装、地点和道具版本。
3. 生成关键视觉帧；审核通过的帧才允许派生视频或动态漫画镜头。
4. 镜头级生成、技术 QC、人工粗剪；失败仅重试该镜头/资产版本。
5. TTS、字幕、音乐/音效、FFmpeg 合成；终审通过才生成可发布 render 与 publication record。

**建议的人工门**：季纲锁定、角色/画风 bible 锁定、每集分镜锁定、首轮关键帧、粗剪、终审/发布。它们是质量和权利责任的边界，不能因自动化目标而删掉。

镜头数没有可靠通用基准，必须由试产测得。可复用参考项目 `zenstory-ai/drama-skills` 自报一个 68 秒、20 镜头、约 ¥40 视频成本样例；这是该项目的示例而非市场成本或 Toonflow 实测。[官方仓库](https://github.com/zenstory-ai/drama-skills) (`CASE`)。

## 3. 30–100 集剧情：有限上下文下的持久记忆设计

### 3.1 分层产物而非无限对话历史（`PROPOSED`）

> **对账注**：下表的 `SeriesBible / SeasonPlan / EpisodeCard / ScenePlan / ShotSpec` 分层在语义上已由 `v118workflow` 落地为文件契约——`画布/剧本.json` 存「项目配置、来源索引、章节事件、故事骨架与分集大纲、改编策略、各集正式剧本」，`画布/资产.json` 存「基础与衍生资产、父子关系、视觉设定、提示词、已选图片、使用集数」，`画布/分集/第NNN集.json` 存「导演规划、分镜表、参考素材、分镜图、视频、选用与交付记录」。**建议沿用这套命名与布局**，把本节内容作为其 schema 化目标，而不是另起一套实体名。

```
Theme / Audience / Format
  └─ SeriesBible (世界规则、主题、禁区、结局约束)
      └─ SeasonPlan[] (季弧、角色弧、集群冲突、回收目标)
          └─ EpisodeCard[] (A/B 线、钩子、时长、状态变更)
              └─ ScenePlan[] (地点、出场角色、目标、冲突、结果)
                  └─ ShotSpec[] (画面/动作/对白/音效/资产版本)
```

每层都是经审核、版本化的结构化 JSON，而不是模型上下文中的散文。每次生成仅组装固定上下文包：已锁定的 bible 摘要、当前集卡、相关角色/地点版本、最近 N 集压缩摘要、该集未回收伏笔、以及由 ID 查询到的事实。原始对话、整季全文和已淘汰草稿不得不断拼入 prompt。

最小持久对象：

| 对象 | 不可变/版本化内容 | 连续性用途 |
| --- | --- | --- |
| `StoryFact` | `subject/predicate/object/evidenceEpisode/status` | 角色生死、地点、道具、世界规则。 |
| `Event` | 发生集/场、参与者、因果前提、后果 | 防止事件重复或无因果跳转。 |
| `Foreshadow` | 埋设、承诺回收窗口、回收证据 | 跟踪伏笔与逾期未收。 |
| `CharacterState` | 关系、目标、伤病、服装、声音、情绪状态 | 跨集行为与外观约束。 |
| `EpisodeSummary` | 经过人工确认的短摘要和状态变更 | 检索而非全文回放。 |

### 3.2 Qwen3.6-27B 的分工（`PROPOSED`）

现有 Qwen 文本路径可作为：主题→季纲草案、季纲→集卡、集卡→剧本/镜头描述、结构化 JSON 修复、以及独立“连续性审稿人”。每一步必须使用 JSON Schema/枚举、低温度或等价稳定参数、输入哈希、模型版本、prompt 版本和来源 ID 记录。`VERIFIED` 的仅是当前文本服务已接入；本机尚未验证结构化输出合规率、长季一致性或并发稳定性。

连续性检查采用三层：

- **确定性规则**：已死亡角色不得出场；场景资产、服装和镜头角色必须引用锁定版本；事件先后必须拓扑可排序。
- **检索式审稿**：从事实/事件/伏笔表取与当前集有关的有限证据，要求模型逐条给出冲突 ID 或 `NO_CONFLICT`。
- **人工裁决**：冲突可被“有意反转”豁免，但必须保存理由和新版本，不能静默覆盖旧事实。

这避免了无限上下文窗口依赖；也避免把向量检索当真相源。向量检索只用于召回，最终约束来自版本化结构化事实。

## 4. 角色、场景与跨集视觉/声音一致性

`PROPOSED`：把“提示词”升级为可审计资产合同。

| 合同 | 锁定字段 | 生成时必须带入 | 审核失败后的动作 |
| --- | --- | --- | --- |
| CharacterBible | 身份、年龄段、脸/发型、比例、禁改项、参考图、基础服装 | `characterAssetVersionId`、风格包、negative constraints | 新建候选版本；不得改写已发布版本。 |
| CostumeLook | 剧情状态、衣物、损耗、配饰、集/场适用范围 | `costumeLookVersionId` | 改分镜或批准换装事件。 |
| LocationBible | 空间布局、时段、色板、可复用背景、禁入物 | `locationAssetVersionId` | 新版本并回填受影响镜头。 |
| VoiceProfile | 权利/同意、语言、音色服务、情绪范围、版本 | `voiceProfileVersionId` | 停用、替代或重录；不能复制无授权真人声音。 |
| StylePack | 模型、工作流、LoRA/参考图、种子策略、许可证 | `stylePackVersionId` | 需兼容性测试才能升级。 |

研究候选：StoryDiffusion 公开论文/代码声称以一致性注意力改善多图故事中的角色风格与服装一致性，代码仓库声明 Apache-2.0；这证明其可作为**实验候选**，不证明 Apple Silicon、当前模型组合或生产质量。[项目页](https://storydiffusion.github.io/) / [许可证](https://github.com/HVision-NKU/StoryDiffusion/blob/main/LICENSE) (`VERIFIED` 许可证，`UNVERIFIED` 本机效果)。实际生产的主控制面仍应是参考图、锁定资产版本、图像审核和镜头级对比 QC，而不依赖单个论文算法。

## 5. 分镜、动态漫画与视频策略

### 5.1 ShotSpec（`PROPOSED`）

每个镜头至少有：`episodeId/sceneId/shotNo`、时长、构图、景别、相机运动、动作、台词时间码、角色/服装/地点/道具版本、输入资产、输出目标、生成 provider/workflow、重试上限、审核状态、质量规则与成本上限。镜头状态不能只靠文件名推断。

### 5.2 优先动态漫画，选择性真视频

| 镜头类型 | 首选 | 原因 |
| --- | --- | --- |
| 对话、内心独白、信息交代、静景 | 关键帧 + 平移/缩放/景深/分层视差 + FFmpeg | 角色漂移少、成本和失败率低。 |
| 表情、转身、短动作、气氛变化 | 图生视频或受控局部动画 | 只让视频模型处理其价值最高的运动。 |
| 打斗、追逐、复杂镜头、关键转场 | 图生视频/云端视频候选 + 人工复核 | 质量上限更高，也最容易产生时序和身份错误。 |

这使“60 秒”不等于每个镜头都调用视频模型。媒体任务的独立 ID、输入指纹、输出 manifest 和可重试 policy 是必要条件。

现有 Toonflow `packages/providers/types.d.ts` 已定义 `generateImage`、`generateVideo`、`generateAudio`、`resumeTask`，服务器 `apps/server/src/utils/media/generation.ts` 和 `tasks.ts` 能保存/恢复单媒体任务（`VERIFIED`）。它仍缺少镜头依赖、批量租约、跨任务幂等、质量门和资产版本（`MISSING`）。

本机视频研究结论维持不变：LTX Desktop 对 Apple Silicon 公开建议 48GB、且仍是 beta；Wan2.2 官方基线路径以 CUDA 为中心；HunyuanVideo 1.5 公开运行说明也以 Linux/CUDA 为中心。因此 Mac Studio 64GB 上的本地视频只能进入隔离基准后再决策，不能与稳定 Qwen 服务并发承诺生产吞吐。见 [本地媒体可行性](./LOCAL_AI_MEDIA_FEASIBILITY.md)。

## 6. 配音、字幕、后期与质量门

| 环节 | 候选 | 许可/适配结论 |
| --- | --- | --- |
| 中文多角色 TTS | [CosyVoice](https://github.com/QwenAudio/CosyVoice) | 官方仓库标注 Apache-2.0；仍须逐权重、依赖、音色来源和克隆同意核验。`UNVERIFIED` 本机效果。 |
| 字幕/ASR 校验 | [Whisper](https://github.com/openai/whisper) 或 [faster-whisper](https://github.com/SYSTRAN/faster-whisper) | Whisper 官方说明代码和权重 MIT；faster-whisper 仓库标注 MIT。作为字幕校验，不替代台词时间轴。 |
| 合成/检测 | FFmpeg | Toonflow 有 `packages/ffmpeg` 基础。发行前必须核验实际分发二进制是否启用 GPL 组件；FFmpeg 官方说明基础 LGPL 但可选组件可使整体为 GPL。[官方说明](https://ffmpeg.org/legal.html) |

流程：TTS 应输出台词片段及可用时长边界；无可靠边界时，ASR/强制对齐生成候选字幕，再以音频波形与人工抽检校正。合成 manifest 应保存帧率、分辨率、响度、字幕、每个镜头源文件哈希。技术 QC 至少检查空文件、编码/帧率、黑帧/静帧异常、时长漂移、缺字幕、音频削波/静音段和台词-字幕差异；叙事、肖像、版权和审美 QC 仍需人工门。

## 7. 连续生产领域模型与可恢复管理

`PROPOSED` 的核心实体：

```
Series → Season → Episode → Scene → Shot
     ├→ Character / Location / Relationship / StoryFact / Event / Foreshadow
     ├→ Asset → AssetVersion → GenerationTask → TaskAttempt
     └→ Review → Render → Publication
                         └→ CostLedger
```

- `GenerationTask` 是可重入的业务命令，不是 provider 轮询记录；包含输入快照、幂等键、依赖、租约、重试预算和 provider task reference。
- `TaskAttempt` 保存每次提交/轮询/取消/错误，便于恢复与成本归因。
- `Review` 是显式批准、驳回、标注或豁免，只有批准版本可流向下游。
- `Render` 由批准镜头 manifest 派生；`Publication` 与 render 分离，默认人工发布，保留平台披露、版权和审核证据。

支持几十集的关键不是一次创建数千任务，而是滚动窗口：仅释放已批准且在容量/预算内的 1–3 集；按季纲预建计划，按集锁定输入；暂停时不持有独占租约；服务重启后通过持久状态、输入哈希和 provider 查询恢复。禁止把本地文件目录当唯一任务数据库。

## 8. 可复用开源组件：准入表

| 候选 | 可复用内容 | 许可证/维护证据 | Mac/Toonflow 成本 | 结论 |
| --- | --- | --- | --- | --- |
| [drama-skills](https://github.com/zenstory-ai/drama-skills) | 创意→剧集→资产→分镜→审核的技能/提示词/检查清单 | 仓库 MIT；公开说明需 FFmpeg | 低：提取 schema 与验收方法，不嵌入 agent runtime | `PROPOSED` 参考其产物合同；不作为核心依赖。 |
| [ComfyUI](https://github.com/comfy-org/comfyui) | 图片/视频工作流、队列/API 适配对象 | 官方仓库；具体节点与模型须单独准入 | 中：独立 sidecar，不能占用 Qwen 端口/内存 | 继续作为图片和实验视频适配层候选。 |
| [StoryDiffusion](https://github.com/HVision-NKU/StoryDiffusion) | 跨图一致性实验 | Apache-2.0 | 高：依赖/模型/Apple 路径未验证 | 仅实验，不构成生产承诺。 |
| [CosyVoice](https://github.com/QwenAudio/CosyVoice) | 多语言 TTS 候选 | Apache-2.0（代码）；权重/音色另审 | 中：单独服务、音色授权账本 | P2 之后隔离评估。 |
| [Whisper](https://github.com/openai/whisper) / faster-whisper | ASR、字幕 QA | 官方 Whisper 代码/权重 MIT；faster-whisper MIT | 中：离线批处理可避免上传 | 可作为字幕校验候选。 |
| [FFmpeg](https://ffmpeg.org/) | 合成、转码、可验证 render | LGPL/GPL 取决于构建 | 低：Toonflow 已有包装，但须核验二进制 | 复用，先完成许可与 reproducible render 审计。 |

“开源”不自动等于可商用：代码、权重、LoRA、工作流节点、训练数据、参考图、声音、音乐、字体、输出分发平台条款均须分别入账。近期维护状态应在实际引入时按固定 commit/release 复核；本表不将网页当前活跃度当作长期承诺。

## 9. 成本、渠道和商业风险

没有跨模型、跨地区、跨质量目标的可靠公共单集成本，因此以下只提供计算框架，不给出伪精确报价。

```
单集可变成本 = Σ(图片尝试数 × 图片单价)
              + Σ(视频秒数 × 视频单价)
              + TTS/ASR/音乐费用 + 云存储/流量
              + 人工审核小时 × 内部费率
              + 失败重试预算
单集摊销 = 设备购置与维护 / 预计有效成片分钟数
```

| 方案 | 成本/时间判断 | 限制 |
| --- | --- | --- |
| 完全本地 | 变量 API 成本可低，设备、能耗、等待和失败重跑不可忽略 | 本机视频质量、64GB 内存与 Qwen 错峰吞吐均 `UNVERIFIED`。 |
| 本地文本/图 + 云视频 | 推荐 MVP 基线；把最不稳定、最高质量上限的镜头交由受控云备选 | 需要逐供应商预算、隐私与训练使用条款。 |
| 全云 | 易扩展和批量并发，但支出与数据出站最大 | 供应商锁定、队列/配额、平台政策风险。 |

10/50/100 集成本只能以实际镜头清单、重试率和供应商价表乘算；未完成试产前全部为 `UNVERIFIED`。收入也不能由播放量推断。发布前应分别核对：平台合成内容披露、重复/低价值内容的货币化规则、音乐/肖像/IP 授权和地区广告规则。TikTok 要求对逼真 AI 生成内容进行披露，[官方说明](https://support.tiktok.com/en/using-tiktok/creating-videos/ai-generated-content)；YouTube 有合成/修改内容披露与货币化质量规则，[披露](https://support.google.com/youtube/answer/14328491) / [货币化](https://support.google.com/youtube/answer/1311392)。这些规则会变化，发布时必须重新核验。

## 10. 本轮研究的决策边界

- 不把任何本地视频模型标为“已可生产”；先做隔离、限时、可回滚基准。
- 不让 Mac Studio 同时承载稳定 Qwen 和大型视频模型；错峰或独立服务是前提。
- 不自动发布；`Publication` 必须在人工终审、权利检查和平台披露确认后创建。
- 不在本轮更新 `ToDo.md`，候选施工项仅在路线图中列出，等待批准。

## 11. 来源与复核清单

1. [Toonflow 生产审查](./TOONFLOW_PRODUCTION_AUDIT.md)（源码证据基线）。
2. [Mac Studio 本地媒体可行性](./LOCAL_AI_MEDIA_FEASIBILITY.md)（模型、硬件和隔离边界）。
3. [DramaChain Bench](https://arxiv.org/abs/2609.00646)、[连续短剧研究原型](https://arxiv.org/abs/2605.22144)（研究资料，不是生产 SLA）。
4. [TikTok AIGC 披露](https://support.tiktok.com/en/using-tiktok/creating-videos/ai-generated-content)、[YouTube 合成内容披露](https://support.google.com/youtube/answer/14328491)、[YouTube 货币化](https://support.google.com/youtube/answer/1311392)（渠道规则，发布前重新核验）。
5. 本章各组件的官方仓库和许可证链接。所有商业结论须在采用具体版本、权重和供应商时由人工/法务复核。

## 12. 图片与视频模型专项选型结论（研究，不是设备验收）

### 12.1 严格状态标记

本节覆盖的每一个候选都必须同时经过下表的状态判断，不能以“官方有仓库”“能在 ComfyUI 里看到节点”或“别人在 NVIDIA 上跑过”替代本机证据。

| 状态 | 本轮可使用的条件 | 本轮结论 |
| --- | --- | --- |
| `RESEARCHED` | 已核对官方仓库、公开许可/模型卡、主张的能力与运行说明 | FLUX、Qwen-Image、SDXL、LTX、Wan2.2、HunyuanVideo 1.5。 |
| `COMPATIBLE_ON_PAPER` | 官方或可复核维护方公开 Apple Silicon/MPS/Metal 路径，且硬件容量不与公开建议明显冲突 | ComfyUI 图片路径、FLUX/SDXL 的 ComfyUI-MPS 路径、LTX Desktop。它不是成功运行证明。 |
| `VERIFIED_ON_DEVICE` | 在本 Mac Studio、锁定版本和隔离维护窗口内完成可复现记录：启动、峰值内存、5 秒输出、失败/取消、产物与日志 | **无。** |
| `PRODUCTION_READY` | 除设备测试外，已通过并发/恢复/安全/许可证/质量/成本与 Toonflow 适配验收 | **无。** |

当前稳定 Qwen 文本服务是用户确认已正常接入的基础设施；这不外推到任何图片或视频工作负载。

### 12.2 图片候选比较

| 候选 | 官方仓库/许可 | Apple Silicon 与 M5 Max 64GB | 中文、动漫与一致性 | API / Toonflow 适配 | 当前状态与取舍 |
| --- | --- | --- | --- | --- |
| **FLUX.1 schnell** | [BFL/flux](https://github.com/black-forest-labs/flux)；官方表列 schnell 为 Apache-2.0，dev 为非商用 | ComfyUI 有 Apple Silicon/MPS 路径，故 `COMPATIBLE_ON_PAPER`；官方没有本项目 M5 Max 64GB 的峰值内存/吞吐承诺 | 中文文字和动漫不是其已验证优势；角色一致性需要参考图、ControlNet/IP-Adapter/LoRA 或固定工作流 | ComfyUI `/prompt`/history/websocket 或独立 sidecar 通过既有 `generateImage` 适配 | **推荐商业图片基线**；`RESEARCHED` + `COMPATIBLE_ON_PAPER`，尚无本机验收。 |
| **原版 Qwen-Image** | [模型卡](https://huggingface.co/Qwen/Qwen-Image)，标 Apache-2.0 | 可研究 Diffusers/ComfyUI/社区 Apple 路径；本机内存、MPS 算子和速度未实测，至多 `COMPATIBLE_ON_PAPER` | 官方模型卡强调中文文字、编辑与动漫风格；跨镜头身份稳定仍须资产合同和参考图流程 | 优先经版本锁定的 ComfyUI/sidecar，不直接把模型加载进 Toonflow server | **中文漫画资产候选/备用路线**；画质、内存和 API 稳定性均未验证。 |
| **SDXL 基座/经审计派生模型** | [Stability-AI/generative-models](https://github.com/Stability-AI/generative-models)；具体模型卡、checkpoint、LoRA 各自许可 | ComfyUI MPS 生态较成熟，故图片路径 `COMPATIBLE_ON_PAPER`；64GB 是总内存而非该栈保证 | ControlNet/LoRA/IP-Adapter 生态丰富，动漫派生也多；恰因派生多，商用许可与来源风险更高 | ComfyUI 适合工作流/API；以固定工作流 hash 作接口合同 | **技术对照和兼容性备用**，不作为默认商用权重，除非逐权重审计。 |
| **FLUX dev / Qwen-Image 2.1 / 未审计 LoRA** | FLUX dev 非商用；[Qwen-Image 2.1 LICENSE](https://github.com/QwenLM/Qwen-Image-2.1/blob/main/LICENSE) 为研究/非商用 | 与设备是否能跑无关 | 可能有吸引力的编辑/参考能力 | 不应进入商业默认 provider | `RESEARCHED`，**商业生产排除**，除非取得适用商业授权并重审。 |

**内存回答**：没有可引用的、针对“Mac Studio M5 Max 64GB + 当前软件版本 + 指定分辨率/批量”的 FLUX、Qwen-Image 或 SDXL 峰值统一内存数字，因此任何 GB 数值都是 `UNVERIFIED`。64GB 只说明有比低内存机器更大的候选空间，不能证明与 Qwen 服务共存、不会内存压力或能稳定批量。图片验收时必须记录：冷启动/热启动、峰值 unified memory、swap/内存压力、每张首图和后续图时长、目标分辨率、并发=1 的成功率、取消后的资源释放和连续 20+ 次任务失败率。

**一致性方案（`PROPOSED`）**：先人工批准 `CharacterBible`（turnaround、表情、体型、禁改特征）、`CostumeLook`、`LocationBible`、`StylePack`；镜头只引用这些 asset version，不从自由提示词重新定义角色。每次图像任务保存参考图 hash、工作流 JSON/hash、模型/权重/LoRA/VAE 版本、seed、prompt 模板和输出 hash。IP-Adapter/ControlNet/LoRA 是可能的控制手段，不是已验证的 Toonflow 功能或商业可用承诺。相似度/身份检测只能作为后续候选 QC，首版必须人工 gate。

**资源冲突**：MPS/Metal 或 MLX 路径共享 Mac Studio 的统一内存和 GPU 资源；不同端口/不同进程不会使内存隔离。图片作业不可自动与 :8000 Qwen 重任务并发。推荐的资源策略是 `qwen-text` 与 `media-heavy` 两个互斥 resource group、媒体最大并发 1、人工批准的空闲时段，以及失败时不重启/不改动 Qwen。

### 12.3 视频候选比较

| 候选 | 实际 Apple 路径、I2V 与动漫动作 | 64GB / 速度证据 | 自动 API 与一致性 | 商业使用 | 当前结论 |
| --- | --- | --- | --- | --- | --- |
| **LTX 2.5 Fast / LTX Desktop** | [LTX Desktop](https://github.com/Lightricks/ltx-desktop) 公开支持 Apple Silicon，列出 T2V、I2V（首帧/可选尾帧）、队列和资产持久化；因此是唯一首选的 `COMPATIBLE_ON_PAPER` 本地视频候选。I2V 可由批准角色/场景关键帧起步，但不等于保证动漫身份/动作稳定。 | Desktop 公示 macOS 26+、48GB+ 建议；64GB 满足该公开容量门槛，但 M5 Max、目标时长/分辨率、与 Qwen 共存、峰值内存和 5 秒速度均 `UNVERIFIED`。没有可采纳的 M5 Max 64GB 可靠基准，禁止写入 SLA。 | Desktop 队列不自动等价为稳定服务 API；先验证是否存在版本稳定、可认证、可查询/取消的 sidecar 接口。人物/场景一致性仍取决于输入关键帧、短镜头、资产锁定、审核。 | 应区分 app 代码与权重；[LTXV Open Weights License](https://static.lightricks.com/legal/LTX-Video-Open-Weights-License-0.X.pdf) 及所选版本的 gated 条款/收入门槛必须逐版本确认。 | **本地视频第一验证对象**，仅 `RESEARCHED` + `COMPATIBLE_ON_PAPER`。 |
| **Wan2.2 TI2V-5B** | [Wan2.2](https://github.com/Wan-Video/Wan2.2) 公开 T2V/I2V、720p/24fps、ComfyUI/Diffusers 集成；图生视频可用作关键帧驱动。官方主运行/基准路径面向 CUDA/NVIDIA，未构成可维护的 Apple Silicon 方案。动漫动作效果没有本机或官方针对本目标的保证。 | 5B 名称或 4090 示例不能换算为统一内存；MPS/MLX、offload、5 秒性能、稳定性全部 `UNVERIFIED`。 | ComfyUI 可提供工作流级接口，理论上可异步轮询；需要先验证 Apple 平台生成、取消、重启恢复和工作流可重复。身份保持仍需 I2V 输入、短片段和人工 QC。 | 仓库 [LICENSE.txt](https://github.com/Wan-Video/Wan2.2/blob/main/LICENSE.txt) 标 Apache-2.0；实际权重、节点、LoRA 和输入/输出权利必须逐项复核。 | **第二验证候选**，`RESEARCHED`，尚不能标为 Apple 兼容或生产可用。 |
| **HunyuanVideo 1.5** | [官方 README](https://github.com/Tencent-Hunyuan/HunyuanVideo-1.5) 的运行要求为 Linux + NVIDIA CUDA；虽然具备 T2V/I2V/生态集成描述，但不构成 Apple 路径。 | CUDA 的 14GB/offload 宣称不能转写为 M5 Max 内存需求或速度。 | 即使存在 ComfyUI 节点，也不解决平台要求和一致性。 | Tencent Hunyuan Community License / AUP，须单独审核。 | `RESEARCHED`，**不是当前 Mac Studio 本地路线**。 |

**视频一致性策略（`PROPOSED`）**：以已批准图片作为 I2V 起始帧；单镜头短、动作单一；镜头输出不能反向覆盖角色/场景 canonical asset；高风险动作先低分辨率候选，人工选择后才升格；把真正视频只留给运动/转场/情绪关键镜头，对话和静景采用图片分层、镜头推拉、视差、字幕和 FFmpeg。没有模型能在本报告范围内证明“自动保证人物和场景一致”。

### 12.4 推荐组合与云端降级

**推荐试验顺序，不是部署指令：**

1. 商业图片基线：`FLUX.1 schnell + ComfyUI sidecar + 批准的角色/场景参考资产`。
2. 中文/漫画对照：`原版 Qwen-Image + 同一资产合同/同一测试集`。
3. 本地视频第一关：`LTX 2.5 Fast/Desktop` 的隔离单作业验收；只在 Qwen 明确空闲、独立端口、无敏感素材的批准维护窗口进行。
4. 若第一关失败或不达标，本地图片保持、视频采用**经合同批准的云端 I2V provider**；选择标准是可查询异步 job、webhook/poll、取消、区域/保留/训练使用条款、版本/输入可追溯、预算上限和明确商业输出条款，而非“当前最热门”。本轮不指定供应商或价格，也不调用付费 API。

这就是最合理的云端备用：本地 Qwen 负责规划、本地图片负责资产和大部分动态漫画、云端只承担经人工批准的少量高运动 I2V 镜头，并在 Toonflow provider 中以 capability contract、逐镜头预算和人工降级决定接入。若云端也不符合权利/成本门，则回退为无真视频的动态漫画，而不是悄然替换模型或自动外传素材。

### 12.5 未解决风险和后续验证关卡

| 验证关卡 | 必须回答的问题 | 通过证据 | 当前状态 |
| --- | --- | --- | --- |
| IMG-G0：许可/供应链 | 指定模型、权重、节点、LoRA、参考资产能否商用，是否可分发 | 锁定 revision、许可证台账、人工批准 | 未开始。 |
| IMG-G1：设备基准 | M5 Max 64GB 能否在不影响 Qwen 的条件下生成目标图片 | 内存/功耗/耗时/失败率/恢复记录和样本 | 未开始。 |
| IMG-G2：一致性 | 角色、服装、地点在 20+ 镜头内是否可接受 | 盲审评分、返工率、完整输入 manifest | 未开始。 |
| VID-G0：可维护运行 | 是否有可安装、可停用、可升级回滚的 Apple 视频路径 | 隔离启动/停止、端口、日志和版本记录 | 未开始。 |
| VID-G1：单镜头 | 能否产出并导出 5 秒 I2V，速度/内存/取消如何 | 至少冷/热作业、峰值内存、错误分类、产物 QC | 未开始。 |
| VID-G2：工作流 | 是否存在稳定 API、状态查询、取消、断电/重启恢复 | 契约测试和故障注入证据 | 未开始。 |
| VID-G3：生产质量/成本 | 身份、场景、动作、时序是否足够；重试是否可承受 | 盲审、重试率、实际单镜头成本和人工工时 | 未开始。 |

正式施工前需要用户的技术决策：

1. 允许进入基准的**精确图片模型/权重/LoRA**与商业许可立场；是否接受 FLUX schnell 为首基线、Qwen-Image 为中文对照。
2. 是否授权在指定维护窗口下载/运行**一个**隔离图片候选；Qwen 是否必须始终在线、以及可接受的资源互斥策略。
3. 是否把 LTX 列为唯一首个本地视频基准对象，还是直接采取“本地图片 + 云端视频”的 MVP 路线。
4. 云视频备用的预算上限、可传输的素材类别、数据地域/保留/训练使用红线以及允许的供应商清单。
5. 成片规格（9:16、分辨率、帧率、5 秒片段定义）、可接受的人工审核返工率和首个 MVP 的质量门。
6. 是否接受动态漫画优先，避免把每一镜头都设为视频生成，从而控制成本、风险和一致性问题。

**直接回答本轮专项问题**：图片目前缺模型/工作流的本机基准、角色资产合同实测、许可台账和 Toonflow sidecar 契约；视频目前缺可维护的 Apple 运行证据、M5 速度/内存/质量数据、稳定 API/恢复验证和可接受商业条款。推荐组合是现有 Qwen + FLUX.1 schnell（图片基线）+ 原版 Qwen-Image（中文/动漫对照）+ ComfyUI sidecar，视频先验证 LTX、失败则采用受控云 I2V；所有这些媒体部分均非 `VERIFIED_ON_DEVICE`，更非 `PRODUCTION_READY`。

## 13. 对账修订记录（2026-11-15，CodexQ 复核）

本研究的公开资料结论、许可证判断、模型选型状态分级（`RESEARCHED` / `COMPATIBLE_ON_PAPER` / `VERIFIED_ON_DEVICE` / `PRODUCTION_READY`）与「当前无一项达到设备验证」的结论**全部继续有效**。本轮修订集中在「与 Toonflow 既有能力的重叠」被低估的问题。完整对账见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md)。

| 章节 | 修订 | 依据 |
| --- | --- | --- |
| §1 结论 | 补入三个已安装 skill 的存在，并把「没有连续剧生产领域模型」限定为「没有**服务端强制的**领域模型」 | `packages/skills/`、`data/skills/{canvas,v118workflow,workflow}` |
| §3.1 分层产物 | 标注与 `v118workflow` 三画布契约语义重叠；建议沿用既有命名与目录布局，本节改为 schema 化目标 | `packages/skills/v118workflow/SKILL.md` |
| §3.2 连续性三层检查 | 「确定性规则 / 检索式审稿 / 人工裁决」中的**规则目录已存在**：PART A `A0.1 剧情保真锁`（事实来源优先级、五类绝对禁止改动、具体化边界、不补前史、模糊保持模糊、空缺/矛盾处理）、`A8 跨段连续性检查`、`A11 结构校验`。工作性质由「设计规则」改为「把规则固化为可执行校验 + 测试」 | `packages/skills/workflow/SKILL.md:96-169,714-781,1037-1143` |
| §4 资产/声音合同 | `CharacterBible/CostumeLook/LocationBible/VoiceProfile/StylePack` 应与 `v118workflow` 资产画布字段（角色识别锚点、场景空间与色光、道具结构、变体差异、可执行提示词、采用版本、使用集数）及 `stateAndDelivery.md` 的「确认绑定历史快照」「生成前保存提示词+参数+引用+授权快照」对齐，避免双轨命名 | `packages/skills/v118workflow/references/{baseAssets,deriveAssets,generateAssets}.md`、`packages/skills/workflow/references/stateAndDelivery.md` |
| §5.2 视频策略 | 补入现成可复用节点 `director3dNode`：文字生成场景/动画/运镜、关键帧导出 PNG 图片节点、动画导出 MP4 视频节点，适合动态漫画的运镜与关键帧；**但 `renderVideo` 使用浏览器 `MediaRecorder`（avc1.420028、6 Mbps、~30fps）且要求窗口可见，不可无人值守** | `packages/nodes/director3dNode/{readme.md,src/renderMedia.ts:5,22}` |
| §6 配音/字幕/后期 | 补入源码事实：HTTP `audioGenerationSchema` 已支持 `voice/speed/volume/pitch/language/format/sampleRate`，`MediaModel.voices` 已定义，`emotion` 仅在 provider 合同；音频节点 UI 无 voice 选择器。合成侧 `concat`/`complexFilter` 已可达，缺 manifest/字幕产物/规格门禁 | `apps/server/src/utils/media/schemas.ts:44-59`、`packages/providers/types.d.ts:25,175`、`packages/ffmpeg/src/browserTypes.ts:3-24` |
| §7 领域模型 | 补入选项：**第一阶段真相源可为「工作区文件 + 只读索引器」**，数据库化留待 `P1-002` ADR 决定；`GenerationTask/TaskAttempt` 可复用 `utils/media/tasks.ts` 已有的 `taskId/nodeId/canvasPath/chargeId/chargeStatus/checkpoint/files[].hash` 字段语义 | `apps/server/src/utils/media/tasks.ts:38-50` |
| §9 成本框架 | 补入既有能力：`multiTenantBillingLedger`（requestId 幂等、reserved/settled/released、requestHash、usage、micros 金额）、媒体任务 charge 绑定与按秒结算、`billingUsage.ts` 三协议 token 解析、`getMediaPrice`/`priceReference.ts` 预估。**个人/单机模式无任何成本记录**。并明确数据口径：`MEASURED` / `PUBLIC` / `ESTIMATE` 必须分列，本机当前**无任何 `MEASURED` 数据** | `packages/multiTenant/db/{sqlite,mysql}.ts`、`apps/server/src/utils/media/{tasks,billing,priceReference}.ts`、`apps/server/src/utils/ai/billingUsage.ts`、`apps/server/src/agent/tools/index.ts:52-59` |
| §10 决策边界 | 新增一条：**禁止自动卸载 Qwen**。当前无法判断其他项目是否在使用（其他客户端直连 `:8000` 绕过 Toonflow；`utils/ai` 与 `routes/ai` 无任何在途/并发计数；`fetchProviderModels` 仅存活探测），模型切换一律人工审批 + 维护窗口 | 对账报告 §8.1；`apps/server/src/utils/ai/models.ts:12-26` |
| §12 模型选型 | 结论不变；补入统一接入口径：所有本地模型（图片/视频/音频）一律实现既有 provider 合同并经 `routes/providers/media/{add,save,validate}.ts` 安装，**不新建媒体 API** | `apps/server/src/routes/providers/media/`、`apps/server/src/utils/media/provider.ts` |

`RESEARCH_RECONCILED`
`NO_MODEL_VERIFIED_ON_DEVICE`
`IMPLEMENTATION_NOT_STARTED`
