# Mac Studio 本地 AI 媒体生产可行性研究

> 日期：2026-11-15；范围：方案研究与源码适配评估。未安装权重、未启动 ComfyUI/视频生成、未占用 :8000、未改变 Qwen/MLX/OpenCode。
> 状态词：`VERIFIED` = 官方源码/模型卡或 Toonflow 源码确认；`USER-CONFIRMED` = 用户确认 Qwen 已接入；`UNVERIFIED` = 未在本机实测；`PROPOSED` = 推荐设计。

## 1. 结论与推荐

**本地优先的现实路径是：Qwen 文本 + 本地图片 + 云端视频兜底，而不是把 60 秒视频全压在现有 Mac Studio。**

| 层 | 推荐 | 状态 | 原因 |
| --- | --- | --- | --- |
| 文本/剧本 | 现有 Qwen3.6-27B OptiQ 4bit | `USER-CONFIRMED` | 已正常接入 Toonflow；保持原服务、端口与参数不变 |
| 商用本地图片基线 | FLUX.1 `schnell` + 角色/场景参考工作流 | `VERIFIED`（许可/API候选） | 官方列为 Apache-2.0；速度优先，适合批量草图和候选筛选 |
| 商用本地图片进阶 | 原版 Qwen-Image（非 2.1）用于中文文字、编辑和漫画素材试验 | `VERIFIED`（原版模型卡 Apache-2.0） | 官方模型卡强调中文文字、编辑与动漫风格；Apple Silicon 实测性能仍 `UNVERIFIED` |
| 工作流/API 层 | 单独 ComfyUI sidecar，MPS/Metal | `VERIFIED`（Apple Silicon 支持） | 有成熟 workflow/API 生态；不得与 Qwen 同时进行重任务 |
| 本地视频首选试验 | LTX 2.5 Fast / LTX Desktop 或其明确 API | `VERIFIED`（Apple Silicon 支持） | 官方 LTX Desktop 支持 Apple Silicon、T2V/I2V、共享队列和重启持久化；64GB 超过官方 48GB 推荐下限 |
| 本地视频候选试验 | Wan2.2 TI2V-5B | `PARTIAL` | 官方支持 T2V/I2V、720p/24fps、ComfyUI；官方基准/脚本是 CUDA/NVIDIA，Mac Studio 不可承诺运行或速度 |
| 不作为当前本地路径 | HunyuanVideo 1.5 | `VERIFIED` 不适配 | 官方系统要求 Linux + NVIDIA CUDA，虽有 I2V/ComfyUI 计划，非 Apple Silicon 首选 |
| 视频生产兜底 | 已合同化云视频 API | `PROPOSED` | 对本地不可用、排队过久或 QC 失败的镜头按预算人工批准后降级 |

**不得以模型“开源/开权重”推断商业可用。** FLUX.1 dev 明确非商用；Qwen-Image 2.1 明确研究/非商用；LTX 权重在大于等于 1,000 万美元年收入实体上须另取商业许可。每个实际下载的权重和 LoRA 都要逐个登记许可证、版本和获取日期。

## 2. 现有资源与保护边界

| 资源 | 约束 |
| --- | --- |
| Mac Studio M5 Max / 64GB 统一内存 | 足以成为**错峰**本地图片和 LTX 视频的候选平台；不是 Qwen + 大视频模型并发容量保证 |
| Qwen MLX Serve :8000 | 生产稳定基线；不得重启、替换、改端口、改模型/上下文/并发配置 |
| Mac mini | 仅作 Toonflow、编排、任务队列、FFmpeg 和远程调度控制平面 |
| 本次研究 | 未测得任何实际吞吐/内存/画质；下文所有性能数字均为规划假设或 `UNVERIFIED` |

推荐方案 A（低风险）：**时隙互斥**。Qwen 服务处于正常生产时，只执行文本/编排；本地图片或视频必须由 Mac mini 队列在人工批准的维护窗口提交，确认 Qwen 空闲后才加载，生成完卸载并归档指标。

方案 B（中风险）：LTX Desktop/sidecar 独立服务运行在 Studio 的不同端口、独立目录和单作业队列，但只在 Qwen 被明确置于空闲窗口后启动。独立进程不是独立内存，不能解决统一内存争用。

方案 C（推荐长期）：Mini 负责 scheduler/API，Studio 保持受控推理 worker；采用 lease、心跳、最多一个重媒体 job。失败时只回退到经批准的云视频 provider，不自动改变 Studio 服务。

## 3. 本地图片生成研究

| 候选 | 一致性/动漫能力 | Apple Silicon 路径 | 商业许可 | 生产判断 |
| --- | --- | --- | --- | --- |
| FLUX.1 schnell | 参考/一致性需靠 ControlNet、IP-Adapter/LoRA 或一致性 workflow，均待验收 | ComfyUI MPS；MLX 社区实现存在但非本次运行验证 | Apache-2.0 | **首选商用草图/批量候选基线** |
| FLUX.1 dev/Fill/Redux/Kontext dev | 编辑、结构控制和变体能力强 | MPS/ComfyUI 可研究 | dev 系列为非商用；商业需 BFL 许可 | 不进入商用生产，除非采购许可 |
| SDXL | LoRA/ControlNet 生态成熟，动漫 checkpoint/LoRA 丰富；每个派生权重需独立审核 | ComfyUI MPS；官方模型卡提示 Apple 可切至 MPS | 具体 checkpoint 许可证不统一；示例 0.9 为研究/非生产许可 | 可作技术对照，不作为默认商用基础权重 |
| 原版 Qwen-Image | 官方称支持中文文字、图像编辑和动漫风格 | Diffusers/社区 MLX 路径；原生 Mac 性能待测 | 官方模型卡标 Apache-2.0 | **可作为中文漫画资产候选**，先做隔离基准 |
| Qwen-Image 2.1 | 参考图/编辑潜力需独立测 | 有 Mac 生态线索，非生产证据 | Qwen Research License，非商用 | 仅研究，不进入商业产线 |

官方资料：FLUX.1 schnell/ dev 许可证差异见 [BFL 官方仓库](https://github.com/black-forest-labs/flux)；Qwen-Image 官方模型卡称其擅长中文文字、编辑和动漫风格并标注 Apache-2.0，[模型卡](https://huggingface.co/Qwen/Qwen-Image)；Qwen-Image 2.1 的官方 LICENSE 为非商用研究许可，[LICENSE](https://github.com/QwenLM/Qwen-Image-2.1/blob/main/LICENSE)；ComfyUI 官方提供 Apple Silicon 安装路径，[README](https://github.com/comfy-org/comfyui)。

### 一致性工作流（PROPOSED）

1. Qwen 先输出经 schema 校验的 `CharacterBible`、`SceneBible`、`ShotSpec`，其中每个角色/服装/场景/道具拥有稳定 ID。
2. 用图片模型生成并**人工批准**角色 turnaround、表情表、服装集、场景 keyframe；写入不可变 asset version 与提示词/seed/workflow hash。
3. 每一镜头只引用批准版本，不让自由文本覆盖 canonical asset。先生成低分辨率候选，人工选片后升采样/精修。
4. 由 ComfyUI workflow 接收 `prompt + negativePrompt + referenceAssetIds + seed + styleLoraVersion`；返回 media URL、完整 workflow JSON、模型/LoRA 哈希、seed、耗时。
5. 角色相似度/场景连续性目前没有可信本地自动评分证据，先由人工 gate；自动 QC 后续独立验证。

## 4. 本地视频生成研究

| 模型 | 官方能力 | 官方硬件/Apple 证据 | API/ComfyUI | 许可 | 5 秒性能判断 |
| --- | --- | --- | --- | --- | --- |
| LTX 2.5 Fast | LTX Desktop 宣称本地 T2V、I2V（首帧/可选尾帧）、A2V、编辑/扩展 | 官方 desktop 支持 Apple Silicon；推荐 macOS 26+、48GB+；64GB 符合容量建议，M5 Max 未单独测 | Desktop 有队列/资产持久化；Toonflow 需 sidecar API adapter | app Apache-2.0；权重另有条款/可能 gated；旧 LTXV 对大收入实体有额外商用许可 | `UNVERIFIED`：没有可引用 M5 Max 64GB 5 秒实测，不写入 SLA |
| Wan2.2 TI2V-5B | T2V + I2V、720p、24fps；官方列出 ComfyUI/Diffusers 集成 | 官方 5B 说明称可跑消费级 4090；原生脚本/优化路径为 CUDA，不证明 MPS/MLX | ComfyUI 官方整合；可做异步 provider | 仓库 `LICENSE.txt` 为 Apache-2.0；下载权重仍需在锁定版本核验 | `UNVERIFIED`：禁止把 4090 结果换算为 M5 Max |
| HunyuanVideo 1.5 (8.3B) | T2V/I2V；官方列 ComfyUI、Diffusers、LightX2V | 官方要求 Linux + NVIDIA CUDA，最低 14GB 显存（offload） | 有 ComfyUI 计划/支持 | Tencent Hunyuan Community License/Acceptable Use | 当前 Mac 路径 `NOT_RECOMMENDED`；性能不估算 |

来源：Wan2.2 官方仓库确认 TI2V-5B 的 T2V/I2V、720p/24fps 及 ComfyUI 集成，[Wan2.2](https://github.com/Wan-Video/Wan2.2)；其许可证文本为 [Apache-2.0](https://github.com/Wan-Video/Wan2.2/blob/main/LICENSE.txt)。LTX Desktop 官方列 Apple Silicon、队列、持久资产及 48GB+ 建议，[LTX Desktop](https://github.com/Lightricks/ltx-desktop)；LTX 权重许可见 [LTXV Open Weights License](https://static.lightricks.com/legal/LTX-Video-Open-Weights-License-0.X.pdf)。HunyuanVideo 1.5 官方明确 Linux/CUDA 要求，[README](https://github.com/Tencent-Hunyuan/HunyuanVideo-1.5)。

### 视频选择结论

- **第一个本地视频验收对象：LTX 2.5 Fast。** 原因是官方 Apple Silicon 支持与 64GB 容量建议最直接；但它仍是 Beta，且默认桌面应用不是 Toonflow production API。先验证可部署、可启动、可用 API/队列、可取消和可读取资产，才考虑 adapter。
- **第二个候选：Wan2.2 TI2V-5B。** 其 5B TI2V 规格和 Apache-2.0 表面上更有吸引力，但当前 Apple MPS/MLX 与 64GB 的实际成功率、速度、offload 兼容性都未验证。
- **不选 HunyuanVideo 1.5 做现阶段本地服务。** 不因为其 14GB CUDA offload 需求小就推断能在 Metal 上运行。

## 5. Toonflow 最小集成架构

`VERIFIED`：Toonflow 的 provider 合同已定义 `generateImage`、`generateVideo`、**`generateAudio`**、`processMedia`、`resumeTask`、`saveTaskData`（`packages/providers/types.d.ts:238-266`）；服务端 `apps/server/src/utils/media/tasks.ts` 持久化 `providerData` 与产物哈希，可恢复单个媒体任务。`videoGenerationNode` 可用文字、图片、视频、音频参考输入。

`VERIFIED`（**本轮补充：唯一正确接入通道**）：本地 sidecar 必须实现上述合同，并经**现有自定义 provider 安装通道**注册——`apps/server/src/routes/providers/media/{add,save,validate}.ts` + `apps/server/src/utils/media/provider.ts`。运行时清单由 `utils/media/generation.ts#listMediaModels` 只列出**已安装且已配置**的 provider。**不得新建任何媒体生成 API**：`POST /api/ai/media/generate`（`apps/server/src/routes/ai/media/generate.ts`）已统一承载 image/video/audio 的提交、查询（`tasks/status`）、取消（`tasks/cancel`）与恢复。

`VERIFIED`（**本轮补充：批量生产的真实前置**）：现有两条执行路径必须区分——

1. **画布路径**：Agent 经 `canvas` 工具调用 `node:generateImage` 等，实际由前端执行（`apps/server/src/agent/bridge/canvas.ts:10,45`，120s 超时、要求画布打开、回传仅进程内有效）→ **不可无人值守**。
2. **无头路径**：`context.media` 已把 `listModels/generateImage/generateVideo/generateAudio` 注入工具上下文（`apps/server/src/agent/tools/index.ts:146-151`；类型 `packages/toolScaffold/src/runtime.ts:140-145`），底层直连服务端 `generateMedia(cwd, ...)`（`utils/media/generation.ts:243`，默认落盘 `assets/generated`），**不需要浏览器**；但**当前无任何工具消费 `context.media`**。

因此在实现 sidecar provider 的同时，必须新增**一个消费 `context.media` 的薄插件工具**（与 `packages/tools/*` 同构），否则批量生产仍会被浏览器绑定。该工具是 `P2-005` 的必要子项，也是 `P4-001` 的前置。

```text
Toonflow node / future production DAG
    │  providerId / modelId / requestId
    ▼
localMediaProvider (新增、薄适配层)
    │ HTTPS 或局域网受控 HTTP；jobId、input hash、deadline
    ├── imageSidecar: ComfyUI API + pinned workflow
    ├── videoSidecar: LTX API/worker (首选试验)
    └── fallback: 已批准云 provider（仅人工授权降级）
    ▼
Mac Studio worker supervisor
    ├── Qwen slot: :8000，现有稳定服务，不受修改
    └── Media slot: maxConcurrent=1，与 Qwen 重任务互斥
```

适配器必须保存 `jobId`、输入/工作流/模型哈希、产物 URI、状态、可恢复截止时间和错误分类；绝不保存 API key 到任务 checkpoint。进度查询可以映射为 Toonflow 的 `mediaTasks.status`，取消映射为远端 cancel；若 sidecar 不支持 cancel，适配器须显式报告“已请求、未保证终止”。

当前仅支持单个媒体任务的恢复，**不支持**完整短剧批量自动化；后者仍依赖既有 `P4-001` 持久 DAG 队列，并以上述无头工具为前置。

**音频接入补充（`SOURCE_VERIFIED`）**：HTTP 层 `audioGenerationSchema`（`apps/server/src/utils/media/schemas.ts:44-59`）已支持 `voice`、`speed`、`volume`、`pitch`、`language`、`format`、`sampleRate`；模型能力可经 `MediaModel.voices`（`packages/toolScaffold/src/runtime.ts:93`）声明。`emotion` **仅在** provider 合同 `AudioRequest`（`packages/providers/types.d.ts:175`），**不在 HTTP schema**，需经 `other` 透传或后续扩展 schema。`audioGenerationNode` UI **未暴露** voice 选择器（`src/index.vue:102`），故多角色配音目前只能由 Agent/API 直传 `voice`。TTS sidecar 应实现 `generateAudio` 并填 `voices`，**不得占用 `:8000`**。

## 6. 单集 60 秒 MVP（PROPOSED）

假设：中文 60 秒、12 个约 5 秒镜头、720p/24fps、1–2 位角色、4 个场景、人工仅在 script/asset/shot/render gate 审核；不含真人肖像、受保护角色、未经许可的声音/音乐。

```text
主题 → Qwen: ScriptSpec → Qwen: Character/Scene Bible → Qwen: 12×ShotSpec
  → 本地图片: 批准角色/场景 keyframe → I2V 每镜头
  → TTS/字幕（经批准 provider） → FFmpeg render manifest → 技术 QC → 人工发布批准
```

### 时间与成本：必须区分规划假设和实测

| 模式 | 单集时间（规划估计） | 边际模型/API成本 | 可扩展性/风险 |
| --- | --- | --- | --- |
| 全本地 | `UNVERIFIED`。公式为 `12 × 本地每镜头尝试次数 × 本地5秒耗时 + 图片/合成/人工`；在取得 M5 实测前不得承诺时长 | API 可近似 0；仍有电力、存储、维护和失败重跑成本 | 最低数据外传；单机单媒体 job，吞吐低，许可/稳定性风险最高 |
| 本地文本/图片 + 云视频 | 云视频异步 SLA + 人工审核，通常由供应商队列主导；本机只负担图片/合成 | 云视频按秒/次数 + 存储；必须从合同/账单实测 | **MVP 推荐**：质量和吞吐较稳，成本可按镜头封顶 |
| 本地文本 + 云图片/视频 | 供应商队列主导 | 最高 API 账单；最低本地维护 | 最快验证产品流程，但数据外传和供应商锁定最高 |

为便于容量计划（非性能宣称），先按每镜头 `N=3` 候选、12 镜头计算 36 次视频尝试。若隔离基准测得单次 5 秒输出耗时为 `T` 分钟，则媒体墙钟下限约为 `36 × T / 60` 小时（单并发，未含图片、合成和人工）。例如，只有当基准确认 `T=10` 分钟时才可写作约 6 小时；`T` 在本机验证前为 `UNVERIFIED`。电力成本应以实测平均墙功率 × 墙钟小时 × 本地电价计算，不使用设备 TDP 代替。

## 7. 商业、隐私与操作决策

- 只有 FLUX.1 schnell（Apache-2.0）可作为当前明确的商用本地图片候选；仍需审查所有 LoRA、VAE、ControlNet、素材与输出用途。
- 原版 Qwen-Image 模型卡标 Apache-2.0；**Qwen-Image 2.1 不可混用**，其官方 Research License 仅限非商用。
- LTX Desktop 的代码 Apache-2.0 不等于模型权重 Apache；权重的 gated/许可和企业收入门槛需法务逐版本确认。
- Wan2.2 仓库许可是 Apache-2.0；实际所取权重、第三方 Comfy 节点和 LoRA 仍须锁定 SHA 后再审。
- 云端后备必须记录数据地域、保留/训练使用条款、内容审核、成本上限与供应商故障退出方案。
- Qwen/媒体错峰是必要的资源隔离，不是安全边界；所有 sidecar 必须单独端口、loopback/VPN 限制、密钥隔离和可审计任务日志。

## 8. 验收顺序

1. 先完成原始 P0/P1，不开始模型安装。
2. 人工选择许可证可接受的图片/视频候选与预算；完成 `P1-003` 基准协议。
3. 在 Qwen 空闲维护窗口，以 LTX 2.5 Fast 做非生产、单任务、无敏感素材的能力测试；记录内存、功耗、首作业/热作业耗时、失败率、取消/恢复。
4. 仅当图片或视频 sidecar 通过隔离基准，才让 OpenCode 实现薄 provider adapter；不得直接让 agent 安装模型或改 Studio 配置。
5. 单集 MVP 先采用“本地图片 + 云视频”成功交付一集，再决定是否扩展本地视频。

`RESEARCH_COMPLETE`
`LOCAL_VIDEO_PRODUCTION_UNVERIFIED`
`NO_MODEL_OR_INFRASTRUCTURE_CHANGE_PERFORMED`

## 9. 对账修订记录（2026-11-15，CodexQ 复核）

本文件的模型选型、许可证判断与「错峰互斥」结论**继续有效**；本轮只补充接入事实与纠正遗漏。完整对账见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md)。

| 修订项 | 内容 | 依据 |
| --- | --- | --- |
| §5 provider 合同 | 补入 `generateAudio` 与 `processMedia`（原文只列 image/video/resume/saveTaskData） | `packages/providers/types.d.ts:238-266` |
| §5 接入通道 | 明确本地 sidecar **必须**经 `routes/providers/media/{add,save,validate}.ts` 安装为自定义 provider，**不得新建媒体 API** | 同上 + `apps/server/src/routes/ai/media/generate.ts` |
| §5 批量前置 | 新增「画布路径 vs 无头路径」区分，并把「消费 `context.media` 的薄插件工具」列为 `P2-005` 必要子项、`P4-001` 前置 | `apps/server/src/agent/bridge/canvas.ts:10,45`、`apps/server/src/agent/tools/index.ts:146-151`、`utils/media/generation.ts:243` |
| 音频接入 | 补入 HTTP schema 已支持 `voice/speed/volume/pitch/language/format/sampleRate`；`emotion` 仅在 provider 合同；节点 UI 无 voice 选择器 | `utils/media/schemas.ts:44-59`、`providers/types.d.ts:25,175`、`audioGenerationNode/src/index.vue:102` |
| 资源互斥 | 强化为**强制规则**：当前无法判断其他项目是否在用 Qwen（其他客户端直连 `:8000` 绕过 Toonflow，且 Toonflow 无在途计数），故**禁止自动卸载**，一律人工审批 + 维护窗口 | 对账报告 §8.1；`MODEL_RESOURCE_SCHEDULING.md` §10 |
| 本机现状 | 补入运行时事实：`data/providers/` 仅 3 家云端适配器、无任何本地 provider；`data/workspaces/myTvProject` 为 0 文件、`data/assets/` 为空 → 本机**无端到端跑通证据**，所有性能/成本仍为 `UNVERIFIED` | `data/` 目录盘点（未读取 `settings.json`） |
| 可复用能力 | 补入此前遗漏的 `director3dNode`（运镜预演、关键帧导出 PNG、动画导出 MP4，但浏览器 `MediaRecorder` 绑定）与 FFmpeg `concat`/`complexFilter` 工具层可达 | `packages/nodes/director3dNode/src/renderMedia.ts:5,22`、`packages/ffmpeg/src/browserTypes.ts:3-24` |

`RECONCILED`
`NO_MODEL_OR_INFRASTRUCTURE_CHANGE_PERFORMED`
