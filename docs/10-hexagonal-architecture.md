# Chapter 10: Hexagonal Architecture (Ports & Adapters)

## Introduction

In Chapter 9, you learned about Layered Architecture—organizing your code into Domain, Application, and Infrastructure layers with dependencies flowing inward. Now we'll explore **Hexagonal Architecture** (also known as **Ports and Adapters**), an architectural pattern that takes the same core principles but provides even more flexibility and symmetry.

Created by **Alistair Cockburn** in 2005, Hexagonal Architecture offers a powerful mental model: your application sits at the center, surrounded by ports (interfaces) and adapters (implementations) that connect it to the external world. Whether it's a REST API, a CLI, a database, or a message queue—they're all just adapters that plug into your application's ports.

### What is Hexagonal Architecture?

Hexagonal Architecture is an architectural pattern that:
- Places your **application at the center** (the hexagon)
- Defines **ports** as interfaces for communication
- Implements **adapters** that connect external systems to ports
- Makes **all external dependencies pluggable**

Think of it like a universal power adapter: the socket (port) stays the same, but you can plug in different adapters depending on where you are. Your application defines what it needs (the port), and adapters provide various implementations (database, API, mock, etc.).

### The Key Insight

**Traditional thinking:** "My app is built on Rails/Phoenix/React and needs PostgreSQL"

**Hexagonal thinking:** "My application is independent. It defines ports for what it needs, and I can plug in adapters for Rails, Phoenix, React, PostgreSQL, MongoDB, or anything else."

The application doesn't depend on frameworks, databases, or delivery mechanisms. They depend on *it*.

### Why "Hexagonal"?

The hexagon shape is just a convenient way to draw it—it has multiple sides for multiple adapters. The number of sides doesn't matter; what matters is that the application is at the center with ports all around it, and adapters on the outside.

```
         ╔═══════════════════════════╗
         ║                           ║
    REST ➜  Port (HTTP)          Port (DB) ➜ PostgreSQL
    API  ║                           ║      
         ║    APPLICATION CORE       ║
     CLI ➜  Port (CLI)       Port (Queue) ➜ RabbitMQ
         ║                           ║
         ║                           ║
GraphQL ➜  Port (GraphQL)     Port (Email) ➜ SendGrid
         ║                           ║
         ╚═══════════════════════════╝
```

**What You'll Learn:**
- The difference between driving and driven ports
- How to define ports in TypeScript and Elixir
- How to implement multiple adapters for the same port
- When to use Hexagonal vs Layered Architecture
- Project structure for both React and Elixir applications
- Common pitfalls and how to avoid them

Let's start by looking at the problem Hexagonal Architecture solves.

## The Problem

Without explicit ports and adapters, applications become tightly coupled to their infrastructure, making changes expensive and risky.

### The Tight Coupling Problem

```typescript
// ❌ BAD: Tightly coupled to Stripe
class OrderService {
  async placeOrder(order: Order) {
    // Directly dependent on Stripe SDK
    const stripe = new Stripe(process.env.STRIPE_KEY);
    const charge = await stripe.charges.create({
      amount: order.total * 100,
      currency: 'usd',
      source: order.paymentToken
    });
    
    // Directly dependent on PostgreSQL
    await db.query(
      'INSERT INTO orders VALUES ($1, $2, $3)',
      [order.id, order.customerId, charge.id]
    );
    
    // Directly dependent on SendGrid
    const sgMail = require('@sendgrid/mail');
    await sgMail.send({
      to: order.customerEmail,
      from: 'orders@example.com',
      subject: 'Order Confirmation',
      text: `Your order ${order.id} is confirmed`
    });
  }
}
```

**Problems:**
- ❌ **Can't swap implementations**: Switching from Stripe to PayPal requires changing OrderService
- ❌ **Can't test easily**: Testing requires mocking Stripe SDK, database client, and SendGrid
- ❌ **Framework lock-in**: Stuck with specific libraries and their APIs
- ❌ **Hard to support multiple adapters**: Can't easily add a CLI alongside the REST API
- ❌ **No clear boundaries**: Where does business logic end and infrastructure begin?

### The Multiple Interface Problem

What if you need to:
- Support both a web UI and a mobile app?
- Add a CLI for admin tasks?
- Support both REST and GraphQL APIs?
- Use different databases for different deployments?

Without ports and adapters, you end up duplicating logic or creating messy abstractions that leak implementation details.

Hexagonal Architecture solves these problems with **explicit ports** for all external communication.

## Core Concepts

Hexagonal Architecture has three main concepts: the Application Core, Ports, and Adapters.

### Application Core

The **Application Core** contains all your business logic, domain models, and use cases. It's framework-agnostic and has no dependencies on external systems.

**What's inside:**
- Domain entities (Order, Customer, Product)
- Value objects (Email, Money, Address)
- Business rules and validation
- Use case implementations
- Domain services

**Characteristics:**
- ✅ Pure TypeScript/Elixir—no framework imports
- ✅ No knowledge of HTTP, databases, or UI
- ✅ Fast to test—just plain functions/classes
- ✅ Long-lived—changes only when business changes

### Ports (Interfaces)

**Ports** are interfaces that define how the outside world communicates with your application. There are two types:

#### Primary Ports (Driving Ports)

**Primary Ports** define how external actors **use** your application. These represent your use cases.

**Examples:**
- Place an order
- Cancel an order
- Process a payment
- Generate a report

**Also called:** Driving ports, inbound ports, use case interfaces

#### Secondary Ports (Driven Ports)

**Secondary Ports** define how your application **uses** external systems. These represent dependencies your application needs.

**Examples:**
- OrderRepository (for persistence)
- PaymentGateway (for payments)
- EmailService (for notifications)
- EventPublisher (for events)

**Also called:** Driven ports, outbound ports, infrastructure interfaces

### Adapters (Implementations)

**Adapters** are concrete implementations of ports. They translate between your application and the external world.

#### Primary Adapters (Driving Adapters)

**Primary Adapters** call into your application. They implement the "input" side.

**Examples:**
- REST API controller
- GraphQL resolver
- CLI command
- React component
- Background job
- Event listener

#### Secondary Adapters (Driven Adapters)

**Secondary Adapters** are called by your application. They implement the "output" side.

**Examples:**
- PostgreSQL repository
- MongoDB repository
- In-memory repository (for tests)
- Stripe payment gateway
- SendGrid email service
- RabbitMQ event publisher

### The Symmetry

Hexagonal Architecture is **symmetric**:

```
LEFT SIDE (Driving)              CENTER                RIGHT SIDE (Driven)
====================            ========               ====================
REST Controller  ➜           Primary Port           ➜  Repository Interface
GraphQL Resolver ➜         (Use Case: Place       ➜  Payment Gateway Interface
CLI Command      ➜          Order Service)         ➜  Email Service Interface
React Component  ➜                                 ➜  Event Publisher Interface

             Adapters call ports → Application Core → calls ports ← Adapters implement
```

The application doesn't care who's calling it (left side) or what implementations exist (right side). It only knows about ports.

## Hexagonal vs Layered Architecture

Both patterns share the same core principles, but they differ in structure and emphasis.

### Similarities

**Both patterns:**
- ✅ Separate concerns
- ✅ Use dependency inversion
- ✅ Put domain logic at the center
- ✅ Make infrastructure swappable
- ✅ Improve testability

**Layered Architecture structure:**
```
┌─────────────────────────────┐
│   Infrastructure Layer      │  (Controllers, DB, UI)
│   ↓ depends on              │
├─────────────────────────────┤
│   Application Layer         │  (Use Cases)
│   ↓ depends on              │
├─────────────────────────────┤
│   Domain Layer              │  (Entities, Rules)
│   ↓ no dependencies         │
└─────────────────────────────┘
```

**Hexagonal Architecture structure:**
```
        Primary Adapters               Secondary Adapters
     (REST, CLI, GraphQL)           (DB, Queue, Email, API)
              ↓                              ↑
        Primary Ports                  Secondary Ports
        (Use Cases)                  (Repositories, Gateways)
              ↓                              ↑
              └──→  APPLICATION CORE  ←──────┘
                   (Domain + Use Cases)
```

### Key Differences

**1. Explicit Ports**

**Layered:** Layers talk to each other directly
**Hexagonal:** All communication goes through explicit ports (interfaces)

**2. Symmetry**

**Layered:** Vertical (top-down) dependency flow
**Hexagonal:** Symmetric—ports on all sides of the hexagon

**3. Flexibility**

**Layered:** Good for single UI, single database
**Hexagonal:** Better for multiple adapters (web + CLI + mobile, PostgreSQL + MongoDB + In-Memory)

**4. Emphasis**

**Layered:** Emphasizes layers and dependency flow
**Hexagonal:** Emphasizes ports/adapters and complete isolation

### When to Choose Which

**Use Layered Architecture when:**
- Single interface (just a web app)
- Single database
- Team is new to clean architecture
- Simpler mental model preferred

**Use Hexagonal Architecture when:**
- Multiple interfaces (REST + CLI + GraphQL)
- Might change databases
- Complex integrations that could change
- Need to swap implementations frequently
- Want maximum flexibility

**Reality:** They're more similar than different. Many teams use layered architecture but think of repositories and controllers as "adapters" in their heads. Choose what works for your team.

## Defining Ports

Let's see how to define ports in both TypeScript and Elixir.

### TypeScript: Primary Port (Use Case)

A primary port represents a use case—something your application can do.

```typescript
// domain/ports/primary/PlaceOrderUseCase.ts
import { Result } from '../../shared/Result';
import { PlaceOrderCommand } from '../../../application/commands/PlaceOrderCommand';
import { OrderId } from '../../order/OrderId';

/**
 * Primary Port: Represents the "Place Order" use case.
 * This is what the outside world uses to place orders.
 */
export interface PlaceOrderUseCase {
  execute(command: PlaceOrderCommand): Promise<Result<OrderId, Error>>;
}
```

The application service implements this port:

```typescript
// application/services/PlaceOrderService.ts
import { PlaceOrderUseCase } from '../../domain/ports/primary/PlaceOrderUseCase';
import { OrderRepository } from '../../domain/ports/secondary/OrderRepository';
import { PaymentGateway } from '../../domain/ports/secondary/PaymentGateway';
import { Result, ok, err } from '../../domain/shared/Result';

/**
 * Application Service implements the Primary Port.
 */
export class PlaceOrderService implements PlaceOrderUseCase {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly paymentGateway: PaymentGateway,
    private readonly ebookRepository: EbookRepository
  ) {}

  async execute(command: PlaceOrderCommand): Promise<Result<OrderId, Error>> {
    // 1. Fetch ebook
    const ebookResult = await this.ebookRepository.findById(command.ebookId);
    if (ebookResult.isErr()) {
      return err(new Error('Ebook not found'));
    }

    // 2. Create domain model
    const orderResult = Order.create({
      id: OrderId.generate(),
      ebookId: command.ebookId,
      customerEmail: command.customerEmail,
      quantity: command.quantity,
      pricePerUnit: ebookResult.value.price
    });

    if (orderResult.isErr()) {
      return err(orderResult.error);
    }

    const order = orderResult.value;

    // 3. Process payment
    const paymentResult = await this.paymentGateway.charge({
      amount: order.calculateTotal(),
      customer: order.customerEmail
    });

    if (paymentResult.isErr()) {
      return err(new Error('Payment failed'));
    }

    // 4. Save order
    const saveResult = await this.orderRepository.save(order);
    if (saveResult.isErr()) {
      return err(new Error('Failed to save order'));
    }

    return ok(order.id);
  }
}
```

### TypeScript: Secondary Port (Repository)

A secondary port represents a dependency your application needs.

```typescript
// domain/ports/secondary/OrderRepository.ts
import { Result } from '../../shared/Result';
import { Order } from '../../order/Order';
import { OrderId } from '../../order/OrderId';

/**
 * Secondary Port: Defines what the application needs from persistence.
 * The application doesn't care if it's PostgreSQL, MongoDB, or in-memory.
 */
export interface OrderRepository {
  save(order: Order): Promise<Result<OrderId, Error>>;
  findById(id: OrderId): Promise<Result<Order, Error>>;
  findByCustomerEmail(email: string): Promise<Result<Order[], Error>>;
}
```

Multiple adapters can implement this port:

```typescript
// adapters/secondary/persistence/PostgresOrderRepository.ts
import { OrderRepository } from '../../../domain/ports/secondary/OrderRepository';
import { Order } from '../../../domain/order/Order';
import { Result, ok, err } from '../../../domain/shared/Result';

export class PostgresOrderRepository implements OrderRepository {
  constructor(private readonly pool: pg.Pool) {}

  async save(order: Order): Promise<Result<OrderId, Error>> {
    try {
      const query = `
        INSERT INTO orders (id, ebook_id, customer_email, quantity, total)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING id
      `;
      const values = [
        order.id.value,
        order.ebookId,
        order.customerEmail.toString(),
        order.quantity,
        order.calculateTotal().toNumber()
      ];

      const result = await this.pool.query(query, values);
      return ok(OrderId.fromString(result.rows[0].id).value);
    } catch (error) {
      return err(new Error('Database error'));
    }
  }

  async findById(id: OrderId): Promise<Result<Order, Error>> {
    // Implementation
  }

  async findByCustomerEmail(email: string): Promise<Result<Order[], Error>> {
    // Implementation
  }
}

// adapters/secondary/persistence/InMemoryOrderRepository.ts
export class InMemoryOrderRepository implements OrderRepository {
  private orders: Map<string, Order> = new Map();

  async save(order: Order): Promise<Result<OrderId, Error>> {
    this.orders.set(order.id.value, order);
    return ok(order.id);
  }

  async findById(id: OrderId): Promise<Result<Order, Error>> {
    const order = this.orders.get(id.value);
    return order ? ok(order) : err(new Error('Order not found'));
  }

  async findByCustomerEmail(email: string): Promise<Result<Order[], Error>> {
    const orders = Array.from(this.orders.values())
      .filter(o => o.customerEmail.toString() === email);
    return ok(orders);
  }
}
```

### Elixir: Primary Port (Use Case)

In Elixir, we use behaviours to define ports.

```elixir
# lib/shop/domain/ports/primary/place_order_use_case.ex
defmodule Shop.Domain.Ports.Primary.PlaceOrderUseCase do
  @moduledoc """
  Primary Port: Represents the "Place Order" use case.
  This is what the outside world uses to place orders.
  """

  alias Shop.Application.Commands.PlaceOrderCommand
  alias Shop.Domain.Order

  @callback execute(PlaceOrderCommand.t()) ::
    {:ok, Order.id()} | {:error, term()}
end
```

The application service implements this behaviour:

```elixir
# lib/shop/application/services/place_order_service.ex
defmodule Shop.Application.Services.PlaceOrderService do
  @moduledoc """
  Application Service implements the Primary Port.
  """

  @behaviour Shop.Domain.Ports.Primary.PlaceOrderUseCase

  alias Shop.Application.Commands.PlaceOrderCommand
  alias Shop.Domain.{Order, OrderId}
  
  # Secondary ports (configured)
  @order_repo Application.compile_env(:shop, :order_repository)
  @ebook_repo Application.compile_env(:shop, :ebook_repository)
  @payment_gateway Application.compile_env(:shop, :payment_gateway)

  @impl true
  def execute(%PlaceOrderCommand{} = command) do
    with {:ok, ebook} <- @ebook_repo.find_by_id(command.ebook_id),
         {:ok, order} <- create_order(command, ebook),
         {:ok, payment} <- @payment_gateway.charge(
           order.calculate_total(),
           order.customer_email
         ),
         {:ok, order_id} <- @order_repo.save(order) do
      {:ok, order_id}
    else
      {:error, :ebook_not_found} -> {:error, :ebook_not_found}
      {:error, :payment_failed} -> {:error, :payment_failed}
      {:error, reason} -> {:error, reason}
    end
  end

  defp create_order(command, ebook) do
    Order.new(
      OrderId.generate(),
      command.ebook_id,
      command.customer_email,
      command.quantity,
      ebook.price
    )
  end
end
```

### Elixir: Secondary Port (Repository)

```elixir
# lib/shop/domain/ports/secondary/order_repository.ex
defmodule Shop.Domain.Ports.Secondary.OrderRepository do
  @moduledoc """
  Secondary Port: Defines what the application needs from persistence.
  The application doesn't care if it's PostgreSQL, Mnesia, or in-memory.
  """

  alias Shop.Domain.{Order, OrderId}

  @callback save(Order.t()) :: {:ok, OrderId.t()} | {:error, term()}
  @callback find_by_id(OrderId.t()) :: {:ok, Order.t()} | {:error, :not_found}
  @callback find_by_customer_email(String.t()) :: {:ok, [Order.t()]} | {:error, term()}
end
```

Multiple adapters can implement this behaviour:

```elixir
# lib/shop/adapters/secondary/persistence/postgres_order_repository.ex
defmodule Shop.Adapters.Secondary.Persistence.PostgresOrderRepository do
  @moduledoc """
  PostgreSQL implementation of OrderRepository port.
  """

  @behaviour Shop.Domain.Ports.Secondary.OrderRepository

  alias Shop.Repo
  alias Shop.Adapters.Secondary.Persistence.OrderSchema
  alias Shop.Domain.{Order, OrderId, Email, Money}

  @impl true
  def save(%Order{} = order) do
    params = %{
      id: OrderId.to_string(order.id),
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
      {:ok, schema} -> {:ok, OrderId.from_string(schema.id)}
      {:error, _changeset} -> {:error, :save_failed}
    end
  end

  @impl true
  def find_by_id(%OrderId{} = id) do
    case Repo.get(OrderSchema, OrderId.to_string(id)) do
      nil -> {:error, :not_found}
      schema -> schema_to_domain(schema)
    end
  end

  @impl true
  def find_by_customer_email(email) when is_binary(email) do
    schemas =
      OrderSchema
      |> where([o], o.customer_email == ^email)
      |> Repo.all()

    orders =
      schemas
      |> Enum.map(&schema_to_domain/1)
      |> Enum.filter(&match?({:ok, _}, &1))
      |> Enum.map(fn {:ok, order} -> order end)

    {:ok, orders}
  end

  defp schema_to_domain(%OrderSchema{} = schema) do
    Order.new(
      OrderId.from_string(schema.id),
      schema.ebook_id,
      schema.customer_email,
      schema.quantity,
      Money.new(schema.price_per_unit, :USD)
    )
  end
end

# lib/shop/adapters/secondary/persistence/in_memory_order_repository.ex
defmodule Shop.Adapters.Secondary.Persistence.InMemoryOrderRepository do
  @moduledoc """
  In-memory implementation of OrderRepository port.
  Used for testing.
  """

  @behaviour Shop.Domain.Ports.Secondary.OrderRepository

  use Agent

  alias Shop.Domain.{Order, OrderId}

  def start_link(_opts) do
    Agent.start_link(fn -> %{} end, name: __MODULE__)
  end

  @impl true
  def save(%Order{} = order) do
    Agent.update(__MODULE__, fn orders ->
      Map.put(orders, OrderId.to_string(order.id), order)
    end)
    {:ok, order.id}
  end

  @impl true
  def find_by_id(%OrderId{} = id) do
    case Agent.get(__MODULE__, &Map.get(&1, OrderId.to_string(id))) do
      nil -> {:error, :not_found}
      order -> {:ok, order}
    end
  end

  @impl true
  def find_by_customer_email(email) when is_binary(email) do
    orders =
      Agent.get(__MODULE__, fn orders ->
        orders
        |> Map.values()
        |> Enum.filter(&(Email.to_string(&1.customer_email) == email))
      end)

    {:ok, orders}
  end

  def clear do
    Agent.update(__MODULE__, fn _ -> %{} end)
  end
end
```

## Multiple Driving Adapters

One of the key benefits of Hexagonal Architecture is supporting multiple ways to interact with your application. Let's see how the same use case can be called from different adapters.

### React Component Adapter

```typescript
// adapters/primary/web/OrderForm.tsx
import React, { useState } from 'react';
import { PlaceOrderCommand } from '../../../application/commands/PlaceOrderCommand';
import { usePlaceOrderUseCase } from '../../../composition/hooks';

/**
 * Primary Adapter: React component
 * Calls the PlaceOrderUseCase port
 */
export function OrderForm({ ebookId }: { ebookId: string }) {
  const placeOrderUseCase = usePlaceOrderUseCase();
  const [email, setEmail] = useState('');
  const [quantity, setQuantity] = useState(1);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      const command = new PlaceOrderCommand(ebookId, email, quantity);
      const result = await placeOrderUseCase.execute(command);

      if (result.isOk()) {
        alert(`Order placed! Order ID: ${result.value.value}`);
        // Redirect or show success message
      } else {
        setError(result.error.message);
      }
    } catch (err) {
      setError('An unexpected error occurred');
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
        {loading ? 'Processing...' : 'Place Order'}
      </button>
      {error && <div className="error">{error}</div>}
    </form>
  );
}
```

### REST API Adapter (Phoenix)

```elixir
# lib/shop_web/controllers/order_controller.ex
defmodule ShopWeb.OrderController do
  @moduledoc """
  Primary Adapter: REST API Controller
  Calls the PlaceOrderUseCase port
  """

  use ShopWeb, :controller

  alias Shop.Application.Commands.PlaceOrderCommand
  
  # Get configured implementation of the port
  @place_order_use_case Application.compile_env(:shop, :place_order_use_case)

  def create(conn, params) do
    with {:ok, command} <- PlaceOrderCommand.from_params(params),
         {:ok, order_id} <- @place_order_use_case.execute(command) do
      conn
      |> put_status(:created)
      |> json(%{order_id: order_id})
    else
      {:error, :invalid_params} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Invalid parameters"})

      {:error, :ebook_not_found} ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Ebook not found"})

      {:error, :payment_failed} ->
        conn
        |> put_status(:payment_required)
        |> json(%{error: "Payment failed"})

      {:error, reason} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: to_string(reason)})
    end
  end
end
```

### CLI Adapter (Elixir)

```elixir
# lib/shop/adapters/primary/cli/order_command.ex
defmodule Shop.Adapters.Primary.CLI.OrderCommand do
  @moduledoc """
  Primary Adapter: Command-line interface
  Calls the PlaceOrderUseCase port
  """

  alias Shop.Application.Commands.PlaceOrderCommand
  
  @place_order_use_case Application.compile_env(:shop, :place_order_use_case)

  def run(args) do
    case parse_args(args) do
      {:ok, ebook_id, email, quantity} ->
        place_order(ebook_id, email, quantity)

      {:error, reason} ->
        IO.puts("Error: #{reason}")
        print_usage()
        System.halt(1)
    end
  end

  defp place_order(ebook_id, email, quantity) do
    command = %PlaceOrderCommand{
      ebook_id: ebook_id,
      customer_email: email,
      quantity: quantity
    }

    case @place_order_use_case.execute(command) do
      {:ok, order_id} ->
        IO.puts("\n✓ Order created successfully!")
        IO.puts("Order ID: #{order_id}")
        IO.puts("Email confirmation sent to: #{email}")
        System.halt(0)

      {:error, :ebook_not_found} ->
        IO.puts("\n✗ Error: Ebook not found")
        System.halt(1)

      {:error, :payment_failed} ->
        IO.puts("\n✗ Error: Payment failed")
        System.halt(1)

      {:error, reason} ->
        IO.puts("\n✗ Error: #{reason}")
        System.halt(1)
    end
  end

  defp parse_args([ebook_id, email, quantity_str]) do
    case Integer.parse(quantity_str) do
      {quantity, ""} when quantity > 0 ->
        {:ok, ebook_id, email, quantity}

      _ ->
        {:error, "Invalid quantity"}
    end
  end

  defp parse_args(_), do: {:error, "Invalid arguments"}

  defp print_usage do
    IO.puts("""
    Usage: shop orders place <ebook_id> <email> <quantity>

    Example:
      shop orders place ebook-123 customer@example.com 5
    """)
  end
end
```

### GraphQL Adapter (TypeScript)

```typescript
// adapters/primary/graphql/OrderResolver.ts
import { PlaceOrderUseCase } from '../../../domain/ports/primary/PlaceOrderUseCase';
import { PlaceOrderCommand } from '../../../application/commands/PlaceOrderCommand';

/**
 * Primary Adapter: GraphQL resolver
 * Calls the PlaceOrderUseCase port
 */
export class OrderResolver {
  constructor(private readonly placeOrderUseCase: PlaceOrderUseCase) {}

  async placeOrder(
    _: any,
    args: { input: PlaceOrderInput },
    context: Context
  ): Promise<PlaceOrderPayload> {
    const { ebookId, customerEmail, quantity } = args.input;

    const command = new PlaceOrderCommand(ebookId, customerEmail, quantity);
    const result = await this.placeOrderUseCase.execute(command);

    if (result.isOk()) {
      return {
        success: true,
        orderId: result.value.value,
        message: 'Order placed successfully'
      };
    } else {
      return {
        success: false,
        orderId: null,
        message: result.error.message
      };
    }
  }
}

interface PlaceOrderInput {
  ebookId: string;
  customerEmail: string;
  quantity: number;
}

interface PlaceOrderPayload {
  success: boolean;
  orderId: string | null;
  message: string;
}
```

**Key Point:** All these adapters call the same `PlaceOrderUseCase` port. The business logic is executed once, consistently, regardless of how the user interacts with your application.

## Project Structure

Let's see how to organize projects using Hexagonal Architecture.

### React/TypeScript Structure

```
src/
├── domain/                          # Application Core (Domain + Ports)
│   ├── models/
│   │   ├── Order.ts                # Entity
│   │   ├── Ebook.ts
│   │   └── Customer.ts
│   ├── value-objects/
│   │   ├── OrderId.ts              # Value objects
│   │   ├── Email.ts
│   │   └── Money.ts
│   ├── ports/
│   │   ├── primary/                # Driving ports (use cases)
│   │   │   ├── PlaceOrderUseCase.ts
│   │   │   ├── CancelOrderUseCase.ts
│   │   │   └── GetOrderUseCase.ts
│   │   └── secondary/              # Driven ports (dependencies)
│   │       ├── OrderRepository.ts
│   │       ├── EbookRepository.ts
│   │       ├── PaymentGateway.ts
│   │       └── NotificationService.ts
│   └── shared/
│       └── Result.ts
│
├── application/                     # Application Services (Port Implementations)
│   ├── commands/
│   │   ├── PlaceOrderCommand.ts
│   │   └── CancelOrderCommand.ts
│   └── services/
│       ├── PlaceOrderService.ts    # Implements PlaceOrderUseCase
│       └── CancelOrderService.ts
│
└── adapters/
    ├── primary/                     # Driving adapters (input)
    │   ├── web/
    │   │   ├── OrderForm.tsx       # React component
    │   │   ├── OrderList.tsx
    │   │   └── OrderDetail.tsx
    │   ├── api/
    │   │   └── OrderController.ts  # REST API
    │   └── graphql/
    │       └── OrderResolver.ts    # GraphQL
    │
    └── secondary/                   # Driven adapters (output)
        ├── persistence/
        │   ├── HttpOrderRepository.ts       # Real implementation
        │   ├── HttpEbookRepository.ts
        │   └── InMemoryOrderRepository.ts   # Test implementation
        ├── payment/
        │   ├── StripePaymentGateway.ts      # Real implementation
        │   └── FakePaymentGateway.ts        # Test implementation
        └── notification/
            ├── EmailNotificationService.ts
            └── MockNotificationService.ts
```

### Elixir/Phoenix Structure

```
lib/
├── shop/                            # Application Context
│   ├── domain/                      # Application Core (Domain + Ports)
│   │   ├── models/
│   │   │   ├── order.ex            # Entity
│   │   │   ├── ebook.ex
│   │   │   └── customer.ex
│   │   ├── value_objects/
│   │   │   ├── order_id.ex         # Value objects
│   │   │   ├── email.ex
│   │   │   └── money.ex
│   │   └── ports/
│   │       ├── primary/             # Driving ports (use cases)
│   │       │   ├── place_order_use_case.ex
│   │       │   ├── cancel_order_use_case.ex
│   │       │   └── get_order_use_case.ex
│   │       └── secondary/           # Driven ports (dependencies)
│   │           ├── order_repository.ex
│   │           ├── ebook_repository.ex
│   │           ├── payment_gateway.ex
│   │           └── notification_service.ex
│   │
│   ├── application/                 # Application Services (Port Implementations)
│   │   ├── commands/
│   │   │   ├── place_order_command.ex
│   │   │   └── cancel_order_command.ex
│   │   └── services/
│   │       ├── place_order_service.ex      # Implements PlaceOrderUseCase
│   │       └── cancel_order_service.ex
│   │
│   └── adapters/
│       ├── primary/                 # Driving adapters (input)
│       │   ├── cli/
│       │   │   └── order_command.ex
│       │   └── graphql/
│       │       └── order_resolver.ex
│       │
│       └── secondary/               # Driven adapters (output)
│           ├── persistence/
│           │   ├── postgres_order_repository.ex    # Real
│           │   ├── postgres_ebook_repository.ex
│           │   ├── in_memory_order_repository.ex   # Test
│           │   ├── order_schema.ex                 # Ecto schema
│           │   └── ebook_schema.ex
│           ├── payment/
│           │   ├── stripe_payment_gateway.ex       # Real
│           │   └── fake_payment_gateway.ex         # Test
│           └── notification/
│               ├── sendgrid_notification_service.ex
│               └── mock_notification_service.ex
│
└── shop_web/                        # Web Interface (Primary Adapter)
    ├── controllers/
    │   └── order_controller.ex      # REST API adapter
    ├── views/
    │   └── order_view.ex
    ├── router.ex
    └── endpoint.ex
```

### Configuration

```elixir
# config/config.exs
config :shop,
  # Primary ports
  place_order_use_case: Shop.Application.Services.PlaceOrderService,
  
  # Secondary ports
  order_repository: Shop.Adapters.Secondary.Persistence.PostgresOrderRepository,
  ebook_repository: Shop.Adapters.Secondary.Persistence.PostgresEbookRepository,
  payment_gateway: Shop.Adapters.Secondary.Payment.StripePaymentGateway,
  notification_service: Shop.Adapters.Secondary.Notification.SendGridNotificationService

# config/test.exs
config :shop,
  # Same ports, different adapters for testing
  place_order_use_case: Shop.Application.Services.PlaceOrderService,
  
  order_repository: Shop.Adapters.Secondary.Persistence.InMemoryOrderRepository,
  ebook_repository: Shop.Adapters.Secondary.Persistence.InMemoryEbookRepository,
  payment_gateway: Shop.Adapters.Secondary.Payment.FakePaymentGateway,
  notification_service: Shop.Adapters.Secondary.Notification.MockNotificationService
```

## Testing Benefits

Hexagonal Architecture makes testing straightforward because you can swap adapters easily. This is one of the most compelling benefits of the pattern.

### The Testing Pyramid with Hexagonal Architecture

```
        /\
       /  \      E2E Tests (Few)
      /────\     - Test with real adapters
     /      \    - Slow, brittle
    /────────\   
   /          \  Integration Tests (Some)
  /────────────\ - Test adapter implementations
 /              \
/────────────────\ Unit Tests (Many)
Application Core  - Test with fake adapters
   (Fast)         - Fast, deterministic
```

With Hexagonal Architecture, you can test at each level effectively:

**1. Unit Tests (Application Core):**
- Use fake/in-memory adapters
- Fast (milliseconds)
- No external dependencies
- Test business logic thoroughly

**2. Integration Tests (Adapters):**
- Test each adapter implementation
- Ensure adapters work correctly
- Test database queries, API calls, etc.

**3. E2E Tests (Full System):**
- Test with real adapters
- Ensure everything works together
- Fewer tests needed (unit tests cover logic)

### Testing with All In-Memory Adapters

```typescript
// application/services/PlaceOrderService.test.ts
describe('PlaceOrderService', () => {
  let service: PlaceOrderService;
  let orderRepo: InMemoryOrderRepository;
  let ebookRepo: InMemoryEbookRepository;
  let paymentGateway: FakePaymentGateway;

  beforeEach(() => {
    // All secondary adapters are test doubles
    orderRepo = new InMemoryOrderRepository();
    ebookRepo = new InMemoryEbookRepository();
    paymentGateway = new FakePaymentGateway();

    // Service uses test adapters
    service = new PlaceOrderService(orderRepo, ebookRepo, paymentGateway);

    // Seed test data
    const ebook = Ebook.create({
      id: 'ebook-1',
      title: 'Domain-Driven Design',
      price: Money.fromAmount(49.99, 'USD')
    }).value;
    ebookRepo.save(ebook);
  });

  it('places order successfully', async () => {
    const command = new PlaceOrderCommand(
      'ebook-1',
      'customer@example.com',
      2
    );

    const result = await service.execute(command);

    expect(result.isOk()).toBe(true);

    // Verify order was saved
    const savedOrders = await orderRepo.findAll();
    expect(savedOrders).toHaveLength(1);
    expect(savedOrders[0].quantity).toBe(2);

    // Verify payment was charged
    expect(paymentGateway.charges).toHaveLength(1);
    expect(paymentGateway.charges[0].amount.toNumber()).toBe(99.98);
  });

  it('fails when payment is declined', async () => {
    // Configure fake gateway to decline
    paymentGateway.setShouldDecline(true);

    const command = new PlaceOrderCommand(
      'ebook-1',
      'customer@example.com',
      1
    );

    const result = await service.execute(command);

    expect(result.isErr()).toBe(true);
    expect(result.error.message).toContain('Payment failed');

    // Verify no order was saved
    const savedOrders = await orderRepo.findAll();
    expect(savedOrders).toHaveLength(0);
  });

  it('handles invalid business rules', async () => {
    const command = new PlaceOrderCommand(
      'ebook-1',
      'invalid-email',  // Invalid email
      1
    );

    const result = await service.execute(command);

    expect(result.isErr()).toBe(true);
    expect(result.error.message).toContain('email');
  });
});
```

### Testing Different Scenarios with Fake Adapters

The beauty of hexagonal architecture is that you can configure fake adapters to simulate various scenarios:

```typescript
// Test adapter with configurable behavior
class FakePaymentGateway implements PaymentGateway {
  private shouldDecline = false;
  private shouldTimeout = false;
  public charges: Array<{ amount: Money; customer: Email }> = [];

  setShouldDecline(decline: boolean) {
    this.shouldDecline = decline;
  }

  setShouldTimeout(timeout: boolean) {
    this.shouldTimeout = timeout;
  }

  async charge(amount: Money, customer: Email): Promise<Result<PaymentId, Error>> {
    if (this.shouldTimeout) {
      await new Promise(resolve => setTimeout(resolve, 30000));
      return err(new Error('Payment timeout'));
    }

    if (this.shouldDecline) {
      return err(new Error('Card declined'));
    }

    this.charges.push({ amount, customer });
    return ok(PaymentId.generate());
  }
}
```

Now you can test:
- ✅ Happy path (payment succeeds)
- ✅ Card declined scenario
- ✅ Payment timeout scenario
- ✅ Network error scenario

All without touching a real payment gateway!

### Testing in Elixir

```elixir
# test/shop/application/place_order_service_test.exs
defmodule Shop.Application.PlaceOrderServiceTest do
  use ExUnit.Case, async: true

  alias Shop.Application.{PlaceOrderService, PlaceOrderCommand}
  alias Shop.Domain.{Order, Money}
  alias Shop.Test.Support.{InMemoryOrderRepository, InMemoryEbookRepository, FakePaymentGateway}

  setup do
    # Start in-memory repositories
    {:ok, _} = InMemoryOrderRepository.start_link([])
    {:ok, _} = InMemoryEbookRepository.start_link([])
    {:ok, _} = FakePaymentGateway.start_link([])

    # Seed test data
    ebook = %Ebook{
      id: "ebook-1",
      title: "Domain-Driven Design",
      price: Money.new(4999, :USD)
    }
    InMemoryEbookRepository.save(ebook)

    :ok
  end

  describe "execute/1" do
    test "places order successfully" do
      command = %PlaceOrderCommand{
        ebook_id: "ebook-1",
        customer_email: "customer@example.com",
        quantity: 2
      }

      assert {:ok, order_id} = PlaceOrderService.execute(command)
      assert is_binary(order_id)

      # Verify order was saved
      assert {:ok, order} = InMemoryOrderRepository.find_by_id(order_id)
      assert order.quantity == 2

      # Verify payment was charged
      assert {:ok, charges} = FakePaymentGateway.get_charges()
      assert length(charges) == 1
    end

    test "fails when payment is declined" do
      # Configure fake gateway to decline
      FakePaymentGateway.set_should_decline(true)

      command = %PlaceOrderCommand{
        ebook_id: "ebook-1",
        customer_email: "customer@example.com",
        quantity: 1
      }

      assert {:error, :payment_failed} = PlaceOrderService.execute(command)

      # Verify no order was saved
      assert {:ok, []} = InMemoryOrderRepository.find_all()
    end

    test "handles concurrent orders correctly" do
      # Test race conditions with in-memory adapters
      tasks = for i <- 1..10 do
        Task.async(fn ->
          command = %PlaceOrderCommand{
            ebook_id: "ebook-1",
            customer_email: "customer#{i}@example.com",
            quantity: 1
          }
          PlaceOrderService.execute(command)
        end)
      end

      results = Task.await_many(tasks)
      
      # All should succeed
      assert Enum.all?(results, &match?({:ok, _}, &1))
      
      # All 10 orders should be saved
      assert {:ok, orders} = InMemoryOrderRepository.find_all()
      assert length(orders) == 10
    end
  end
end
```

### Test vs Production Adapters

The power of hexagonal architecture shows in how easily you can swap configurations:

**Test configuration:**
- InMemoryOrderRepository (fast, no database needed)
- FakePaymentGateway (no real charges, configurable behavior)
- MockNotificationService (no real emails, verify calls)
- Runs in milliseconds
- Deterministic and reliable
- Can simulate edge cases

**Production configuration:**
- PostgresOrderRepository (real database)
- StripePaymentGateway (real payments)
- SendGridNotificationService (real emails)
- Handles actual infrastructure
- Idempotent and resilient
- Monitors and logs

**Staging configuration:**
- PostgresOrderRepository (real database, staging data)
- StripeTestModePaymentGateway (Stripe test mode)
- LogNotificationService (logs emails instead of sending)
- Tests real infrastructure without side effects

**Same business logic, different adapters.** This is the power of ports!

### Contract Testing for Adapters

Since multiple adapters implement the same port, you can write contract tests to ensure they all behave correctly:

```typescript
// test/contracts/OrderRepositoryContract.test.ts
export function orderRepositoryContractTests(
  createRepository: () => OrderRepository,
  cleanup: () => Promise<void>
) {
  let repository: OrderRepository;

  beforeEach(() => {
    repository = createRepository();
  });

  afterEach(async () => {
    await cleanup();
  });

  it('saves and retrieves an order', async () => {
    const order = Order.create({
      id: OrderId.generate(),
      ebookId: 'ebook-1',
      customerEmail: 'test@example.com',
      quantity: 1,
      pricePerUnit: Money.fromAmount(10, 'USD')
    }).value;

    const saveResult = await repository.save(order);
    expect(saveResult.isOk()).toBe(true);

    const findResult = await repository.findById(order.id);
    expect(findResult.isOk()).toBe(true);
    expect(findResult.value.id).toEqual(order.id);
  });

  it('returns error when order not found', async () => {
    const result = await repository.findById(OrderId.generate());
    expect(result.isErr()).toBe(true);
  });

  // More contract tests...
}

// Test PostgresOrderRepository
describe('PostgresOrderRepository', () => {
  orderRepositoryContractTests(
    () => new PostgresOrderRepository(testDbPool),
    async () => await cleanupTestDatabase()
  );
});

// Test InMemoryOrderRepository
describe('InMemoryOrderRepository', () => {
  orderRepositoryContractTests(
    () => new InMemoryOrderRepository(),
    async () => {} // No cleanup needed
  );
});
```

Both implementations must pass the same contract tests, ensuring they behave identically from the application's perspective.

## Real-World Example: System Evolution

Let's see how Hexagonal Architecture enables a system to evolve over time without major rewrites.

### Year 1: Simple Web App

**Initial requirements:**
- Web application for placing orders
- PostgreSQL database
- Stripe for payments

**Architecture:**
```typescript
// Application Core (unchanged throughout evolution)
class PlaceOrderService implements PlaceOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private paymentGateway: PaymentGateway
  ) {}
  
  async execute(command: PlaceOrderCommand) {
    // Business logic stays the same
  }
}

// Initial adapters
- PostgresOrderRepository (secondary)
- StripePaymentGateway (secondary)
- OrderController (REST API) (primary)
```

### Year 2: Add Mobile Support

**New requirement:** Mobile app needs to place orders

**Changes needed:**
```typescript
// Application Core: NO CHANGES
// Just add a new primary adapter

// New adapter
class GraphQLOrderResolver {
  constructor(private placeOrderUseCase: PlaceOrderUseCase) {}
  
  async placeOrder(input) {
    return this.placeOrderUseCase.execute(command);
  }
}
```

**Effort:** 1 day to add GraphQL adapter  
**Risk:** Low (existing web app unaffected)

### Year 3: Add CLI for Operations Team

**New requirement:** Operations team needs CLI to place orders manually

**Changes needed:**
```elixir
# Application Core: NO CHANGES
# Just add another primary adapter

# New adapter
defmodule Shop.Adapters.Primary.CLI.OrderCommand do
  @place_order_use_case Application.compile_env(:shop, :place_order_use_case)
  
  def run(args) do
    command = parse_args(args)
    @place_order_use_case.execute(command)
  end
end
```

**Effort:** 2 days to add CLI adapter  
**Risk:** Low (web and mobile apps unaffected)

### Year 4: Switch from Stripe to Multi-Gateway Support

**New requirement:** Support both Stripe and PayPal, customer chooses

**Changes needed:**
```typescript
// Application Core: MINIMAL CHANGES
// Just update service to choose gateway

class PlaceOrderService implements PlaceOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private stripeGateway: PaymentGateway,
    private paypalGateway: PaymentGateway
  ) {}
  
  async execute(command: PlaceOrderCommand) {
    // Choose gateway based on command
    const gateway = command.paymentMethod === 'paypal' 
      ? this.paypalGateway 
      : this.stripeGateway;
      
    // Rest of logic unchanged
  }
}

// New adapter
class PayPalPaymentGateway implements PaymentGateway {
  async charge(amount: Money, customer: Email) {
    // PayPal implementation
  }
}
```

**Effort:** 1 week to implement PayPal adapter  
**Risk:** Low (can test thoroughly before enabling)

### Year 5: Add Event-Driven Processing

**New requirement:** Process orders from event queue for better scalability

**Changes needed:**
```elixir
# Application Core: NO CHANGES
# Just add new primary adapter

# New adapter
defmodule Shop.Adapters.Primary.Events.OrderEventHandler do
  @place_order_use_case Application.compile_env(:shop, :place_order_use_case)
  
  def handle_event(%{type: "order.placed"} = event) do
    command = PlaceOrderCommand.from_event(event)
    @place_order_use_case.execute(command)
  end
end
```

**Effort:** 3 days to add event adapter  
**Risk:** Low (runs alongside existing adapters)

### Year 6: Support for Different Databases per Environment

**New requirement:**
- Production: PostgreSQL
- Development: MongoDB (for faster local setup)
- Testing: In-memory

**Changes needed:**
```elixir
# Application Core: NO CHANGES
# Just configure different adapters per environment

# config/prod.exs
config :shop, :order_repository, Shop.Adapters.PostgresOrderRepository

# config/dev.exs
config :shop, :order_repository, Shop.Adapters.MongoOrderRepository

# config/test.exs
config :shop, :order_repository, Shop.Adapters.InMemoryOrderRepository

# New adapter
defmodule Shop.Adapters.Secondary.Persistence.MongoOrderRepository do
  @behaviour Shop.Domain.Ports.Secondary.OrderRepository
  
  # MongoDB implementation
end
```

**Effort:** 2 weeks to implement MongoDB adapter  
**Risk:** Low (test thoroughly in dev before prod)

### Summary: 6 Years of Evolution

**Total major changes:**
- Added 3 primary adapters (GraphQL, CLI, Events)
- Added 2 payment gateway adapters (PayPal, plus original Stripe)
- Added 2 database adapters (MongoDB, plus original PostgreSQL)
- Plus testing adapters for all secondary ports

**Application Core changes:** Minimal (mostly configuration)

**Risk during evolution:** Low at each step

**Key insight:** Because the application core only depends on ports (interfaces), you can add/change/remove adapters without fear. Each change is isolated and testable.

**Without Hexagonal Architecture:**
- Each change would require touching multiple files
- High risk of breaking existing functionality
- Expensive regression testing
- Fear of making changes
- Probably would have rewritten the system 2-3 times

**With Hexagonal Architecture:**
- Each change adds a new adapter
- Existing functionality untouched
- New adapter can be tested in isolation
- Confidence in making changes
- Same core logic running for 6 years

This is the real value of Hexagonal Architecture: **enabling evolution without rewrites**.

## Before/After Comparison

### Before: Tightly Coupled

```typescript
// ❌ Everything coupled together
class OrderService {
  async placeOrder(ebookId: string, email: string, quantity: number) {
    // Tightly coupled to Stripe
    const stripe = new Stripe(process.env.STRIPE_KEY);
    
    // Tightly coupled to PostgreSQL
    const ebook = await db.query('SELECT * FROM ebooks WHERE id = $1', [ebookId]);
    
    // Business logic mixed with infrastructure
    const total = ebook.price * quantity;
    const charge = await stripe.charges.create({
      amount: total * 100,
      currency: 'usd'
    });
    
    // More tight coupling
    await db.query('INSERT INTO orders...', []);
    await sendgrid.send({ to: email, subject: 'Order Confirmation' });
  }
}

// Can only use from one place
// Hard to test
// Can't swap implementations
```

### After: Hexagonal Architecture

```typescript
// ✅ Application Core (domain + ports)
interface PlaceOrderUseCase {
  execute(command: PlaceOrderCommand): Promise<Result<OrderId, Error>>;
}

interface OrderRepository {
  save(order: Order): Promise<Result<OrderId, Error>>;
}

interface PaymentGateway {
  charge(amount: Money, customer: Email): Promise<Result<PaymentId, Error>>;
}

// Application Service (implements primary port)
class PlaceOrderService implements PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,     // Secondary port
    private readonly paymentGateway: PaymentGateway  // Secondary port
  ) {}

  async execute(command: PlaceOrderCommand): Promise<Result<OrderId, Error>> {
    const order = Order.create(command);
    await this.paymentGateway.charge(order.calculateTotal(), order.customerEmail);
    return this.orderRepo.save(order);
  }
}

// Multiple adapters can use it
const service = new PlaceOrderService(
  new PostgresOrderRepository(),
  new StripePaymentGateway()
);

// Or with different adapters
const testService = new PlaceOrderService(
  new InMemoryOrderRepository(),
  new FakePaymentGateway()
);

// React calls it
function OrderForm() {
  const useCase = usePlaceOrderUseCase();
  await useCase.execute(command);
}

// CLI calls it
function cliMain() {
  const useCase = getPlaceOrderUseCase();
  await useCase.execute(command);
}

// GraphQL calls it
function resolver() {
  const useCase = getPlaceOrderUseCase();
  await useCase.execute(command);
}
```

**Benefits:**
- ✅ **No knowledge of infrastructure**: Service doesn't know about Stripe, PostgreSQL, or SendGrid
- ✅ **Easy to test**: Use in-memory adapters
- ✅ **Easy to swap**: Change from Stripe to PayPal by changing one line
- ✅ **Multiple interfaces**: REST, CLI, GraphQL all use the same port
- ✅ **Clear boundaries**: Ports make dependencies explicit

## Dependency Inversion

Hexagonal Architecture relies heavily on the Dependency Inversion Principle.

### The Dependency Flow

```
┌─────────────────┐                    ┌─────────────────┐
│ REST Controller │                    │ PostgreSQL Repo │
│ (Primary        │                    │ (Secondary      │
│  Adapter)       │                    │  Adapter)       │
└────────┬────────┘                    └────────▲────────┘
         │                                      │
         │ depends on                           │ implements
         ▼                                      │
┌─────────────────┐                    ┌───────┴─────────┐
│ PlaceOrderUse   │────depends on─────▶│ OrderRepository │
│ Case (Primary   │                    │ (Secondary      │
│ Port/Interface) │                    │  Port/Interface)│
└────────▲────────┘                    └─────────────────┘
         │
         │ implements
         │
┌────────┴────────┐
│ PlaceOrder      │
│ Service         │
│ (Application    │
│  Core)          │
└─────────────────┘
```

**All arrows point INWARD to the application core.**

### Configuring Adapters

**TypeScript (Composition Root):**
```typescript
// composition/ServiceFactory.ts
export class ServiceFactory {
  static createPlaceOrderUseCase(): PlaceOrderUseCase {
    // Wire up secondary adapters
    const orderRepo: OrderRepository = new PostgresOrderRepository(db);
    const paymentGateway: PaymentGateway = new StripePaymentGateway(apiKey);
    const ebookRepo: EbookRepository = new PostgresEbookRepository(db);

    // Return primary port implementation
    return new PlaceOrderService(orderRepo, paymentGateway, ebookRepo);
  }
}
```

**Elixir (Configuration):**
```elixir
# config/config.exs
config :shop,
  place_order_use_case: Shop.Application.Services.PlaceOrderService,
  order_repository: Shop.Adapters.Secondary.Persistence.PostgresOrderRepository,
  payment_gateway: Shop.Adapters.Secondary.Payment.StripePaymentGateway
```

## Common Pitfalls

### ❌ Pitfall 1: Too Many Ports

```typescript
// ❌ BAD: One port per method (port explosion)
interface SaveOrderPort {
  save(order: Order): Promise<void>;
}

interface FindOrderByIdPort {
  findById(id: string): Promise<Order>;
}

interface FindOrderByEmailPort {
  findByEmail(email: string): Promise<Order[]>;
}

interface DeleteOrderPort {
  delete(id: string): Promise<void>;
}

// Service needs 4 different ports!
class PlaceOrderService {
  constructor(
    private savePort: SaveOrderPort,
    private findByIdPort: FindOrderByIdPort,
    private findByEmailPort: FindOrderByEmailPort,
    private deletePort: DeleteOrderPort
  ) {}
}
```

```typescript
// ✅ GOOD: Group related operations in one port
interface OrderRepository {
  save(order: Order): Promise<Result<OrderId, Error>>;
  findById(id: OrderId): Promise<Result<Order, Error>>;
  findByEmail(email: string): Promise<Result<Order[], Error>>;
  delete(id: OrderId): Promise<Result<void, Error>>;
}

// Service needs just one port
class PlaceOrderService {
  constructor(private orderRepo: OrderRepository) {}
}
```

### ❌ Pitfall 2: Leaky Abstractions

```typescript
// ❌ BAD: Port exposes adapter details (PostgreSQL specific)
interface OrderRepository {
  executeQuery(sql: string, params: any[]): Promise<QueryResult>;
  beginTransaction(): Promise<Transaction>;
  commit(tx: Transaction): Promise<void>;
}

// Now every adapter must deal with SQL transactions!
```

```typescript
// ✅ GOOD: Port defines what app needs, not how adapters provide it
interface OrderRepository {
  save(order: Order): Promise<Result<OrderId, Error>>;
  findById(id: OrderId): Promise<Result<Order, Error>>;
}

// Adapter handles its own transaction logic internally
class PostgresOrderRepository implements OrderRepository {
  async save(order: Order): Promise<Result<OrderId, Error>> {
    // Transaction handling is internal to this adapter
    return this.db.transaction(async (tx) => {
      // Implementation details hidden
    });
  }
}
```

### ❌ Pitfall 3: Ports in Wrong Place

```typescript
// ❌ BAD: Adapter defines the port
// infrastructure/PostgresOrderRepository.ts
export interface OrderRepository { /* ... */ }
export class PostgresOrderRepository implements OrderRepository { /* ... */ }

// Now domain depends on infrastructure!
```

```typescript
// ✅ GOOD: Domain defines the port
// domain/ports/secondary/OrderRepository.ts
export interface OrderRepository { /* ... */ }

// infrastructure/PostgresOrderRepository.ts
import { OrderRepository } from '../domain/ports/secondary/OrderRepository';
export class PostgresOrderRepository implements OrderRepository { /* ... */ }

// Infrastructure depends on domain, not vice versa
```

### ❌ Pitfall 4: Over-Engineering

```typescript
// ❌ BAD: Hexagonal architecture for a simple CRUD operation
// Just need: GET /users/:id

interface GetUserUseCase {
  execute(id: string): Promise<Result<User, Error>>;
}

class GetUserService implements GetUserUseCase {
  constructor(private userRepo: UserRepository) {}
  async execute(id: string) {
    return this.userRepo.findById(id);
  }
}

// This adds no value - it's just a pass-through!
```

```typescript
// ✅ GOOD: Simple CRUD doesn't need ports
// Just use the repository directly from the controller
class UserController {
  constructor(private userRepo: UserRepository) {}
  
  async getUser(req, res) {
    const user = await this.userRepo.findById(req.params.id);
    res.json(user);
  }
}

// Use hexagonal architecture when you have actual business logic
```

## When to Use Hexagonal Architecture

### Good Fit ✅

**Multiple User Interfaces:**
- Web application + mobile app + CLI
- REST API + GraphQL + gRPC
- Interactive UI + background jobs + scheduled tasks

**Complex Integrations:**
- Multiple external systems that might change (payment gateways, shipping providers)
- Different implementations for different environments
- Need to support multiple databases or message queues

**Long-Lived Applications:**
- Will be maintained for years
- Requirements will evolve
- Technology will change

**High Testability Requirements:**
- Need fast unit tests without infrastructure
- Need to test business logic in isolation
- Need deterministic tests

### Overkill ❌

**Simple CRUD Applications:**
- Just reading and writing database records
- No complex business rules
- Single interface, single database

**Prototypes and MVPs:**
- Speed > structure
- Unclear requirements
- May throw away the code

**Single Interface + Single Database:**
- Only ever accessed via REST API
- Only ever using PostgreSQL
- No plans to change either

**Scripts and Utilities:**
- One-off data migrations
- Build scripts
- Simple CLI tools

## Integration with Other Patterns

Hexagonal Architecture works well with other patterns you've learned:

**Repository Pattern:**
- Repositories are secondary adapters
- They implement secondary ports

**Command Pattern:**
- Commands cross the primary port boundary
- They carry data from adapters to use cases

**Application Services:**
- Services implement primary ports
- They orchestrate domain and secondary ports

**Domain Models:**
- Live in the application core
- Contain all business logic
- No infrastructure dependencies

## Conclusion

Hexagonal Architecture provides maximum flexibility by making all external dependencies explicit through ports and adapters. Your application defines what it needs (ports), and you plug in implementations (adapters) as needed.

### Key Takeaways

**Core Concepts:**
- **Application Core**: Domain + Use Cases (no dependencies)
- **Ports**: Interfaces for communication (primary for driving, secondary for driven)
- **Adapters**: Implementations (multiple adapters can implement the same port)

**Benefits:**
- ✅ **Complete isolation**: Application has zero infrastructure dependencies
- ✅ **Maximum flexibility**: Swap any adapter without touching application
- ✅ **Multiple interfaces**: Support REST + CLI + GraphQL simultaneously
- ✅ **Easy testing**: Use in-memory adapters for fast tests
- ✅ **Clear boundaries**: Ports make all dependencies explicit

**When to Use:**
- Multiple user interfaces (web + mobile + CLI)
- Complex integrations that might change
- Long-lived applications
- High testability requirements

**When to Skip:**
- Simple CRUD applications
- Prototypes and MVPs
- Single interface + single database
- Scripts and utilities

### Hexagonal vs Layered

Both are excellent patterns. Choose based on your needs:
- **Layered**: Simpler, good for most applications
- **Hexagonal**: More flexible, better for multiple adapters

### Putting It Into Practice

Start by:
1. Identify your primary ports (use cases)
2. Identify your secondary ports (dependencies)
3. Implement ports as interfaces/behaviours
4. Create adapters for each port
5. Wire everything together in composition root

**Remember:** Don't over-engineer. Add ports only when you need the flexibility. Start simple and refactor to hexagonal when you're adding a second adapter for the same port.

---

**Next**: [Chapter 11: Testing Strategies](11-testing-strategies.md)  
**Previous**: [Chapter 9: Layered Architecture](09-layered-architecture.md)

*The best architecture is the one that makes your application easy to change. Hexagonal Architecture gives you that flexibility by making every external dependency pluggable through well-defined ports.* ✨
