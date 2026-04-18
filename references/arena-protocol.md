# Arena Protocol

The arena is where candidates compete. Protocol design decisions here determine whether the system actually learns or just produces noise. This document specifies the protocol and why each detail matters.

---

## Design goals

1. **Reduce judge bias.** LLM judges have well-documented biases: position bias (favor the first or last option), length bias (favor longer outputs), self-preference bias (favor outputs from the same model family), and stylistic drift (favor more formal or more structured responses regardless of fit). The protocol must counteract as many of these as practical.

2. **Produce debuggable decisions.** Every win or loss should come with reasoning that a human can inspect. If a candidate wins on reasons the user disagrees with, that's important information.

3. **Surface uncertainty.** When the arena can't decide, say so. Pretending to decide is worse than deferring to human judgment.

4. **Stay within budget.** Comparisons cost tokens. The protocol must be efficient enough to run dozens of matches per cultivation round.

---

## Arena structure

For a cultivation round with N candidates (typically 4-6) and M arena prompts (typically 15-25):

- Each candidate is paired against every other candidate → N×(N-1)/2 pairs
- For each pair, every prompt produces one match → M matches per pair
- Each match is run twice with output positions swapped → 2M judgments per pair
- Total judgments per round: N×(N-1)/2 × 2M

For N=5, M=15: 10 pairs × 30 judgments = 300 judgments. Each judgment is a sub-agent call with one prompt's two outputs and the judging rubric.

### Why this much

- Pairwise (not all-at-once) — LLMs are dramatically more consistent judging pairs than ranking many options
- Every pair (not just against a reference) — catches cases where the reference is the problem
- Position swap — eliminates single-pair position bias
- Multiple prompts — catches prompt-specific wins

If budget is constrained, reduce M before reducing N. Diversity of candidates matters more than number of arena prompts.

---

## The match protocol

For each match (candidate A vs candidate B on prompt P):

### Step 1: Produce outputs

Spawn two sub-agents in parallel:
- Sub-agent 1 runs candidate A's SKILL.md on prompt P → output O_A
- Sub-agent 2 runs candidate B's SKILL.md on prompt P → output O_B

Both sub-agents receive identical prompt and context. They do not know they are being compared.

### Step 2: Judge in position (A, B)

Spawn a judge sub-agent with:

```
You are an independent judge evaluating two outputs of a Skill. You do not know
which version produced which output.

The Skill's declared intent:
<manifest.intent.one_line>

The user's prompt:
<P>

Output 1:
<O_A>

Output 2:
<O_B>

Evaluation criteria specific to this Skill type:
<per-type rubric, see below>

Respond in this format:
preference: 1 | 2 | tie
primary_reason: <one sentence>
supporting_evidence: <what in the output supports this; quote if needed>
concerns_about_winner: <one sentence; if none, say "none">
```

### Step 3: Judge in position (B, A)

Spawn a fresh judge sub-agent with outputs in reversed order. Same prompt template, but O_B is labeled "Output 1" and O_A is labeled "Output 2".

### Step 4: Reconcile

- If both judges prefer A (regardless of position label): A wins this match
- If both judges prefer B: B wins this match
- If judges flip on swap: match is a **tie** (position bias detected)
- If one judge says tie and the other picks a side: half-point to the picked side, treat as weak signal

Record the reconciled result plus both judges' reasoning.

### Step 5: Tally

Across all M prompts for this pair: compute wins, losses, ties, and a win-rate differential.

---

## Per-type judging rubrics

### Functional rubric

```
Evaluate the two outputs against these criteria in order of priority:
1. Task completion: does the output do what the user asked?
2. Correctness: is the output factually and structurally correct?
3. Format adherence: does the output follow the declared format?
4. Economy: is the output appropriately concise for the task?
```

### Aesthetic rubric

```
Evaluate the two outputs against these criteria in order of priority:
1. Style consistency with anchors: which output better embodies the declared aesthetic?
2. Internal consistency: within the output, is the style applied consistently?
3. Compositional quality: layout, hierarchy, balance
4. Task completion: does it do what was asked?

Reference anchor images/examples are attached below for grounding.
```

### Creative rubric

```
Evaluate the two outputs against these criteria in order of priority:
1. Voice authenticity: does the output sound like the declared voice, or does it sound AI-written?
2. Task completion: does it accomplish what the user asked?
3. Structural fit: does the structure (prose/list/dialogue) match what the content warrants?
4. Economy: is it free of filler and AI-characteristic phrasing?
```

### Mixed rubric

Combine the dominant-dimension rubric with the secondary-dimension rubric as a secondary check. The judge evaluates primarily on dominant, and if both candidates are approximately equal on dominant, breaks the tie using secondary.

---

## Close-call detection

After the round completes, identify whether the result is a close call:

```
top_candidate_win_rate - second_candidate_win_rate < noise_band
OR total_decisive_matches < min_matches
```

`noise_band` and `min_matches` are in the manifest. Defaults: 0.15 and 8.

If close call: escalate to user oracle (see below).
If not close call: top candidate proceeds to holdout validation.

---

## User oracle protocol

When triggered:

1. Present the top 2-3 candidates with their arena win rates
2. Select 3-5 "divergent prompts" — prompts where the candidates produced notably different outputs (not where they produced similar outputs)
3. For each divergent prompt, show both outputs without indicating which candidate produced which
4. Ask the user to vote on each pair, with an optional "why" field

The protocol:
```
Prompt: <P>

Output A:
<O from one candidate>

Output B:
<O from another candidate>

Your preference? A / B / both fine / neither fine
(Optional) Why?
```

Record every vote in `_runtime/taste-db.jsonl` as:
```json
{
  "timestamp": "...",
  "skill": "skill-name",
  "prompt": "...",
  "winner_strategy": "strategy_category_of_chosen",
  "loser_strategy": "strategy_category_of_rejected",
  "user_reason": "...",
  "arena_had_predicted": "winner|loser|tie"
}
```

The `arena_had_predicted` field enables later analysis: when does the arena agree with the user, and when does it disagree? Systematic disagreement points to judge calibration problems.

---

## Taste database: how it improves judging over time

Every user vote is data. After accumulating N votes (suggest N=30 before using), the system can:

1. **Detect judge drift.** If the judge systematically disagrees with the user on certain criteria (e.g., judge prefers longer outputs, user prefers shorter), inject a calibration note into future judge prompts for this user.

2. **Weight strategies by user preference.** If the user consistently prefers "demonstrate" over "tighten" candidates, Stage 3 can skew its candidate generation toward demonstrate strategies. Not exclusively — diversity still matters — but as a tiebreaker.

3. **Detect preference stability.** If the user's votes on similar pairs change over time, surface the drift. Either the user's taste evolved (fine, update the baseline), or the judge is making inconsistent calls.

The taste DB is the user's personal calibration file. Treat it as a long-lived asset.

---

## Anchor gate (aesthetic and creative Skills)

Before any candidate proceeds from arena to holdout, run the anchor gate:

1. Generate candidate output on a sample prompt
2. Compute similarity to positive anchors (should be high) and negative anchors (should be low)
3. If positive similarity below threshold OR negative similarity above threshold: candidate is **disqualified** regardless of arena wins
4. Log the disqualification reason to the round's outcome file

See `references/anchor-system.md` for how similarity is computed.

The anchor gate exists because LLM judges cannot reliably evaluate style fidelity. They'll happily declare a stylistically-drifted candidate "clearer" or "more effective" without registering that the aesthetic contract has been violated. The gate is a separate, more reliable check.

---

## Holdout validation protocol

Once a candidate wins the arena (and passes the anchor gate if applicable):

1. Sample K prompts from the holdout set (40% of the log, never seen by arena). Minimum K = manifest.holdout_config.min_holdout_size, default 5.
2. Run the winning candidate against these prompts → outputs
3. Run the **original** (pre-cultivation) Skill against the same prompts → baseline outputs
4. Judge pairwise using the same protocol as arena (position swap, reconciliation)
5. Report the candidate's win rate on holdout

Decision rule:
- Candidate wins >50% on holdout → **confirmed improvement**, commit
- Candidate wins 40-50% → **weak confirmation**, commit with low-confidence flag
- Candidate wins <40% → **overfit detected**, reject

If rejected, record to the cultivation log and inform the user. Do not automatically try the next candidate — the failure of the arena winner on holdout often means the arena prompts themselves were unrepresentative. The user should investigate before the next round.

---

## Dry-run mode

Sub-agents may not always be available or may be prohibitively expensive. Skill Gardener supports dry-run mode with degraded guarantees:

- Candidates are generated in the main session (lose independence)
- Judgments are made in the main session (lose independence)
- All outputs are marked `confidence: dry_run` in logs
- Do not extract principles from dry-run rounds — the evidence isn't reliable enough to generalize
- Do not overwrite the main Skill from a dry-run round without explicit user confirmation

Dry-run mode is better than no optimization, but its outputs should be treated as preliminary. If something wins in dry-run, re-run it in full mode before committing.

---

## Common judge failure modes

Watch for these patterns; they signal protocol-level problems:

- **Length correlation >0.7**: judge almost always picks the longer output. Position swap won't catch this. Manual spot-check needed; may need to add a length-parity instruction to the judge prompt.
- **Judge flips on swap >30% of the time**: judge is unstable. Try a more explicit rubric, or use a more capable judge model.
- **Judge reasoning doesn't match its preference**: judge said "output A is more concise" but picked B. Flag; may indicate rubric confusion or that the prompt is eliciting performative rather than substantive judgment.
- **User oracle disagrees with judge >50% of the time**: judge is not calibrated to this user's taste. Update judge prompt with user preferences from taste DB; if still problematic, escalate more often to user.

Record these patterns when they appear. The meta-analysis of judge behavior is itself a principle-generating activity.
