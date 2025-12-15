# Chapter 8: Command Pattern

## Introduction

The Command Pattern is a behavioral design pattern that encapsulates requests as objects. Instead of passing raw primitives (strings, numbers, booleans) to functions, you create structured data objects that represent a user's intention.

In Chapter 7, we introduced Application Services that accept commands, but we didn't dive deep into *why* commands are so valuable or how to implement them effectively. This chapter fills that gap.

### What is the Command Pattern?

At its core, the Command Pattern transforms this:

```typescript
placeOrder(ebookId, email, quantity, price, discount, shippingAddress)
```

Into this:

```typescript
placeOrder(new PlaceOrderCommand(ebookId, email, quantity))
```

The command object represents a **user intention**—what they want to do. Common examples include:
- `PlaceOrderCommand` - User wants to place an order
- `CancelOrderCommand` - User wants to cancel an order
- `UpdateProfileCommand` - User wants to update their profile
- `SendMessageCommand` - User wants to send a message

### Why Commands Matter

**Benefits:**
- ✅ **Self-documenting**: Command names reveal intent (`PlaceOrder` is clearer than a function with 6 parameters)
- ✅ **Type-safe**: Compilers catch mistakes like wrong parameter order
- ✅ **Testable**: Easy to create and verify command objects
- ✅ **Serializable**: Can be logged, queued, or sent over the wire
- ✅ **Validatable**: Validate the entire command before execution
- ✅ **Traceable**: Easy to audit what users intended to do

### Commands Separate "What" from "How"

Commands focus on **what** the user wants, not **how** it gets done:

```typescript
// WHAT: User wants to place an order
const command = new PlaceOrderCommand('ebook-123', 'user@example.com', 2);

// HOW: Service figures out the execution details
await placeOrderService.execute(command);
```

This separation gives you flexibility: you can change *how* orders are placed without changing *what* placing an order means.

## The Problem: Primitive Obsession

Let's see what happens when you pass primitives directly to functions.

### TypeScript: Function with Many Parameters

```typescript
// OrderService.ts - Hard to understand and error-prone
class OrderService {
  async placeOrder(
    ebookId: string,
    customerEmail: string,
    quantity: number,
    pricePerUnit: number,
    discount?: number,
    shippingAddress?: string,
    giftMessage?: string,
    rushDelivery?: boolean
  ): Promise<string> {
    // Implementation...
  }
}

// Usage - which parameter is which?
await orderService.placeOrder(
  'ebook-123',
  'user@example.com',
  2,
  19.99,
  0.1,
  '123 Main St',
  'Happy Birthday!',
  true
);

// Easy to make mistakes:
await orderService.placeOrder(
  'user@example.com',  // Oops! Swapped email and ebookId
  'ebook-123',
  2,
  19.99
);
```

**Problems:**
- ❌ Parameter order must be memorized
- ❌ Easy to swap parameters of same type
- ❌ Adding new parameters breaks all call sites
- ❌ Optional parameters create many combinations
- ❌ No validation until runtime
- ❌ Hard to understand what each value means

### Elixir: Functions with Maps

```elixir
# order_service.ex - Unclear what keys are required
defmodule OrderService do
  def place_order(params) do
    # What keys does params need? Who knows!
    ebook_id = Map.get(params, :ebook_id)
    email = Map.get(params, :customer_email)
    quantity = Map.get(params, :quantity)
    
    # Easy to typo key names
    discount = Map.get(params, :discoutn)  # Typo! Returns nil
    
    # Validation scattered throughout
    if is_nil(ebook_id), do: {:error, :missing_ebook_id}
    if is_nil(email), do: {:error, :missing_email}
    # ...
  end
end

# Usage - completely unclear what's needed
OrderService.place_order(%{
  ebook_id: "123",
  customer_email: "test@example.com",
  quantity: 2,
  discoutn: 0.1  # Typo that silently fails!
})
```

**Problems:**
- ❌ No compile-time checking of required keys
- ❌ Typos in key names fail silently
- ❌ No clear contract of what's required vs optional
- ❌ Validation logic scattered
- ❌ Hard to discover what parameters are needed

## The Solution: Command Objects

Commands solve these problems by creating explicit, type-safe data structures that represent user intentions.

### React/TypeScript: Command Interface

```typescript
// commands/PlaceOrderCommand.ts
export interface PlaceOrderCommand {
  readonly ebookId: string;
  readonly customerEmail: string;
  readonly quantity: number;
}

// Usage - crystal clear!
const command: PlaceOrderCommand = {
  ebookId: 'ebook-123',
  customerEmail: 'user@example.com',
  quantity: 2
};

await placeOrderService.execute(command);
```

**Benefits:**
- ✅ TypeScript ensures all required fields are present
- ✅ Can't swap fields—they're named
- ✅ Auto-completion in IDE
- ✅ Clear contract: "This is everything you need to place an order"
- ✅ Easy to add optional fields without breaking existing code

### TypeScript: Command Class (with Validation)

For commands that need validation, use a class:

```typescript
// commands/PlaceOrderCommand.ts
export class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // Structural validation in constructor
    if (!ebookId || ebookId.trim() === '') {
      throw new Error('ebookId is required');
    }
    
    if (!customerEmail || !customerEmail.includes('@')) {
      throw new Error('Invalid email address');
    }
    
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
  }
}

// Usage - validation happens immediately
try {
  const command = new PlaceOrderCommand(
    'ebook-123',
    'user@example.com',
    2
  );
  
  await placeOrderService.execute(command);
} catch (error) {
  console.error('Invalid command:', error.message);
}

// This throws immediately - invalid email
const badCommand = new PlaceOrderCommand('ebook-123', 'invalid-email', 2);
```

### Elixir: Command Struct

```elixir
# lib/shop/commands/place_order_command.ex
defmodule Shop.Commands.PlaceOrderCommand do
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

  @doc """
  Creates a new PlaceOrderCommand with validation.
  
  ## Examples
  
      iex> PlaceOrderCommand.new(%{
      ...>   ebook_id: "ebook-123",
      ...>   customer_email: "user@example.com",
      ...>   quantity: 2
      ...> })
      {:ok, %PlaceOrderCommand{...}}
      
      iex> PlaceOrderCommand.new(%{quantity: 0})
      {:error, :invalid_command}
  """
  @spec new(map()) :: {:ok, t()} | {:error, :invalid_command}
  def new(%{ebook_id: ebook_id, customer_email: email, quantity: qty} = _params)
      when is_binary(ebook_id) and is_binary(email) and is_integer(qty) and qty > 0 do
    if String.contains?(email, "@") and byte_size(ebook_id) > 0 do
      {:ok, %__MODULE__{
        ebook_id: ebook_id,
        customer_email: email,
        quantity: qty
      }}
    else
      {:error, :invalid_command}
    end
  end

  def new(_), do: {:error, :invalid_command}
end
```

**Benefits:**
- ✅ `@enforce_keys` ensures required fields at compile time
- ✅ Pattern matching validates structure
- ✅ Typespecs document expectations
- ✅ `new/1` factory provides validation
- ✅ Can't typo field names—compile error!

### Elixir: Using Ecto Changeset for Validation (Alternative)

For more complex validation, use Ecto changesets:

```elixir
defmodule Shop.Commands.PlaceOrderCommand do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false
  embedded_schema do
    field :ebook_id, :string
    field :customer_email, :string
    field :quantity, :integer
  end

  @doc """
  Creates and validates a PlaceOrderCommand from parameters.
  """
  def new(params) do
    %__MODULE__{}
    |> cast(params, [:ebook_id, :customer_email, :quantity])
    |> validate_required([:ebook_id, :customer_email, :quantity])
    |> validate_format(:customer_email, ~r/@/)
    |> validate_number(:quantity, greater_than: 0)
    |> case do
      %{valid?: true} = changeset ->
        {:ok, apply_changes(changeset)}
      
      %{valid?: false} = changeset ->
        {:error, changeset}
    end
  end
end

# Usage
case PlaceOrderCommand.new(params) do
  {:ok, command} ->
    OrderService.execute(command)
    
  {:error, changeset} ->
    errors = format_errors(changeset)
    {:error, errors}
end
```

## Command Validation: Where and What

A common question: **Where should validation happen?**

The answer: **It depends on the type of validation.**

### Two Types of Validation

**1. Structural Validation** - Validate in Command Constructor/Factory
- Required fields present
- Correct data types
- Basic format checks (email has @, IDs not empty)
- Range checks (quantity > 0)

**2. Business Rule Validation** - Validate in Domain or Application Service
- Does the ebook exist?
- Is there enough inventory?
- Does the customer have permission?
- Are there duplicate orders?

### TypeScript: Validation Layers

```typescript
// commands/PlaceOrderCommand.ts
export class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // ✅ Structural validation: Do the inputs make sense?
    if (!ebookId || ebookId.trim() === '') {
      throw new Error('ebookId is required');
    }
    
    if (!customerEmail.includes('@')) {
      throw new Error('Email must contain @');
    }
    
    if (quantity <= 0 || !Number.isInteger(quantity)) {
      throw new Error('Quantity must be a positive integer');
    }
  }
}

// services/PlaceOrderService.ts
export class PlaceOrderService {
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // ✅ Business validation: Does this operation make business sense?
    
    // Check if ebook exists
    const ebookResult = await this.ebookRepo.findById(command.ebookId);
    if (ebookResult.isErr()) {
      return err(new Error('Ebook not found'));
    }
    const ebook = ebookResult.value;
    
    // Check inventory
    if (ebook.stock < command.quantity) {
      return err(new Error(`Only ${ebook.stock} copies available`));
    }
    
    // Check for duplicate orders (business rule)
    const duplicateCheck = await this.orderRepo.findRecentOrder(
      command.customerEmail,
      command.ebookId
    );
    if (duplicateCheck.isOk()) {
      return err(new Error('Duplicate order detected'));
    }
    
    // All validations passed - proceed with business logic
    const order = Order.create({
      ebookId: command.ebookId,
      customerEmail: command.customerEmail,
      quantity: command.quantity,
      pricePerUnit: ebook.price
    });
    
    return await this.orderRepo.save(order.value);
  }
}
```

### Elixir: Validation Layers

```elixir
# lib/shop/commands/place_order_command.ex
defmodule Shop.Commands.PlaceOrderCommand do
  @enforce_keys [:ebook_id, :customer_email, :quantity]
  defstruct [:ebook_id, :customer_email, :quantity]

  # ✅ Structural validation in factory
  def new(params) do
    with {:ok, ebook_id} <- validate_ebook_id(params[:ebook_id]),
         {:ok, email} <- validate_email(params[:customer_email]),
         {:ok, quantity} <- validate_quantity(params[:quantity]) do
      {:ok, %__MODULE__{
        ebook_id: ebook_id,
        customer_email: email,
        quantity: quantity
      }}
    end
  end

  defp validate_ebook_id(nil), do: {:error, :ebook_id_required}
  defp validate_ebook_id(""), do: {:error, :ebook_id_required}
  defp validate_ebook_id(id) when is_binary(id), do: {:ok, id}
  defp validate_ebook_id(_), do: {:error, :ebook_id_invalid}

  defp validate_email(nil), do: {:error, :email_required}
  defp validate_email(email) when is_binary(email) do
    if String.contains?(email, "@") do
      {:ok, email}
    else
      {:error, :email_invalid}
    end
  end
  defp validate_email(_), do: {:error, :email_invalid}

  defp validate_quantity(qty) when is_integer(qty) and qty > 0 do
    {:ok, qty}
  end
  defp validate_quantity(_), do: {:error, :quantity_invalid}
end

# lib/shop/services/place_order_service.ex
defmodule Shop.Services.PlaceOrderService do
  alias Shop.Commands.PlaceOrderCommand
  alias Shop.Repositories.{EbookRepository, OrderRepository}
  alias Shop.Domain.Order

  # ✅ Business validation in service
  def execute(%PlaceOrderCommand{} = command) do
    with {:ok, ebook} <- fetch_ebook(command.ebook_id),
         :ok <- check_inventory(ebook, command.quantity),
         :ok <- check_duplicate_order(command),
         {:ok, order} <- create_order(command, ebook),
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

  defp check_inventory(ebook, requested_quantity) do
    if ebook.stock >= requested_quantity do
      :ok
    else
      {:error, {:insufficient_inventory, ebook.stock}}
    end
  end

  defp check_duplicate_order(command) do
    case OrderRepository.find_recent_order(
      command.customer_email,
      command.ebook_id
    ) do
      {:ok, _order} -> {:error, :duplicate_order}
      {:error, :not_found} -> :ok
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
    OrderRepository.save(order)
  end
end
```

### Best Practices for Validation

**DO:**
- ✅ Validate structure in command constructor/factory
- ✅ Validate business rules in domain/service layer
- ✅ Fail fast with structural validation
- ✅ Return descriptive errors
- ✅ Use types to prevent invalid states

**DON'T:**
- ❌ Put database queries in command validation
- ❌ Mix structural and business validation
- ❌ Validate the same thing in multiple places
- ❌ Swallow validation errors silently



## Command Handlers

Command handlers are functions or classes that execute commands. They contain the workflow logic that implements the user's intention.

### TypeScript: Command Handler Interface

```typescript
// handlers/CommandHandler.ts
export interface CommandHandler<TCommand, TResult> {
  execute(command: TCommand): Promise<TResult>;
}

// handlers/PlaceOrderHandler.ts
import { Result } from 'neverthrow';
import { PlaceOrderCommand } from '../commands/PlaceOrderCommand';
import { CommandHandler } from './CommandHandler';
import { OrderRepository } from '../domain/OrderRepository';
import { EbookRepository } from '../domain/EbookRepository';
import { Order } from '../domain/Order';

export class PlaceOrderHandler implements CommandHandler<PlaceOrderCommand, Result<string, Error>> {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly ebookRepo: EbookRepository
  ) {}

  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // 1. Fetch ebook to get price
    const ebookResult = await this.ebookRepo.findById(command.ebookId);
    if (ebookResult.isErr()) {
      return err(new Error('Ebook not found'));
    }
    
    // 2. Create domain model
    const orderResult = Order.create({
      ebookId: command.ebookId,
      customerEmail: command.customerEmail,
      quantity: command.quantity,
      pricePerUnit: ebookResult.value.price
    });
    
    if (orderResult.isErr()) {
      return orderResult;
    }
    
    // 3. Persist
    const savedResult = await this.orderRepo.save(orderResult.value);
    
    return savedResult;
  }
}
```

### Elixir: Command Handler Module

```elixir
# lib/shop/handlers/place_order_handler.ex
defmodule Shop.Handlers.PlaceOrderHandler do
  @moduledoc """
  Handles the PlaceOrderCommand by orchestrating the order placement workflow.
  """

  alias Shop.Commands.PlaceOrderCommand
  alias Shop.Repositories.{OrderRepository, EbookRepository}
  alias Shop.Domain.Order

  @type success :: {:ok, String.t()}
  @type failure :: {:error, atom() | {atom(), any()}}

  @doc """
  Executes the place order command.
  
  Returns {:ok, order_id} on success or {:error, reason} on failure.
  """
  @spec handle(PlaceOrderCommand.t()) :: success() | failure()
  def handle(%PlaceOrderCommand{} = command) do
    with {:ok, ebook} <- fetch_ebook(command.ebook_id),
         {:ok, order} <- create_order(command, ebook),
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

  defp create_order(command, ebook) do
    Order.new(
      command.ebook_id,
      command.customer_email,
      command.quantity,
      ebook.price
    )
  end

  defp save_order(order) do
    OrderRepository.save(order)
  end
end
```

### Handler Design Principles

**Handlers should:**
- ✅ Be stateless (no instance variables that mutate)
- ✅ Accept one command type
- ✅ Have a single public method/function (`execute` or `handle`)
- ✅ Orchestrate, not implement business logic
- ✅ Return consistent result types
- ✅ Handle errors gracefully

**Handlers should NOT:**
- ❌ Contain business rules (that's domain layer)
- ❌ Access infrastructure directly (use repositories)
- ❌ Make UI decisions (return data, not views)
- ❌ Handle multiple command types


## Integration with React Components

Commands make React components cleaner by separating UI concerns from business logic.

### Before: Logic Scattered in Component

```typescript
// BeforeOrderForm.tsx - Everything mixed together
import React, { useState } from 'react';
import axios from 'axios';

interface Props {
  ebookId: string;
}

export function BeforeOrderForm({ ebookId }: Props) {
  const [email, setEmail] = useState('');
  const [quantity, setQuantity] = useState(1);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    // Validation in component
    if (!email.includes('@')) {
      setError('Invalid email');
      setLoading(false);
      return;
    }

    if (quantity <= 0) {
      setError('Invalid quantity');
      setLoading(false);
      return;
    }

    // Business logic in component
    try {
      const ebookResponse = await axios.get(`/api/ebooks/${ebookId}`);
      const ebook = ebookResponse.data;
      const total = ebook.price * quantity;

      const orderResponse = await axios.post('/api/orders', {
        ebook_id: ebookId,
        customer_email: email,
        quantity,
        total
      });

      alert(`Order placed! ID: ${orderResponse.data.order_id}`);
    } catch (err) {
      setError('Failed to place order');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* Form JSX */}
    </form>
  );
}
```

### After: Commands and Services

```typescript
// AfterOrderForm.tsx - Clean separation
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { PlaceOrderCommand } from '../commands/PlaceOrderCommand';
import { usePlaceOrderHandler } from '../hooks/useHandlers';

interface Props {
  ebookId: string;
}

export function AfterOrderForm({ ebookId }: Props) {
  const navigate = useNavigate();
  const placeOrderHandler = usePlaceOrderHandler();
  
  const [email, setEmail] = useState('');
  const [quantity, setQuantity] = useState(1);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      // Create command - validation happens here
      const command = new PlaceOrderCommand(ebookId, email, quantity);
      
      // Execute via handler - business logic happens here
      const result = await placeOrderHandler.execute(command);
      
      if (result.isOk()) {
        navigate(`/orders/${result.value}`);
      } else {
        setError(result.error.message);
      }
    } catch (err) {
      // Command validation error
      setError(err instanceof Error ? err.message : 'Invalid input');
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

### Dependency Injection Hook

```typescript
// hooks/useHandlers.ts
import { useMemo } from 'react';
import { PlaceOrderHandler } from '../handlers/PlaceOrderHandler';
import { HttpOrderRepository } from '../infrastructure/HttpOrderRepository';
import { HttpEbookRepository } from '../infrastructure/HttpEbookRepository';

export function usePlaceOrderHandler(): PlaceOrderHandler {
  return useMemo(() => {
    const orderRepo = new HttpOrderRepository();
    const ebookRepo = new HttpEbookRepository();

    return new PlaceOrderHandler(orderRepo, ebookRepo);
  }, []);
}
```

**Benefits of this approach:**
- ✅ Component focuses on UI rendering and user interaction
- ✅ Command encapsulates all input data with validation
- ✅ Handler contains reusable business logic
- ✅ Easy to test each piece independently
- ✅ Clear separation of concerns


## Integration with Phoenix Controllers

Phoenix controllers become thin routing layers when using commands.

### Before: Logic in Controller

```elixir
# lib/shop_web/controllers/order_controller.ex - Before
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller

  def create(conn, params) do
    # Validation in controller
    with {:ok, ebook_id} <- validate_ebook_id(params["ebook_id"]),
         {:ok, email} <- validate_email(params["customer_email"]),
         {:ok, quantity} <- validate_quantity(params["quantity"]),
         # Business logic in controller
         {:ok, ebook} <- fetch_ebook(ebook_id),
         {:ok, order} <- create_order(ebook_id, email, quantity, ebook.price),
         {:ok, saved_order} <- save_order(order) do
      conn
      |> put_status(:created)
      |> json(%{order_id: saved_order.id})
    else
      {:error, :invalid_ebook_id} ->
        conn |> put_status(:bad_request) |> json(%{error: "Invalid ebook ID"})
      
      {:error, :invalid_email} ->
        conn |> put_status(:bad_request) |> json(%{error: "Invalid email"})
      
      # Many error cases...
    end
  end

  defp validate_ebook_id(nil), do: {:error, :invalid_ebook_id}
  defp validate_ebook_id(id), do: {:ok, id}
  # More validation functions...
end
```

### After: Commands and Handlers

```elixir
# lib/shop_web/controllers/order_controller.ex - After
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller

  alias Shop.Commands.PlaceOrderCommand
  alias Shop.Handlers.PlaceOrderHandler

  @doc """
  POST /api/orders
  
  Creates a new order using the PlaceOrderCommand and PlaceOrderHandler.
  """
  def create(conn, params) do
    with {:ok, command} <- PlaceOrderCommand.new(params),
         {:ok, order_id} <- PlaceOrderHandler.handle(command) do
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

      {:error, {:insufficient_inventory, available}} ->
        conn
        |> put_status(:conflict)
        |> json(%{error: "Insufficient inventory", available: available})

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
    # Query logic (no command needed for simple queries)
    case OrderRepository.find_by_id(id) do
      {:ok, order} ->
        json(conn, order)
      
      {:error, :not_found} ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Order not found"})
    end
  end
end
```

**Benefits:**
- ✅ Controller is just HTTP routing
- ✅ Command handles validation
- ✅ Handler contains business logic
- ✅ Easy to test each layer
- ✅ Can reuse handler from other contexts (CLI, tests, jobs)

### Real-World Example: Multiple Commands

```elixir
# lib/shop_web/controllers/order_controller.ex
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller

  alias Shop.Commands.{PlaceOrderCommand, CancelOrderCommand, RefundOrderCommand}
  alias Shop.Handlers.{PlaceOrderHandler, CancelOrderHandler, RefundOrderHandler}

  def create(conn, params) do
    with {:ok, command} <- PlaceOrderCommand.new(params),
         {:ok, order_id} <- PlaceOrderHandler.handle(command) do
      json_response(conn, :created, %{order_id: order_id})
    else
      error -> handle_error(conn, error)
    end
  end

  def cancel(conn, %{"id" => order_id}) do
    with {:ok, command} <- CancelOrderCommand.new(%{order_id: order_id}),
         {:ok, _} <- CancelOrderHandler.handle(command) do
      json_response(conn, :ok, %{message: "Order cancelled"})
    else
      error -> handle_error(conn, error)
    end
  end

  def refund(conn, %{"id" => order_id} = params) do
    with {:ok, command} <- RefundOrderCommand.new(params),
         {:ok, refund_id} <- RefundOrderHandler.handle(command) do
      json_response(conn, :ok, %{refund_id: refund_id})
    else
      error -> handle_error(conn, error)
    end
  end

  # Centralized error handling
  defp handle_error(conn, {:error, :invalid_command}) do
    json_response(conn, :bad_request, %{error: "Invalid parameters"})
  end

  defp handle_error(conn, {:error, :not_found}) do
    json_response(conn, :not_found, %{error: "Resource not found"})
  end

  defp handle_error(conn, {:error, reason}) do
    json_response(conn, :unprocessable_entity, %{error: to_string(reason)})
  end

  defp json_response(conn, status, data) do
    conn
    |> put_status(status)
    |> json(data)
  end
end
```

## Before/After Comparison

Let's see the full transformation side-by-side.

### Before: Primitive Parameters

**TypeScript:**
```typescript
// Hard to understand, easy to mess up
function placeOrder(
  ebookId: string,
  email: string,
  quantity: number,
  price: number,
  discount?: number,
  shippingAddress?: string
): Promise<string> {
  // ...
}

// Usage - what do these values mean?
await placeOrder(
  'ebook-123',
  'user@example.com',
  2,
  19.99,
  0.1,
  '123 Main St'
);
```

**Elixir:**
```elixir
# Unclear what keys are required
def place_order(params) do
  ebook_id = Map.get(params, :ebook_id)
  email = Map.get(params, :customer_email)
  quantity = Map.get(params, :quantity)
  # ...
end

# Usage - typos fail silently
place_order(%{
  ebook_id: "123",
  costumer_email: "user@example.com",  # Typo!
  quantity: 2
})
```

### After: Command Objects

**TypeScript:**
```typescript
// Clear, self-documenting
interface PlaceOrderCommand {
  ebookId: string;
  customerEmail: string;
  quantity: number;
}

function placeOrder(command: PlaceOrderCommand): Promise<string> {
  // ...
}

// Usage - crystal clear
await placeOrder({
  ebookId: 'ebook-123',
  customerEmail: 'user@example.com',
  quantity: 2
});
```

**Elixir:**
```elixir
# Explicit contract
defmodule PlaceOrderCommand do
  @enforce_keys [:ebook_id, :customer_email, :quantity]
  defstruct [:ebook_id, :customer_email, :quantity]
  
  def new(params) do
    # Validation...
  end
end

# Usage - compile-time safety
{:ok, command} = PlaceOrderCommand.new(%{
  ebook_id: "123",
  customer_email: "user@example.com",
  quantity: 2
})
place_order(command)
```

### Comparison Table

| Aspect | Before (Primitives) | After (Commands) |
|--------|-------------------|------------------|
| **Clarity** | Function signature with 6+ params | Single command object |
| **Type Safety** | Can swap params of same type | Named fields prevent mistakes |
| **Validation** | Scattered throughout code | Centralized in command |
| **Testing** | Hard to create test data | Easy to construct commands |
| **Evolution** | Adding params breaks call sites | Add optional fields safely |
| **Documentation** | Parameter comments needed | Self-documenting structure |
| **Serialization** | Manual parameter packing | Natural object serialization |
| **Error Messages** | "Parameter 3 is invalid" | "quantity must be positive" |


## Testing Benefits

Commands make tests dramatically clearer and easier to write.

### TypeScript: Testing with Commands

```typescript
// __tests__/PlaceOrderHandler.test.ts
import { PlaceOrderCommand } from '../commands/PlaceOrderCommand';
import { PlaceOrderHandler } from '../handlers/PlaceOrderHandler';
import { InMemoryOrderRepository } from './helpers/InMemoryOrderRepository';
import { InMemoryEbookRepository } from './helpers/InMemoryEbookRepository';

describe('PlaceOrderHandler', () => {
  let handler: PlaceOrderHandler;
  let orderRepo: InMemoryOrderRepository;
  let ebookRepo: InMemoryEbookRepository;

  beforeEach(() => {
    orderRepo = new InMemoryOrderRepository();
    ebookRepo = new InMemoryEbookRepository();
    handler = new PlaceOrderHandler(orderRepo, ebookRepo);

    // Seed test data
    ebookRepo.save({
      id: 'ebook-1',
      title: 'Clean Code',
      price: 29.99,
      stock: 100
    });
  });

  it('successfully places an order', async () => {
    // Arrange - crystal clear what we're testing
    const command = new PlaceOrderCommand(
      'ebook-1',
      'test@example.com',
      2
    );

    // Act
    const result = await handler.execute(command);

    // Assert
    expect(result.isOk()).toBe(true);
    
    const orderId = result.value;
    const savedOrder = await orderRepo.findById(orderId);
    
    expect(savedOrder.value).toMatchObject({
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 2
    });
  });

  it('fails when ebook not found', async () => {
    const command = new PlaceOrderCommand(
      'nonexistent',
      'test@example.com',
      1
    );

    const result = await handler.execute(command);

    expect(result.isErr()).toBe(true);
    expect(result.error.message).toContain('not found');
  });

  it('validates quantity in command constructor', () => {
    expect(() => {
      new PlaceOrderCommand('ebook-1', 'test@example.com', 0);
    }).toThrow('Quantity must be positive');
  });

  it('validates email in command constructor', () => {
    expect(() => {
      new PlaceOrderCommand('ebook-1', 'invalid-email', 1);
    }).toThrow('Invalid email');
  });
});
```

### Elixir: Testing with Commands

```elixir
# test/shop/handlers/place_order_handler_test.exs
defmodule Shop.Handlers.PlaceOrderHandlerTest do
  use Shop.DataCase, async: true

  alias Shop.Commands.PlaceOrderCommand
  alias Shop.Handlers.PlaceOrderHandler
  alias Shop.Repositories.{EbookRepository, OrderRepository}

  describe "handle/1" do
    setup do
      # Seed test data
      {:ok, ebook} = EbookRepository.save(%{
        id: "ebook-1",
        title: "Clean Code",
        price: 2999,
        stock: 100
      })

      %{ebook: ebook}
    end

    test "successfully places an order", %{ebook: ebook} do
      # Arrange - clear what we're testing
      {:ok, command} = PlaceOrderCommand.new(%{
        ebook_id: ebook.id,
        customer_email: "test@example.com",
        quantity: 2
      })

      # Act
      assert {:ok, order_id} = PlaceOrderHandler.handle(command)

      # Assert
      assert is_binary(order_id)
      {:ok, order} = OrderRepository.find_by_id(order_id)
      assert order.ebook_id == ebook.id
      assert order.customer_email == "test@example.com"
      assert order.quantity == 2
    end

    test "fails when ebook not found" do
      {:ok, command} = PlaceOrderCommand.new(%{
        ebook_id: "nonexistent",
        customer_email: "test@example.com",
        quantity: 1
      })

      assert {:error, :ebook_not_found} = PlaceOrderHandler.handle(command)
    end

    test "validates quantity in command" do
      assert {:error, :invalid_command} = PlaceOrderCommand.new(%{
        ebook_id: "ebook-1",
        customer_email: "test@example.com",
        quantity: 0
      })
    end

    test "validates email format in command" do
      assert {:error, :invalid_command} = PlaceOrderCommand.new(%{
        ebook_id: "ebook-1",
        customer_email: "invalid",
        quantity: 1
      })
    end
  end
end
```

### Why Commands Improve Tests

**Benefits:**
- ✅ **Clear Intent**: `new PlaceOrderCommand(...)` tells you exactly what you're testing
- ✅ **Easy Setup**: Creating test commands is trivial
- ✅ **Type Safety**: Compiler catches test data mistakes
- ✅ **Reusable**: Share command fixtures across tests
- ✅ **Fast**: No need for complex mocks—commands are plain data
- ✅ **Focused**: Test command validation separately from handler logic

## Command Bus Pattern (Advanced)

For large applications, a Command Bus can route commands to their handlers automatically.

### TypeScript: Simple Command Bus

```typescript
// infrastructure/CommandBus.ts
type Constructor<T> = new (...args: any[]) => T;

export interface CommandHandler<TCommand, TResult> {
  execute(command: TCommand): Promise<TResult>;
}

export class CommandBus {
  private handlers = new Map<Constructor<any>, CommandHandler<any, any>>();

  register<TCommand, TResult>(
    commandType: Constructor<TCommand>,
    handler: CommandHandler<TCommand, TResult>
  ): void {
    this.handlers.set(commandType, handler);
  }

  async execute<TCommand, TResult>(command: TCommand): Promise<TResult> {
    const handler = this.handlers.get(command.constructor as Constructor<TCommand>);
    
    if (!handler) {
      throw new Error(`No handler registered for ${command.constructor.name}`);
    }
    
    return handler.execute(command);
  }
}

// Setup
const bus = new CommandBus();
bus.register(PlaceOrderCommand, new PlaceOrderHandler(orderRepo, ebookRepo));
bus.register(CancelOrderCommand, new CancelOrderHandler(orderRepo));

// Usage
const command = new PlaceOrderCommand('ebook-1', 'user@example.com', 2);
const result = await bus.execute(command);
```

### TypeScript: Command Bus with Middleware

```typescript
// infrastructure/CommandBus.ts
export type CommandMiddleware = <TCommand>(
  command: TCommand,
  next: (command: TCommand) => Promise<any>
) => Promise<any>;

export class CommandBus {
  private handlers = new Map<Constructor<any>, CommandHandler<any, any>>();
  private middlewares: CommandMiddleware[] = [];

  use(middleware: CommandMiddleware): void {
    this.middlewares.push(middleware);
  }

  register<TCommand, TResult>(
    commandType: Constructor<TCommand>,
    handler: CommandHandler<TCommand, TResult>
  ): void {
    this.handlers.set(commandType, handler);
  }

  async execute<TCommand, TResult>(command: TCommand): Promise<TResult> {
    const handler = this.handlers.get(command.constructor as Constructor<TCommand>);
    
    if (!handler) {
      throw new Error(`No handler registered for ${command.constructor.name}`);
    }

    // Build middleware chain
    let index = 0;
    const executeMiddleware = async (cmd: TCommand): Promise<TResult> => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        return middleware(cmd, executeMiddleware);
      }
      return handler.execute(cmd);
    };

    return executeMiddleware(command);
  }
}

// Middleware examples
const loggingMiddleware: CommandMiddleware = async (command, next) => {
  console.log(`Executing: ${command.constructor.name}`, command);
  const startTime = Date.now();
  
  try {
    const result = await next(command);
    console.log(`Success: ${command.constructor.name} (${Date.now() - startTime}ms)`);
    return result;
  } catch (error) {
    console.error(`Failed: ${command.constructor.name}`, error);
    throw error;
  }
};

const validationMiddleware: CommandMiddleware = async (command, next) => {
  // Validate command before execution
  if (typeof command.validate === 'function') {
    command.validate();
  }
  return next(command);
};

// Setup
const bus = new CommandBus();
bus.use(loggingMiddleware);
bus.use(validationMiddleware);
bus.register(PlaceOrderCommand, placeOrderHandler);
```

### Elixir: Command Router (Functional Approach)

```elixir
# lib/shop/command_router.ex
defmodule Shop.CommandRouter do
  @moduledoc """
  Routes commands to their appropriate handlers.
  """

  alias Shop.Commands.{PlaceOrderCommand, CancelOrderCommand}
  alias Shop.Handlers.{PlaceOrderHandler, CancelOrderHandler}

  @doc """
  Routes a command to its handler.
  """
  def route(%PlaceOrderCommand{} = command) do
    PlaceOrderHandler.handle(command)
  end

  def route(%CancelOrderCommand{} = command) do
    CancelOrderHandler.handle(command)
  end

  def route(command) do
    {:error, {:unknown_command, command.__struct__}}
  end
end

# Usage
command = %PlaceOrderCommand{...}
CommandRouter.route(command)
```

### When to Use a Command Bus

**Use when:**
- ✅ You have many commands (10+ different use cases)
- ✅ You need cross-cutting concerns (logging, validation, authorization)
- ✅ You want to decouple command creation from execution
- ✅ You need command auditing or replay functionality

**Skip when:**
- ❌ You have < 5 commands
- ❌ Direct handler calls are clearer
- ❌ Your team finds it over-engineered
- ❌ You don't need middleware features


## Common Pitfalls

### ❌ Pitfall 1: Anemic Commands (Just Data Bags)

```typescript
// ❌ Bad: No validation
interface PlaceOrderCommand {
  ebookId: string;
  customerEmail: string;
  quantity: number;
}

// Anyone can create invalid commands
const command = {
  ebookId: '',
  customerEmail: 'not-an-email',
  quantity: -5
} as PlaceOrderCommand;
```

```typescript
// ✅ Good: Validate structure
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    if (!ebookId) throw new Error('ebookId required');
    if (!customerEmail.includes('@')) throw new Error('Invalid email');
    if (quantity <= 0) throw new Error('Quantity must be positive');
  }
}
```

### ❌ Pitfall 2: Business Logic in Commands

```typescript
// ❌ Bad: Business logic in command
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // Business rule - belongs in domain!
    this.total = this.quantity * this.pricePerUnit;
    
    // Another business rule
    if (this.total > 1000) {
      this.discount = this.total * 0.1;
    }
  }
}
```

```typescript
// ✅ Good: Commands are just data
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // Only structural validation, no business logic
    if (!ebookId) throw new Error('ebookId required');
  }
}

// Business logic in domain
class Order {
  calculateTotal(): Money {
    const subtotal = this.quantity * this.pricePerUnit;
    if (subtotal.isGreaterThan(Money.fromAmount(1000))) {
      return subtotal.applyDiscount(0.1);
    }
    return subtotal;
  }
}
```

### ❌ Pitfall 3: Commands with Too Many Fields

```typescript
// ❌ Bad: God object command
interface PlaceOrderCommand {
  ebookId: string;
  customerEmail: string;
  quantity: number;
  shippingAddress: string;
  billingAddress: string;
  paymentMethod: string;
  cardNumber: string;
  cvv: string;
  expirationDate: string;
  giftMessage?: string;
  giftWrap?: boolean;
  rushDelivery?: boolean;
  newsletter?: boolean;
  referralCode?: string;
  // 20+ fields!
}
```

```typescript
// ✅ Good: Break into focused commands
interface PlaceOrderCommand {
  ebookId: string;
  customerEmail: string;
  quantity: number;
}

interface ProcessPaymentCommand {
  orderId: string;
  paymentMethod: PaymentMethod;
}

interface AddGiftOptionsCommand {
  orderId: string;
  giftMessage: string;
  giftWrap: boolean;
}
```

### ❌ Pitfall 4: Mutable Commands

```typescript
// ❌ Bad: Mutable command
class PlaceOrderCommand {
  public ebookId: string;
  public customerEmail: string;
  public quantity: number;

  setQuantity(qty: number) {
    this.quantity = qty; // Can change after creation!
  }
}

const command = new PlaceOrderCommand();
command.ebookId = 'ebook-1';
// Later...
command.quantity = -5; // Oops!
```

```typescript
// ✅ Good: Immutable command
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // Validation in constructor
  }
  
  // No setters - immutable!
}
```

### ❌ Pitfall 5: Validating Business Rules in Constructor

```typescript
// ❌ Bad: Database query in command validation
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // Structural validation - OK
    if (!ebookId) throw new Error('ebookId required');
    
    // Business validation - NOT OK in constructor!
    const ebook = await ebookRepo.findById(ebookId);
    if (!ebook) throw new Error('Ebook not found');
    
    if (ebook.stock < quantity) {
      throw new Error('Insufficient inventory');
    }
  }
}
```

```typescript
// ✅ Good: Separate structural and business validation
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {
    // ✅ Structural validation only
    if (!ebookId) throw new Error('ebookId required');
    if (quantity <= 0) throw new Error('Quantity must be positive');
  }
}

class PlaceOrderHandler {
  async execute(command: PlaceOrderCommand): Promise<Result<string, Error>> {
    // ✅ Business validation in handler
    const ebook = await this.ebookRepo.findById(command.ebookId);
    if (!ebook) {
      return err(new Error('Ebook not found'));
    }
    
    if (ebook.stock < command.quantity) {
      return err(new Error('Insufficient inventory'));
    }
    
    // Continue...
  }
}
```

### Best Practices Summary

**DO:**
- ✅ Commands are immutable data structures
- ✅ Validate structure in command, business rules in domain/service
- ✅ One command per use case
- ✅ Clear, intention-revealing names (verb + object: PlaceOrder, CancelSubscription)
- ✅ Use readonly fields in TypeScript
- ✅ Use @enforce_keys in Elixir

**DON'T:**
- ❌ Put business logic in commands
- ❌ Make commands mutable
- ❌ Create god object commands with too many fields
- ❌ Validate business rules in command constructors
- ❌ Use generic names (DoStuff, HandleRequest)

## Commands vs Events

Commands and events are often confused. Let's clarify the difference.

### Commands: Intent to Do Something

Commands are **imperative**—they express an intention that *can be rejected*.

**Characteristics:**
- Named in imperative tense (PlaceOrder, CancelSubscription)
- Represent user intent
- Can fail (validation, business rules)
- Sent to a single handler
- Present/future tense ("I want to place an order")

**Examples:**
```typescript
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {}
}

// Handler can reject
const result = await handler.execute(command);
if (result.isErr()) {
  // Command was rejected
}
```

### Events: Something That Happened

Events are **past tense**—they announce that something *already happened* and cannot be rejected.

**Characteristics:**
- Named in past tense (OrderPlaced, SubscriptionCancelled)
- Represent facts
- Cannot fail (already happened)
- Broadcast to multiple listeners
- Past tense ("An order was placed")

**Examples:**
```typescript
class OrderPlacedEvent {
  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly total: Money,
    public readonly timestamp: Date
  ) {}
}

// Multiple listeners react to event
eventBus.publish(new OrderPlacedEvent(...));
```

### Side-by-Side Comparison

| Aspect | Command | Event |
|--------|---------|-------|
| **Tense** | Imperative (PlaceOrder) | Past (OrderPlaced) |
| **Intent** | Request to do something | Announcement of what happened |
| **Can Fail?** | Yes (validation, business rules) | No (already happened) |
| **Recipients** | Single handler | Multiple listeners |
| **Direction** | Tell the system what to do | Tell the system what happened |
| **Example** | `CancelOrder` | `OrderCancelled` |

### Workflow: Commands → Events

A typical workflow uses both:

```typescript
// 1. User sends command (intent)
const command = new PlaceOrderCommand('ebook-1', 'user@example.com', 2);

// 2. Handler processes command
const result = await placeOrderHandler.execute(command);

if (result.isOk()) {
  // 3. Handler publishes event (fact)
  await eventBus.publish(
    new OrderPlacedEvent(
      result.value, // orderId
      'user-123',
      Money.fromAmount(59.98),
      new Date()
    )
  );
}

// 4. Multiple listeners react to event
// - Send confirmation email
// - Update analytics
// - Notify warehouse
// - Update user dashboard
```

### When to Use What

**Use Commands when:**
- ✅ User wants to perform an action
- ✅ The action can be rejected
- ✅ You need validation before execution
- ✅ You have a single handler responsible

**Use Events when:**
- ✅ Something significant happened
- ✅ Multiple parts of system need to react
- ✅ You want loose coupling between components
- ✅ You need audit trail or event sourcing

## When NOT to Use Commands

Commands add structure and clarity, but they're not always necessary.

### Skip Commands For:

**1. Simple Queries**

```typescript
// ❌ Over-engineered: Command for simple query
class GetUserCommand {
  constructor(public readonly userId: string) {}
}
const command = new GetUserCommand('user-123');
const user = await getUserHandler.execute(command);

// ✅ Better: Direct query
const user = await userRepository.findById('user-123');
```

**2. Functions with 1-2 Parameters**

```typescript
// ❌ Over-engineered: Command for simple function
class SendEmailCommand {
  constructor(
    public readonly to: string,
    public readonly subject: string
  ) {}
}

// ✅ Better: Direct function call
await sendEmail('user@example.com', 'Welcome!');
```

**3. Internal Service-to-Service Calls**

```typescript
// ❌ Over-engineered: Commands between services in same process
class CalculateTaxCommand {
  constructor(public readonly amount: Money) {}
}
const taxCommand = new CalculateTaxCommand(total);
const tax = await taxService.execute(taxCommand);

// ✅ Better: Direct method call
const tax = taxService.calculateTax(total);
```

**4. CRUD Operations Without Business Logic**

```typescript
// ❌ Over-engineered: Command for simple CRUD
class UpdateUserEmailCommand {
  constructor(
    public readonly userId: string,
    public readonly email: string
  ) {}
}

// ✅ Better: Direct repository call (if no business logic)
await userRepository.update(userId, { email: newEmail });
```

### Use Commands When:

**1. Complex User Actions**

```typescript
// ✅ Good: Multi-step process with business rules
class PlaceOrderCommand {
  constructor(
    public readonly ebookId: string,
    public readonly customerEmail: string,
    public readonly quantity: number
  ) {}
}
// Involves: validation, inventory check, payment, notifications
```

**2. Operations Crossing Multiple Domains**

```typescript
// ✅ Good: Touches orders, payments, inventory, notifications
class CheckoutCommand {
  constructor(
    public readonly cartId: string,
    public readonly paymentMethod: PaymentMethod,
    public readonly shippingAddress: Address
  ) {}
}
```

**3. Actions That Need Audit Trail**

```typescript
// ✅ Good: Need to log who did what when
class ApproveRefundCommand {
  constructor(
    public readonly orderId: string,
    public readonly approvedBy: string,
    public readonly reason: string
  ) {}
}
```

**4. Operations from External Sources**

```typescript
// ✅ Good: Deserialized from API, queue, or webhook
const command = PlaceOrderCommand.fromJson(apiRequest.body);
await handler.execute(command);
```

### Decision Matrix

| Factor | Use Command? |
|--------|--------------|
| 3+ parameters | ✅ Yes |
| Business logic involved | ✅ Yes |
| Needs validation | ✅ Yes |
| Multiple steps | ✅ Yes |
| External input (API, queue) | ✅ Yes |
| Crosses domain boundaries | ✅ Yes |
| Simple query | ❌ No |
| 1-2 parameters | ❌ No |
| No business logic | ❌ No |
| Internal-only call | ❌ No |

## Conclusion

The Command Pattern transforms how you handle user intentions in your application. By encapsulating requests as first-class objects, you gain clarity, type safety, and testability.

### Key Takeaways

**Commands:**
- ✅ Are immutable data structures representing user intent
- ✅ Have clear, imperative names (PlaceOrder, CancelSubscription)
- ✅ Validate structure in constructor/factory
- ✅ Keep business logic in domain/service layers
- ✅ Make code self-documenting and type-safe
- ✅ Enable easy testing and evolution

**Remember:**
- Commands are data, not behavior
- Separate structural validation (in command) from business validation (in handler)
- One command per use case keeps focus clear
- Not every function needs a command—use judgment
- Commands represent *intent*, events represent *facts*

### What's Next?

Now that you understand commands, you're ready to explore:

- **Chapter 9: Layered Architecture** - How commands fit into the complete architecture
- **Chapter 10: Event-Driven Architecture** - Publishing events after successful commands
- **Chapter 11: CQRS Pattern** - Separating commands from queries
- **Chapter 12: Testing Strategies** - Advanced testing techniques for commands and handlers

### Further Reading

- *Domain-Driven Design* by Eric Evans (Application Services and Commands)
- *Implementing Domain-Driven Design* by Vaughn Vernon (Commands chapter)
- *Patterns of Enterprise Application Architecture* by Martin Fowler (Command pattern)

---

*Remember: Commands are about expressing **what** users want to do, not **how** it gets done. Keep them focused, immutable, and self-validating. Your code will thank you.* 🎯
