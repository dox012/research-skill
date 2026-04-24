---
name: paper
description: 把一个用 /research 启动的项目的 state/ 目录（research_log、lessons、best 分数、manifest）变成结构化的 Markdown 研究报告。在 cwd 或给定目录里识别 state/，抽取时间线 / 进展表 / lessons 分类 / 瓶颈分析，生成可直接 GitHub 渲染的 MD。不生成 LaTeX、不写代码、不做新实验。 | Generate a Markdown research report from an existing /research project's state/ directory (log, lessons, best scores, manifest).
---

# /paper — 研究报告生成

用户参数（可选）：**$ARGUMENTS**

本 skill 不跑新实验。它把**现有** `/research` 项目的 `state/` 状态汇编成一份 Markdown 报告。

## 默认语言

跟随 `/research` 的约定：项目 state 文件中文为主 → 报告全中文；英文项目 → 英文报告。`$ARGUMENTS` 里要求明确语言 → 用那个。

## 输入识别

$ARGUMENTS 里如果有目录路径（绝对或相对），用那个。否则用 cwd。

必须存在：
- `state/manifest.json`
- `state/research_log.md`
- `state/lessons.md`
- `state/best/<id>.score.json`（每个 active target）

缺任何一个 → 告诉用户不是 /research 项目、请 `cd` 到正确目录。

## 输出

默认写到 `<project-dir>/REPORT.md`（或用户通过 $ARGUMENTS 给的文件名）。如果该文件已存在，问用户是覆盖、追加 `-<日期>` 后缀、还是中止。

## 报告结构（MD 模板）

```markdown
# <研究主题> — Research Report

> 生成时间：<YYYY-MM-DD HH:MM>
> 总轮数：<N>  |  当前 Phase：<phase>

## 摘要

<3–5 句话>：问题是什么、最好分数、用了哪些关键技术、距目标多少。

## 研究设置

- **评测指标**：<从 score.json 字段推断 + CLAUDE.md 里的说明>
- **Target 集**：<N 个，列 id / difficulty / source>
- **工具链**：<tools/ 下的脚本简述>
- **Phase 划分**：<从 CLAUDE.md 提>

## 进展总表

| Target | 初始 score | 最终 score | Δ | 距目标 |
|---|---|---|---|---|
<从 iterations/<id>/001/score.json 与 state/best/<id>.score.json 对比生成>

## 关键突破时间线

| 日期时间 | 轮次 | Target | Δ | 改动 |
|---|---|---|---|---|
<从 research_log.md 提 **只带 ★★/★★★ 的行**（Δ ≥ +0.02）或显式的 ## 段落标题>

## Lessons 精选（按类别）

从 lessons.md 提出**最高价值的 10–15 条**（按三条标准排序：
1. 引起过 ≥+0.005 的 Δ
2. 跨 target / 跨主题可推广
3. 推翻过之前误判）

按分类列出，每条用：**规则（粗体）** + 理由 + 来源迭代 + 一行"什么时候用"

不要 copy-paste 全部 lessons——只挑精华。

## 瓶颈与下一阶段候选

- **当前 residual 分析**：看 best 的 `rmse` / `pixel_match_*` / SSIM 差距，指出还差多少
- **已识别的 Phase 上限**：哪些瓶颈是当前 Phase 能力触不到的（字体、素材、API 依赖）
- **下一阶段候选能力**：如果 Phase N+1 启动会做什么

## 失败轮次回顾

精选 3–5 个有**启发性的失败**（那些推翻了一条 lesson、或揭示了方法论缺陷）。一行描述 + 学到了什么。

简单的"位置调错了再调回去"不要。

## 附录 A：全时间线

研究日志中所有 ★ 行的简洁列表（不含 ✗ 和 steer）。

## 附录 B：工具和基建清单

每个 tool 脚本一句话职责描述 + 行数。基建 hotfix（如 scrollbar fix）单独列。
```

## 流程

1. 检测 state/
2. 扫 manifest（targets/phase/counters）
3. 解析 research_log.md 逐行（parse pattern `YYYY-MM-DD HH:MM | tgt=XXX it=NNN t5=X.XX Δ=±X.XX [★|✗|steer]`）
4. 读 lessons.md 按分类
5. 读 state/best/<id>.score.json 和 iterations/<id>/001/score.json 比对
6. 按模板生成 MD
7. 写文件，stdout 打印路径 + 摘要

## 风格约束

- **不堆砌数据**——报告要读着像文章，不是 JSON dump
- **每节 < 15 行**（附录可长）
- 表格 / 列表 / 粗体优于长段落
- 不编造内容——所有信息都从 state/ 抽
- 指标数字保留 4 位小数（0.1234）
- 日期格式 `YYYY-MM-DD`（ISO）

## 不做的事

- 不跑新实验
- 不改 state（只读）
- 不生成 LaTeX / PDF / HTML（用户需要时自己 pandoc）
- 不写推文 / 营销文案
- 不猜测未来结果

## 错误处理

- `state/` 不存在 → "此目录不是 /research 项目。`cd` 到正确位置或先 `/research <topic>` bootstrap。"
- `research_log.md` 为空或只有 bootstrap 行 → "研究刚开始（N 轮），报告会很短。继续？"
- 数据不一致（manifest 提到 target 但 score.json 缺）→ 警告、用可用数据生成、在报告里标注"部分缺失"

## 输出约定

结束时告诉用户：
- 报告路径
- 总字数 / 节数
- 3 个顶级 lessons 预览
- 建议下一步（`/research` 继续 / `/idea` 想新方向 / 直接提交 paper）
