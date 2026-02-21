# Phase 1: Safe Refactoring

## Goal

Make every class in the legacy ASP.NET Web Forms (.NET 4.8) codebase **testable in isolation** without changing any observable behavior.

At the end of Phase 1, it must be possible to instantiate and exercise every piece of business logic without starting a web server, connecting to a database, or touching any external system.

---

## Phase Gate

Phase 1 is complete when **all** of the following are true:

```markdown
- [ ] Full inventory of all classes, pages, and modules is complete
- [ ] Seam analysis complete for every class
- [ ] All HttpContext dependencies wrapped behind an interface
- [ ] All Session dependencies wrapped behind an interface
- [ ] All Response/Request dependencies wrapped behind an interface
- [ ] All direct database calls (SqlCommand, SqlConnection, DataAdapter, DataSet) abstracted
- [ ] All static method calls in business logic replaced with instance method calls
- [ ] All direct DateTime.Now/UtcNow usages replaced with ISystemClock
- [ ] All external service calls (email, file I/O, HTTP) abstracted
- [ ] Build passes with zero new errors or warnings introduced by Phase 1 changes
- [ ] No behavior changes — all changes approved by Refactoring Expert Agent as behavior-preserving
- [ ] Fake/stub implementations exist for every new interface
- [ ] Orchestrator has reviewed and approved phase transition
```

---

## Agents Active in Phase 1

| Agent                        | Role                                                                      |
| ---------------------------- | ------------------------------------------------------------------------- |
| **Legacy Code Expert Agent** | Leads — identifies seams, designs interfaces, implements safe changes     |
| **Refactoring Expert Agent** | Reviews every change for behavior preservation; approves each step        |
| **Clean Code Expert Agent**  | Light advisory — names new interfaces and classes appropriately           |
| **Microsoft Agent**          | Flags .NET 4.8 APIs that will need special treatment in Phase 3           |
| **Product Expert Agent**     | Standby — alerted if a seam boundary would hide visible business behavior |
| **QA Agent**                 | Begins mapping user-facing workflows from .aspx pages                     |
| **Orchestrator**             | Tracks per-class progress; enforces gate; routes conflicts                |

---

## Step-by-Step Process

### Step 1: Codebase Inventory

Before any refactoring begins, build a complete map of the codebase.

```markdown
Inventory Checklist:

- [ ] List all .aspx pages with their code-behind files
- [ ] List all App_Code classes
- [ ] List all HTTP Handlers (.ashx)
- [ ] List all HTTP Modules
- [ ] List all Global.asax event handlers
- [ ] List all stored procedures called from code
- [ ] Identify all third-party dependencies (NuGet packages, DLL references)
- [ ] Count direct uses of: HttpContext, Session, SqlCommand, DataSet, static helpers
- [ ] Identify all static classes used as utilities/helpers
```

Output: `inventory.md` and `dependency-map.md`

---

### Step 2: Prioritization

Not all classes are equal. Prioritize by:

1. **Classes with the most business logic** — highest value seams
2. **Classes called by the most other classes** — breaking their static dependencies unblocks the most
3. **Classes with the fewest external dependencies** — easiest wins first to build confidence

---

### Step 3: Per-Class Seam Introduction

For each class (in priority order):

```
A. Map all external dependencies (see Legacy Code Expert Agent taxonomy)
B. For each dependency:
   1. Create the interface (e.g., IOrderRepository, IEmailService, ISystemClock)
   2. Create the real implementation wrapping the existing code
   3. Create a fake/test-double implementation for use in Phase 2 tests
   4. Modify the class to accept the dependency via constructor
      - Add default constructor for backward compatibility (new RealImpl())
      - Add overload constructor taking the interface
C. Refactoring Expert Agent reviews the change — is it behavior-preserving?
D. Build must pass after each class is done
E. Log completed class in Orchestrator's progress tracker
```

---

### Step 4: Static Dependency Elimination

Static calls are the hardest barrier to testability. Process for each:

```csharp
// BEFORE: static call — untestable
public decimal CalculateShipping(Order order)
{
    var rate = ShippingRateHelper.GetRate(order.DestinationZip);
    return order.Weight * rate;
}

// STEP A: Create interface
public interface IShippingRateProvider
{
    decimal GetRate(string destinationZip);
}

// STEP B: Create real implementation wrapping the static
public class ShippingRateProviderAdapter : IShippingRateProvider
{
    public decimal GetRate(string destinationZip)
        => ShippingRateHelper.GetRate(destinationZip);  // unchanged static call here
}

// STEP C: Create fake for tests
public class FakeShippingRateProvider : IShippingRateProvider
{
    private readonly decimal _fixedRate;
    public FakeShippingRateProvider(decimal fixedRate) { _fixedRate = fixedRate; }
    public decimal GetRate(string destinationZip) => _fixedRate;
}

// STEP D: Update the class — backward compatible
public class ShippingCalculator
{
    private readonly IShippingRateProvider _rateProvider;

    public ShippingCalculator()  // backward-compatible default
        : this(new ShippingRateProviderAdapter()) { }

    public ShippingCalculator(IShippingRateProvider rateProvider)
        => _rateProvider = rateProvider;

    public decimal CalculateShipping(Order order)
    {
        var rate = _rateProvider.GetRate(order.DestinationZip);
        return order.Weight * rate;
    }
}
```

---

### Step 5: HttpContext Quarantine

All `HttpContext.Current` usages must be quarantined behind an interface by end of Phase 1.

```csharp
public interface IWebContext
{
    string? GetSessionValue(string key);
    void SetSessionValue(string key, string value);
    void ClearSession(string key);
    string? GetRequestFormValue(string key);
    string? GetRequestQueryValue(string key);
    string GetClientIpAddress();
    void RedirectTo(string url);
}

public class HttpWebContext : IWebContext
{
    public string? GetSessionValue(string key)
        => HttpContext.Current.Session[key]?.ToString();

    public void SetSessionValue(string key, string value)
        => HttpContext.Current.Session[key] = value;

    // ... all other members delegate to HttpContext.Current
}
```

The goal is that **no business logic class directly references `System.Web`** by the end of Phase 1.

---

### Step 6: Database Abstraction

All `SqlCommand`, `SqlConnection`, `DataAdapter`, `DataSet` usage in business logic must be behind a repository interface.

```markdown
For each unique data access pattern found:

- [ ] Identify the logical entity/aggregate being accessed
- [ ] Create IEntityRepository interface with methods matching the query patterns
- [ ] Create SqlEntityRepository implementing the interface using existing ADO.NET code
- [ ] Create FakeEntityRepository returning canned, in-memory data
- [ ] Update calling code to depend on interface
```

**Important:** The ADO.NET SQL code moves into the `SqlEntityRepository`. It is not deleted. It is not improved. The goal is isolation, not improvement — that is Phase 3's job.

---

## Deliverables

By end of Phase 1, the Legacy Code Expert Agent produces:

| Deliverable            | Description                                                                     |
| ---------------------- | ------------------------------------------------------------------------------- |
| `inventory.md`         | Full list of all pages, classes, and modules                                    |
| `dependency-map.md`    | External dependencies per class                                                 |
| `seam-registry.json`   | Per-class seam log (interfaces introduced, statics removed)                     |
| `interfaces/`          | All new interface definitions                                                   |
| `fakes/`               | All test-double implementations                                                 |
| `risks.md`             | Changes that were difficult or potentially risky, flagged for Phase 2 attention |
| Phase 2 gate checklist | Completed and signed off by Orchestrator                                        |

---

## Safety Rules

> These are non-negotiable constraints for Phase 1.

1. **No feature changes** — Phase 1 changes are structural only
2. **No rewrites** — copy behavior, do not improve it
3. **Build must pass after every class** — never commit a broken build
4. **Each change reviewed by Refactoring Expert Agent** before merging
5. **Use IDE-automated refactoring where possible** — manual edits introduce risk
6. **Default constructors preserved** — existing call sites must not require changes
7. **No database schema changes** — Phase 1 does not touch the database
