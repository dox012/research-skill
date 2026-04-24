---
name: survey
description: 对一个研究主题做快速背景调研——找 SOTA、benchmark、常见方法、gap，产出结构化 Markdown survey 供 /idea 或 /research 引用。不做完整文献综述，只做"够用的坐标系"（15-20 分钟级别的摸底），3-8 条高信号来源即止。默认中文。 | Quick background survey for a research topic — finds SOTA / benchmarks / method families / gaps from 3-8 sources, produces a structured markdown survey to inform /idea or /research. Not a full lit review.
---

# /survey — 研究主题背景调研

用户主题：**$ARGUMENTS**

本 skill 做**快速摸坐标**，不做完整文献综述。目标是让使用者对领域"有个轮廓"，足以下一步去 `/idea` 或 `/research`。

## 默认语言

topic 中文 → 中文产出；英文 → 英文。

## 流程

### Step 1 — 主题检查

如果 $ARGUMENTS 太宽（"AI"、"机器学习"），先用 `AskUserQuestion` 让用户收敛到 1-2 级子领域。例如从"AI"→"LLM 推理优化 / 图像生成 / 多模态 / ..."。

如果已经具体（"vLLM 的 continuous batching"、"Diffusion model 上的 LoRA finetuning"），直接开搜。

### Step 2 — 查询规划

为主题生成 **3-5 条** 查询，覆盖：
- **SOTA 现状**（"X state of the art 2026"、"X benchmark leaderboard"）
- **经典综述**（"X survey 2024"、"X review"）
- **关键方法**（"X methods comparison"、"X architecture recent"）
- **开放问题**（"X open problems"、"X challenges"）

### Step 3 — 搜索 + 抓取

用 `WebSearch` 跑查询。按相关性和权威性挑 **3-8 条**结果用 `WebFetch` 抓全文。优先：
- 综述类 paper / 知名 blog（Anthropic / OpenAI / DeepMind / ACL Anthology）
- 有数字有 benchmark 的（不是纯观点文）
- 1-2 年内的（看"最近进展"）
- 有时也要 1-2 条更老的经典奠基工作

**跳过**：
- 明显营销文 / 非技术综述
- 纯代码仓库 README（除非就是要摸该项目状态）
- 引用了但本身空洞的 listicles

### Step 4 — 综合

在 cwd 写 `survey.md`（如果该文件已存在问用户覆盖/追加/新名）。结构：

```markdown
# <主题> — Survey

> 生成时间：<YYYY-MM-DD>
> 查询深度：浅（N 条来源），不是完整文献综述

## 总览

<2-3 句话>：领域现状一句、主流方法一句、当前最大争议或 gap 一句。

## SOTA 与基准

- **主流 benchmark**：<benchmark 名 + 指标名 + 当前最好数字及方法>
- **公开榜单/排行**：<URL + 当前第一名>
- **怎么自测**：<哪个 dataset / metric 最容易复用>

## 方法分类

按 approach 族分 3-5 类，每类：
- 代表工作（1-2 篇，含 URL）
- 核心思路（1-2 句话）
- 擅长 / 短板

## 关键 Gap / 开放问题

3-5 条，每条一行 + 1 行"为什么难"。

## 来源清单

- [<标题>](<URL>) — <1 句相关性>
- ...

## 后续建议

- 如果想做 benchmark：看 <X>
- 如果想做方法改进：关注 <Y> 线
- 如果想写综述：<Z> 是好起点
```

## 核心约束

- **不复制抄写**：所有引述必须**改写**，别整段粘贴别人的段落。引用数字/比较/具体主张时给来源 URL。
- **不编造引用**：如果搜到的链接抓失败或内容不相关，就不引。宁可来源少也别杜撰。
- **3-8 条**：多了就是走向 full lit review 了，那不是此 skill 的职责。
- **< 500 行**：survey.md 要能一屏翻完找到坐标。长就失去摸底价值。
- **承认不确定**：领域快速变化时，写"截至 2026 年 Q2 看起来 X 是 SOTA，但正在被 Y 挑战"比装确定靠谱。

## 与其它 skill 的关系

- **/idea** 会读同目录下 `survey.md`（如果存在）作为上下文——所以 survey 的"Gap / 开放问题"段对 /idea 产出候选有直接影响
- **/research** bootstrap 时如果 cwd 有 survey.md 会把它放进 CLAUDE.md 的 "背景" 段，让每轮 iteration 都有上下文
- **/paper** 的"相关工作"段可以引用 survey.md 里的来源（但自己再过一遍不要全抄）

## 不做的事

- 不做完整系统性文献综述（> 30 篇）——那是 `/survey --deep` 或 `/paper` 的相关工作段
- 不做代码搜索（用 `gh search code` / `grep`）
- 不调 Claude API 分析论文 PDF——用 WebFetch 能拿到的 abstract / intro 够用了
- 不自动决定"做不做某个方向"——那是 /idea 的职责

## 错误处理

- WebSearch 返回全是营销文：告诉用户，让他们提供更具体的 term 或给几篇种子 paper URL
- 所有 WebFetch 都失败：报告并建议直接用 /idea 跳过 survey
- 主题涉及敏感领域（医疗诊断 / 金融交易策略 / 国防）：照做但在 survey.md 开头提醒"仅学术背景，不构成建议"

## 输出约定

结束时告诉用户：
- survey.md 路径
- 字数 / 来源数
- 3 个最值得深挖的 gap（按杠杆排序）
- 建议下一步（`/idea` 基于 gap 形成问题 / `/research` 直接跑已明确的 baseline / `/survey` 深化某个分支）
