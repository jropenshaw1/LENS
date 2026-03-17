# LENS Functional Spec v1.0
**Status:** Ratified
**Date:** March 16, 2026
**Follows:** Data Dictionary v1.0
**Precedes:** ADR Log v1.0

---

## 1. Purpose

This document defines how LENS operates mechanically — the exact sequence of AI actions from
trigger detection to OpenBrain write, the pattern detection algorithm, the TTL implementation
for raw prompt retention, and the block-based prompt structure recommendation. Where the Charter
and Use Case Spec define *what* LENS does, this document defines *how* it does it.

---

## 2. Capture Flow

### 2.1 Overview

The capture flow has five stages: Detect, Evaluate, Propose, Ratify, Write. The flow is
identical for CLARIFICATION_TRIGGER and ASSUMPTION_TRIGGER. SUCCESS_PATTERN (LENS+) enters
at Stage 3.

```
CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER:
  Stage 1: Detect → Stage 2: Evaluate → Stage 3: Propose → Stage 4: Ratify → Stage 5: Write

SUCCESS_PATTERN (LENS+):
  Stage 3: Propose → Stage 4: Ratify → Stage 5: Write
```

---

### 2.2 Stage 1 — Detect

**Trigger:** CLARIFICATION_TRIGGER
- The AI determines it cannot proceed without additional information
- The AI asks a clarifying question
- The question is blocking — the response would materially change depending on the answer
- The session is a human-authored conversational session (authorship rule from Use Case Spec)
- The prompt is not a coding or technical execution prompt (domain exclusion rule)

**Trigger:** ASSUMPTION_TRIGGER
- The AI detects multiple plausible interpretations of the prompt
- The AI chooses one interpretation silently without asking
- The choice would materially affect the output

**Not a trigger (detection gates):**
- Conversational follow-up questions ("tell me more")
- Confirmations of completed work ("does that look right?")
- Gee's trained improvement offers
- Short confirmatory answers from the user ("yes", "agreed", "proceed") — these are resolved
  intent, not underspecified prompts
- Session cap reached (5 captures) — LENS goes quiet, no detection until next session

**Session cap check:** Before firing Stage 1, check the session capture count. If count >= 5,
suppress the trigger silently. Do not notify the user unless asked.

---

### 2.3 Stage 2 — Evaluate

After the clarification exchange is complete and the response has been delivered, the AI
constructs the capture record:

1. **trigger_type** — set based on what fired (CLARIFICATION_TRIGGER or ASSUMPTION_TRIGGER)
2. **timestamp** — AI system time at this moment
3. **ai_model** — self-report: `provider:model_name:version`
4. **prompt_type** — classify the original prompt's dominant intent
5. **original_prompt** — the prompt as submitted, before clarification
6. **ambiguity_type** — apply the canonical taxonomy from Data Dictionary Section 5
7. **missing_dimension** — one sentence: what was absent from the original prompt
8. **assumption_resolution_mode** — populate only for ASSUMPTION_TRIGGER
9. **optimized_prompt** — author the prompt that would have produced the best result without
   clarification, based on what the exchange revealed
10. **delta_reason** — one sentence: why the optimized prompt produces a materially better result
11. **underspecification_confidence** — score 1–5 based on certainty that the prompt was
    genuinely underspecified vs. intentionally brief

**Evaluation rule:** If underspecification_confidence is 1 or 2, do not propose the capture.
Low-confidence triggers produce noise. The session cap is preserved for higher-signal captures.

---

### 2.4 Stage 3 — Propose

The AI presents the proposed capture to the user for ratification. Format:

> **LENS capture proposed**
>
> **Trigger:** [CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER]
> **Original prompt:** "[exact original prompt]"
> **What was missing:** [missing_dimension — one sentence]
> **Optimized prompt:** "[AI-authored optimized prompt]"
> **Why it's better:** [delta_reason — one sentence]
> **Confidence:** [underspecification_confidence]/5
>
> Confirm, edit, or skip?

For SUCCESS_PATTERN (LENS+):

> **LENS+ capture proposed**
>
> **Exchange:** [brief description of the preceding exchange]
> **What worked:** [effective_pattern — one sentence]
>
> Confirm, edit, or skip?

**Timing:** The proposal appears after the response is delivered — never interrupting the
response itself. LENS is additive. It does not delay execution.

---

### 2.5 Stage 4 — Ratify

Three user responses are valid:

| Response | Action |
|----------|--------|
| "Confirm" / "yes" / affirmative | Proceed to Stage 5 with record as proposed |
| Edit (user provides corrections) | Update the record with user's edits, then proceed to Stage 5 |
| "Skip" / "no" / negative | No capture. No record. LENS does not retry. Session cap is NOT incremented. |

**Context contamination rule:** LENS captures are write-only from the executing AI during
live sessions. The AI proposing the capture writes it. No other AI client writes to another
client's LENS captures.

---

### 2.6 Stage 5 — Write

On user confirmation, the AI writes to OpenBrain using `capture_thought`:

**Thought content format:**
```
[domain:LENS] [artifact:LENS-capture]
[trigger:TRIGGER_TYPE] [model:provider:model_name] [prompt_type:value] [ambiguity_type:value]

LENS CAPTURE — [timestamp]
trigger_type: [value]
ai_model: [value]
prompt_type: [value]
original_prompt: [value]
ambiguity_type: [value]
missing_dimension: [value]
assumption_resolution_mode: [value or null]
optimized_prompt: [value]
delta_reason: [value]
underspecification_confidence: [value]
outcome_status: [null — populated later if user provides feedback]
improvement_dimension: [null — populated later]
effective_pattern: [null — or value for SUCCESS_PATTERN]
```

After write, increment the session capture count by 1.

Confirm to user: "Captured." — nothing more. Keep it lightweight.

---

## 3. Pattern Detection Algorithm

### 3.1 Overview

Pattern detection synthesizes LENS-correction thoughts into LENS-pattern thoughts. It runs
on-demand when the user requests a prompting skill review — not on a schedule.

### 3.2 Detection trigger

User requests one of:
- "Show me my LENS patterns"
- "What am I underspecifying most?"
- "LENS report"
- "How is my prompting?"
- Any similar prompting skill review request

### 3.3 Algorithm

**Step 1 — Retrieve corpus**
Search OpenBrain for all thoughts tagged `[artifact:LENS-capture]`. Retrieve full content.
Minimum corpus size for meaningful analysis: 10 captures. If fewer than 10, surface what
exists but note the dataset is too small for reliable pattern conclusions.

**Step 2 — Frequency analysis**
Count occurrences of each `ambiguity_type` value across all captures. Rank by frequency.
Secondary grouping by `prompt_type` — which prompt types produce the most underspecification?

**Step 3 — Trend analysis**
If captures span more than 14 days, compare frequency distribution across time periods.
Is a pattern improving, stable, or worsening? A declining frequency in a previously dominant
ambiguity_type is a measurable improvement signal.

**Step 4 — Model bias check**
Group captures by `ai_model`. If a pattern appears predominantly with one model and not others,
flag it as potentially model-behavioral rather than user-behavioral. Do not attribute
model-specific behavior to the user's prompting skill.

**Step 5 — Synthesize**
Identify the top 1–3 patterns by frequency and impact. For each:
- Name the pattern (ambiguity_type + prompt_type combination)
- State the frequency and trend
- Name the cost (what kind of output degradation it causes based on delta_reason patterns)
- Suggest one concrete behavior change

**Step 6 — Report**
Deliver conversationally per Use Case Spec Section 10:
1. Present frequency table
2. State impact for top pattern
3. Open discussion for the user to explore, question, and decide

**Step 7 — Capture LENS-pattern thought**
After discussion, if a meaningful pattern was identified, capture it:
```
[domain:LENS] [artifact:LENS-pattern]
LENS PATTERN — [date]: [pattern name]. Frequency: [N] captures, [% of corpus].
Trend: [improving/stable/worsening]. Cost: [what this pattern produces in output quality].
Behavior change: [one concrete suggestion]. Model bias check: [clean/flagged].
```

---

## 4. Raw Prompt TTL Implementation

### 4.1 Policy
Raw prompt fields (`original_prompt`, `optimized_prompt`) are retained for 90 days per
Data Dictionary Section 6. This is a policy decision; implementation is the AI's responsibility
at report time.

### 4.2 Tiered Retention Model

LENS uses a two-tier retention approach that preserves analytical value while reducing
long-term privacy exposure:

**Tier A — Granular (0–90 days)**
Full capture records retained including `original_prompt` and `optimized_prompt`. Supports
detailed per-capture analysis and pattern detection against raw content.

**Tier B — Aggregate (90–180 days)**
At the 90-day mark, raw prompt fields are redacted and replaced with a `LENS-aggregate`
thought summarizing the window. The aggregate captures:
- Frequency distribution of `ambiguity_type` for the 90-day window
- Top `prompt_type` + `ambiguity_type` combinations
- Average `underspecification_confidence`
- Model breakdown (captures per AI model)
- Trend direction vs. prior window

**After 180 days:** Aggregate thoughts are retained indefinitely. They contain no raw content
and carry full long-term analytical value including cross-AI-release comparisons.

**Value of tiered retention:** When a new AI model releases, aggregate thoughts allow
comparison of prompting patterns across model generations — revealing whether behavior changes
are improvements, regressions, or model-specific artifacts rather than user-behavioral shifts.

### 4.3 Implementation approach

OpenBrain does not currently support automatic TTL on individual fields. The implementation
is manual, triggered at pattern detection report time:

**At each LENS report request:**
1. Retrieve all LENS-capture thoughts
2. Identify captures older than 90 days with unredacted raw prompt fields
3. Synthesize a `LENS-aggregate` thought for the expiring window (format per Section 4.2)
4. Propose redaction:
   > "[N] LENS captures from [date range] have raw prompt fields due for redaction.
   > Aggregate summary has been prepared. Redact now? (Replaces original_prompt and
   > optimized_prompt with [REDACTED-90d] while preserving all derived metadata.)"
5. On user confirmation, capture the aggregate thought, then update expired captures

**Note:** OpenBrain does not currently expose an update tool. Until an update tool is available,
the AI should flag expired captures for manual review rather than attempting to redact.
This is a v1.0 limitation — see ADR Log for the full decision.

---

## 5. Block-Based Prompt Structure Recommendation

### 5.1 Purpose
The block-based prompt structure is a recommended best practice — not mandated. It is documented
here as the highest-fidelity input format for LENS, meaning it minimizes CLARIFICATION_TRIGGER
frequency by front-loading the information LENS captures would otherwise reveal as missing.

### 5.2 Structure

A block-based prompt has five optional components. Not all are needed for every prompt — use
the components that are relevant to the task:

```
[CONTEXT]
What the AI needs to know to understand the situation.
Who, what, where, relevant background.

[WHY]
The intent behind the request — what you are trying to achieve or accomplish.
Why this matters, what decision or outcome it supports.

[TASK]
What you are asking the AI to do. Specific verb: analyze, generate, plan, critique, etc.

[CONSTRAINTS]
Boundaries, limitations, format requirements, length, audience, tone.

[SUCCESS CRITERIA]
What a good result looks like. How you will evaluate the output.
```

**Note on [WHY]:** Intent is one of the most powerful — and most commonly omitted — prompt
dimensions. Stating why you are asking a question gives the AI a complete frame to work from
and eliminates the most common source of output drift: the AI solving the stated problem
rather than the actual underlying need. Including [WHY] is a prompting behavior worth
developing deliberately.

### 5.3 Example

Unstructured (likely to trigger CLARIFICATION_TRIGGER):
> "Help me prepare for my interview next week."

Block-structured (unlikely to trigger CLARIFICATION_TRIGGER):
> [CONTEXT] I have a final-round interview at CSC Global next Thursday for a Senior Manager
> of Technology role. I'm coming from 10 years as Director of Cloud FinOps at Choice Hotels.
>
> [WHY] I want to walk in with prepared, specific responses that demonstrate strategic
> leadership and FinOps credibility — not generic interview answers.
>
> [TASK] Help me prepare behavioral and strategic responses for likely interview questions.
>
> [CONSTRAINTS] Focus on leadership, transformation, and FinOps/cloud credibility. Avoid
> generic frameworks — I want specific examples I can draw from.
>
> [SUCCESS CRITERIA] I should be able to walk into the interview with 5–7 prepared response
> outlines that I can adapt in the room.

### 5.4 When to use it
Block structure is most valuable for:
- Complex analysis or planning requests
- Any prompt where context, constraints, or success criteria are non-obvious
- Prompts where you have previously seen a CLARIFICATION_TRIGGER fire

It is not necessary for:
- Simple lookup or research requests
- Short conversational exchanges
- Confirmatory responses

---

## 6. Decisions Locked in This Document

| Decision | Value |
|----------|-------|
| Capture flow | 5-stage: Detect → Evaluate → Propose → Ratify → Write |
| Low-confidence suppression | Confidence 1–2 suppresses the capture proposal |
| Session cap behavior | Silently suppressed; declined captures do not increment count |
| Proposal timing | After response delivery — never interrupts execution |
| Pattern detection trigger | On-demand only — not scheduled |
| Minimum corpus for patterns | 10 captures |
| Model bias check | Required in every pattern report |
| Tiered retention | Granular 0–90 days; aggregate 90–180 days; indefinite thereafter |
| TTL implementation | Manual at report time; field-level redaction pending update tool |
| Block-based structure | Five components including [WHY] — recommended best practice, not mandated |
| [WHY] component | Explicitly added as a high-value prompting behavior to develop |

---

*LENS Functional Spec v1.0 — Ratified March 16, 2026*
*Next document: ADR Log v1.0*
