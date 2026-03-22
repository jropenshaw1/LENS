# LENS ADR Log v1.0
**Status:** Ratified
**Date:** March 16, 2026
**Follows:** Functional Spec v1.0
**Precedes:** Definition of Done v1.0

---

## Purpose

This log records all Architecture Decision Records (ADRs) for LENS v1.0. Each ADR documents
a significant design decision — the context that drove it, the options considered, the decision
made, and the consequences. ADRs are immutable once ratified. Superseding decisions create
new ADRs; they do not modify existing ones.

---

## ADR-001: OpenBrain as Sole Repository — Including Failure Mode Behavior

**Date:** March 10 / March 16, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw

### Context
LENS captures need a persistent store that is cross-client retrievable. OpenBrain (OB1 v2,
self-hosted Supabase) was selected as the sole repository. However, the initial decision did
not define behavior when OpenBrain is unavailable or writes fail. This ADR covers both the
repository selection and the failure mode decisions.

### Repository Decision
OpenBrain (OB1 v2) is the sole repository for LENS captures. Portability to users without
OpenBrain is deferred indefinitely. The analytical value of LENS depends on structured,
queryable data that AI personalization files cannot support.

### Failure Mode Decisions

**OpenBrain unavailable:**
No capture attempt. LENS goes silent for that trigger event. The user is not notified
unless they ask. Rationale: surfacing infrastructure errors mid-conversation contradicts
the passive, non-interruptive design principle.

**Write fails:**
Surface the failure once, offer a single retry. If retry fails, drop the capture silently.
Do not retry repeatedly or block the conversation. Log the failure in session context only —
not to OpenBrain (since it is unavailable).

**Duplicate writes:**
Before writing, check for an existing capture with the same timestamp and trigger_type.
If a duplicate is detected, surface it to the user and ask whether to overwrite or skip.
Do not write silently over an existing record.

**Retrieval drift across clients:**
OpenBrain is an eventually consistent system. Different clients reading captures in close
temporal proximity may see different states. This is a known limitation of the architecture.
LENS does not attempt to resolve retrieval drift at v1.0. It is acknowledged and accepted.

### Consequences
- LENS operates silently when OpenBrain is unavailable — no user disruption
- Captures lost due to write failures are not recoverable
- Duplicate detection adds one read operation per capture — accepted overhead
- Retrieval drift is a known gap; cross-client consistency is not guaranteed

---

## ADR-002: Authorship Rule as Scope Boundary — Including Uncertainty Default

**Date:** March 16, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw

### Context
LENS needs a durable rule for determining which AI sessions are in scope. A client-based list
was rejected in favor of an authorship-based rule: LENS fires when the user is the human
author of the prompt. However, the initial decision did not define behavior when authorship
is genuinely uncertain.

### Decision
LENS fires when the user is the human author of the prompt. When authorship is uncertain,
LENS defaults to no fire.

### Authorship Uncertainty Default
When the AI cannot determine whether the prompt was human-authored or AI-generated,
LENS does not fire. The AI does not ask the user to clarify authorship mid-conversation,
and does not fire with an uncertainty tag. Rationale: (1) asking mid-conversation violates
the passive, non-interruptive principle; (2) uncertainty-tagged captures add noise to a
dataset that is already probabilistic; (3) a false negative (missed capture) is less harmful
than a false positive (polluted dataset entry).

### Consequences
- No maintained client list required
- AI-to-AI relay sessions are permanently out of scope
- Borderline authorship cases are suppressed, not tagged
- Some valid captures will be missed in ambiguous sessions — accepted as preferable to noise

---

## ADR-003: Ratification Model — Confidence-Gated Auto-Capture with Deferred Options

**Date:** March 14 / March 16, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw, with input from Claude, Gee, Copi

### Context
Strict require-ratification-on-every-capture was the initial design. Upon review, a valid
concern was identified: requiring the user to confirm every capture means the dataset quality
depends on whether the user was available and willing to engage at that moment. Valid signals
are missed when the user is focused on the primary task. Three alternative models were
evaluated.

### Options Considered

**Option A — Deferred capture queue (deferred to v1.1)**
LENS maintains a session queue of proposed captures. At session end or on demand
("show my LENS queue"), the user reviews and batch-approves, edits, or drops them.
Lower friction per capture, same user control, no interruption to primary work.
*Deferred because:* batch review requires building queue state management that adds
implementation complexity not warranted before the base capture system is validated.

**Option B — Trust escalation (deferred to v1.1)**
LENS tracks ratification history. If the user confirms N consecutive captures without
edits, LENS earns the right to auto-capture with notification but without blocking on
confirmation. Trust level resets if the user starts editing captures frequently.
*Deferred because:* trust calibration requires real capture history to set meaningful
thresholds. Cannot be designed correctly before v1.0 produces data. Also introduces
complexity in trust state management across sessions.

**Option C — Confidence-gated auto-capture (adopted for v1.0)**
Captures with `underspecification_confidence` of 4 or 5 auto-capture without explicit
ratification — the AI notifies the user but does not block on confirmation. Captures
with confidence 3 or below require manual ratification. The confidence score already
exists as a quality gate; using it for ratification routing is architecturally consistent.

### Decision
Option C: confidence-gated auto-capture. Confidence 4–5 captures are written automatically
with notification. Confidence 1–2 captures are suppressed entirely (per Functional Spec
Stage 2 evaluation rule). Confidence 3 captures require manual ratification.

### Rationale
The confidence score is the AI's own assessment of signal quality. High-confidence captures
(4–5) are clear-cut trigger events where the AI's interpretation is unlikely to require
correction — auto-capturing them reduces missed signals without materially degrading dataset
quality. Confidence 3 captures sit at the boundary where user judgment adds the most value.
Low-confidence captures (1–2) are suppressed regardless — they are noise by definition.

### Notification format for auto-captures (confidence 4–5)
> "LENS auto-captured: [brief description of trigger and missing dimension].
> Review with 'show LENS queue' or confirm with 'LENS ok'."

This keeps the user informed without requiring a response.

### Consequences
- High-confidence captures (4–5) are written automatically — fewer missed signals
- Confidence 3 captures still require ratification — user judgment applied where it matters most
- The dataset will contain both auto-captured and user-ratified entries — both are valid signals
- Option A (deferred queue) and Option B (trust escalation) are documented candidates for v1.1
- Option A is the natural complement to Option C — batch review of the auto-capture log

---

## ADR-004: Diagnostic Primary; Explicit Non-Training Boundary

**Date:** March 14, 2026 / clarified March 16, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw

### Context
LENS v1.0 builds a structured corpus of prompt captures, optimized prompts, and ambiguity
classifications. This corpus is training-adjacent by nature — it could theoretically be used
to fine-tune AI model behavior. The original decision deferred training use cases without
explicitly prohibiting them. Upon review, the lack of a formal boundary was identified as
a gap.

### Decision
LENS v1.0 is diagnostic only. The following uses of the LENS capture corpus are explicitly
prohibited at v1.0 and require a new ADR to enable at any future version:

1. Fine-tuning or adapting any AI model using LENS capture data
2. Sharing LENS capture data with any AI provider or third party
3. Using LENS capture data for any purpose other than the user's own personal prompting
   pattern analysis and improvement

### Rationale
Building the diagnostic foundation first produces the dataset that would eventually support
training use cases. The sequencing is correct — measure first, train later if warranted.
Formalizing the prohibition removes ambiguity about permissible uses and protects the
integrity of the personal dataset.

### Consequences
- LENS v1.0 measures and surfaces patterns; it does not modify AI behavior
- The capture dataset is retained in a form compatible with future training use cases
  if the prohibition is lifted by a future ADR
- Any future training use case requires explicit re-evaluation of ADR-005 (raw prompt
  storage) and Lexi's privacy risk analysis

---

## ADR-005: Raw Prompt Storage — Accepted with Honest Assessment of TTL Gap

**Date:** March 14–16, 2026
**Status:** Accepted with acknowledged gap
**Decided by:** Jonathan Openshaw
**Risk flag raised by:** Lexi (Perplexity), March 14, 2026

### Context
The LENS schema stores `original_prompt` and `optimized_prompt` as Tier 1 required fields.
Lexi's privacy analysis identified raw prompt storage as the highest-risk decision in the
design. A 90-day retention policy was adopted. However, OpenBrain (OB1 v2) does not expose
a field-level update or delete tool. This means the retention policy has no enforcement
mechanism at v1.0.

### Decision
Raw prompts are stored in full. A 90-day retention policy is the stated intent. The
enforcement gap is acknowledged explicitly: the 90-day TTL is aspirational, not
architectural, at v1.0.

### Honest Assessment
The retention policy cannot be automatically enforced with the current OpenBrain capability.
Manual redaction at report time is the only available mechanism. This means raw prompts
may persist beyond 90 days if the user does not request a LENS report. This is a core
control dependency, not a minor implementation detail. The user accepts this gap explicitly
for v1.0 as a personal tool with self-hosted infrastructure.

### Remediation path
This gap is resolved when OpenBrain exposes a field-level update or delete tool. At that
point, TTL enforcement becomes architectural rather than aspirational. Until then, the AI
flags expired captures at every LENS report and the user controls redaction manually.

### Consequences
- Raw prompts may persist beyond 90 days without a LENS report being requested
- The retention policy is a stated intent with manual-only enforcement
- If LENS is ever made portable or multi-user, this gap becomes a blocking issue
- Lexi's nine privacy mitigations are the hardening roadmap for any portable version

---

## ADR-006: Tiered Retention Deferred to v1.1

**Date:** March 16, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw

### Context
During Functional Spec review, a tiered retention model was proposed: granular captures
for 90 days, aggregate summaries for 90–180 days, indefinite retention of aggregates
thereafter. The aggregate layer would preserve model-comparison utility across AI releases.
This was recognized as over-engineering for v1.0 before real capture data has been assessed.

### Decision
Tiered retention and aggregate synthesis are deferred to v1.1. At v1.0, the only retention
behavior is: raw prompt fields are flagged for manual redaction at 90 days via LENS report.
No aggregate thoughts are synthesized. No longitudinal analysis infrastructure is built.

### Rationale
Aggregate synthesis assumes the capture data is high enough quality and volume to produce
meaningful summaries. That assumption cannot be validated before v1.0 is live and collecting
data. The v1.1 decision will be informed by real capture history.

### Consequences
- No LENS-aggregate thoughts in v1.0
- Cross-AI-release longitudinal comparison is not supported at v1.0
- The tiered retention model remains a documented design candidate for v1.1
- ADR-005 gap (manual TTL) is the only retention mechanism at v1.0

---

## ADR-007: Session Capture Cap of 5

**Date:** March 16, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw

### Context
Without a session cap, exploratory sessions with frequent AI clarifications could produce
a high volume of low-signal captures. Two noise control mechanisms were evaluated: (1)
per-session cap, and (2) explicit opt-out ("LENS off").

### Decision
Per-session cap of 5 confirmed captures. Declined captures do not count. Cap resets at
session start.

### Rationale
The user restarts sessions frequently, which naturally resets the cap. An explicit opt-out
requires remembering to invoke it — contradicting the passive design principle.

### Consequences
- Maximum 5 captures per session
- Cap may be adjusted in v1.1 based on practice
- "LENS off" opt-out may be added in v1.1 if cap proves insufficient

---

## ADR-008: LENS Is a Protocol Specification, Not a Middleware Enforcement Layer

**Date:** March 22, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw
**Context input from:** Gee (ChatGPT) pressure test, March 2026; Claude analysis

### Context
During a structured pressure test of LENS v1.0 by Gee (ChatGPT), the following critique was
raised:

> "Right now, LENS behaves like a prompt discipline framework. Prompts are not enforceable.
> They degrade across chains. They are not auditable. If LENS stays prompt-based, it caps out
> as a methodology, not a product."

The critique proposed evolving LENS into a middleware enforcement layer — one that intercepts
prompts, injects constraints as structured objects, evaluates outputs post-call, scores
compliance, and optionally rejects or retries. The implicit framing was that remaining
"protocol-based" is a limitation to be overcome.

This ADR records the evaluation of that critique and the architectural decision that follows.

### Options Considered

**Option A — Evolve LENS into middleware**

LENS becomes a runtime enforcement component: an ingress/egress layer sitting between the
user and AI providers. It intercepts calls, injects constraints, evaluates outputs, and
produces compliance scores. Enforcement is architectural — non-compliant outputs can be
rejected or retried automatically.

*Consequences:*
- Requires infrastructure build: intercept layer, evaluation engine, scoring pipeline
- Couples LENS to a specific runtime architecture
- Increases implementation complexity by an order of magnitude
- Competes with AegisRelay's scope rather than complementing it
- Shifts LENS from a protocol that any AI client can adopt to a product requiring
  specific integration work

*Valid for:* A funded product development effort targeting enterprise middleware buyers.

**Option B — Remain as protocol specification, but make the positioning deliberate**

LENS is a protocol specification. It defines expected behaviors, trigger conditions,
compliance criteria, and an enforcement model. The implementing AI client is the enforcement
agent — it reads the protocol, loads the canonical reference, and executes accordingly.
LENS does not enforce at runtime because it is not a runtime component. It specifies what
compliant runtime behavior looks like.

This is the same relationship that:
- TCP/IP has with the networking stack that implements it
- ITIL has with the IT service management operations that satisfy it
- NIST CSF has with the security controls that implement it
- COBIT has with the governance practices built on top of it

None of these frameworks enforce at runtime. All define what compliant behavior looks like,
who is responsible for compliance, and how compliance is measured. Their value is precisely
that they are portable — any implementation can adopt them without the framework owning
the enforcement layer.

*Consequences:*
- LENS remains portable across AI clients without integration work
- Enforcement responsibility is explicit and documented (Charter Section 11)
- Compliance is measurable through the learning loop and compliance rubric
  (Functional Spec Section 7) without requiring a runtime evaluation engine
- LENS and AegisRelay remain complementary: LENS is the constraint specification;
  AegisRelay is a natural home for runtime implementations that enforce LENS constraints
- The protocol/middleware distinction is a narrative strength, not a limitation

### Decision
**Option B.** LENS is and remains a protocol specification. This is the correct
architecture for LENS v1.0 and the foreseeable future.

The Gee critique correctly identified that LENS has no runtime enforcement boundary. The
error in the critique was treating that as a flaw. It is a design choice — one that gives
LENS portability, vendor neutrality, and a natural integration point with AegisRelay rather
than competing with it.

The appropriate response to the critique is not to build middleware. It is to make the
protocol positioning explicit in the documentation, define the enforcement model clearly
(Charter Section 11), and establish a compliance framework that makes LENS measurable
without requiring a runtime enforcement engine (Functional Spec Section 7).

### Why "Protocol, Not Middleware" Is the Stronger Position

A middleware enforcement layer would require every AI client to integrate with LENS
infrastructure. That creates vendor lock-in, implementation friction, and maintenance
surface area. It also means LENS only governs sessions where the middleware is deployed.

A protocol specification governs any session where the implementing AI has loaded the
LENS canonical reference — regardless of provider, client, or infrastructure. That is
a broader governance surface, not a narrower one.

The enterprise framing is accurate: organizations do not want AI governed only where a
specific middleware product is deployed. They want AI that behaves correctly everywhere
it is used. Protocol specifications achieve that. Middleware products do not.

### Relationship to AegisRelay

This decision explicitly positions AegisRelay as the natural runtime enforcement layer
for LENS constraints when enforcement infrastructure is warranted. AegisRelay intercepts
AI calls. LENS defines what those calls should look like and what compliant outputs contain.
Together they form a governed multi-model execution architecture. Separately, they are
both valid: LENS without AegisRelay is still a governance protocol any AI client can adopt;
AegisRelay without LENS is a routing layer without a constraint specification.

The integration of LENS constraint evaluation into AegisRelay is a separate backlog item.
This ADR does not block or require it.

### Consequences
- The "protocol not middleware" positioning is explicit and documented — not an apparent
  oversight in the absence of enforcement machinery
- Charter Section 11 (Enforcement Model) added to define compliance responsibility
  and violation semantics at the protocol level
- Functional Spec Section 7 (Compliance Framework) added to establish measurable
  compliance criteria per behavior
- Option A (middleware evolution) remains a documented path — it is the correct choice
  if LENS is ever developed as a standalone enterprise product; that decision requires
  a new ADR
- AegisRelay is identified as the natural enforcement layer for teams that want runtime
  constraint evaluation on top of the LENS protocol

---

## Note on [WHY] Prompt Component

The addition of [WHY] as the second component of the block-based prompt structure is a
prompting standard, not an architectural decision. It does not meet the threshold for an
ADR. The decision and rationale are documented in Functional Spec Section 5.2.

---

---

## ADR-009: AegisRelay as Runtime Enforcement Layer

**Date:** March 22, 2026
**Status:** Accepted
**Decided by:** Jonathan Openshaw
**Related:** AegisRelay ADR-009 (mirror decision from AegisRelay perspective)
**Follows:** ADR-008 (Protocol Positioning)

### Context

ADR-008 established that LENS is a protocol specification, not middleware, and explicitly
named AegisRelay as "the natural runtime enforcement layer for LENS constraints when
enforcement infrastructure is warranted." Charter Section 11 defined the enforcement chain:

```
LENS Protocol (specifies) → AI Client (implements) → Session behavior (enforces)
```

And noted that if AegisRelay is deployed, it becomes an additional enforcement agent.

What neither document defined was the precise integration architecture — how AegisRelay
implements LENS constraints, where each behavior maps in AegisRelay's call flow, and what
changes (if anything) in the LENS protocol as a result of having a runtime enforcement
layer available.

This ADR closes that gap from LENS's perspective.

### Decision

AegisRelay implements LENS via two hooks that bracket the provider call, plus two existing
pipeline stage assignments. The full mapping is defined in AegisRelay ADR-009. From LENS's
perspective, three things are now formally true:

**1. The enforcement chain has a named concrete implementation.**

The enforcement chain from Charter Section 11 is updated to reflect the AegisRelay
integration:

```
LENS Protocol (specifies)
        ↓
AegisRelay governance hooks (enforce at relay boundary)
        ↓
AI Client canonical reference (enforce in conversational sessions)
        ↓
Session behavior (observable output)
```

Both enforcement paths are valid and complementary. AegisRelay governs provider calls.
AI client canonical reference governs conversational sessions. Neither replaces the other.

**2. The six behaviors have a defined runtime home in AegisRelay.**

| LENS Behavior | AegisRelay Implementation |
|---------------|---------------------------|
| Decision Checkpoints | Pre-call hook — fires before provider invocation |
| Assumption Surfacing | Pre-call hook — fires before provider invocation |
| Prompt Reflection | Post-call hook — fires after response received |
| Reframe Offers | Post-call hook — fires after response received |
| Uncertainty Flagging | Pipeline Stage 4 (Mark Uncertainty) |
| Cognitive Model Disclosure | Pipeline Stage 3 (Classify) |

**3. The machine-readable constraint schema is the integration contract.**

The JSON constraint schema from Functional Spec Section 7.5 is the specification
AegisRelay's hooks implement. It is not aspirational — it is the v1 integration contract.

### What Does Not Change in LENS

The protocol itself is unchanged by this ADR. All six behaviors, their trigger conditions,
violation definitions, and compliance rubric (Functional Spec Section 7.2) remain as
specified. AegisRelay implements LENS; it does not modify it.

The learning loop (LENS-correction, LENS-pattern thoughts in OpenBrain) remains active
for conversational sessions regardless of AegisRelay deployment. AegisRelay adds a second
enforcement surface; it does not replace the existing one.

### v1 Enforcement Boundary — Updated

Charter Section 11.3 stated that "violation detection is retrospective and user-driven
at v1.0." This remains true for conversational sessions. For AegisRelay-mediated sessions,
the enforcement boundary at v1 is:

**Observational enforcement:** AegisRelay pre-call and post-call hooks fire, evaluate
against LENS trigger conditions, and write observations to `governance_events`. No
blocking, no automated violation detection, no LENS-correction thoughts written
automatically. Human operator reviews governance_events to identify violations.

**v2 enforcement boundary (defined, not yet implemented):**
- Pre-call hook becomes a blocking gate for Decision Checkpoint events
- Post-call hook evaluates against the full compliance rubric and writes
  LENS-correction thoughts automatically tagged `[source:automated]`
- Per-session LENS compliance score produced from governance_events analysis

The v2 boundary directly addresses the gap Gee identified in the March 2026 pressure test:
"What happens when a model violates LENS in production?" At v2, the answer is:
"AegisRelay detects it, records it, and surfaces it — automatically."

### OpenBrain Sync Implications

AegisRelay's governance_events rows for LENS observations are not automatically synced to
OpenBrain at v1. OpenBrain remains the repository for conversational session LENS captures
(LENS-correction, LENS-pattern thoughts).

At v2, high-confidence LENS violation detections from AegisRelay may be synced to OpenBrain
as LENS-correction thoughts tagged `[source:automated] [source:aegisrelay]`, enabling
unified pattern analysis across both enforcement surfaces. This requires AegisRelay's
optional OpenBrain sync module (ADR-002) to be extended with a LENS-specific export filter.

### Consequences

- The LENS enforcement chain is concrete, not theoretical — it names a specific
  implementation (AegisRelay hooks) alongside the existing conversational enforcement path
- The six behaviors have a documented runtime home for AegisRelay-mediated sessions
- v1 enforcement is observational — the protocol's existing learning loop handles detection
- v2 path is defined and traceable: blocking gate, automated violation detection,
  cross-surface pattern analysis via OpenBrain sync
- LENS portability is preserved — this ADR defines AegisRelay as one implementation;
  other systems can implement LENS independently using the same constraint schema

---

*LENS ADR Log v1.0 — Ratified March 16, 2026*
*ADR-008 added March 22, 2026*
*ADR-009 added March 22, 2026*
*Next document: Definition of Done v1.0*
