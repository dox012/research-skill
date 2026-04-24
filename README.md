# claude-research-skills

四个配合使用的 Claude Code skill，把科研工作流拆成"摸领域 → 想问题 → 做实验 → 写报告"四段：

| skill | 做什么 | 什么时候用 |
|---|---|---|
| [`/survey`](./survey/) | 快速背景调研——SOTA / benchmark / 方法分类 / gap | 进入陌生领域，想快速摸到坐标系 |
| [`/idea`](./idea/) | 粗主题 → 3–5 个具体可证伪的问题候选 | 有方向但问题还没定义清楚 |
| [`/research`](./research/) | 闭环驱动研究项目（假设→尝试→打分→教训循环） | 问题定义好了，要跑实验 |
| [`/paper`](./paper/) | 从 state/ 目录生成 Markdown 研究报告 | 做了若干轮 /research，要汇报或存档 |

---

## 典型流水线

```
# 0. 这领域在搞什么？
/survey 大模型推理优化

# → 生成 survey.md，标出 SOTA / gap

# 1. 我想研究什么？
/idea 降低大模型推理延迟
# → /idea 读同目录 survey.md 作上下文，产出 3 个候选，你选其一

# 2. 开干
cd ~/projects/llm-latency
/research 在 vLLM 0.6 上，能否把 8B 模型 p95 延迟从 120ms 降到 80ms
# → /research bootstrap + /loop 自主迭代

# 3. 写报告
/paper
# → 生成 REPORT.md
```

四个 skill **独立可用**。可以跳过 `/survey` `/idea` 直接 `/research`，或对已有 state/ 直接 `/paper`。

---

## 安装

所有 skill 都是用户级，Claude Code 会话启动时自动发现。

```bash
# Linux / macOS
mkdir -p ~/.claude/skills/{survey,idea,research,paper}
cp survey/SKILL.md  ~/.claude/skills/survey/SKILL.md
cp idea/SKILL.md    ~/.claude/skills/idea/SKILL.md
cp research/SKILL.md ~/.claude/skills/research/SKILL.md
cp research/README.md ~/.claude/skills/research/README.md   # 可选
cp paper/SKILL.md   ~/.claude/skills/paper/SKILL.md
```

```powershell
# Windows
$skills = "$env:USERPROFILE\.claude\skills"
foreach ($s in "survey","idea","research","paper") {
    New-Item -ItemType Directory -Force -Path "$skills\$s" | Out-Null
    Copy-Item "$s\SKILL.md" "$skills\$s\SKILL.md"
}
```

装好后在 Claude Code 里输 `/` 应看到 `survey` / `idea` / `research` / `paper` 四个 skill。

---

## 设计原则

三个 skill 共享的核心理念：

- **State 胜于记忆**：lessons / research_log / best 持久化到磁盘，跨会话不丢
- **假设先于尝试**：每轮 /research 必须先写可证伪的 hypothesis.md
- **规则不是故事**：lessons.md 是 playbook 条目；叙事进 log
- **默认中文**：topic 用中文则全中文产出；英文则英文；代码注释/文件名保持英文
- **Steering 优先于自动化**：用户任何时候都能打断 /loop 调整方向，不用停项目

---

## 各 skill 详细文档

- [`research/README.md`](./research/README.md) — 10 步协议 / 节奏规则 / 反 plateau 阶梯 / 项目目录布局等
- `survey/SKILL.md` — 查询规划、来源筛选、survey.md 模板
- `idea/SKILL.md` — 问题形式化流程、反 pattern、候选模板
- `paper/SKILL.md` — 报告模板、风格约束、错误处理

---

## License

MIT。用、fork、改都随意。
