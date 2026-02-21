# Workflow Definition

## Overview

This document describes how the specialist agents and the Orchestrator interact: the message protocols, decision trees, escalation paths, and the overall flow from initial analysis to final delivery.

> **Note:** The phase-specific workflows below were authored for the **Net48 WebForms** migration system. The **.NET 8 Apps** system follows the same communication patterns — all messages route through the Orchestrator, the same message types apply, and the same peer review protocol is enforced — but with different phase activities. Refer to the phase documents in [`../Net8 Apps/phases/`](../Net8%20Apps/phases/) for the .NET 8–specific phase flows.

---

## Agent Communication Model

All agent communication routes through the **Orchestrator**. Agents do not directly instruct each other — they make requests and the Orchestrator routes them.

```
┌────────────────────────────────────────────────────────────────────────┐
│                           ORCHESTRATOR                                  │
│                  (All messages route through here)                      │
└─┬──────────┬──────────┬──────────┬───────────┬──────────┬──────────┘
  │          │          │          │           │          │
  ▼          ▼          ▼          ▼           ▼          ▼
Legacy    Refactor   Clean     Microsoft  Product     QA
Code      Expert     Code       Agent     Expert     Agent
Expert     Agent    Expert               Agent
Agent
             │                                  │
             ▼                                  ▼
          Architect                         Code Historian
            Agent                               Agent
        (VSA / KISS / YAGNI)             (Chronicle & BR guard)

                             ▼
                       Documentation
                           Agent
                    (/docs + ADO wiki)
```

### Message Types

| Type               | Description                                 | Example                                                                            |
| ------------------ | ------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Task**           | Orchestrator assigns work to an agent       | "Legacy Code Expert Agent: analyze CustomerOrderProcessor for seams"               |
| **Review Request** | Agent requests another agent's review       | "Legacy Code Expert → Orchestrator: please ask Refactoring Expert to review"       |
| **Finding**        | Agent reports an observation                | "Product Expert: BR-012 confirmed, test needed for edge case"                      |
| **Question**       | Agent cannot proceed without information    | "Product Expert: is the $200 discount threshold intentional?"                      |
| **Escalation**     | Agent flags a decision beyond its authority | "Legacy Code Expert: found code called only via reflection — cannot safely remove" |
| **Approval**       | Agent approves a proposed action            | "Refactoring Expert: change is behavior-preserving — approved"                     |
| **Block**          | Agent prevents progress until resolved      | "Product Expert: cannot approve Phase 2 gate — 3 rules unresolved"                 |
| **Sign-Off**       | Agent formally approves a phase gate        | "QA Agent: all P1/P2 tests passing — Phase 3 sign-off granted"                     |
| **Chronicle**      | Code Historian records a completed change   | "Code Historian: CHR-042 logged — ready to commit"                                 |

---

## Phase 1 Workflow

```
START
  │
  ▼
Orchestrator: Initiate Phase 1
  │
  ▼
Legacy Code Expert Agent: Build codebase inventory
  │  Produces: inventory.md, dependency-map.md
  │
  ▼
Orchestrator: Review inventory; confirm completeness
  │
  ▼
──── FOR EACH CLASS (in priority order) ────────────────────────────────────
│                                                                           │
│  Legacy Code Expert Agent: Analyse class dependencies                               │
│    │                                                                      │
│    ▼                                                                      │
│  Legacy Code Expert Agent: Design seams and interfaces                              │
│    │                                                                      │
│    ▼                                                                      │
│  Legacy Code Expert Agent: Implement change (smallest safe step)                    │
│    │                                                                      │
│    ▼                                                                      │
│  Refactoring Expert Agent: Review change for behavior preservation                    │
│    │                                                                      │
│    ├─ APPROVED → Feathers commits; updates seam-registry.json             │
│    │                                                                      │
│    └─ REJECTED → Feathers revises or escalates to Orchestrator            │
│                                                                           │
│  Clean Code Expert Agent: Review new interface/class names                           │
│    │                                                                      │
│    └─ Suggest renames if needed (Feathers applies)                        │
│                                                                           │
│  Microsoft Agent: Flag any .NET 4.8-only APIs for future migration risk   │
│                                                                           │
│  Orchestrator: Update class status → "Phase 1 Complete"                   │
│                                                                           │
──── END FOR EACH CLASS ────────────────────────────────────────────────────
  │
  ▼
Orchestrator: Run Phase 1 → Phase 2 gate checklist
  │
  ├─ ALL PASSED → Advance to Phase 2
  │
  └─ GAPS FOUND → Assign remaining work; recheck gate
```

---

## Phase 2 Workflow

```
START Phase 2
  │
  ▼
QA Agent: Begin workflow mapping from .aspx pages (parallel with testing)
  │
  ▼
──── FOR EACH CLASS (in coverage priority order) ────────────────────────────
│                                                                            │
│  Legacy Code Expert Agent: Exercise class with fakes; observe all behaviors │
│    │                                                                        │
│    ▼                                                                        │
│  Legacy Code Expert Agent: Write ONE characterization test (BDD naming)    │
│    │                                                                        │
│    ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  PEER REVIEW (all coding agents)                                    │   │
│  │  Refactoring Expert: test does not touch production code?           │   │
│  │  Clean Code Expert: BDD naming + BR-NNN referenced?                 │   │
│  │  Microsoft Agent: xUnit pattern correct?                            │   │
│  │  Architect Agent: test in right project/folder?                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│    │                                                                        │
│    ├─ Any ⚠️  / ❌ → resolve per peer-review-protocol.md                   │
│    ├─ All ✅ → Code Historian logs CHR-NNN entry                           │
│    │                                                                        │
│    ▼                                                                        │
│  COMMIT: git commit "[PHASE-2] [CHR-NNN] {description}"                   │
│  Build green. All tests pass. One test per commit.                         │
│                                                                             │
│  Legacy Code Expert Agent → Orchestrator: Submit inferred business rules   │
│    │                                                                        │
│    ▼                                                                        │
│  Product Expert Agent: Classify each rule                                  │
│    │                                                                        │
│    ├─ CONFIRMED → Record in catalogue; ensure test exists                  │
│    │                                                                        │
│    ├─ SUSPECTED → Escalate to Orchestrator → Human                         │
│    │    Human decides: Confirmed / Bug / Accepted Bug                      │
│    │                                                                        │
│    └─ DISPUTED → Escalate to Orchestrator → Human                         │
│         Human decides: which behavior is authoritative                     │
│                                                                             │
│  Legacy Code Expert Agent: Add edge case tests (one per commit)            │
│  Orchestrator: Update class coverage status                                │
│                                                                             │
──── END FOR EACH CLASS ─────────────────────────────────────────────────────
  │
  ▼ (in parallel)
QA Agent: Produce draft E2E test cases from workflow maps + business rules
  │
  ▼
Product Expert Agent: Final catalogue review — no "Unknown" items remain
  │
  ▼
Orchestrator: Run Phase 2 → Phase 3 gate checklist
  │
  ├─ ALL PASSED → Product Expert Agent signs off; QA Agent delivers draft suite
  │               Advance to Phase 3
  │
  └─ GAPS FOUND → Assign remaining work; recheck gate
```

---

## Phase 3 Workflow

```
START Phase 3
  │
  ▼
Architect Agent: Produce solution structure doc (VSA folder layout, Shared kernel rules)
  │  Produces: architecture-decision.md (ARCH-001, ARCH-002, ARCH-003 logged)
  │  Orchestrator approves
  │
  ▼
Microsoft Agent + Clean Code Expert Agent: Align on .NET 10 platform decisions
  │  (EF Core config, DI registration pattern, security strategy)
  │
  ▼
──── FOR EACH FEATURE SLICE (in dependency order) ───────────────────────────
│                                                                            │
│  Refactoring Expert Agent: Analyse current code for smells; produce refactoring plan  │
│    │                                                                       │
│    ▼                                                                       │
│  Clean Code Expert Agent: Review target design — SOLID, KISS, naming, no over-eng   │
│    │                                                                       │
│    ▼                                                                       │
│  Architect Agent: Confirm slice folder layout and shared kernel usage      │
│    │  (Right place? YAGNI violations? Cross-slice imports?)                │
│    │                                                                       │
│    ▼                                                                       │
│  Refactoring Expert Agent: Apply ONE refactoring step                    │
│    │ Run characterization tests — must still pass                         │
│    │                                                                       │
│    ▼                                                                       │
│  ┌─────────────────────────────────────────────┐                        │
│  │  PEER REVIEW (all coding agents)                    │                        │
│  │  Legacy Code Expert: no legacy patterns remain?     │                        │
│  │  Clean Code Expert Agent: SOLID / KISS / naming?    │                        │
│  │  Microsoft Agent: DI + EF Core correct?             │                        │
│  │  Architect Agent: VSA placement + no cross-slice?   │                        │
│  └─────────────────────────────────────────────┘                        │
│    │                                                                       │
│    ├─ Any ⚠️ / ❌ → resolve per peer-review-protocol.md                     │
│    ├─ All ✅ → Code Historian logs CHR-NNN entry                           │
│    │                                                                       │
│    ▼                                                                       │
│  COMMIT: git commit “[PHASE-3] [CHR-NNN] {description}”                   │
│  Build green. All characterization tests pass.                            │
│    │                                                                       │
│    ▼ ← repeat for each refactoring step in this slice                     │
│                                                                            │
│  Microsoft Agent: Implement .NET 10 infrastructure for this slice         │
│    │ (EF Core DbContext direct, DI registration, endpoint)                │
│    │                                                                       │
│    ▼                                                                       │
│  Legacy Code Expert Agent: Review — no legacy patterns remain in this slice         │
│                                                                            │
│  Product Expert Agent: Verify all BR-NNN rules for this slice are met     │
│    │                                                                       │
│    ├─ CONFIRMED → Slice complete; move to next                             │
│    │                                                                       │
│    └─ RULE VIOLATED → Refactoring Expert/Microsoft fix; Product Expert re-reviews │
│                                                                            │
│  QA Agent: Execute P1/P2 test cases for this feature                      │
│    │                                                                       │
│    ├─ ALL PASS → Slice signed off                                          │
│    │                                                                       │
│    └─ FAILURES → Log defects; fix; re-execute                             │
│                                                                            │
│  Orchestrator: Update feature migration status                             │
│                                                                            │
──── END FOR EACH FEATURE SLICE ────────────────────────────────────────────
  │
  ▼
Microsoft Agent: Security checklist review
  │
  ▼
Architect Agent: Final architecture conformance tests — all green (NetArchTest)
  │
  ▼
Clean Code Expert Agent: Clean code review — no over-engineering, all SOLID rules
  │
  ▼
QA Agent: Full regression execution across all features
  │
  ▼
QA Agent: Formal UAT sign-off
  │
  ▼
Documentation Agent: Capture final code metrics; publish /docs to Azure DevOps wiki
  │  Writes: docs/metrics/final-metrics.md (delta vs baseline)
  │  Writes: docs/technical/*, docs/business-rules/*, docs/chronicle/*
  │
  ▼
Orchestrator: Run Phase 3 → Done gate checklist
  │
  ├─ ALL PASSED → Project complete
  │
  └─ GAPS FOUND → Assign remaining work; recheck gate
```

---

## Conflict Resolution Workflow

```
Agent A and Agent B disagree
  │
  ▼
Either agent escalates to Orchestrator with:
  - Their position (clearly stated)
  - Their rationale
  - The current phase context
  │
  ▼
Orchestrator applies decision hierarchy:
  1. Current phase safety rule wins (testability in P1, accuracy in P2, quality in P3)
  2. More conservative option wins if uncertain
  3. Escalate to human if:
     a. Both options are equally safe/unsafe
     b. Business intent is required
     c. A compliance/security concern exists
  │
  ├─ DECISION MADE → Document in decision log; notify both agents; proceed
  │
  └─ ESCALATED TO HUMAN → Orchestrator presents both positions clearly;
                           human decides; decision logged; proceed
```

---

## Escalation Matrix

| Trigger                                            | Agent That Escalates | Escalates To | Human Required?           |
| -------------------------------------------------- | -------------------- | ------------ | ------------------------- |
| Code called via reflection — cannot confirm unused | Feathers             | Orchestrator | Yes                       |
| Business rule cannot be inferred from code         | Product Expert       | Orchestrator | Yes                       |
| Two contradictory behaviors for same condition     | Product Expert       | Orchestrator | Yes                       |
| Schema change required to proceed                  | Microsoft            | Orchestrator | Yes                       |
| Legal/compliance implication                       | Any                  | Orchestrator | Yes                       |
| Agents disagree on safe refactoring                | Fowler/Feathers      | Orchestrator | No (Orchestrator decides) |
| Naming disagreement                                | Martin/Microsoft     | Orchestrator | No (Orchestrator decides) |
| P1 test case fails after Phase 3 change            | QA                   | Orchestrator | No (assigned to fix)      |
| Architecture test fails                            | Martin               | Orchestrator | No (assigned to fix)      |

---

## Progress Tracking

The Orchestrator maintains a living status document:

```markdown
## Modernization Progress

**Current Phase:** 2
**Phase 1 completion:** 2026-01-30
**Estimated Phase 2 completion:** 2026-02-28

### Phase 1 Summary

- Classes processed: 47 / 47
- Interfaces created: 23
- Static dependencies removed: 31
- Fakes created: 23

### Phase 2 Progress

- Classes characterised: 31 / 47
- Business rules confirmed: 84
- Business rules pending human review: 7
- Characterization tests written: 312
- Average coverage: 86%

### Phase 3 Status

- Not started

### Open Escalations

- ESC-007: Is the 28-day month assumption in BillingCalculator intentional? [AWAITING HUMAN]
- ESC-009: OrderHistory.GetByDateRange silently ignores invalid dates — bug or feature? [AWAITING HUMAN]
```

---

## Agent Interaction Constraints

| Rule                                                                          | Rationale                               |
| ----------------------------------------------------------------------------- | --------------------------------------- |
| No agent modifies production code without Orchestrator awareness              | Maintain audit trail                    |
| No agent advances a phase gate unilaterally                                   | Gates require Orchestrator sign-off     |
| No agent deletes code without Product Expert and Fowler both approving        | Deletions are irreversible              |
| No agent makes a database schema change without Orchestrator + human approval | Schema changes are high-risk            |
| Refactoring Expert Agent must review every code change in Phase 1             | Behavior preservation is non-negotiable |
| Product Expert Agent must review every business rule before it is "Confirmed" | Business accuracy is non-negotiable     |
| QA Agent must re-execute P1 tests after every Phase 3 feature slice           | Regression detection is non-negotiable  |
