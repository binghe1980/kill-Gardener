# Skill Gardener

> A long-term cultivation system for Claude Code Skills.  
> Pairwise preferences over single scores. Real logs over synthetic prompts. Parallel candidates over hill-climbing. Style anchors over rubric adjectives.

---

## What this does

Skill Gardener helps you evolve your SKILL.md files over time by:

1. **Recording how they actually get used** — not how you imagine they get used
2. **Diagnosing specific failures** from a catalog of known Skill failure modes, backed by log evidence
3. **Generating multiple diverse candidate fixes** instead of one speculative improvement
4. **Running a pairwise tournament** between candidates on real prompts from your usage history
5. **Validating wins on held-out prompts** to catch overfitting
6. **Gating aesthetic and creative Skills against your reference examples** (anchors), not abstract rubrics
7. **Extracting cross-Skill principles** from each successful optimization so the library gets smarter over time
8. **Building a taste database** from your tie-breaking votes, so the judge learns your preferences

---

## What it does NOT do

Honesty matters more than marketing here:

- **It does not produce a single quality score.** Deliberately. A single number compresses multi-dimensional tradeoffs and invites gaming.
- **It does not optimize Skills without evidence.** If you have no usage logs and refuse to supply anchors or retrospective data, the system will decline rather than hallucinate improvements.
- **It does not guarantee improvements.** Sometimes no candidate survives the holdout. The system will tell you rather than commit a questionable change.
- **It is not magic.** The quality of optimization is bounded by the quality of your log data, your anchors, and your feedback. Garbage in, garbage out still applies.
- **It does not pretend sub-agent evaluation is truly independent.** Claude judging Claude's outputs has known biases. The protocol mitigates them; the user oracle catches the rest.

If you want a one-click "make my Skill better" button, this isn't it. If you want a system that grows with your work over months and accumulates real knowledge about your Skills and your taste, start here.

---

## Core design choices

### Pairwise, not absolute

LLM judges are unstable when giving absolute scores (78? 82? 85?) but relatively stable when choosing between two options. The arena uses pairwise comparisons exclusively. Outputs: win counts, reasons, and confidence intervals — not "this Skill is now 87/100."

### Real usage, not synthetic prompts

Two or three imagined test prompts cannot represent real usage distribution. Skill Gardener uses your actual invocation logs (collected passively) as the evaluation set, split into train and holdout. Synthetic prompts are a cold-start fallback only, and flagged as lower-confidence.

### Parallel candidates, not single-point iteration

For each round, 4-6 candidates are generated from different strategy categories (tighten, demonstrate, remove, reorder, decompose, scaffold, reframe, rewrite). They compete in a tournament. This is the only way to approximate evolution; one-candidate-at-a-time iteration is just hill-climbing.

### Anchors, not rubric adjectives

"Warm, refined, authoritative" is not a specification for aesthetic Skills. Reference examples are. For any aesthetic or creative Skill, the system requires 5-10 positive anchors and 3-5 negative anchors. These gate every candidate before it's considered a winner.

### Type-specific cycles

Functional, aesthetic, and creative Skills have different failure modes and need different evaluation logic. The cycle adapts to the Skill type. Forcing a single rubric across all types silently damages the types whose success criteria don't match the rubric's assumptions.

### Principles across Skills

When an optimization wins, the system extracts a general principle (e.g., "for Skills claiming colloquial voice, few-shot examples beat banned-phrase lists"). These accumulate in a principles library that informs future candidate generation for other Skills. The Skill you optimize today makes the next Skill's optimization better.

### Taste database as long-term asset

Every tie-breaking vote you cast is recorded. Over time, this becomes your personal calibration file — a high-fidelity digitization of your aesthetic judgment. It's portable, inspectable, and grows more valuable the longer you use the system.

---

## Installation

```bash
# Clone or copy the skill-gardener folder into your Claude Code Skills directory
cp -r skill-gardener ~/.claude/skills/
```

First invocation will set up runtime state (`_runtime/` folder for logs, principles, taste database).

---

## Basic usage

```
# First time: classify your Skills and set up logging
"help me set up skill-gardener"

# Diagnose a Skill without changing it
"diagnose my landing-page Skill"

# Run a full cultivation cycle
"cultivate my landing-page Skill"

# For aesthetic/creative Skills: set up anchors first
"set up anchors for my landing-page Skill"

# Review what's been learned
"show me the principles library"
"show me my taste database"
"show me the gardening history"
```

Natural language works throughout. For power users, short commands (`~cultivate`, `~diagnose`, `~anchor`, `~log`, `~principles`, `~taste`, `~review`) are equivalent.

---

## When to use it

**Good fit:**

- You have 5+ Skills you use regularly and want to maintain over time
- You have a body of historical Skill output to seed anchors or retrospective logs
- You're willing to invest 15-30 minutes setting up each Skill for gardening
- You care about quality enough to review close-call decisions rather than auto-accept
- You want your Skill maintenance practice to compound knowledge over months

**Poor fit:**

- You have one or two Skills you rarely use
- You want a one-shot optimizer that runs unattended
- You're not willing to set up anchors for aesthetic Skills (the system won't lie about having optimized them)
- You expect a single "is it good?" score as output
- You need this to work without usage data and have no reference examples

---

## Architecture overview

```
skill-gardener/
├── SKILL.md                       # Entry point — the cultivation orchestrator
├── README.md                      # This file
└── references/
    ├── skill-types.md             # Functional/Aesthetic/Creative/Mixed taxonomy + manifest
    ├── failure-modes.md           # Catalog of failure modes with detection heuristics
    ├── candidate-strategies.md    # Menu of improvement strategies by failure mode
    ├── arena-protocol.md          # Pairwise comparison and judge-agent protocol
    └── anchor-system.md           # How to build and maintain style anchors
```

Runtime state (generated, not shipped):

```
_runtime/
├── principles.md                  # Cross-Skill principles learned over time
├── taste-db.jsonl                 # One line per user preference vote
├── skill-registry.yaml            # Skill classifications and manifest paths
└── logs/<skill-name>/
    ├── usage.jsonl                # Passive usage log
    └── cultivation-<timestamp>/   # One folder per cultivation round
        ├── diagnosis.md
        ├── candidates/
        ├── arena-results.md
        ├── holdout-results.md
        └── outcome.md
```

---

## Honest limitations

The design section above alluded to these; here they are plainly:

- **Sub-agent independence is imperfect.** Same model family, same training. The arena protocol mitigates position bias, length bias, and self-preference to a degree, but cannot eliminate systematic biases that affect all Claude-family models equally. The user oracle is the main remedy.

- **Usage logging requires setup.** Claude Code does not natively record all Skill invocations in a structured log. You will need to either (a) manually append to `_runtime/logs/<skill-name>/usage.jsonl` after significant sessions, or (b) rely on retrospective / cold-start modes. A future version may include a session-hook for automatic logging.

- **Compute cost is real.** A full cultivation round for one Skill involves 4-6 candidate generations, 300+ arena judgments, anchor gating, and holdout validation. For a large Skill library, plan for this. Budget-aware mode (`~cultivate --fast`) reduces judgments at the cost of confidence.

- **Principles can be wrong.** A principle extracted from one successful optimization may not generalize. The library should be reviewed periodically and principles marked as deprecated when they've misled the system.

- **Cold-start mode is genuinely worse.** If you run without logs, anchors, and retrospective data, the system falls back to Claude-designed test prompts. This works but produces lower-confidence results. Don't ship confidence you didn't earn.

---

## License

MIT.

---

## A closing note

Skills are long-term artifacts. They don't benefit from bursty one-shot optimization; they benefit from patient observation and principled adjustment. If you're looking for something that makes your Skills dramatically better in one afternoon, this isn't it. If you're looking for a system that, over months, makes your Skill library quietly and measurably more useful while teaching you something about your own taste — that's the promise.

Contributions welcome. Issues for failure modes the catalog missed are especially welcome.
