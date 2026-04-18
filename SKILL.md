---
name: skill-gardener
description: A long-term Skill cultivation system that diagnoses, evolves, and validates SKILL.md files using real usage data, pairwise preferences, diverse candidate generation, and style anchors. Use this skill whenever the user asks to "improve a skill", "optimize a skill", "audit a skill", "check if this skill is good", "make this skill better", "评估 skill", "打磨 skill", "升级 skill", "诊断 skill", when the user shares a SKILL.md file and asks for feedback, or when the user mentions any Skill has been producing unsatisfying outputs. Works across functional, aesthetic, and creative Skill types.
---

# Skill Gardener

> A gardener, not a trainer. Skills aren't models — they're living artifacts. This system observes how a Skill actually performs, diagnoses specific failures, proposes diverse fixes, and validates improvements against real usage. No magic numbers. No single scores. Just patient, evidence-based cultivation.

---

## Why this exists

Most Skill review tools rely on a single LLM-generated score. That fails for three reasons:

One — Skill quality is multi-objective. A Skill can get better at intent-matching while getting worse at style fidelity. A single number hides the tradeoff.

Two — synthetic test prompts have low signal. Two or three imagined prompts cannot approximate the distribution of real usage. They also invite overfitting.

Three — "improvement" claimed by an LLM that just wrote the improvement is not trustworthy. Same model, same session, same biases.

Skill Gardener replaces absolute scoring with **pairwise preference**, synthetic prompts with **real usage logs**, and single-candidate hill-climbing with **parallel candidate tournaments**. It also separates Skills by type and applies different evaluation logic to each, because an aesthetic Skill and a functional Skill should not be judged by the same rubric.

---

## Core principles

1. **Evaluate on real use, not imagination.** Every Skill under gardening builds a usage log. Optimization pulls evaluation prompts from that log, never from Claude's guesses about what users "might" ask. Split into train and holdout at optimization time; never let the holdout leak into training.

2. **Prefer pairwise over absolute.** Instead of scoring a Skill 78/100, compare two versions on the same prompt and ask which output is better, with evidence. Pairwise judgments are more stable, more debuggable, and directly produce the signal needed for selection.

3. **Diversify before selecting.** When proposing improvements, generate 4-6 candidates from genuinely different strategies, not one. Run them in a tournament. This is the only way to approximate evolution; single-candidate iteration is just hill-climbing in a costume.

4. **Respect Skill type.** Functional, aesthetic, and creative Skills have different failure modes and different success criteria. A single rubric applied to all three will silently deform the aesthetic ones. See `references/skill-types.md`.

5. **Build the user's taste, not just the Skill.** Every time the user casts a tie-breaking vote between two candidates, that vote enters a persistent taste database. Over time, the judge model is calibrated by the user's own preferences — across Skills, across sessions, across years.

6. **Extract principles, don't just apply diffs.** When an optimization succeeds, write down the general lesson it taught. A Skill-specific fix that teaches a cross-Skill principle is ten times more valuable than a fix that teaches nothing.

7. **Be honest about uncertainty.** When the margin between two candidates is inside the noise band, say so and ask the human. Do not fake confidence to seem decisive.

---

## Commands

Natural language works everywhere. For power users, short commands also work.

| Command | Natural language trigger | What it does |
|---|---|---|
| `~cultivate [skill]` | "cultivate X", "improve X", "打磨 X" | Full cycle: diagnose, generate candidates, arena, validate, commit |
| `~diagnose [skill]` | "diagnose X", "what's wrong with X", "X 问题出在哪" | Analysis only, no changes to Skill files |
| `~anchor [skill]` | "set up anchors for X", "给 X 建锚点" | Create or update style anchors for an aesthetic/creative Skill |
| `~log [skill]` | "show X's usage log" | Inspect what's been recorded for a Skill |
| `~principles` | "show principles", "看看学到的原则" | Browse the cross-Skill principles library |
| `~taste` | "show taste database" | View or export the user's preference history |
| `~review` | "show gardening history" | Summary of all optimizations across all Skills |

If Skill name is omitted, Skill Gardener asks the user to pick from available Skills.

---

## First-time setup

On first invocation, detect whether `_runtime/` exists in the Skill Gardener installation folder. If not, run setup:

1. Create `_runtime/principles.md`, `_runtime/taste-db.jsonl`, `_runtime/logs/`.
2. Ask the user to classify their existing Skills into types (functional / aesthetic / creative / mixed). See `references/skill-types.md` for the distinction. Store classifications in `_runtime/skill-registry.yaml`.
3. Explain to the user that meaningful optimization needs usage data, and offer three paths:
   - **Live mode**: start logging now, revisit in a week or two when logs have accumulated.
   - **Retrospective mode**: paste recent Skill outputs (and ideally the prompts that produced them) into `_runtime/logs/<skill-name>/retrospective.jsonl`.
   - **Cold start mode** (least reliable): proceed with Claude-generated test prompts, but flag all results as lower-confidence.

Say this honestly: live mode produces the best results, but requires patience. Retrospective mode is a good compromise. Cold start mode is available but its outputs should be trusted less.

---

## The cultivation cycle

Invoked by `~cultivate [skill]`. Eight stages.

### Stage 1 — Prepare

Check the target Skill exists. Read its manifest (see `references/skill-types.md` for format). If no manifest exists, generate one with the user's input and save it alongside the Skill. Classify the Skill: functional, aesthetic, creative, or mixed.

Check the usage log. If fewer than 10 entries exist and the Skill is not in cold-start mode, warn the user and ask whether to proceed anyway or wait for more data.

Create a git branch: `gardener/<skill-name>/YYYYMMDD-HHMM`. All changes live on this branch until accepted.

### Stage 2 — Diagnose

Load `references/failure-modes.md`. For each failure mode in the catalog relevant to this Skill's type, scan the usage log for instances. Output a prioritized **failure report**:

- Which failure modes are present, how many times, with log-entry references
- Which failure mode has the highest frequency × severity product — this becomes the target

Show the failure report to the user. **Stop and wait for confirmation** before moving to Stage 3. The user can redirect: "ignore mode X this time, focus on Y."

### Stage 3 — Generate candidates

Load `references/candidate-strategies.md`. For the target failure mode, identify 4-6 candidate strategies from different categories (not five variations of "add a constraint"). Prefer diverse strategies over similar ones, even if some seem less likely to work — diversity is the point.

For each strategy, spawn an independent sub-agent to draft the concrete change. Each sub-agent receives:
- The full current SKILL.md
- The target failure mode with evidence
- The specific strategy it should apply
- The constraint: **do not change the Skill's core intent, only how it achieves that intent**

Each sub-agent returns:
- A proposed new SKILL.md
- A one-paragraph explanation of what it changed and why
- A prediction of one side-effect that might appear

Save all candidates to `_runtime/logs/<skill-name>/cultivation-<timestamp>/candidates/`. Never merge into the Skill directory at this stage.

### Stage 4 — Arena

Load `references/arena-protocol.md`. Sample prompts from the usage log — use 60% as the training arena set (used here), reserve 40% as the holdout (used in Stage 6).

For each pair of candidates, for each arena-set prompt, spawn a **judge sub-agent** that:
1. Receives both outputs (A and B) **without knowing which strategy produced which**
2. Outputs a preference (A / B / tie) plus a one-sentence reason
3. The match is then **run again with A and B swapped** to check for position bias
4. If the judge flips on swap, the match is counted as a tie

Tally wins, losses, and ties. Compute simple win rates. For aesthetic and creative Skills, also run the **anchor check** (see `references/anchor-system.md`): candidates that drift from the style anchors beyond the configured threshold are automatically disqualified regardless of arena performance.

### Stage 5 — Human oracle (conditional)

If the top candidate's win margin over the second candidate is within the noise band (default: win rate difference < 15%, or fewer than 8 decisive matches), the arena result is "close call." Close calls go to the user:

- Present the top 2-3 candidates with their win rates
- Show 3-5 representative head-to-head prompts where they diverged
- Ask the user to vote on each pair
- Record every vote in `_runtime/taste-db.jsonl`

The user's votes determine the winner. If the user declines to vote, the arena's top candidate wins by default, but the result is flagged as "weak preference."

For wins that are *not* close calls, the arena decides alone.

### Stage 6 — Holdout validation

Take the winning candidate and run it against the **holdout** prompts (the 40% never used in arena). Run the original Skill against the same holdout prompts. Judge the pairs, same protocol as Stage 4.

If the candidate still wins on holdout: confirmed improvement.

If the candidate performs worse or equal on holdout: **overfitting detected**. Report to the user and recommend rejection. The user can override, but the default is to reject.

### Stage 7 — Commit and learn

If the winner survived Stage 6:

1. Write the new SKILL.md to the Skill directory
2. `git commit` on the gardener branch with a detailed message: the failure mode, the strategy that won, the arena and holdout win rates, a reference to the cultivation log
3. **Extract a principle.** Spawn a sub-agent with the before/after diff, the failure mode, and the winning strategy. Ask it to articulate the general lesson in one paragraph, tagged with Skill types it likely applies to. Append to `_runtime/principles.md`.
4. Ask the user to review the commit and merge to master, or request additional iteration.

If no winner survived: commit nothing. Write a `_runtime/logs/<skill-name>/cultivation-<timestamp>/outcome.md` explaining why (arena indecisive, holdout rejected, anchors failed). Ask the user whether to try a different failure mode or stop for now.

### Stage 8 — Report

Present a clean summary:

- Target failure mode
- Winning strategy (or: no winner)
- Arena and holdout win rates
- Side effects observed or confirmed absent
- New principle extracted (if any)
- Suggested next step (another round, different failure mode, or stop)

Do not summarize with a single score. If the user asks "is it better now?", answer with the evidence: "On 23 of 40 matched holdout prompts, the new version was preferred; on 9 the old version was preferred; 8 were ties. The dominant failure mode that prompted this round (verbosity bloat) dropped from 31% to 8% in the holdout sample."

---

## How to handle the three Skill types

See `references/skill-types.md` for full definitions. Here is the short version of how each type changes the cycle:

**Functional Skills** (workflow automation, data processing, API integration): emphasize arena on real usage logs. Low weight on style anchors. Common failure modes: intent drift, format violation, over-constraint.

**Aesthetic Skills** (design, layout, visual generation): **anchors are mandatory.** Before cultivation can begin, the user must establish 5-10 positive anchors (outputs they love) and 3-5 negative anchors (outputs that miss the mark). The arena still runs, but anchor similarity is a hard gate. A candidate that wins the arena but drifts from anchors is rejected.

**Creative Skills** (writing, storytelling, ideation): combination of the above plus a dedicated AI-voice detector. Common failure modes: formulaic phrasing, tonal drift, over-structured output. User oracle is triggered more often because creative judgment is harder to delegate.

**Mixed Skills**: treat dominant dimension first, secondary dimension as a hard check.

---

## When to refuse or defer

Do not run cultivation when:

- The Skill has fewer than 5 log entries and the user refuses to supply retrospective data or anchors. Explain why: optimization without evidence is theater.
- The user asks for a "full score" or "overall rating." Explain that this system deliberately does not produce a single number, and offer the failure report as a more useful alternative.
- The user wants to optimize a Skill they didn't write and don't own. Gardening modifies the Skill file; confirm ownership first.
- The Skill's core intent is unclear. A Skill that cannot state what success looks like cannot be optimized. Run the manifest-creation subflow first.

---

## Constraint rules

1. **Never change a Skill's declared intent**, only how it achieves that intent. The manifest is the contract.
2. **Never exceed 150% of original file size** unless the user explicitly asks for a rewrite.
3. **Never introduce new external dependencies** (new scripts, new references, new network calls) without explicit user approval.
4. **Never commit to the main branch directly.** All gardener work lives on `gardener/*` branches until user-approved merge.
5. **Never skip the holdout.** If the holdout set is empty, optimization halts; the user must supply more log data.
6. **Never collapse the evaluation to a single score** in any output.
7. **Respect stop requests.** If the user says "stop", halt immediately and report state.

---

## Output etiquette

Reports should:

- Lead with the decision (improved / unchanged / rejected), not the process
- Show evidence (match counts, specific prompts, diff highlights) before conclusions
- Mark confidence levels explicitly ("strong preference," "weak preference," "cold-start estimate")
- Name the failure modes being addressed in plain language, not jargon

Reports should not:

- Produce a single composite score
- Claim improvements not validated on holdout
- Hide the rejected candidates — list them with their win rates
- Use celebratory language ("great progress!"). State the result plainly.

---

## Reference files

Load on demand when the relevant stage needs them:

- `references/skill-types.md` — Skill type taxonomy, manifest format, per-type logic
- `references/failure-modes.md` — Catalog of failure modes with detection heuristics and examples
- `references/candidate-strategies.md` — Menu of improvement strategies, organized by failure mode
- `references/arena-protocol.md` — Detailed pairwise comparison and judge-agent protocol
- `references/anchor-system.md` — How to build, maintain, and evaluate against style anchors

Runtime state (created on first use, not shipped):

- `_runtime/principles.md` — accumulated cross-Skill principles
- `_runtime/taste-db.jsonl` — user preference votes, line-per-vote
- `_runtime/skill-registry.yaml` — classification and manifest paths for each Skill under gardening
- `_runtime/logs/<skill-name>/` — usage logs, cultivation records, and sub-agent outputs

---

## A note on honesty

This system is not magic. It cannot turn a bad Skill into a great one without real data about how the Skill is failing. It cannot evaluate aesthetic quality without the user's own reference examples. It cannot prove improvement beyond what the holdout set can tell us.

What it can do, reliably, is: surface specific failures with evidence, generate diverse candidate fixes, let them compete on real prompts, validate that wins hold up on held-out data, and accumulate a principles library and a taste database that become more valuable over time.

Think of it as a long-term relationship with your Skills, not a one-shot optimizer. The value compounds across months, not minutes.
