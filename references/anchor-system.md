# Anchor System

Anchors are the backbone of evaluation for aesthetic and creative Skills. They replace abstract rubrics with concrete examples of what "correct" looks like. This document explains how to set up anchors, how the system uses them, and how to maintain them as they age.

---

## Why anchors exist

LLM judges cannot reliably evaluate aesthetic quality by prose description. Give a judge the instruction "evaluate whether this layout has good visual rhythm" and you'll get plausible-sounding answers with low inter-rater reliability and significant bias toward certain surface features (more structure, more color, more everything).

Anchors solve this by turning evaluation from prose-judgment into **similarity-to-reference**. Instead of "is this well-designed?" the question becomes "does this look like the references the user marked as correct, and unlike the ones marked as wrong?" That question has much higher signal.

The tradeoff: setting up anchors takes user effort upfront. There's no way around this. The user has to identify what they want, concretely, before the system can evaluate against it.

---

## Anchor structure

For each aesthetic or creative Skill, create:

```
_runtime/anchors/<skill-name>/
├── positive/                # 5-10 examples of outputs the user loves
│   ├── 01.html              # or .txt, .md, .png — whatever the Skill produces
│   ├── 01.note.md           # why this one is good (user's words)
│   ├── 02.html
│   ├── 02.note.md
│   └── ...
├── negative/                # 3-5 examples the user considers wrong
│   ├── 01.html
│   ├── 01.note.md           # what's wrong with it
│   └── ...
└── anchor-config.yaml       # similarity method, thresholds
```

Every anchor has a note file explaining what makes it positive or negative. The notes are not just for humans — they're injected into judge prompts when evaluating candidates, providing grounded context beyond raw similarity.

### Anchor notes format

```markdown
# 01 note

**Status:** positive

**What's right about this:**
- <specific, concrete attribute 1>
- <specific, concrete attribute 2>

**Key signals:**
- <what a judge should check first when assessing similarity>

**What this is NOT an example of:**
- <disambiguation if needed>
```

Keep notes concrete. "It feels right" is less useful than "the spacing between paragraphs is tight and the heading typography is heavy enough to anchor the page."

---

## Anchor setup flow

Invoked by `~anchor [skill-name]` or natural language "set up anchors for X".

### Step 1: Explain the commitment

Before starting, explain to the user:
- Setting up anchors takes 15-30 minutes
- The more anchors, and the better the notes, the better the system can evaluate
- Anchors age. Plan to refresh them every few months or when the style evolves.
- Without anchors, aesthetic and creative Skills cannot be optimized responsibly.

### Step 2: Collect positives

Ask the user to provide 5-10 outputs from this Skill (or outputs they wish this Skill produced) that they consider correct examples. Sources can be:
- Past Skill outputs the user approved
- Outputs the user manually edited and is happy with
- External references the user aspires to
- Manually crafted target examples

For each, ask: "What makes this one right? Give me one or two specific attributes." Save to note file.

If the user can't produce 5 positives, stop. Either the Skill doesn't have a clear enough aesthetic yet, or the user needs to create reference material first. Don't proceed with too few anchors — thin anchor sets produce worse evaluations than no anchors at all.

### Step 3: Collect negatives

Ask for 3-5 outputs that the user considers wrong. These should be outputs that are close to correct but miss — not wildly irrelevant examples. Hard negatives teach the system more than easy negatives.

For each, ask: "What's wrong with it? Be specific." Save to note file.

### Step 4: Configure

Generate `anchor-config.yaml`:

```yaml
similarity_method:
  visual: structural-similarity   # for HTML/image outputs
  textual: embedding-cosine       # for text outputs
  
thresholds:
  min_positive_similarity: 0.65   # candidate must be this close to positives
  max_negative_similarity: 0.40   # candidate must be this far from negatives
  
anchor_weights:
  # positive anchors can be weighted if some are more representative
  01: 1.0
  02: 1.0
  03: 1.2                         # user marked this as especially canonical

refresh_cadence_days: 90          # system will prompt to review after this

last_refreshed: <date>
```

Initial thresholds are defaults. They should be tuned after the first few cultivation rounds based on how often candidates fail the gate unnecessarily vs. how often drift slips through.

### Step 5: Sanity check

Run a calibration check: for each positive anchor, compute similarity to the full anchor set using the chosen method. Every positive anchor should score above the positive threshold when evaluated against the other positives. If not, adjust thresholds or re-examine anchor selection.

---

## How similarity is computed

Method depends on output type.

### For HTML/CSS outputs (common for aesthetic Skills)

Use structural similarity, not pixel similarity. Compare:
- DOM structure (tag hierarchy, class patterns)
- CSS property distribution (colors used, fonts used, spacing values)
- Layout grammar (grid/flex patterns, breakpoints)

A rough approach:
1. Extract a feature vector from each HTML: {color palette, font set, spacing values, element counts by type, hierarchy depth}
2. Compute cosine or weighted euclidean distance in feature space

This is implementation-specific; the arena can delegate similarity computation to a dedicated sub-agent that receives both outputs and returns a similarity score plus reasoning.

### For image outputs (if the Skill generates visuals)

Use a lightweight perceptual similarity model. CLIP or similar if available. If not available, delegate to a vision-capable judge with the instruction: "rate these two images on a 0-1 scale for style similarity, then justify."

### For text outputs (creative Skills)

Multi-signal approach:
- **Lexical similarity**: TF-IDF or embedding cosine
- **Stylistic features**: sentence length distribution, vocabulary richness, punctuation patterns
- **AI-voice score**: frequency of banned phrases from `failure-modes.md` D1 list
- **Voice similarity**: embedding comparison specifically on voice-loaded attributes

Combine into a composite score with tunable weights.

### When similarity computation is unreliable

Some aesthetic judgments exceed what automated similarity can capture (especially truly novel compositions). When the similarity score is ambiguous (e.g., in a band between two thresholds), the system should escalate to user oracle rather than decide autonomously.

---

## The anchor gate in the cultivation cycle

Applied during Stage 4 (Arena) after initial pairwise matches, before Stage 6 (Holdout):

```
For each candidate that won or tied in arena:
  For each arena prompt (or a sample):
    Generate candidate output on that prompt
    Compute similarity to positive anchor set → pos_score
    Compute similarity to negative anchor set → neg_score
    
    If pos_score < min_positive_similarity:
      disqualify candidate; reason = "insufficient similarity to positive anchors"
      
    If neg_score > max_negative_similarity:
      disqualify candidate; reason = "too similar to negative anchors"
      
    If neither: candidate passes the gate

If top candidate is disqualified, next candidate (in arena ranking) is gated.
If all candidates are disqualified: round yields no winner. 
  Report to user: "The arena found winners, but none match your aesthetic. 
   Possible causes: (a) anchors are outdated, (b) candidates all drift in the same direction, 
   (c) thresholds are too tight."
```

---

## Anchor maintenance

Anchors age. A Skill that was optimized against anchors from six months ago will not match the user's current taste if the taste has evolved. The system should actively surface anchor staleness:

### Triggered prompts to refresh:

- Time-based: after `refresh_cadence_days` (default 90), prompt the user to review.
- Disagreement-based: if the user rejects candidates that passed the anchor gate more than twice in a row, the gate is misaligned with current taste. Prompt a refresh.
- Expansion-based: if the user's taste has widened (e.g., they're now happy with outputs the anchor system would reject), add the newly-approved outputs to the positive set.

### Refresh protocol:

1. Show the user their current positive anchor set
2. Ask: "Do any of these no longer match your current taste?" — remove any flagged
3. Ask: "Do you have new examples you'd add?" — add new positives
4. Same for negatives
5. Recompute threshold calibration
6. Save new `last_refreshed` timestamp

Don't force a wholesale replacement. Anchor sets are living collections; gradual drift is normal.

---

## Common anchor setup mistakes

- **Too few anchors.** Fewer than 5 positives or 3 negatives produces unreliable gating. Stop and collect more.
- **Homogeneous positives.** If all 10 positives are the same kind of output (all landing pages, all product pages), the anchor set will disqualify valid variations. Include stylistic diversity within the correct-aesthetic bounds.
- **Weak negatives.** Negatives that are obviously wrong don't teach the gate anything. Good negatives are close-but-miss, not wildly off-target.
- **Notes that only say "good" or "bad".** Notes need specificity. If the user can't articulate what's right or wrong, the system can't help them.
- **Never updating.** Anchors from months ago can silently blocks legitimate evolution. Schedule reviews.

---

## A note on creative voice anchors

For creative Skills, positive anchors are typically paragraphs or passages the user wrote (or approved) in the voice they want. Negatives are passages that sound AI-written or miss the voice.

Two additional considerations:

- **Voice anchors should span domains.** A voice that's only demonstrated on one topic will be hard to transfer. Include the voice applied to multiple subjects.
- **Don't anchor on accidental features.** If every positive anchor happens to start with a single-sentence paragraph, the system may conclude "correct voice requires single-sentence opening." Vary structural features while keeping voice consistent.

The goal is anchors that capture *what voice is*, not *what these specific passages happen to have in common*.
