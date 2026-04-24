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
- Tasks where rounds don't share state (e.g., independent bug fixes)

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
6. **Shrink step size** — large moves overshoot; try smaller ones
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
  `YYYY-MM-DD HH:MM | tgt=XXX it=NNN score=X.XX Δ=±X.XX [★|✗] | one-line change`
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
cd ~/projects/your-research-project
/research <your topic>
```

Claude will:

1. Enter plan mode, ask 2–3 questions about metric / target format / horizon
2. Survey your parent project for reusable evaluation code
3. Scaffold `tools/`, `targets/`, `iterations/`, `state/`, and a tailored `CLAUDE.md`
4. Run one smoke test to verify the pipeline works
5. Hand off — you then start `/loop` to iterate autonomously

From then on, `/loop` keeps the research moving.

---

## Design principles

- **Infrastructure before iteration** — the first job of the skill is to write `score.py`, `measure.py`, and `select_next.py`. Without a reliable score, more rounds just add noise.
- **State beats memory** — `lessons.md`, `research_log.md`, and `best/` persist between sessions. Claude's in-context "I remember" is irrelevant here.
- **Hypothesis before attempt** — `hypothesis.md` is written *first*, and must be falsifiable with an expected Δ. No hypothesis → no round.
- **Failed rounds are data** — ✗ rounds stay on disk with their attempt and hypothesis. They constrain future rounds ("don't re-try what didn't work").
- **Lessons are rules, not stories** — `lessons.md` entries read like entries in a playbook. Narrative goes in the log.
- **Laggard priority** — rotation allocates attention to the weakest-scoring target when the gap is large. Prevents always optimizing the easy ones.
- **One hypothesis per round** — single variable preferred. Bundles OK only when a group of changes is one conceptual shift.

---

## Requirements

- **Claude Code** with `/loop` skill available (bundled by default)
- **A working directory** you want the research project in (`cd` there before invoking)
- **Whatever domain-specific toolchain** your metric needs (Python + Pillow for images, a test runner for code, etc.) — the skill detects and reuses what's installed

---

## FAQ

**Q: Will it keep running if I close Claude Code?**
No — `/loop` is session-local. For truly persistent research, pair with `/schedule` (Anthropic cloud cron) or wrap Claude Code in a restart loop.

**Q: Can I interrupt and redirect?**
Yes. Hit Esc or just type instructions. Claude will read the current `state/` and fit your intent into the ongoing protocol.

**Q: What if I want to change the metric mid-project?**
Edit `tools/score.py` and document the change with a `## Hotfix` section in `research_log.md`. All existing bests should be re-scored so future rounds are comparable.

**Q: Is there a cost control?**
Each iteration is one Claude turn with artifact reads + edits + a scoring call — typically cheap. Default 30-minute pacing gives ~2 iters/hour. Adjust by telling the loop `/loop 1h …` instead.

---

## License

MIT. Use, fork, modify freely.
