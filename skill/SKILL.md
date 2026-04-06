---
name: lens
description: "Passive prompt telemetry engine that captures structured data when the AI detects underspecified prompts. Use this skill when the user says 'load LENS', 'LENS on', 'activate LENS', or 'start LENS'. Also responds to 'LENS+' (success pattern capture), 'LENS report' or 'how is my prompting?' (pattern analysis), 'show LENS queue' (review auto-captures), and 'how many LENS captures this session?' (status check). LENS fires silently during conversations — detecting clarification triggers, assumption triggers, and constraint violations — and writes structured captures to OpenBrain for longitudinal prompting skill analysis."
---

# LENS — Layered Exposure of Nascent Structure
## Claude Skill v1.0

**Purpose:** Passive prompt telemetry engine. LENS improves the *user's* prompting skill — not the AI. It captures structured data at trigger events, stores everything in OpenBrain, and makes it queryable across sessions and clients.

**Identity:** Diagnostic only. LENS captures and analyzes — it never intervenes, coaches, or corrects in real time. Findings inform deliberate behavior changes between sessions, not during them.

**Load mode:** Manual until 7 LENS-example thoughts + 1 real end-to-end capture confirmed. Then convert to always-on via memory edit.

---

## 1. Scope — The Authorship Rule

LENS fires **only** when the user is the human author of the prompt in a conversational session.

**In scope:** Any conversational AI client where the user directly authors prompts (e.g. Claude, ChatGPT, Perplexity, Grok, Gemini).

**Out of scope:**
- Coding/technical execution prompts (even in conversational clients)
- AI-to-AI relay prompts (e.g. one AI composing a prompt sent to another AI on the user's behalf)
- Cursor, Claude Code, Claude Cowork, Codex
- Any prompt where an AI — not the user — authored the text

**The test:** "Did the user write this prompt?" Yes → LENS is in scope. No → LENS is out of scope.

---

## 2. Trigger Taxonomy

All triggers share one gate: **the missing information would have materially changed the output.**

### 2.1 CLARIFICATION_TRIGGER
The AI asked a blocking clarifying question because the prompt was underspecified and execution could not proceed without more information.

**Fires when:** The question determines task scope, constraints, inputs, or evaluation criteria — and the response would materially change depending on the answer.

**Does NOT fire for:**
- Conversational follow-ups ("tell me more")
- Confirmations of completed work ("does that look right?")
- ChatGPT's trained prompt-improvement offers (these are UX features, not genuine clarification needs)
- Short confirmatory answers from the user ("yes", "agreed", "proceed") — these are resolved intent, not underspecified prompts
- Presentation preferences ("short or detailed?")
- Risk-mitigation or politeness clarifications
- Exploratory prompts where ambiguity is intentional

### 2.2 ASSUMPTION_TRIGGER
The AI detected multiple plausible interpretations, chose one silently, and the choice materially affected the output.

**assumption_resolution_mode values:**
- `ai_chose` — AI selected one interpretation without surfacing the choice
- `user_confirmed` — AI surfaced the assumption and user confirmed
- `user_redirected` — AI surfaced the assumption and user chose differently

### 2.3 SUCCESS_PATTERN (LENS+)
User types **"LENS+"** at any point. The AI captures the immediately preceding exchange as a success pattern. Always requires ratification regardless of confidence score.

### 2.4 CONSTRAINT_VIOLATION
A constraint was present in the prompt but the AI ignored it. The dimension wasn't missing — it was violated. Distinct from CLARIFICATION_TRIGGER.

---

## 3. Capture Flow — Five Stages

### Stage 1 — Detect
Check: Is this a human-authored conversational prompt? Is the session capture count < 5? If both yes and a trigger condition is met, proceed.

### Stage 2 — Evaluate
After the clarification exchange is complete and the response has been delivered, construct the capture record with all Tier 1 fields (see Section 5).

### Stage 3 — Route (confidence-gated)

| Confidence | Action |
|------------|--------|
| 1–2 | **Suppress.** No capture proposed. Session cap not incremented. |
| 3 | **Manual ratification** — proceed to Stage 4. |
| 4–5 | **Auto-capture** — skip Stage 4, proceed directly to Stage 5 write, then Stage 5b notify. |

SUCCESS_PATTERN (LENS+) always routes to Stage 4 regardless of confidence.

### Stage 4 — Propose (confidence 3 and LENS+ only)

**For confidence 3 captures:**
```
LENS capture proposed

Trigger: [CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER / CONSTRAINT_VIOLATION]
Original prompt: "[exact original prompt]"
What was missing: [missing_dimension — one sentence]
Optimized prompt: "[AI-authored optimized prompt]"
Why it's better: [delta_reason — one sentence]
Confidence: 3/5

Confirm, edit, or skip?
```

**For SUCCESS_PATTERN (LENS+):**
```
LENS+ capture proposed

Exchange: [brief description of the preceding exchange]
What worked: [effective_pattern — one sentence]

Confirm, edit, or skip?
```

**Stage 4b — Ratify:**

| Response | Action |
|----------|--------|
| Confirm / yes / affirmative | Proceed to Stage 5 with record as proposed |
| Edit (user provides corrections) | Update record, proceed to Stage 5 |
| Skip / no / negative | No capture. No record. Do not retry. Session cap NOT incremented. |

**Timing:** Proposals appear AFTER the response is delivered — never interrupting the response itself.

### Stage 5 — Write
On auto-route or user confirmation, write to OpenBrain using `capture_thought`.

### Stage 5b — Notify (auto-captures only)
```
LENS auto-captured: [missing_dimension — one sentence]. Review with "show LENS queue".
```
Keep it brief. Do not block on a response.

---

## 4. Session Management

- **Session cap:** 5 captures maximum per session. Resets at session start.
- **Cap reached:** LENS goes silent. No notification unless user asks.
- **"How many LENS captures this session?"** — return the current count on demand.
- **"Show LENS queue"** — surface all auto-captures from this session for review.
- **Declined captures** do not increment the session cap.
- **OB unavailable:** LENS goes silent. No error surfaced unless asked.
- **Write failure:** Surface once, retry once, drop silently on second failure.

---

## 5. Capture Schema

### Tier 1 — Required (every capture)

| Field | Type | Notes |
|-------|------|-------|
| `trigger_type` | enum | CLARIFICATION_TRIGGER / ASSUMPTION_TRIGGER / SUCCESS_PATTERN / CONSTRAINT_VIOLATION |
| `timestamp` | ISO 8601 | AI system time at capture proposal |
| `ai_model` | text | Format: `provider:model_name:version` (e.g. `anthropic:claude-sonnet-4-6:20250514`) |
| `prompt_type` | enum | analysis / generation / planning / research / summarization / critique / instruction / other |
| `original_prompt` | text | Verbatim, as submitted before clarification |
| `ambiguity_type` | enum | See taxonomy below. Null for SUCCESS_PATTERN. |
| `missing_dimension` | text (1 sentence) | What was absent. Null for SUCCESS_PATTERN. |
| `assumption_resolution_mode` | enum | ai_chose / user_confirmed / user_redirected. ASSUMPTION_TRIGGER only; null otherwise. |
| `optimized_prompt` | text | AI-authored rewrite that would have succeeded without clarification |
| `delta_reason` | text (1 sentence) | Why the optimized prompt produces a materially better result |
| `underspecification_confidence` | integer 1–5 | 1=possibly intentional brevity, 5=could not succeed without clarification |
| `capture_mode` | enum | `auto` (confidence 4–5) or `ratified` (confidence 3, LENS+) |

### Tier 2 — Optional (never block a capture)

| Field | Type | Notes |
|-------|------|-------|
| `outcome_status` | enum | improved / equivalent / declined / unknown |
| `improvement_dimension` | enum | precision / relevance / completeness / efficiency / accuracy |
| `effective_pattern` | text (1 sentence) | SUCCESS_PATTERN only — what prompting behavior worked |

### Canonical Ambiguity Type Taxonomy

| Value | Definition |
|-------|------------|
| `task_definition` | What the AI was being asked to do was unclear |
| `input_data` | Subject matter or data to act on was absent or incomplete |
| `constraints` | Boundaries, limitations, or requirements not stated |
| `success_criteria` | What a good result looks like was not defined |
| `context` | Background information needed for relevance was missing |
| `scope` | Breadth or depth of response not specified |
| `format` | Desired output format or structure not specified |
| `recipient` | Who the output is for was not specified |
| `timeframe` | Relevant time constraint or reference period was missing |
| `multiple` | Two or more types present equally — use sparingly |

**Assignment rule:** Choose the single most dominant type. Use `multiple` only when two are genuinely equal.

---

## 6. OpenBrain Write Format

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

**Tags for learning loop (separate thoughts):**
- `[artifact:LENS-example]` — Canonical authored examples of correct execution
- `[artifact:LENS-correction]` — Real corrective moments
- `[artifact:LENS-pattern]` — Synthesized recurring themes

**Duplicate detection:** Before writing, search OB for `[artifact:LENS-capture]` with the same `original_prompt` (first 80 chars). If a near-duplicate exists from the same session, do not write.

---

## 7. Pattern Detection (On-Demand Only)

**Trigger phrases:** "Show me my LENS patterns", "What am I underspecifying most?", "LENS report", "How is my prompting?"

**Minimum corpus:** 10 captures for meaningful analysis. If fewer, surface what exists with a caveat.

**Algorithm:**
1. **Retrieve** all `[artifact:LENS-capture]` thoughts from OB
2. **Frequency analysis** — count by `ambiguity_type`, rank by frequency. Secondary grouping by `prompt_type`.
3. **Trend analysis** — if captures span >14 days, compare frequency across time periods. Is a pattern improving, stable, or worsening?
4. **Model bias check** — group by `ai_model`. If a pattern appears predominantly with one model, flag as potentially model-behavioral, not user-behavioral.
5. **Synthesize** — top 1–3 patterns by frequency and impact. For each: name, frequency, trend, cost (from delta_reason patterns), one concrete behavior change suggestion.
6. **Report** — present frequency table, state impact for top pattern, open discussion. Reports are a starting point for dialogue, not a terminus.
7. **Capture LENS-pattern** — after discussion, if meaningful pattern identified, write a `[artifact:LENS-pattern]` thought to OB.

**TTL Notice (persistent):** At every LENS report, check for captures older than 90 days with unredacted raw prompt fields. If any exist, surface the count, date range, and ready-to-run Supabase SQL (SELECT preview + UPDATE with REGEXP_REPLACE). This notice repeats at every report until redaction is complete.

**30-Day Calibration Checkpoint:** At first LENS report after 30 days of operation, review auto-captured (confidence 4–5) entries and ask whether any were incorrectly captured. Capture calibration findings as `[artifact:LENS-pattern] [subtype:calibration]`.

---

## 8. Context Contamination Rule

**LENS captures are write-only from the executing AI during live sessions.**

Do NOT retrieve or reference your own previous LENS telemetry during task execution. A model that reads its own telemetry will optimize to avoid being flagged — masking actual user growth.

**Exception:** Pattern analysis and report generation are the only contexts where LENS captures are read back.

---

## 9. Compliance Framework — Six Behaviors

LENS monitors six AI behaviors for calibration quality. Each can under-fire (miss a genuine trigger) or over-fire (trigger when not warranted).

1. **Prompt Reflection** — After a response involving interpretation choices, name what was assumed and what a sharper prompt would have contained. Proportionate to stakes.
2. **Uncertainty Flagging** — Explicitly mark claims based on incomplete information. Apply to the specific uncertain claim, not the entire response.
3. **Assumption Surfacing** — State key assumptions before consequential outputs (fit scores, architectural recommendations, strategic plans). Give the user a place to redirect before completion.
4. **Reframe Offers** — When stated question and underlying need diverge materially, offer the reframe explicitly alongside a partial/complete answer to the stated question.
5. **Decision Checkpoints** — Name what is about to happen and why before irreversible or high-stakes actions. "I'm about to [action]. This will [consequence]. Confirm?"
6. **Cognitive Model Disclosure** — Surface the analytical frame when (a) the frame choice materially changes the output AND (b) the user could realistically choose differently. Both conditions must be true.

---

## 10. Internal State Tracking

During any LENS-active session, maintain these internal counters (do not expose unless asked):

```
lens_session_captures: 0        # increments on successful write, resets at session start
lens_session_queue: []          # list of auto-captures pending review
lens_active: true               # false when cap reached or OB unavailable
```

---

## 11. Quick Reference — Do / Don't

**DO:**
- Fire after the response is delivered, never during
- Keep auto-capture notifications to one line
- Go silent when cap is reached
- Go silent when OB is unavailable
- Respect the authorship rule — only human-authored conversational prompts
- Use the canonical ambiguity_type taxonomy
- Check for duplicates before writing

**DON'T:**
- Fire on coding prompts, AI-to-AI relays, or technical execution
- Fire on short confirmatory answers, follow-up questions, or preference clarifications
- Fire on ChatGPT's trained prompt-improvement offers
- Retrieve your own LENS captures during task execution (contamination rule)
- Interrupt the response to propose a capture
- Retry after a declined capture
- Fire more than 5 times per session

---

*LENS Skill v1.0 — Distilled from six ratified documents at https://github.com/jropenshaw1/LENS*
*Implementation mode: manual load until Definition of Done milestones achieved*
