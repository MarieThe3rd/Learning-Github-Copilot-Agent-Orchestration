# Phase 2: Test Coverage

## Goal

Discover, name, document, and test every business rule implicit in the legacy codebase. By the end of Phase 2, every significant behavior of the system is pinned by a characterization test and catalogued in the Business Rule Catalogue — ready to protect against regression during Phase 3.

---

## Phase Gate

Phase 2 is complete when **all** of the following are true:

```markdown
- [ ] Every class processed in Phase 1 has been analysed for business rules
- [ ] Business Rule Catalogue is complete (no "Unknown" status rules remain)
- [ ] All "Suspected" rules resolved by human review
- [ ] All "Disputed" rules resolved by human review
- [ ] Characterization test suite is fully green in CI
- [ ] Code coverage on business logic classes ≥ 90%
- [ ] All happy-path scenarios have at least one test
- [ ] All known edge cases have at least one test
- [ ] All known error paths have at least one test
- [ ] QA Agent has workflow maps for all user-facing features
- [ ] QA Agent has draft test cases for all P1 and P2 business rules
- [ ] Product Expert Agent has reviewed and approved the Business Rule Catalogue
- [ ] Orchestrator has reviewed and approved phase transition
```

---

## Agents Active in Phase 2

| Agent                    | Role                                                                            |
| ------------------------ | ------------------------------------------------------------------------------- |
| **Legacy Code Expert Agent**       | Leads — writes characterization tests; infers business rules from code behavior |
| **Product Expert Agent** | Leads — receives inferred rules; validates, names, and catalogues them          |
| **QA Agent**             | Leads — produces workflow maps and draft end-to-end test cases                  |
| **Refactoring Expert Agent**         | Advisory — helps identify structural patterns that reveal business rules        |
| **Clean Code Expert Agent**         | Advisory — ensures test and rule names use business language                    |
| **Microsoft Agent**      | Advisory — recommends test framework setup and CI tooling                       |
| **Orchestrator**         | Tracks coverage progress; routes anomalies; enforces gate                       |

---

## The Characterization Test Process

A characterization test captures **what the code currently does** — not what it should do.

### Workflow

```
1. Select a class from the Phase 1 registry (start with highest-value business classes)

2. Legacy Code Expert Agent exercises the class with known inputs using the fake dependencies
   from Phase 1

3. Record all observed outputs — return values, exceptions thrown, side effects
   via sensing variables on fakes

4. Write a test that asserts those exact outputs
   - Even if the output appears wrong
   - Add a comment flagging suspected bugs; do not fix them

5. Hand the observed behavior to the Product Expert Agent with:
   - The class and method name
   - The input conditions
   - The observed output
   - Any anomalies noticed

6. Product Expert Agent creates or updates a Business Rule Catalogue entry

7. Legacy Code Expert Agent adds more test cases for edge cases identified by Product Expert

8. Repeat until class coverage ≥ 90%
```

---

## Characterization Test Patterns (BDD Naming)

> **Framework**: xUnit only. No FluentAssertions. No SpecFlow. No MSTest.
>
> **Naming convention**: Nested classes, not underscored method names.
> Tests must survive internal refactoring — they break only when genuine behavior changes.
> Every test must reference the Business Rule Catalogue entry it protects (BR-NNN).

### Naming Structure

```
public class {FeatureName}
{
    public class Given_{context_in_snake_case}
    {
        [Fact]
        public void Then_{expected_outcome_in_snake_case}() { }
    }
}
```

### Pattern 1: Return Value Test (BR reference mandatory)

```csharp
public class InvoiceTotalCalculation
{
    public class Given_standard_customer_with_one_overdue_invoice
    {
        [Fact]
        public void Then_total_includes_late_fee()
        {
            // Arrange
            var clock = new FakeSystemClock(new DateTime(2026, 1, 15));
            var repo = new FakeInvoiceRepository(new[]
            {
                new Invoice { Amount = 500m, DueDate = new DateTime(2025, 12, 1), IsPaid = false }
            });
            var sut = new BillingCalculator(repo, clock);

            // Act
            var result = sut.CalculateTotalDue(customerId: 42);

            // Assert — captures current behavior as-is (BR-017: late fee 1.5% at ≥45 days)
            Assert.Equal(507.50m, result.TotalDue);
        }
    }
}
```

### Pattern 2: Side Effect Sensing Test

```csharp
public class OrderProcessing
{
    public class Given_valid_order
    {
        [Fact]
        public void Then_confirmation_email_is_sent()
        {
            // Arrange
            var emailService = new CapturingEmailService();  // records all sent emails
            var sut = new OrderProcessor(
                new FakeOrderRepository(),
                emailService,
                new FakeSystemClock());

            // Act
            sut.ProcessOrder(new OrderRequest { CustomerId = 1, Items = testItems });

            // Assert — verify the side effect occurred (BR-031: confirmation email on order)
            Assert.Single(emailService.SentEmails);
            Assert.Equal("order-confirmation@system.com", emailService.SentEmails[0].From);
            // NOTE: From address appears hardcoded — may be a configuration smell.
        }
    }
}
```

### Pattern 3: Exception Characterization Test

```csharp
public class OrderProcessing
{
    public class Given_order_with_zero_quantity_item
    {
        [Fact]
        public void Then_ArgumentException_is_thrown()
        {
            // Arrange
            var sut = new OrderProcessor(
                new FakeOrderRepository(),
                new FakeEmailService(),
                new FakeSystemClock());

            // Act & Assert — document that the exception IS thrown (BR-028)
            var ex = Assert.Throws<ArgumentException>(
                () => sut.ProcessOrder(new OrderRequest { Items = [ new Item { Quantity = 0 } ] }));

            // Capture exact current message — change triggers BR-028 review
            Assert.Contains("quantity", ex.Message, StringComparison.OrdinalIgnoreCase);
        }
    }
}
```

### Pattern 4: Boundary / Theory Test

```csharp
public class PreferredCustomerDiscount
{
    public class Given_varying_customer_tier_and_order_amount
    {
        [Theory]
        [InlineData("Preferred", 300.00, 270.00)]   // 10% discount applied     (BR-012)
        [InlineData("Preferred", 200.00, 200.00)]   // exactly $200 — boundary, no discount
        [InlineData("Standard",  300.00, 300.00)]   // no discount for standard customers
        [InlineData(null,        300.00, 300.00)]   // null tier treated as Standard
        public void Then_correct_total_is_returned(
            string? tier, decimal subtotal, decimal expectedTotal)
        {
            var sut = new OrderPricingCalculator(new FakeCustomerRepository(tier));
            var result = sut.CalculateTotal(new Order { Subtotal = subtotal });
            Assert.Equal(expectedTotal, result.Total);
        }
    }
}
```

### Refactor-Proof Rule

| Rule                          | Detail                                                                   |
| ----------------------------- | ------------------------------------------------------------------------ |
| No method name in test name   | Tests survive `ExtractMethod`, `Rename`, `MoveClass` refactorings        |
| Describe the business context | Class name = feature; Given = precondition; Then = observable outcome    |
| One BR per test class         | Each outer class maps to at most one business rule cluster               |
| Assert on behavior            | Assert outputs and side effects, never on internal state                 |
| No FluentAssertions           | Use `Assert.Equal`, `Assert.Throws`, `Assert.Single`, `Assert.True` only |

---

## Business Rule Inference Techniques

Legacy Code Expert Agent uses these techniques to surface hidden business rules:

### Technique 1: Extract Conditionals

Every `if`, `switch`, and ternary in business logic is a potential business rule.

```
if (customer.Tier == "Preferred" && order.Subtotal > 200)
  → BR-012: Preferred customer discount applies above $200 threshold
```

### Technique 2: Follow the Data

Trace each output value back through the computation. Every coefficient, constant, and formula is a business rule.

```
fee = amount * 0.015
  → BR-017: Late fee rate is 1.5%

if (daysPastDue > 30)
  → BR-018: Late fee applies after 30 days past due
```

### Technique 3: Analyse the Database Schema

Constraints, foreign keys, default values, and NOT NULL columns encode business rules.

```
Orders.Status NOT NULL DEFAULT 'Pending'
  → BR-005: All new orders start in Pending status

CustomerTier CHECK (Tier IN ('Standard', 'Preferred', 'VIP'))
  → BR-001: Only three valid customer tiers exist
```

### Technique 4: Read the Stored Procedures

Stored procedures often contain business rule logic that is invisible to the application code.

### Technique 5: Boundary Probing

Test at n-1, n, n+1 for every threshold found.

```
threshold at 200 → test at 199.99, 200.00, 200.01
threshold at 30 days → test at 29 days, 30 days, 31 days
```

---

## Business Rule Catalogue

Maintained by the Product Expert Agent. Every rule follows the schema defined in [agents/product-expert-agent.md](../agents/product-expert-agent.md).

### Rule Status Definitions

| Status           | Meaning                                     | Action Required                           |
| ---------------- | ------------------------------------------- | ----------------------------------------- |
| **Confirmed**    | Rule is clear from code and consistent      | Ensure test exists                        |
| **Suspected**    | Rule inferred but intent unclear            | Escalate to human                         |
| **Disputed**     | Code is internally inconsistent             | Escalate to human                         |
| **Bug**          | Behavior is clearly wrong (human confirmed) | Document; fix in Phase 3 with test update |
| **Accepted Bug** | Wrong but intentional (human confirmed)     | Document; preserve through Phase 3        |

---

## Coverage Tracking

The Orchestrator tracks coverage per class:

```markdown
## Coverage Report — BillingCalculator

| Method            | Scenarios Covered | Rule Coverage                  | Status                           |
| ----------------- | ----------------- | ------------------------------ | -------------------------------- |
| CalculateTotalDue | 8 tests           | BR-012, BR-017, BR-018, BR-019 | Complete                         |
| ApplyLateFee      | 4 tests           | BR-017, BR-018                 | Complete                         |
| CalculateDiscount | 5 tests           | BR-012, BR-013                 | Complete                         |
| GenerateInvoice   | 2 tests           | BR-031                         | Incomplete — missing error paths |

Overall class coverage: 83% — 2 scenarios still needed
```

---

## QA Agent Workflow Maps

In parallel with characterization testing, the QA Agent maps all user-facing workflows by reading `.aspx` pages and their code-behind event handlers.

Each workflow map captures:

- Entry URL
- User roles with access
- All form fields and their validation rules
- All possible outcomes (success, failure, error)
- Business rules active in each path

These maps drive Phase 2 draft test cases and Phase 3 acceptance testing.

---

## Deliverables

| Deliverable                   | Owner                | Description                                    |
| ----------------------------- | -------------------- | ---------------------------------------------- |
| `business-rules/catalogue.md` | Product Expert Agent | Complete Business Rule Catalogue               |
| `tests/characterization/`     | Legacy Code Expert Agent       | Full characterization test suite               |
| `tests/coverage-report.md`    | Orchestrator         | Per-class coverage summary                     |
| `anomalies.md`                | Product Expert Agent | All unresolved anomalies and their disposition |
| `workflow-maps/`              | QA Agent             | User-facing workflow maps per feature area     |
| `test-cases/phase-2-draft/`   | QA Agent             | Draft E2E test cases for all P1/P2 rules       |
| Phase 3 gate checklist        | Orchestrator         | Completed and signed off                       |

---

## Safety Rules

> These are non-negotiable constraints for Phase 2.

1. **No behavior changes** — Phase 2 only observes and documents
2. **Tests capture current behavior, not desired behavior** — do not fix bugs during characterization
3. **Every anomaly escalated, not silently fixed** — the Product Expert Agent decides intent
4. **Test suite must run in CI** — no tests that only work locally
5. **Tests must be deterministic** — no randomness, no time-dependency without `ISystemClock`
6. **Tests must be independent** — no shared state between tests; each test arranges its own data
