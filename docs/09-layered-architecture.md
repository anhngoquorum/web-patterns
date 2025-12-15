# Chapter 9: Layered Architecture

## Introduction

Software architecture is about organizing code so it's easy to understand, test, and change. One of the most fundamental and enduring patterns for achieving this is **Layered Architecture**—a way of organizing your application into distinct layers, each with a clear responsibility and well-defined boundaries.

In the previous chapters, you've learned individual patterns: domain models, repositories, application services, and commands. Now it's time to see how all these pieces fit together into a coherent, maintainable architecture.

This chapter shows you how to organize both React/TypeScript and Elixir/Phoenix applications into layers that make your code easier to work with, whether you're adding features, fixing bugs, or refactoring for performance.

### What is Layered Architecture?

Layered Architecture is a pattern that organizes code into horizontal layers, where each layer:
- Has a **single, clear responsibility**
- **Depends only on layers below it** (or sometimes not at all)
- Provides a **well-defined interface** to layers above it
- Can be **tested independently** from other layers

Think of it like a building: the foundation (domain logic) supports the structure (application services), which supports the roof (user interface). You don't build the roof first, and you don't let the roof dictate how the foundation is built.

### Traditional 3-Tier vs Domain-Driven Layers

**Traditional 3-tier architecture:**
```
┌─────────────────────────────┐
│   Presentation Layer        │  (UI, Controllers)
├─────────────────────────────┤
│   Business Logic Layer      │  (Services, Validation)
├─────────────────────────────┤
│   Data Access Layer         │  (Database, Repositories)
└─────────────────────────────┘
```

This works, but tends to make the data layer the center of everything, leading to anemic domain models and database-driven design.

**Domain-Driven Layered Architecture:**
```
┌───────────────────────────────────┐
│   Infrastructure Layer            │  (Controllers, DB, APIs, UI)
│   ↓ depends on                    │
├───────────────────────────────────┤
│   Application Layer               │  (Use Cases, Workflows)
│   ↓ depends on                    │
├───────────────────────────────────┤
│   Domain Layer (Core)             │  (Entities, Value Objects, Rules)
│   ↓ depends on nothing            │
└───────────────────────────────────┘
```

This inverts the dependencies, making the domain the center of your application. The domain is pure, isolated, and independent of infrastructure.

### Why Layered Architecture Matters

**Benefits:**
- ✅ **Maintainability**: Changes are localized to specific layers
- ✅ **Testability**: Each layer can be tested independently
- ✅ **Flexibility**: Swap implementations without touching other layers
- ✅ **Understanding**: Clear structure helps developers navigate the code
- ✅ **Team Collaboration**: Different teams can work on different layers
- ✅ **Technology Evolution**: Upgrade frameworks without rewriting business logic

**What You'll Learn:**
- The three core layers and their responsibilities
- The dependency rule and why it's crucial
- How to structure React and Elixir projects
- Communication patterns between layers
- Testing strategies for each layer
- Common pitfalls and how to avoid them
- When to use (and not use) this architecture

Let's start by looking at what happens when you *don't* use layers.

## The Problem: Big Ball of Mud

Without layers, applications turn into a "big ball of mud"—a tangled mess where everything depends on everything else. Let's see what this looks like.

### React: Everything in One Component

```typescript
// OrderForm.tsx - The Big Ball of Mud 🔴
import React, { useState } from 'react';
import axios from 'axios';

interface Props {
  ebookId: string;
}

export function OrderForm({ ebookId }: Props) {
  const [email, setEmail] = useState('');
  const [quantity, setQuantity] = useState(1);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      // UI logic mixed with validation
      if (!email || !email.includes('@')) {
        setError('Invalid email address');
        setLoading(false);
        return;
      }

      if (quantity <= 0 || quantity > 100) {
        setError('Quantity must be between 1 and 100');
        setLoading(false);
        return;
      }

      // Business logic in component
      const ebookResponse = await axios.get(`/api/ebooks/${ebookId}`);
      const ebook = ebookResponse.data;

      // More business logic
      const subtotal = ebook.price * quantity;
      const tax = subtotal * 0.08;
      const total = subtotal + tax;

      // Even more business logic
      let discount = 0;
      if (quantity >= 5) {
        discount = subtotal * 0.1;
      }
      const finalTotal = total - discount;

      // Direct database access mixed in
      const orderResponse = await axios.post('/api/orders', {
        ebook_id: ebookId,
        customer_email: email,
        quantity: quantity,
        subtotal: subtotal,
        tax: tax,
        discount: discount,
        total: finalTotal
      });

      // More side effects
      await axios.post('/api/notifications/send', {
        type: 'order_confirmation',
        email: email,
        order_id: orderResponse.data.id
      });

      // Yet another side effect
      await axios.post('/api/analytics/track', {
        event: 'order_placed',
        order_id: orderResponse.data.id,
        amount: finalTotal
      });

      alert(`Order placed! Total: $${finalTotal.toFixed(2)}`);
    } catch (err) {
      setError('Failed to place order: ' + (err as Error).message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="email">Email:</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
        />
      </div>
      <div>
        <label htmlFor="quantity">Quantity:</label>
        <input
          id="quantity"
          type="number"
          value={quantity}
          onChange={(e) => setQuantity(parseInt(e.target.value, 10))}
        />
      </div>
      <button type="submit" disabled={loading}>
        {loading ? 'Processing...' : 'Place Order'}
      </button>
      {error && <div className="error">{error}</div>}
    </form>
  );
}
```

### Elixir: Everything in Controller

```elixir
# order_controller.ex - The Big Ball of Mud 🔴
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  
  alias Shop.Repo
  alias Shop.Ebook
  alias Shop.Order

  def create(conn, params) do
    # Validation in controller
    email = params["customer_email"]
    unless email && String.contains?(email, "@") do
      conn
      |> put_status(:bad_request)
      |> json(%{error: "Invalid email"})
      |> halt()
    end

    quantity = params["quantity"]
    unless is_integer(quantity) && quantity > 0 && quantity <= 100 do
      conn
      |> put_status(:bad_request)
      |> json(%{error: "Invalid quantity"})
      |> halt()
    end

    # Database access directly in controller
    ebook = Repo.get(Ebook, params["ebook_id"])
    unless ebook do
      conn
      |> put_status(:not_found)
      |> json(%{error: "Ebook not found"})
      |> halt()
    end

    # Business calculations in controller
    subtotal = ebook.price * quantity
    tax = subtotal * 0.08
    
    # More business logic
    discount = if quantity >= 5 do
      subtotal * 0.1
    else
      0
    end
    
    total = subtotal + tax - discount

    # Database writes in controller
    order_params = %{
      ebook_id: ebook.id,
      customer_email: email,
      quantity: quantity,
      subtotal: subtotal,
      tax: tax,
      discount: discount,
      total: total,
      status: "pending"
    }

    case Repo.insert(Order.changeset(%Order{}, order_params)) do
      {:ok, order} ->
        # Side effects in controller
        send_confirmation_email(email, order)
        track_analytics("order_placed", order)
        update_inventory(ebook.id, quantity)
        
        conn
        |> put_status(:created)
        |> json(%{order_id: order.id, total: total})
        
      {:error, changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{errors: translate_errors(changeset)})
    end
  end

  defp send_confirmation_email(email, order) do
    # Email logic here
  end

  defp track_analytics(event, order) do
    # Analytics logic here
  end

  defp update_inventory(ebook_id, quantity) do
    # Inventory update logic here
  end

  defp translate_errors(changeset) do
    # Error translation logic here
  end
end
```

### Problems with This Approach

**1. Hard to Test**
- Can't test business logic without mocking HTTP calls
- Can't test validation without rendering components
- Can't test calculations without database access

**2. Hard to Change**
- Want to change tax calculation? Must edit UI code
- Want to change discount rules? Must edit controller
- Want to add a new notification channel? Must edit the order creation code

**3. Hard to Understand**
- Where is the business logic? Mixed with everything else
- What are the actual business rules? Buried in implementation details
- How does order creation work? Have to read through 100+ lines

**4. Not Reusable**
- Can't place an order from CLI
- Can't place an order from background job
- Can't place an order from different API endpoint
- Each context needs to duplicate all this logic

**5. Brittle**
- Change email validation, might break database code
- Change HTTP library, might break business logic
- Change UI framework, have to rewrite everything

**6. Dependencies Everywhere**
```
UI Component
  ↓ depends on
HTTP Client
  ↓ depends on  
Business Logic (mixed in component)
  ↓ depends on
Database (through HTTP)
  ↓ circular dependencies!
Validation (in multiple places)
```

This is what we need to fix with layered architecture.


## The Three Core Layers

Layered architecture solves the "big ball of mud" by organizing code into three distinct layers, each with a clear responsibility.

### 1. Domain Layer (The Core)

**Responsibility:** Contains all business logic, rules, and domain concepts

**What lives here:**
- Entities (Order, Customer, Ebook)
- Value Objects (Email, Money, OrderId)
- Domain Services (PricingService, TaxCalculator)
- Business Rules and Validation
- Domain Events

**Characteristics:**
- ✅ Pure business logic—no framework dependencies
- ✅ No infrastructure concerns (no HTTP, no database)
- ✅ Fast to test (just plain functions/classes)
- ✅ Framework-agnostic (works with React, Vue, Elixir, anything)
- ✅ Long-lived (changes only when business changes)

**Example - TypeScript:**
```typescript
// domain/Order.ts
import { Result, ok, err } from 'neverthrow';
import { Email } from './Email';
import { Money } from './Money';

export class Order {
  private constructor(
    public readonly id: string,
    public readonly ebookId: string,
    public readonly customerEmail: Email,
    public readonly quantity: number,
    public readonly pricePerUnit: Money
  ) {}

  static create(data: {
    id: string;
    ebookId: string;
    customerEmail: string;
    quantity: number;
    pricePerUnit: Money;
  }): Result<Order, Error> {
    // Business validation
    if (data.quantity <= 0) {
      return err(new Error('Quantity must be positive'));
    }

    if (data.quantity > 100) {
      return err(new Error('Cannot order more than 100 copies'));
    }

    const emailResult = Email.create(data.customerEmail);
    if (emailResult.isErr()) {
      return err(emailResult.error);
    }

    return ok(new Order(
      data.id,
      data.ebookId,
      emailResult.value,
      data.quantity,
      data.pricePerUnit
    ));
  }

  calculateSubtotal(): Money {
    return this.pricePerUnit.multiply(this.quantity);
  }

  calculateDiscount(): Money {
    const subtotal = this.calculateSubtotal();
    
    // Business rule: bulk discount
    if (this.quantity >= 5) {
      return subtotal.multiply(0.1); // 10% discount
    }
    
    return Money.zero(subtotal.currency);
  }

  calculateTax(): Money {
    const subtotal = this.calculateSubtotal();
    const discount = this.calculateDiscount();
    const taxableAmount = subtotal.subtract(discount);
    
    // Business rule: tax rate
    return taxableAmount.multiply(0.08); // 8% tax
  }

  calculateTotal(): Money {
    const subtotal = this.calculateSubtotal();
    const discount = this.calculateDiscount();
    const tax = this.calculateTax();
    
    return subtotal.subtract(discount).add(tax);
  }
}

// domain/Email.ts
export class Email {
  private constructor(private readonly value: string) {}

  static create(email: string): Result<Email, Error> {
    if (!email || !email.includes('@')) {
      return err(new Error('Invalid email format'));
    }
    
    return ok(new Email(email.toLowerCase().trim()));
  }

  toString(): string {
    return this.value;
  }
}
```

**Example - Elixir:**
```elixir
# lib/shop/domain/order.ex
defmodule Shop.Domain.Order do
  @moduledoc """
  Domain model for an Order.
  Contains all business logic for orders.
  """

  alias Shop.Domain.{Email, Money}

  @enforce_keys [:id, :ebook_id, :customer_email, :quantity, :price_per_unit]
  defstruct [:id, :ebook_id, :customer_email, :quantity, :price_per_unit]

  @type t :: %__MODULE__{
    id: String.t(),
    ebook_id: String.t(),
    customer_email: Email.t(),
    quantity: pos_integer(),
    price_per_unit: Money.t()
  }

  @doc """
  Creates a new Order with business validation.
  """
  @spec new(String.t(), String.t(), pos_integer(), Money.t()) ::
    {:ok, t()} | {:error, term()}
  def new(id, ebook_id, email_string, quantity, price_per_unit) do
    with :ok <- validate_quantity(quantity),
         {:ok, email} <- Email.new(email_string) do
      order = %__MODULE__{
        id: id,
        ebook_id: ebook_id,
        customer_email: email,
        quantity: quantity,
        price_per_unit: price_per_unit
      }
      {:ok, order}
    end
  end

  @doc """
  Calculates the subtotal (price * quantity).
  """
  @spec calculate_subtotal(t()) :: Money.t()
  def calculate_subtotal(%__MODULE__{} = order) do
    Money.multiply(order.price_per_unit, order.quantity)
  end

  @doc """
  Calculates discount based on business rules.
  """
  @spec calculate_discount(t()) :: Money.t()
  def calculate_discount(%__MODULE__{quantity: quantity} = order) do
    subtotal = calculate_subtotal(order)
    
    # Business rule: bulk discount
    if quantity >= 5 do
      Money.multiply(subtotal, 0.1) # 10% discount
    else
      Money.zero(subtotal.currency)
    end
  end

  @doc """
  Calculates tax based on business rules.
  """
  @spec calculate_tax(t()) :: Money.t()
  def calculate_tax(%__MODULE__{} = order) do
    subtotal = calculate_subtotal(order)
    discount = calculate_discount(order)
    taxable_amount = Money.subtract(subtotal, discount)
    
    # Business rule: 8% tax rate
    Money.multiply(taxable_amount, 0.08)
  end

  @doc """
  Calculates the final total.
  """
  @spec calculate_total(t()) :: Money.t()
  def calculate_total(%__MODULE__{} = order) do
    subtotal = calculate_subtotal(order)
    discount = calculate_discount(order)
    tax = calculate_tax(order)
    
    subtotal
    |> Money.subtract(discount)
    |> Money.add(tax)
  end

  defp validate_quantity(quantity) when quantity > 0 and quantity <= 100, do: :ok
  defp validate_quantity(quantity) when quantity <= 0, do: {:error, :quantity_must_be_positive}
  defp validate_quantity(_), do: {:error, :quantity_too_large}
end

# lib/shop/domain/email.ex
defmodule Shop.Domain.Email do
  @moduledoc """
  Value object representing an email address.
  """

  @enforce_keys [:value]
  defstruct [:value]

  @type t :: %__MODULE__{value: String.t()}

  @spec new(String.t()) :: {:ok, t()} | {:error, :invalid_email}
  def new(email) when is_binary(email) do
    normalized = email |> String.trim() |> String.downcase()
    
    if String.contains?(normalized, "@") and byte_size(normalized) > 3 do
      {:ok, %__MODULE__{value: normalized}}
    else
      {:error, :invalid_email}
    end
  end

  def new(_), do: {:error, :invalid_email}

  def to_string(%__MODULE__{value: value}), do: value
end
```

**Key Point:** Notice how the domain layer has NO dependencies on frameworks, databases, or HTTP. It's pure business logic.

### 2. Application Layer (Use Case Orchestration)

**Responsibility:** Orchestrates use cases by coordinating domain objects and infrastructure

**What lives here:**
- Application Services (PlaceOrderService, CancelOrderService)
- Commands (PlaceOrderCommand, CancelOrderCommand)
- Command Handlers
- Use Case Workflows
- Transaction Boundaries

**Characteristics:**
- ✅ Orchestrates, doesn't implement business logic
- ✅ Depends on Domain Layer only
- ✅ Defines transaction boundaries
- ✅ Calls repositories and external services
- ✅ Triggers side effects (notifications, events)
- ✅ Thin—most logic is in domain

**Example - TypeScript:**
```typescript
// application/PlaceOrderService.ts
import { Result, ok, err } from 'neverthrow';
import { PlaceOrderCommand } from './PlaceOrderCommand';
import { Order } from '../domain/Order';
import { OrderRepository } from '../domain/OrderRepository';
import { EbookRepository } from '../domain/EbookRepository';
import { NotificationService } from './NotificationService';

export class PlaceOrderService {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly ebookRepo: EbookRepository,
    private readonly notifier: NotificationService
  ) {}

  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // 1. Fetch ebook (to get price)
    const ebookResult = await this.ebookRepo.findById(command.ebookId);
    if (ebookResult.isErr()) {
      return err(new Error('Ebook not found'));
    }
    const ebook = ebookResult.value;

    // 2. Create domain model (domain does the validation)
    const orderResult = Order.create({
      id: this.generateId(),
      ebookId: command.ebookId,
      customerEmail: command.customerEmail,
      quantity: command.quantity,
      pricePerUnit: ebook.price
    });

    if (orderResult.isErr()) {
      return err(orderResult.error);
    }
    const order = orderResult.value;

    // 3. Save order
    const saveResult = await this.orderRepo.save(order);
    if (saveResult.isErr()) {
      return err(new Error('Failed to save order'));
    }
    const orderId = saveResult.value;

    // 4. Trigger side effects
    await this.notifier.sendOrderConfirmation(order);

    return ok(orderId);
  }

  private generateId(): string {
    return `order-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// application/PlaceOrderCommand.ts
export class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // Structural validation only
    if (!ebookId) throw new Error('ebookId is required');
    if (!customerEmail) throw new Error('customerEmail is required');
    if (quantity <= 0) throw new Error('quantity must be positive');
  }
}
```

**Example - Elixir:**
```elixir
# lib/shop/application/place_order_service.ex
defmodule Shop.Application.PlaceOrderService do
  @moduledoc """
  Application service for placing orders.
  Orchestrates the use case.
  """

  alias Shop.Domain.Order
  alias Shop.Application.PlaceOrderCommand
  alias Shop.Infrastructure.{OrderRepository, EbookRepository, NotificationService}

  @spec execute(PlaceOrderCommand.t()) :: {:ok, String.t()} | {:error, term()}
  def execute(%PlaceOrderCommand{} = command) do
    # 1. Fetch ebook
    with {:ok, ebook} <- EbookRepository.find_by_id(command.ebook_id),
         # 2. Create domain model (domain validates)
         {:ok, order} <- Order.new(
           generate_id(),
           command.ebook_id,
           command.customer_email,
           command.quantity,
           ebook.price
         ),
         # 3. Save order
         {:ok, order_id} <- OrderRepository.save(order) do
      # 4. Trigger side effects
      NotificationService.send_order_confirmation(order)
      
      {:ok, order_id}
    end
  end

  defp generate_id do
    "order-#{System.system_time(:millisecond)}-#{:crypto.strong_rand_bytes(4) |> Base.encode16()}"
  end
end

# lib/shop/application/place_order_command.ex
defmodule Shop.Application.PlaceOrderCommand do
  @enforce_keys [:ebook_id, :customer_email, :quantity]
  defstruct [:ebook_id, :customer_email, :quantity]

  @type t :: %__MODULE__{
    ebook_id: String.t(),
    customer_email: String.t(),
    quantity: pos_integer()
  }

  @spec new(map()) :: {:ok, t()} | {:error, :invalid_command}
  def new(%{ebook_id: ebook_id, customer_email: email, quantity: qty})
      when is_binary(ebook_id) and is_binary(email) and is_integer(qty) and qty > 0 do
    {:ok, %__MODULE__{
      ebook_id: ebook_id,
      customer_email: email,
      quantity: qty
    }}
  end

  def new(_), do: {:error, :invalid_command}
end
```

**Key Point:** Application services are thin coordinators. They call domain objects to execute business logic and repositories to persist/retrieve data.

### 3. Infrastructure Layer (External World)

**Responsibility:** Handles all communication with external systems

**What lives here:**
- Repository Implementations (EctoOrderRepository, RestOrderRepository)
- HTTP Controllers (OrderController)
- API Clients
- Database Schemas
- Email Services
- React Components (UI)
- External Service Adapters

**Characteristics:**
- ✅ Implements interfaces defined in domain/application
- ✅ Depends on both application and domain layers
- ✅ Framework-specific code lives here
- ✅ Changes when technology changes
- ✅ Can be swapped without touching domain

**Example - TypeScript:**
```typescript
// infrastructure/HttpOrderRepository.ts
import { Result, ok, err } from 'neverthrow';
import { Order } from '../domain/Order';
import { OrderRepository } from '../domain/OrderRepository';
import axios from 'axios';

export class HttpOrderRepository implements OrderRepository {
  constructor(private readonly baseUrl: string) {}

  async save(order: Order): Promise<Result<string, Error>> {
    try {
      const response = await axios.post(`${this.baseUrl}/orders`, {
        id: order.id,
        ebook_id: order.ebookId,
        customer_email: order.customerEmail.toString(),
        quantity: order.quantity,
        price_per_unit: order.pricePerUnit.toNumber(),
        total: order.calculateTotal().toNumber()
      });

      return ok(response.data.id);
    } catch (error) {
      return err(new Error('Failed to save order'));
    }
  }

  async findById(id: string): Promise<Result<Order, Error>> {
    try {
      const response = await axios.get(`${this.baseUrl}/orders/${id}`);
      const data = response.data;

      const orderResult = Order.create({
        id: data.id,
        ebookId: data.ebook_id,
        customerEmail: data.customer_email,
        quantity: data.quantity,
        pricePerUnit: Money.fromAmount(data.price_per_unit, 'USD')
      });

      return orderResult;
    } catch (error) {
      return err(new Error('Order not found'));
    }
  }
}

// infrastructure/OrderForm.tsx (React Component)
import React, { useState } from 'react';
import { PlaceOrderCommand } from '../application/PlaceOrderCommand';
import { usePlaceOrderService } from '../hooks/useServices';

export function OrderForm({ ebookId }: { ebookId: string }) {
  const placeOrderService = usePlaceOrderService();
  const [email, setEmail] = useState('');
  const [quantity, setQuantity] = useState(1);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      const command = new PlaceOrderCommand(ebookId, email, quantity);
      const result = await placeOrderService.execute(command);

      if (result.isOk()) {
        alert(`Order placed! ID: ${result.value}`);
      } else {
        setError(result.error.message);
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="number"
        value={quantity}
        onChange={(e) => setQuantity(parseInt(e.target.value, 10))}
        min="1"
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Processing...' : 'Place Order'}
      </button>
      {error && <div className="error">{error}</div>}
    </form>
  );
}
```

**Example - Elixir:**
```elixir
# lib/shop/infrastructure/ecto_order_repository.ex
defmodule Shop.Infrastructure.EctoOrderRepository do
  @moduledoc """
  Ecto implementation of OrderRepository.
  Implements the repository interface defined in the domain.
  """

  alias Shop.Repo
  alias Shop.Infrastructure.OrderSchema
  alias Shop.Domain.{Order, Money, Email}

  @behaviour Shop.Domain.OrderRepository

  @impl true
  def save(%Order{} = order) do
    params = %{
      id: order.id,
      ebook_id: order.ebook_id,
      customer_email: Email.to_string(order.customer_email),
      quantity: order.quantity,
      price_per_unit: Money.to_integer(order.price_per_unit),
      total: Money.to_integer(Order.calculate_total(order))
    }

    %OrderSchema{}
    |> OrderSchema.changeset(params)
    |> Repo.insert()
    |> case do
      {:ok, schema} -> {:ok, schema.id}
      {:error, _changeset} -> {:error, :save_failed}
    end
  end

  @impl true
  def find_by_id(id) do
    case Repo.get(OrderSchema, id) do
      nil -> {:error, :not_found}
      schema -> schema_to_domain(schema)
    end
  end

  defp schema_to_domain(%OrderSchema{} = schema) do
    Order.new(
      schema.id,
      schema.ebook_id,
      schema.customer_email,
      schema.quantity,
      Money.new(schema.price_per_unit, :USD)
    )
  end
end

# lib/shop_web/controllers/order_controller.ex
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller

  alias Shop.Application.{PlaceOrderService, PlaceOrderCommand}

  def create(conn, params) do
    with {:ok, command} <- PlaceOrderCommand.new(params),
         {:ok, order_id} <- PlaceOrderService.execute(command) do
      conn
      |> put_status(:created)
      |> json(%{order_id: order_id})
    else
      {:error, :invalid_command} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Invalid parameters"})

      {:error, reason} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: to_string(reason)})
    end
  end
end
```

**Key Point:** Infrastructure layer implements interfaces defined by inner layers. It depends on them, not the other way around.


## The Dependency Rule

The most important principle in layered architecture is the **Dependency Rule**:

> **Dependencies point inward. Outer layers depend on inner layers, never the reverse.**

```
┌────────────────────────────────────────┐
│     Infrastructure Layer               │
│     ↓ depends on                       │
├────────────────────────────────────────┤
│     Application Layer                  │
│     ↓ depends on                       │
├────────────────────────────────────────┤
│     Domain Layer                       │
│     ↓ depends on NOTHING               │
└────────────────────────────────────────┘
```

### Why This Matters

**Without the dependency rule:**
```typescript
// ❌ BAD: Domain depends on infrastructure
class Order {
  async save() {
    // Domain directly depends on database!
    await db.query('INSERT INTO orders...');
  }
}
```

Problems:
- Can't test Order without a database
- Can't change databases without changing Order
- Domain logic coupled to infrastructure

**With the dependency rule:**
```typescript
// ✅ GOOD: Domain defines interface, infrastructure implements it

// Domain Layer - defines what it needs
interface OrderRepository {
  save(order: Order): Promise<Result<string, Error>>;
}

class Order {
  // No infrastructure dependencies!
  // Just pure business logic
}

// Infrastructure Layer - implements the interface
class PostgresOrderRepository implements OrderRepository {
  async save(order: Order): Promise<Result<string, Error>> {
    // Database-specific code here
  }
}

class MongoOrderRepository implements OrderRepository {
  async save(order: Order): Promise<Result<string, Error>> {
    // MongoDB-specific code here
  }
}
```

### Dependency Inversion in Practice

The key technique is **Dependency Inversion Principle (DIP)**:
1. Domain defines interfaces (what it needs)
2. Infrastructure implements interfaces (how it's done)
3. Application wires everything together

**TypeScript Example:**
```typescript
// Domain defines the contract
export interface OrderRepository {
  save(order: Order): Promise<Result<string, Error>>;
  findById(id: string): Promise<Result<Order, Error>>;
}

// Application depends on the interface
export class PlaceOrderService {
  constructor(private readonly orderRepo: OrderRepository) {}
  
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // Uses the interface, doesn't care about implementation
    await this.orderRepo.save(order);
  }
}

// Infrastructure provides implementations
export class HttpOrderRepository implements OrderRepository {
  // HTTP implementation
}

export class InMemoryOrderRepository implements OrderRepository {
  // In-memory implementation for tests
}

// Composition root wires it all up
const orderRepo = new HttpOrderRepository('https://api.example.com');
const service = new PlaceOrderService(orderRepo);
```

**Elixir Example:**
```elixir
# Domain defines behaviour
defmodule Shop.Domain.OrderRepository do
  @callback save(Order.t()) :: {:ok, String.t()} | {:error, term()}
  @callback find_by_id(String.t()) :: {:ok, Order.t()} | {:error, term()}
end

# Application depends on behaviour
defmodule Shop.Application.PlaceOrderService do
  @order_repo Application.compile_env(:shop, :order_repository)
  
  def execute(%PlaceOrderCommand{} = command) do
    # Uses the behaviour, doesn't care about implementation
    @order_repo.save(order)
  end
end

# Infrastructure implements behaviour
defmodule Shop.Infrastructure.EctoOrderRepository do
  @behaviour Shop.Domain.OrderRepository
  
  @impl true
  def save(order), do: # Ecto implementation
  
  @impl true
  def find_by_id(id), do: # Ecto implementation
end

# config/config.exs configures which implementation to use
config :shop, :order_repository, Shop.Infrastructure.EctoOrderRepository

# config/test.exs uses a different implementation for tests
config :shop, :order_repository, Shop.Test.InMemoryOrderRepository
```


## React/TypeScript Project Structure

Here's how to organize a React/TypeScript project with layered architecture:

```
src/
├── domain/                          # Domain Layer (Core Business Logic)
│   ├── order/
│   │   ├── Order.ts                # Entity with business logic
│   │   ├── OrderRepository.ts      # Interface (not implementation!)
│   │   └── OrderId.ts              # Value object
│   ├── ebook/
│   │   ├── Ebook.ts
│   │   ├── EbookRepository.ts
│   │   └── Price.ts
│   └── shared/
│       ├── Email.ts                # Shared value object
│       ├── Money.ts
│       └── Result.ts               # Result type helper
│
├── application/                     # Application Layer (Use Cases)
│   ├── orders/
│   │   ├── PlaceOrderCommand.ts    # Command
│   │   ├── PlaceOrderService.ts    # Application service
│   │   ├── CancelOrderCommand.ts
│   │   └── CancelOrderService.ts
│   ├── ebooks/
│   │   └── GetEbookQuery.ts
│   └── shared/
│       └── NotificationService.ts   # Port (interface)
│
├── infrastructure/                  # Infrastructure Layer
│   ├── repositories/
│   │   ├── HttpOrderRepository.ts   # HTTP implementation
│   │   ├── HttpEbookRepository.ts
│   │   └── InMemoryOrderRepository.ts  # For testing
│   ├── api/
│   │   └── ApiClient.ts             # HTTP client wrapper
│   ├── notifications/
│   │   └── EmailNotificationService.ts  # Implementation
│   └── components/                   # React UI components
│       ├── orders/
│       │   ├── OrderForm.tsx
│       │   ├── OrderList.tsx
│       │   └── OrderDetail.tsx
│       └── shared/
│           └── Button.tsx
│
├── composition/                      # Composition Root
│   └── ServiceFactory.ts             # Wires everything together
│
└── main.tsx                         # Entry point
```

### Dependency Flow

```
React Components (infrastructure/components)
    ↓ uses
Application Services (application/)
    ↓ uses
Domain Models (domain/)
    ↑ implements
Repository Implementations (infrastructure/repositories)
```

### Example Files

**Domain Interface:**
```typescript
// src/domain/order/OrderRepository.ts
import { Result } from '../shared/Result';
import { Order } from './Order';

export interface OrderRepository {
  save(order: Order): Promise<Result<string, Error>>;
  findById(id: string): Promise<Result<Order, Error>>;
  findByCustomerEmail(email: string): Promise<Result<Order[], Error>>;
}
```

**Infrastructure Implementation:**
```typescript
// src/infrastructure/repositories/HttpOrderRepository.ts
import { OrderRepository } from '../../domain/order/OrderRepository';
import { Order } from '../../domain/order/Order';
import { Result, ok, err } from '../../domain/shared/Result';
import { ApiClient } from '../api/ApiClient';

export class HttpOrderRepository implements OrderRepository {
  constructor(private readonly apiClient: ApiClient) {}

  async save(order: Order): Promise<Result<string, Error>> {
    try {
      const response = await this.apiClient.post('/orders', {
        id: order.id,
        ebook_id: order.ebookId,
        customer_email: order.customerEmail.toString(),
        quantity: order.quantity
      });
      return ok(response.data.id);
    } catch (error) {
      return err(new Error('Failed to save order'));
    }
  }

  async findById(id: string): Promise<Result<Order, Error>> {
    // Implementation
  }

  async findByCustomerEmail(email: string): Promise<Result<Order[], Error>> {
    // Implementation
  }
}
```

**Composition Root:**
```typescript
// src/composition/ServiceFactory.ts
import { PlaceOrderService } from '../application/orders/PlaceOrderService';
import { HttpOrderRepository } from '../infrastructure/repositories/HttpOrderRepository';
import { HttpEbookRepository } from '../infrastructure/repositories/HttpEbookRepository';
import { EmailNotificationService } from '../infrastructure/notifications/EmailNotificationService';
import { ApiClient } from '../infrastructure/api/ApiClient';

export class ServiceFactory {
  private static apiClient = new ApiClient(import.meta.env.VITE_API_URL);
  
  static createPlaceOrderService(): PlaceOrderService {
    const orderRepo = new HttpOrderRepository(this.apiClient);
    const ebookRepo = new HttpEbookRepository(this.apiClient);
    const notificationService = new EmailNotificationService(this.apiClient);
    
    return new PlaceOrderService(orderRepo, ebookRepo, notificationService);
  }
}
```

**React Hook for DI:**
```typescript
// src/hooks/useServices.ts
import { useMemo } from 'react';
import { ServiceFactory } from '../composition/ServiceFactory';

export function usePlaceOrderService() {
  return useMemo(() => ServiceFactory.createPlaceOrderService(), []);
}
```

## Elixir/Phoenix Project Structure

Here's how to organize an Elixir/Phoenix project with layered architecture:

```
lib/
├── shop/                            # Application Context
│   ├── domain/                      # Domain Layer
│   │   ├── order.ex                # Entity
│   │   ├── order_repository.ex     # Behaviour (interface)
│   │   ├── email.ex                # Value object
│   │   ├── money.ex
│   │   └── ebook.ex
│   │
│   ├── application/                 # Application Layer
│   │   ├── commands/
│   │   │   ├── place_order_command.ex
│   │   │   └── cancel_order_command.ex
│   │   └── services/
│   │       ├── place_order_service.ex
│   │       └── cancel_order_service.ex
│   │
│   └── infrastructure/              # Infrastructure Layer
│       ├── persistence/
│       │   ├── order_schema.ex      # Ecto schema
│       │   ├── ebook_schema.ex
│       │   └── ecto_order_repository.ex  # Behaviour implementation
│       ├── notifications/
│       │   └── email_notifier.ex
│       └── http/
│           └── order_controller.ex   # Only in older Phoenix, newer uses shop_web
│
├── shop_web/                        # Web Interface (Infrastructure)
│   ├── controllers/
│   │   └── order_controller.ex
│   ├── views/
│   │   └── order_view.ex
│   ├── templates/
│   │   └── order/
│   ├── router.ex
│   └── endpoint.ex
│
└── shop.ex                          # Application supervisor
```

### Dependency Flow

```
Controllers (shop_web/)
    ↓ uses
Application Services (shop/application/)
    ↓ uses
Domain Models (shop/domain/)
    ↑ implements
Repository Implementations (shop/infrastructure/)
```

### Example Files

**Domain Behaviour:**
```elixir
# lib/shop/domain/order_repository.ex
defmodule Shop.Domain.OrderRepository do
  @moduledoc """
  Repository behaviour for Orders.
  Defines what the domain needs from persistence.
  """

  alias Shop.Domain.Order

  @callback save(Order.t()) :: {:ok, String.t()} | {:error, term()}
  @callback find_by_id(String.t()) :: {:ok, Order.t()} | {:error, term()}
  @callback find_by_customer_email(String.t()) :: {:ok, [Order.t()]} | {:error, term()}
end
```

**Infrastructure Implementation:**
```elixir
# lib/shop/infrastructure/persistence/ecto_order_repository.ex
defmodule Shop.Infrastructure.Persistence.EctoOrderRepository do
  @moduledoc """
  Ecto-based implementation of OrderRepository.
  """

  @behaviour Shop.Domain.OrderRepository

  alias Shop.Repo
  alias Shop.Infrastructure.Persistence.OrderSchema
  alias Shop.Domain.{Order, Email, Money}

  @impl true
  def save(%Order{} = order) do
    params = %{
      id: order.id,
      ebook_id: order.ebook_id,
      customer_email: Email.to_string(order.customer_email),
      quantity: order.quantity,
      price_per_unit: Money.to_integer(order.price_per_unit),
      total: Money.to_integer(Order.calculate_total(order))
    }

    %OrderSchema{}
    |> OrderSchema.changeset(params)
    |> Repo.insert()
    |> case do
      {:ok, schema} -> {:ok, schema.id}
      {:error, changeset} -> {:error, {:validation_failed, changeset}}
    end
  end

  @impl true
  def find_by_id(id) do
    case Repo.get(OrderSchema, id) do
      nil -> {:error, :not_found}
      schema -> schema_to_domain(schema)
    end
  end

  @impl true
  def find_by_customer_email(email) do
    schemas = 
      OrderSchema
      |> where([o], o.customer_email == ^email)
      |> Repo.all()

    orders = Enum.map(schemas, fn schema ->
      case schema_to_domain(schema) do
        {:ok, order} -> order
        {:error, _} -> nil
      end
    end)
    |> Enum.filter(&(&1 != nil))

    {:ok, orders}
  end

  defp schema_to_domain(%OrderSchema{} = schema) do
    Order.new(
      schema.id,
      schema.ebook_id,
      schema.customer_email,
      schema.quantity,
      Money.new(schema.price_per_unit, :USD)
    )
  end
end
```

**Configuration:**
```elixir
# config/config.exs
config :shop,
  order_repository: Shop.Infrastructure.Persistence.EctoOrderRepository,
  notification_service: Shop.Infrastructure.Notifications.EmailNotifier

# config/test.exs
config :shop,
  order_repository: Shop.Test.Support.InMemoryOrderRepository,
  notification_service: Shop.Test.Support.MockNotifier
```

**Application Service Using Config:**
```elixir
# lib/shop/application/services/place_order_service.ex
defmodule Shop.Application.Services.PlaceOrderService do
  @order_repo Application.compile_env!(:shop, :order_repository)
  @notification_service Application.compile_env!(:shop, :notification_service)

  alias Shop.Application.Commands.PlaceOrderCommand
  alias Shop.Domain.Order

  def execute(%PlaceOrderCommand{} = command) do
    # Uses configured implementations
    with {:ok, ebook} <- fetch_ebook(command.ebook_id),
         {:ok, order} <- create_order(command, ebook),
         {:ok, order_id} <- @order_repo.save(order) do
      @notification_service.send_order_confirmation(order)
      {:ok, order_id}
    end
  end

  defp fetch_ebook(ebook_id) do
    # Implementation
  end

  defp create_order(command, ebook) do
    Order.new(
      generate_id(),
      command.ebook_id,
      command.customer_email,
      command.quantity,
      ebook.price
    )
  end

  defp generate_id, do: Ecto.UUID.generate()
end
```


## Communication Between Layers

Let's trace how a request flows through all the layers.

### React Example: User Places an Order

```
1. User clicks "Place Order" button
   ↓
2. OrderForm.tsx (Infrastructure - UI)
   - Handles form submission
   - Creates PlaceOrderCommand from form data
   ↓
3. PlaceOrderCommand (Application)
   - Validates structural data (email format, quantity > 0)
   ↓
4. PlaceOrderService.execute(command) (Application)
   - Fetches ebook via EbookRepository
   - Creates Order domain entity
   - Order validates business rules
   ↓
5. Order.create() (Domain)
   - Validates business rules (quantity limits, email format)
   - Calculates totals, discounts, tax
   - Returns Result<Order, Error>
   ↓
6. orderRepository.save(order) (Domain Interface)
   - Application calls the interface
   ↓
7. HttpOrderRepository.save() (Infrastructure)
   - Converts Order to JSON
   - Makes HTTP POST request
   - Returns Result<string, Error> with order ID
   ↓
8. NotificationService.send() (Infrastructure)
   - Sends confirmation email
   ↓
9. Result flows back through layers
   - Service returns order ID to component
   - Component updates UI with success message
```

**Code Example:**
```typescript
// 1. User interaction (Infrastructure)
function OrderForm() {
  const service = usePlaceOrderService();
  
  const handleSubmit = async (e) => {
    // 2. Create command (Application)
    const command = new PlaceOrderCommand(ebookId, email, quantity);
    
    // 3. Execute service (Application)
    const result = await service.execute(command);
    
    // 9. Handle result (Infrastructure)
    if (result.isOk()) {
      navigate(`/orders/${result.value}`);
    }
  };
}

// 4. Service orchestrates (Application)
class PlaceOrderService {
  async execute(command: PlaceOrderCommand) {
    // Fetch data
    const ebook = await this.ebookRepo.findById(command.ebookId);
    
    // 5. Create domain model (Domain)
    const orderResult = Order.create({
      ebookId: command.ebookId,
      customerEmail: command.customerEmail,
      quantity: command.quantity,
      pricePerUnit: ebook.price
    });
    
    if (orderResult.isErr()) return orderResult;
    
    // 6-7. Save via repository (Infrastructure)
    const orderId = await this.orderRepo.save(orderResult.value);
    
    // 8. Side effects (Infrastructure)
    await this.notifier.sendConfirmation(orderResult.value);
    
    return ok(orderId);
  }
}
```

### Elixir Example: HTTP Request to Database

```
1. HTTP POST /api/orders
   ↓
2. Router (Infrastructure)
   - Routes to OrderController.create/2
   ↓
3. OrderController.create (Infrastructure)
   - Extracts params from request
   - Creates PlaceOrderCommand
   ↓
4. PlaceOrderCommand.new(params) (Application)
   - Validates structure (types, required fields)
   ↓
5. PlaceOrderService.execute(command) (Application)
   - Orchestrates the workflow
   - Fetches ebook
   - Creates Order
   ↓
6. Order.new() (Domain)
   - Validates business rules
   - Calculates totals
   ↓
7. OrderRepository.save(order) (Domain Behaviour)
   - Application calls behaviour
   ↓
8. EctoOrderRepository.save() (Infrastructure)
   - Converts to Ecto schema
   - Inserts into database
   - Returns {:ok, order_id}
   ↓
9. Controller returns JSON response
   - 201 Created with order_id
```

**Code Example:**
```elixir
# 1-2. Router (Infrastructure)
scope "/api", ShopWeb do
  post "/orders", OrderController, :create
end

# 3. Controller (Infrastructure)
defmodule ShopWeb.OrderController do
  def create(conn, params) do
    with {:ok, command} <- PlaceOrderCommand.new(params),  # 4
         {:ok, order_id} <- PlaceOrderService.execute(command) do  # 5-8
      conn  # 9
      |> put_status(:created)
      |> json(%{order_id: order_id})
    end
  end
end

# 5. Service (Application)
defmodule Shop.Application.PlaceOrderService do
  @order_repo Application.compile_env!(:shop, :order_repository)
  
  def execute(%PlaceOrderCommand{} = command) do
    with {:ok, ebook} <- fetch_ebook(command.ebook_id),
         {:ok, order} <- Order.new(...),  # 6
         {:ok, order_id} <- @order_repo.save(order) do  # 7-8
      {:ok, order_id}
    end
  end
end

# 6. Domain
defmodule Shop.Domain.Order do
  def new(id, ebook_id, email, quantity, price) do
    # Business validation
    # Calculations
    {:ok, %Order{...}}
  end
end

# 8. Infrastructure
defmodule Shop.Infrastructure.EctoOrderRepository do
  @behaviour Shop.Domain.OrderRepository
  
  def save(%Order{} = order) do
    # Convert to schema and insert
  end
end
```

### Key Observations

1. **Each layer has a clear job**:
   - Infrastructure: Handle external systems (HTTP, DB, UI)
   - Application: Orchestrate the use case
   - Domain: Execute business logic

2. **Dependencies flow inward**:
   - Infrastructure depends on Application and Domain
   - Application depends on Domain
   - Domain depends on nothing

3. **Data transforms as it flows**:
   - HTTP params → Command → Domain Model → Database Schema
   - Each layer speaks its own language

4. **Errors propagate outward**:
   - Domain errors (business rule violations)
   - Application errors (not found, etc.)
   - Infrastructure errors (network, database)
   - All handled at appropriate layer

## Before/After Comparison

Let's see the transformation side-by-side.

### Before: No Layers

```typescript
// Everything in one file
async function placeOrder(ebookId, email, quantity) {
  // Validation
  if (!email.includes('@')) throw new Error('Invalid email');
  
  // Fetch data
  const ebook = await axios.get(`/api/ebooks/${ebookId}`);
  
  // Business logic
  const total = ebook.price * quantity;
  const tax = total * 0.08;
  const discount = quantity >= 5 ? total * 0.1 : 0;
  
  // Save
  await axios.post('/api/orders', { /* ... */ });
  
  // Side effects
  await axios.post('/api/emails/send', { /* ... */ });
}
```

**Problems:**
- ❌ Can't test without HTTP
- ❌ Can't reuse from CLI or jobs
- ❌ Business logic mixed with infrastructure
- ❌ Hard to change any part
- ❌ No clear structure

### After: Layered

```typescript
// Domain Layer
class Order {
  static create(data) { /* validation */ }
  calculateTotal() { /* business logic */ }
  calculateDiscount() { /* business logic */ }
}

// Application Layer
class PlaceOrderService {
  async execute(command: PlaceOrderCommand) {
    const order = Order.create(/* ... */);
    await this.orderRepo.save(order);
    await this.notifier.send(order);
  }
}

// Infrastructure Layer
class HttpOrderRepository {
  async save(order) { /* HTTP call */ }
}

class OrderForm {
  const handleSubmit = async () => {
    const command = new PlaceOrderCommand(/* ... */);
    await service.execute(command);
  };
}
```

**Benefits:**
- ✅ Test each layer independently
- ✅ Reuse service from anywhere
- ✅ Business logic isolated
- ✅ Easy to change any layer
- ✅ Clear structure and responsibilities

## Testing Strategy by Layer

Each layer has a different testing approach.

### Domain Layer: Pure Unit Tests

**No mocks needed—domain is pure:**

```typescript
// domain/Order.test.ts
describe('Order', () => {
  it('calculates discount for bulk orders', () => {
    const order = Order.create({
      id: 'order-1',
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 10,  // Bulk order
      pricePerUnit: Money.fromAmount(10, 'USD')
    }).value;

    const discount = order.calculateDiscount();
    
    // 10 * $10 = $100
    // 10% discount = $10
    expect(discount.amount).toBe(10);
  });

  it('rejects invalid quantity', () => {
    const result = Order.create({
      id: 'order-1',
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 0,  // Invalid
      pricePerUnit: Money.fromAmount(10, 'USD')
    });

    expect(result.isErr()).toBe(true);
    expect(result.error.message).toContain('positive');
  });
});
```

**Elixir:**
```elixir
# test/shop/domain/order_test.exs
defmodule Shop.Domain.OrderTest do
  use ExUnit.Case, async: true
  
  alias Shop.Domain.{Order, Money}

  describe "calculate_discount/1" do
    test "applies 10% discount for orders >= 5" do
      {:ok, order} = Order.new(
        "order-1",
        "ebook-1",
        "test@example.com",
        10,
        Money.new(1000, :USD)
      )

      discount = Order.calculate_discount(order)
      
      # 10 * $10.00 = $100.00
      # 10% = $10.00 = 1000 cents
      assert Money.to_integer(discount) == 1000
    end
  end

  describe "new/5" do
    test "rejects invalid quantity" do
      assert {:error, :quantity_must_be_positive} = Order.new(
        "order-1",
        "ebook-1",
        "test@example.com",
        0,  # Invalid
        Money.new(1000, :USD)
      )
    end
  end
end
```

**Characteristics:**
- ✅ Fast (milliseconds)
- ✅ No external dependencies
- ✅ No mocks needed
- ✅ Test business logic thoroughly

### Application Layer: Service Tests with Mocks

**Mock repositories and infrastructure:**

```typescript
// application/PlaceOrderService.test.ts
describe('PlaceOrderService', () => {
  let service: PlaceOrderService;
  let orderRepo: InMemoryOrderRepository;
  let ebookRepo: InMemoryEbookRepository;
  let notifier: MockNotifier;

  beforeEach(() => {
    orderRepo = new InMemoryOrderRepository();
    ebookRepo = new InMemoryEbookRepository();
    notifier = new MockNotifier();
    
    service = new PlaceOrderService(orderRepo, ebookRepo, notifier);
    
    // Seed test data
    ebookRepo.save(Ebook.create({
      id: 'ebook-1',
      title: 'Clean Code',
      price: Money.fromAmount(29.99, 'USD')
    }).value);
  });

  it('places order successfully', async () => {
    const command = new PlaceOrderCommand(
      'ebook-1',
      'test@example.com',
      2
    );

    const result = await service.execute(command);

    expect(result.isOk()).toBe(true);
    
    // Verify order was saved
    const orders = await orderRepo.findAll();
    expect(orders).toHaveLength(1);
    expect(orders[0].quantity).toBe(2);
    
    // Verify notification sent
    expect(notifier.sentEmails).toHaveLength(1);
  });

  it('fails when ebook not found', async () => {
    const command = new PlaceOrderCommand(
      'nonexistent',
      'test@example.com',
      1
    );

    const result = await service.execute(command);

    expect(result.isErr()).toBe(true);
    
    // Verify no order was created
    const orders = await orderRepo.findAll();
    expect(orders).toHaveLength(0);
  });
});
```

**Elixir:**
```elixir
# test/shop/application/place_order_service_test.exs
defmodule Shop.Application.PlaceOrderServiceTest do
  use Shop.DataCase, async: true

  alias Shop.Application.{PlaceOrderService, PlaceOrderCommand}
  alias Shop.Test.Support.{EbookFactory, OrderFactory}

  describe "execute/1" do
    setup do
      # Use test database
      ebook = EbookFactory.insert_ebook(%{
        id: "ebook-1",
        price: Money.new(2999, :USD)
      })
      
      %{ebook: ebook}
    end

    test "places order successfully", %{ebook: ebook} do
      {:ok, command} = PlaceOrderCommand.new(%{
        ebook_id: ebook.id,
        customer_email: "test@example.com",
        quantity: 2
      })

      assert {:ok, order_id} = PlaceOrderService.execute(command)
      assert is_binary(order_id)
      
      # Verify in database
      assert {:ok, order} = OrderRepository.find_by_id(order_id)
      assert order.quantity == 2
    end

    test "fails when ebook not found" do
      {:ok, command} = PlaceOrderCommand.new(%{
        ebook_id: "nonexistent",
        customer_email: "test@example.com",
        quantity: 1
      })

      assert {:error, :ebook_not_found} = PlaceOrderService.execute(command)
    end
  end
end
```

**Characteristics:**
- ✅ Test use case workflows
- ✅ Use in-memory implementations
- ✅ Test error handling
- ✅ Verify side effects

### Infrastructure Layer: Integration Tests

**Test with real systems:**

```typescript
// infrastructure/HttpOrderRepository.integration.test.ts
describe('HttpOrderRepository (Integration)', () => {
  let repository: HttpOrderRepository;
  let testServer: TestServer;

  beforeAll(async () => {
    testServer = await createTestServer();
    repository = new HttpOrderRepository(testServer.url);
  });

  afterAll(async () => {
    await testServer.close();
  });

  it('saves order to API', async () => {
    const order = Order.create({
      id: 'order-1',
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 2,
      pricePerUnit: Money.fromAmount(29.99, 'USD')
    }).value;

    const result = await repository.save(order);

    expect(result.isOk()).toBe(true);
    
    // Verify it was actually saved
    const fetchResult = await repository.findById(result.value);
    expect(fetchResult.isOk()).toBe(true);
  });
});
```

**Elixir:**
```elixir
# test/shop/infrastructure/ecto_order_repository_test.exs
defmodule Shop.Infrastructure.EctoOrderRepositoryTest do
  use Shop.DataCase, async: true

  alias Shop.Infrastructure.Persistence.EctoOrderRepository
  alias Shop.Domain.{Order, Money}

  describe "save/1" do
    test "persists order to database" do
      {:ok, order} = Order.new(
        "order-1",
        "ebook-1",
        "test@example.com",
        2,
        Money.new(2999, :USD)
      )

      assert {:ok, order_id} = EctoOrderRepository.save(order)
      
      # Verify in database
      assert {:ok, fetched_order} = EctoOrderRepository.find_by_id(order_id)
      assert fetched_order.quantity == 2
    end
  end
end
```

**Characteristics:**
- ✅ Test with real databases/APIs
- ✅ Slower than unit tests
- ✅ Test adapters work correctly
- ✅ Catch integration issues

## Common Pitfalls

### ❌ Pitfall 1: Skipping Layers

```typescript
// ❌ BAD: UI calls domain directly
function OrderForm() {
  const handleSubmit = async () => {
    // Skipping application layer!
    const order = Order.create(data);
    await httpOrderRepo.save(order);
  };
}
```

**Why it's bad:**
- No single place for the use case
- Can't add cross-cutting concerns (logging, transactions)
- Duplicates orchestration logic

```typescript
// ✅ GOOD: UI → Application → Domain
function OrderForm() {
  const service = usePlaceOrderService();
  
  const handleSubmit = async () => {
    const command = new PlaceOrderCommand(data);
    await service.execute(command);
  };
}
```

### ❌ Pitfall 2: Wrong Dependency Direction

```typescript
// ❌ BAD: Domain depends on infrastructure
class Order {
  async save() {
    // Domain calling database directly!
    await db.query('INSERT INTO orders...');
  }
}
```

**Why it's bad:**
- Can't test domain without database
- Can't change database without changing domain
- Violates the dependency rule

```typescript
// ✅ GOOD: Infrastructure depends on domain
interface OrderRepository {
  save(order: Order): Promise<Result<string, Error>>;
}

class Order {
  // No save method - just business logic
  calculateTotal() { /* ... */ }
}

class PostgresOrderRepository implements OrderRepository {
  async save(order: Order) {
    // Infrastructure knows about domain, not vice versa
  }
}
```

### ❌ Pitfall 3: Leaking Implementation Details

```typescript
// ❌ BAD: Service returns database entity
class PlaceOrderService {
  async execute(command): Promise<OrderEntity> {
    // Returns Ecto/ActiveRecord entity
    return await Order.insert(data);
  }
}
```

**Why it's bad:**
- Exposes infrastructure to upper layers
- Couples everything to database structure
- Can't change implementation

```typescript
// ✅ GOOD: Return domain model or DTO
class PlaceOrderService {
  async execute(command): Promise<Result<string, Error>> {
    const order = Order.create(data);
    const orderId = await this.orderRepo.save(order);
    return ok(orderId);  // Just the ID
  }
}
```

### ❌ Pitfall 4: Too Many Layers

```typescript
// ❌ BAD: Over-engineering
Domain
  ↓
DomainService
  ↓
ApplicationService
  ↓
ApplicationServiceFacade
  ↓
ServiceProxy
  ↓
ServiceAdapter
  ↓
RepositoryInterface
  ↓
RepositoryImplementation
  ↓
DatabaseAdapter
```

**Keep it simple:** Domain → Application → Infrastructure is enough for most apps.

### ❌ Pitfall 5: Business Logic in Infrastructure

```typescript
// ❌ BAD: Business logic in controller
class OrderController {
  async create(req, res) {
    // Business logic in controller!
    const total = req.body.quantity * req.body.price;
    const discount = total > 100 ? total * 0.1 : 0;
    // ...
  }
}
```

```typescript
// ✅ GOOD: Business logic in domain
class Order {
  calculateTotal() {
    // Business logic here
  }
}

class OrderController {
  async create(req, res) {
    // Just orchestration
    const command = PlaceOrderCommand.fromRequest(req);
    const result = await this.service.execute(command);
    res.json(result);
  }
}
```

## When to Use Layered Architecture

### Good Fit ✅

**Complex Business Logic:**
- Rich domain models with business rules
- Multiple validation scenarios
- Complex calculations
- State machines

**Long-Lived Applications:**
- Will be maintained for years
- Team needs to understand structure
- Requirements will evolve

**Team Collaboration:**
- Multiple developers working simultaneously
- Clear boundaries help parallel work
- Easier code reviews

**Need for Flexibility:**
- Might change databases
- Might change UI framework
- Multiple clients (web, mobile, CLI)

### Might Be Overkill ❌

**Simple CRUD Apps:**
```typescript
// Simple read/write, no business logic
async function getUser(id) {
  return await db.users.findOne({ id });
}

async function updateUser(id, data) {
  return await db.users.update({ id }, data);
}
// Layers add complexity with no benefit
```

**Prototypes/MVPs:**
- Speed > structure
- Unclear requirements
- Might throw away code

**Very Small Applications:**
- < 5 entities
- Single developer
- Simple use cases

**Scripts and Tools:**
- One-off data migrations
- CLI tools
- Build scripts

## Composition Root

The composition root is where everything gets wired together. This is the only place where you new up concrete implementations.

### React Composition Root

```typescript
// src/composition/ServiceFactory.ts
import { PlaceOrderService } from '../application/orders/PlaceOrderService';
import { HttpOrderRepository } from '../infrastructure/repositories/HttpOrderRepository';
import { HttpEbookRepository } from '../infrastructure/repositories/HttpEbookRepository';
import { EmailNotificationService } from '../infrastructure/notifications/EmailNotificationService';
import { ApiClient } from '../infrastructure/api/ApiClient';

// Configuration
const config = {
  apiUrl: import.meta.env.VITE_API_URL || 'http://localhost:3000/api',
};

// Shared instances
const apiClient = new ApiClient(config.apiUrl);

// Factory functions
export class ServiceFactory {
  static createPlaceOrderService(): PlaceOrderService {
    const orderRepo = new HttpOrderRepository(apiClient);
    const ebookRepo = new HttpEbookRepository(apiClient);
    const notificationService = new EmailNotificationService(apiClient);
    
    return new PlaceOrderService(orderRepo, ebookRepo, notificationService);
  }
  
  static createCancelOrderService(): CancelOrderService {
    const orderRepo = new HttpOrderRepository(apiClient);
    
    return new CancelOrderService(orderRepo);
  }
}

// React hooks for dependency injection
export function usePlaceOrderService() {
  return useMemo(() => ServiceFactory.createPlaceOrderService(), []);
}

export function useCancelOrderService() {
  return useMemo(() => ServiceFactory.createCancelOrderService(), []);
}
```

**Usage in React:**
```typescript
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/orders/new" element={<OrderForm />} />
        <Route path="/orders/:id" element={<OrderDetail />} />
      </Routes>
    </BrowserRouter>
  );
}

function OrderForm({ ebookId }) {
  // Dependency injection via hook
  const placeOrderService = usePlaceOrderService();
  
  const handleSubmit = async (data) => {
    const command = new PlaceOrderCommand(ebookId, data.email, data.quantity);
    const result = await placeOrderService.execute(command);
    // ...
  };
  
  return <form onSubmit={handleSubmit}>{/* ... */}</form>;
}
```

### Elixir Composition Root

```elixir
# config/config.exs - Configuration
config :shop,
  order_repository: Shop.Infrastructure.Persistence.EctoOrderRepository,
  ebook_repository: Shop.Infrastructure.Persistence.EctoEbookRepository,
  notification_service: Shop.Infrastructure.Notifications.EmailNotifier

# config/test.exs - Test configuration
config :shop,
  order_repository: Shop.Test.Support.InMemoryOrderRepository,
  ebook_repository: Shop.Test.Support.InMemoryEbookRepository,
  notification_service: Shop.Test.Support.MockNotifier

# lib/shop/application/services/place_order_service.ex
defmodule Shop.Application.Services.PlaceOrderService do
  # Get configured implementations
  @order_repo Application.compile_env!(:shop, :order_repository)
  @ebook_repo Application.compile_env!(:shop, :ebook_repository)
  @notifier Application.compile_env!(:shop, :notification_service)
  
  def execute(%PlaceOrderCommand{} = command) do
    # Use configured implementations
    with {:ok, ebook} <- @ebook_repo.find_by_id(command.ebook_id),
         {:ok, order} <- create_order(command, ebook),
         {:ok, order_id} <- @order_repo.save(order) do
      @notifier.send_order_confirmation(order)
      {:ok, order_id}
    end
  end
  
  defp create_order(command, ebook) do
    Order.new(
      generate_id(),
      command.ebook_id,
      command.customer_email,
      command.quantity,
      ebook.price
    )
  end
  
  defp generate_id, do: Ecto.UUID.generate()
end
```

**Usage in Controller:**
```elixir
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller

  alias Shop.Application.{PlaceOrderService, PlaceOrderCommand}

  def create(conn, params) do
    with {:ok, command} <- PlaceOrderCommand.new(params),
         {:ok, order_id} <- PlaceOrderService.execute(command) do
      conn
      |> put_status(:created)
      |> json(%{order_id: order_id})
    else
      {:error, reason} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: to_string(reason)})
    end
  end
end
```

## Conclusion

Layered Architecture provides a proven way to organize web applications for maintainability, testability, and flexibility. By separating your code into Domain, Application, and Infrastructure layers with clear dependency rules, you create a structure that's easy to understand, change, and test.

### Key Takeaways

**The Three Layers:**
1. **Domain** (Core): Pure business logic, no dependencies
2. **Application**: Use case orchestration, coordinates domain and infrastructure
3. **Infrastructure**: External systems (DB, HTTP, UI), implements interfaces

**The Dependency Rule:**
- Dependencies point inward: Infrastructure → Application → Domain
- Domain depends on nothing
- Infrastructure implements interfaces defined by inner layers

**Benefits:**
- ✅ Testable: Each layer tested independently
- ✅ Flexible: Swap implementations without touching other layers
- ✅ Maintainable: Changes localized to specific layers
- ✅ Understandable: Clear structure guides developers

**When to Use:**
- Complex business logic
- Long-lived applications
- Team collaboration needed
- Multiple clients or changing technology

**When to Skip:**
- Simple CRUD operations
- Prototypes and MVPs
- Very small applications
- Scripts and tools

### Putting It All Together

You've now seen how domain models, repositories, application services, commands, and handlers all fit together in a cohesive architecture. Each pattern serves a purpose, and layered architecture is the structure that makes them work together harmoniously.

**Remember:**
- Start simple, add layers as needed
- Keep dependencies pointing inward
- Test each layer appropriately
- Don't over-engineer for simple cases
- Focus on business value, not perfect architecture

### What's Next

- **Chapter 10: Hexagonal Architecture** - Alternative to layered architecture with explicit ports and adapters
- **Chapter 11: Testing Strategies** - Deep dive into testing each layer effectively
- **Chapter 12: Refactoring to Layers** - Step-by-step guide to introduce layers to existing code

### Further Reading

- *Domain-Driven Design* by Eric Evans
- *Clean Architecture* by Robert C. Martin  
- *Implementing Domain-Driven Design* by Vaughn Vernon
- *Patterns of Enterprise Application Architecture* by Martin Fowler

---

**Next**: [Chapter 10: Hexagonal Architecture](10-hexagonal-architecture.md)  
**Previous**: [Chapter 8: Command Pattern](08-command-pattern.md)

*Remember: Layered architecture is about separation of concerns and managing dependencies. Keep your domain pure, your application thin, and your infrastructure isolated. Your future self will thank you.* ✨

