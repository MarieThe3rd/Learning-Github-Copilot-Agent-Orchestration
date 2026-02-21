# Peer Review Protocol

## Principle

> No code change is merged on a single agent's authority.
> All coding agents review every change. Debate is expected. Consensus is required.

This protocol applies to **every code change** in all three phases, without exception. Small changes still require review. "Obvious" changes still require review. The goal is not bureaucracy — it is collective ownership and early error detection.

---

## Who Reviews What

### Coding Agents (must review all code changes)

| Agent                    | Review Focus                                                                 |
| ------------------------ | ---------------------------------------------------------------------------- |
| Legacy Code Expert Agent | Testability impact; seam correctness; no behavior change (Phase 1)           |
| Refactoring Expert Agent | Behavior preservation; correct catalog entry applied; mechanical correctness |
| Clean Code Expert Agent  | Naming; function size; SOLID; KISS; no over-engineering                      |
| Microsoft Agent          | .NET idioms; DI correctness; EF Core usage; no prohibited packages           |
| Architect Agent          | VSA folder placement; shared kernel rules; no cross-slice imports; YAGNI     |

### Supporting Agents (consulted on relevant changes)

| Agent                | Consulted When                                                       |
| -------------------- | -------------------------------------------------------------------- |
| Product Expert Agent | Any change that touches business logic or a BR-NNN-related code path |
| QA Agent             | Any change that might affect a P1/P2 E2E test case                   |

### Record Keepers (always notified)

| Agent                | Role in Every Review                                                         |
| -------------------- | ---------------------------------------------------------------------------- |
| Code Historian Agent | Logs the complete review record as a CHR-NNN entry before merge is permitted |

---

## Review Lifecycle

```
┌────────────────────────────────────────────────────────────────────┐
│  1. PROPOSE                                                        │
│     Coding agent proposes a single, minimal change                 │
│     Posts: before snippet, after snippet, intent, BR-NNN ref       │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  2. REVIEW ROUND 1                                                 │
│     All other coding agents post their position:                   │
│       ✅ Approved  — no objections                                 │
│       ⚠️ Requested Change — specific issue; must be resolved       │
│       ❌ Objection — fundamental disagreement; triggers debate      │
└───────────────────────────────┬────────────────────────────────────┘
                                │
               ┌────────────────┼──────────────────────┐
               │                │                      │
               ▼                ▼                      ▼
         All ✅            ⚠️ present            ❌ present
               │                │                      │
               │    Proposer addresses    Debate Round 1:
               │    each Requested       Agents argue positions
               │    Change               Orchestrator summarises
               │    → Re-review          → Review Round 2
               │                              │
               │                    Still ❌ → Debate Round 2
               │                              │
               │                    Still ❌ → Orchestrator decides
               │                              │
               └──────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  3. CONSENSUS                                                      │
│     All coding agents show ✅                                      │
│     Product Expert / QA sign off if consulted                      │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  4. CHRONICLE                                                      │
│     Code Historian logs CHR-NNN entry with:                        │
│     - before/after diff                                            │
│     - all review positions and debate transcript                   │
│     - final decision and consensus statement                       │
│     - BR-NNN references                                            │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  5. COMMIT                                                         │
│     Commit message: [PHASE-N] [CHR-NNN] Short description          │
│     Single change per commit — no batching                         │
│     Build must be green on commit                                  │
│     All existing tests must pass on commit                         │
└────────────────────────────────────────────────────────────────────┘
```

---

## Debate Rules

When an agent raises an `❌ Objection`, debate proceeds as follows:

1. **Each agent states their position in one paragraph** — clear claim + rationale
2. **Each agent responds to the other positions** — acknowledge valid points, rebut invalid ones
3. **Agents may change their position** — changing your position when presented with a better argument is correct behaviour, not weakness
4. **Orchestrator summarises** the state of agreement after each round
5. **Maximum two debate rounds** before Orchestrator decides unilaterally
6. **Orchestrator's decision is final** — reason is logged; all agents acknowledge and proceed

### What Makes a Good Argument

| Good argument                                                       | Poor argument                                                          |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| "This violates BR-012 because..."                                   | "I don't like it"                                                      |
| "This will make the test brittle because..."                        | "It's not how I would do it"                                           |
| "NetArchTest will fail because the import crosses a slice boundary" | "This isn't the pattern we agreed on" (without citing which agreement) |
| "The commit is too large — split at Extract Interface"              | "It should be smaller"                                                 |

---

## Iterative Change Rules

These rules are enforced by the Orchestrator and verified by the Code Historian before a CHR-NNN entry is recorded as MERGED:

| Rule                                                           | Enforcement                                                            |
| -------------------------------------------------------------- | ---------------------------------------------------------------------- |
| One logical change per commit                                  | Code Historian rejects CHR entries covering multiple unrelated changes |
| Build green before commit                                      | Code Historian requires CI confirmation in CHR entry                   |
| All existing tests green before commit                         | Code Historian requires test run result in CHR entry                   |
| Test written before or alongside code change                   | Refactoring Expert and Clean Code Expert verify in review              |
| No change larger than can be reviewed in a single review round | Legacy Code Expert can request a split; proposer must comply           |
| Rollback is always possible                                    | Each commit must be independently revertable with `git revert`         |

### Commit Size Guidelines

| Change type                                              | Typical commit size                                                     |
| -------------------------------------------------------- | ----------------------------------------------------------------------- |
| Extract interface from one class                         | 1 commit                                                                |
| Add one characterization test                            | 1 commit                                                                |
| Rename one class                                         | 1 commit                                                                |
| Add one slice (Request + Response + Validator + Handler) | 1 commit per file, or 1 commit for the whole slice if all files are new |
| Migrate one Web Forms page to Razor Page                 | 1 commit                                                                |
| Add EF Core entity configuration for one table           | 1 commit                                                                |

---

### Branch Rules

| Rule                                                                         | Enforcement                                                           |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Never commit directly to main or master                                      | Orchestrator rejects any review that targets the default branch       |
| Create a feature branch at the start of every work session                   | Branch is created before the first file is changed                    |
| Branch naming: `[phase-N]-[brief-description]` or descriptive kebab-case     | Reviewers check branch name before accepting the PR                   |
| Each branch covers one logical unit of work (one class, one slice, one seam) | Legacy Code Expert or Refactoring Expert can request a split          |
| Merge only after all coding agents have approved and CHR-NNN is logged       | Code Historian confirms entry exists; Orchestrator approves the merge |

---

## Rollback Protocol

If a committed change is later found to be incorrect:

1. **Identify** the CHR-NNN entry for the bad commit
2. **Legacy Code Expert Agent** confirms the change is the source of the problem
3. **Refactoring Expert Agent** confirms `git revert CHR-NNN-commit-sha` is safe
4. **Orchestrator** approves the revert
5. **Code Historian** logs a CHR-REVERT-NNN entry with:
   - Original CHR-NNN reference
   - Reason for revert
   - What was found to be wrong
   - All agents who approved the revert
6. Commit the revert: `git revert {sha}` with message `[REVERT] [CHR-NNN] Reason`
7. Work begins again from the reverted state

---

## Phase-Specific Peer Review Focus

### Phase 1

| Priority | What reviewers look for                                            |
| -------- | ------------------------------------------------------------------ |
| 1        | Is the behavior identical before and after? (Refactoring Expert)   |
| 2        | Is the seam correct — will a fake work here? (Legacy Code Expert)  |
| 3        | Is the new interface name intention-revealing? (Clean Code Expert) |
| 4        | Is this in the right place per VSA? (Architect Agent)              |
| 5        | Is the DI registration pattern correct? (Microsoft Agent)          |

### Phase 2

| Priority | What reviewers look for                                                                       |
| -------- | --------------------------------------------------------------------------------------------- |
| 1        | Does the test name use the BDD nested class pattern? (Clean Code Expert)                      |
| 2        | Does the test reference the correct BR-NNN? (Product Expert)                                  |
| 3        | Does the test cover the stated behavior and nothing else? (Legacy Code Expert)                |
| 4        | Does adding this test require any production code change? (Refactoring Expert — should be NO) |

### Phase 3

| Priority | What reviewers look for                                                               |
| -------- | ------------------------------------------------------------------------------------- |
| 1        | Do all characterization tests from Phase 2 still pass? (Legacy Code Expert)           |
| 2        | Is the slice in the correct VSA folder? (Architect Agent)                             |
| 3        | Does the handler use DbContext directly, not a generic repository? (Microsoft Agent)  |
| 4        | Does the code satisfy KISS — is it the simplest thing that works? (Clean Code Expert) |
| 5        | Is this a correct behavior-preserving transformation? (Refactoring Expert)            |
