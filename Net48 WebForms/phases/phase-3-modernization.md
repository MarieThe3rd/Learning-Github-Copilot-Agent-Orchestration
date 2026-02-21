# Phase 3: Modernization

## Goal

Transform the testable, characterized, legacy codebase into a well-architected **.NET 10** application — with confidence, because every business rule is tested and every behavior is documented.

Phase 3 is where the real refactoring and re-platforming happens. It is only safe because Phases 1 and 2 created the safety net.

---

## Phase Gate

Phase 3 is complete when **all** of the following are true:

```markdown
### Architecture

- [ ] Target architecture chosen, documented, and approved by Orchestrator
- [ ] All code in the correct layer per clean architecture
- [ ] Architecture conformance tests (NetArchTest) passing
- [ ] No circular dependencies between projects

### Platform Migration

- [ ] All Web Forms (.aspx/.aspx.cs) pages migrated to ASP.NET Core
- [ ] All code-behind business logic extracted to Application/Domain layers
- [ ] All ADO.NET (SqlCommand, DataSet) replaced with EF Core
- [ ] All static dependencies replaced with proper DI
- [ ] All System.Web references removed
- [ ] Configuration migrated to appsettings.json + IOptions<T>
- [ ] Logging migrated to ILogger<T>
- [ ] Authentication migrated to ASP.NET Core Identity
- [ ] Zero .NET 4.8-only assembly references

### Code Quality

- [ ] All SOLID principles adhered to (Clean Code Expert Agent sign-off)
- [ ] All code smells from Refactoring Expert Agent catalogue addressed
- [ ] Test coverage maintained ≥ 90% on business logic
- [ ] No new technical debt introduced without documented justification

### Testing

- [ ] All characterization tests from Phase 2 still pass
- [ ] New unit tests cover new code in Domain and Application layers
- [ ] Integration tests cover repository and infrastructure implementations
- [ ] All QA Agent P1 and P2 test cases executed and passing
- [ ] QA Agent UAT sign-off received

### Operations

- [ ] CI/CD pipeline operational (build, test, deploy)
- [ ] Health check endpoint live
- [ ] Structured logging operational
- [ ] Security checklist complete (Microsoft Agent sign-off)
```

---

## Agents Active in Phase 3

| Agent                    | Role                                                                                          |
| ------------------------ | --------------------------------------------------------------------------------------------- |
| **Architect Agent**      | Leads — owns solution structure, VSA folder layout, YAGNI/KISS enforcement, ARCH log          |
| **Refactoring Expert Agent**         | Leads — drives the refactoring roadmap; selects and sequences catalog entries                 |
| **Clean Code Expert Agent**         | Leads — enforces SOLID, clean code, naming, no over-engineering                               |
| **Microsoft Agent**      | Leads — owns platform migration decisions, EF Core, DI, security, observability               |
| **Product Expert Agent** | Authority — validates that every confirmed business rule is preserved in the new code         |
| **QA Agent**             | Executes end-to-end test suite; provides UAT sign-off                                         |
| **Legacy Code Expert Agent**       | Advisory — ensures new test doubles and seams are clean; reviews any residual legacy patterns |
| **Orchestrator**         | Tracks per-feature migration progress; enforces gate; resolves conflicts                      |

---

## Migration Sequence

Phase 3 proceeds **one vertical slice at a time** — migrating all layers of one feature before moving to the next. This ensures the characterization tests for that feature remain green throughout.

```
For each feature (in priority order — start with lowest coupling):

  Step 1: Domain Layer
    - Extract domain entities (pure C# classes, no framework deps)
    - Identify value objects (Money, OrderStatus, CustomerId)
    - Define domain interfaces (IOrderRepository, etc.)
    - Write/update domain unit tests

  Step 2: Vertical Slice Handler
    - Create Request record, Response record, Validator (plain C# static Validate()), Handler class
    - Handler injects AppDbContext directly — no generic repository (ARCH-003)
    - Register handler in Program.cs via explicit DI
    - Run characterization tests — must still pass

  Step 3: Infrastructure Layer
    - Implement repository with EF Core (or Dapper for complex queries)
    - Create EF Core entity configuration
    - Write integration tests with Testcontainers

  Step 4: Presentation Layer
    - Replace Web Forms page with Razor Page or Minimal API endpoint
    - Wire up DI container
    - Map request → command → response

  Step 5: QA Validation
    - QA Agent executes E2E test cases for this feature
    - All P1 cases must pass before moving to next feature

  Step 6: Product Expert Acceptance
    - Product Expert Agent confirms all business rules in catalogue
      for this feature are met by the new implementation
```

---

## Domain Model Design

### Entity Design Principles

```csharp
// Rich domain entity — behavior in the entity
public class Order
{
    private readonly List<OrderLine> _lines = [];

    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Subtotal => Money.Sum(_lines.Select(l => l.LineTotal));

    private Order() { }  // EF Core constructor

    public static Order Create(CustomerId customerId)
    {
        if (customerId is null) throw new ArgumentNullException(nameof(customerId));
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };
        return order;
    }

    public void AddLine(Product product, int quantity)
    {
        // BR-028: quantity must be positive (from characterization tests)
        if (quantity <= 0) throw new DomainException("Quantity must be greater than zero.");
        _lines.Add(new OrderLine(product, quantity));
    }

    public void Place()
    {
        // BR-005: order must be in Pending to be placed
        if (Status != OrderStatus.Pending)
            throw new DomainException($"Cannot place an order in {Status} status.");
        Status = OrderStatus.Placed;
    }
}
```

### Value Object Design

```csharp
// Money — BR-015: all currency calculations use decimal, never float
public record Money(decimal Amount, string Currency = "USD")
{
    public static Money Zero => new(0m);

    public static Money Sum(IEnumerable<Money> items)
        => items.Aggregate(Zero, (acc, m) => acc + m);

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new DomainException("Cannot add money of different currencies.");
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public Money ApplyPercentage(decimal percentage)
        => new(Amount * (percentage / 100m), Currency);
}
```

---

## Vertical Slice Handler Pattern

> File layout: `Features/Orders/PlaceOrder/` — four files, one folder.
> No MediatR. No generic repository. DbContext used directly (ARCH-003).

```csharp
// Features/Orders/PlaceOrder/PlaceOrderRequest.cs
public record PlaceOrderRequest(int CustomerId, IReadOnlyList<OrderLineRequest> Lines);

// Features/Orders/PlaceOrder/PlaceOrderResponse.cs
public record PlaceOrderResponse(int OrderId, decimal Total);

// Features/Orders/PlaceOrder/PlaceOrderValidator.cs
public static class PlaceOrderValidator
{
    public static IReadOnlyList<string> Validate(PlaceOrderRequest request)
    {
        var errors = new List<string>();
        if (request.Lines is null || request.Lines.Count == 0)
            errors.Add("Order must contain at least one line.");
        if (request.Lines?.Any(l => l.Quantity <= 0) == true)
            errors.Add("All line quantities must be greater than zero.");  // BR-028
        return errors;
    }
}

// Features/Orders/PlaceOrder/PlaceOrderHandler.cs
public class PlaceOrderHandler(AppDbContext db, IEmailService emailService)
{
    public async Task<PlaceOrderResponse> HandleAsync(
        PlaceOrderRequest request, CancellationToken ct = default)
    {
        var errors = PlaceOrderValidator.Validate(request);
        if (errors.Count > 0) throw new ValidationException(errors);

        var customer = await db.Customers.FindAsync([request.CustomerId], ct)
            ?? throw new NotFoundException($"Customer {request.CustomerId} not found.");

        var order = Order.Create(customer.Id);

        foreach (var line in request.Lines)
        {
            var product = await db.Products.FindAsync([line.ProductId], ct)
                ?? throw new NotFoundException($"Product {line.ProductId} not found.");
            order.AddLine(product, line.Quantity);
        }

        // BR-012: preferred customer discount above $200 threshold
        if (customer.Tier == CustomerTier.Preferred && order.Subtotal.Amount > 200m)
            order.ApplyDiscount(10m);

        order.Place();

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);

        await emailService.SendOrderConfirmationAsync(order, customer, ct);  // BR-031

        return new PlaceOrderResponse(order.Id.Value, order.Total.Amount);
    }
}
```

---

## EF Core Integration

```csharp
// Repository implementation
public class EfOrderRepository(AppDbContext context) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default)
        => await context.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task SaveAsync(Order order, CancellationToken ct = default)
    {
        context.Orders.Add(order);
        await context.SaveChangesAsync(ct);
    }
}

// Integration test using Testcontainers
public class OrderRepositoryTests(SqlServerContainer container) : IClassFixture<SqlServerContainer>
{
    [Fact]
    public async Task SaveAsync_PlacedOrder_CanBeRetrievedById()
    {
        await using var context = CreateContext(container.ConnectionString);
        var repo = new EfOrderRepository(context);

        var order = Order.Create(new CustomerId(1));
        order.AddLine(testProduct, quantity: 2);
        order.Place();

        await repo.SaveAsync(order);
        var retrieved = await repo.GetByIdAsync(order.Id);

        Assert.NotNull(retrieved);
        Assert.Equal(OrderStatus.Placed, retrieved.Status);
    }
}
```

---

## Razor Page Migration Pattern

```csharp
// BEFORE: Web Forms code-behind
protected void btnPlaceOrder_Click(object sender, EventArgs e)
{
    // Mixed UI, business, and data access
}

// AFTER: Razor Page PageModel
public class PlaceOrderModel(PlaceOrderHandler handler) : PageModel
{
    [BindProperty]
    public PlaceOrderFormModel Form { get; set; } = new();

    public OrderConfirmationModel? Confirmation { get; private set; }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();

        var command = new PlaceOrderCommand(
            new CustomerId(Form.CustomerId),
            Form.Lines.Select(l => new OrderLineRequest(l.ProductId, l.Quantity)).ToList());

        var result = await handler.HandleAsync(command);

        Confirmation = new OrderConfirmationModel(result.OrderId, result.Total);
        return Page();
    }
}
```

---

## Architecture Conformance Tests

Using `NetArchTest.Rules`:

```csharp
// Framework: xUnit + NetArchTest.Rules. No FluentAssertions.
public class ArchitectureTests
{
    private static readonly Types AllTypes = Types.InSolution();

    [Fact]
    public void Shared_kernel_must_not_depend_on_Features()
    {
        var result = AllTypes.That().ResideInNamespace("MyApp.Shared")
            .Should().NotHaveDependencyOn("MyApp.Features")
            .GetResult();
        Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void Handlers_must_reside_in_Features_namespace()
    {
        var result = AllTypes.That().HaveNameEndingWith("Handler")
            .Should().ResideInNamespace("MyApp.Features")
            .GetResult();
        Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void No_slice_may_depend_on_another_slice()
    {
        // Slices within Features must only import from Shared, never from sibling slices
        var result = AllTypes.That().ResideInNamespace("MyApp.Features.Orders")
            .Should().NotHaveDependencyOn("MyApp.Features.Customers")
            .GetResult();
        Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void Validators_must_be_static_classes()
    {
        var result = AllTypes.That().HaveNameEndingWith("Validator")
            .Should().BeStatic()
            .GetResult();
        Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
    }
}
```

---

## Deliverables

| Deliverable                 | Owner                     | Description                                                 |
| --------------------------- | ------------------------- | ----------------------------------------------------------- |
| Domain project              | Fowler + Clean Code Expert Agents    | Entities, value objects, domain events, domain interfaces   |
| Application project         | Fowler + Clean Code Expert Agents    | Use cases, commands, validators, mappings                   |
| Infrastructure project      | Microsoft Agent           | EF Core, repositories, external services                    |
| Web project                 | Microsoft Agent           | Razor Pages or Minimal API endpoints, middleware, DI wiring |
| Architecture tests          | Martin + Microsoft Agents | NetArchTest rules; must be in CI                            |
| Security checklist          | Microsoft Agent           | All items checked and signed off                            |
| QA executed test suite      | QA Agent                  | All test cases run with Pass/Fail/Blocked results           |
| UAT sign-off                | QA Agent                  | Formal sign-off document                                    |
| Product Expert acceptance   | Product Expert Agent      | All BR-NNN rules validated against new implementation       |
| CI/CD pipeline              | Microsoft Agent           | GitHub Actions or Azure DevOps YAML                         |
| Final architecture document | All agents                | Living document describing what was built and why           |
