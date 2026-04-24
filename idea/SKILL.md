---
name: idea
description: 在启动 /research 之前帮用户**把模糊主题收敛成具体可做的研究问题**。输入一个粗糙的方向（例如"提升图像生成质量"、"让 LLM 更会写代码"），产出 3–5 个具体、可证伪、有评测指标、可行性明确的问题候选，每个带 scope 和 why-interesting。不做实验、不跑代码，只做问题形式化。 | Converge a vague research direction into 3–5 concrete, falsifiable problem formulations with metrics and scope — the step before /research.
---

# /idea — 研究问题形式化

用户主题：**$ARGUMENTS**

这个 skill 做一件事：把"我想研究 X"变成"3–5 个可以马上 `/research` 的具体问题"。

## 默认语言

topic 是中文 → 中文回；英文 → 英文。

## 核心反 pattern（最重要的一条）

别产出这种：

> "探索 X 的新方法" / "研究 Y 如何更好" / "设计一个 Z 的框架"

这些不是**问题**，是**主题**。它们无法 /research 因为没指标、没收敛条件。

真正的研究问题长这样：

> "在 CodeContests benchmark 上，能否通过 RAG 把 Claude 3.5 的通过率从 42% 提到 50%+？"

有：指标（pass@1）/ 基准（现状 42%）/ 目标（50%+）/ 约束（在哪评测）。

## 流程

### Step 1 — 快速摸清领域（≤5 分钟）

用 `WebSearch` 或 `WebFetch` 抓 **2–3 条**：
- 这个领域当前 SOTA 大概什么水平、用什么指标
- 有没有公开 benchmark / dataset
- 最近 1–2 年有哪些相关工作方向

**不要**做完整文献综述——目的是让自己对问题空间有个**粗坐标**，不是列参考文献。

如果是纯工程/实验主题（"重建 UI"、"调通我的 pipeline"），跳过这步。

### Step 2 — 和用户做 1–2 轮对话澄清

用 `AskUserQuestion`：
- **时间尺度**：几天内能跑完 / 几周长期 / 无限
- **成功长啥样**：数字达标 / 跑出一个 demo / 搞清楚一个疑问
- **资源约束**：只能跑 local / 可以调 API / 预算多少
- **已有 baseline**（如果有）

1–2 个问题为限。过度审问比错漏信息更烦。

### Step 3 — 产出 3–5 个问题候选

每个候选用这个模板：

```markdown
### 候选 N：<一句话 problem statement>

**形式化**：<用"在 X 上，能否通过 Y 把 A 从 B 提到 C" 或类似结构化表述>

**指标**：<具体数值指标 + 数据来源>

**基线**：<当前水平 + 怎么拿到 baseline>

**为什么值得做**：<1-2 句话——解决谁的痛、回答谁的疑问、有什么 downstream value>

**可行性**：<一句话——需要什么资源、预期多少轮 /research 能收敛、最大技术风险>

**边界**（scope）：<明确写**不**做什么——不做 X、不考虑 Y，这样 /research 才能收敛>
```

### Step 4 — 按三个维度排序

三维打分（每个 1-5）：
1. **Tractability**（能否可行量化评测）
2. **Learnable**（过程能学到多少通用洞见）
3. **Leverage**（做完影响多大——对用户本身的痛、对领域、对下游）

给出推荐排序 + 每个一句话 rationale。

### Step 5 — 交接

最后问用户：
- 选哪个？
- 要不要现在启动 `/research <选中的 problem statement>`？
- 或者 `/idea <refined topic>` 再聚一次？

如果用户选了，**不要自己调 `/research`**——告诉他们命令让他们键入。

## 风格约束

- 每个候选 < 10 行
- 避免"可能"、"或许"、"也许"——要么给具体数字要么不给
- 避免罗列技术词——"用 attention + RAG + finetuning 提升" 是假大空；"在 HumanEval 上 pass@1 从 60% 提 65%" 是真问题
- 每个候选应该足够小，能用一个 `/research` 闭环搞定（几十轮规模）——太大的拆成多个

## 反 pattern（别做）

- 不要产出"探索 X" / "研究 Y" 类主题
- 不要自己执行 `/research`（保持 skill 职责单一）
- 不要生成 > 5 个候选（用户会懵）
- 不要对每个候选写 500 字——每个 10 行为限
- 不要引用"xxx 论文说"除非真的搜到了并且相关（瞎编最糟）

## 与 /research 的关系

`/idea` 产出可以直接粘进 `/research` 的 problem statement。例如：

```
# 用户: /idea 提升 Claude 代码能力
# → /idea 产出 3 个候选
# → 用户选候选 2
# → 用户: /research 在 HumanEval 上，通过 RAG 提升 Claude 3.5 Sonnet pass@1 从 ~75% 到 80%+
```

两个 skill 的分工：
- `/idea`：什么问题值得做（定义阶段）
- `/research`：开闭环闸门持续迭代（执行阶段）

/idea 不接管 research，只交接。
