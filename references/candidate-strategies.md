# Candidate Strategies

Menu of improvement strategies. Organized by failure mode, with guidance on when each strategy tends to help and when it tends to backfire.

The single most important rule: **when generating candidates, pick strategies from different categories**. Five variants of "add a constraint" is not diversity. A constraint, a few-shot, a reorder, a removal, and a checkpoint — that's diversity.

---

## Strategy categories

Every strategy below belongs to one of these categories. Stage 3 should sample candidates across categories, not within.

1. **Tighten** — add constraints, schemas, explicit rules
2. **Demonstrate** — add examples, counterexamples, few-shot pairs
3. **Remove** — delete contradicting, stale, or redundant instructions
4. **Reorder** — change sequence of steps or priority of rules
5. **Decompose** — split a complex instruction into staged sub-steps
6. **Scaffold** — add checkpoints, confirmations, or self-check prompts
7. **Reframe** — change the persona or mental model the Skill adopts
8. **Rewrite** — structural reorganization of SKILL.md (use sparingly)

---

## Strategies by failure mode

### A1. Intent drift

**Tighten** — add an explicit "clarify before acting" step when input is ambiguous. Risk: over-triggers clarification for clear cases, annoying users. Mitigation: specify what counts as ambiguous.

**Demonstrate** — add a few-shot pair showing an ambiguous input and the correct clarification response. Usually more effective than rules for intent problems.

**Scaffold** — add a "restate the user's request in your own words before responding" step. Risk: adds latency and visible ceremony. Best for high-stakes Skills.

**Reframe** — change the Skill's declared persona to one that naturally seeks intent (e.g., "a senior editor who always confirms scope" rather than "an expert at X"). Risk: changes voice.

**Avoid:** tightening the output format to fix an intent problem. Wrong layer.

### A2. Trigger misfire

**Tighten** — rewrite the description's "when to use" clause with more specific trigger phrases. The description is the primary trigger mechanism.

**Demonstrate** — add trigger examples and counter-trigger examples to the description ("Use when user asks X. Do NOT use when user asks Y, which sounds similar but requires a different approach.")

**Remove** — delete over-broad trigger phrases that pull the Skill into irrelevant contexts.

**Avoid:** changing SKILL.md body to fix a triggering problem. The body only matters after the Skill has been selected.

### A3. Context dropout

**Tighten** — add an explicit "list all user-provided constraints before responding" step.

**Scaffold** — add a final self-check: "did your output address each of these inputs: [listed]?"

**Demonstrate** — show a pair where specific user context was preserved vs. dropped.

**Avoid:** making the entire Skill more verbose to ensure context coverage. Creates verbosity bloat.

### B1. Format violation

**Tighten** — add an explicit output schema or template block. Make it visually distinct in SKILL.md.

**Demonstrate** — provide a complete example of the correct format, not just a description.

**Scaffold** — add a final format validation step before output.

**Avoid:** piling on format rules without examples. Rules alone are less effective than one complete example.

### B2. Verbosity bloat

**Remove** — delete instructions that encourage preamble, summary, or meta-commentary. Common culprits: "walk the user through your reasoning", "explain your approach".

**Tighten** — add explicit anti-bloat rules targeting specific phrases. Use a banned-phrase list rather than abstract "be concise."

**Demonstrate** — show a bloated output and a tight output side by side with commentary.

**Reframe** — shift persona toward brevity ("speak like a busy senior engineer"). Risk: may overshoot into curtness.

**Avoid:** adding "be concise" without specificity. It's ignored because every Skill already tries to be.

### B3. Under-specification leak

**Tighten** — add the specific parameter/format/value that's been varying.

**Demonstrate** — lock in the expected form with a concrete example.

**Decompose** — if the Skill handles multiple cases, split into case-specific branches, each tightly specified.

**Avoid:** rewriting the whole Skill to fix one underspecified section.

### B4. Over-specification rigidity

**Remove** — delete overly prescriptive rules, especially those that forbid contextually-warranted variation.

**Reframe** — shift from "always do X" to "default to X unless user signals Y."

**Scaffold** — add an explicit "if the default doesn't fit this case, explain why and adapt" branch.

**Avoid:** adding more rules to compensate. Rigidity is rarely solved by additional rules.

### C1. Instruction-example contradiction

**Remove** — delete one side of the contradiction. Usually easier to fix the examples than to fix the rules that everyone ignored.

**Demonstrate** — replace contradicting examples with aligned ones.

**Rewrite** — if contradictions are systemic, restructure the Skill around a consistent internal logic.

**Avoid:** adding meta-commentary like "when the examples conflict with rules, follow the rules." The model won't.

### C2. Stale assumption

**Remove** — delete the stale references.

**Tighten** — replace with current references.

This is housekeeping, not optimization. One round per staleness event.

### C3. Dependency break

Fix the broken dependencies or remove them. Not really optimization territory — just maintenance.

### D1. AI-voice leakage

**Demonstrate** — add 3-5 before/after examples showing AI-voice rewriting into natural voice. Few-shot is dramatically more effective here than rules.

**Remove** — delete instructions that subtly encourage formal/structured output when the Skill claims casual voice. E.g., "provide a comprehensive analysis" will produce formal voice regardless of other constraints.

**Scaffold** — add a final self-check pass that scans for banned phrases and rewrites them.

**Reframe** — make the declared persona more specific. "A writer" produces AI voice; "a specific writer who talks like [reference]" tends to produce more natural voice.

**Avoid:** listing banned phrases as the only fix. The model may avoid those specific phrases but produce similar-feeling alternatives. Pair with examples.

### D2. Tonal drift

**Demonstrate** — add a reference example showing the declared tone in action, with commentary on what makes it work.

**Reframe** — tighten the persona declaration to include tonal specifics.

**Avoid:** prescribing tone via adjective lists ("warm, authoritative, but accessible"). Adjectives are too abstract. Examples work better.

### D3. Over-structured creative output

**Remove** — delete instructions to "organize your output with headers" or similar when the Skill is creative.

**Demonstrate** — add a prose-form reference output.

**Tighten** — explicitly instruct "deliver as flowing prose, not bullets" when appropriate.

### E1. Style drift

**Primary action:** check anchors. Style drift is usually an anchor-management problem, not a SKILL.md problem. Update anchors if they're stale.

**Tighten** — add explicit style constants (exact color values, exact font names, exact spacing values) to SKILL.md if they're generalizable.

**Demonstrate** — expand the set of anchor examples referenced in SKILL.md.

**Avoid:** trying to describe style in prose. "Warm, refined, cinematic" is not a specification. Anchors are the specification.

### E2. Structural inconsistency

**Tighten** — add explicit structural rules (e.g., "every card in this layout has 24px vertical padding").

**Decompose** — if the layout has recurring elements, define them once and reference by name.

**Scaffold** — add a final consistency check ("verify all cards have identical padding before output").

### E3. Reference contamination

**Remove** — remove over-specific anchor content from the Skill's visible context. Anchors should be in the anchor folder, not pasted into SKILL.md.

**Reframe** — describe the style pattern abstractly in SKILL.md, reserve specifics for anchor folder.

### F1. User confirmation skipped

**Scaffold** — insert explicit confirmation checkpoints. This is the fix.

**Tighten** — specify exactly what triggers confirmation and what the confirmation prompt should say.

### F2. Runaway length

**Tighten** — declare a max output length in the Skill.

**Scaffold** — add a length self-check near the end of the workflow.

**Decompose** — if length is driven by trying to cover too many cases, split into multiple narrower Skills.

### F3. Silent fallback to generic

**Scaffold** — add an explicit "if you encounter an edge case you can't confidently handle, tell the user rather than producing generic output" instruction.

**Demonstrate** — show what acknowledging-an-edge-case looks like.

---

## Strategy generation heuristics

When generating 4-6 candidates for Stage 3:

- **At least one from a different category than the "obvious" fix.** If the obvious move for intent drift is "add clarify-before-acting," also try a demonstrate strategy and a reframe strategy. Obvious moves win often enough, but the non-obvious ones teach more when they do win.
- **Include at least one "do less" candidate.** Our bias is to add instructions. Deleting instructions is often the right move and rarely tried. Every candidate batch should include one "remove" option if plausible.
- **Avoid strategies that violate the 150% size cap.** If the candidate's change blows the budget, shrink the Skill in another way to stay under.
- **Never combine strategies in a single candidate.** One candidate = one strategy = one dimension changed. Attribution requires it.

## Anti-patterns

- **"Belt and suspenders"** candidates that apply two strategies simultaneously. Can't attribute which one helped.
- **"Copy what works elsewhere"** candidates that import patterns from other Skills without confirming they fit here. Cross-Skill principles should come from the principles library, applied deliberately.
- **"Add one more rule"** candidates for Skills that already have too many rules. Rule fatigue is real.
- **"Rewrite from scratch"** candidates triggered casually. Stage 3 is iterative; full rewrites should only be proposed after multiple rounds of failure, and require explicit user consent.

## What to record about each candidate

When a sub-agent returns a candidate, it should output:

```yaml
candidate_id: c1
strategy_category: demonstrate
strategy_description: "Added 3 before/after pairs showing intent clarification."
target_failure_mode: A1_intent_drift
changed_sections:
  - "Stage: Triage"
  - "new Examples section"
size_delta_pct: +12
predicted_side_effect: "May increase output length for simple queries; needs holdout check."
```

This makes every candidate self-documenting and traceable.
