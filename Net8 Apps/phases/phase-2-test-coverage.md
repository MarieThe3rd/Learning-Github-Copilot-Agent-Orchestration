# Phase 2: Test Coverage

## Goal

Write unit, integration, and component tests that pin business behavior. Discover, name, and document every business rule. By the end of Phase 2, every significant behavior of the system is covered by tests and catalogued in the Business Rule Catalogue.

---

## Phase Gate

Phase 2 is complete when **all** of the following are true:

```markdown
- [ ] All coupling hotspots from Phase 1 have interfaces extracted and seams introduced
- [ ] Business Rule Catalogue is complete — no "Unknown" or "Suspected" rules remain
- [ ] All "Disputed" rules resolved by human review
- [ ] Code coverage on business logic classes ≥ 80%
- [ ] All happy-path scenarios have at least one test
- [ ] All known edge cases have at least one test
- [ ] All known error paths have at least one test
- [ ] All tests green in CI
- [ ] For Blazor apps: all non-trivial components have bUnit tests
- [ ] For console apps: all workers and services have integration tests via TestHost
- [ ] QA Agent has workflow maps for all user-facing features
- [ ] QA Agent has draft manual test cases for all P1 and P2 business rules
- [ ] Product Expert Agent has reviewed and approved the Business Rule Catalogue
- [ ] Orchestrator has reviewed and approved phase transition
```

---

## Agents Active in Phase 2

| Agent                         | Role                                                                                  |
| ----------------------------- | ------------------------------------------------------------------------------------- |
| **Blazor Expert Agent**       | Leads (Blazor apps) — writes bUnit component tests; extracts component interfaces     |
| **Console App Expert Agent**  | Leads (console apps) — writes TestHost integration tests; extracts service interfaces |
| **Product Expert Agent**      | Leads — validates, names, and catalogues business rules; approves catalogue           |
| **QA Agent**                  | Leads — produces workflow maps and draft end-to-end test cases                        |
| **Refactoring Expert Agent**  | Advisory — helps identify structural patterns that reveal business rules              |
| **Clean Code Expert Agent**   | Advisory — ensures test and rule names use business language                          |
| **Microsoft Practices Agent** | Advisory — recommends test framework setup, mocking library, CI tooling               |
| **Code Historian Agent**      | Records CHR-NNN entries; guards the immutable BR catalogue                            |
| **Orchestrator**              | Tracks coverage progress; routes anomalies; enforces gate                             |

---

## Test Types by App Type

### Console App Tests

**Unit tests** (xUnit + NSubstitute or Moq):

- Test each service class in isolation
- All dependencies injected as interfaces (introduced in Phase 1 or Phase 2 setup)
- No database, no file system, no network in unit tests

**Integration tests** (xUnit + `Microsoft.Extensions.Hosting.Testing` or `WebApplicationFactory`):

- Use `TestHost` / `HostBuilder` with test service registrations
- Substitute real infrastructure (DB, email, HTTP) with test doubles
- Test full worker execution cycles

**Example — unit test for a service:**

```csharp
public class OrderCalculationServiceTests
{
    [Fact]
    public void Calculate_AppliesDiscountForOrdersOver200()
    {
        // BR-007: orders over $200 receive 10% discount
        var service = new OrderCalculationService();
        var result = service.Calculate(new Order { Subtotal = 250m });
        result.Discount.Should().Be(25m);
    }
}
```

**Example — integration test for a worker:**

```csharp
public class OrderProcessingWorkerTests
{
    [Fact]
    public async Task Worker_ProcessesPendingOrdersOnEachCycle()
    {
        var orderRepo = Substitute.For<IOrderRepository>();
        var emailService = Substitute.For<IEmailService>();
        orderRepo.GetPendingOrders().Returns([new Order { Id = 1 }]);

        using var host = Host.CreateDefaultBuilder()
            .ConfigureServices(services =>
            {
                services.AddSingleton(orderRepo);
                services.AddSingleton(emailService);
                services.AddHostedService<OrderProcessingWorker>();
            })
            .Build();

        await host.StartAsync();
        await Task.Delay(TimeSpan.FromSeconds(2));
        await host.StopAsync();

        await orderRepo.Received(1).GetPendingOrders();
    }
}
```

### Blazor App Tests (bUnit)

**Component unit tests** (bUnit + xUnit):

- Render components in isolation using bUnit's `TestContext`
- Inject fake services
- Assert rendered markup, user interactions, and state changes

**Example — bUnit component test:**

```csharp
public class OrderListTests : TestContext
{
    [Fact]
    public void OrderList_ShowsEmptyMessage_WhenNoOrders()
    {
        // BR-003: when no orders exist, show "No orders found"
        var orderService = Substitute.For<IOrderService>();
        orderService.GetOrders().Returns([]);
        Services.AddSingleton(orderService);

        var cut = RenderComponent<OrderList>();
        cut.Find(".empty-message").TextContent.Should().Be("No orders found");
    }
}
```

---

## Business Rule Discovery Process

For each class or component:

1. **Expert agent** exercises it with known inputs and records all outputs
2. **Product Expert Agent** names the rule in business language
3. **Product Expert Agent** assigns a BR-NNN identifier
4. **Human** reviews any "Suspected" or "Disputed" rules before they are locked
5. A test is written that explicitly references the BR-NNN in its name or comment

### Business Rule Catalogue Entry Format

```
BR-007
Name: Bulk order discount
Status: Confirmed
Rule: Orders with a subtotal ≥ $200 receive a 10% discount applied before tax
Source: OrderCalculationService.Calculate(), line 84
Test: OrderCalculationServiceTests.Calculate_AppliesDiscountForOrdersOver200
Agent: Console App Expert Agent
Approved by: Product Expert Agent
```

---

## Interface Extraction (Where Still Needed)

If a coupling hotspot from Phase 1 was deferred, Phase 2 begins by extracting the interface before writing tests. This is the only structural change permitted in Phase 2.

Pattern:

1. Console App or Blazor Expert proposes minimal interface
2. Refactoring Expert approves as behavior-preserving
3. Clean Code Expert approves naming
4. Extract, register in DI, commit: `[PHASE-2] [CHR-NNN] Extract IOrderRepository from OrderRepository`

---

## Characterization Tests

When exercising a code path reveals behavior that is unclear, suspected to be wrong, or is complex enough that it must not regress through Phase 3, write a **characterization test** alongside the regular unit tests.

A characterization test documents what the code _currently does_ — not what it _should_ do. It is the safety net that lets Phase 3 restructure the code confidently.

```csharp
[Fact]
public void OrderCalculationService_Calculate_CharacterizesCurrentDiscountBehavior()
{
    // CHARACTERIZATION TEST — documents existing behavior.
    // Do NOT change the assertion without Product Expert Agent approval and a BR-NNN update.
    var options = Options.Create(new BillingOptions { LateFeePercentage = 1.5m });
    var sut = new OrderCalculationService(options);

    var result = sut.Calculate(new Order { Subtotal = 250m, Region = "EU" });

    Assert.Equal(302.50m, result.TotalDue);
    // NOTE: discount applies before tax — flagged for domain review. See ESC-012.
}
```

**Rules for characterization tests:**

- Mark the test body with `// CHARACTERIZATION TEST` so its intent is visible
- Never assert “what it should do” — assert the exact current output, even if it looks wrong
- Add a `// NOTE:` comment whenever something seems incorrect — flag it for Product Expert Agent
- Any change to a characterization test assertion must be approved by Product Expert Agent and result in a BR-NNN update or a confirmed bug entry
