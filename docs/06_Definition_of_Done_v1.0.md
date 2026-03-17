# LENS Definition of Done v1.0
**Status:** Draft — pending ratification
**Date:** March 16, 2026
**Follows:** ADR Log v1.0
**Completes:** LENS v1.0 documentation set

---

## 1. Purpose

This document defines the criteria that must be satisfied for LENS v1.0 to be considered
complete and operational. It is the final gate before implementation begins. Nothing in this
document is aspirational — every criterion is binary: done or not done.

---

## 2. Documentation Criteria

All six documents must be ratified before implementation begins. No exceptions.

| Document | Status |
|----------|--------|
| 01 Charter v1.0 | ✅ Ratified |
| 02 Use Case Spec v1.0 | ✅ Ratified |
| 03 Data Dictionary v1.0 | ✅ Ratified |
| 04 Functional Spec v1.0 | ✅ Ratified |
| 05 ADR Log v1.0 | ✅ Ratified |
| 06 Definition of Done v1.0 | 🔄 This document |

**Gate:** All six documents committed to `jropenshaw1/LENS` on `main` branch with Ratified
status before any implementation work begins.

---

## 3. Implementation Criteria

### 3.1 Capture Flow
- [ ] LENS fires on CLARIFICATION_TRIGGER in Claude conversational sessions
- [ ] LENS fires on ASSUMPTION_TRIGGER in Claude conversational sessions
- [ ] LENS+ keyword fires SUCCESS_PATTERN capture
- [ ] Confidence 4–5 captures write automatically with lightweight notification
- [ ] Confidence 3 captures propose for manual ratification
- [ ] Confidence 1–2 captures are suppressed silently
- [ ] Session cap of 5 is enforced; cap resets at session start
- [ ] Declined captures do not increment the session cap
- [ ] Authorship rule correctly excludes coding sessions and AI-to-AI relay sessions
- [ ] `capture_mode` field correctly set to `auto` or `ratified`

### 3.2 OpenBrain Integration
- [ ] LENS captures written as OpenBrain thoughts with correct tag format
- [ ] All Tier 1 fields populated on every capture
- [ ] Duplicate detection fires before each write
- [ ] Write failure surfaces once, retries once, drops silently on second failure
- [ ] OB unavailable scenario produces no user-visible error

### 3.3 Pattern Detection
- [ ] "LENS report" and equivalent phrases trigger pattern detection
- [ ] Frequency table produced for `ambiguity_type` across corpus
- [ ] Model bias check performed and reported
- [ ] Trend analysis fires when corpus spans more than 14 days
- [ ] Report opens conversational discussion — does not terminate at table output
- [ ] LENS-pattern thought captured to OpenBrain after discussion

### 3.4 TTL / Retention
- [ ] At each LENS report, captures older than 90 days are identified and flagged
- [ ] TTL notice is persistent — fires at every LENS report until redaction is complete
- [ ] Ready-to-run SQL generated fresh at every TTL notice (SELECT preview + UPDATE)
- [ ] SQL targets Supabase `thoughts` table with correct LIKE filters for LENS-capture tag
- [ ] No automated redaction attempted — OpenBrain update tool not available at v1.0

### 3.5 Confidence Score Validation
- [ ] 30-day calibration checkpoint fires at first LENS report after 30 days of operation
- [ ] Checkpoint reviews auto-captured (confidence 4–5) entries for false positives
- [ ] Calibration findings captured as LENS-pattern thought tagged `[subtype:calibration]`
- [ ] Threshold adjustment documented if calibration reveals systematic miscalibration

---

## 4. Learning Loop Criteria

- [ ] Seven LENS-example thoughts authored — one per core behavior — tagged `[artifact:LENS-example]`
- [ ] LENS-correction thoughts captured automatically at corrective moments tagged `[artifact:LENS-correction]`
- [ ] LENS-pattern thoughts captured after pattern detection discussions tagged `[artifact:LENS-pattern]`

**Note on LENS-example thoughts:** These must be authored before LENS is considered fully
operational. They are the canonical reference the AI searches before executing each behavior.
Authoring them is part of v1.0 implementation, not deferred to v1.1.

---

## 5. Operational Criteria

- [ ] LENS operates without interrupting conversational flow
- [ ] Auto-capture notifications are lightweight — one line, no response required
- [ ] LENS goes silent when session cap is reached — no notification unless asked
- [ ] LENS goes silent when OpenBrain is unavailable — no error surfaced unless asked
- [ ] "Show LENS queue" surfaces the session's auto-captures for review on demand
- [ ] "How many LENS captures this session?" returns the current count on demand

---

## 6. v1.0 Known Gaps (Accepted, Not Blocking)

These gaps are acknowledged and accepted for v1.0. They do not block the Definition of Done.
Each has a documented ADR or deferral decision.

| Gap | ADR | v1.1 candidate |
|-----|-----|----------------|
| TTL enforcement is manual — 90-day policy is aspirational | ADR-005 | Yes — blocked on OB update tool |
| Confidence scoring is AI self-reported — calibration validated empirically at 30 days | ADR-003 | Ongoing |
| Tiered retention and aggregate synthesis not implemented | ADR-006 | Yes — after data quality assessed |
| Option A (deferred queue) not implemented | ADR-003 | Yes |
| Option B (trust escalation) not implemented | ADR-003 | Yes |
| Retrieval drift across clients accepted | ADR-001 | Deferred indefinitely |
| ASSUMPTION_TRIGGER client support uneven | ADR-002 | Revisit per client capability |

---

## 7. v1.1 Backlog (Out of Scope for v1.0)

Items explicitly deferred from v1.0 that form the natural v1.1 scope:

1. Tiered retention with LENS-aggregate thought synthesis (ADR-006)
2. Option A — deferred capture queue with batch approval (ADR-003)
3. Option B — trust escalation model (ADR-003)
4. Automated TTL enforcement when OpenBrain exposes update tool (ADR-005)
5. "LENS off" explicit opt-out for exploratory sessions (ADR-007)
6. Session cap adjustment based on v1.0 practice data (ADR-007)
7. Confidence threshold adjustment based on 30-day calibration results (ADR-003)
8. Worked examples enrichment as organic LENS-correction data accumulates (learning loop)

---

## 8. Sign-Off

LENS v1.0 is complete when:
1. All six documents are ratified and committed to `main`
2. All implementation criteria in Sections 3–5 are checked
3. Seven LENS-example thoughts are authored and captured to OpenBrain
4. At least one real CLARIFICATION_TRIGGER capture has been confirmed end-to-end

The last criterion — one real end-to-end capture — is the acceptance test. Until a live
trigger has fired, been evaluated, proposed, ratified (or auto-captured), and written to
OpenBrain with correct field population, LENS is in implementation, not operation.

---

*LENS Definition of Done v1.0 — Draft*
*This completes the LENS v1.0 documentation set.*
