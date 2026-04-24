# research — a long-term self-improving research skill for Claude Code

> Turn any measurable problem into a disciplined, autonomous research loop.
> Claude iterates, measures, records lessons, and gets better over time — without forgetting between sessions.

`research` is a Claude Code skill that bootstraps a project directory, builds evaluation infrastructure, runs hypothesis → attempt → score → lesson cycles, and integrates with [`/loop`](https://docs.claude.com/en/docs/claude-code/skills) so progress continues while you're away.

It's the opposite of asking Claude to "try harder". It gives Claude *a protocol*: one falsifiable hypothesis per round, a log of what worked and what didn't, and a set of reverse-plateau strategies when things get stuck.

---

## When to use it

Use `/research` when:

- The problem has a **measurable metric** (pixel match, SSIM, accuracy, BLEU, test-pass rate, A/B judgment, etc.)
- You expect it to take **many rounds** to solve well — not a one-shot answer
- You want lessons from round N to actually help round N+1, and for it to keep going when you close the laptop

Don't use it for:

- One-off questions or narrow fixes (just ask Claude directly)
- Problems with no evaluation signal (you can't tell if you're getting better)
- Tasks where rounds don't share a state (e.g., independent bug fixes)

Example prompts:

```
/research reconstruct this UI screenshot into HTML/CSS at pixel-level
/research tune my RAG pipeline to beat 0.85 on the eval set
/research improve this code generator until it passes the full test suite
```

---

## How it works

### Two phases, detected automatically

1. **Bootstrap** (first call in a fresh directory) — Claude scopes the problem, asks 2–3 clarifying questions about your success metric and target/attempt shapes, surveys existing tools in the parent project, and scaffolds a full project layout with a living protocol document (`CLAUDE.md`).

2. **Iteration** (every subsequent `/loop` firing) — Claude runs a single round of the 10-step protocol and schedules the next wake-up.

### The 10-step protocol

Every iteration:

1. **Orient** — read manifest, relevant lessons, prior hypothesis + diff
2. **Select** — pick next (target, iteration) with laggard-priority rotation
3. **Study** — examine target and current best attempt
4. **Hypothesize** — write a *falsifiable, single-variable* hypothesis with backup candidates
5. **Generate** — produce an attempt based on current best (not from scratch)
6. **Evaluate** — run the pipeline → score
7. **Judge** — check for cheating, compare vs best, promote or discard
8. **Record** — append one line to `research_log.md`; add a *rule* to `lessons.md` only when a generalizable insight emerges
9. **Stop/continue** — update `best/` if strictly better
10. **Self-pace** — schedule next wake-up with a delay informed by outcome

### Pacing

Claude self-selects the next wake-up interval:

| Outcome                       | Delay        | Why                                                     |
| ----------------------------- | ------------ | ------------------------------------------------------- |
| Breakthrough (Δ > +0.02)      | 15–20 min    | Strike while the insight is fresh                       |
| Normal progress / small Δ     | 30 min       | Default — enough to think, within prompt-cache window  |
| Plateau (3 flat rounds)       | 40 min       | Extra time to construct a categorically new hypothesis |

Never goes below 15 min — below that it becomes reflexive code-slinging, not research.

### Reverse-plateau ladder

When stuck, Claude cycles through these in order:

1. **Measure harder** — write a probe tool that extracts exact values from the target instead of eyeballing
2. **Check infrastructure** — is the pipeline itself right? (wrong viewport, missing asset, off-by-one cropping)
3. **Find the dominant residual** — which specific region/row/class contributes most? Attack that, not "general improvement"
4. **Test your own lessons** — a lesson may be wrong; re-derive from data
5. **Swap coupling model** — flex → absolute, implicit sizing → explicit
6. **Shrink step size** — 20px moves overshoot; try 5px
7. **Radical rewrite** — every ~20 rounds, redo one block from scratch using current lessons
8. **Activate next-phase capability** — flip on something you'd been saving (external assets, judge model, human eval)

---

## Project anatomy

A typical project under `/research` looks like:

```
your-project/
├── CLAUDE.md                        # living protocol — Claude reads this every round
├── tools/
│   ├── produce.py                   # target → attempt (domain-specific)
│   ├── score.py                     # target + attempt → score.json
│   ├── select_next.py               # laggard-priority target/iter picker
│   └── measure.py                   # probe target for ground-truth values
├── targets/<id>/
│   ├── target.<ext>                 # input/reference
│   └── meta.json                    # { id, source, difficulty, notes }
├── iterations/<id>/<NNN>/
│   ├── hypothesis.md                # what this round tests and why
│   ├── attempt.<ext>                # this round's artifact
│   ├── score.json                   # full metrics
│   └── diff.<ext>                   # visual/structured diff
└── state/
    ├── manifest.json                # phase, rotation, counters, last_improved
    ├── best/<id>.<ext>              # current best attempt per target
    ├── best/<id>.score.json
    ├── lessons.md                   # categorized, evolving rules (not a narrative)
    └── research_log.md              # one line per iteration (★ win, ✗ fail)
```

### What `lessons.md` and `research_log.md` are *for*

- **`research_log.md`** is the *timeline*. One line per round, parseable format:
  `YYYY-MM-DD HH:MM | tgt=XXX it=NNN t5=X.XX Δ=±X.XX [★|✗] | one-line change`
- **`lessons.md`** is the *rulebook*. Each entry is one categorized rule: the rule itself, *why* it's true, *how to apply*, and *when it was learned*. Not stories, not narratives. If a later round refutes a rule, you **edit it in place** — lessons evolve.

Keeping stories out of `lessons.md` is critical: Claude reads it every round and needs high-signal rules, not logs.

---

## Installation

```bash
# Linux / macOS
mkdir -p ~/.claude/skills/research
cp SKILL.md ~/.claude/skills/research/SKILL.md

# Windows
mkdir "%USERPROFILE%\.claude\skills\research"
copy SKILL.md "%USERPROFILE%\.claude\skills\research\SKILL.md"
```

No registration — Claude Code auto-discovers user-level skills at session start.

Verify by typing `/` in Claude Code — you should see `research` in the list.

---

## Quickstart

```
cd ~/projects/ui-reconstruction        # or wherever you want the project
/research reconstruct UI screenshots into HTML/CSS at pixel level
```

Claude will:

1. Enter plan mode, ask 2–3 questions about metric / target format / scale
2. Survey your parent project for reusable evaluation code
3. Scaffold `tools/`, `targets/`, `iterations/`, `state/`, and a tailored `CLAUDE.md`
4. Run one smoke test to verify the pipeline works
5. Hand off:
   > Start with: `/loop 按 CLAUDE.md 的 10 步协议推进一轮 <topic> 研究`

From then on, `/loop` keeps the research moving autonomously.

---

## Case study: UI pixel reconstruction

Real progression from our dogfood project: three GitHub/Baidu/Google screenshots, targeting pixel-level HTML/CSS reconstruction (`pixel_match_t5 ≥ 0.99`).

### Results (9 iterations on target 003, GitHub)

| Iter | t5     | SSIM  | Δ         | What changed                                                          |
| ---- | ------ | ----- | --------- | --------------------------------------------------------------------- |
| 001  | 0.746  | 0.663 | baseline  | First attempt at 1912×922 scale                                       |
| 002  | 0.729  | 0.664 | **-0.017** ✗ | Batch anchor re-positioning (20px shifts) — **overshot**              |
| 003  | 0.746  | 0.661 | -0.001 ✗  | Banner top 5px — too small a signal to matter                         |
| 004  | 0.753  | 0.672 | **+0.007** ★ | Banner strict `height: 82px` instead of padding-driven                |
| 005  | 0.754  | 0.672 | +0.001 ★  | Infrastructure fix: render.py now paints Windows scrollbar           |
| 006  | 0.754  | 0.672 | +0.0001   | Banner text color → GitHub default (not custom warning brown)         |
| 007  | 0.759  | 0.682 | **+0.0045** ★ | README block shifted +28px (pixel-sampled vs eyeballed)              |
| 008  | 0.778  | 0.685 | **+0.019** ★★ | Callout: removed `#ddf4ff` blue bg, matched GitHub quote style       |
| 009  | 0.778  | 0.684 | ~0 ✗      | rdm top pullback — symmetric displacement, net zero                   |

### Key meta-lessons recorded

From `lessons.md` — verbatim categorized rules Claude accumulated:

- **visual similarity ≠ pixel match** — 20px shifts can make things look more similar while scoring *worse* (because approximate overlap creates more "near-miss" pixels than complete misalignment). Use ≤5px single-step moves.
- **flex margin is a coupling variable** — changing one `margin-bottom` moves every downstream sibling. For position tuning, use `position: absolute` anchors instead.
- **padding-driven heights drift** — `padding + text-content` lets font metrics decide final height. Explicit `height: Npx + align-items: center` is 10× more predictable. (+0.007 single-variable test.)
- **lessons can be wrong** — a lesson written from a failed experiment may have the wrong root cause. Re-verify when a later round contradicts it.
- **target.png may include Windows scrollbar** — Playwright headless doesn't paint one; the 17px stripe at `x=1895+` is pure noise unless you fix it in render.

And discovered infrastructure bugs that a naive loop would have missed:

- Targets were being 0.781× scaled down during init — real elements rendered at 1.28× their should-be size until `init_targets.py` and `render.py` both moved to native 1912×922.
- Playwright omits the system scrollbar; our target screenshots include it. Added a post-process to paint a gray band.

Neither of these would be fixable by trying harder. The protocol surfaces infrastructure bugs because every plateau triggers "check the pipeline itself".

---

## Design principles

- **Infrastructure before iteration** — the first job of the skill is to write `score.py`, `measure.py`, and `select_next.py`. Without a reliable score, more rounds just add noise.
- **State beats memory** — `lessons.md`, `research_log.md`, and `best/` persist between sessions. Claude's in-context "I remember" is irrelevant here.
- **Hypothesis before attempt** — `hypothesis.md` is written *first*, and it must be falsifiable with an expected Δ. No hypothesis → no round.
- **Failed rounds are data** — ✗ rounds stay on disk with their failed attempt and hypothesis. They constrain future rounds ("don't re-try what didn't work").
- **Lessons are rules, not stories** — `lessons.md` entries read like rules in a playbook. Narrative goes elsewhere.
- **Laggard priority** — rotation allocates attention to the weakest-scoring target when the gap exceeds 0.10. Prevents always optimizing the easy ones.
- **One hypothesis per round** — single variable preferred. Bundles OK only when a group of changes is one conceptual shift (e.g., "re-anchor all y-positions together").

---

## Requirements

- **Claude Code** with `/loop` skill available (bundled by default)
- **A working directory** you want the research project in (`cd` there before invoking)
- **Whatever domain-specific toolchain** your metric needs (Python + Pillow for images, a test runner for code, etc.) — the skill will detect and reuse what's installed

---

## FAQ

**Q: Will it keep running if I close Claude Code?**
No — `/loop` is session-local. For truly persistent research, pair with `/schedule` (Anthropic cloud cron) or set up a wrapper that restarts Claude Code hourly.

**Q: Can I interrupt and redirect?**
Yes. Hit Esc or just type instructions. Claude will read the current `state/` and fit your intent into the ongoing protocol.

**Q: What if I want to change the metric mid-project?**
Edit `tools/score.py` and document the change with a `## Hotfix` section in `research_log.md`. All existing bests should be re-scored so future rounds are comparable.

**Q: How many rounds does it typically take?**
Depends on the problem. In the UI recon case study, 9 rounds moved the hardest target from 0.746 → 0.778 (~4% absolute). Closing the last ~20% usually requires a Phase shift (e.g., introducing external assets, a judge model, etc.).

**Q: Is there a cost control?**
Each iteration is one Claude turn with image reads + CSS edits + one Bash call — typically cheap. Default 30-minute pacing gives ~2 iters/hour. Adjust by telling the loop `/loop 1h …` instead.

---

## License

MIT. Use, fork, modify freely.

---

## Credits

Developed through dogfood on a UI pixel reconstruction research project in April 2026. Skill text is the distilled protocol; the meta-lessons in the case study are the real payload.
