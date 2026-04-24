---
name: research
description: 在任何有可量化指标的主题上，启动并驱动一个长期自主研究项目。建立项目目录和评测基建，跑"假设→尝试→打分→教训"循环，跨轮累积 lessons，并集成 /loop 自主推进。适用于"研究 X"、"把 X 改到近完美"、"持续尝试 X 并学习"——任何通过 hypothesis→attempt→score→lesson 循环能有效推进的场景。 | Bootstrap and drive a long-term self-improving research project on any topic with a measurable success metric. Builds measurement infrastructure, runs hypothesis→attempt→score→lesson cycles, integrates with /loop for autonomous progress.
---

# /research — 长期自主研究 skill

用户主题：**$ARGUMENTS**

这个 skill 接收一个主题，用结构化迭代推进它。不是一次性回答——它搭基建，然后把研究交给循环。

## 默认语言：中文

除非用户用英文写，或明确要求英文输出：

- `CLAUDE.md`、`lessons.md`、`research_log.md`、`hypothesis.md` 全部用**中文**
- 回复用户用中文
- 代码注释和文件名保持英文（互操作性）
- `$ARGUMENTS` 语言决定默认：以中文为主 → 全中文；以英文为主 → 全英文；混杂时 bootstrap 问一次
- 一旦一个中文项目 bootstrap 完成，后续所有迭代保持中文，即便触发 prompt（`/loop …`）恰好是英文

## 阶段识别（先做这一步）

判断当前是哪种模式：

- **`state/manifest.json` 存在 + 用户消息来自 `/loop` 触发** → ITERATION 阶段（自主），跑 10 步协议
- **`state/manifest.json` 存在 + 用户在跟你对话** → STEERING 阶段（人机协同），见下面 "STEERING" 章节，**不要**自动启新一轮
- **`state/manifest.json` 不存在** → BOOTSTRAP 阶段，走 bootstrap 流程

如果用户主题提到"继续攻 X"、"再跑一轮"这种延续性措辞但没 manifest，问用户是否忘了 `cd` 到已有研究目录。

## BOOTSTRAP 阶段

非平凡任务——除非主题极其狭窄（单个快速问题），否则用 `EnterPlanMode`。Plan mode 让你先摸环境、问用户、对齐范围再搭架子。

### Step 1 — 确定研究范围（用 AskUserQuestion，2–3 个问题为限）

根据 `$ARGUMENTS` 里还不清楚的点，挑以下问：

1. **评测指标（success metric）**——打分函数是什么？例子：
   - 数值型（像素匹配、SSIM、BLEU、准确率）
   - 比较型（另一个模型 / 人类对 A vs B 裁判）
   - 二元里程碑（"这个 test 过了吗"）
   - 定性 + rubric
   
   选**一个**主指标 + 可选次指标。用户不确定时基于主题提议一个让他们确认。

2. **Target 与 Attempt 形态**——每轮输入什么、输出什么？
   - Target：参考 / 输入是什么？（一张图、一个 prompt、一个 codebase、一个 benchmark 集）
   - Attempt：Claude 每轮产出什么？（HTML、Python 代码、文本、通过别的工具生成的图）
   - 如何把 attempt 转成分数？（渲染 + diff、跑测试、人工、LLM judge）

3. **时间尺度**——单会话 / 多天 / 无限；单 target 还是多 target。

任何答案会显著影响基建决策的都要问。其它用默认值往前走。

### Step 2 — 扫既有工具

搭任何东西前，先看父项目有没有已有工具（评测、打分、渲染、数据）。用 Glob/Grep/Explore 搜：

- 评测脚本（`evaluate.py`、`score.py`、`eval/*`）
- 渲染 / 产物生成工具（`render.py`、`screenshot.py`、推理 harness）
- 参考数据集
- 既有 `.claude/` 记忆或 `CLAUDE.md` 提示的约定

**复用优于重写**。如果有可用的打分器就 import；没有才写最小可用版本。

### Step 3 — 搭项目骨架

创建以下结构（按主题调整名字——codegen 就把 `attempt.html` 改成 `attempt.py` 等）：

```
<project-dir>/
  CLAUDE.md                    # 活的研究协议（这个 skill 的输出）
  tools/
    produce.py                 # target → attempt（可选，也可能由 Claude 直接产）
    score.py                   # target + attempt → score.json
    select_next.py             # 从 manifest 挑下一个 target+iter_n（laggard 优先）
    measure.py                 # 领域探测工具（可选但推荐）
  targets/<id>/
    target.<ext>               # 输入 / 参考
    meta.json                  # {id, source, difficulty, notes}
  iterations/<id>/<NNN>/
    hypothesis.md              # 本轮测什么、为什么
    attempt.<ext>              # 本轮产物
    score.json                 # 完整指标
    diff.<ext>                 # 视觉或结构化 diff（如适用）
    [其它产物]
  state/
    manifest.json              # {phase, targets[], rotation[], rotation_idx, counters, last_improved, plateau_window}
    best/<id>.<ext>            # 每个 target 的当前最佳
    best/<id>.score.json
    lessons.md                 # 分类的、演化的规则
    research_log.md            # 叙事时间线（★ 突破、✗ 假设失败）
```

关键文件：

- `state/lessons.md`——从空的分类骨架开始：Layout/Content、Metrics、Tools、Meta，加上主题相关分类。每条一行：**规则 → 为什么 → 怎么用 → 什么时候学到**。

- `state/research_log.md`——从 bootstrap 条目起。每行格式：`YYYY-MM-DD HH:MM | tgt=XXX it=NNN score=X.XX Δ=±X.XX [★|✗|steer] | 一句话改动`

- `CLAUDE.md`——每轮协议。见下方模板。

- `state/manifest.json`——`{"phase": 0, "targets": [...], "rotation": [...], "rotation_idx": 0, "counters": {...}, "last_improved": {...}, "plateau_window": 5}`

### Step 4 — 写 CLAUDE.md（10 步迭代协议）

CLAUDE.md 的正文是用户通过 `/loop` 调用的迭代协议。按主题调整，保留这 10 步：

1. **Orient**——读 manifest、当前 lessons（只读分类相关的）、上轮 hypothesis + diff
2. **Select**——`python tools/select_next.py` 返回 `(target_id, iteration_number)`，laggard 优先
3. **Study**——看 target、当前 best、上轮 diff 视觉
4. **Hypothesize**——写 `hypothesis.md`：基线分、观察、可证伪预测、备用候选、预期 Δ
5. **Generate**——基于 `best.<ext>` 改出 `attempt.<ext>`（不是从零重写）
6. **Evaluate**——跑流水线：produce → score
7. **Judge**——反作弊（如适用）、对比 best 的 Δ、升级或保持
8. **Record**——`research_log.md` 追加一行；只在获得**可推广、可证伪**的新洞见时更新 `lessons.md`
9. **Stop/continue**——分数严格优于 best 才更新 `best/`，同步 manifest 的 `last_improved`
10. **Self-pace**——按结果调用 ScheduleWakeup（见下方节奏规则）

### Step 5 — 播种初始 targets

按时间尺度挑 1–3 个起步。单 target 从最容易的开始；多 target 挑跨难度区间的。让用户可调整。

### Step 6 — 烟测

交给 /loop 之前：
- 用一个 trivial attempt 跑一轮端到端流水线，验证 produce + score + diff 都 OK
- 验证反作弊（如适用，比如把 target 本身当 attempt 提交应该打零分或被标记作弊）
- 确认 `state/` 被正确更新

### Step 7 — 交接给 /loop

告诉用户：`/loop 按 CLAUDE.md 的 10 步协议推进一轮 <topic> 研究`（本地化到他们的语言）。**不要**自动启动——让他们确认。

## ITERATION 阶段（单轮）

每次 `/loop` 触发。按 CLAUDE.md 的 10 步执行。关键纪律：

- **每轮一个假设**，明确写在 `hypothesis.md`（首选单变量；只在一组相关变量是**一个概念性改动**时才打包）
- **提前列备选候选**——主要假设失败时有方向可切，不至于乱试
- **失败轮也是数据**——log 标 `✗`，不更新 best；如果失败暴露了方法论真相，记一条 Meta lesson

## STEERING 阶段（loop 之间的用户对话）

当用户发的消息不是 `/loop` 触发、而且研究项目已存在，把它当成 **steering**——他们想检视、调整、重定向正在进行的研究。**不要**自动启新一轮，除非他们明确要求。

Steering 操作识别与处理：

### 1. 查看 / 检视状态
- "当前进度？" / "progress?" → 汇总 `best/*.score.json` + 最近几行 `research_log.md` + 当前 Phase
- "最近一轮做了什么？" → 读上轮 `hypothesis.md` + log 行 + 显示 diff 图
- "哪个 target 最弱？" / "which target is hardest?" → 读 best 分数，识别 laggard
- "给我看 lessons" → 读 `lessons.md`（可按分类筛）
- "给我看 iter N 的 diff" → 加载并显示那一轮的 diff 产物

### 2. 调整策略 / 范围
- "下一轮专注攻 X" → 写 override 到 `state/next_override.json` 之类，确保下次 `/loop` 读到
- "暂停 target 002" → `manifest.json` 里把它 `targets[<id>].active = false`
- "gap 改成试 30 不是 20" → 作为操作员提示加到 `lessons.md`
- "换个激进策略" → 基于最近 diff 提 2–3 个候选假设，等用户选
- "跳过下一轮" → 取消 / 跳过 ScheduleWakeup

### 3. 按用户指示直接编辑状态
- "把这条 lesson 删了" → 编辑 `lessons.md`，在 `research_log.md` 留一行删除注记
- "重置 003 的 best" → 先把 `best/003.*` 归档再覆盖；**破坏性操作前先确认**
- "Phase 提到 2" → 编辑 `manifest.json.phase`，在 `research_log.md` 记理由

### 4. 控制 loop 本身
- "停了" / "stop" → 取消排队的 wakeup，确认 loop 已停
- "现在跑一轮" / "run now" → **不要**自己调 loop skill；内联跑一次 10 步协议，跑完问用户是否恢复 `/loop`
- "加快点" / "慢一点" → 调整下次 `ScheduleWakeup` 的 delaySeconds 并告诉用户选了多少
- "换到 15 分钟间隔" → 把节奏偏好存入 `manifest.json.pacing_preference`，后续轮次尊重

### 5. 研究方法论问题
- "怎么确认这条 lesson 是对的？" → 提议受控实验（开 / 关对比，各一轮）
- "这个方向没前途" → 承认，询问替代方向，基于当前 diff 列 3 个高杠杆的转向候选

### Steering 纪律

- **Steering 期间不要静默迭代**——用户在 steering 是想对话，不是多跑几轮。叫跑就跑一次，跑完停
- **把 steering 决策写下来**——任何 state 改动（策略、范围、lessons 编辑）都在 `research_log.md` 加一行 `[steer]` 标签，让未来轮次看到上下文
- **破坏性 steering 先确认**——删迭代、清 best、整改 manifest 需要明确 "yes, do it" 才编辑
- **干净恢复 /loop**——用户说"好，继续 /loop"时，先验证 `state/` 一致性、上次排的 wakeup 是否还有意义；pacing 改过就按新值重排

## 节奏规则（ScheduleWakeup 用）

- 默认：`delaySeconds = 1800`（30 min）——够思考，且单轮 < 5 min 工作时在 cache 窗内
- 突破（`Δ > +0.02` 或主题等价的"大胜"）：`900–1200s`（15–20 min），趁热打铁
- Plateau（`连续 3 轮 Δ ≈ 0 或 < 0`）：拉长到 `2400s`（40 min）。多出的时间用来构造**类别不同**的新假设（换战术，不是换数值）
- 不低于 900s——再低就变反射式写代码，不是研究了
- 用户说停、或主目标已达，就不排

## 反 plateau 策略

卡住时按顺序循环：

1. **Measure harder**——写或扩展 `measure.py`，从 target 直接抽真值而非肉眼估。大多数 plateau 都是"我其实不知道 target 长什么样"
2. **Check for infrastructure errors**——打分工具对吗？渲染流水线与 target 流水线匹配吗？（UI 重建中这暴露过：viewport 错、缺 scrollbar、scale 因子错）
3. **Find the dominant residual**——算 per-row / per-region / per-class diff，找出最大贡献切片，攻它，不是"整体优化"
4. **Test your own lessons**——lesson 可能是错的。从 target 数据重新验证
5. **Swap coupling model**——在用 flex / 隐式布局？换显式（absolute、固定尺寸）。显式更好推理
6. **Shrink step size**——20px 移动会过冲；试 5px。大步长立竿见影，小步长里的改进反而稀有
7. **Radical rewrite**——每 ~20 轮一次，用当前 lessons 把某个块从零重写。微编辑累积会困在局部最优
8. **Activate next-phase capability**——允许的话（外部素材、辅助模型调用、人工裁判），打开一直留着的能力

## 反 pattern

- **不写假设就跑**——每轮都得先落一份可证伪的东西在 `hypothesis.md`
- **分数打平也升 best**——只有严格大于才升；打平保持 best 稳定
- **Lessons 写成叙事**——`lessons.md` 是**规则**，不是故事。故事进 `research_log.md`
- **删失败迭代**——那是训练数据，目录留着
- **静悄悄改基建**——改 `tools/score.py` 或 `render.py` 要把所有 best 重新打分，`research_log.md` 加 `## Hotfix` 段

## 输出约定（给用户的返回）

Bootstrap 之后：项目布局简述、初始挑的 target、烟测结果、启动 /loop 的命令。

每轮迭代后：{指标：before, after, Δ, ★|✗} 表、一句话诊断、下次 wakeup 时间。

N 轮后或用户询问时：进度 roll-up——每 target 分数 vs 起始、top 3 lessons、剩余瓶颈。
