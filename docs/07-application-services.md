# Chapter 7: Application Services

## Introduction

Application Services are the orchestration layer of your architecture. They sit between the presentation layer (UI components or controllers) and the domain layer, coordinating the execution of use cases. Unlike domain services that contain core business rules, application services are workflow coordinators—they fetch data through repositories, invoke domain objects to apply business logic, persist results, and trigger side effects.

In this chapter, we'll explore how to implement Application Services in both React/TypeScript and Elixir/Phoenix applications. You'll learn to recognize when business logic has leaked into the wrong layer and how to refactor it into well-structured services that make your codebase more maintainable, testable, and clear.

### Why Application Services Matter

Without application services, business workflows get scattered across your codebase:
- Controllers become bloated with orchestration logic
- React components mix UI concerns with use case execution
- The same workflow gets duplicated across multiple endpoints or components
- Testing requires spinning up the entire stack
- Changing a workflow means hunting through controller/component code

Application services solve these problems by giving each use case a single, clear home.

### What You'll Learn

- The distinction between Application Services and Domain Services
- How to structure services around use cases
- The Command pattern for representing user intentions
- Implementing services in TypeScript and Elixir
- Transaction management at the application layer
- Testing strategies for isolated service testing
- Common pitfalls and best practices

## Table of Contents

1. [The Problem: Scattered Business Logic](#the-problem-scattered-business-logic)
2. [The Solution: Application Services](#the-solution-application-services)
3. [Application Service vs Domain Service](#application-service-vs-domain-service)
4. [The Command Pattern](#the-command-pattern)
5. [React/TypeScript Implementation](#reacttypescript-implementation)
6. [Elixir/Phoenix Implementation](#elixirphoenix-implementation)
7. [Transaction Management](#transaction-management)
8. [Error Handling](#error-handling)
9. [Testing Application Services](#testing-application-services)
10. [Integration Patterns](#integration-patterns)
11. [Best Practices](#best-practices)
12. [Common Pitfalls](#common-pitfalls)
13. [When NOT to Use](#when-not-to-use)
14. [Conclusion](#conclusion)

## The Problem: Scattered Business Logic

Let's look at what happens when you don't use application services. Business logic ends up scattered across controllers and components, making code hard to test, reuse, and maintain.

### Before: Logic in React Component

```typescript
// OrderButton.tsx - Everything mixed together
import React, { useState } from 'react';

interface Props {
  ebookId: string;
  quantity: number;
}

function OrderButton({ ebookId, quantity }: Props) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [email, setEmail] = useState('');

  const handlePlaceOrder = async () => {
    setLoading(true);
    setError(null);

    try {
      // Validation logic in component
      if (!email || !email.includes('@')) {
        setError('Invalid email address');
        setLoading(false);
        return;
      }

      if (quantity <= 0) {
        setError('Quantity must be positive');
        setLoading(false);
        return;
      }

      // Fetch ebook details
      const ebookResponse = await fetch(`/api/ebooks/${ebookId}`);
      if (!ebookResponse.ok) {
        throw new Error('Ebook not found');
      }
      const ebook = await ebookResponse.json();

      // Business calculation
      const totalPrice = ebook.price * quantity;

      // Create order
      const orderResponse = await fetch('/api/orders', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ebook_id: ebookId,
          customer_email: email,
          quantity: quantity,
          total: totalPrice
        })
      });

      if (!orderResponse.ok) {
        throw new Error('Failed to create order');
      }

      const order = await orderResponse.json();

      // Send notification
      await fetch('/api/notifications/send', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          type: 'order_confirmation',
          email: email,
          order_id: order.id
        })
      });

      // Success handling
      alert(`Order placed! ID: ${order.id}`);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Your email"
      />
      <button onClick={handlePlaceOrder} disabled={loading}>
        {loading ? 'Processing...' : 'Place Order'}
      </button>
      {error && <div className="error">{error}</div>}
    </div>
  );
}
```

### Before: Logic in Phoenix Controller

```elixir
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller

  def create(conn, params) do
    # Validation in controller
    with {:ok, validated_params} <- validate_params(params),
         # Fetch ebook
         {:ok, ebook} <- fetch_ebook(validated_params.ebook_id),
         # Calculate total
         total <- calculate_total(ebook.price, validated_params.quantity),
         # Create order
         {:ok, order} <- create_order(validated_params, total),
         # Send notification
         :ok <- send_notification(order) do
      conn
      |> put_status(:created)
      |> json(%{order_id: order.id})
    else
      {:error, :invalid_params} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Invalid parameters"})

      {:error, :ebook_not_found} ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Ebook not found"})

      {:error, reason} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: reason})
    end
  end

  defp validate_params(params) do
    # Validation logic
  end

  defp fetch_ebook(ebook_id) do
    # Database query
  end

  defp calculate_total(price, quantity) do
    # Business calculation
  end

  defp create_order(params, total) do
    # Database insertion
  end

  defp send_notification(order) do
    # Email sending
  end
end
```

### Problems with This Approach

1. **Hard to Test**: Testing requires mocking HTTP calls or database access
2. **Not Reusable**: Can't reuse this logic from a CLI command, background job, or different endpoint
3. **Mixed Responsibilities**: UI/HTTP concerns mixed with business workflow
4. **Hard to Change**: Changing the order placement workflow means editing controller/component code
5. **Domain Logic Leaks Out**: Business rules are scattered, not centralized in domain models
6. **Transaction Boundaries Unclear**: Where does the atomic operation begin and end?

## The Solution: Application Services

Application services solve these problems by extracting the use case workflow into a dedicated service class or module. The service becomes the **single source of truth** for how a particular use case should execute.

### Key Characteristics

**Application Services:**
- ✅ Orchestrate use cases (the "what" of the workflow)
- ✅ Coordinate domain objects and repositories
- ✅ Define transaction boundaries
- ✅ Handle cross-cutting concerns (logging, events, notifications)
- ✅ Are stateless (no instance variables that change)
- ✅ Have one public method/function per service (one use case)
- ✅ Accept commands as input (structured data)
- ✅ Return results (success/failure, IDs, or domain models)

**Application Services DON'T:**
- ❌ Contain core business rules (that's domain models)
- ❌ Access infrastructure directly (use repositories/ports)
- ❌ Make UI decisions (return data, not views)
- ❌ Maintain state between calls

### Service Structure

```
┌─────────────────────────────────────┐
│     Presentation Layer              │
│  (Controllers, Components)          │
└──────────────┬──────────────────────┘
               │ Commands
               ▼
┌─────────────────────────────────────┐
│     Application Services            │
│  - Orchestrate use cases            │
│  - Define transactions              │
│  - Trigger side effects             │
└──────────────┬──────────────────────┘
               │ Uses
               ▼
┌─────────────────────────────────────┐
│     Domain Layer                    │
│  (Entities, Value Objects,          │
│   Domain Services)                  │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│     Infrastructure                  │
│  (Repositories, APIs, Email)        │
└─────────────────────────────────────┘
```

## Application Service vs Domain Service

This distinction trips up many developers. Let's clarify:

### Domain Services

**Purpose**: Contain core business logic that doesn't naturally fit in an entity or value object

**Characteristics**:
- Implement business rules
- Work with domain models
- Are pure/stateless
- No infrastructure dependencies
- Part of the domain layer

**Example**: `PricingService` that calculates discounts based on customer tier and product category

```typescript
// Domain Service
class PricingService {
  calculateDiscount(
    customer: Customer,
    product: Product,
    quantity: number
  ): Money {
    // Complex business rules for pricing
    if (customer.tier === 'premium' && quantity > 10) {
      return product.price.multiply(0.15);
    }
    return Money.zero();
  }
}
```

### Application Services

**Purpose**: Orchestrate use cases by coordinating domain objects, repositories, and infrastructure

**Characteristics**:
- Coordinate workflows
- Call domain services and entities
- Use repositories for persistence
- Trigger side effects (emails, events)
- Define transaction boundaries
- Part of the application layer

**Example**: `PlaceOrderService` that orchestrates the entire order placement workflow

```typescript
// Application Service
class PlaceOrderService {
  constructor(
    private orderRepo: OrderRepository,
    private pricingService: PricingService, // Domain service
    private notificationService: NotificationService
  ) {}

  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // Orchestrate the use case
    const discount = this.pricingService.calculateDiscount(...);
    const order = Order.create(...);
    const orderId = await this.orderRepo.save(order);
    await this.notificationService.send(...);
    return ok(orderId);
  }
}
```

### Quick Reference

| Aspect | Domain Service | Application Service |
|--------|---------------|---------------------|
| **Layer** | Domain | Application |
| **Purpose** | Business logic | Use case orchestration |
| **Dependencies** | Only domain models | Repositories, infrastructure |
| **Called by** | Application services | Controllers, components |
| **Example** | Calculate shipping cost | Place an order |

## The Command Pattern

Commands are simple data structures that represent a user's intention. They're the input to application services.

### Why Commands?

**Benefits:**
- ✅ Clear contract for what data is needed
- ✅ Easy to validate before execution
- ✅ Can be serialized (useful for queues, events)
- ✅ Self-documenting (the command name tells you what it does)
- ✅ Testable (just plain data)

### Command Characteristics

- Immutable data structures
- Named after the user intention (PlaceOrder, UpdateProfile, CancelSubscription)
- Contain all data needed to execute the use case
- No behavior (just data)
- Usually validated before reaching the service

## React/TypeScript Implementation

Let's implement the PlaceOrder use case properly with an application service.

### 1. Define the Command

```typescript
// commands/PlaceOrderCommand.ts
export interface PlaceOrderCommand {
  readonly ebookId: string;
  readonly customerEmail: string;
  readonly quantity: number;
}

// Optional: Command validation
export function validatePlaceOrderCommand(
  command: PlaceOrderCommand
): Result<PlaceOrderCommand, ValidationError> {
  const errors: string[] = [];

  if (!command.ebookId || command.ebookId.trim() === '') {
    errors.push('ebookId is required');
  }

  if (!command.customerEmail || !command.customerEmail.includes('@')) {
    errors.push('valid email is required');
  }

  if (command.quantity <= 0) {
    errors.push('quantity must be positive');
  }

  if (errors.length > 0) {
    return err(new ValidationError(errors));
  }

  return ok(command);
}
```

### 2. Implement the Application Service

```typescript
// services/PlaceOrderService.ts
import { Result, ok, err } from 'neverthrow';
import { Order } from '../domain/Order';
import { OrderRepository } from '../domain/OrderRepository';
import { EbookRepository } from '../domain/EbookRepository';
import { NotificationService } from './NotificationService';
import { PlaceOrderCommand } from '../commands/PlaceOrderCommand';

export class PlaceOrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly ebookRepository: EbookRepository,
    private readonly notificationService: NotificationService
  ) {}

  async execute(
    command: PlaceOrderCommand
  ): Promise<Result<string, Error>> {
    // 1. Fetch ebook price from repository
    const ebookResult = await this.ebookRepository.findById(command.ebookId);
    if (ebookResult.isErr()) {
      return err(new Error('Ebook not found'));
    }
    const ebook = ebookResult.value;

    // 2. Create domain model (domain logic happens here)
    const orderResult = Order.create({
      ebookId: command.ebookId,
      customerEmail: command.customerEmail,
      quantity: command.quantity,
      pricePerUnit: ebook.price
    });

    if (orderResult.isErr()) {
      return err(orderResult.error);
    }
    const order = orderResult.value;

    // 3. Persist via repository
    const saveResult = await this.orderRepository.save(order);
    if (saveResult.isErr()) {
      return err(new Error('Failed to save order'));
    }
    const orderId = saveResult.value;

    // 4. Trigger side effect (notification)
    await this.notificationService.sendOrderConfirmation(order);

    // 5. Return success with order ID
    return ok(orderId);
  }
}
```

### 3. React Component Using the Service

```typescript
// components/PlaceOrderForm.tsx
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { usePlaceOrderService } from '../hooks/useServices';
import { PlaceOrderCommand } from '../commands/PlaceOrderCommand';

interface FormData {
  ebookId: string;
  email: string;
  quantity: number;
}

export function PlaceOrderForm({ ebookId }: { ebookId: string }) {
  const navigate = useNavigate();
  const placeOrderService = usePlaceOrderService();
  
  const [email, setEmail] = useState('');
  const [quantity, setQuantity] = useState(1);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    const command: PlaceOrderCommand = {
      ebookId,
      customerEmail: email,
      quantity
    };

    const result = await placeOrderService.execute(command);

    if (result.isOk()) {
      navigate(`/orders/${result.value}`);
    } else {
      setError(result.error.message);
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
          required
        />
      </div>
      
      <div>
        <label htmlFor="quantity">Quantity:</label>
        <input
          id="quantity"
          type="number"
          value={quantity}
          onChange={(e) => setQuantity(parseInt(e.target.value, 10))}
          min="1"
          required
        />
      </div>

      <button type="submit" disabled={loading}>
        {loading ? 'Placing Order...' : 'Place Order'}
      </button>

      {error && <div className="error">{error}</div>}
    </form>
  );
}
```

### 4. Dependency Injection Hook

```typescript
// hooks/useServices.ts
import { useMemo } from 'react';
import { PlaceOrderService } from '../services/PlaceOrderService';
import { HttpOrderRepository } from '../infrastructure/HttpOrderRepository';
import { HttpEbookRepository } from '../infrastructure/HttpEbookRepository';
import { EmailNotificationService } from '../infrastructure/EmailNotificationService';

export function usePlaceOrderService(): PlaceOrderService {
  return useMemo(() => {
    const orderRepo = new HttpOrderRepository();
    const ebookRepo = new HttpEbookRepository();
    const notificationService = new EmailNotificationService();

    return new PlaceOrderService(
      orderRepo,
      ebookRepo,
      notificationService
    );
  }, []);
}
```

### Benefits of This Approach

**Component is Clean:**
- Focused on UI concerns only
- Easy to understand and maintain
- Minimal logic

**Service is Reusable:**
- Can be called from anywhere (CLI, background job, API endpoint)
- Easy to test in isolation
- Single source of truth for the workflow

**Domain Logic Stays in Domain:**
- Order validation is in the Order class
- Price calculations are in domain models
- Service just coordinates

## Elixir/Phoenix Implementation

Now let's see the same pattern in Elixir's functional paradigm.

### 1. Define the Command Struct

```elixir
# lib/shop/application/commands/place_order_command.ex
defmodule Shop.Application.PlaceOrderCommand do
  @moduledoc """
  Command to place a new order for an ebook.
  Represents the user's intention to purchase.
  """

  @enforce_keys [:ebook_id, :customer_email, :quantity]
  defstruct [:ebook_id, :customer_email, :quantity]

  @type t :: %__MODULE__{
    ebook_id: String.t(),
    customer_email: String.t(),
    quantity: pos_integer()
  }

  @spec new(map()) :: {:ok, t()} | {:error, :invalid_command}
  def new(%{"ebook_id" => ebook_id, "customer_email" => email, "quantity" => qty}) 
      when is_binary(ebook_id) and is_binary(email) and is_integer(qty) and qty > 0 do
    command = %__MODULE__{
      ebook_id: ebook_id,
      customer_email: email,
      quantity: qty
    }
    {:ok, command}
  end

  def new(_), do: {:error, :invalid_command}
end
```

### 2. Implement the Application Service

```elixir
# lib/shop/application/services/place_order_service.ex
defmodule Shop.Application.PlaceOrderService do
  @moduledoc """
  Application service for placing orders.
  Orchestrates the order placement use case.
  """

  alias Shop.Domain.{Order, OrderRepository, EbookRepository}
  alias Shop.Application.PlaceOrderCommand
  alias Shop.Infrastructure.NotificationService

  @type success :: {:ok, String.t()}
  @type failure :: {:error, :ebook_not_found | :order_creation_failed | :save_failed | term()}

  @doc """
  Executes the place order use case.
  
  Steps:
  1. Fetch ebook to get price
  2. Create domain model (applies business rules)
  3. Save order via repository
  4. Send confirmation notification
  
  Returns {:ok, order_id} or {:error, reason}
  """
  @spec execute(PlaceOrderCommand.t()) :: success() | failure()
  def execute(%PlaceOrderCommand{} = command) do
    with {:ok, ebook} <- fetch_ebook(command.ebook_id),
         {:ok, order} <- create_order(command, ebook),
         {:ok, order_id} <- save_order(order),
         :ok <- send_confirmation(order) do
      {:ok, order_id}
    end
  end

  # Private functions that orchestrate the workflow

  defp fetch_ebook(ebook_id) do
    case EbookRepository.find_by_id(ebook_id) do
      {:ok, ebook} -> {:ok, ebook}
      {:error, :not_found} -> {:error, :ebook_not_found}
    end
  end

  defp create_order(command, ebook) do
    Order.new(
      command.ebook_id,
      command.customer_email,
      command.quantity,
      ebook.price
    )
  end

  defp save_order(order) do
    case OrderRepository.save(order) do
      {:ok, saved_order} -> {:ok, saved_order.id}
      error -> error
    end
  end

  defp send_confirmation(order) do
    # Fire and forget - don't fail the order if notification fails
    Task.start(fn ->
      NotificationService.send_order_confirmation(order)
    end)
    :ok
  end
end
```

### 3. Phoenix Controller Using the Service

```elixir
# lib/shop_web/controllers/order_controller.ex
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller

  alias Shop.Application.{PlaceOrderService, PlaceOrderCommand}

  @doc """
  POST /api/orders
  
  Creates a new order using the PlaceOrderService.
  """
  def create(conn, params) do
    with {:ok, command} <- PlaceOrderCommand.new(params),
         {:ok, order_id} <- PlaceOrderService.execute(command) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/orders/#{order_id}")
      |> json(%{order_id: order_id})
    else
      {:error, :invalid_command} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Invalid parameters"})

      {:error, :ebook_not_found} ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Ebook not found"})

      {:error, reason} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: to_string(reason)})
    end
  end

  @doc """
  GET /api/orders/:id
  """
  def show(conn, %{"id" => id}) do
    # Implementation omitted for brevity
  end
end
```

### 4. Alternative: Using Ecto.Multi for Complex Workflows

For more complex workflows with multiple database operations:

```elixir
defmodule Shop.Application.PlaceOrderService do
  alias Shop.Repo
  alias Ecto.Multi

  def execute(%PlaceOrderCommand{} = command) do
    Multi.new()
    |> Multi.run(:ebook, fn _repo, _changes ->
      fetch_ebook(command.ebook_id)
    end)
    |> Multi.run(:order, fn _repo, %{ebook: ebook} ->
      create_order(command, ebook)
    end)
    |> Multi.run(:save_order, fn repo, %{order: order} ->
      OrderRepository.save(order, repo)
    end)
    |> Multi.run(:reserve_inventory, fn _repo, %{order: order} ->
      reserve_inventory(order)
    end)
    |> Multi.run(:notification, fn _repo, %{save_order: order} ->
      send_confirmation(order)
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{save_order: order}} -> {:ok, order.id}
      {:error, _step, reason, _changes} -> {:error, reason}
    end
  end
end
```

## Transaction Management

Application services are responsible for defining transaction boundaries. A transaction typically covers one complete use case.

### TypeScript Transaction Management

```typescript
// services/PlaceOrderService.ts
export class PlaceOrderService {
  constructor(
    private readonly db: Database,
    private readonly orderRepository: OrderRepository,
    private readonly inventoryRepository: InventoryRepository,
    private readonly paymentService: PaymentService
  ) {}

  async execute(
    command: PlaceOrderCommand
  ): Promise<Result<string, Error>> {
    // Begin transaction
    const transaction = await this.db.beginTransaction();

    try {
      // 1. Create order within transaction
      const order = Order.create({ /* ... */ });
      if (order.isErr()) {
        await transaction.rollback();
        return order;
      }

      // 2. Reserve inventory
      const inventoryResult = await this.inventoryRepository.reserve(
        command.ebookId,
        command.quantity,
        transaction
      );
      if (inventoryResult.isErr()) {
        await transaction.rollback();
        return inventoryResult;
      }

      // 3. Process payment
      const paymentResult = await this.paymentService.charge(
        command.customerId,
        order.value.total,
        transaction
      );
      if (paymentResult.isErr()) {
        await transaction.rollback();
        return paymentResult;
      }

      // 4. Save order
      const orderId = await this.orderRepository.save(
        order.value,
        transaction
      );

      // Commit all changes
      await transaction.commit();

      // 5. Side effects AFTER successful transaction
      await this.notificationService.sendOrderConfirmation(order.value);

      return ok(orderId);
    } catch (error) {
      await transaction.rollback();
      return err(error instanceof Error ? error : new Error('Unknown error'));
    }
  }
}
```

### Elixir Transaction Management

Elixir's `Repo.transaction/1` provides excellent transaction support:

```elixir
defmodule Shop.Application.PlaceOrderService do
  alias Shop.Repo
  alias Shop.Domain.{Order, OrderRepository, InventoryRepository}
  alias Shop.Application.PlaceOrderCommand

  @spec execute(PlaceOrderCommand.t()) :: {:ok, String.t()} | {:error, term()}
  def execute(%PlaceOrderCommand{} = command) do
    Repo.transaction(fn ->
      with {:ok, ebook} <- fetch_ebook(command.ebook_id),
           {:ok, order} <- create_order(command, ebook),
           {:ok, _} <- reserve_inventory(command.ebook_id, command.quantity),
           {:ok, _} <- process_payment(order),
           {:ok, saved_order} <- save_order(order) do
        # Return the order ID - transaction commits automatically
        saved_order.id
      else
        {:error, reason} ->
          # Rollback the transaction
          Repo.rollback(reason)
      end
    end)
    |> case do
      {:ok, order_id} ->
        # Side effects AFTER successful transaction
        send_confirmation_async(order_id)
        {:ok, order_id}

      {:error, reason} ->
        {:error, reason}
    end
  end

  defp fetch_ebook(ebook_id) do
    # Repository call
  end

  defp create_order(command, ebook) do
    Order.new(
      command.ebook_id,
      command.customer_email,
      command.quantity,
      ebook.price
    )
  end

  defp reserve_inventory(ebook_id, quantity) do
    InventoryRepository.reserve(ebook_id, quantity)
  end

  defp process_payment(order) do
    # Payment processing
  end

  defp save_order(order) do
    OrderRepository.save(order)
  end

  defp send_confirmation_async(order_id) do
    Task.start(fn ->
      # Send confirmation email
    end)
  end
end
```

### Transaction Best Practices

1. **Keep transactions short**: Only include operations that must be atomic
2. **Side effects outside**: Send emails, publish events AFTER transaction commits
3. **Idempotency**: Design operations to be safely retryable
4. **Error handling**: Always roll back on any error
5. **Logging**: Log transaction boundaries for debugging

## Error Handling

Consistent error handling is crucial for maintainable application services.

### TypeScript Error Handling with Result Types

```typescript
// Use neverthrow or similar Result type library
import { Result, ok, err } from 'neverthrow';

// Define domain errors
export class EbookNotFoundError extends Error {
  constructor(ebookId: string) {
    super(`Ebook not found: ${ebookId}`);
    this.name = 'EbookNotFoundError';
  }
}

export class InsufficientInventoryError extends Error {
  constructor(available: number, requested: number) {
    super(`Insufficient inventory: ${available} available, ${requested} requested`);
    this.name = 'InsufficientInventoryError';
  }
}

// Service returns Result type
export class PlaceOrderService {
  async execute(
    command: PlaceOrderCommand
  ): Promise<Result<string, EbookNotFoundError | InsufficientInventoryError | Error>> {
    const ebookResult = await this.ebookRepository.findById(command.ebookId);
    
    if (ebookResult.isErr()) {
      return err(new EbookNotFoundError(command.ebookId));
    }

    const ebook = ebookResult.value;

    if (ebook.stock < command.quantity) {
      return err(new InsufficientInventoryError(ebook.stock, command.quantity));
    }

    // Continue with happy path...
    return ok(orderId);
  }
}

// Component handles specific errors
const result = await placeOrderService.execute(command);

result.match(
  (orderId) => navigate(`/orders/${orderId}`),
  (error) => {
    if (error instanceof EbookNotFoundError) {
      setError('This ebook is no longer available');
    } else if (error instanceof InsufficientInventoryError) {
      setError('Not enough copies in stock');
    } else {
      setError('An unexpected error occurred');
    }
  }
);
```

### Elixir Error Handling with Tagged Tuples

```elixir
defmodule Shop.Application.PlaceOrderService do
  @type error_reason ::
    :ebook_not_found
    | :insufficient_inventory
    | :payment_failed
    | :invalid_email
    | term()

  @spec execute(PlaceOrderCommand.t()) :: {:ok, String.t()} | {:error, error_reason()}
  def execute(%PlaceOrderCommand{} = command) do
    with {:ok, ebook} <- fetch_ebook(command.ebook_id),
         :ok <- validate_inventory(ebook, command.quantity),
         {:ok, order} <- create_order(command, ebook),
         {:ok, _} <- process_payment(order),
         {:ok, order_id} <- save_order(order) do
      {:ok, order_id}
    end
  end

  defp fetch_ebook(ebook_id) do
    case EbookRepository.find_by_id(ebook_id) do
      {:ok, ebook} -> {:ok, ebook}
      {:error, :not_found} -> {:error, :ebook_not_found}
    end
  end

  defp validate_inventory(ebook, requested_quantity) do
    if ebook.stock >= requested_quantity do
      :ok
    else
      {:error, {:insufficient_inventory, ebook.stock, requested_quantity}}
    end
  end
end

# Controller handles specific errors
def create(conn, params) do
  case PlaceOrderService.execute(command) do
    {:ok, order_id} ->
      conn
      |> put_status(:created)
      |> json(%{order_id: order_id})

    {:error, :ebook_not_found} ->
      conn
      |> put_status(:not_found)
      |> json(%{error: "Ebook not found"})

    {:error, {:insufficient_inventory, available, _requested}} ->
      conn
      |> put_status(:conflict)
      |> json(%{error: "Insufficient inventory", available: available})

    {:error, :payment_failed} ->
      conn
      |> put_status(:payment_required)
      |> json(%{error: "Payment failed"})

    {:error, reason} ->
      conn
      |> put_status(:internal_server_error)
      |> json(%{error: "Internal error"})
  end
end
```

## Testing Application Services

Application services are highly testable because they coordinate dependencies that can be mocked.

### TypeScript Testing

```typescript
// __tests__/PlaceOrderService.test.ts
import { describe, it, expect, beforeEach } from '@jest/globals';
import { PlaceOrderService } from '../services/PlaceOrderService';
import { InMemoryOrderRepository } from '../test-helpers/InMemoryOrderRepository';
import { InMemoryEbookRepository } from '../test-helpers/InMemoryEbookRepository';
import { MockNotificationService } from '../test-helpers/MockNotificationService';
import { Ebook } from '../domain/Ebook';
import { Money } from '../domain/Money';

describe('PlaceOrderService', () => {
  let service: PlaceOrderService;
  let orderRepo: InMemoryOrderRepository;
  let ebookRepo: InMemoryEbookRepository;
  let notificationService: MockNotificationService;

  beforeEach(() => {
    orderRepo = new InMemoryOrderRepository();
    ebookRepo = new InMemoryEbookRepository();
    notificationService = new MockNotificationService();

    service = new PlaceOrderService(
      orderRepo,
      ebookRepo,
      notificationService
    );

    // Seed test data
    const ebook = Ebook.create({
      id: 'ebook-1',
      title: 'Clean Architecture',
      price: Money.fromAmount(29.99, 'USD'),
      stock: 100
    }).value;
    
    ebookRepo.save(ebook);
  });

  it('successfully places an order', async () => {
    const command = {
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 2
    };

    const result = await service.execute(command);

    expect(result.isOk()).toBe(true);
    const orderId = result.value;
    
    // Verify order was saved
    const savedOrder = await orderRepo.findById(orderId);
    expect(savedOrder.isOk()).toBe(true);
    expect(savedOrder.value.customerEmail).toBe('test@example.com');
    expect(savedOrder.value.quantity).toBe(2);
    
    // Verify notification was sent
    expect(notificationService.sentEmails).toHaveLength(1);
    expect(notificationService.sentEmails[0].to).toBe('test@example.com');
  });

  it('fails when ebook not found', async () => {
    const command = {
      ebookId: 'nonexistent',
      customerEmail: 'test@example.com',
      quantity: 1
    };

    const result = await service.execute(command);

    expect(result.isErr()).toBe(true);
    expect(result.error.message).toContain('not found');
    
    // Verify no order was saved
    const orders = await orderRepo.findAll();
    expect(orders).toHaveLength(0);
    
    // Verify no notification was sent
    expect(notificationService.sentEmails).toHaveLength(0);
  });

  it('fails when quantity is invalid', async () => {
    const command = {
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 0
    };

    const result = await service.execute(command);

    expect(result.isErr()).toBe(true);
  });

  it('calculates total price correctly', async () => {
    const command = {
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 3
    };

    const result = await service.execute(command);
    const order = (await orderRepo.findById(result.value)).value;

    expect(order.total.amount).toBe(89.97); // 29.99 * 3
  });
});

// Test helper: InMemoryOrderRepository
export class InMemoryOrderRepository implements OrderRepository {
  private orders = new Map<string, Order>();

  async save(order: Order): Promise<Result<string, Error>> {
    this.orders.set(order.id, order);
    return ok(order.id);
  }

  async findById(id: string): Promise<Result<Order, Error>> {
    const order = this.orders.get(id);
    if (!order) {
      return err(new Error('Order not found'));
    }
    return ok(order);
  }

  async findAll(): Promise<Order[]> {
    return Array.from(this.orders.values());
  }
}

// Test helper: MockNotificationService
export class MockNotificationService {
  sentEmails: Array<{ to: string; subject: string; body: string }> = [];

  async sendOrderConfirmation(order: Order): Promise<void> {
    this.sentEmails.push({
      to: order.customerEmail,
      subject: 'Order Confirmation',
      body: `Your order ${order.id} has been placed`
    });
  }
}
```

### Elixir Testing

```elixir
# test/shop/application/place_order_service_test.exs
defmodule Shop.Application.PlaceOrderServiceTest do
  use Shop.DataCase, async: true

  alias Shop.Application.{PlaceOrderService, PlaceOrderCommand}
  alias Shop.Domain.{Ebook, EbookRepository, OrderRepository}
  alias Shop.Domain.Money

  describe "execute/1" do
    setup do
      # Seed test data
      {:ok, ebook} = 
        %Ebook{
          id: "ebook-1",
          title: "Clean Architecture",
          price: Money.new(2999, :USD),
          stock: 100
        }
        |> EbookRepository.save()

      %{ebook: ebook}
    end

    test "successfully places an order", %{ebook: ebook} do
      command = %PlaceOrderCommand{
        ebook_id: ebook.id,
        customer_email: "test@example.com",
        quantity: 2
      }

      assert {:ok, order_id} = PlaceOrderService.execute(command)
      assert is_binary(order_id)

      # Verify order was saved
      {:ok, order} = OrderRepository.find_by_id(order_id)
      assert order.customer_email == "test@example.com"
      assert order.quantity == 2
      assert Money.amount(order.total) == 5998 # 29.99 * 2
    end

    test "fails when ebook not found" do
      command = %PlaceOrderCommand{
        ebook_id: "nonexistent",
        customer_email: "test@example.com",
        quantity: 1
      }

      assert {:error, :ebook_not_found} = PlaceOrderService.execute(command)

      # Verify no order was created
      assert [] = OrderRepository.all()
    end

    test "fails when quantity is zero" do
      command = %PlaceOrderCommand{
        ebook_id: "ebook-1",
        customer_email: "test@example.com",
        quantity: 0
      }

      # Command validation should fail
      assert {:error, :invalid_command} = PlaceOrderCommand.new(%{
        "ebook_id" => "ebook-1",
        "customer_email" => "test@example.com",
        "quantity" => 0
      })
    end

    test "fails when email is invalid" do
      command_params = %{
        "ebook_id" => "ebook-1",
        "customer_email" => "invalid",
        "quantity" => 1
      }

      # If using changeset validation
      assert {:error, :invalid_command} = PlaceOrderCommand.new(command_params)
    end
  end

  describe "with Mox for notifications" do
    import Mox

    setup :verify_on_exit!

    test "sends notification after successful order" do
      # Setup expectation
      expect(Shop.NotificationServiceMock, :send_order_confirmation, fn order ->
        assert order.customer_email == "test@example.com"
        :ok
      end)

      command = %PlaceOrderCommand{
        ebook_id: "ebook-1",
        customer_email: "test@example.com",
        quantity: 1
      }

      assert {:ok, _order_id} = PlaceOrderService.execute(command)
    end
  end
end
```

### Testing Best Practices

1. **Use in-memory implementations**: Fast tests without database
2. **Test happy path**: Verify the complete workflow works
3. **Test error cases**: Each possible failure scenario
4. **Verify side effects**: Check notifications, events were triggered
5. **Test boundaries**: Transaction commits/rollbacks work correctly
6. **Use mocks sparingly**: Prefer in-memory implementations when possible

## Integration Patterns

### Dependency Injection

**TypeScript Constructor Injection:**

```typescript
export class PlaceOrderService {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly ebookRepo: EbookRepository,
    private readonly notifications: NotificationService
  ) {}
}

// IoC container or manual injection
const service = new PlaceOrderService(
  new HttpOrderRepository(),
  new HttpEbookRepository(),
  new EmailNotificationService()
);
```

**Elixir Configuration-based Injection:**

```elixir
# config/config.exs
config :shop, :notification_service, Shop.Infrastructure.EmailNotificationService

# In service
@notification_service Application.get_env(:shop, :notification_service)

def execute(command) do
  # Use @notification_service
end
```

### Service Composition

Services can call other services for complex workflows:

```typescript
export class CheckoutService {
  constructor(
    private readonly placeOrderService: PlaceOrderService,
    private readonly sendReceiptService: SendReceiptService,
    private readonly updateInventoryService: UpdateInventoryService
  ) {}

  async execute(command: CheckoutCommand): Promise<Result<CheckoutResult, Error>> {
    // Orchestrate multiple services
    const orderResult = await this.placeOrderService.execute({
      ebookId: command.ebookId,
      customerEmail: command.email,
      quantity: command.quantity
    });

    if (orderResult.isErr()) {
      return err(orderResult.error);
    }

    await this.sendReceiptService.execute({ orderId: orderResult.value });
    await this.updateInventoryService.execute({ ebookId: command.ebookId, quantity: -command.quantity });

    return ok({ orderId: orderResult.value });
  }
}
```

### Event-Driven Architecture

Services can publish domain events:

```typescript
export class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // ... place order logic ...

    // Publish domain event
    await this.eventBus.publish(
      new OrderPlacedEvent({
        orderId: orderId,
        customerId: command.customerId,
        total: order.total,
        timestamp: new Date()
      })
    );

    return ok(orderId);
  }
}

// Event handlers react to events
class OrderPlacedEventHandler {
  async handle(event: OrderPlacedEvent): Promise<void> {
    await this.sendConfirmationEmail(event.customerId, event.orderId);
    await this.updateAnalytics(event);
    await this.notifyWarehouse(event);
  }
}
```

## Best Practices

### 1. One Service, One Use Case

```typescript
// ✅ Good: Single responsibility
class PlaceOrderService {
  execute(command: PlaceOrderCommand): Promise<Result<string, Error>>
}

class CancelOrderService {
  execute(command: CancelOrderCommand): Promise<Result<void, Error>>
}

// ❌ Bad: Multiple use cases in one service
class OrderService {
  placeOrder(command: PlaceOrderCommand): Promise<Result<string, Error>>
  cancelOrder(command: CancelOrderCommand): Promise<Result<void, Error>>
  updateOrder(command: UpdateOrderCommand): Promise<Result<void, Error>>
  refundOrder(command: RefundOrderCommand): Promise<Result<void, Error>>
}
```

### 2. Services Are Stateless

```typescript
// ✅ Good: Stateless service
class PlaceOrderService {
  constructor(private readonly orderRepo: OrderRepository) {}
  
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // All state comes from parameters
  }
}

// ❌ Bad: Stateful service
class PlaceOrderService {
  private currentOrder?: Order; // State between calls!
  
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    this.currentOrder = Order.create(...); // Mutating state
  }
}
```

### 3. Use Commands for Clear Contracts

```typescript
// ✅ Good: Structured command
interface PlaceOrderCommand {
  ebookId: string;
  customerEmail: string;
  quantity: number;
}

service.execute(command);

// ❌ Bad: Primitive parameters
service.execute(ebookId, email, quantity, price, tax, shipping);
```

### 4. Keep Services Thin

```typescript
// ✅ Good: Service orchestrates, domain does work
class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    const order = Order.create(command); // Domain logic in entity
    const orderId = await this.orderRepo.save(order);
    return ok(orderId);
  }
}

// ❌ Bad: Business logic in service
class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // Don't do calculations here!
    const total = command.quantity * command.price;
    const tax = total * 0.08;
    const grandTotal = total + tax;
    
    // Don't do validation here!
    if (grandTotal < 0) throw new Error('Invalid total');
    
    // Service should just coordinate!
  }
}
```

### 5. Return Results, Not Entities

```typescript
// ✅ Good: Return ID or simple result
async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
  return ok(orderId);
}

// ✅ Also good: Return DTO
async execute(command: PlaceOrderCommand): Promise<Result<OrderDTO, Error>> {
  return ok({ id: order.id, total: order.total, status: 'pending' });
}

// ❌ Bad: Returning full entity leaks domain details
async execute(command: PlaceOrderCommand): Promise<Result<Order, Error>> {
  return ok(order); // Now caller has access to all entity methods!
}
```

### 6. Handle Errors Consistently

```typescript
// ✅ Good: Consistent error handling
class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<Result<string, OrderError>> {
    // All errors are typed and expected
  }
}

// ❌ Bad: Throwing exceptions
class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<string> {
    throw new Error('Ebook not found'); // Unexpected for caller
  }
}
```

## Common Pitfalls

### ❌ Pitfall 1: Putting Domain Logic in Services

```typescript
// ❌ Bad: Business rule in service
class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    const total = command.quantity * command.price;
    
    // This is a business rule - belongs in domain!
    if (total > 1000) {
      const discount = total * 0.1;
      const finalTotal = total - discount;
    }
  }
}

// ✅ Good: Business rule in domain
class Order {
  calculateTotal(): Money {
    const subtotal = this.quantity * this.pricePerUnit;
    
    // Discount logic encapsulated in domain
    if (subtotal.isGreaterThan(Money.fromAmount(1000))) {
      return subtotal.applyDiscount(0.1);
    }
    
    return subtotal;
  }
}

class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    const order = Order.create(command);
    const total = order.calculateTotal(); // Domain handles it
  }
}
```

### ❌ Pitfall 2: Service Calling Service Calling Service

```typescript
// ❌ Bad: Deep service chains
class CheckoutService {
  async execute(cmd: CheckoutCommand) {
    await this.placeOrderService.execute(cmd.orderData);
  }
}

class PlaceOrderService {
  async execute(cmd: PlaceOrderCommand) {
    await this.validateOrderService.execute(cmd);
  }
}

class ValidateOrderService {
  async execute(cmd: PlaceOrderCommand) {
    await this.checkInventoryService.execute(cmd.ebookId);
  }
}

// ✅ Good: Flat service composition or use domain services
class PlaceOrderService {
  async execute(cmd: PlaceOrderCommand): Promise<Result<string, Error>> {
    // Validate inline or use domain service
    const validationResult = this.orderValidator.validate(cmd);
    
    // Check inventory via repository
    const ebook = await this.ebookRepo.findById(cmd.ebookId);
    
    // Create order
    const order = Order.create(cmd, ebook);
    
    // Save
    return await this.orderRepo.save(order);
  }
}
```

### ❌ Pitfall 3: Skipping Commands

```typescript
// ❌ Bad: Primitive obsession
function placeOrder(ebookId: string, email: string, qty: number) {
  // Easy to pass parameters in wrong order
}

placeOrder(email, ebookId, qty); // Oops!

// ✅ Good: Use command object
interface PlaceOrderCommand {
  ebookId: string;
  customerEmail: string;
  quantity: number;
}

function placeOrder(command: PlaceOrderCommand) {
  // Can't mix up the order
}
```

### ❌ Pitfall 4: Leaking Infrastructure Details

```typescript
// ❌ Bad: Service exposes infrastructure
class PlaceOrderService {
  async execute(cmd: PlaceOrderCommand): Promise<AxiosResponse> {
    // Returns HTTP response object!
  }
}

// ✅ Good: Service returns domain result
class PlaceOrderService {
  async execute(cmd: PlaceOrderCommand): Promise<Result<string, Error>> {
    // Returns domain-level result
  }
}
```

### ❌ Pitfall 5: Not Defining Transaction Boundaries

```elixir
# ❌ Bad: Multiple database calls without transaction
def execute(command) do
  {:ok, order} = OrderRepository.save(order)
  # If this fails, order is already saved!
  {:ok, _} = InventoryRepository.update(inventory)
  {:ok, _} = PaymentRepository.charge(payment)
end

# ✅ Good: Use transaction
def execute(command) do
  Repo.transaction(fn ->
    with {:ok, order} <- OrderRepository.save(order),
         {:ok, _} <- InventoryRepository.update(inventory),
         {:ok, _} <- PaymentRepository.charge(payment) do
      order.id
    else
      {:error, reason} -> Repo.rollback(reason)
    end
  end)
end
```

## When NOT to Use

Application services add a layer of indirection. Sometimes that's unnecessary:

### Skip Services For:

1. **Simple CRUD operations**: Just reading/writing data without business logic

```typescript
// Don't need a service for this
async function getUser(id: string): Promise<User> {
  return await userRepository.findById(id);
}
```

2. **Pure queries**: Fetching data for display without modification

```typescript
// Don't need a service for this
async function listOrders(customerId: string): Promise<Order[]> {
  return await orderRepository.findByCustomerId(customerId);
}
```

3. **Very small applications**: If your app is 5 endpoints, services might be overkill

4. **Prototypes**: When validating ideas, add structure later

### Use Services When:

1. **Multiple steps**: Use case involves multiple repositories or domain objects
2. **Business workflow**: There's actual orchestration logic
3. **Transaction needed**: Multiple operations must succeed together
4. **Side effects**: Sending emails, publishing events, calling external APIs
5. **Testing isolation**: You want to test business logic without infrastructure
6. **Reusability**: Same workflow called from multiple places

## Conclusion

Application Services are the orchestration layer that coordinates use cases in your application. They provide a clear, testable boundary between your presentation layer and domain layer.

### Key Takeaways

**Application Services:**
- ✅ Orchestrate use cases, don't contain business rules
- ✅ Are stateless and have one responsibility
- ✅ Accept commands, return results
- ✅ Define transaction boundaries
- ✅ Coordinate repositories, domain objects, and infrastructure
- ✅ Are highly testable with mock dependencies

**Remember:**
- Application services coordinate, domain models execute
- One service per use case keeps code focused
- Commands make contracts explicit
- Transactions belong at the application layer
- Keep services thin—push logic to the domain

### What's Next?

Now that you understand application services, you're ready to explore:

- **Chapter 8: Command Pattern** - Deep dive into commands, validation, and command buses
- **Chapter 9: Layered Architecture** - How services fit into the complete architecture
- **Chapter 10: Event-Driven Architecture** - Services publishing and consuming events
- **Chapter 11: Testing Strategies** - Advanced testing techniques for services

### Further Reading

- *Domain-Driven Design* by Eric Evans (Chapter on Application Services)
- *Implementing Domain-Driven Design* by Vaughn Vernon (Application Services section)
- *Clean Architecture* by Robert C. Martin (Use Cases chapter)

---

*Remember: Application services are about orchestration, not implementation. Keep them thin, focused, and testable. Your domain models do the real work; services just coordinate the dance.* 🎭
