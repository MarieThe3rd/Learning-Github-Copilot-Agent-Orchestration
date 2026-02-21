# Getting Started — .NET 8 Application Modernization with Agent Orchestration

This guide is for a developer who wants to use this agent system to modernize a **.NET 8 console app** or **.NET 8 Blazor app**. If you are migrating a legacy ASP.NET Web Forms (.NET 4.8) application, refer to the [Net48 WebForms getting-started guide](../Net48%20WebForms/getting-started.md) instead.

---

## How This Differs From the Web Forms System

The Web Forms system focuses on making untestable code testable by introducing seams. The .NET 8 system focuses on code quality, test coverage, and architecture — the apps are already on modern .NET.

| Web Forms System                                             | .NET 8 System                                             |
| ------------------------------------------------------------ | --------------------------------------------------------- |
| Phase 1: Introduce seams (break HttpContext/DB coupling)     | Phase 1: Assess codebase, remove dead code, enforce style |
| Phase 2: Write characterization tests (pin current behavior) | Phase 2: Write unit/integration/component tests           |
| Phase 3: Migrate to .NET 10, clean architecture              | Phase 3: Enforce Vertical Slice Architecture, clean code  |

---

## Prerequisites

| Requirement                       | Detail                                                               |
| --------------------------------- | -------------------------------------------------------------------- |
| **VS Code**                       | Latest stable release                                                |
| **GitHub Copilot extension**      | Installed and signed in — `GitHub.copilot`                           |
| **GitHub Copilot Chat extension** | Installed — `GitHub.copilot-chat`                                    |
| **GitHub Copilot plan**           | Individual, Business, or Enterprise                                  |
| **Git**                           | Must be on a feature branch before any work begins — never on `main` |
| **.NET 8 SDK**                    | Installed and matching your project's target                         |
| **xUnit**                         | For console app and service tests                                    |
| **bUnit**                         | For Blazor component tests                                           |
| **NSubstitute or Moq**            | For test doubles                                                     |

---

## Step 1 — Create a Working Branch

```bash
git checkout -b phase-1-code-assessment
```

Never work on `main` or `master`.

---

## Step 2 — Identify Your App Type

This determines which specialist agent you will use alongside the shared agents.

| App type                            | Primary expert agent                                                       |
| ----------------------------------- | -------------------------------------------------------------------------- |
| Blazor Server or Blazor WebAssembly | [`agents/blazor-expert-agent.md`](agents/blazor-expert-agent.md)           |
| Console app or Worker Service       | [`agents/console-app-expert-agent.md`](agents/console-app-expert-agent.md) |

---

## Step 3 — Start an Orchestrator Session

1. Open GitHub Copilot Chat: **`Ctrl+Shift+I`**
2. Open [`orchestrator.md`](orchestrator.md) — copy everything inside the `## System Prompt` triple-backtick block
3. Paste it as your first message and send it
4. Tell the Orchestrator what you are working on:

**For a Blazor app:**

> _"We are starting Phase 1 on a .NET 8 Blazor Server application. It has minimal test coverage and I suspect there is dead code and logic in @code blocks that should be in services."_

**For a console app:**

> _"We are starting Phase 1 on a .NET 8 console app using Generic Host and two BackgroundService workers. Test coverage is near zero. Configuration is a mix of magic strings and IConfiguration key reads."_

The Orchestrator will assign the first task to the appropriate expert agent.

---

## Step 4 — Run the Phase 1 Inventory

Open a new Copilot Chat tab and load your expert agent's system prompt. Then:

**Blazor:**

> _"Produce a Phase 1 component inventory for this Blazor application. Here are the .razor files: [use #file: references or paste code]"_

**Console app:**

> _"Produce a Phase 1 service inventory for this console application. Here are the service classes and Program.cs: [use #file: references]"_

The agent returns a structured inventory table. Paste it back to the Orchestrator and ask for the next task.

### Referencing files without copy-pasting

```
Analyze this component: #file:Pages/Orders/OrderList.razor
```

---

## Step 5 — Dead Code Removal and Style Cleanup

The Orchestrator will route these tasks to the **Clean Code Expert Agent** ([`../Net48 WebForms/agents/clean-code-expert-agent.md`](../Net48%20WebForms/agents/clean-code-expert-agent.md)).

For each dead code item:

1. Clean Code Expert proposes removal
2. Refactoring Expert confirms it is safe
3. Remove and commit one class/method at a time:

```bash
git commit -m "[PHASE-1] [CHR-001] Remove unused OrderHistoryArchiver class"
```

---

## Step 6 — Phase 2: Writing Tests

Once Phase 1 is signed off, create a new branch:

```bash
git checkout -b phase-2-test-coverage
```

Open an expert agent session and request tests for a specific service or component:

**Console app — unit test:**

> _"Write xUnit unit tests for OrderCalculationService. All dependencies are injected. Here is the class: #file:Services/OrderCalculationService.cs"_

**Blazor — bUnit test:**

> _"Write bUnit tests for the OrderList component. The component uses IOrderService. Here is the component: #file:Pages/Orders/OrderList.razor"_

Each test references the business rule it protects (BR-NNN).

---

## Step 7 — Business Rule Discovery

While writing tests, the **Product Expert Agent** ([`../Net48 WebForms/agents/product-expert-agent.md`](../Net48%20WebForms/agents/product-expert-agent.md)) is brought in to name and catalogue every rule the tests reveal.

Paste the test output to the Orchestrator and say:

> _"The expert agent has inferred this behavior from the code. Please ask the Product Expert Agent to name and catalogue this as a business rule."_

Each confirmed rule gets a BR-NNN identifier and a CHR-NNN log entry.

---

## Step 8 — Phase 3: Clean Architecture

Create a new branch:

```bash
git checkout -b phase-3-clean-architecture
```

The Orchestrator drives the slice extraction. Every file move is its own commit:

```bash
git commit -m "[PHASE-3] [CHR-042] Move OrderService to Features/Orders/"
```

The **Architect Agent** ([`../Net48 WebForms/agents/architect-agent.md`](../Net48%20WebForms/agents/architect-agent.md)) defines the target folder structure. Nothing moves without its approval.

---

## Typical Day Workflow

```
1. git checkout -b [branch-name]
2. Open Orchestrator chat tab — load system prompt from orchestrator.md
3. Tell Orchestrator the current phase and what you're working on today
4. Orchestrator assigns a task to a named agent
5. Open new tab — load that agent's system prompt
6. Give agent the task (reference files with #file:)
7. Paste agent output back to Orchestrator
8. Orchestrator names peer reviewer(s)
9. Open new tab — load reviewer's system prompt — request review
10. Paste approval back to Orchestrator
11. Open new tab — load Code Historian system prompt — log CHR entry
12. git commit -m "[PHASE-N] [CHR-NNN] Description"
13. Repeat from step 4
```

---

## Agent Quick Reference

### Agents in This Folder

| Agent                 | File                                                                       | When to use                      |
| --------------------- | -------------------------------------------------------------------------- | -------------------------------- |
| Orchestrator (.NET 8) | [`orchestrator.md`](orchestrator.md)                                       | Always open — master controller  |
| Blazor Expert         | [`agents/blazor-expert-agent.md`](agents/blazor-expert-agent.md)           | Blazor apps — all phases         |
| Console App Expert    | [`agents/console-app-expert-agent.md`](agents/console-app-expert-agent.md) | Console/worker apps — all phases |

### Reused Agents (load from Net48 WebForms folder)

| Agent               | File                                                                                                               |
| ------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Refactoring Expert  | [`../Net48 WebForms/agents/refactoring-expert-agent.md`](../Net48%20WebForms/agents/refactoring-expert-agent.md)   |
| Clean Code Expert   | [`../Net48 WebForms/agents/clean-code-expert-agent.md`](../Net48%20WebForms/agents/clean-code-expert-agent.md)     |
| Architect           | [`../Net48 WebForms/agents/architect-agent.md`](../Net48%20WebForms/agents/architect-agent.md)                     |
| Microsoft Practices | [`../Net48 WebForms/agents/microsoft-practices-agent.md`](../Net48%20WebForms/agents/microsoft-practices-agent.md) |
| Product Expert      | [`../Net48 WebForms/agents/product-expert-agent.md`](../Net48%20WebForms/agents/product-expert-agent.md)           |
| QA Agent            | [`../Net48 WebForms/agents/qa-agent.md`](../Net48%20WebForms/agents/qa-agent.md)                                   |
| Code Historian      | [`../Net48 WebForms/agents/code-historian-agent.md`](../Net48%20WebForms/agents/code-historian-agent.md)           |
| Documentation       | [`../Net48 WebForms/agents/documentation-agent.md`](../Net48%20WebForms/agents/documentation-agent.md)             |

---

## Common Mistakes

| Mistake                                          | What to do instead                                 |
| ------------------------------------------------ | -------------------------------------------------- |
| Working on `main`                                | Create a feature branch first                      |
| Moving multiple files in one commit              | One file move = one commit                         |
| Skipping peer review                             | Every change needs reviewer approval before commit |
| Skipping the Code Historian                      | Every committed change needs a CHR-NNN entry       |
| Writing tests and refactoring in the same change | Tests first, green, commit — then refactor         |
| Asking the Architect Agent to write code         | It defines structure; coding agents implement      |
| Adding `#region` blocks                          | Extract a class or method instead                  |
