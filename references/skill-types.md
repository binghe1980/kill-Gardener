# Skill Types and Manifests

This document defines the type system Skill Gardener uses to decide how to evaluate and optimize a Skill. The key claim is that one-size-fits-all rubrics deform Skills whose success criteria differ from the rubric's implicit assumptions. Typing the Skill first avoids that.

## The three types (plus a fourth)

### Functional Skill

A Skill whose success is objectively verifiable or mechanically reproducible. The output either completes the user's intent or it doesn't; the criteria are mostly non-aesthetic.

Examples of functional intent:
- "Parse this CSV and output a cleaned version with these columns"
- "Given a meeting transcript, extract action items as a bulleted list"
- "Look up flight prices for these dates and summarize the cheapest options"
- "Generate a valid OpenAPI spec from these endpoint descriptions"

Functional Skills typically have:
- Clear input/output contracts
- Predominantly correctness-based failure (wrong format, missing data, wrong value)
- Minimal subjective judgment required
- Well-defined "does this solve the task?" test

### Aesthetic Skill

A Skill whose success depends on visual, typographical, or design quality. The output may be technically correct and still unacceptable if it violates the intended aesthetic.

Examples of aesthetic intent:
- "Generate a landing page in [specific visual style]"
- "Produce a slide deck with [specific typography and color system]"
- "Design a social card for this content with [specific brand feel]"
- "Render a data visualization with [specific chart grammar]"

Aesthetic Skills typically have:
- Hard-to-verbalize success criteria ("it just feels right")
- Style drift as the dominant failure mode
- Heavy dependence on reference examples (anchors)
- Resistance to rubric-based evaluation

### Creative Skill

A Skill whose success depends on tone, voice, narrative, or conceptual originality. Distinct from aesthetic because the output is primarily linguistic rather than visual.

Examples of creative intent:
- "Write a short-form video script in a colloquial voice about this topic"
- "Draft a chapter of a web novel with the story hooks genre convention expects"
- "Generate blog intro paragraphs that don't sound AI-written"
- "Produce character dialogue with consistent voice across scenes"

Creative Skills typically have:
- AI-voice leakage as a dominant failure mode
- Style/voice consistency as a key criterion
- Strong need for human oracle (judgment isn't easily delegable)
- Subjective but detectable quality signals

### Mixed Skill

A Skill that meaningfully combines two or three of the above. Treat the dominant dimension as primary for evaluation, and treat the secondary dimension as a hard check.

Example: "Generate a newsletter issue with this typography and this voice." Functional requirements exist (the newsletter has a defined structure), but both aesthetic (typography) and creative (voice) dimensions are load-bearing. Pick the one most likely to fail for this user — that's the primary. Let the other dimensions gate-check the result.

## How to classify a Skill

Ask the user this diagnostic sequence:

1. **If the output were technically correct but "felt wrong," would the user still reject it?**
   - Yes → Aesthetic or Creative
   - No → Functional

2. **If rejected, is the felt-wrongness primarily visual or primarily linguistic?**
   - Visual → Aesthetic
   - Linguistic → Creative
   - Both → Mixed

3. **Can the user articulate the success criteria without showing an example?**
   - Yes, fully → Functional-leaning
   - Only partially → Mixed
   - No, only by showing examples → Aesthetic or Creative

4. **How often does the user reject outputs that a stranger would call acceptable?**
   - Rarely → Functional
   - Sometimes → Mixed
   - Often → Aesthetic or Creative

Default to the more evaluation-strict type when uncertain. It's safer to require anchors for a functional Skill than to skip them for an aesthetic one.

## The manifest format

Every Skill under gardening gets a manifest. The manifest is the contract — it declares what the Skill promises to do, and becomes the reference for all evaluation.

Save as `<skill-directory>/manifest.yaml` alongside SKILL.md.

```yaml
name: skill-name
version: 1                      # bump on significant intent changes
type: functional                # functional | aesthetic | creative | mixed
primary_dimension: functional   # if mixed, which dimension dominates

intent:
  one_line: "What this Skill does, in one sentence, in the user's words."
  target_user: "Who invokes this Skill and in what context."
  success_looks_like:
    - "A concrete signal of success (not abstract)."
    - "Another concrete signal."
    - "Ideally 3-5 signals total."

known_failure_modes:
  - mode: "What can go wrong, in plain language."
    severity: high              # high | medium | low
    example: "A specific instance, if available."

style_constraints:              # optional, mostly for aesthetic/creative Skills
  must_have: []
  must_not_have: []
  tone_reference: []            # point to anchor files if any

arena_config:
  noise_band: 0.15              # default; tune per Skill if needed
  min_matches: 8                # minimum decisive matches to avoid close-call flag
  sample_size: 20               # prompts sampled from log per arena round

holdout_config:
  train_ratio: 0.6              # of usage log
  holdout_ratio: 0.4
  min_holdout_size: 5

anchor_paths:                   # required for aesthetic/creative
  positive: _runtime/anchors/skill-name/positive/
  negative: _runtime/anchors/skill-name/negative/

cold_start_mode: false          # set true if proceeding without logs
```

The manifest should be reviewed whenever the Skill's intent changes materially. Do not silently drift the manifest to match what the Skill currently does — that defeats the purpose.

## Per-type cycle adjustments

### Functional cycle

- Stage 2 (Diagnose): weight correctness-based failure modes higher
- Stage 3 (Generate): prefer strategies that tighten contracts
- Stage 4 (Arena): judge focuses on "did this complete the task?"
- Stage 5 (Oracle): rarely needed unless arena is genuinely indeterminate
- Stage 6 (Holdout): mandatory, full rigor

### Aesthetic cycle

- Stage 1 (Prepare): refuse to proceed if anchors don't exist — run anchor setup first
- Stage 2 (Diagnose): always include style-drift check, even if other modes seem dominant
- Stage 3 (Generate): flag candidates that change visual structure beyond what the failure mode required
- Stage 4 (Arena): judges receive anchor references as part of their judgment context
- Stage 4.5 (Anchor gate): candidates below similarity threshold are disqualified regardless of arena wins
- Stage 5 (Oracle): triggered often for style calls
- Stage 6 (Holdout): check anchors hold up on held-out prompts too

### Creative cycle

- Stage 2 (Diagnose): always run AI-voice detection
- Stage 3 (Generate): prefer few-shot examples over rule-based constraints; rules tend to deaden voice
- Stage 4 (Arena): judge protocol asks specifically about voice and tone alongside task completion
- Stage 5 (Oracle): triggered often because creative judgment resists delegation
- Stage 6 (Holdout): check for overfitting by running on stylistically different holdout prompts, not just content-different ones

### Mixed cycle

- Run the dominant dimension's cycle as primary
- Apply the secondary dimension as a gate check: winner must not lose on the secondary dimension
- If winner does lose on secondary, escalate to user oracle

## Common classification mistakes

- **Treating "code generation" as purely functional.** Code has style (readability, idioms, naming). If the user cares about how the code looks and reads, it's mixed.
- **Treating "writing a report" as functional.** Reports have voice and structure. If the user has opinions about tone, it's at least mixed.
- **Treating "make a chart" as purely aesthetic.** Charts have correctness (right data, right scale). Usually mixed with functional dominant.
- **Skipping classification for small Skills.** Even a three-line Skill benefits from type clarity. The type affects which failures to look for.

When in doubt, ask the user directly: "If I gave you an output that did the job but felt wrong, would you accept it?" Their answer tells you the type.
