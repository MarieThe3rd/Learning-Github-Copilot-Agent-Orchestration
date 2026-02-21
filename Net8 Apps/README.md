# .NET 8 Application Modernization — Agent Orchestration System

## Who This Is For

This folder contains the agent orchestration system for modernizing **.NET 8 console applications** and **.NET 8 Blazor applications**. These apps are already on modern .NET, so the problem is different from a Web Forms migration — the goal here is **architecture, test coverage, and clean code**, not framework migration.

If you are migrating a legacy ASP.NET Web Forms (.NET 4.8) application, use the [`Net48 WebForms`](../Net48%20WebForms) folder instead.

---

## The Problem

| Challenge      | Console Apps                                         | Blazor Apps                                           |
| -------------- | ---------------------------------------------------- | ----------------------------------------------------- |
| Framework      | .NET 8 — modern, but may not use it well             | .NET 8 — components exist, but may be monolithic      |
| Test coverage  | Often zero or near-zero                              | Often zero — components untested                      |
| Code style     | Dead code, missing DI, tight coupling in places      | Code-behind heavy components, mixed concerns          |
| Business rules | Implicit in Program.cs, services, workers            | Scattered across components and services              |
| Target         | Clean architecture, full test coverage, maintainable | Proper component design, bUnit-tested, VSA-structured |

---

## How This Differs From the Web Forms System

| Aspect            | Web Forms (.NET 4.8)                             | .NET 8 Apps                                                                             |
| ----------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| Phase 1 goal      | Introduce seams to break HttpContext/DB coupling | Introduce seams for missing DI, concrete deps, static helpers; plus codebase assessment |
| DI container      | Must be retrofitted                              | Already available — check it is used consistently                                       |
| Test framework    | Characterization tests (pin current behavior)    | Unit + integration tests (verify intended behavior)                                     |
| Primary concern   | Making code testable at all                      | Making code well-tested and well-structured                                             |
| Seam introduction | Mandatory for almost every class                 | Only where coupling is still present                                                    |
| Component model   | Web Forms pages (hard to test)                   | Blazor components (testable via bUnit)                                                  |

---

## Three Phases

| Phase                            | Goal                                                                                  | Gate                                                                            |
| -------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Phase 1** — Code Assessment    | Inventory the codebase, remove dead code, enforce code style, map coupling            | Inventory complete, build clean, dead code removed, style violations catalogued |
| **Phase 2** — Test Coverage      | Write unit, integration, and component tests; document business rules                 | Coverage ≥ 80% on business logic; Business Rule Catalogue approved              |
| **Phase 3** — Clean Architecture | Enforce Vertical Slice Architecture, SOLID, complete code quality; upgrade to .NET 10 | Architecture conformance tests pass; CI/CD green; targeting .NET 10 LTS         |

---

## Agent Map

### Agents in This Folder

| Agent                        | File                                                                         | Purpose                                                         |
| ---------------------------- | ---------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Orchestrator (.NET 8)        | [`orchestrator.md`](orchestrator.md)                                         | Adapted master controller for .NET 8 context                    |
| Legacy Code Expert (.NET 8)  | [`agents/legacy-code-expert-agent.md`](agents/legacy-code-expert-agent.md)   | Seam introduction for missing DI, concrete deps, static helpers |
| Microsoft Practices (.NET 8) | [`agents/microsoft-practices-agent.md`](agents/microsoft-practices-agent.md) | .NET 8/10 patterns — no migration table, Serilog + logz.io      |
| Blazor Expert                | [`agents/blazor-expert-agent.md`](agents/blazor-expert-agent.md)             | Blazor component architecture, state, bUnit                     |
| Console App Expert           | [`agents/console-app-expert-agent.md`](agents/console-app-expert-agent.md)   | Generic Host, Worker Services, integration testing              |

### Shared Agents (framework-agnostic)

These agents are shared across all orchestration systems and are used unchanged:

| Agent              | File                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| Refactoring Expert | [`../Shared Agents/refactoring-expert-agent.md`](../Shared%20Agents/refactoring-expert-agent.md) |
| Clean Code Expert  | [`../Shared Agents/clean-code-expert-agent.md`](../Shared%20Agents/clean-code-expert-agent.md)   |
| Architect          | [`../Shared Agents/architect-agent.md`](../Shared%20Agents/architect-agent.md)                   |
| Product Expert     | [`../Shared Agents/product-expert-agent.md`](../Shared%20Agents/product-expert-agent.md)         |
| QA Agent           | [`../Shared Agents/qa-agent.md`](../Shared%20Agents/qa-agent.md)                                 |
| Code Historian     | [`../Shared Agents/code-historian-agent.md`](../Shared%20Agents/code-historian-agent.md)         |
| Documentation      | [`../Shared Agents/documentation-agent.md`](../Shared%20Agents/documentation-agent.md)           |

---

## Shared Process Documents

Both the Net48 WebForms and .NET 8 Apps systems follow the same peer review and agent communication protocols:

- **Peer review rules:** [`../Shared Agents/peer-review-protocol.md`](../Shared%20Agents/peer-review-protocol.md)
- **Agent communication model:** [`../Shared Agents/workflow.md`](../Shared%20Agents/workflow.md)

---

## Where to Start

Read [`getting-started.md`](getting-started.md) first.
