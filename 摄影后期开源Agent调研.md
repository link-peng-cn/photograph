# 摄影后期方向开源 Agent 项目调研

> 调研时间：2026-10 ｜ 数据来源：GitHub API 元数据 + 论文原文 + 项目 README/描述
> 星标数为查询时快照，仅用于判断热度量级。

---

## 一、结论速览

**有，但不存在一个"高星、开箱即用、专做摄影后期"的成熟开源应用。**

现状是"学术侧很热、工程侧刚起步、中间层（MCP 桥接）最实用"，可以分成四条路线：

| 路线 | 代表项目 | 成熟度 | 适合谁 |
|---|---|---|---|
| A. 端到端修图 Agent（学术） | JarvisArt、PerTouch、EyeControl、IEA | 论文级，需 8×A100 级别训练 | 研究者、想复现/二次训练 |
| B. MCP / 工具桥接（工程） | lightroom-mcp 系列、chemigram、mcp-photo-edit | **最可用**，接上就能干活 | 想让 Claude/Cursor 直接修图的摄影师 |
| C. Agent Skills / 提示词包 | photo-skills、drshy-org/lightroom-py | 轻量，零训练 | 想低成本试水 |
| D. 相邻环节：选片/评分 | facet、pixcull | 相当成熟 | 婚礼/活动/鸟类摄影师 |

**最值得先看的三个：**
1. [LYL1015/JarvisArt](https://github.com/LYL1015/JarvisArt) —— 这个方向的标杆，NeurIPS 2025
2. [Automaat/lightroom-mcp](https://github.com/Automaat/lightroom-mcp) —— 生态里最活跃的工具层
3. [chipi/chemigram](https://github.com/chipi/chemigram) —— 完全开源栈（darktable）的 Agent 方案

---

## 二、路线 A：端到端修图 Agent（学术派）

### 1. JarvisArt — 这个方向的标杆 ⭐

- **仓库**：https://github.com/LYL1015/JarvisArt
- **论文**：[arXiv:2506.17612](https://ar5iv.labs.arxiv.org/html/2506.17612)（NeurIPS 2025）
- **数据**：约 **865★ / 41 fork**，Python，License 为 NOASSERTION（自定义，商用前务必读）
- **最近推送**：2026-04

**它做了什么**：MLLM 驱动的修图 Agent，理解自然语言意图 → 模仿职业修图师的推理 → 编排 **200+ 个 Lightroom 操作**。支持文本、框选、笔刷三种输入，支持任意分辨率。

**核心技术点**（值得借鉴的部分）：
- **两阶段训练**：CoT 监督微调打底 → **GRPO-R** 强化学习（自定义三类奖励：格式奖励 + 修图操作准确度奖励 + 感知质量奖励）
- **MMArt-55K 数据集**：5K 标准指令 + 50K CoT 增强样本，多粒度（场景级 / 区域级）
- **MMArt-Bench**：200 实例，覆盖人像/风光/街拍/静物；区域级另有 50 张人像带 mask
- **A2L（Agent-to-Lightroom）协议**：五阶段 client-server 接口（握手 → 文件校验 → 沙箱执行 → 异步处理 → 结果返回），把 ROC（修图操作配置）翻译成 Lightroom Lua 脚本

**效果**：论文称在 MMArt-Bench 的像素级内容保真度指标上比 GPT-4o 提升 60%，指令遵循能力相当。

**注意**：基座是 Qwen2.5-VL-7B，训练用了 8×A100(80G) 做 SFT + 16×A100 做 RL。**个人用户基本只能跑推理，复现训练不现实。**

---

### 2. 其他学术 Agent（同方向，规模小一档）

| 项目 | 会议 | 星标 | 一句话 |
|---|---|---|---|
| [Auroral703/PerTouch](https://github.com/Auroral703/PerTouch) | AAAI 2026 | 28★ | VLM 驱动的**个性化 + 语义**修图 Agent，强调"千人千面" |
| [DragonisCV/EyeControl](https://github.com/DragonisCV/EyeControl) | ECCV 2026 | 15★ | 意图驱动的修图 Agent，专注**视觉焦点增强**（把观众视线引到主体） |
| [OpenDFM/Image_Edit_Agent](https://github.com/OpenDFM/Image_Edit_Agent) | CVPR 2026 Findings | 11★ | IEA：面向**业余用户**的对话式图像编辑 Agent，三阶段多任务对齐 |
| [MediaX-SJTU/Agentic-Retoucher](https://github.com/MediaX-SJTU/Agentic-Retoucher) | CVPR 2026 | — | ⚠️ 是 **文生图**方向的 retoucher，不是摄影后期，别混淆 |
| [Visionary-Laboratory/PhotoFlow](https://github.com/Visionary-Laboratory/PhotoFlow) | NeurIPS 2026 | 43★ | Agentic 3D 虚拟摄影任务 —— 管的是**拍摄前**，不是后期 |

**只有论文没找到开源代码的**：
- **RetouchAgent**（AAAI 2026，*Towards Interactive and Explainable Image Retouching with MLLM Agents*）—— 搜索未发现官方仓库
- **PhotoArtAgent**（[arXiv:2505.23130](https://web3.arxiv.org/pdf/2505.23130)，*Intelligent Photo Retouching with Language Model-Based Artist Agents*）—— 同样未找到官方实现

**论文列表（找选题用）**：
- [ATH-MaaS/Awesome-Agentic-Visual-Creation](https://github.com/ATH-MaaS/Awesome-Agentic-Visual-Creation)（18★，含 image/video 生成与编辑的 Agent 论文合集）
- [YinmingHuang/Awesome-agentic-visual-generation-model](https://github.com/YinmingHuang/Awesome-agentic-visual-generation-model)

---

## 三、路线 B：MCP / 工具桥接 —— 目前最实用的一条路

思路不是"训练一个修图模型"，而是**把专业修图软件做成 Agent 的工具**，让 Claude / Cursor / Codex 这类通用 Agent 直接调用。

### Lightroom Classic 阵营

| 项目 | 星标 | 语言 | 说明 |
|---|---|---|---|
| [Automaat/lightroom-mcp](https://github.com/Automaat/lightroom-mcp) | **121★** | Lua | 生态里最活跃，MIT，2026-10 仍在更新，29 fork |
| [noopz/lightroom_mcp](https://github.com/noopz/lightroom_mcp) | 15★ | Lua | Apache-2.0 |
| [synthet/lightroom-mcp](https://github.com/synthet/lightroom-mcp) | 8★ | Python | master 分支 |
| [varunkumar/lightroom-mcp](https://github.com/varunkumar/lightroom-mcp) | 6★ | Python | — |
| [4xiomdev/lightroom-classic-mcp](https://github.com/4xiomdev/lightroom-classic-mcp) | — | — | macOS 专用 MCP bridge |
| [par4987/lightroom-mcp](https://github.com/par4987/lightroom-mcp) | 1★ | Lua | 覆盖 catalog/develop/masks/previews，**带"灰尘检测 + 修图建议"的 agent skill** |
| [MichalCervenansky/lightroom-mcp](https://github.com/MichalCervenansky/lightroom-mcp) | — | — | 带一篇实战 blog post，值得读 |
| [drshy-org/lightroom-py](https://github.com/drshy-org/lightroom-py) | — | Python | 非官方 Python 库 + CLI + **Claude/Codex agent skill** |

### 全开源栈阵营（不依赖 Adobe）

| 项目 | 说明 |
|---|---|
| [chipi/chemigram](https://github.com/chipi/chemigram) | 自我定位："**Chemigram is to photos what Claude Code is to code**"。通过 MCP 在 **darktable** 上做 Agent 驱动的修图。有 docs/prd 目录，工程化程度看起来不错 |
| [joshua5201/mcp-photo-edit](https://github.com/joshua5201/mcp-photo-edit) | GPL-3.0，Python。"MCP server for agent-driven Lightroom-like RAW photo editing with **RawTherapee**" |
| [w1ne/darktable-mcp](https://glama.ai/mcp/servers/w1ne/darktable-mcp) | darktable MCP server |
| [maorcc/gimp-mcp](https://github.com/maorcc/gimp-mcp) / [gimp-mcp-server](https://pypi.org/project/gimp-mcp-server/) | GIMP 侧 MCP |
| [alisaitteke/photoshop-mcp](https://github.com/alisaitteke/photoshop-mcp) | Photoshop MCP，README 有中文版 |
| [Focus-GTS/firefly-services-mcp](https://github.com/Focus-GTS/firefly-services-mcp) | 把 Adobe Firefly / Photoshop API / Lightroom API 暴露成 MCP 工具 |

> **重要信号**：darktable 官方文档（development 版）已经出现 `special-topics/program-invocation/darktable-mcp` 页面 —— 说明**开源修图软件正在把 MCP 当作一等公民接口来做**。这是这个方向最值得盯的趋势。

### 国产 / 中文项目

| 项目 | 星标 | 说明 |
|---|---|---|
| [jeremywei201-tech/camera-raw-agent](https://github.com/jeremywei201-tech/camera-raw-agent) | 15★ | TypeScript，"Your best assistant for photography retouching"，另有 release 仓库 |
| [shuhaolin63-hash/photo_agent](https://github.com/shuhaolin63-hash/photo_agent) | 6★ | MIT，Python。"你的私人摄影指导团队"，**多 Agent 架构**，topics 含 claude-code / lightroom / multi-agent |
| [zzzhhha/retouchflow-ai](https://github.com/zzzhhha/retouchflow-ai) | 1★ | MIT，Python + FastAPI。面向 **Lightroom Classic + Photoshop + 本地像素处理** 的修图流程助手 |
| [wmk233/photo-retouch-agent](https://github.com/wmk233/photo-retouch-agent) | 1★ | Apache-2.0，Python。自动 P 图美化，支持自定义 |
| [John-owo/photo-agent](https://github.com/John-owo/photo-agent) | 1★ | MIT，TypeScript。摄影工作流 CLI，**可追溯的编辑会话 + 撤销恢复** + 可选 Lightroom MCP 后端 |
| [jiasongqi/ai-retouch-agent](https://github.com/jiasongqi/ai-retouch-agent) | 0★ | Python，2026-09 新建 |

---

## 四、路线 C：Agent Skills / 提示词包（零训练，最轻）

| 项目 | 星标 | 说明 |
|---|---|---|
| [tuozhekongqi/photo-skills](https://github.com/tuozhekongqi/photo-skills) | 5★ | **Photo Skills R8**，中文。三合一：真实摄影后期 + Photo Zine 编辑设计 + 人像真实处理。专为 AI Agent 工作流设计的技能包 |
| [drshy-org/lightroom-py](https://github.com/drshy-org/lightroom-py) | — | 同时提供 Claude/Codex agent skill |

这类项目的价值是**把修图经验沉淀成可复用的 prompt/技能资产**，缺点是质量参差、缺少评测。

---

## 五、路线 D：相邻环节 —— AI 选片 / 评分（最成熟的一段）

摄影后期流水线的前置环节，开源项目质量明显更高：

| 项目 | 星标 | 说明 |
|---|---|---|
| [ncoevoet/facet](https://github.com/ncoevoet/facet) | **256★** | 本地 AI 照片打分/选片/图库，人脸识别 + 语义搜索，**无云无订阅**。技术栈 PyTorch + CLIP + TOPIQ + FastAPI + Angular |
| [ChrisChen667788/pixcull](https://github.com/ChrisChen667788/pixcull) | **120★** | MIT，本地优先。**六维评分标准**，导出 XMP/IPTC，Lightroom 和 Capture One 可直接读 |
| [RawLabo/QuickRawPicker](https://github.com/RawLabo/QuickRawPicker) | 69★ | C 写的 RAW 选片器，兼容 Adobe/ darktable XMP 与 RawTherapee PP3 |
| [duartebarbosadev/PhotoSort](https://github.com/duartebarbosadev/PhotoSort) | 25★ | Apache-2.0，重复照片查找 + 相似度检测 + 自动旋转 |
| [Yuumi0221/Cullumi](https://github.com/Yuumi0221/Cullumi) | 26★ | 离线便携 Windows 画质/相似度筛选 |

> 另外 `XIXIJCrG/PhotoTriage-AI`（本地优先的 JPG/PNG + RAW AI 选片桌面应用，兼容 OpenAI 接口模型，CSV/XMP 工作流）也在同一赛道。

---

## 六、空白与机会（如果你想自己做）

调研下来，明显没人做好的几件事：

1. **没有面向个人的"开箱即用"端到端方案**。学术项目要 A100，工具项目要你自己拼装 MCP + 提示词 + 工作流。
2. **没有"风格一致性"能力**。所有项目都是单张图的一次性编辑，**没有跨整组照片保持统一色调/风格**的 Agent —— 这是婚礼、商业摄影最痛的点。
3. **没有闭环评估**。JarvisArt 建了 MMArt-Bench，但工具层项目全都没有客观评测，"修得好不好"只能人眼看。
4. **XMP 侧车文件是最大的接口红利**。pixcull、QuickRawPicker 都靠 XMP 打通了 Lightroom/darktable/RawTherapee/Capture One。**只写 XMP、不碰像素**的非破坏式 Agent，兼容性成本最低、天花板最高 —— 目前没人专门做这个。
5. **批量 + 可回滚的工作流缺失**。只有 `John-owo/photo-agent` 提到了"可追溯的编辑会话 + 恢复"。
6. **中文语境下的审美偏好**（肤色处理、日系/胶片调）几乎没有专门的数据集和评测。

### 如果要从零起步，建议路径

```
先做 MCP 工具层（接 darktable 或 Lightroom）
  → 用 XMP 侧车做非破坏式写入
  → 加"整组风格一致性"约束（这是空白点）
  → 用 MMArt-Bench 或自建小评测集做客观打分
```

技术选型上，**MCP + XMP + 通用大模型**的组合，比"训练一个 7B 修图模型"的性价比高一个数量级。

---

## 七、快速上手建议

| 你的目标 | 直接去看 |
|---|---|
| 理解这个方向能做成什么样 | JarvisArt 论文（[ar5iv 全文](https://ar5iv.labs.arxiv.org/html/2506.17612)） |
| 今天就想让 AI 帮我修图 | [Automaat/lightroom-mcp](https://github.com/Automaat/lightroom-mcp) + Claude/Cursor |
| 完全开源、不碰 Adobe | [chipi/chemigram](https://github.com/chipi/chemigram)（darktable）或 [mcp-photo-edit](https://github.com/joshua5201/mcp-photo-edit)（RawTherapee） |
| 先解决选片痛点 | [facet](https://github.com/ncoevoet/facet)、[pixcull](https://github.com/ChrisChen667788/pixcull) |
| 找选题 / 追前沿 | [Awesome-Agentic-Visual-Creation](https://github.com/ATH-MaaS/Awesome-Agentic-Visual-Creation) |

---

## 附：未能完全核实的信息

- `chipi/chemigram`、`drshy-org/lightroom-py`：调研期间 GitHub API 出现 DNS 解析异常，星标数与最近更新时间未取到，描述来自 GitHub 页面与搜索结果
- **RetouchAgent**（AAAI 2026）：论文确认存在，未找到官方代码仓库
- **PhotoArtAgent**：论文确认存在（arXiv:2505.23130），未找到官方代码仓库
- 星标数为快照值，且调研期间部分请求受限，仅供参考量级
