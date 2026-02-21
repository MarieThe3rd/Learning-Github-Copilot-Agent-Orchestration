# Phase 1: Code Assessment & Stabilization

## Goal

Understand the codebase. Identify and remove dead code. Enforce code style rules. Map coupling hotspots. Produce a health report that drives Phase 2 priorities.

This phase does **not** add features or refactor architecture. The only permitted changes are:

- Removing provably dead code
- Fixing code style violations (regions, comments, naming)
- Introducing seams only where tight coupling would prevent testing at all

---

## Phase Gate

Phase 1 is complete when **all** of the following are true:

```markdown
- [ ] Full inventory of all classes, services, components, and hosted workers is complete
- [ ] Dependency graph is drawn — every significant coupling is mapped
- [ ] Dead code audit complete — all unused classes, methods, and parameters are removed or flagged for human decision
- [ ] No #region or #endregion in any C# file
- [ ] No commented-out code blocks anywhere in the codebase
- [ ] No TODO or FIXME comments — all converted to issue tracker items
- [ ] Build passes with zero errors and zero new warnings
- [ ] All existing tests (if any) are green
- [ ] Coupling hotspots report produced — classes that are difficult to test in isolation
- [ ] DI registration audit complete — all services registered correctly; no service-locator anti-patterns
- [ ] Configuration audit complete — all magic strings replaced with IOptions<T> or constants
- [ ] Orchestrator has reviewed and approved phase transition
```

---

## Agents Active in Phase 1

| Agent                                 | Role                                                                                         |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Legacy Code Expert Agent (.NET 8)** | Primary — identify coupling hotspots; design and introduce seams for all HIGH/CRITICAL items |
| **Blazor Expert Agent**               | Primary (Blazor apps) — inventory components, identify code-behind coupling, map state       |
| **Console App Expert Agent**          | Primary (console apps) — inventory services, workers, map Generic Host registration          |
| **Clean Code Expert Agent**           | Primary — identify and remove dead code; flag all code style violations                      |
| **Refactoring Expert Agent**          | Advisory — validate that seam introductions and dead code removals are behavior-preserving   |
| **Architect Agent**                   | Advisory — identify structural concerns that will affect Phase 3 slice boundaries            |
| **Microsoft Practices Agent**         | Advisory — flag non-idiomatic .NET 8 patterns (service-locator, manual DI, etc.)             |
| **Code Historian Agent**              | Records every CHR-NNN entry for approved changes                                             |
| **Orchestrator**                      | Tracks inventory progress; routes tasks; enforces gate                                       |

---

## Phase 1 Process

### Step 1 — Inventory

The appropriate expert agent (Blazor or Console) produces a structured inventory:

**Console App inventory format:**

```
SERVICE INVENTORY
  - OrderProcessingWorker (BackgroundService): 847 lines — coupling hotspot
  - OrderCalculationService: 203 lines — no interface, injected as concrete
  - EmailService: 89 lines — SmtpClient directly instantiated, not injected
  - Program.cs service registrations: 34 services registered

WORKERS
  - OrderProcessingWorker: runs every 60s, reads from SQL, sends email

CONFIGURATION
  - 12 magic strings found in appsettings.json key reads
  - 0 IOptions<T> usages found
```

**Blazor app inventory format:**

```
COMPONENT INVENTORY
  - Pages/Orders/OrderList.razor: 312 lines — code-behind heavy, 4 injected services
  - Pages/Orders/OrderDetail.razor: 189 lines — direct HttpClient calls in OnInitializedAsync
  - Shared/StatusBadge.razor: 23 lines — clean, reusable
  - Services/OrderService.cs: 445 lines — no interface, concrete dependency

STATE MANAGEMENT
  - No state management library detected — state passed via parameters and cascading values

JAVASCRIPT INTEROP
  - 3 JSRuntime.InvokeAsync calls found — no abstraction layer
```

### Step 2 — Dead Code Removal

Clean Code Expert Agent leads. For each dead code candidate:

1. Confirm zero callers (static analysis + grep)
2. Refactoring Expert Agent approves removal as safe
3. Remove and commit: `[PHASE-1] [CHR-NNN] Remove unused OrderHistoryArchiver class`

Dead code categories:

- Unused classes with zero references
- Unused private methods
- Unused method parameters
- Unreachable branches (`if (false)`, code after `return` in all paths)
- Commented-out code blocks

### Step 3 — Code Style Enforcement

Clean Code Expert Agent leads. One violation type per commit:

- Remove all `#region` / `#endregion` blocks
- Remove all comments that explain _what_ code does (extract named methods instead)
- Convert all TODO/FIXME to issue tracker items, then delete the comment

### Step 4 — Coupling Hotspot Report

Expert agent identifies classes that will be hardest to test. Report format:

```
COUPLING HOTSPOTS
  HIGH PRIORITY (must add seam before Phase 2 can test):
    - OrderProcessingWorker: directly instantiates EmailService and SqlConnection
      → Recommend: inject IEmailService and IOrderRepository

  MEDIUM PRIORITY (injectable but no interface):
    - OrderCalculationService: no IOrderCalculationService interface
      → Recommend: extract interface in Phase 2 setup

  LOW PRIORITY (already well-structured):
    - StatusBadge.razor: pure component, no service dependencies
```

---

## What Is NOT Permitted in Phase 1

| Not permitted                                         | Why                        |
| ----------------------------------------------------- | -------------------------- |
| Changing business logic                               | No tests yet — too risky   |
| Refactoring to Vertical Slice Architecture            | Phase 3 activity           |
| Adding new features                                   | Out of scope entirely      |
| Renaming public API surfaces                          | Breaking change risk       |
| Changing DI registrations (unless removing dead code) | May break runtime behavior |

---

## Seam Introduction

Seam introduction is a **standard Phase 1 activity** for .NET 8 apps, not an exception. After the coupling hotspot report is produced, the **Legacy Code Expert Agent (.NET 8)** leads seam work for all HIGH and CRITICAL priority hotspots. Phase 1 cannot be gated until these seams exist.

**Process for each HIGH/CRITICAL hotspot:**

1. Legacy Code Expert Agent identifies the coupling type (missing interface, static helper, newed-up dependency, direct `new HttpClient()`, etc.)
2. Legacy Code Expert Agent proposes the minimum interface and constructor change
3. Refactoring Expert Agent confirms the change is behavior-preserving
4. All coding agents approve per [`../../Shared Agents/peer-review-protocol.md`](../../Shared%20Agents/peer-review-protocol.md)
5. DI registration is updated in Program.cs
6. Change is committed: `[PHASE-1] [CHR-NNN] Extract IEmailService from OrderProcessingService`

MEDIUM priority hotspots (injectable but no interface) may be deferred to Phase 2 setup if the Orchestrator agrees. LOW priority items are never a Phase 1 gate blocker.

All seams introduced in Phase 1 are recorded with a `[SEAM]` tag in the CHR entry.
