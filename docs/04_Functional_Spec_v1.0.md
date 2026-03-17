# LENS Functional Spec v1.0
**Status:** Ratified — updated to reflect ADR-003 (confidence-gated auto-capture) and ADR-006 (tiered retention deferred to v1.1)
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

The capture flow has five stages: Detect, Evaluate, Route, Write/Ratify, Confirm.
Routing at Stage 3 depends on `underspecification_confidence` — high-confidence captures
(4–5) bypass ratification and write automatically. Medium-confidence captures (3) require
manual ratification. SUCCESS_PATTERN (LENS+) always requires ratification regardless of
confidence.

```
CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER (confidence 4–5):
  Stage 1: Detect → Stage 2: Evaluate → Stage 3: Route → Stage 5: Auto-Write → Stage 5b: Notify

CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER (confidence 3):
  Stage 1: Detect → Stage 2: Evaluate → Stage 3: Route → Stage 4: Propose → Stage 4b: Ratify → Stage 5: Write

CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER (confidence 1–2):
  Stage 1: Detect → Stage 2: Evaluate → Stage 3: Route → Suppressed (no capture)

SUCCESS_PATTERN (LENS+):
  Stage 4: Propose → Stage 4b: Ratify → Stage 5: Write
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

---

### 2.4 Stage 3 — Route

Route the capture based on `underspecification_confidence`:

| Confidence | Action |
|------------|--------|
| 1–2 | Suppress. No capture proposed. Session cap not incremented. |
| 3 | Proceed to Stage 4 — manual ratification required. |
| 4–5 | Skip Stage 4 — proceed directly to Stage 5 auto-write, then Stage 5b notify. |

**Rationale for confidence-gated routing (ADR-003):** High-confidence captures (4–5) are
clear-cut trigger events where the AI's interpretation is unlikely to require correction.
Auto-capturing them reduces missed signals without materially degrading dataset quality.
Confidence 3 captures sit at the boundary where user judgment adds the most value.

---

### 2.5 Stage 4 — Propose (confidence 3 and SUCCESS_PATTERN only)

**For confidence 3 captures:**

> **LENS capture proposed**
>
> **Trigger:** [CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER]
> **Original prompt:** "[exact original prompt]"
> **What was missing:** [missing_dimension — one sentence]
> **Optimized prompt:** "[AI-authored optimized prompt]"
> **Why it's better:** [delta_reason — one sentence]
> **Confidence:** 3/5
>
> Confirm, edit, or skip?

**For SUCCESS_PATTERN (LENS+):**

> **LENS+ capture proposed**
>
> **Exchange:** [brief description of the preceding exchange]
> **What worked:** [effective_pattern — one sentence]
>
> Confirm, edit, or skip?

**Stage 4b — Ratify:**

| Response | Action |
|----------|--------|
| "Confirm" / "yes" / affirmative | Proceed to Stage 5 with record as proposed |
| Edit (user provides corrections) | Update record with user's edits, proceed to Stage 5 |
| "Skip" / "no" / negative | No capture. No record. LENS does not retry. Session cap NOT incremented. |

**Context contamination rule:** LENS captures are write-only from the executing AI during
live sessions. The AI proposing the capture writes it. No other AI client writes to another
client's LENS captures.

**Timing:** Proposals appear after the response is delivered — never interrupting the response
itself. LENS is additive. It does not delay execution.

---

### 2.6 Stage 5 — Write

On auto-route (confidence 4–5) or user confirmation (confidence 3, LENS+), the AI writes
to OpenBrain using `capture_thought`:

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
capture_mode: [auto | ratified]
outcome_status: [null — populated later if user provides feedback]
improvement_dimension: [null — populated later]
effective_pattern: [null — or value for SUCCESS_PATTERN]
```

Note: `capture_mode` field added to distinguish auto-captured entries from ratified ones.
This enables filtering by capture mode in pattern analysis.

After write, increment the session capture count by 1.

**Stage 5b — Notify (auto-captures only):**

> "LENS auto-captured: [missing_dimension — one sentence]. Review with 'show LENS queue'."

Keep it brief. Do not block on a response.

---

## 3. Pattern Detection Algorithm

### 3.1 Overview

Pattern detection synthesizes LENS-capture thoughts into LENS-pattern thoughts. It runs
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
Data Dictionary Section 6. Tiered retention with aggregate synthesis is deferred to v1.1
per ADR-006. At v1.0, only manual redaction is supported.

### 4.2 v1.0 Implementation — Manual Redaction at Report Time

OpenBrain (OB1 v2) does not currently expose a field-level update or delete tool. The
90-day TTL is aspirational, not architectural, at v1.0 (see ADR-005 for the honest assessment
of this gap).

**At each LENS report request:**
1. Retrieve all LENS-capture thoughts
2. Identify captures older than 90 days with unredacted raw prompt fields
3. If any exist, surface them:
   > "[N] LENS captures from [date range] have raw prompt fields past the 90-day retention
   > policy. Manual redaction is required — OpenBrain does not yet support automated TTL.
   > Review and redact manually in Supabase Table Editor, or acknowledge and continue."
4. Do not attempt to write redacted values — OpenBrain update tool is not available.

### 4.3 Deferred to v1.1
Tiered retention (granular 0–90 days, aggregate 90–180 days, indefinite thereafter) and
LENS-aggregate thought synthesis are deferred to v1.1 after real capture data quality has
been assessed. See ADR-006.

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
| Capture flow | 5-stage with confidence-gated routing at Stage 3 |
| Confidence 4–5 | Auto-capture with lightweight notification — no ratification |
| Confidence 3 | Manual ratification required |
| Confidence 1–2 | Suppressed — no capture proposed |
| capture_mode field | Added to distinguish auto vs. ratified entries |
| Session cap behavior | Silently suppressed; declined captures do not increment count |
| Proposal timing | After response delivery — never interrupts execution |
| Pattern detection trigger | On-demand only — not scheduled |
| Minimum corpus for patterns | 10 captures |
| Model bias check | Required in every pattern report |
| TTL implementation | Manual flag at report time — no automated redaction at v1.0 |
| Tiered retention | Deferred to v1.1 (ADR-006) |
| Block-based structure | Five components including [WHY] — recommended best practice, not mandated |

---

*LENS Functional Spec v1.0 — Updated March 16, 2026*
*Next document: ADR Log v1.0*
