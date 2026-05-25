# Dynamics 365 Agentic Claims Orchestration — Architecture Explained

## Overview

This project implements an **agentic AI workflow for insurance claim processing** inside Microsoft Dynamics 365. When a new claim arrives as a Case record, an AI agent reads the claim details, performs three distinct reasoning steps — coverage validation, fraud detection, and customer communication drafting — and writes its analysis directly back to the Case. The flow then routes the Case to one of three classification outcomes based on the agent's decision: auto-approved, pending approval, or needs human review.

The architecture is designed around five principles: **event-driven, intelligent, autonomous, governed, and human-in-the-loop**. Each principle maps to a specific design decision visible in the diagram, and each is explained below.

---

## 1. Claim Created — The Entry Point

The workflow starts where every insurance claim starts: a Case record being created in Dynamics 365 Customer Service. The diagram shows multiple incoming channels — web, email, phone, chat — because in a real Dynamics 365 Customer Service deployment, Cases can be created from any of these sources through standard D365 Channel Integration, omnichannel routing, or email-to-Case automation.

What matters for this architecture is not *how* the Case arrives, but *what it contains*. The Case record carries the structured claim data the downstream agent needs to reason about:

- **Claim Type** — the line of insurance (motor, home, health, etc.)
- **Claim Amount** — the monetary value being claimed
- **Claim Summary** — the free-text description of the incident
- Other contextual fields — customer, priority, severity, account

The Case is the single source of truth. Everything downstream — the AI's reasoning, the decision, the writeback — anchors to this record. This is deliberate: it means the AI's work is always attached to the customer-facing record, never living in a parallel system the business can't see.

---

## 2. Event Trigger — Reactive, Not Batch

The moment a Case is created, a Dataverse event fires. A Power Automate cloud flow subscribed to the `incident` table with a **"When a row is added"** trigger picks up this event in near-real-time.

This is a critical architectural choice. The system is **event-driven, not batch-driven**. Claims are processed within seconds of creation, not in a nightly job. This matters for three reasons:

- **Customer experience.** A claim acknowledged within seconds feels responsive. A claim queued for batch processing feels neglected.
- **Operational responsiveness.** High-risk claims surface immediately, not after a 24-hour delay where evidence may degrade or escalation windows may close.
- **Scalability.** Event-driven architectures scale horizontally — Dataverse webhook subscriptions handle volume by distributing work across flow runs, not by sequential batch processing.

The trigger is scoped to **Organization**, meaning any Case created anywhere in the org wakes the flow up. The change type is **Added** only — deliberately avoiding "Modified" to prevent the flow from triggering itself when it writes its analysis back to the Case (a common infinite-loop trap in Dataverse automation).

---

## 3. Agent Flow Orchestration — The Reasoning Layer

This is the heart of the architecture and the place where most insurance automation has historically stopped at rules engines. Instead of hard-coded business logic ("if claim amount > X, route to queue Y"), the flow invokes an AI agent that reasons about the claim holistically.

The agent invoked is the **Insurance Operations Orchestrator** — a Copilot Studio agent configured with structured output (the "Run an agent" action in Power Automate). It receives the claim data as input and returns six structured fields, one structured response per Case.

The agent's role, as the diagram describes, is to:

- **Understand claim context** — parse the type, amount, and incident description
- **Route tasks to the right specialist reasoning** — internally walk through coverage, fraud, and communication considerations in sequence
- **Synthesize outputs** into a single structured response
- **Determine next best action** — set the approval recommendation flag
- **Ensure explainability and governance** — return a human-readable recommendation alongside structured signals

The orchestrator is designed so that its instructions can be extended or specialist agents can be split out as the system matures. In the current implementation, the three specialist reasoning steps run inside a single agent invocation; as the architecture evolves, each can be promoted to a separately addressable Copilot Studio agent with dedicated knowledge sources and tools.

---

## 4. Specialist Reasoning Roles — Three Concerns, One Pass

The diagram shows three specialist agents because the **reasoning pattern** the orchestrator follows is genuinely three-part. Each role has a distinct concern, and treating them as distinct in design is what makes the system extensible. In the current POC, all three roles execute inside the orchestrator's single reasoning pass; the design contract for each is described below.

### Policy Validation

**Role:** assess whether the claim type, amount, and circumstances appear consistent with typical coverage for that line of insurance.

**Outputs:** the `coveragestatus` field, returned as one of three values — *Likely Covered*, *Likely Not Covered*, or *Unclear*. The recommendation field captures the reasoning, including any policy-side gaps (missing documentation references, ambiguous incident location, etc.).

**Honest scope:** without a customer's actual policy schedule attached as a knowledge source, this role provides a triage-level read, not a binding coverage decision. The output is intentionally probabilistic ("Likely") and explicitly defers final coverage adjudication to a claims handler with policy access.

### Fraud Detection

**Role:** evaluate the claim against known fraud indicators — policy timing, claim value relative to type, descriptive consistency, suspicious patterns — and assign a calibrated fraud score.

**Outputs:** `fraudscore` (integer 0–100, with explicit calibration bands), `risklevel` (Low/Medium/High aligned to the score), and a contribution to the overall approval recommendation. The agent is instructed to default to caution: when in doubt, score higher and flag for review.

**Honest scope:** without a historical fraud database or rule engine attached, scores reflect pattern-matching on the claim text against general fraud heuristics rather than statistical risk modelling. The system architecture allows for this to be hardened later by attaching real fraud rule sources.

### Communication Drafting

**Role:** produce a professional, jurisdictionally appropriate customer-facing message acknowledging the claim and conveying its current status — without making promises about approval, payout, or liability.

**Outputs:** `customercommunication`, a 3–5 sentence draft message tailored to the decision the agent just made. Draft is written for handler review before sending, never auto-sent to the customer.

**Honest scope:** the draft is reviewed by a human before reaching the customer. It is positioned as a productivity aid (give the handler a clean starting point), not as autonomous customer outreach.

The reason these three roles are separable in design even when collapsed in implementation: each one has different requirements as the system matures. Policy Validation needs policy documents and product schedules. Fraud Detection needs historical claim data and fraud rules. Communication needs jurisdictional templates and brand voice guidelines. Treating them as distinct concerns now means each can be hardened independently later.

---

## 5. Decision Engine — Deterministic Logic on Top of Probabilistic Reasoning

This is the architectural seam where the AI's probabilistic output meets the system's deterministic requirements. Insurance operations cannot run on "the AI said maybe" — there has to be a clear, auditable rule that converts the agent's structured signals into a specific business outcome.

The Decision Engine sits between the agent's output and the Case writeback. It performs three functions:

1. **Evaluates the agent's structured output** — reads `approvalrequired`, `fraudscore`, and other signals
2. **Applies business rules** — currently a routing rule keyed on `approvalrequired` (boolean), with explicit defaults for null or unexpected values
3. **Determines outcome** — selects one of three Status Reason values to write to the Case

This separation is deliberate and important. The AI is permitted to **recommend**, but only the Decision Engine is permitted to **route**. This means:

- Business rule changes (raising the auto-approval threshold, requiring human review for any claim above €X, etc.) happen in the flow, not in the agent's prompt. They are auditable, version-controlled, and modifiable without re-prompting the AI.
- When the AI behaves unexpectedly — returns null, drifts from the schema, contradicts itself — the Decision Engine catches it with safe defaults that route to human review rather than auto-approval.
- Compliance and audit conversations have a clear answer to "why was this claim routed this way?" — it is the rule, written here, that decided. The AI provided inputs to a rule, it didn't make the decision unilaterally.

In the current implementation, the engine is a Power Automate Switch action with three branches and a fail-safe default. The architecture allows for this engine to grow into multi-factor logic (combining fraud score thresholds, claim amount caps, and coverage status) as business policy is formalized.

---

## 6. Case Classification — Three Operational Outcomes

The Decision Engine produces one of three Status Reason values written to the Case. These are not cosmetic labels — they drive downstream operational behavior in D365 Customer Service: queue routing, SLA timers, agent assignments, escalation rules, and reporting.

### AI Reviewed — Auto Approved (green)

The agent assessed the claim as low-risk, within coverage parameters, and within the auto-approval threshold. The Case is flagged for fast-track processing. A human still has final sign-off; "auto-approved" means "the AI sees no reason to escalate," not "the AI authorized payment."

### AI Reviewed — Pending Approval (amber)

The agent determined the claim needs human approval before proceeding. This is the expected default for most claims — moderate amounts, partial information, or any factor outside the auto-approval criteria. The Case routes to the standard approval queue.

### AI Reviewed — Needs Human Review (red)

The agent flagged something that requires investigation beyond standard approval — high fraud score, large claim value, ambiguous coverage, contradictory information, or any signal the Decision Engine treats as a fail-safe condition. The Case routes to a senior review queue.

The three-state classification is intentionally simple. Adding more states proliferates operational complexity without adding decision value. Each state corresponds to a clear next action and a clear queue.

---

## 7. Update Case — Writeback as the Audit Trail

The final step writes the agent's analysis back to the Case record. This is not optional infrastructure — it is the single most important step for governance.

The fields updated:

- **Fraud Score** — the calibrated 0–100 score
- **Coverage Status** — the enumerated coverage read
- **AI Recommendation** — the agent's reasoning in natural language
- **Customer Communication** — the draft message for handler review
- **Status Reason** — the routing decision (one of the three classifications)
- **Case Status** — left as Active (the AI does not close Cases)

This writeback design serves three functions:

1. **Visibility.** A claims handler opening the Case sees the AI's full reasoning immediately, in the same form they already work in. No separate AI dashboard, no parallel system.
2. **Audit trail.** Every AI-assisted Case carries the AI's outputs as Case data. When compliance asks "what did the AI say and what did the system decide?", the answer is in the Case record itself.
3. **Reversibility.** A handler who disagrees with the AI can overwrite any field. The AI's contribution is a recommendation, not a lock.

The writeback explicitly does not modify the parent `statecode` (Active/Resolved/Cancelled). Lifecycle state remains under human control. Only the Status Reason (substate) is set by the flow.

---

## Cross-Cutting Capabilities

These five capabilities are architectural properties, not separate components — they emerge from how the system is built.

### Security & Compliance

Built on Microsoft Dataverse, the architecture inherits its security model: role-based access control on the Case table, field-level security for sensitive columns, comprehensive audit logging of every Case modification, and tenant-level data residency controls. The AI doesn't bypass Dataverse security — it operates within it, as a service principal with explicit permissions.

### Explainability

Every agent invocation produces a natural-language recommendation (`airecommendation`) alongside the structured outputs. Every Case carries this reasoning in its own field. Every Decision Engine routing is deterministic and inspectable in the flow run history. A claim's full AI-assisted journey is reconstructable from the Case record alone.

### Scalability

The event-driven architecture scales horizontally. Dataverse webhook subscriptions handle parallel Case creation by spinning up parallel flow runs. The agent invocation is stateless — each Case is reasoned about independently, no cross-Case context, no race conditions. Bottlenecks, if they appear, will be in agent throughput (API rate limits), not in the orchestration layer.

### Human-in-the-Loop

The architecture treats human oversight as the rule, not the exception. The "Request human assistance when unsure" flag is enabled on the agent. Two of the three classification outcomes route to human queues. Customer communications are drafted, never auto-sent. Status (lifecycle) changes require human action. The AI accelerates handler work — it does not replace it.

### Extensibility

The single-agent implementation is the starting point. The architecture is designed so that:

- Specialist reasoning roles can be promoted to separate Copilot Studio agents with dedicated knowledge sources
- The Decision Engine can grow from single-factor (boolean) to multi-factor (compound conditions) without restructuring
- New classification outcomes can be added by extending the Status Reason choice set and adding Switch branches
- Additional channels (email-to-Case, omnichannel) can feed into the same trigger without modification

---

## End-to-End Flow Summary

The bottom bar of the diagram captures the journey in six steps:

1. **Claim Created** — Case record arrives in Dynamics 365 from any channel
2. **Event Triggered** — Dataverse webhook fires Power Automate flow on row added
3. **AI Orchestration** — Insurance Operations Orchestrator reasons through coverage, fraud, and communication concerns
4. **Decision Made** — Decision Engine applies business rules and selects routing outcome
5. **Case Updated** — Agent outputs and routing decision written back to Case record
6. **Better Outcomes** — Handler picks up Case with AI analysis already attached, makes final decision faster and with more context

Total latency from Case creation to fully-updated Case with AI analysis: typically under 30 seconds.

---

## What This Is, and What It Isn't

This is a working POC demonstrating how Microsoft's agentic AI primitives (Copilot Studio agents, the "Run an agent" Power Automate action, Dataverse webhook triggers) can be composed into an operational claims processing workflow.

It is **not** a production claims system. Specifically:

- The agent's reasoning is currently grounded only in its prompt instructions, not in attached policy documents, historical claim data, or fraud rule databases. Hardening this is the most important next step for any production deployment.
- The Decision Engine currently uses single-factor routing (approval required boolean). Production deployments would require multi-factor business rules involving claim amount, fraud score thresholds, and coverage status.
- Human-in-the-loop routing is configured but not connected to specific queues, teams, or escalation paths. Production deployments would wire these to actual claims operations structures.

The architecture is designed to absorb each of these hardenings without restructuring. The POC proves the wiring works and the pattern is sound. Production-readiness is a matter of grounding, not redesign.

---

## Repository

This README accompanies the [Dynamics-365-Agentic-Claims-Orchestration](https://github.com/Pradyumna-yes/Dynamics-365-Agentic-Claims-Orchestration) repository, which contains the full Power Automate flow definitions, agent prompts, Dataverse schema additions, and architecture diagrams.

---

## Quick Start

### Prerequisites

- Dynamics 365 Customer Service environment (version 9.0 or later)
- Power Automate with cloud flow capabilities
- Copilot Studio access
- Appropriate permissions to create flows, agents, and custom columns

### Setup Steps

1. **Clone or download the repository** to your local machine
2. **Import the Power Automate flow** into your D365 environment
3. **Configure the Copilot Studio agent** with the provided prompt templates
4. **Add custom columns** to the incident (Case) table for fraud score, coverage status, etc.
5. **Activate the flow** and test with sample claims

Detailed setup instructions are in the `/setup` directory.

---

## Key Files

- `/flows` — Power Automate cloud flow definitions (.json)
- `/agents` — Copilot Studio agent instructions and schemas
- `/schemas` — Dataverse custom column definitions
- `/diagrams` — Architecture and flow diagrams (PNG, SVG)
- `ARCHITECTURE.md` — This file
- `SETUP.md` — Detailed setup and configuration guide

---

## Next Steps / Future Work

- Attach real policy documents as knowledge sources to the Policy Validation agent
- Integrate with a fraud rule database or historical claims API
- Wire the three specialist roles into separate Copilot Studio agents (true multi-agent orchestration)
- Extend the Decision Engine with multi-factor routing (fraud thresholds + claim amount caps + coverage)
- Connect human review queues to Microsoft Teams or assignment rules
- Add monitoring and telemetry for agent performance tracking

---

## License

This project is provided as-is for learning and POC purposes.

---

## Contact

For questions or feedback on this architecture, please open an issue in the repository.