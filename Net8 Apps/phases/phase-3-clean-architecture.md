# Phase 3: Clean Architecture

## Goal

Enforce Vertical Slice Architecture. Apply SOLID principles throughout. Ensure every part of the codebase is structured so that a competent developer unfamiliar with the project can understand, modify, and extend any feature without touching anything unrelated to it.

---

## Phase Gate

Phase 3 is complete when **all** of the following are true:

```markdown
- [ ] All features organized into vertical slices (one folder per feature)
- [ ] No horizontal layers (no Controllers/, Services/, Repositories/ as top-level folders)
- [ ] No cross-slice dependencies — slices are self-contained
- [ ] Shared kernel is minimal — only concepts used by 3+ slices live in Shared/
- [ ] All classes obey Single Responsibility Principle
- [ ] No class exceeds 200 lines
- [ ] No method exceeds 20 lines
- [ ] No method has more than 3 parameters
- [ ] All tests still green after structural changes
- [ ] Code coverage maintained at or above Phase 2 level
- [ ] Zero code style violations (no regions, no dead code, no TODO comments)
- [ ] Architecture conformance tests written and passing
- [ ] For Blazor apps: component hierarchy is clean, state management is explicit
- [ ] For console apps: each worker is focused on one job; configuration is typed (IOptions<T>)
- [ ] CI/CD pipeline green
- [ ] Orchestrator has reviewed and approved phase completion
```

---

## Agents Active in Phase 3

| Agent                         | Role                                                                                         |
| ----------------------------- | -------------------------------------------------------------------------------------------- |
| **Architect Agent**           | Primary — defines and enforces slice structure; approves all structural decisions            |
| **Blazor Expert Agent**       | Primary (Blazor apps) — drives component restructuring; enforces component design principles |
| **Console App Expert Agent**  | Primary (console apps) — drives worker and service restructuring                             |
| **Clean Code Expert Agent**   | Primary — enforces naming, SOLID, method size, cohesion throughout                           |
| **Refactoring Expert Agent**  | Primary — sequences and executes all refactoring catalog steps                               |
| **Microsoft Practices Agent** | Advisory — validates .NET 8 idioms, DI patterns, EF Core usage                               |
| **Product Expert Agent**      | Authority — Business Rule Catalogue is now immutable; no rule may change                     |
| **QA Agent**                  | Validates — runs full regression suite; signs off UAT readiness                              |
| **Code Historian Agent**      | Records every CHR-NNN; ensures no change bypasses peer review                                |
| **Documentation Agent**       | Publishes final architecture docs, ADR records, and Azure DevOps wiki                        |

---

## Vertical Slice Architecture

### Target Structure

**Console App:**

```
src/
  Features/
    OrderProcessing/
      ProcessOrdersWorker.cs       ← BackgroundService for this feature
      ProcessOrdersService.cs      ← Business logic
      IProcessOrdersService.cs
      ProcessOrdersOptions.cs      ← IOptions<T> config model
      OrderProcessingTests/        ← Tests live next to the feature
        ProcessOrdersServiceTests.cs
        ProcessOrdersWorkerTests.cs
    EmailNotification/
      EmailNotificationService.cs
      IEmailNotificationService.cs
      EmailOptions.cs
  Shared/
    Persistence/                   ← Only if used by 3+ features
    Extensions/
  Program.cs
```

**Blazor App:**

```
src/
  Features/
    Orders/
      OrderList.razor
      OrderList.razor.cs           ← Code-behind only for lifecycle; logic in service
      OrderDetail.razor
      OrderDetail.razor.cs
      IOrderService.cs
      OrderService.cs
      OrderListTests/
        OrderListTests.cs          ← bUnit tests
    Shared/
      StatusBadge.razor            ← Genuinely reusable UI components only
  Layout/
  Program.cs
```

### Slice Extraction Process

For each feature:

1. **Architect Agent** defines the slice boundary and names the feature folder
2. **Refactoring Expert Agent** moves files one at a time (smallest safe unit)
3. **Build must pass** after every single file move
4. **Tests must stay green** after every single file move
5. One file move = one commit: `[PHASE-3] [CHR-NNN] Move OrderService to Features/Orders/`

---

## Blazor-Specific Clean Architecture

### Component Design Rules

| Rule               | Bad                                            | Good                                                           |
| ------------------ | ---------------------------------------------- | -------------------------------------------------------------- |
| Component size     | 300-line .razor file with business logic       | < 100-line .razor with all logic in injected service           |
| State management   | State scattered in component fields            | Explicit state class or Fluxor store                           |
| Code-behind        | All logic in @code block                       | Code-behind (`.razor.cs`) for lifecycle only; logic in service |
| JavaScript interop | Raw `JSRuntime.InvokeAsync` calls in component | Abstracted behind `IJsInteropService`                          |
| Service injection  | 5+ @inject statements in one component         | Inject one aggregate service per concern                       |

### State Management

When Phase 3 introduces a state management pattern:

- Blazor Expert Agent recommends the approach (Fluxor, custom store, cascading state)
- Architect Agent approves the pattern
- Product Expert Agent confirms business rules are still correctly reflected
- One state concern extracted per commit

---

## Console App-Specific Clean Architecture

### Worker Design Rules

| Rule                  | Bad                                                         | Good                                                                   |
| --------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------------- |
| Worker responsibility | OrderProcessingWorker sends email, updates DB, logs metrics | OrderProcessingWorker only orchestrates; delegates to focused services |
| Configuration         | `_config["EmailServer"]` magic string                       | `IOptions<EmailOptions>` with typed properties                         |
| Error handling        | `catch (Exception e) { Console.WriteLine(e); }`             | Structured error handling with ILogger; dead-letter strategy           |
| Retry logic           | Home-grown retry loops                                      | Polly with typed `IAsyncPolicy`                                        |
| Scheduling            | `Thread.Sleep(60000)`                                       | `PeriodicTimer` or a proper scheduling library                         |

---

## Architecture Conformance Tests

Before the phase gate, the Architect Agent specifies architecture conformance tests using NetArchTest or a custom convention test:

```csharp
[Fact]
public void Features_ShouldNotDependOnEachOtherDirectly()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("Features.Orders")
        .ShouldNot().HaveDependencyOn("Features.EmailNotification")
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}

[Fact]
public void Services_ShouldHaveInterfaces()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Service")
        .And().AreNotAbstract()
        .Should().ImplementInterface(typeof(IDisposable))  // or a marker
        .GetResult();
    // Customize assertion to your interface naming convention
}
```
