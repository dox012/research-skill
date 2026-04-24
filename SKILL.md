---
name: research
description: Bootstrap + drive a long-term self-improving research project on any topic that has a measurable success metric. Creates a project directory, builds measurement/evaluation infrastructure, runs iterative loops that accumulate lessons across rounds, and integrates with /loop for autonomous progress. Use when the user wants to "research X", "improve X until near-perfect", "keep trying X and learn" — anything where rounds of hypothesis→attempt→score→lesson are productive.
---

# /research — long-term self-improving research skill

User topic: **$ARGUMENTS**

This skill takes a topic and drives it through structured iteration. It's not a one-shot answer — it builds infrastructure and returns.

## Default language: Chinese（中文）

Unless the user writes in English or explicitly asks for English output:

- Write `CLAUDE.md`, `lessons.md`, `research_log.md`, and `hypothesis.md` **in Chinese**.
- Reply to the user in Chinese.
- Code comments and filenames stay English (interoperability).
- The user's topic string in `$ARGUMENTS` determines language: mostly-Chinese topic → Chinese everything; mostly-English topic → English everything; ambiguous → ask once at bootstrap.

When a Chinese project has been bootstrapped, all future iterations keep Chinese even if the trigger prompt ("/loop …") happens to be English.

## Phase detection (do this FIRST)

Detect which mode you're in:

- **`state/manifest.json` exists + user message is a `/loop` fire** → ITERATION phase (autonomous). Run the 10-step protocol.
- **`state/manifest.json` exists + user is talking to you** → STEERING phase (human-in-the-loop). See "Steering" section below. Do NOT force a new iteration.
- **`state/manifest.json` does not exist** → BOOTSTRAP phase. Run the bootstrap flow.

If the user's topic references continuing existing work ("继续攻 X", "再跑一轮") but no manifest exists, ask if they meant to `cd` to an existing research dir.

## BOOTSTRAP phase

Non-trivial — use `EnterPlanMode` unless the topic is extremely narrow (single quick question). Plan mode lets you survey the environment, ask the user clarifying questions, and commit to a scope before scaffolding.

### Step 1 — Scope the research (use AskUserQuestion; keep it to 2–3 questions)

Ask the user about whichever of these are not already clear from `$ARGUMENTS`:

1. **Success metric** — what's the scoring function? Examples:
   - Numerical (pixel match, SSIM, BLEU, accuracy %)
   - Comparison-based (A vs B judgment by another model / human)
   - Binary milestone ("did this test pass")
   - Qualitative with rubric
   Pick ONE primary + optional secondaries. If the user is unsure, propose one based on the topic and confirm.

2. **Target & attempt shapes** — what goes in, what comes out each iteration?
   - Target: what's the input/reference? (an image, a prompt, a codebase, a benchmark set)
   - Attempt: what does Claude produce each round? (HTML, Python code, text, an image via another tool)
   - How is the attempt turned into a score? (render + diff, run tests, human eval, LLM judge)

3. **Horizon & scale** — single session / multi-day / indefinite; one target or many.

If any answer would change infrastructure decisions significantly, ask. Otherwise commit defaults and move on.

### Step 2 — Survey existing tools

Before building anything, check if the parent project already has relevant utilities (measurement, scoring, rendering, data). Search with Glob/Grep/Explore for:

- Evaluation scripts (`evaluate.py`, `score.py`, `eval/*`)
- Rendering/producing utilities (`render.py`, `screenshot.py`, inference harness)
- Reference datasets
- Any prior `.claude/` memory or `CLAUDE.md` that hints at conventions

**Reuse over rewrite**. If a reasonable scorer already exists, import it. If not, write the minimum viable one.

### Step 3 — Scaffold the project

Create this structure (adapt names to the topic — `attempt.html` → `attempt.py` if the domain is codegen, etc.):

```
<project-dir>/
  CLAUDE.md                    # the living research protocol (this skill's output)
  tools/
    produce.py                 # target → attempt (optional, may be Claude-direct)
    score.py                   # target + attempt → score.json
    select_next.py             # pick next target+iter_n from manifest (laggard-priority)
    measure.py                 # domain probe (optional but recommended)
  targets/<id>/
    target.<ext>               # the input/reference
    meta.json                  # {id, source, difficulty, notes}
  iterations/<id>/<NNN>/
    hypothesis.md              # what this round tests and why
    attempt.<ext>              # this round's produced artifact
    score.json                 # full metrics
    diff.<ext>                 # visual or structured diff (if applicable)
    [extra artifacts]
  state/
    manifest.json              # {phase, targets[], rotation[], rotation_idx, counters, last_improved, plateau_window}
    best/<id>.<ext>            # current best attempt per target
    best/<id>.score.json
    lessons.md                 # categorized, evolving guidelines
    research_log.md            # narrative timeline (★ breakthrough, ✗ failed hypothesis)
```

Key files:

- `state/lessons.md` — start with empty categorized skeleton: Layout/Content, Metrics, Tools, Meta, plus topic-specific categories. One rule per bullet: **rule → why → how to apply → when it was learned**.

- `state/research_log.md` — start with a bootstrap entry. Format per line: `YYYY-MM-DD HH:MM | tgt=XXX it=NNN score=X.XX Δ=±X.XX [★|✗] | one-line change description`.

- `CLAUDE.md` — the per-round protocol. See template below.

- `state/manifest.json` — `{"phase": 0, "targets": [...], "rotation": [...], "rotation_idx": 0, "counters": {...}, "last_improved": {...}, "plateau_window": 5}`.

### Step 4 — Write CLAUDE.md (the 10-step iteration protocol)

The body of CLAUDE.md is the iteration protocol the user will invoke via `/loop`. Adapt to topic but keep these 10 steps:

1. **Orient** — read manifest, current lessons (category-relevant only), prior hypothesis + diff
2. **Select** — `python tools/select_next.py` returns `(target_id, iteration_number)` with laggard priority
3. **Study** — read target, current best attempt, last diff visually
4. **Hypothesize** — write `hypothesis.md` with: baseline score, observation, falsifiable prediction, backup candidates, expected Δ
5. **Generate** — produce `attempt.<ext>` based on `best.<ext>` (not from zero)
6. **Evaluate** — run the pipeline: produce → score
7. **Judge** — cheating check (if applicable), Δ vs best, promote or keep
8. **Record** — append one line to `research_log.md`; update `lessons.md` only when a *generalizable, falsifiable* new insight emerges
9. **Stop/continue** — update `best/` if improved, update `last_improved` in manifest
10. **Self-pace** — call ScheduleWakeup with a delay informed by outcome (see pacing rules below)

### Step 5 — Seed initial targets

Depending on horizon, pick 1–3 targets to start. For single-target work, start with the easiest case; for multi-target, pick across a difficulty spectrum. Let the user redirect.

### Step 6 — Smoke test

Before handing off to /loop:
- Run the pipeline end-to-end once on target 001 with a trivial attempt to verify produce + score + diff all work
- Verify anti-cheat (if applicable — e.g., submitting the target itself as attempt should score 0 or flag cheating)
- Check that `state/` gets updated

### Step 7 — Hand off to /loop

Tell the user: `/loop 按 CLAUDE.md 的 10 步协议推进一轮 <topic> 研究` (or localize to their language). Don't auto-start — let them confirm.

## ITERATION phase (single round)

This is invoked each time `/loop` fires. Follow CLAUDE.md's 10 steps. Key discipline:

- **One hypothesis per round**, explicit in `hypothesis.md` (single variable preferred; bundle only when a group of related vars is one conceptual change).
- **Backup candidates** listed in advance — when the primary fails, you know where to go next instead of flailing.
- **Failed rounds are data** — record as `✗` in the log; don't update best; add a Meta lesson if the failure revealed a methodological truth.

## STEERING phase (user conversation between loops)

When the user types something that is NOT a `/loop` firing and a research project exists, treat it as **steering** — they want to inspect, adjust, or redirect the ongoing research. Do not kick off a new iteration unless they explicitly ask.

Steering operations you should recognize and handle:

### 1. Inspect / query state
- "当前进度？" / "progress?" → summarize `best/*.score.json` + last few `research_log.md` lines + current Phase
- "最近一轮发生了什么？" → read last `hypothesis.md` + last log line + show the diff image
- "哪个 target 最弱？" / "which target is hardest?" → read best scores, identify laggard
- "show me lessons" → read `lessons.md` (or filter by category)
- "给我看 iter N 的 diff" → load + display that iteration's diff artifact

### 2. Adjust strategy / scope
- "下一轮专注攻 X" → note override in a `state/next_override.json` or similar, ensure next `/loop` reads it
- "暂停 target 002" → set `targets[<id>].active = false` in `manifest.json`
- "把 gap 改成试 30 不是 20" → add to `lessons.md` as operator-supplied hint
- "换个激进策略" → suggest 2–3 candidate hypotheses based on recent diff; wait for user pick
- "跳过下一轮" → delete/skip ScheduleWakeup

### 3. Edit state directly on user's instruction
- "把这条 lesson 删了" → edit `lessons.md`, keep a one-line note in `research_log.md` about the removal
- "重置 003 的 best" → offer to move current `best/003.*` to an archive path before nuking; confirm before destructive
- "把 Phase 提到 2" → edit `manifest.json.phase` and record reason in `research_log.md`

### 4. Control the loop itself
- "停了" / "stop" → cancel pending wakeup, confirm loop is inactive
- "再跑一轮" / "run now" → do NOT call the loop skill yourself; run the 10-step protocol once inline, then re-prompt them whether to resume `/loop`
- "加快点" / "slow down" → adjust next `ScheduleWakeup` delaySeconds and tell them what you picked
- "换到 15 分钟间隔" → update pacing preference in `manifest.json.pacing_preference` so future rounds respect it

### 5. Research methodology questions
- "怎么确认这条 lesson 是对的？" → propose a controlled experiment (one iter with the lesson applied vs disabled)
- "我觉得这个方向没前途" → acknowledge, ask for alternative direction, list the 3 highest-leverage pivots based on current diff

### Steering discipline

- **Don't silently iterate during steering** — the user steering means they want a dialogue, not more rounds. If they ask you to run, run once, then stop.
- **Write down steering decisions** — any change to state (strategy, scope, lessons edit) gets a one-line entry in `research_log.md` tagged `[steer]` so future rounds see the context.
- **Confirm before destructive steering** — deleting iterations, nuking bests, changing manifest wholesale needs explicit "yes, do it" before the edit.
- **Resume cleanly** — when the user says "好 继续 /loop"，verify `state/` is coherent and the last wakeup still makes sense; if pacing changed, reschedule with the new value.

## Pacing rules (for ScheduleWakeup)

- Default: `delaySeconds = 1800` (30 min) — enough to think, within cache if rounds are under 5 min of work.
- Breakthrough (`Δ > +0.02` or topic-equivalent "big win"): `900–1200s` (15–20 min) to strike while insight is fresh.
- Plateau (`3 rounds of Δ ≈ 0 or < 0`): extend to `2400s` (40 min). Use the extra time to construct a *categorically different* hypothesis (change tactic, not just change value).
- Don't go below 900s — that's reflexive code-slinging, not research.
- Don't schedule if the user said stop, or if the primary goal is met.

## Reverse-plateau strategies

When stuck, cycle through these in order:

1. **Measure harder** — write or extend `measure.py` to extract exact positions/values from target instead of eyeballing. Most plateaus are "I don't know what target actually looks like".
2. **Check for infrastructure errors** — is the scoring tool right? Is the rendering pipeline matching the target's pipeline? (In UI recon this surfaced: wrong viewport, missing scrollbar, wrong scale factor.)
3. **Find the dominant residual** — compute per-row / per-region / per-class diff; identify which specific slice contributes most. Attack that, not "general improvement".
4. **Test your own lessons** — a lesson can be wrong. Re-derive it from target data.
5. **Swap coupling model** — if you're using flex/implicit layout, switch to explicit (absolute, fixed size). Explicit is easier to reason about in iteration.
6. **Shrink step size** — 20px moves can overshoot; try 5px. Big-step moves that help look obvious; small-step moves that help are rarer.
7. **Radical rewrite** — once per every ~20 rounds, rewrite one block from scratch using current lessons. Accumulated micro-edits can trap you in a local optimum.
8. **Activate next-phase capability** — if permitted (external assets, auxiliary model call, human judge), flip on the capability you've been saving.

## Anti-patterns to avoid

- **Running without a hypothesis** — every round needs something falsifiable written down before the attempt.
- **Updating best on ties** — only strictly-better scores promote; ties keep best stable.
- **Lessons as narrative** — lessons.md is for *rules*, not stories. Stories go in research_log.md.
- **Deleting failed iterations** — they're the training data. Keep the directory.
- **Silent infrastructure changes** — if you change `tools/score.py` or `render.py`, rebaseline all bests and write a Hotfix section in research_log.md.

## Output contract (what to return to the user)

After bootstrap: a short summary of the project layout, initial targets picked, smoke-test result, and the /loop command to start.

After each iteration: a table of {metric: before, after, Δ, ★|✗}, one-sentence diagnosis, next wakeup time.

After N rounds or when asked: a progress roll-up showing per-target scores vs initial, top 3 lessons learned, remaining bottlenecks.
