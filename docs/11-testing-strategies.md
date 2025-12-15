# Chapter 11: Testing Strategies

## Introduction

Clean architecture isn't just about organizing code—it's about making your application **testable**. When your domain logic is pure, your repositories are abstractions, and your dependencies are inverted, testing becomes straightforward, fast, and reliable.

In this chapter, you'll learn how to test each layer of your architecture effectively. We'll explore how clean separation enables different testing strategies for different layers: pure unit tests for domain logic, service tests with test doubles, and integration tests for infrastructure. You'll see concrete examples in both TypeScript and Elixir, learn about property-based testing, and discover patterns that make your tests maintainable.

### Why Clean Architecture Makes Testing Easier

Traditional architectures make testing painful:
- Business logic tangled with frameworks requires full application setup
- Database dependencies slow down tests
- Coupled code means tests break when implementation details change
- Mock explosion makes tests brittle and hard to maintain

Clean architecture solves these problems through separation:
- **Domain Layer**: Pure functions and classes with no dependencies—test in milliseconds
- **Application Layer**: Dependencies injected through interfaces—swap real implementations for test doubles
- **Infrastructure Layer**: Isolated adapters—test database queries without testing business logic

### The Testing Pyramid for Clean Architecture

```
         /\
        /  \      E2E Tests (Few)
       /────\     Full system, real infrastructure
      /      \    Slow but high confidence
     /────────\   
    /          \  Integration Tests (Some)
   /────────────\ Test adapters work correctly
  /              \ Database, APIs, file system
 /────────────────\ 
/──────────────────\ Unit Tests (Many)
  Domain + App       Fast, isolated, deterministic
    Services         No external dependencies
```

**The strategy:**
- **Many unit tests** for domain logic and application services (milliseconds per test)
- **Some integration tests** for repositories and external integrations (seconds per test)
- **Few E2E tests** for critical user flows (minutes per test)

### What You'll Learn

- How to structure tests for each architectural layer
- Testing domain models without dependencies
- Using in-memory repositories for application service tests
- Integration testing strategies for real infrastructure
- React component testing with dependency injection
- Phoenix controller testing with test configuration
- Test builders and factories for maintainable tests
- Property-based testing for edge cases
- Common pitfalls and how to avoid them

Let's start by examining what makes tests painful and how clean architecture addresses these problems.

## The Problem: Testing Pain Points

Before exploring solutions, let's understand what makes tests painful in traditional architectures.

### Problem 1: Tests Requiring Full Database Setup

**The symptom:**
```typescript
// ❌ BAD: Test requires real database
describe('OrderService', () => {
  beforeEach(async () => {
    await setupTestDatabase();
    await runMigrations();
    await seedTestData();
  });

  afterEach(async () => {
    await cleanupDatabase();
    await closeConnections();
  });

  it('places order', async () => {
    // Test takes 2-3 seconds because of database
    const service = new OrderService(realDatabase);
    await service.placeOrder(data);
    // ...
  });
});
```

**Why it's painful:**
- Tests take seconds instead of milliseconds
- Flaky failures from database state
- Complex setup and teardown
- Can't run tests in parallel
- Requires test database infrastructure

### Problem 2: Tests Coupled to Framework

**The symptom:**
```elixir
# ❌ BAD: Business logic requires Phoenix context
defmodule MyApp.OrderTest do
  use MyAppWeb.ConnCase  # Brings in entire Phoenix stack

  test "calculates order total", %{conn: conn} do
    # Need conn even though this is pure calculation
    order = %Order{price: 1000, quantity: 3}
    # Business logic buried in controller
    response = conn
      |> post("/orders", order_params)
      |> json_response(200)
    
    assert response["total"] == 3000
  end
end
```

**Why it's painful:**
- Can't test business logic without framework
- Test setup is heavyweight
- Tests are slow
- Unclear what you're actually testing

### Problem 3: Slow Test Suites

**The symptom:**
```bash
$ npm test
Running 150 tests...
[██████████░░░░░░░░░░] 50% (3 minutes elapsed)
```

**Why it happens:**
- Every test hits database
- Every test makes HTTP calls
- Every test renders full component tree
- Tests run sequentially because of shared state

**The cost:**
- Developers skip running tests
- CI/CD takes forever
- Slow feedback loop
- Reduced productivity

### Problem 4: Brittle Tests Breaking on Refactoring

**The symptom:**
```typescript
// Refactor: Change internal implementation
class OrderService {
  // Before: calculateDiscount()
  // After: applyPricing() - better name, same logic
  private applyPricing(order: Order): Money {
    // Same logic, different method name
  }
}

// Result: 47 tests fail
// All tests coupled to internal method name
```

**Why it's painful:**
- Tests verify implementation, not behavior
- Refactoring breaks unrelated tests
- Fear of changing code
- Tests become maintenance burden

Clean architecture addresses all these problems through proper separation of concerns. Let's see how.

## The Three Testing Layers

Clean architecture gives us three distinct layers, each with its own testing strategy:

### 1. Domain Layer: Pure Unit Tests

**What to test:**
- Entity creation and validation
- Value object behavior
- Business rule enforcement
- Domain calculations

**Testing approach:**
- No mocks or stubs needed
- Pure input/output testing
- Fast (milliseconds)
- Deterministic

**Example structure:**
```typescript
describe('Order', () => {
  it('calculates total correctly', () => {
    // Pure domain test - no dependencies
    const order = Order.create({...});
    expect(order.totalAmount).toBe(expectedValue);
  });
});
```

### 2. Application Layer: Test Doubles for Dependencies

**What to test:**
- Use case orchestration
- Service coordination
- Transaction boundaries
- Error handling

**Testing approach:**
- Use in-memory repositories
- Use fake implementations for external services
- Focus on workflow logic
- Still fast (milliseconds)

**Example structure:**
```typescript
describe('PlaceOrderService', () => {
  it('coordinates order placement', async () => {
    // Use test doubles for all dependencies
    const orderRepo = new InMemoryOrderRepository();
    const paymentGateway = new FakePaymentGateway();
    const service = new PlaceOrderService(orderRepo, paymentGateway);
    
    await service.execute(command);
    
    // Verify workflow executed correctly
    expect(orderRepo.orders.size).toBe(1);
    expect(paymentGateway.charges.length).toBe(1);
  });
});
```

### 3. Infrastructure Layer: Integration Tests

**What to test:**
- Database queries work correctly
- API calls format correctly
- File system operations succeed
- External service integrations

**Testing approach:**
- Use real infrastructure (test database, test APIs)
- Slower but necessary
- Fewer tests (trust the adapters)
- Test contracts, not business logic

**Example structure:**
```typescript
describe('PostgresOrderRepository', () => {
  it('saves and retrieves order', async () => {
    // Real database test
    const repo = new PostgresOrderRepository(testDb);
    await repo.save(order);
    
    const retrieved = await repo.findById(order.id);
    expect(retrieved).toEqual(order);
  });
});
```

Now let's see concrete examples for each layer.

## Domain Layer Testing

Domain tests are the easiest and fastest—pure functions with no dependencies.

### TypeScript Domain Testing

```typescript
// domain/Order.test.ts
import { Order, OrderId } from './Order';
import { Email } from './Email';
import { Money } from './Money';

describe('Order', () => {
  describe('creation', () => {
    it('creates valid order', () => {
      const result = Order.create({
        id: OrderId.generate(),
        ebookId: 'ebook-123',
        customerEmail: 'customer@example.com',
        quantity: 2,
        pricePerUnit: Money.fromAmount(19.99, 'USD')
      });

      expect(result.isOk()).toBe(true);
      expect(result.value.quantity).toBe(2);
    });

    it('rejects invalid email', () => {
      const result = Order.create({
        id: OrderId.generate(),
        ebookId: 'ebook-123',
        customerEmail: 'invalid-email',
        quantity: 1,
        pricePerUnit: Money.fromAmount(19.99, 'USD')
      });

      expect(result.isErr()).toBe(true);
      expect(result.error.message).toContain('email');
    });

    it('rejects zero quantity', () => {
      const result = Order.create({
        id: OrderId.generate(),
        ebookId: 'ebook-123',
        customerEmail: 'customer@example.com',
        quantity: 0,
        pricePerUnit: Money.fromAmount(19.99, 'USD')
      });

      expect(result.isErr()).toBe(true);
      expect(result.error.message).toContain('quantity');
    });

    it('rejects negative price', () => {
      const result = Order.create({
        id: OrderId.generate(),
        ebookId: 'ebook-123',
        customerEmail: 'customer@example.com',
        quantity: 1,
        pricePerUnit: Money.fromAmount(-10, 'USD')
      });

      expect(result.isErr()).toBe(true);
      expect(result.error.message).toContain('price');
    });
  });

  describe('business rules', () => {
    it('calculates total correctly', () => {
      const order = Order.create({
        id: OrderId.generate(),
        ebookId: 'ebook-123',
        customerEmail: 'customer@example.com',
        quantity: 3,
        pricePerUnit: Money.fromAmount(19.99, 'USD')
      }).value;

      const total = order.calculateTotal();

      expect(total.amount).toBe(5997); // 19.99 * 3 * 100 cents
      expect(total.currency).toBe('USD');
    });

    it('applies discount correctly', () => {
      const order = Order.create({
        id: OrderId.generate(),
        ebookId: 'ebook-123',
        customerEmail: 'customer@example.com',
        quantity: 5,
        pricePerUnit: Money.fromAmount(20, 'USD')
      }).value;

      // Business rule: 10% discount for orders of 5 or more
      order.applyBulkDiscount();

      expect(order.calculateTotal().amount).toBe(9000); // $100 - 10% = $90
    });

    it('prevents quantity changes after confirmation', () => {
      const order = Order.create({
        id: OrderId.generate(),
        ebookId: 'ebook-123',
        customerEmail: 'customer@example.com',
        quantity: 2,
        pricePerUnit: Money.fromAmount(19.99, 'USD')
      }).value;

      order.confirm();
      const result = order.changeQuantity(5);

      expect(result.isErr()).toBe(true);
      expect(result.error.message).toContain('confirmed');
      expect(order.quantity).toBe(2); // Unchanged
    });
  });

  describe('value objects', () => {
    it('Email validates format', () => {
      expect(Email.create('valid@example.com').isOk()).toBe(true);
      expect(Email.create('invalid').isErr()).toBe(true);
      expect(Email.create('').isErr()).toBe(true);
    });

    it('Email comparison works', () => {
      const email1 = Email.create('test@example.com').value;
      const email2 = Email.create('test@example.com').value;
      const email3 = Email.create('other@example.com').value;

      expect(email1.equals(email2)).toBe(true);
      expect(email1.equals(email3)).toBe(false);
    });

    it('Money arithmetic works', () => {
      const m1 = Money.fromAmount(10, 'USD');
      const m2 = Money.fromAmount(5, 'USD');

      expect(m1.add(m2).amount).toBe(1500);
      expect(m1.subtract(m2).amount).toBe(500);
      expect(m1.multiply(2).amount).toBe(2000);
    });
  });
});
```

**Key points:**
- ✅ No mocks, no stubs—pure input/output
- ✅ Fast—entire suite runs in milliseconds
- ✅ Comprehensive—test all business rules
- ✅ Maintainable—tests describe business behavior

### Elixir Domain Testing

```elixir
# test/shop/domain/order_test.exs
defmodule Shop.Domain.OrderTest do
  use ExUnit.Case, async: true

  alias Shop.Domain.{Order, OrderId, Email, Money}

  describe "new/5" do
    test "creates valid order" do
      assert {:ok, order} = Order.new(
        OrderId.generate(),
        "ebook-123",
        "customer@example.com",
        2,
        Money.new(1999, :USD)
      )

      assert order.quantity == 2
      assert order.ebook_id == "ebook-123"
    end

    test "rejects invalid email" do
      assert {:error, :invalid_email} = Order.new(
        OrderId.generate(),
        "ebook-123",
        "invalid-email",
        1,
        Money.new(1999, :USD)
      )
    end

    test "rejects zero quantity" do
      assert {:error, :invalid_quantity} = Order.new(
        OrderId.generate(),
        "ebook-123",
        "customer@example.com",
        0,
        Money.new(1999, :USD)
      )
    end

    test "rejects negative price" do
      assert {:error, :invalid_price} = Order.new(
        OrderId.generate(),
        "ebook-123",
        "customer@example.com",
        1,
        Money.new(-1000, :USD)
      )
    end
  end

  describe "calculate_total/1" do
    test "calculates total correctly" do
      {:ok, order} = Order.new(
        OrderId.generate(),
        "ebook-123",
        "customer@example.com",
        3,
        Money.new(1999, :USD)
      )

      total = Order.calculate_total(order)

      assert total.amount == 5997
      assert total.currency == :USD
    end

    test "applies bulk discount for large orders" do
      {:ok, order} = Order.new(
        OrderId.generate(),
        "ebook-123",
        "customer@example.com",
        5,
        Money.new(2000, :USD)
      )

      # Business rule: 10% discount for 5+ items
      order = Order.apply_bulk_discount(order)
      total = Order.calculate_total(order)

      assert total.amount == 9000  # $100 - 10% = $90
    end
  end

  describe "confirm/1" do
    test "prevents quantity changes after confirmation" do
      {:ok, order} = Order.new(
        OrderId.generate(),
        "ebook-123",
        "customer@example.com",
        2,
        Money.new(1999, :USD)
      )

      confirmed_order = Order.confirm(order)
      result = Order.change_quantity(confirmed_order, 5)

      assert {:error, :order_already_confirmed} = result
      assert confirmed_order.quantity == 2  # Unchanged
    end
  end

  describe "value objects" do
    test "Email validates format" do
      assert {:ok, _email} = Email.new("valid@example.com")
      assert {:error, :invalid_format} = Email.new("invalid")
      assert {:error, :invalid_format} = Email.new("")
    end

    test "Email equality works" do
      {:ok, email1} = Email.new("test@example.com")
      {:ok, email2} = Email.new("test@example.com")
      {:ok, email3} = Email.new("other@example.com")

      assert Email.equals?(email1, email2)
      refute Email.equals?(email1, email3)
    end

    test "Money arithmetic works" do
      m1 = Money.new(1000, :USD)
      m2 = Money.new(500, :USD)

      assert Money.add(m1, m2).amount == 1500
      assert Money.subtract(m1, m2).amount == 500
      assert Money.multiply(m1, 2).amount == 2000
    end
  end
end
```

**Key points:**
- ✅ `async: true`—tests run in parallel
- ✅ Pattern matching on results
- ✅ No dependencies—just pure functions
- ✅ Fast feedback—milliseconds per test

## Application Layer Testing

Application services orchestrate use cases. Test them with in-memory implementations of dependencies.

### TypeScript Application Service Testing

```typescript
// application/services/PlaceOrderService.test.ts
import { PlaceOrderService } from './PlaceOrderService';
import { PlaceOrderCommand } from '../commands/PlaceOrderCommand';
import { InMemoryOrderRepository } from '../../adapters/persistence/InMemoryOrderRepository';
import { InMemoryEbookRepository } from '../../adapters/persistence/InMemoryEbookRepository';
import { FakePaymentGateway } from '../../adapters/payment/FakePaymentGateway';
import { Ebook, Money, OrderId } from '../../domain';

describe('PlaceOrderService', () => {
  let service: PlaceOrderService;
  let orderRepo: InMemoryOrderRepository;
  let ebookRepo: InMemoryEbookRepository;
  let paymentGateway: FakePaymentGateway;

  beforeEach(() => {
    // Setup test doubles
    orderRepo = new InMemoryOrderRepository();
    ebookRepo = new InMemoryEbookRepository();
    paymentGateway = new FakePaymentGateway();

    // Inject test doubles into service
    service = new PlaceOrderService(
      orderRepo,
      ebookRepo,
      paymentGateway
    );

    // Seed test data
    const ebook = Ebook.create({
      id: 'ebook-123',
      title: 'Clean Architecture',
      price: Money.fromAmount(29.99, 'USD')
    }).value;
    ebookRepo.save(ebook);
  });

  describe('successful order placement', () => {
    it('creates order with correct data', async () => {
      const command = new PlaceOrderCommand(
        'ebook-123',
        'customer@example.com',
        2
      );

      const result = await service.execute(command);

      expect(result.isOk()).toBe(true);
      
      const orders = await orderRepo.findAll();
      expect(orders.length).toBe(1);
      expect(orders[0].quantity).toBe(2);
      expect(orders[0].ebookId).toBe('ebook-123');
    });

    it('charges correct amount', async () => {
      const command = new PlaceOrderCommand(
        'ebook-123',
        'customer@example.com',
        3
      );

      await service.execute(command);

      expect(paymentGateway.charges.length).toBe(1);
      expect(paymentGateway.charges[0].amount.toNumber()).toBe(89.97);
    });

    it('returns order id', async () => {
      const command = new PlaceOrderCommand(
        'ebook-123',
        'customer@example.com',
        1
      );

      const result = await service.execute(command);

      expect(result.isOk()).toBe(true);
      expect(result.value).toBeInstanceOf(OrderId);
    });
  });

  describe('failure scenarios', () => {
    it('fails when ebook not found', async () => {
      const command = new PlaceOrderCommand(
        'nonexistent-ebook',
        'customer@example.com',
        1
      );

      const result = await service.execute(command);

      expect(result.isErr()).toBe(true);
      expect(result.error.message).toContain('Ebook not found');
      
      // Verify no order was created
      const orders = await orderRepo.findAll();
      expect(orders.length).toBe(0);
    });

    it('fails when payment is declined', async () => {
      paymentGateway.setShouldDecline(true);

      const command = new PlaceOrderCommand(
        'ebook-123',
        'customer@example.com',
        1
      );

      const result = await service.execute(command);

      expect(result.isErr()).toBe(true);
      expect(result.error.message).toContain('Payment failed');
      
      // Verify no order was created
      const orders = await orderRepo.findAll();
      expect(orders.length).toBe(0);
    });

    it('handles invalid command data', async () => {
      const command = new PlaceOrderCommand(
        'ebook-123',
        'invalid-email',
        1
      );

      const result = await service.execute(command);

      expect(result.isErr()).toBe(true);
      expect(result.error.message).toContain('email');
    });

    it('prevents negative quantities', async () => {
      const command = new PlaceOrderCommand(
        'ebook-123',
        'customer@example.com',
        -5
      );

      const result = await service.execute(command);

      expect(result.isErr()).toBe(true);
    });
  });

  describe('edge cases', () => {
    it('handles concurrent order placement', async () => {
      const commands = Array.from({ length: 10 }, (_, i) => 
        new PlaceOrderCommand(
          'ebook-123',
          `customer${i}@example.com`,
          1
        )
      );

      const results = await Promise.all(
        commands.map(cmd => service.execute(cmd))
      );

      const successCount = results.filter(r => r.isOk()).length;
      expect(successCount).toBe(10);

      const orders = await orderRepo.findAll();
      expect(orders.length).toBe(10);
    });

    it('handles large quantities', async () => {
      const command = new PlaceOrderCommand(
        'ebook-123',
        'customer@example.com',
        1000
      );

      const result = await service.execute(command);

      expect(result.isOk()).toBe(true);
      expect(paymentGateway.charges[0].amount.toNumber()).toBe(29990);
    });
  });
});
```

**Key benefits:**
- ✅ No real database—tests run in milliseconds
- ✅ No real payment gateway—no external calls
- ✅ Full control over test scenarios
- ✅ Can simulate any failure condition

### In-Memory Test Repository

```typescript
// adapters/persistence/InMemoryOrderRepository.ts
import { OrderRepository } from '../../domain/ports/OrderRepository';
import { Order, OrderId } from '../../domain';
import { Result, ok, err } from '../../domain/shared/Result';

export class InMemoryOrderRepository implements OrderRepository {
  private orders: Map<string, Order> = new Map();

  async save(order: Order): Promise<Result<OrderId, Error>> {
    this.orders.set(order.id.value, order);
    return ok(order.id);
  }

  async findById(id: OrderId): Promise<Result<Order, Error>> {
    const order = this.orders.get(id.value);
    return order 
      ? ok(order) 
      : err(new Error('Order not found'));
  }

  async findByCustomerEmail(email: string): Promise<Result<Order[], Error>> {
    const orders = Array.from(this.orders.values())
      .filter(o => o.customerEmail.toString() === email);
    return ok(orders);
  }

  async findAll(): Promise<Order[]> {
    return Array.from(this.orders.values());
  }

  // Test helper methods
  clear(): void {
    this.orders.clear();
  }

  count(): number {
    return this.orders.size;
  }
}
```

### Fake Payment Gateway

```typescript
// adapters/payment/FakePaymentGateway.ts
import { PaymentGateway } from '../../domain/ports/PaymentGateway';
import { Money, Email } from '../../domain';
import { Result, ok, err } from '../../domain/shared/Result';

export class FakePaymentGateway implements PaymentGateway {
  public charges: Array<{ amount: Money; customer: Email }> = [];
  private shouldDecline = false;
  private shouldTimeout = false;

  async charge(amount: Money, customer: Email): Promise<Result<string, Error>> {
    if (this.shouldTimeout) {
      throw new Error('Payment timeout');
    }

    if (this.shouldDecline) {
      return err(new Error('Payment declined'));
    }

    this.charges.push({ amount, customer });
    return ok(`charge_${Date.now()}`);
  }

  // Test control methods
  setShouldDecline(decline: boolean): void {
    this.shouldDecline = decline;
  }

  setShouldTimeout(timeout: boolean): void {
    this.shouldTimeout = timeout;
  }

  reset(): void {
    this.charges = [];
    this.shouldDecline = false;
    this.shouldTimeout = false;
  }
}
```

### Elixir Application Service Testing

```elixir
# test/shop/application/services/place_order_service_test.exs
defmodule Shop.Application.Services.PlaceOrderServiceTest do
  use ExUnit.Case, async: true

  alias Shop.Application.Services.PlaceOrderService
  alias Shop.Application.Commands.PlaceOrderCommand
  alias Shop.Domain.{Ebook, Money}
  alias Shop.Test.Support.{
    InMemoryOrderRepository,
    InMemoryEbookRepository,
    FakePaymentGateway
  }

  setup do
    # Start in-memory repositories and services
    {:ok, _} = start_supervised(InMemoryOrderRepository)
    {:ok, _} = start_supervised(InMemoryEbookRepository)
    {:ok, _} = start_supervised(FakePaymentGateway)

    # Seed test data
    ebook = %Ebook{
      id: "ebook-123",
      title: "Clean Architecture",
      price: Money.new(2999, :USD)
    }
    InMemoryEbookRepository.save(ebook)

    :ok
  end

  describe "execute/1 - success scenarios" do
    test "creates order with correct data" do
      command = %PlaceOrderCommand{
        ebook_id: "ebook-123",
        customer_email: "customer@example.com",
        quantity: 2
      }

      assert {:ok, order_id} = PlaceOrderService.execute(command)
      assert is_binary(order_id)

      # Verify order was saved
      {:ok, orders} = InMemoryOrderRepository.find_all()
      assert length(orders) == 1
      assert hd(orders).quantity == 2
      assert hd(orders).ebook_id == "ebook-123"
    end

    test "charges correct amount" do
      command = %PlaceOrderCommand{
        ebook_id: "ebook-123",
        customer_email: "customer@example.com",
        quantity: 3
      }

      assert {:ok, _order_id} = PlaceOrderService.execute(command)

      {:ok, charges} = FakePaymentGateway.get_charges()
      assert length(charges) == 1
      assert hd(charges).amount.amount == 8997
    end
  end

  describe "execute/1 - failure scenarios" do
    test "fails when ebook not found" do
      command = %PlaceOrderCommand{
        ebook_id: "nonexistent",
        customer_email: "customer@example.com",
        quantity: 1
      }

      assert {:error, :ebook_not_found} = PlaceOrderService.execute(command)

      # Verify no order was created
      {:ok, orders} = InMemoryOrderRepository.find_all()
      assert orders == []
    end

    test "fails when payment is declined" do
      FakePaymentGateway.set_should_decline(true)

      command = %PlaceOrderCommand{
        ebook_id: "ebook-123",
        customer_email: "customer@example.com",
        quantity: 1
      }

      assert {:error, :payment_failed} = PlaceOrderService.execute(command)

      # Verify no order was created
      {:ok, orders} = InMemoryOrderRepository.find_all()
      assert orders == []
    end

    test "handles invalid email" do
      command = %PlaceOrderCommand{
        ebook_id: "ebook-123",
        customer_email: "invalid-email",
        quantity: 1
      }

      assert {:error, :invalid_email} = PlaceOrderService.execute(command)
    end
  end

  describe "execute/1 - edge cases" do
    test "handles concurrent orders" do
      tasks = for i <- 1..10 do
        Task.async(fn ->
          command = %PlaceOrderCommand{
            ebook_id: "ebook-123",
            customer_email: "customer#{i}@example.com",
            quantity: 1
          }
          PlaceOrderService.execute(command)
        end)
      end

      results = Task.await_many(tasks)

      # All should succeed
      assert Enum.all?(results, &match?({:ok, _}, &1))

      # Verify all orders were saved
      {:ok, orders} = InMemoryOrderRepository.find_all()
      assert length(orders) == 10
    end
  end
end
```

## Infrastructure Layer Testing

Infrastructure tests verify that adapters work correctly with real external systems.

### TypeScript Infrastructure Testing

```typescript
// adapters/persistence/PostgresOrderRepository.test.ts
import { PostgresOrderRepository } from './PostgresOrderRepository';
import { Order, OrderId, Money } from '../../domain';
import { createTestDatabase, cleanupTestDatabase } from '../test-helpers';

describe('PostgresOrderRepository', () => {
  let repository: PostgresOrderRepository;
  let db: any;

  beforeAll(async () => {
    db = await createTestDatabase();
    repository = new PostgresOrderRepository(db);
  });

  afterAll(async () => {
    await cleanupTestDatabase(db);
  });

  beforeEach(async () => {
    await db.query('TRUNCATE TABLE orders CASCADE');
  });

  it('saves order to database', async () => {
    const order = Order.create({
      id: OrderId.generate(),
      ebookId: 'ebook-123',
      customerEmail: 'test@example.com',
      quantity: 2,
      pricePerUnit: Money.fromAmount(19.99, 'USD')
    }).value;

    const result = await repository.save(order);

    expect(result.isOk()).toBe(true);

    // Verify in actual database
    const rows = await db.query(
      'SELECT * FROM orders WHERE id = $1',
      [order.id.value]
    );
    expect(rows.length).toBe(1);
    expect(rows[0].quantity).toBe(2);
  });

  it('retrieves order from database', async () => {
    const order = Order.create({
      id: OrderId.generate(),
      ebookId: 'ebook-123',
      customerEmail: 'test@example.com',
      quantity: 1,
      pricePerUnit: Money.fromAmount(29.99, 'USD')
    }).value;

    await repository.save(order);
    const result = await repository.findById(order.id);

    expect(result.isOk()).toBe(true);
    expect(result.value.id.equals(order.id)).toBe(true);
    expect(result.value.quantity).toBe(1);
  });

  it('handles not found correctly', async () => {
    const result = await repository.findById(OrderId.generate());

    expect(result.isErr()).toBe(true);
    expect(result.error.message).toContain('not found');
  });
});
```

**Key points:**
- ✅ Tests actual database interactions
- ✅ Slower but necessary
- ✅ Fewer tests than unit tests
- ✅ Focus on adapter contract, not business logic

## React Component Testing

Test React components with injected dependencies via Context.

### Component with Dependency Injection

```typescript
// components/OrderForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { OrderForm } from './OrderForm';
import { ServiceContext } from '../context/ServiceContext';
import { InMemoryOrderRepository } from '../adapters/persistence/InMemoryOrderRepository';
import { FakePaymentGateway } from '../adapters/payment/FakePaymentGateway';
import { PlaceOrderService } from '../application/services/PlaceOrderService';

describe('OrderForm', () => {
  it('places order successfully', async () => {
    // Setup test dependencies
    const orderRepo = new InMemoryOrderRepository();
    const paymentGateway = new FakePaymentGateway();
    const placeOrderService = new PlaceOrderService(
      orderRepo,
      ebookRepo,
      paymentGateway
    );

    // Render with test context
    render(
      <ServiceContext.Provider value={{ placeOrderService }}>
        <OrderForm ebookId="ebook-123" />
      </ServiceContext.Provider>
    );

    // Fill form
    fireEvent.change(screen.getByLabelText('Email'), {
      target: { value: 'test@example.com' }
    });
    fireEvent.change(screen.getByLabelText('Quantity'), {
      target: { value: '2' }
    });

    // Submit
    fireEvent.click(screen.getByText('Place Order'));

    // Verify success
    await waitFor(() => {
      expect(screen.getByText(/Order placed successfully/i)).toBeInTheDocument();
    });

    // Verify service was called correctly
    const orders = await orderRepo.findAll();
    expect(orders.length).toBe(1);
  });

  it('displays error on payment failure', async () => {
    const paymentGateway = new FakePaymentGateway();
    paymentGateway.setShouldDecline(true);

    const placeOrderService = new PlaceOrderService(
      new InMemoryOrderRepository(),
      ebookRepo,
      paymentGateway
    );

    render(
      <ServiceContext.Provider value={{ placeOrderService }}>
        <OrderForm ebookId="ebook-123" />
      </ServiceContext.Provider>
    );

    fireEvent.change(screen.getByLabelText('Email'), {
      target: { value: 'test@example.com' }
    });
    fireEvent.click(screen.getByText('Place Order'));

    await waitFor(() => {
      expect(screen.getByText(/Payment failed/i)).toBeInTheDocument();
    });
  });
});
```

## Phoenix Controller Testing

Test Phoenix controllers with test configuration for repositories.

```elixir
# test/shop_web/controllers/order_controller_test.exs
defmodule ShopWeb.OrderControllerTest do
  use ShopWeb.ConnCase, async: true

  alias Shop.Test.Support.{InMemoryOrderRepository, InMemoryEbookRepository}
  alias Shop.Domain.{Ebook, Money}

  setup do
    # Start test repositories
    {:ok, _} = start_supervised(InMemoryOrderRepository)
    {:ok, _} = start_supervised(InMemoryEbookRepository)

    # Seed test data
    ebook = %Ebook{
      id: "ebook-123",
      title: "Test Book",
      price: Money.new(1999, :USD)
    }
    InMemoryEbookRepository.save(ebook)

    {:ok, conn: build_conn()}
  end

  describe "POST /api/orders" do
    test "creates order successfully", %{conn: conn} do
      params = %{
        "ebook_id" => "ebook-123",
        "customer_email" => "test@example.com",
        "quantity" => 2
      }

      conn = post(conn, "/api/orders", params)

      assert %{"order_id" => order_id} = json_response(conn, 201)
      assert is_binary(order_id)
    end

    test "returns error for invalid email", %{conn: conn} do
      params = %{
        "ebook_id" => "ebook-123",
        "customer_email" => "invalid",
        "quantity" => 1
      }

      conn = post(conn, "/api/orders", params)

      assert %{"error" => error} = json_response(conn, 400)
      assert error =~ "email"
    end

    test "returns error when ebook not found", %{conn: conn} do
      params = %{
        "ebook_id" => "nonexistent",
        "customer_email" => "test@example.com",
        "quantity" => 1
      }

      conn = post(conn, "/api/orders", params)

      assert %{"error" => "Ebook not found"} = json_response(conn, 404)
    end
  end
end
```

## Test Patterns

### Test Builders

Build complex domain objects easily:

```typescript
// test/builders/OrderBuilder.ts
export class OrderBuilder {
  private id = OrderId.generate();
  private ebookId = 'ebook-123';
  private email = 'test@example.com';
  private quantity = 1;
  private price = Money.fromAmount(19.99, 'USD');

  withId(id: OrderId): this {
    this.id = id;
    return this;
  }

  withEbookId(ebookId: string): this {
    this.ebookId = ebookId;
    return this;
  }

  withEmail(email: string): this {
    this.email = email;
    return this;
  }

  withQuantity(quantity: number): this {
    this.quantity = quantity;
    return this;
  }

  withPrice(price: Money): this {
    this.price = price;
    return this;
  }

  build(): Order {
    return Order.create({
      id: this.id,
      ebookId: this.ebookId,
      customerEmail: this.email,
      quantity: this.quantity,
      pricePerUnit: this.price
    }).value;
  }
}

// Usage
const order = new OrderBuilder()
  .withEmail('custom@example.com')
  .withQuantity(5)
  .build();
```

### Property-Based Testing

Test with random inputs to find edge cases:

```elixir
# test/shop/domain/money_test.exs
defmodule Shop.Domain.MoneyTest do
  use ExUnit.Case, async: true
  use ExUnitProperties

  alias Shop.Domain.Money

  property "addition is commutative" do
    check all a <- integer(0..1_000_000),
              b <- integer(0..1_000_000) do
      m1 = Money.new(a, :USD)
      m2 = Money.new(b, :USD)

      assert Money.add(m1, m2) == Money.add(m2, m1)
    end
  end

  property "subtraction then addition returns original" do
    check all amount <- integer(100..1_000_000),
              subtract_amount <- integer(1..99) do
      original = Money.new(amount, :USD)
      to_subtract = Money.new(subtract_amount, :USD)

      result = original
        |> Money.subtract(to_subtract)
        |> Money.add(to_subtract)

      assert result.amount == original.amount
    end
  end

  property "multiplication is distributive" do
    check all a <- integer(1..10_000),
              b <- integer(1..10),
              c <- integer(1..10) do
      m = Money.new(a, :USD)

      # (b + c) * a == b * a + c * a
      left = Money.multiply(m, b + c)
      right = Money.add(
        Money.multiply(m, b),
        Money.multiply(m, c)
      )

      assert left.amount == right.amount
    end
  end
end
```

### Test Organization

```
test/
├── domain/              # Pure unit tests
│   ├── order_test.ts
│   ├── email_test.ts
│   └── money_test.ts
├── application/         # Service tests with test doubles
│   └── services/
│       └── place_order_service_test.ts
├── adapters/            # Integration tests
│   ├── persistence/
│   │   └── postgres_order_repository_test.ts
│   └── payment/
│       └── stripe_payment_gateway_test.ts
├── components/          # Component tests
│   └── order_form_test.tsx
└── support/             # Test helpers
    ├── builders/
    │   └── order_builder.ts
    └── in_memory/
        ├── in_memory_order_repository.ts
        └── fake_payment_gateway.ts
```

## Common Pitfalls

### ❌ Pitfall 1: Testing Implementation Instead of Behavior

```typescript
// ❌ BAD: Coupled to internal method
it('calls calculateDiscount', () => {
  const spy = jest.spyOn(order, 'calculateDiscount');
  order.applyBulkPricing();
  expect(spy).toHaveBeenCalled();
});

// ✅ GOOD: Test behavior, not implementation
it('applies 10% discount for bulk orders', () => {
  order.applyBulkPricing();
  expect(order.calculateTotal().amount).toBe(9000);
});
```

### ❌ Pitfall 2: Overusing Mocks

```typescript
// ❌ BAD: Too many mocks
const orderRepo = mock<OrderRepository>();
const ebookRepo = mock<EbookRepository>();
const paymentGateway = mock<PaymentGateway>();
const emailService = mock<EmailService>();
const analytics = mock<Analytics>();

// ✅ GOOD: Use real test implementations
const orderRepo = new InMemoryOrderRepository();
const ebookRepo = new InMemoryEbookRepository();
// etc.
```

### ❌ Pitfall 3: Brittle Tests

```typescript
// ❌ BAD: Breaks on internal changes
expect(service['internalMethod']).toBeDefined();

// ✅ GOOD: Test public interface only
expect(await service.execute(command)).toBeOk();
```

## Benefits

### Before: Painful Testing

```typescript
// Takes 5 seconds, requires database
describe('OrderService', () => {
  beforeEach(async () => {
    await setupDatabase();
    await runMigrations();
  });
  
  it('places order', async () => {
    // Complex setup
    // Slow execution
    // Brittle assertions
  });
});

// Result: Developers skip running tests
```

### After: Fast, Reliable Testing

```typescript
// Takes 5 milliseconds, no dependencies
describe('Order', () => {
  it('calculates total', () => {
    const order = new OrderBuilder().build();
    expect(order.calculateTotal()).toBe(expected);
  });
});

describe('PlaceOrderService', () => {
  it('places order', async () => {
    const service = new PlaceOrderService(
      new InMemoryOrderRepository(),
      new FakePaymentGateway()
    );
    const result = await service.execute(command);
    expect(result).toBeOk();
  });
});

// Result: Tests run constantly, catching bugs early
```

### Metrics Improvement

**Before Clean Architecture:**
- Test suite: 15 minutes
- Tests requiring database: 85%
- Flaky test rate: 20%
- Developers running tests: 40%

**After Clean Architecture:**
- Test suite: 30 seconds
- Tests requiring database: 10%
- Flaky test rate: 2%
- Developers running tests: 95%

## Conclusion

Clean architecture transforms testing from a painful chore into a fast, reliable safety net. By separating concerns and inverting dependencies, you gain the ability to test each layer with the appropriate strategy:

**Domain Layer:**
- Pure unit tests
- No mocks needed
- Milliseconds per test
- Comprehensive coverage

**Application Layer:**
- Test doubles for dependencies
- Fast service tests
- Focus on workflows
- Still no real infrastructure

**Infrastructure Layer:**
- Integration tests
- Real systems
- Verify adapters work
- Fewer tests needed

### Key Takeaways

1. **Fast tests enable confidence**: When tests run in milliseconds, you run them constantly
2. **Test behavior, not implementation**: Focus on what code does, not how it does it
3. **Use real implementations when possible**: Prefer in-memory repositories over mocks
4. **Test at the right level**: Domain tests for business logic, integration tests for infrastructure
5. **Property-based testing finds edge cases**: Random inputs discover bugs you wouldn't think to test

### Next Steps

- Start with domain layer tests—they're easiest and most valuable
- Build in-memory implementations for all repositories
- Create test builders for complex objects
- Add property-based tests for critical calculations
- Reserve integration tests for adapter contracts

**Remember:** The best test suite is one that runs fast enough that developers actually run it. Clean architecture makes that possible.

---

**Previous**: [Chapter 10: Hexagonal Architecture](10-hexagonal-architecture.md)  
**Next**: Chapter 12: Refactoring React Code *(coming soon)*

*Testing is not about catching bugs—it's about enabling fearless refactoring. Clean architecture gives you the structure to write tests that make change safe.* ✨
