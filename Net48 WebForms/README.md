# Legacy .NET Modernization — Agent Orchestration System

## Vision

Transform a legacy **ASP.NET Web Forms (.NET 4.8)** application with no tests into a well-architected **.NET 10** application through a fully orchestrated, multi-agent refactoring pipeline. Every step is safe, traceable, and governed by the collective wisdom of Fowler, Feathers, Martin, and Microsoft.

---

## The Problem

| Challenge      | Detail                                                                  |
| -------------- | ----------------------------------------------------------------------- |
| Framework      | ASP.NET Web Forms (.NET 4.8)                                            |
| Test coverage  | 0% — no unit, integration, or acceptance tests                          |
| Code style     | Tightly coupled, large code-behind files, static dependencies           |
| Business rules | Implicit — buried in event handlers, stored procedures, and God classes |
| Target         | .NET 10, clean architecture, fully tested, maintainable                 |

---

## The Solution: Nine-Agent Orchestration

A master **Orchestrator** delegates to nine specialized agents across three disciplined phases.

```
                                   ┌─────────────────────┐
                                   │     ORCHESTRATOR     │
                                   │   (Phase Controller) │
                                   └──────────┬──────────┘
                                              │
    ┌─────────────┬──────────────┬────────────┼────────────┬─────────────┬─────────────┐
    │             │              │            │            │             │             │
┌───▼───┐   ┌────▼────┐   ┌─────▼─────┐ ┌───▼───┐  ┌────▼───┐  ┌──────▼─────┐ ┌───▼────┐
│LEGACY │   │REFACTOR │   │CLEAN CODE │ │  MS   │  │PRODUCT │  │  ARCHITECT │ │  QA    │
│ CODE  │   │ EXPERT  │   │  EXPERT   │ │AGENT  │  │EXPERT  │  │   AGENT    │ │ AGENT  │
└───────┘   └─────────┘   └───────────┘ └───────┘  └────────┘  └────────────┘ └────────┘

                    ┌──────────────────┐        ┌──────────────────┐
                    │  CODE HISTORIAN  │        │  DOCUMENTATION   │
                    │     AGENT        │        │      AGENT       │
                    │ (Change audit &  │        │  (/docs + ADO    │
                    │  BR guardian)    │        │   wiki publish)  │
                    └──────────────────┘        └──────────────────┘
```

---

## Three Phases

| Phase                                                           | Goal                                                                          | Primary Agents                                          | Gate to Next Phase                                                           |
| --------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------- |
| [Phase 1: Safe Refactoring](phases/phase-1-safe-refactoring.md) | Make legacy code testable without changing behavior                           | Legacy Code Expert + Refactoring Expert                 | All classes have at least one seam; no static dependencies in business logic |
| [Phase 2: Test Coverage](phases/phase-2-test-coverage.md)       | Infer business rules, write characterization tests, achieve scenario coverage | Legacy Code Expert + Clean Code Expert + Product Expert | ≥ 90% business rule coverage; all edge cases documented; catalogue locked    |
| [Phase 3: Modernization](phases/phase-3-modernization.md)       | Refactor to clean .NET 10 architecture with confidence                        | Refactoring Expert + Clean Code Expert + Microsoft      | CI/CD green; architecture conformance tests pass                             |

---

## Agent Roster

| Agent                    | Authority                                                                  | Document                                                                                       |
| ------------------------ | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Orchestrator             | Phase control, conflict resolution, progress tracking                      | [orchestrator.md](orchestrator.md)                                                             |
| Architect Agent          | Vertical slice architecture, solution structure, KISS/YAGNI                | [../Shared Agents/architect-agent.md](../Shared%20Agents/architect-agent.md)                   |
| Legacy Code Expert Agent | Legacy code, seam detection, characterization tests                        | [agents/legacy-code-expert-agent.md](agents/legacy-code-expert-agent.md)                       |
| Refactoring Expert Agent | Refactoring catalog, behavior-preserving transformations                   | [../Shared Agents/refactoring-expert-agent.md](../Shared%20Agents/refactoring-expert-agent.md) |
| Clean Code Expert Agent  | Clean code, SOLID, KISS, naming, cohesion, no over-engineering             | [../Shared Agents/clean-code-expert-agent.md](../Shared%20Agents/clean-code-expert-agent.md)   |
| Microsoft Agent          | .NET 10, ASP.NET Core, EF Core, xUnit, MIT-licensed packages               | [agents/microsoft-practices-agent.md](agents/microsoft-practices-agent.md)                     |
| Product Expert Agent     | Business rule catalogue (immutable after Phase 2 approval)                 | [../Shared Agents/product-expert-agent.md](../Shared%20Agents/product-expert-agent.md)         |
| QA Agent                 | End-to-end manual test cases, workflow maps, UAT sign-off                  | [../Shared Agents/qa-agent.md](../Shared%20Agents/qa-agent.md)                                 |
| Code Historian Agent     | BR catalogue guardian; immutable change chronicle; peer review enforcement | [../Shared Agents/code-historian-agent.md](../Shared%20Agents/code-historian-agent.md)         |
| Documentation Agent      | /docs folder; Azure DevOps wiki; metrics, dependencies, external systems   | [../Shared Agents/documentation-agent.md](../Shared%20Agents/documentation-agent.md)           |

---

## Workflow

See [../Shared Agents/workflow.md](../Shared%20Agents/workflow.md) for the complete orchestration flow, decision gates, and agent communication protocols.

See [../Shared Agents/peer-review-protocol.md](../Shared%20Agents/peer-review-protocol.md) for the mandatory peer review process, debate rules, commit cadence, and rollback protocol that apply to every code change.

---

## Guiding Principles

1. **Never break what works** — if there are no tests, every change must be provably safe
2. **Characterize before you change** — understand behavior first, transform second
3. **Small steps, always green** — no change that cannot be immediately reverted
4. **Business rules are sacred** — infer, document, and test every implicit rule before touching it
5. **Disagree loudly, decide clearly** — agents surface conflicts to the Orchestrator; the Orchestrator decides
6. **Commit often, commit small** — every peer-reviewed, passing change is committed immediately so it can be rolled back independently
7. **Test early, not late** — tests are written before or alongside the code change; no change ships without a passing test
8. **Peer review is mandatory** — all coding agents review every code change; no change is merged on a single agent's authority
