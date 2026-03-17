# LENS Data Dictionary v1.0
**Status:** Ratified
**Date:** March 16, 2026
**Follows:** Use Case Spec v1.0
**Precedes:** Functional Spec v1.0

---

## 1. Purpose

This document defines every field in the LENS capture schema — its type, purpose, valid values,
and risk classification. It also defines the retention policy and the canonical ambiguity_type
taxonomy. All implementation decisions that reference field definitions should cite this document.

---

## 2. Schema Overview

LENS captures are stored as OpenBrain thoughts. Each capture corresponds to one trigger event.
The schema has two tiers:

| Tier | Description | Required |
|------|-------------|----------|
| Tier 1 | Core capture fields — required for every capture | Yes |
| Tier 2 | Enrichment fields — optional, added when available | No |

---

## 3. Tier 1 — Required Fields

### 3.1 trigger_type
**Type:** TEXT (enum)
**Required:** Yes
**Valid values:**
- `CLARIFICATION_TRIGGER` — AI asked a clarifying question that blocked execution
- `ASSUMPTION_TRIGGER` — AI chose silently between interpretations without asking
- `SUCCESS_PATTERN` — User-invoked via "LENS+" to capture a high-quality exchange

**Notes:** ASSUMPTION_TRIGGER is client-capability-dependent. Claude supports it. Other clients
may not reliably detect silent assumption selection. When in doubt, default to CLARIFICATION_TRIGGER.

**Risk:** Low. No sensitive content.

---

### 3.2 timestamp
**Type:** TEXT (ISO 8601: YYYY-MM-DDTHH:MM:SS)
**Required:** Yes
**Source:** AI system time at moment of capture proposal

**Notes:** Timestamp reflects when the capture was proposed, not when the user confirmed it.
Consistent with OpenBrain capture_thought behavior.

**Risk:** Low.

---

### 3.3 ai_model
**Type:** TEXT (composite: provider + model_name + model_version)
**Required:** Yes
**Format:** `provider:model_name:version` — e.g., `anthropic:claude-sonnet-4-6:20250514`
**Source:** AI self-report at time of capture

**Notes:** Required to prevent behavioral bias in pattern analysis. Without model tracking,
patterns that appear to reflect the user's prompting may actually reflect model-specific behavior
differences. Gee identified this as a critical field for dataset integrity.

**Risk:** Low.

---

### 3.4 prompt_type
**Type:** TEXT (enum)
**Required:** Yes
**Valid values:**
- `analysis` — asking the AI to evaluate, assess, or interpret something
- `generation` — asking the AI to create content (writing, code, documents)
- `planning` — asking the AI to sequence, schedule, or strategize
- `research` — asking the AI to find, summarize, or synthesize information
- `summarization` — asking the AI to condense existing content
- `critique` — asking the AI to review or provide feedback
- `instruction` — asking the AI to do something specific (execute a task)
- `other` — does not fit above categories

**Notes:** Classify based on the primary intent of the original prompt. When a prompt spans
multiple types, choose the dominant intent.

**Risk:** Low.

---

### 3.5 original_prompt
**Type:** TEXT (full text)
**Required:** Yes
**Source:** The prompt exactly as submitted by the user, before any clarification exchange

**Notes:** This is the highest-risk field in the schema. Raw prompts routinely contain names,
health details, financial information, professional context, and other sensitive content.
Lexi (Perplexity) flagged this field as the primary privacy risk in LENS design (ADR-007).

**Risk:** HIGH. See Section 6 (Retention Policy) and ADR-007 in the ADR Log for the storage
decision rationale. In LENS v1.0, original_prompt is stored as a personal tool with
self-hosted OpenBrain. User accepts this risk explicitly for diagnostic value.
If LENS is ever made portable, this field must be reconsidered.

---

### 3.6 ambiguity_type
**Type:** TEXT (enum)
**Required:** Yes — for CLARIFICATION_TRIGGER and ASSUMPTION_TRIGGER captures
**Not required for:** SUCCESS_PATTERN captures (use null)
**Valid values:** See Section 5 (Canonical Ambiguity Type Taxonomy)

**Notes:** The most analytically valuable Tier 1 field for pattern detection. The ambiguity_type
taxonomy defines why a prompt was underspecified. Patterns in this field over time reveal
systematic gaps in prompting behavior.

**Risk:** Low.

---

### 3.7 missing_dimension
**Type:** TEXT (free text, one sentence)
**Required:** Yes — for CLARIFICATION_TRIGGER and ASSUMPTION_TRIGGER captures
**Not required for:** SUCCESS_PATTERN captures (use null)
**Source:** AI-authored at capture time, based on what the clarifying exchange revealed

**Example:** "Interview stage (first-round vs. final) was not specified, changing the depth
and focus of preparation advice."

**Notes:** Gee identified this as the most important field for long-term dataset value. The
delta_reason explains what changed; missing_dimension names what was absent from the original
prompt. Both are required for a complete diagnostic record.

**Risk:** Medium. May contain professional context. Retained per standard retention policy.

---

### 3.8 assumption_resolution_mode
**Type:** TEXT (enum)
**Required:** Yes — for ASSUMPTION_TRIGGER captures
**Not required for:** CLARIFICATION_TRIGGER and SUCCESS_PATTERN captures (use null)
**Valid values:**
- `ai_chose` — AI selected one interpretation without surfacing the choice
- `user_confirmed` — AI surfaced the assumption and user confirmed before proceeding
- `user_redirected` — AI surfaced the assumption and user chose a different interpretation

**Notes:** Only populated when trigger_type is ASSUMPTION_TRIGGER. Records how the assumption
was handled, which is analytically valuable for understanding where silent AI choices affect output.

**Risk:** Low.

---

### 3.9 optimized_prompt
**Type:** TEXT (full text)
**Required:** Yes
**Source:** AI-authored at capture time — the prompt that would have produced the best result
without requiring clarification

**Notes:** This is not a correction of the user's prompt — it is the AI's interpretation of
what the ideal prompt would have looked like given what the clarifying exchange revealed.
The user may confirm, edit, or decline the optimized prompt at ratification.

**Risk:** HIGH. Same risk profile as original_prompt. Contains AI-constructed prompt language
that may reflect sensitive context from the exchange. Retained per ADR-007.

---

### 3.10 delta_reason
**Type:** TEXT (one sentence)
**Required:** Yes
**Source:** AI-authored at capture time

**Example:** "Without the interview stage, preparation advice defaults to generic frameworks
rather than role-specific behavioral or technical depth."

**Notes:** Explains why the optimized_prompt produces a materially better result than the
original_prompt. Gee identified delta_reason as the field that makes the dataset actionable —
the user can read it and immediately understand what to change about their prompting behavior.

**Risk:** Low to Medium. Professional context may appear. Standard retention.

---

### 3.11 underspecification_confidence
**Type:** INTEGER (1–5 scale)
**Required:** Yes
**Scale:**
- 1 — Very low confidence: the prompt may have been intentionally brief
- 2 — Low confidence: underspecification likely but not certain
- 3 — Medium confidence: clear that more specificity would have improved the output
- 4 — High confidence: underspecification materially affected the output
- 5 — Very high confidence: the prompt could not have produced a useful result without clarification

**Notes:** Provides a noise filter for pattern analysis. Captures where the AI is genuinely
uncertain whether the prompt was underspecified vs. intentionally brief. Low-confidence captures
should be weighted less heavily in pattern detection.

**Risk:** Low.

---

## 4. Tier 2 — Optional Fields

### 4.1 outcome_status
**Type:** TEXT (enum)
**Valid values:**
- `improved` — the clarified/optimized prompt produced a noticeably better result
- `equivalent` — clarification did not materially change the output quality
- `declined` — user declined the optimized prompt at ratification
- `unknown` — outcome not assessed

**Notes:** Populated when the user provides feedback after seeing the result. Useful for
calibrating underspecification_confidence scoring over time.

**Risk:** Low.

---

### 4.2 improvement_dimension
**Type:** TEXT (enum)
**Valid values:**
- `precision` — response became more specific and targeted
- `relevance` — response became more applicable to the actual need
- `completeness` — response covered more of what was needed
- `efficiency` — response required fewer follow-up exchanges
- `accuracy` — response contained fewer assumptions or errors

**Notes:** Populated when outcome_status is `improved`. Identifies which aspect of output
quality benefited most from the clarification.

**Risk:** Low.

---

### 4.3 effective_pattern
**Type:** TEXT (free text, one sentence)
**Notes:** Populated for SUCCESS_PATTERN captures only. Describes the prompting pattern
that made the exchange work well — for use in LENS-example and LENS-pattern OpenBrain tags.

**Example:** "Provided role context, company stage, and specific outcome goal in a single
structured prompt — produced a targeted analysis without any clarification needed."

**Risk:** Low.

---

## 5. Canonical Ambiguity Type Taxonomy

This taxonomy defines the valid values for the `ambiguity_type` field. It answers the question:
why was this prompt underspecified?

| Value | Definition | Example |
|-------|------------|---------|
| `task_definition` | What the AI was being asked to do was unclear | "Help me with this" — help how? |
| `input_data` | The subject matter or data the AI needed to act on was absent or incomplete | "Analyze this" — analyze what? |
| `constraints` | Boundaries, limitations, or requirements were not stated | "Write something short" — how short? |
| `success_criteria` | What a good result looks like was not defined | "Make this better" — better how? |
| `context` | Background information needed to produce a relevant response was missing | "Follow up on this" — follow up with whom, about what? |
| `scope` | The breadth or depth of the response was not specified | "Tell me about cloud" — which aspect, at what depth? |
| `format` | The desired output format or structure was not specified | "Give me the analysis" — as a table, prose, bullets? |
| `recipient` | Who the output is for was not specified, affecting tone and framing | "Write an email" — to whom, in what relationship? |
| `timeframe` | A relevant time constraint or reference period was missing | "Show me the trend" — over what period? |
| `multiple` | Two or more ambiguity types were present simultaneously | Use when more than one type applies equally |

**Assignment rule:** Choose the single most dominant ambiguity type. Use `multiple` only when
two types are genuinely equal contributors and neither can be subordinated.

---

## 6. Retention Policy

### 6.1 Governing decision
LENS v1.0 is a personal tool with self-hosted OpenBrain (Supabase). Formal GDPR/CCPA compliance
obligations are lower at this scale. User has explicitly accepted the storage of raw prompts
(original_prompt, optimized_prompt) for diagnostic value. See ADR-007 in the ADR Log for the
full rationale and Lexi's privacy risk analysis.

### 6.2 Retention periods

| Field category | Retention | Rationale |
|----------------|-----------|-----------|
| Raw prompt fields (original_prompt, optimized_prompt) | 90 days | Highest privacy risk; time-limited to reduce exposure while preserving diagnostic value |
| Derived metadata (ambiguity_type, missing_dimension, delta_reason, prompt_type) | Indefinite | Low risk; high analytical value for long-term pattern detection |
| System metadata (timestamp, ai_model, trigger_type) | Indefinite | No sensitive content; required for dataset integrity |
| Tier 2 fields | Indefinite | Low risk; enrichment only |

### 6.3 Implementation note
The 90-day retention on raw prompt fields is a policy decision. Implementation (automated TTL
or manual purge) is deferred to the Functional Spec. Until implemented, raw prompts persist
indefinitely — this is an accepted risk for v1.0.

---

## 7. OpenBrain Tagging Convention

All LENS captures are stored as OpenBrain thoughts with the following tags:

**Standard capture tags:**
```
[domain:LENS] [artifact:LENS-capture] [trigger:CLARIFICATION_TRIGGER|ASSUMPTION_TRIGGER|SUCCESS_PATTERN]
[model:provider:model_name] [prompt_type:value] [ambiguity_type:value]
```

**Learning loop tags (separate thoughts):**
```
[domain:LENS] [artifact:LENS-example]     ← Canonical authored examples
[domain:LENS] [artifact:LENS-correction]  ← Real corrective moments
[domain:LENS] [artifact:LENS-pattern]     ← Synthesized recurring themes
```

---

## 8. Open Questions for Functional Spec

The following are deferred to the Functional Spec:

1. Capture flow implementation — exact sequence of AI actions from trigger to OB write
2. Pattern detection algorithm — how LENS-pattern thoughts are synthesized from LENS-corrections
3. Automated TTL implementation for raw prompt field retention
4. Block-based prompt structure recommendation — format guidance for users who want to reduce
   CLARIFICATION_TRIGGER frequency

---

## 9. Decisions Locked in This Document

| Decision | Value |
|----------|-------|
| Tier 1 fields | 11 fields — see Section 3 |
| Tier 2 fields | 3 fields — see Section 4 |
| ambiguity_type taxonomy | 10 values — see Section 5 |
| Raw prompt retention | 90 days (implementation deferred to Functional Spec) |
| Derived metadata retention | Indefinite |
| Raw prompt storage | Accepted for v1.0 personal tool — see ADR-007 |
| OpenBrain tagging convention | Defined in Section 7 |

---

*LENS Data Dictionary v1.0 — Ratified March 16, 2026*
*Next document: Functional Spec v1.0*
