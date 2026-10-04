# Lightroom 摄影后期 Agent —— 架构选型方案

> 目标：做一个能操控本机 Lightroom 完成摄影后期作品处理的 Agent
> 三种工作模式：① 自主决策（用户什么都不说）② 预置风格方向（大师风格）③ 对话交互
> 本文回答的核心问题：**该用什么架构**

---

## 一、先诊断：为什么 pi 这类 coding agent 架构不适用

你的直觉是对的，而且理由比"场景不同"更具体。`pi`（[enjoyZhou/pi](https://github.com/enjoyZhou/pi)、[nktkt/pi](https://github.com/nktkt/pi)，earendil-works/pi 的 Rust 移植）本质是一个 **coding agent harness**，它的设计前提是：

| Coding agent 的前提 | 摄影后期的现实 | 后果 |
|---|---|---|
| 状态是**文本**，可 diff、可回滚 | 状态是 **~175 维连续参数向量 + 一张图** | 没有 diff 可言，编辑历史管理要自己设计 |
| 反馈是**符号化的**（编译错误、测试通过） | 反馈是**感知的**（好不好看、风格像不像） | 需要一个视觉评审器，这是全新的一层 |
| 任务长程，动辄 200 步 | 任务短程，**5~15 次工具调用**就够 | 重型 ReAct 脚手架是纯浪费，还增加延迟和成本 |
| 试错**昂贵**（改错代码要调试） | 试错**几乎免费**（Lightroom 非破坏式，一键回退） | 应该**大胆试错**，而不是小心翼翼先规划 |
| 无空间概念 | 局部调整需要**空间 grounding**（人脸、天空、主体） | 需要视觉原语，coding agent 完全没有 |
| 工具语义明确（读文件、跑命令） | 工具语义模糊（"Contrast = 7" 到底是多少？） | LLM 对绝对数值**没有校准感** |

**一句话结论**：coding agent 是"想清楚再动手"，摄影后期应该反过来 —— **"先动手再纠正"**，因为这里的反馈信号既便宜又可靠。

---

## 二、决定架构的三个领域约束

这三条是整个选型的地基，任何方案都必须满足：

### 约束 1：反馈信号便宜、可测量，但需要"渲染"这一步

Lightroom 非破坏式编辑 → 写入参数 → 渲染预览 → 拿回一张 JPEG。这个闭环是可能的，但**每次渲染有秒级延迟**。

→ **推论**：闭环轮次必须少（≤3 轮），且候选方案要**并行**探索。这直接否定了"200 步 ReAct 慢慢磨"的设计。

### 约束 2：参数空间连续、稀疏、物理纠缠

曝光/高光/白色阶/阴影互相影响；`Contrast2012 = 7` 对 LLM 是无意义数字。

→ **推论（本方案最重要的一条）**：**绝不让 LLM 直接输出 175 个数值。** 必须做"语义意图 → 参数增量"的分层映射。详见第六节。

### 约束 3：风格是一个"分布"，不是一个 prompt

"安塞尔·亚当斯风格"不是一个字符串能表达的。它是：
- 一组锚定参数（anchor params）
- 每类原片上的适配规则（同一风格用在欠曝片和过曝片上，参数必然不同）
- 允许变化的范围（param ranges）与禁止越界的约束（negative constraints）

→ **推论**：风格必须**参数化、可插值、可评测、可版本化**。这样它才可复用、可混合（"70% 亚当斯 + 30% 日系"）、可被 critic 打分。

---

## 三、推荐架构：分层闭环（Evaluator-Optimizer）

这是主推方案。形式化描述：

```
Agent(photo) =
    P  = Perceive(photo)                      # 感知：结构化理解原片
    S  = Route(P, user_signal)                # 路由：自主 / 预置风格 / 对话
    θ₀ = Anchor(S) ⊕ Adapt(P, S)              # 风格基线 ⊕ 原片适配

    for k in 1..K:                            # K ≤ 3
        θₖ = Plan(θₖ₋₁, P, S, feedback)       # 参数增量规划（不是绝对值）
        Apply(θₖ);  Iₖ = Render()             # 写入 Lightroom + 取回预览
        (tech, aes, refl) = Critique(photo, Iₖ, θₖ, S)
        if pass(tech, aes): break             # 通过就停
        feedback = refl                       # 否则反思修正（Reflexion 式）

    Commit(θ);  Remember(P, θ, tech, aes)     # 提交 + 记忆
```

**为什么是这个而不是别的**：这正是 Anthropic 在 [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) 里说的 **evaluator-optimizer** 模式的适用场景 —— "**有明确的评价标准，且迭代反馈确实能带来改进**"。摄影后期两条都满足：技术指标是确定性的（直方图溢出可以程序化检测），美学是有模型可打分的。

### 分层展开

```
┌─────────────────────────────────────────────────────────┐
│  L5  记忆与提交层                                         │
│      风格库 │ 案例记忆 │ 用户偏好记忆 │ 提交/导出/XMP        │
└─────────────────────────────────────────────────────────┘
                          ▲ ▼
┌─────────────────────────────────────────────────────────┐
│  L4  评审层 ★架构心脏★                                    │
│      技术评审（确定性规则）  +  美学评审（VLM-as-Judge）      │
│      输出：{technical_pass, style_match, improvement,       │
│             artifacts[], reflection}                     │
└─────────────────────────────────────────────────────────┘
                          ▲ ▼
┌─────────────────────────────────────────────────────────┐
│  L3  执行层                                               │
│      参数快照 │ 幂等写入 │ 触发渲染 │ 取回降采样预览(1024px)  │
└─────────────────────────────────────────────────────────┘
                          ▲ ▼
┌─────────────────────────────────────────────────────────┐
│  L2  决策层（三模式共享同一 Planner）                       │
│      路由 Router → 风格锚定 Anchor → 参数增量 Plan          │
└─────────────────────────────────────────────────────────┘
                          ▲ ▼
┌─────────────────────────────────────────────────────────┐
│  L1  感知层                                               │
│      场景分类 │ 技术指标 │ 美学打分 │ 区域 grounding         │
│      输出结构化 JSON（不是自然语言）                         │
└─────────────────────────────────────────────────────────┘
                          ▲ ▼
┌─────────────────────────────────────────────────────────┐
│  L0  工具层（Lightroom Bridge）  ← 双通道                  │
│      A) Lua 插件 + LrSocket TCP（主通道，可读可写）          │
│      B) XMP sidecar 写入（旁路，批处理/无 UI 依赖/兜底）     │
└─────────────────────────────────────────────────────────┘
```

---

## 四、L0 工具层：Lightroom 怎么操控（这是最容易踩坑的地方）

### 双通道设计

| 通道 | 机制 | 优势 | 劣势 |
|---|---|---|---|
| **A. Lua 插件 + LrSocket**（主） | Lightroom 插件常驻，通过 **localhost TCP** 与 Agent 进程通信 | **可读回参数**（闭环基础）、实时、可应用预设、可导出 | 需装插件、需 Lightroom 运行、要处理自动重连 |
| **B. XMP sidecar**（旁路） | 外部写 `.xmp`，Lightroom "从文件读取元数据" | 无 UI 依赖、可离线批处理、跨软件（C1/darktable/RawTherapee 都认） | 需用户手动触发读取（或开启自动写 XMP）、**无法读回** |

**必须两条都做**。A 负责交互式闭环，B 负责批处理兜底。已有实现可参考：
- [Automaat/lightroom-mcp](https://github.com/Automaat/lightroom-mcp)（121★，Lua，MIT）—— TCP socket + auto-reconnect + 双端口模型
- [4xiomdev/lightroom-classic-mcp](https://github.com/4xiomdev/lightroom-classic-mcp) —— 有完整的 `docs/ARCHITECTURE.md`，暴露 **~175 个 develop settings**，且有 `select_mask_tool`（**说明蒙版是可以程序化控制的**）
- [synthet/lightroom-mcp](https://github.com/synthet/lightroom-mcp) —— 有 `SDK_INTEGRATION.md`

### 工具粒度：三层封装，不要暴露 175 个原子工具

这是决定 Agent 能不能用好的关键设计：

```
L1 原子层（给系统用，不给 LLM 直接选）
    get_develop_settings(photo_id) -> dict
    apply_develop_settings(photo_id, dict)
    render_preview(photo_id, max_size=1024) -> image
    get_histogram(photo_id) -> stats

L2 语义组（LLM 可选）
    exposure_group / tone_group / color_group / detail_group / geometry_group
    apply_preset(name)
    set_mask(region, adjustments)

L3 风格动作（LLM 首选）
    apply_style_direction(name, strength=0.0~1.0)
    match_reference_style(reference_image)
    auto_develop()          # 自主模式入口
    diagnose()              # 只分析不改
```

**理由**：LLM 面对 175 个扁平工具会迷失、参数爆炸、且无法建立"曝光和高光是一组"的直觉。三层封装把选择空间压到 LLM 能处理的规模。

### Lightroom 侧的现实约束（提前知道能省很多时间）

1. **能力边界**：SDK 能读写 develop settings、应用预设、导出；但 **AI 蒙版、生成式移除等新功能没有 API**，需要 GUI 兜底（computer-use）或干脆不支持。
2. **渲染是延迟瓶颈**：每次写参数 → Lightroom 渲染 → 导出预览，秒级。→ 闭环轮次 ≤3，候选并行。
3. **预览必须降采样**：用 1024px JPEG 送回模型，不要用原图。成本差一个数量级。
4. **Windows 下优先 localhost TCP**（LrSocket 原生支持），不要用文件轮询。
5. **必须做参数快照**，任何一轮 critic 不通过就回滚。

---

## 五、L1 感知层：用专用小模型，不用一个通用 VLM 硬扛

**反模式**：把原片丢给 GPT-4o 问"这张照片有什么问题"。
**正确做法**：并行跑多个专用组件，输出**结构化 JSON**。

```
Perceive(photo) -> {
  scene: "landscape|portrait|street|stilllife|architecture",
  subject_boxes: [{label, bbox, confidence}],      # Grounding DINO / SAM 类
  tech: {
    exposure_delta_ev, wb_shift, highlight_clip_pct,
    shadow_clip_pct, noise_level, sharpness, histogram_stats
  },
  aesthetic: { score, model, dims: {composition, lighting, color, impact} },
  references: [similar_cases_from_memory]           # 检索到的相似历史案例
}
```

组件选型建议：
- **场景/主体**：轻量 VLM 或专用分类器 + Grounding DINO（JarvisArt 论文里也用 Grounding DINO 做区域定位，confidence > 0.8）
- **技术指标**：纯程序化，用 OpenCV/numpy 算直方图、溢出比例、白平衡偏移。**这部分不要让 LLM 做** —— 测量比猜测准得多（这条也否定了纯 Plan-and-Execute 架构）
- **美学打分**：NIMA / TOPIQ / CLIP-IQA / Q-Align 这类 no-reference 质量模型。**这是闭环的奖励信号来源**

---

## 六、L2 决策层 ★最重要的一节★

### 核心原则：LLM 输出语义，确定性层翻译成数值

不要让 LLM 说 `{Contrast2012: 7, Highlights2012: -71}`。
让 LLM 说：

```json
{
  "actions": [
    {"op": "lift_shadows",   "amount": "moderate", "target": "global"},
    {"op": "add_contrast",   "amount": "slight",   "target": "global"},
    {"op": "cool_highlights","amount": "slight",   "target": "sky_mask"},
    {"op": "reduce_clarity", "amount": "slight",   "target": "skin_mask"}
  ]
}
```

然后由一个**确定性的参数映射层**翻译成具体数值区间：
- 方式 A：规则表 + 场景系数（简单、可控、可解释，**建议先做这个**）
- 方式 B：原片→目标片成对数据训练的参数回归器（[Neural Preset for Color Style Transfer, CVPR 2023](https://openaccess.thecvf.com/content/CVPR2023/papers/Ke_Neural_Preset_for_Color_Style_Transfer_CVPR_2023_paper.pdf) 就是这条路）
- 方式 C：A 打底、B 兜底 —— 回归器给锚点，LLM 只做语义级修正

**收益**：
1. 约束了输出空间，杜绝"LLM 瞎报数字"
2. 参数可解释（"阴影提太多导致发灰"这个反思能对应到具体参数）
3. 风格库可以被人类读懂、手改、版本化
4. 回归器路线能**抬高自主模式的质量下限**

### 风格库的数据结构

```python
StyleDirection = {
    "id": "ansel_adams_zone_system",
    "display_name": "安塞尔·亚当斯 · 区域曝光",
    "scene_affinity": ["landscape", "architecture"],
    "anchor_params": {...},              # 基准参数向量
    "param_ranges": {...},               # 允许浮动范围
    "constraints": [...],                # 必须满足（如：天空不能溢出）
    "negative_constraints": [...],       # 禁止（如：不要加暗角）
    "description": "...",                # 描述，给 LLM 理解用（不是用来执行的）
    "reference_images": [...],           # 参考图，给风格匹配打分用
    "version": "1.2.0"
}
```

**注意 `description` 的定位**：它是**给 LLM 理解风格意图**用的（帮助做适配性微调），**不是**风格的实现。风格实现永远是 `anchor_params`。这就是"风格是参数向量，不是 prompt"。

### 三种模式如何收敛到同一个 Planner

| 模式 | 输入 | Planner 做什么 |
|---|---|---|
| **自主** | 只有原片 | 场景分类 → 路由选 1~2 个候选风格 → 生成候选参数 → **并行评估，选最优** |
| **预置风格** | 原片 + `style_id` | 取 anchor_params → 做**适配性微调**（这是重点：同一风格在不同原片上参数必须不同） |
| **对话** | 原片 + 自然语言 | NL → 语义动作序列 → 同参数映射层 |

三者的**下游完全一致**，只是候选生成策略不同。自主模式用 **Routing + Parallelization**（生成 2~3 个候选并行评估），比让一个大模型自由发挥要稳得多。

---

## 七、L4 评审层：整个架构的心脏

**双评审并行，不要只用一个 VLM**：

### 技术评审（确定性，不用 LLM）
```python
def technical_critique(before_stats, after_stats, constraints):
    issues = []
    if after_stats.highlight_clip_pct > 1.0: issues.append("高光溢出")
    if after_stats.shadow_clip_pct > 5.0:    issues.append("阴影死黑")
    if skin_hue_shift(before, after) > 0.05: issues.append("肤色偏移")
    for c in constraints:
        if not c.check(after_stats): issues.append(f"违反约束: {c.name}")
    return issues
```

### 美学评审（VLM-as-Judge）
输入：原片 + 当前渲染图 + 目标风格描述 + 参数语义动作
输出：
```json
{
  "style_match": 0.0-1.0,
  "improvement": -1.0-1.0,       # 相对原片是变好还是变差
  "overcooked": true/false,       # 是否过火（后期最常见的失败模式）
  "reflection": "阴影提得过多，画面发灰，建议回调并改用局部提亮"
}
```

### 循环控制
- **最多 3 轮**（受渲染延迟约束）
- **单调性约束**：新一轮总分不得低于上一轮，否则回滚到上一轮参数
- 第 1 轮后就通过 → 直接提交（大多数简单片应该 1 轮过）

**为什么这比"多 Agent 团队互相讨论"好**：固定循环 + 专职 critic 的成本是可预测的（K ≤ 3 次渲染 + K 次 VLM 调用）。而 CrewAI/AutoGen 那种"摄影师 Agent 和修图师 Agent 对话"的模式，轮次不可控、容易互相说服、token 成本爆炸，且**没有引入任何新信息**。

---

## 八、备选架构对比（以及为什么没选）

| 架构 | 优点 | 致命问题 | 结论 |
|---|---|---|---|
| **ReAct 单 Agent** | 简单、好实现 | 无视觉校验，会"自信地调错"；175 维空间盲走效率极低 | ❌ 做原型可以，别做产品 |
| **Plan-and-Execute** | 规划清晰、可解释 | 计划质量依赖对原片的准确描述，而"欠曝多少"应该**测量**而非**猜测** | ❌ 纯规划浪费了确定性测量能力 |
| **多 Agent 团队**（摄影师+修图师+评论家） | 有"讨论"看起来很智能 | 轮次不可控、成本爆炸、无新信息增益 | ❌ 用固定 evaluator-optimizer 更省 |
| **Computer-use / GUI Agent**（点 UI） | 能覆盖 SDK 覆盖不到的功能 | 慢、脆、不可靠、截图成本高 | ⚠️ **仅作兜底**，用在 AI 蒙版等无 API 操作上 |
| **Evaluator-Optimizer 闭环** | 反馈便宜可靠、成本可预测、可直接优化 | 需要自己设计 critic 和状态管理 | ✅ **推荐主架构** |
| **端到端训练小模型**（JarvisArt 路线） | 效果上限最高 | 需 8×A100 训练 + 数据集构建 | ⚠️ 除非要做产品级护城河，否则不划算 |

> **参考**：JarvisArt（NeurIPS 2025，[arXiv:2506.17612](https://ar5iv.labs.arxiv.org/html/2506.17612)）走的是端到端训练路线（Qwen2.5-VL-7B + CoT SFT + GRPO-R，200+ Lightroom 操作，自建 MMArt-55K/MMArt-Bench）。如果你的团队没有 A100 集群，**不要复现它的训练，但一定要借鉴它的三个设计**：CoT 推理结构、A2L 通信协议、多维奖励设计。

---

## 九、需要关注的前沿架构（直接对应你的场景）

| 工作 | 为什么相关 |
|---|---|
| **IMAGAgent**：*Orchestrating Multi-Turn Image Editing via Constraint-Aware Planning and Reflection*（[ar5iv 2603.29602](https://ar5iv.labs.arxiv.org/html/2603.29602)） | **多轮图像编辑 + 约束感知规划 + 反思**，几乎就是你要的骨架 |
| **EditRefiner**：*A Human-Aligned Agentic Framework for Image Editing Refinement*（[ar5iv 2605.07457](https://ar5iv.labs.arxiv.org/html/2605.07457)，代码 [IntMeGroup/EditRefiner](https://github.com/IntMeGroup/EditRefiner)） | 显式的 **EvaluationAgent**，给 具体/美学/保真 三个维度加权打分（`s = 0.3·s_v + 0.4·s_e + 0.3·s_p`）。可直接借鉴它的 critic 设计 |
| **MAGiC**：LLM 驱动的多智能体视觉创作框架（ECAI 2025） | 视觉创作领域多 Agent 编排的参考实现 |
| **PerTouch**（AAAI 2026，[Auroral703/PerTouch](https://github.com/Auroral703/PerTouch)） | **个性化**语义修图 Agent —— 对应你的"用户偏好记忆"模块 |
| **EyeControl**（ECCV 2026，[DragonisCV/EyeControl](https://github.com/DragonisCV/EyeControl)） | 意图驱动的视觉焦点增强，对应"主体突显"这类语义目标 |
| **Neural Preset**（CVPR 2023） | 预设参数预测的权威方法，对应第六节的"参数回归器"路线 |

---

## 十、技术栈选型建议

### 编排框架：不要用重量级多 Agent 框架

**理由**：AutoGen / CrewAI 把"多个角色对话"当作核心抽象，而你的核心抽象是 **"参数状态 + 视觉反馈"**，两者会打架 —— 你会花大量精力把状态塞进"对话"这个不合适的容器里。

| 选项 | 评价 |
|---|---|
| **LangGraph** | ✅ **推荐**。显式状态图，天然适合有环的 evaluator-optimizer；内置 checkpoint 持久化、可中断、可人工介入（"改完先给我看看"这个功能它直接支持） |
| **Pydantic AI** | ✅ 轻量、类型安全，适合工具调用密集的场景。如果团队偏 Python 且想少抽象，选它 |
| **自写状态机** | ✅ 也合理。这个任务的状态机就 6~8 个节点，引入框架的收益可能还不如自己写清楚。**如果你只做单机工具，我倾向这个** |
| AutoGen / CrewAI | ❌ 抽象不匹配 |
| pi / coding agent harness | ❌ 见第一节 |

### 其他
- **工具协议**：**MCP**。理由：能直接复用已有的 lightroom-mcp 生态，且未来接入别的修图软件（darktable/RawTherapee）零成本
- **IPC**：localhost TCP（LrSocket 原生支持），配自动重连
- **状态/记忆**：SQLite（案例记忆）+ 文件版本化的 JSON（风格库）
- **预览**：1024px JPEG，**降采样送模型**是成本控制的关键
- **模型分工**：轻量模型做分类/路由，强模型做规划/反思，专用模型做美学打分。**不要一个模型全包**

---

## 十一、分阶段落地路线

| 阶段 | 目标 | 关键验证点 |
|---|---|---|
| **P0** | 打通 Lightroom 通道 | Lua 插件 + LrSocket 能 `get/apply develop settings` 且**能读回**、能取回预览 |
| **P1** | 感知层 | `Perceive()` 输出的技术指标与真实情况一致（拿直方图人工核对） |
| **P2** | 单风格闭环 | 一个风格 + critic 闭环，能在 3 轮内收敛；**验证渲染延迟可接受** |
| **P3** | 三模式 | 自主路由 + 预置风格 + 对话，共享 Planner |
| **P4** | 风格库扩充 | 大师风格参数化 + 版本管理 + 风格混合（"70% A + 30% B"） |
| **P5** | 记忆与个性化 | 从用户手动修改反推偏好，调整风格基线（这是商业化的差异点） |
| **P6** | XMP 批处理通道 | 无插件环境下的降级方案 |

**建议先做 P2**：一个风格、一张照片、闭环跑通。这一步会暴露 80% 的架构问题（延迟、参数映射、critic 质量）。

---

## 十二、三条最关键的工程判断（如果只记三条）

1. **不要让 LLM 直接输出 175 个参数。** 用"语义动作 → 确定性参数映射"分层。这是可控性的唯一保障。
2. **架构核心是 Critic 闭环，不是 Planner。** 因为领域特性是"试错便宜 + 反馈可测"，闭环的收益远大于更好的规划。
3. **风格是带约束的参数向量，不是 prompt。** 只有这样风格才能被评测、插值、混合、版本化。

---

## 附：参考资料

**架构范式**
- [Anthropic — Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)（evaluator-optimizer / orchestrator-workers 等模式）
- [Common workflow patterns for AI agents and when to use them](https://claude.com/blog/common-workflow-patterns-for-ai-agents-and-when-to-use-them)
- Reflexion: Language Agents with Verbal Reinforcement Learning（[arXiv:2303.11366](https://browse.arxiv.org/pdf/2303.11366)）
- [LangGraph vs Pydantic AI (2026)](https://www.respan.ai/market-map/compare/langgraph-vs-pydantic-ai)

**图像编辑 Agent**
- JarvisArt（NeurIPS 2025）— [arXiv:2506.17612](https://ar5iv.labs.arxiv.org/html/2506.17612) / [GitHub](https://github.com/LYL1015/JarvisArt)
- IMAGAgent — [ar5iv 2603.29602](https://ar5iv.labs.arxiv.org/html/2603.29602)
- EditRefiner — [ar5iv 2605.07457](https://ar5iv.labs.arxiv.org/html/2605.07457) / [GitHub](https://github.com/IntMeGroup/EditRefiner)
- PerTouch（AAAI 2026）— [GitHub](https://github.com/Auroral703/PerTouch)
- EyeControl（ECCV 2026）— [GitHub](https://github.com/DragonisCV/EyeControl)
- MAGiC（ECAI 2025）— [介绍](https://mp.weixin.qq.com/s/_kbpZqdv9mC_O5JZMnx8xg)

**Lightroom 控制**
- [Automaat/lightroom-mcp](https://github.com/Automaat/lightroom-mcp)（LrSocket 双端口 + TCP 自动重连，[解析](https://zread.ai/Automaat/lightroom-mcp/7-lrsocket-binding-and-dual-port-model)）
- [4xiomdev/lightroom-classic-mcp](https://github.com/4xiomdev/lightroom-classic-mcp)（~175 develop settings，含 `select_mask_tool`，[工具列表](https://glama.ai/mcp/servers/4xiomdev/lightroom-classic-mcp)）
- [synthet/lightroom-mcp](https://github.com/synthet/lightroom-mcp)（SDK_INTEGRATION.md）
- [Jovenjr/lightroom-sdk-docs](https://github.com/Jovenjr/lightroom-sdk-docs)（SDK 文档镜像）
- [Jaid/lightroom-sdk-8-examples](https://github.com/Jaid/lightroom-sdk-8-examples)（官方样例）
- [shuangye/lightroom-auto-develop](https://github.com/shuangye/lightroom-auto-develop)（Lua 自动修图插件示例）

**风格参数化**
- Neural Preset for Color Style Transfer（CVPR 2023）— [PDF](https://openaccess.thecvf.com/content/CVPR2023/papers/Ke_Neural_Preset_for_Color_Style_Transfer_CVPR_2023_paper.pdf)
- PPR10K 数据集（人像修图成对数据）

---

## 附：未能核实的信息

调研期间 GitHub / arxiv / ar5iv 等域名出现间歇性 DNS 解析异常，以下内容基于检索摘要而非原文核实：
- `4xiomdev/lightroom-classic-mcp` 的"~175 develop settings"与 `select_mask_tool` 来自 MCP 目录站点的工具描述
- `IMAGAgent`、`EditRefiner` 的方法细节来自标题与检索摘要，建议落地前读原文
- `Automaat/lightroom-mcp` 的 LrSocket 双端口模型来自第三方代码解析站点
