# LENS Functional Spec v1.0
**Status:** Ratified — updated to reflect ADR-003 (confidence-gated auto-capture), ADR-006 (tiered retention deferred to v1.1), persistent TTL notices with SQL, and confidence validation checkpoint
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
per ADR-006. At v1.0, only manual SQL redaction in Supabase is supported.

### 4.2 v1.0 Implementation — Persistent Notice with SQL at Every Report

OpenBrain (OB1 v2) does not currently expose a field-level update or delete tool. The
90-day TTL is aspirational, not architectural, at v1.0 (see ADR-005 for the honest assessment
of this gap).

**Behavior:** The TTL notice is persistent — it fires at every LENS report until redaction
has been completed. Missing a notice does not clear it. The next report will surface it again
with the same SQL, updated to reflect the current count of expired captures.

**At each LENS report request:**
1. Retrieve all LENS-capture thoughts tagged `[artifact:LENS-capture]`
2. Identify captures older than 90 days with unredacted raw prompt fields
3. If any exist, surface the notice AND generate ready-to-run SQL:

> **TTL Notice: [N] LENS captures from [earliest date] to [latest date] have raw prompt
> fields past the 90-day retention policy.**
>
> Run this SQL in the Supabase SQL Editor to redact them:
>
> ```sql
> -- Step 1: Preview captures to be redacted (run this first)
> SELECT id, created_at, LEFT(content, 120) AS content_preview
> FROM thoughts
> WHERE content LIKE '%[artifact:LENS-capture]%'
>   AND created_at < NOW() - INTERVAL '90 days'
>   AND content NOT LIKE '%[REDACTED-90d]%';
>
> -- Step 2: Run redaction after reviewing the preview above
> UPDATE thoughts
> SET content = REGEXP_REPLACE(
>     REGEXP_REPLACE(
>         content,
>         'original_prompt: .*',
>         'original_prompt: [REDACTED-90d]'
>     ),
>     'optimized_prompt: .*',
>     'optimized_prompt: [REDACTED-90d]'
> )
> WHERE content LIKE '%[artifact:LENS-capture]%'
>   AND created_at < NOW() - INTERVAL '90 days'
>   AND content NOT LIKE '%[REDACTED-90d]%';
> ```
>
> This notice will repeat at every LENS report until redaction is complete.

4. Continue with the LENS report after surfacing the notice — do not block on redaction.

**Note:** Always run the SELECT preview before the UPDATE to confirm the correct rows will
be affected. Back up the thoughts table before running UPDATE if capture volume is large.

### 4.3 Confidence Score Validation

Because `underspecification_confidence` is AI self-reported, its calibration must be
validated empirically after v1.0 goes live. The spec cannot guarantee accuracy before
real capture data exists.

**Validation approach — 30-day checkpoint:**
At the first LENS report after 30 days of operation, the AI reviews auto-captured
(confidence 4–5) entries and asks:

> "You have [N] auto-captured LENS entries from the past 30 days. Did any feel incorrectly
> captured — either not a genuine underspecification, or one you would have edited if asked?
> Your feedback will inform whether the confidence threshold needs adjustment."

**Calibration adjustments:**
- Frequent incorrect auto-captures at confidence 4 → raise auto-capture threshold to 5 only
- Very few captures overall → consider lowering manual ratification threshold from 3 to 2

**Documentation:** Calibration findings are captured as a LENS-pattern thought tagged
`[artifact:LENS-pattern] [subtype:calibration]` so the adjustment history is preserved
across sessions.

### 4.4 Deferred to v1.1
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
| TTL notice | Persistent — fires at every LENS report until redaction complete |
| TTL SQL | Generated fresh at every TTL notice — ready to run in Supabase |
| Confidence validation | 30-day checkpoint with calibration adjustment path |
| TTL implementation | Manual SQL in Supabase — no automated redaction at v1.0 |
| Tiered retention | Deferred to v1.1 (ADR-006) |
| Block-based structure | Five components including [WHY] — recommended best practice, not mandated |

---

## 7. Compliance Framework
*(Added March 22, 2026 — follows ADR-008: Protocol Positioning, Charter Section 11)*

### 7.1 Purpose

This section defines what LENS-compliant execution looks like per behavior, what
non-compliant execution looks like, and how compliance is measured at v1.0 without
a runtime evaluation engine. It establishes the measurement foundation that a future
automated scoring system would build on.

The compliance framework makes LENS measurable. "If you can't measure it, you can't
improve it" (Charter Section 9) applies to the protocol itself, not only to the user's
prompting skill.

---

### 7.2 Compliance Rubric — Per Behavior

#### Behavior 1: Prompt Reflection

**Compliant execution:**
- Fires after a response where the AI detected multiple plausible interpretations, chose
  a frame, or required unstated assumptions to proceed
- Names specifically what was assumed and what a sharper prompt would have contained
- Proportionate to the stakes — more detailed on consequential outputs, lighter on routine exchanges
- Does not interrupt the response; appears after delivery

**Non-compliant execution (under-fire):**
- Response required significant assumptions; AI delivered without naming them afterward
- User received output calibrated to one interpretation but was not informed that other interpretations existed

**Non-compliant execution (over-fire):**
- Fires after a response where the prompt was clear and no interpretation choices were required
- Fires performatively — adding metacognitive commentary without substantive content

**Measurable signal:** Frequency of LENS-correction thoughts tagged to Prompt Reflection
over total Prompt Reflection firings. Declining correction rate indicates calibration improvement.

---

#### Behavior 2: Uncertainty Flagging

**Compliant execution:**
- Explicitly marks claims where the AI is reasoning from incomplete information,
  working from assumptions, or operating in territory where its training data is weak
- Language is proportionate: "I'm not certain," "based on assumption," "this may have
  changed," "I'd verify this before relying on it"
- Applied to the specific uncertain claim, not to the entire response

**Non-compliant execution (under-fire):**
- AI presents an uncertain claim with confident language — no qualification, no flag
- User acts on information presented as reliable that was actually assumption-based

**Non-compliant execution (over-fire):**
- AI qualifies every claim regardless of confidence level
- Hedging language applied uniformly, diluting the signal value of genuine uncertainty flags

**Measurable signal:** User-identified instances where a confident-sounding claim turned out
to be incorrect or assumption-based — captured as LENS-correction thoughts. These are the
highest-consequence violation type.

---

#### Behavior 3: Assumption Surfacing

**Compliant execution:**
- States key assumptions before delivering consequential outputs: resume bullets, fit scores,
  architectural recommendations, strategic plans, financial estimates
- Assumptions are specific enough that the user can push back
- Gives the user a concrete place to redirect before the output is complete, not after

**Non-compliant execution (under-fire):**
- Delivers a consequential output without surfacing the assumptions baked into it
- User receives a finished product calibrated to unstated assumptions they may not share

**Non-compliant execution (over-fire):**
- Surfaces assumptions before routine, low-stakes outputs where no user pushback is realistic or valuable
- Adds friction without adding value

**Measurable signal:** LENS-correction thoughts where the user notes they received output
calibrated to wrong assumptions — particularly on high-stakes deliverables.

---

#### Behavior 4: Reframe Offers

**Compliant execution:**
- Identifies when the stated question and the underlying need diverge materially
- Offers the reframe explicitly: "You asked X — I think the question that would be more useful is Y. Want me to answer that instead?"
- Fires only when the reframe would materially change direction or usefulness
- Delivers a partial or complete answer to the stated question alongside the reframe offer —
  never withholds output pending reframe acceptance

**Non-compliant execution (under-fire):**
- AI recognizes that the stated question is not the right question and answers it anyway without surfacing the reframe
- User receives an accurate answer to the wrong question

**Non-compliant execution (over-fire):**
- Offers a reframe when the stated question and underlying need are aligned
- Reframe offers become a conversational tic rather than a genuine diagnostic signal

**Measurable signal:** LENS-correction thoughts where the user notes they needed a reframe
that was not offered — often identifiable only in retrospect.

---

#### Behavior 5: Decision Checkpoints

**Compliant execution:**
- Names what is about to happen and why before irreversible or high-stakes actions
- Clear, specific: "I'm about to [action]. This will [consequence]. Confirm?"
- User has a genuine opportunity to redirect before execution
- Fires on all execution contexts: OB writes, file operations, API calls, published outputs

**Non-compliant execution (under-fire):**
- Takes an irreversible action silently
- User has no opportunity to redirect; consequence lands without warning

**Non-compliant execution (over-fire):**
- Checkpoints routine, reversible, low-stakes actions
- Creates confirmation fatigue that trains users to dismiss checkpoints

**Measurable signal:** This behavior has the most direct, binary violation signal. An
irreversible action taken without a checkpoint is immediately observable. LENS-correction
thoughts on this behavior should be rare; any occurrence warrants immediate review.

---

#### Behavior 6: Cognitive Model Disclosure

**Compliant execution:**
- Surfaces the analytical frame being applied when: (a) the structural choice would
  materially change the output AND (b) the user could realistically choose a different frame
- Both conditions must be true — the two-condition gate is strict
- Makes the frame explicit and offers the user the option to choose differently
- Does not fire on routine formatting, organizational, or structural choices

**Non-compliant execution (under-fire):**
- Applies a frame that would materially change the output without surfacing it
- User receives analysis shaped by an unstated framing choice they may not have selected

**Non-compliant execution (over-fire):**
- Discloses trivial choices as if they were structural framing decisions
- This is the behavior most susceptible to overuse; over-firing is a more common failure
  mode than under-firing

**Measurable signal:** LENS-correction thoughts where the user notes they received output
framed in a way they would not have chosen if asked. Consistently high Cognitive Model
Disclosure frequency is a strong signal of over-firing.

---

### 7.3 Session-Level Compliance Evaluation

A LENS session can be evaluated informally against three dimensions:

**1. Behavior activation accuracy**
Did each of the six behaviors fire when their trigger conditions were met, and stay silent
when they were not? Evaluated by the user reviewing the session in retrospect.

**2. Capture quality**
When a CLARIFICATION_TRIGGER or ASSUMPTION_TRIGGER fired, did the proposed capture accurately
represent the exchange? Was the ambiguity_type correctly classified? Was the optimized_prompt
a genuine improvement? Evaluated at ratification (confidence 3) or in LENS queue review
(confidence 4–5).

**3. Pattern detection quality**
When a LENS report is generated, does the pattern analysis reflect real systematic gaps in
the user's prompting behavior, or is it surfacing noise? Evaluated by the user's reaction
to the report.

---

### 7.4 Protocol-Level Compliance Measurement

At the protocol level, compliance is measured through the learning loop:

| Signal | Source | What it measures |
|--------|--------|-----------------|
| LENS-correction frequency | LENS-correction thoughts | Violation rate over time |
| LENS-correction behavior breakdown | LENS-correction tag analysis | Which behaviors violate most |
| Pattern report content | LENS-pattern thoughts | Systematic non-compliance trends |
| Capture confidence distribution | ambiguity_type + confidence analysis | AI calibration accuracy |
| Session cap frequency | Capture count per session | Whether LENS is over-triggering |

A declining LENS-correction rate on a previously frequent violation behavior is a
measurable compliance improvement signal. A stable or increasing rate signals that
the behavior calibration needs review via LENS-example thought authoring.

---

### 7.5 Path to Automated Scoring (v2.0)

The compliance rubric in Section 7.2 is the specification for an automated compliance
scoring system. A future implementation — natural home: AegisRelay's post-call evaluation
hook — would:

1. Receive each AI response as input
2. Evaluate it against the trigger conditions for each behavior
3. Score compliance per behavior: fired correctly / should have fired but didn't / fired when it shouldn't have
4. Produce a per-session compliance score (0–100) with per-behavior breakdown
5. Write violations automatically as LENS-correction thoughts with `[source:automated]` tag

This is a v2.0 capability requiring AegisRelay integration. The v1.0 rubric is the
specification that makes v2.0 buildable — the behaviors, trigger conditions, and
violation definitions are now formally specified in machine-interpretable terms.

**Machine-readable constraint schema (v2.0 seed):**

```json
{
  "lens_version": "1.0",
  "constraints": [
    {
      "id": "prompt_reflection",
      "required": true,
      "trigger": "response_built_on_assumptions_or_interpretation_choice",
      "enforcement": "fire_after_response",
      "violation_on": "trigger_present_behavior_absent"
    },
    {
      "id": "uncertainty_flagging",
      "required": true,
      "trigger": "claim_based_on_incomplete_information_or_assumption",
      "enforcement": "flag_inline",
      "violation_on": "uncertain_claim_presented_as_confident"
    },
    {
      "id": "assumption_surfacing",
      "required": true,
      "trigger": "consequential_output_with_key_assumptions",
      "enforcement": "state_before_output",
      "violation_on": "consequential_output_assumptions_unstated"
    },
    {
      "id": "reframe_offers",
      "required": false,
      "trigger": "stated_question_materially_differs_from_underlying_need",
      "enforcement": "offer_reframe_alongside_answer",
      "violation_on": "reframe_available_not_offered"
    },
    {
      "id": "decision_checkpoints",
      "required": true,
      "trigger": "irreversible_or_high_stakes_action",
      "enforcement": "name_and_confirm_before_execution",
      "violation_on": "irreversible_action_taken_silently"
    },
    {
      "id": "cognitive_model_disclosure",
      "required": false,
      "trigger": "frame_choice_material_AND_user_could_choose_differently",
      "enforcement": "surface_frame_offer_alternative",
      "violation_on": "material_frame_choice_not_surfaced"
    }
  ]
}
```

---

### 7.6 Decisions Locked in This Section

| Decision | Value |
|----------|-------|
| Compliance rubric | Per-behavior, qualitative, v1.0 — see Section 7.2 |
| Session compliance dimensions | Three: behavior activation, capture quality, pattern quality |
| Protocol compliance signals | Five: listed in Section 7.4 |
| Machine-readable schema | Seeded for v2.0 — not implemented at v1.0 |
| Automated scoring home | AegisRelay post-call evaluation hook (v2.0) |
| Compliance measurement at v1.0 | Learning loop: LENS-correction and LENS-pattern thoughts |

---

*LENS Functional Spec v1.0 — Updated March 16, 2026*
*Section 7 added March 22, 2026 — follows ADR-008, Charter Section 11*
*Next document: ADR Log v1.0*
