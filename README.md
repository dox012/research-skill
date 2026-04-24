# research — Claude Code 长期自主研究 skill

> 把任何可量化的问题变成一条有纪律的、自主迭代的研究流水线。
> Claude 跨轮迭代、测量、累积 lessons，会话关掉再打开也不丢上下文。

`research` 是 Claude Code 的一个 skill：在目录里起个研究项目，搭评测基建，用"假设 → 尝试 → 打分 → 教训"循环推进，并通过 [`/loop`](https://docs.claude.com/en/docs/claude-code/skills) 让研究在你离开时继续跑。

它的反面是"让 Claude 更努力"。它给 Claude 的是**协议**：每轮一个可证伪的假设，一本记录胜负的日志，还有被卡住时的反 plateau 攻略。

---

## 什么时候用

用 `/research` 当：

- 问题有**可量化的评测指标**（像素匹配、SSIM、准确率、BLEU、通过率、A/B 裁判等）
- 预期需要**多轮**才能做好——不是一问一答
- 希望第 N 轮的教训真能帮到 N+1 轮，且关电脑后还能继续跑

别用它：

- 一次性问题 / 小修小补（直接问 Claude 就行）
- 没评测信号的任务（不知道自己变好没）
- 轮次之间不共享状态（比如相互独立的 bug 修复）

---

## 工作原理

### 三种模式自动识别

1. **Bootstrap**（项目首次启动）—— Claude 进 plan mode，问 2–3 个关键问题（评测指标 / 输入输出形态 / 时间尺度），扫父项目已有工具，搭建目录骨架，写 `CLAUDE.md` 作为协议文档。

2. **Iteration**（`/loop` 触发每一轮）—— Claude 执行一轮 10 步协议并排定下次唤醒。

3. **Steering**（你在两轮之间跟 Claude 说话）—— 查状态、调整策略、编辑 lessons、暂停恢复 loop 等，都通过对话完成。详见下面"Steering"章节。

### 10 步协议

每轮：

1. **Orient** — 读 manifest、相关 lessons、上轮 hypothesis 和 diff
2. **Select** — laggard 优先挑选 (target, iteration)
3. **Study** — 看 target 和当前 best
4. **Hypothesize** — 写**可证伪、单变量**的假设，附备选候选
5. **Generate** — 基于 best 改（不是从零重写）
6. **Evaluate** — 跑流水线打分
7. **Judge** — 反作弊、比 best、升级或丢弃
8. **Record** — `research_log.md` 追加一行；`lessons.md` 仅在获得**可推广洞见**时更新
9. **Stop/continue** — 严格优于 best 才升级
10. **Self-pace** — 按结果选择下一次唤醒的 delay

### 节奏

| 结果                   | delay       | 理由                                              |
| ---------------------- | ----------- | ------------------------------------------------ |
| 突破（Δ > +0.02）      | 15–20 min   | 趁热打铁                                          |
| 普通进展 / 小 Δ        | 30 min      | 默认——够思考，且在 prompt-cache 窗内             |
| Plateau（3 轮平淡）    | 40 min      | 多给时间构造**类别不同**的新假设                  |

不低于 15 min——低于这个就变成反射式写代码了。

### 反 plateau 阶梯

卡住时按顺序尝试：

1. **Measure harder** — 写 probe 工具直接从 target 抽取真值，别用肉眼估
2. **Check infrastructure** — 流水线本身对吗？（viewport 错、缺素材、裁切 off-by-one）
3. **Find the dominant residual** — 哪个具体区域 / 行 / 类贡献最大？攻它，不是"整体优化"
4. **Test your own lessons** — lesson 可能是错的；从数据重新验证
5. **Swap coupling model** — flex → absolute，隐式尺寸 → 显式
6. **Shrink step size** — 大步长会过冲；试小的
7. **Radical rewrite** — 每 ~20 轮一次，把某个块从零按当前 lessons 重做
8. **Activate next-phase capability** — 打开之前留着的能力（外部素材、judge 模型、人工评价）

---

## Steering（非 loop 对话）

`/loop` 跑得很爽，但你随时可以打断让 Claude 做别的事——不用停 loop，也不用重建项目。直接打字：

### 1. 查看状态
- `"当前进度？"` → 汇总 best 分数 + 最近几轮 log + 当前 Phase
- `"最近一轮做了什么？"` → 上轮 hypothesis + log 行 + diff 图
- `"哪个 target 最弱？"` → 识别 laggard
- `"给我看 lessons"` → 读 lessons.md（或按分类筛）
- `"给我看 iter 5 的 diff"` → 加载对应图

### 2. 调整策略
- `"下一轮专注攻 X"` → 记录 override
- `"暂停 target 002"` → manifest 里把它 active=false
- `"换个激进策略"` → Claude 基于当前 diff 给 2–3 个候选假设让你选
- `"跳过下一轮"` → 取消下次 wakeup

### 3. 直接编辑状态
- `"把这条 lesson 删了"` → 改 lessons.md，log 留一行注记
- `"重置 003 的 best"` → 先归档再覆盖，**破坏性操作会先问你确认**
- `"Phase 提到 2"` → 改 manifest，log 记录理由

### 4. 控制 loop 本身
- `"停了"` → 取消排队的 wakeup
- `"现在立刻跑一轮"` → 内联跑一轮 10 步协议，问你是否恢复 /loop
- `"换成 15 分钟间隔"` → 存入 manifest 的 pacing 偏好，之后每轮尊重

### 5. 方法论问题
- `"怎么确认这条 lesson 是对的？"` → 提议受控实验（开/关对比）
- `"这个方向没前途"` → 基于当前 diff 列出 3 个高杠杆的转向候选

### Steering 纪律

- **Steering 期间不自动推进迭代**——你想说话不是要多跑几轮；叫跑就跑一次，跑完停下
- **决策写进日志**——策略改动、范围调整、lessons 编辑都在 `research_log.md` 加一行 `[steer]` 标签的记录，下轮能看到
- **破坏性操作先确认**——删迭代、清 best、整改 manifest 前必须明确 yes
- **干净恢复 /loop**——你说"继续 /loop"时 Claude 会验证 state 一致性，如果 pacing 改过会按新值重排

---

## 项目结构

`/research` 建的项目长这样：

```
your-project/
├── CLAUDE.md                        # 活的协议——Claude 每轮都读
├── tools/
│   ├── produce.py                   # target → attempt（领域相关）
│   ├── score.py                     # target + attempt → score.json
│   ├── select_next.py               # laggard 优先的 target/iter 选择
│   └── measure.py                   # target 真值探测工具
├── targets/<id>/
│   ├── target.<ext>                 # 输入 / 参考
│   └── meta.json                    # { id, source, difficulty, notes }
├── iterations/<id>/<NNN>/
│   ├── hypothesis.md                # 本轮假设与预期
│   ├── attempt.<ext>                # 本轮产物
│   ├── score.json                   # 完整指标
│   └── diff.<ext>                   # 差异图 / 结构化 diff
└── state/
    ├── manifest.json                # phase、rotation、counters、last_improved
    ├── best/<id>.<ext>              # 每个 target 的当前最佳
    ├── best/<id>.score.json
    ├── lessons.md                   # 分类的、演化中的规则（不是叙事）
    └── research_log.md              # 每轮一行（★ 胜 / ✗ 败 / [steer] 人工）
```

### `lessons.md` vs `research_log.md` 职责

- **`research_log.md`** 是**时间线**。每轮一行可解析格式：
  `YYYY-MM-DD HH:MM | tgt=XXX it=NNN score=X.XX Δ=±X.XX [★|✗|steer] | 一句话改动`
- **`lessons.md`** 是**规则本**。每条是一条分类规则：规则 + 为什么 + 怎么用 + 什么时候学到的。不写故事，不写叙事。后续轮次推翻某条 → **就地修改**，lessons 会演化。

`lessons.md` 不写故事很关键：Claude 每轮都读它，要高信噪比的规则，不是日志。

---

## 安装

```bash
# Linux / macOS
mkdir -p ~/.claude/skills/research
cp SKILL.md ~/.claude/skills/research/SKILL.md

# Windows
mkdir "%USERPROFILE%\.claude\skills\research"
copy SKILL.md "%USERPROFILE%\.claude\skills\research\SKILL.md"
```

不用注册——Claude Code 会话启动时自动发现用户级 skill。

在 Claude Code 里输 `/` 能看到 `research` 就算装好了。

---

## 快速上手

```
cd ~/projects/your-research-project
/research <你的主题>
```

Claude 会：

1. 进 plan mode，问 2–3 个关于 指标 / 输入输出形态 / 时间尺度 的问题
2. 扫父项目有没有可复用的评测代码
3. 搭 `tools/` `targets/` `iterations/` `state/`，写对应主题的 `CLAUDE.md`
4. 跑一次烟测验证流水线
5. 交给你——你再 `/loop` 开始自动迭代

之后 `/loop` 推着研究往前跑，你想介入随时打字 steering。

---

## 默认语言

中文。`CLAUDE.md` / `lessons.md` / `research_log.md` / `hypothesis.md` 默认中文；Claude 用中文回复你。代码注释和文件名保持英文（互操作性）。如果 topic 是英文的会自动切英文输出。

---

## 设计原则

- **Infrastructure 先于 iteration** —— 先写好 `score.py` / `measure.py` / `select_next.py`，没可靠打分，多跑几轮只是加噪
- **State 胜于记忆** —— `lessons.md` / `research_log.md` / `best/` 跨会话持久化，Claude 上下文内的"我记得"无关紧要
- **Hypothesis 先于 attempt** —— `hypothesis.md` 先写，必须可证伪、有预期 Δ；没有假设就没有这一轮
- **失败轮是数据** —— ✗ 轮连同 attempt/hypothesis 留盘，约束未来（"别再试已失败的"）
- **Lessons 是规则不是故事** —— `lessons.md` 读起来像 playbook 条目；叙事去 log
- **Laggard 优先** —— 分数差距 > 0.10 时轮询关注最弱 target，避免永远优化简单的那个
- **一轮一个假设** —— 单变量优先；多变量只在它们是**一个概念性改动**时才打包

---

## 依赖

- **Claude Code** + `/loop` skill（默认自带）
- **工作目录**——研究项目放哪里就在哪 `cd` 再调用
- **领域相关工具链**——图像用 Python + Pillow、代码用 test runner 等，skill 会自动检测并复用

---

## FAQ

**Q：关了 Claude Code 还会继续跑吗？**
不会——`/loop` 是会话级。想真持久搭 `/schedule`（Anthropic 云 cron），或外层写个重启脚本。

**Q：能打断让它改方向吗？**
能。按 Esc 或直接打字，Claude 会读当前 `state/` 把你的意图嵌入进行中的协议。见 Steering 章节。

**Q：中途改评测指标怎么办？**
改 `tools/score.py`，在 `research_log.md` 加 `## Hotfix` 段说明。所有现有 best 应重测，后续轮次才可比。

**Q：一个项目通常要多少轮？**
看问题难度。单 target 从"像样"到"近完美"通常 20–50 轮；最后 10–20% 常需进入下一阶段能力（引入外部素材、judge 模型等）。

**Q：花多少钱？**
每轮一次 Claude 交互 + 产物读写 + 一次打分调用——单轮很便宜。30min 节奏 ≈ 2 轮/小时。想省就 `/loop 1h …` 拉长。

---

## License

MIT。随意用、fork、改。
