# Getting Started — Legacy .NET Modernization with Agent Orchestration

This guide is for a developer who has never used AI agents before and wants to start using this system to modernize a legacy ASP.NET Web Forms (.NET 4.8) codebase.

---

## Prerequisites

| Requirement                       | Detail                                                                              |
| --------------------------------- | ----------------------------------------------------------------------------------- |
| **VS Code**                       | Latest stable release                                                               |
| **GitHub Copilot extension**      | Installed and signed in — `GitHub.copilot`                                          |
| **GitHub Copilot Chat extension** | Installed — `GitHub.copilot-chat`                                                   |
| **GitHub Copilot plan**           | Individual, Business, or Enterprise (agent/chat features required)                  |
| **Git**                           | Installed; you must be on a feature branch before any work begins — never on `main` |
| **.NET SDK**                      | Whatever version matches your project (for building and verifying changes)          |

---

## How This System Works (30-second version)

Each agent is a **markdown file containing a system prompt** — a set of instructions that tells an AI model how to behave, what it knows, and what it is allowed to decide.

You load a system prompt into GitHub Copilot Chat at the start of a session. The AI then role-plays that agent for the rest of the conversation. You switch agents by starting a new chat and loading a different system prompt.

The **Orchestrator** is the one you talk to first. It tells you which agent to involve next and routes decisions.

---

## Step 1 — Create a Working Branch

Before touching anything, create a branch:

```bash
git checkout -b phase-1-initial-inventory
```

Never work on `main` or `master`. The Orchestrator will reject any work done on the default branch.

---

## Step 2 — Start an Orchestrator Session

1. Open GitHub Copilot Chat: **`Ctrl+Shift+I`** (or click the chat icon in the sidebar)
2. Open [`orchestrator.md`](orchestrator.md) in VS Code
3. Find the `## System Prompt` section — copy everything inside the triple-backtick block
4. Paste it as your **first message** in Copilot Chat, then send it

The model will confirm it has taken on the Orchestrator role. A response like the following is normal:

> _"Understood. I am the Orchestrator. What is the current state of the codebase and which phase are we beginning?"_

5. Tell it what you are working on:

> _"We are starting Phase 1. The codebase is a legacy ASP.NET Web Forms application targeting .NET 4.8. It has zero tests. I need to begin the class inventory."_

The Orchestrator will issue its first task: usually asking the **Legacy Code Expert Agent** to analyze a specific class.

---

## Step 3 — Switch to the Agent the Orchestrator Requests

When the Orchestrator says something like:

> _"Task for Legacy Code Expert Agent: analyze `CustomerOrderProcessor.cs` for seams."_

1. Open a **new Copilot Chat tab** (click `+` in the chat panel)
2. Open the relevant agent file (e.g., [`agents/legacy-code-expert-agent.md`](agents/legacy-code-expert-agent.md))
3. Copy the system prompt text from its `## System Prompt` block
4. Paste it as your first message in the new tab

You now have the Legacy Code Expert Agent active. Give it the task:

> _"Analyze `CustomerOrderProcessor.cs` for seams. Here is the class: [paste code or use `#file:CustomerOrderProcessor.cs`]"_

### Referencing files without copy-pasting

Instead of pasting code, you can reference a file directly in Copilot Chat:

```
Analyse this class for seams: #file:src/CustomerOrderProcessor.cs
```

---

## Step 4 — Understand the Output Format

Each agent returns findings in a structured format. The Legacy Code Expert will return something like:

```
SEAM ANALYSIS: CustomerOrderProcessor
  - SqlConnection (line 42): direct instantiation → Extract ICustomerRepository
  - DateTime.Now (line 87): hardcoded → Extract ISystemClock
  - SmtpClient (line 103): direct instantiation → Extract IEmailService

SEAMS FOUND: 3
PROPOSED CHANGES: [list of small, ordered steps]
REVIEW REQUIRED: Refactoring Expert Agent
```

Copy this output and bring it back to the Orchestrator session. Paste it and say:

> _"Legacy Code Expert has returned this analysis. What is the next step?"_

---

## Step 5 — Run a Peer Review

Before any change is committed, it must be reviewed by at least one other agent. The peer review process is defined in [`peer-review-protocol.md`](peer-review-protocol.md).

**In practice:**

1. The Orchestrator names the reviewing agents required (e.g., _"Refactoring Expert must approve this seam extraction"_)
2. Open a new tab, load the Refactoring Expert system prompt
3. Paste the proposed change and ask:

> _"Please review this proposed seam extraction for behavior-preservation. [paste diff or code]"_

4. The agent returns **Approved**, **Approved with conditions**, or **Blocked**
5. Paste the review result back to the Orchestrator
6. If approved: commit the change (one change = one commit)
7. If blocked: address the issue, then re-review

---

## Step 6 — Log the Change with the Code Historian

Every approved, committed change must be recorded. Open a new tab, load the Code Historian system prompt from [`agents/code-historian-agent.md`](agents/code-historian-agent.md), and say:

> _"Log this change. Before: [original code snippet]. After: [new code snippet]. Proposed by: Legacy Code Expert Agent. Approved by: Refactoring Expert Agent."_

The Code Historian returns a CHR entry:

```
CHR-001
Date: 2026-02-20
Agent: Legacy Code Expert Agent
Reviewers: Refactoring Expert Agent (Approved)
Before: [snippet]
After: [snippet]
Decision: Approved — behavior-preserving seam introduction
```

Include the CHR number in your commit message:

```bash
git commit -m "[PHASE-1] [CHR-001] Extract ICustomerRepository from CustomerOrderProcessor"
```

---

## Step 7 — Check the Phase Gate Before Moving On

When you think Phase 1 is complete, return to the Orchestrator session and say:

> _"All classes have been analyzed and seamed. Please evaluate the Phase 1 gate."_

The Orchestrator will run through the gate checklist from [`phases/phase-1-safe-refactoring.md`](phases/phase-1-safe-refactoring.md) and tell you what remains before you can advance to Phase 2.

---

## Typical Day Workflow

```
1. git checkout -b [branch-name]
2. Open Orchestrator chat tab — load system prompt
3. Tell Orchestrator what you are working on today
4. Orchestrator assigns a task to a named agent
5. Open new tab — load that agent's system prompt
6. Give the agent the task (reference files with #file:)
7. Paste the agent's output back to Orchestrator
8. Orchestrator names peer reviewer(s)
9. Open new tab — load reviewer's system prompt — request review
10. Paste approval back to Orchestrator
11. Open new tab — load Code Historian — log the CHR entry
12. git commit -m "[PHASE-N] [CHR-NNN] Description"
13. Repeat from step 4 for the next task
```

---

## Agent Quick Reference

| Agent               | File                                                                         | When to use                                     |
| ------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------- |
| Orchestrator        | [`orchestrator.md`](orchestrator.md)                                         | Always open — master controller                 |
| Legacy Code Expert  | [`agents/legacy-code-expert-agent.md`](agents/legacy-code-expert-agent.md)   | Seam analysis, characterization tests, Phase 1  |
| Refactoring Expert  | [`agents/refactoring-expert-agent.md`](agents/refactoring-expert-agent.md)   | Behavior-preserving review, refactoring catalog |
| Clean Code Expert   | [`agents/clean-code-expert-agent.md`](agents/clean-code-expert-agent.md)     | Naming, SOLID, structure review, Phase 3        |
| Microsoft Practices | [`agents/microsoft-practices-agent.md`](agents/microsoft-practices-agent.md) | .NET 10, ASP.NET Core, EF Core decisions        |
| Architect           | [`agents/architect-agent.md`](agents/architect-agent.md)                     | Vertical Slice Architecture, solution layout    |
| Product Expert      | [`agents/product-expert-agent.md`](agents/product-expert-agent.md)           | Business rule discovery and approval, Phase 2   |
| QA Agent            | [`agents/qa-agent.md`](agents/qa-agent.md)                                   | End-to-end test cases, UAT sign-off             |
| Code Historian      | [`agents/code-historian-agent.md`](agents/code-historian-agent.md)           | Log every approved change as CHR-NNN            |
| Documentation       | [`agents/documentation-agent.md`](agents/documentation-agent.md)             | /docs, Azure DevOps wiki, Mermaid diagrams      |

---

## Common First-Day Mistakes

| Mistake                                   | What to do instead                                                               |
| ----------------------------------------- | -------------------------------------------------------------------------------- |
| Working directly on `main`                | Always create a feature branch first                                             |
| Making multiple changes before committing | One change, one review, one commit — then the next change                        |
| Skipping the peer review                  | Every change needs at least one agent reviewer before commit                     |
| Skipping the Code Historian               | Every committed change needs a CHR-NNN entry                                     |
| Asking the Orchestrator to write code     | It will refuse — it routes and decides; coding agents write code                 |
| Loading the wrong phase's agent           | Check the Orchestrator's instruction — it names which agent to use               |
| Making a large change at once             | The Orchestrator will reject batched changes — split into the smallest safe unit |
| Adding `#region` blocks                   | Never — extract a class or method instead                                        |
| Leaving TODO comments                     | Open an issue tracker item instead; remove the comment                           |

---

## Where to Go Next

- **Phase details:** [`phases/`](phases/) — gate checklists for each phase
- **Peer review rules:** [`peer-review-protocol.md`](peer-review-protocol.md)
- **Agent communication model:** [`workflow.md`](workflow.md)
- **Full system overview:** [`README.md`](README.md)
