# Failure Mode Catalog

This is the catalog Stage 2 (Diagnose) scans for. Each entry includes: what the failure looks like, how to detect it, which Skill types it commonly affects, and a severity heuristic.

Detection is not perfect. Many failures require reading actual output samples from the usage log and checking for specific signals. When detection is ambiguous, surface the candidates to the user rather than auto-classifying.

---

## Category A: Intent-related failures

### A1. Intent drift

The Skill's output does not match what the user asked for. Most common failure across all types. Often the Skill latches onto a surface keyword and misses the underlying request.

**Detection signals:**
- User follow-up messages in the log containing "不是这个意思", "I meant...", "no, I wanted...", "try again but..."
- User heavy-edits the output before using it (if edit diffs are in the log)
- User abandons the session shortly after the Skill output

**Affects:** All types, dominant in functional
**Severity:** High by default

### A2. Trigger misfire

The Skill activates when it shouldn't, or fails to activate when it should. Caused by over-broad or under-specific triggering descriptions.

**Detection signals:**
- Skill output appears in contexts unrelated to its declared intent
- User explicitly disables the Skill or stops using it for its intended purpose
- Sibling Skills with overlapping domains

**Affects:** All types
**Severity:** High — the Skill's description is the contract

### A3. Context dropout

The user provided specific context (names, dates, constraints, prior decisions) and the Skill ignored it, producing generic output.

**Detection signals:**
- Specific user-provided entities missing from output
- Output matches a template regardless of input specifics
- User follow-up "you forgot that I said X"

**Affects:** All types
**Severity:** Medium to high

---

## Category B: Format and structure failures

### B1. Format violation

The Skill's declared output format is violated: missing sections, wrong heading levels, incorrect field names, invalid structured data.

**Detection signals:**
- Output fails schema validation (if schema is declared)
- Missing manifest-declared section markers
- User comments about structure ("why is there no summary section?")

**Affects:** Functional primarily, mixed sometimes
**Severity:** Medium — easy to fix, but erodes trust

### B2. Verbosity bloat

Output is significantly longer than necessary. Usually caused by the Skill padding with AI-generated scaffolding ("Let me walk you through...", "In summary...", "Here's what I've done..."), preambles, or over-eager structure.

**Detection signals:**
- Output token count > 1.5× baseline for comparable prompts
- Output contains classic bloat phrases (see AI-voice detector list in Category D)
- User truncates or edits out the filler before using

**Affects:** All types, especially functional
**Severity:** Medium — not broken, but degrades usability over time

### B3. Under-specification leak

The Skill's instructions were not specific enough, so outputs vary widely for similar inputs. Different sessions produce structurally different outputs for essentially the same task.

**Detection signals:**
- High variance across comparable log entries
- Inconsistent section ordering
- Format that changes run-to-run

**Affects:** Functional, mixed
**Severity:** Medium

### B4. Over-specification rigidity

The opposite of B3. The Skill's instructions are so detailed that it produces robotic, template-bound output that ignores context that would warrant variation.

**Detection signals:**
- Every output reads like the same template with names swapped
- User feedback about output being "stiff", "formulaic", "mechanical"
- Skill fails gracefully-degraded cases because the template has no room

**Affects:** Functional and creative
**Severity:** Medium — tempting to fix B3 by creating B4

---

## Category C: Quality failures

### C1. Instruction-example contradiction

The Skill's instructions say one thing, but the examples it shows demonstrate something different. The model tends to follow examples more than rules, so this usually manifests as rules being silently ignored.

**Detection signals:**
- Outputs match examples in SKILL.md rather than rules in SKILL.md
- User complaints about ignored constraints that are clearly stated in rules

**Affects:** All types
**Severity:** High — hard to detect without reading SKILL.md carefully

### C2. Stale assumption

SKILL.md assumes facts, tools, APIs, or conventions that have changed since the Skill was written.

**Detection signals:**
- References to outdated model names, deprecated APIs, moved file paths
- Instructions to use tools that no longer exist
- Outputs citing information from a specific time period as current

**Affects:** Functional primarily
**Severity:** High if current, low otherwise

### C3. Dependency break

SKILL.md references files, scripts, or resources that cannot be accessed.

**Detection signals:**
- References to files that return 404 or don't exist
- Scripts that fail on invocation
- External URLs that redirect or have changed content

**Affects:** All types
**Severity:** High — Skill is effectively broken

---

## Category D: Voice failures (creative/mixed primarily)

### D1. AI-voice leakage

Output contains formulaic phrases that signal AI authorship even when the Skill is meant to sound natural or in-voice.

**High-signal phrases to scan for (non-exhaustive, English):**
- "Let me walk you through..."
- "It's important to note that..."
- "In conclusion..."
- "I hope this helps!"
- "Feel free to..."
- "As we can see..."
- "Navigate the complexities of..."
- "Unlock the potential of..."
- "Delve into..."
- "In the ever-evolving landscape of..."
- "It's worth mentioning..."
- "When it comes to..."

**High-signal phrases (Chinese):**
- "让我们一起来看看"
- "值得注意的是"
- "相得益彰"
- "深入浅出"
- "不言而喻"
- "与此同时"
- "综上所述"
- "核心精髓"
- "最后但同样重要的"
- "在这个充满X的Y中"

Languages beyond these should build their own phrase list from the user's historical outputs.

**Affects:** Creative, mixed
**Severity:** High for Skills that claim natural or colloquial voice

### D2. Tonal drift

The Skill has a declared voice (e.g., "casual conversational," "formal business," "authoritative but warm") and outputs drift away from it, usually toward a generic neutral register.

**Detection signals:**
- Voice consistency score (measured by comparing recent outputs to the voice anchor) dropping over time
- User feedback about tone mismatches
- New outputs sounding substantively different from older, approved ones

**Affects:** Creative, mixed
**Severity:** Medium to high

### D3. Over-structured creative output

Creative output gets forced into bullet points, numbered lists, and headings when flowing prose would be more appropriate.

**Detection signals:**
- Narrative content delivered in bullet form
- User edits that remove structure to restore flow
- Scene descriptions or story beats listed rather than written

**Affects:** Creative primarily
**Severity:** Medium

---

## Category E: Aesthetic failures (aesthetic/mixed primarily)

### E1. Style drift

Output violates declared style constraints — wrong color, wrong typography, wrong layout grammar, wrong compositional rhythm.

**Detection signals:**
- Anchor similarity score below threshold
- User edits the visual attributes of the output
- Output resembles generic defaults rather than the specific aesthetic

**Affects:** Aesthetic, mixed
**Severity:** High — style fidelity is usually the primary requirement

### E2. Structural inconsistency

Within a single output, style is applied inconsistently (e.g., one heading uses the declared font, the next doesn't; one card has correct padding, the next doesn't).

**Detection signals:**
- Visual inspection of output samples
- Inconsistent CSS values across similar elements
- User feedback about "uneven" or "sloppy" execution

**Affects:** Aesthetic
**Severity:** Medium to high

### E3. Reference contamination

Output inadvertently copies specific content from anchor examples rather than their style pattern (e.g., same color hex codes when anchors were supposed to inspire palette, not dictate exact values).

**Detection signals:**
- Anchor content literal copies appearing in new output
- Output scope narrower than Skill's declared range

**Affects:** Aesthetic, creative
**Severity:** Medium

---

## Category F: Operational failures

### F1. User confirmation skipped

Skill makes irreversible or high-stakes decisions without user confirmation when the manifest declares confirmation is required.

**Detection signals:**
- Log entries show Skill proceeding through confirmation-gated steps without pause
- User reports unintended actions

**Affects:** Functional
**Severity:** High for Skills affecting files, data, or external systems

### F2. Runaway length

Output grows unboundedly when input is ambiguous. The Skill doesn't know when to stop.

**Detection signals:**
- Outputs that exceed any reasonable length
- User truncates manually before using
- Token-cost anomalies in the usage log

**Affects:** All types
**Severity:** Medium

### F3. Silent fallback to generic

When the Skill hits an edge case, it silently produces generic content rather than acknowledging the edge case to the user.

**Detection signals:**
- Output is suspiciously generic given specific input
- Edge case signals in input (unusual characters, extreme sizes, unfamiliar domains) produce undifferentiated output

**Affects:** All types
**Severity:** Medium

---

## Detection workflow

For each Skill, Stage 2 should:

1. Load the usage log (train split only — never peek at holdout during diagnose)
2. For each failure mode relevant to the Skill's type, run the detection signals against the log
3. Compute: frequency (how often), severity (from table), and impact (frequency × severity)
4. Present top 3-5 failure modes ranked by impact
5. Let the user pick the target failure mode for this cultivation round

Do not try to fix more than one failure mode per round. Attribution becomes impossible if multiple dimensions change at once. This is a non-negotiable constraint — it's the only way to reliably learn what works.

## When detection is uncertain

Some failure modes (B3 under-specification, C1 instruction-example contradiction) require reading the SKILL.md and comparing to outputs, not just reading the log. If detection confidence is low, flag the failure mode as "suspected" and present evidence to the user. Never mark a failure as confirmed without specific log evidence.

When in doubt, show the evidence and let the user decide. The user knows their Skill better than the catalog does.
