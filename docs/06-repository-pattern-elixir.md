# Chapter 6: Repository Pattern in Elixir

## Introduction

The Repository pattern in Elixir takes a functional approach to data persistence, creating a clean boundary between your domain models and database infrastructure. Rather than tightly coupling domain logic to Ecto schemas and queries, repositories act as a **data mapper layer**—translating between the language of your business (domain structs with value objects) and the language of your database (Ecto schemas).

In functional programming, we embrace **explicit transformations** over hidden mutations. The Repository pattern aligns perfectly with this philosophy: repositories are modules of pure functions that fetch data from infrastructure, convert it to domain models, apply business logic, and persist results back through explicit mapping.

### Why Repositories in Elixir?

Without repositories, Elixir applications often mix concerns:
- Phoenix contexts directly use Ecto queries and changesets
- Domain logic gets embedded in changesets or controllers
- Business rules leak into database schemas
- Testing requires database setup for even simple domain logic
- Swapping persistence strategies means changing domain code

Repositories solve these problems by:
- **Separating concerns**: Domain logic stays pure, infrastructure stays isolated
- **Enabling testability**: In-memory repositories make tests fast and focused
- **Supporting flexibility**: Change database strategies without touching domain
- **Clarifying boundaries**: Clear contract between domain and infrastructure
- **Preserving purity**: Domain models remain free of database concerns

### The Functional Advantage

In Elixir, repositories leverage:
- **Behaviors**: Define contracts that multiple implementations satisfy
- **Pattern matching**: Elegant conversion between data representations
- **Result tuples**: Explicit error handling with `{:ok, value}` | `{:error, reason}`
- **Configuration**: Runtime selection of repository implementations
- **Immutability**: All transformations return new data structures

Let's explore how to build repositories that make your Elixir applications more maintainable, testable, and aligned with domain-driven design principles.

## Table of Contents

1. [The Problem: Mixed Concerns](#the-problem-mixed-concerns)
2. [Repository Behavior: Defining the Contract](#repository-behavior-defining-the-contract)
3. [Ecto Implementation](#ecto-implementation)
4. [In-Memory Implementation](#in-memory-implementation)
5. [Phoenix Controller Integration](#phoenix-controller-integration)
6. [Configuration-Based Dependency Injection](#configuration-based-dependency-injection)
7. [Domain ↔ Ecto Mapping](#domain--ecto-mapping)
8. [Transaction Handling](#transaction-handling)
9. [Testing with ExUnit](#testing-with-exunit)
10. [Common Pitfalls](#common-pitfalls)
11. [Ecto Schemas vs Domain Models](#ecto-schemas-vs-domain-models)
12. [Phoenix Contexts vs Repositories](#phoenix-contexts-vs-repositories)
13. [Conclusion](#conclusion)

## The Problem: Mixed Concerns

Let's start by examining a typical Phoenix controller that mixes domain logic, database queries, validation, and business rules together:

```elixir
# lib/shop_web/controllers/order_controller.ex - BEFORE
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  
  alias Shop.Repo
  alias Shop.Orders.Order
  alias Shop.Products.Product
  import Ecto.Query

  def create(conn, %{"order" => order_params}) do
    # Email validation in controller
    email = order_params["customer_email"]
    unless email =~ ~r/@/ do
      conn
      |> put_status(:bad_request)
      |> json(%{error: "Invalid email"})
    end

    # Direct Ecto query in controller
    product_id = order_params["product_id"]
    product = Repo.get!(Product, product_id)

    # Business logic in controller
    quantity = order_params["quantity"]
    if product.stock < quantity do
      conn
      |> put_status(:unprocessable_entity)
      |> json(%{error: "Insufficient stock"})
    end

    # Calculate total inline
    total = product.price * quantity

    # Create order with Ecto.Changeset
    changeset = Order.changeset(%Order{}, %{
      customer_email: email,
      product_id: product_id,
      quantity: quantity,
      total: total,
      status: "pending"
    })

    case Repo.insert(changeset) do
      {:ok, order} ->
        # Update product stock inline
        product
        |> Ecto.Changeset.change(stock: product.stock - quantity)
        |> Repo.update!()

        # Send notification inline
        send_confirmation_email(order)

        conn
        |> put_status(:created)
        |> json(%{order_id: order.id})

      {:error, changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{errors: translate_errors(changeset)})
    end
  end
  
  defp send_confirmation_email(order) do
    # Email logic...
  end
end
```

**Problems with this approach:**

1. **Mixed responsibilities**: HTTP, validation, business logic, and database operations all in one place
2. **Hard to test**: Requires database setup to test any logic
3. **Tight coupling**: Controller knows about Ecto schemas, changesets, and queries
4. **No domain model**: Business rules are scattered, not encapsulated
5. **Difficult to reuse**: Can't use this logic from CLI, background jobs, or tests
6. **Business logic leakage**: Stock checking and total calculation are business rules hidden in controller
7. **Transaction issues**: Stock update and order creation aren't atomic

## Repository Behavior: Defining the Contract

In Elixir, we use behaviors (similar to interfaces) to define repository contracts. This allows multiple implementations to satisfy the same contract.

### Order Repository Behavior

```elixir
# lib/shop/domain/order_repository.ex
defmodule Shop.Domain.OrderRepository do
  @moduledoc """
  Repository behavior for Order persistence operations.
  
  This defines the contract that all Order repository implementations must satisfy.
  Implementations can use Ecto, GenServer, Agent, or any other storage mechanism.
  """

  alias Shop.Domain.Order

  @type order_id :: String.t()
  @type error_reason :: :not_found | :invalid_order | term()

  @doc """
  Save a new order or update an existing one.
  Returns {:ok, order} with the persisted order, or {:error, reason}.
  """
  @callback save(Order.t()) :: {:ok, Order.t()} | {:error, error_reason()}

  @doc """
  Find an order by its unique identifier.
  Returns {:ok, order} if found, or {:error, :not_found} if not.
  """
  @callback find_by_id(order_id()) :: {:ok, Order.t()} | {:error, :not_found}

  @doc """
  Find all orders for a specific customer.
  Returns {:ok, orders} with a list of orders, or {:error, reason}.
  """
  @callback find_by_customer(String.t()) :: {:ok, [Order.t()]} | {:error, error_reason()}

  @doc """
  Find all orders with a specific status.
  Returns {:ok, orders} with a list of orders.
  """
  @callback find_by_status(Order.status()) :: {:ok, [Order.t()]} | {:error, error_reason()}

  @doc """
  Delete an order by its ID.
  Returns :ok if deleted, or {:error, reason} if not.
  """
  @callback delete(order_id()) :: :ok | {:error, error_reason()}
end
```

**Key principles:**

- **Callbacks define contract**: Each `@callback` specifies expected function signatures
- **Return domain models**: Methods return `Order.t()` domain structs, not Ecto schemas
- **Consistent error handling**: All methods use tagged tuples `{:ok, value}` or `{:error, reason}`
- **Type specifications**: Clear documentation of expected types
- **Implementation agnostic**: No mention of Ecto, database, or infrastructure

## Ecto Implementation

Now let's implement the repository using Ecto for database persistence. The key is **explicit mapping** between Ecto schemas and domain models.

### Step 1: Ecto Schema

First, define an Ecto schema that represents the database structure:

```elixir
# lib/shop/infrastructure/schemas/order_schema.ex
defmodule Shop.Infrastructure.OrderSchema do
  @moduledoc """
  Ecto schema for orders table.
  This is the database representation—NOT the domain model.
  """
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  schema "orders" do
    field :customer_email, :string
    field :order_date, :utc_datetime
    field :status, :string
    
    # JSON field for storing items
    field :items, {:array, :map}
    
    timestamps()
  end

  @doc false
  def changeset(order, attrs) do
    order
    |> cast(attrs, [:customer_email, :order_date, :status, :items])
    |> validate_required([:customer_email, :order_date, :status, :items])
    |> validate_format(:customer_email, ~r/@/)
  end
end
```

### Step 2: Domain Model (Review from Chapter 4)

Recall our domain model from Chapter 4:

```elixir
# lib/shop/domain/order.ex
defmodule Shop.Domain.Order do
  @moduledoc """
  Order aggregate root.
  Represents a customer order with items and business rules.
  """
  
  alias Shop.Domain.{OrderItem, Email, Money, Quantity}

  @enforce_keys [:id, :customer_email, :order_date]
  defstruct [
    :id,
    :customer_email,
    :order_date,
    items: [],
    status: :pending
  ]

  @type status :: :pending | :confirmed | :shipped | :delivered | :cancelled
  
  @type t :: %__MODULE__{
    id: String.t(),
    customer_email: Email.t(),
    order_date: DateTime.t(),
    items: [OrderItem.t()],
    status: status()
  }

  # Constructor and business methods...
  @spec new(String.t(), Email.t()) :: {:ok, t()}
  def new(id, %Email{} = customer_email) do
    order = %__MODULE__{
      id: id,
      customer_email: customer_email,
      order_date: DateTime.utc_now(),
      items: [],
      status: :pending
    }
    {:ok, order}
  end

  # Business logic for adding items, calculating total, etc.
  @spec add_item(t(), String.t(), Money.t(), Quantity.t()) :: 
    {:ok, t()} | {:error, atom()}
  def add_item(%__MODULE__{status: :pending} = order, product_id, unit_price, quantity) do
    with {:ok, item} <- OrderItem.new(product_id, unit_price, quantity) do
      {:ok, %{order | items: [item | order.items]}}
    end
  end
  
  @spec calculate_total(t()) :: Money.t()
  def calculate_total(%__MODULE__{items: items}) do
    Enum.reduce(items, Money.zero(:USD), fn item, acc ->
      subtotal = OrderItem.subtotal(item)
      {:ok, new_total} = Money.add(acc, subtotal)
      new_total
    end)
  end
end
```

### Step 3: Ecto Repository Implementation

Now implement the repository with explicit domain ↔ Ecto mapping:

```elixir
# lib/shop/infrastructure/ecto_order_repository.ex
defmodule Shop.Infrastructure.EctoOrderRepository do
  @moduledoc """
  Ecto-based implementation of OrderRepository.
  Handles persistence and conversion between domain models and Ecto schemas.
  """
  
  @behaviour Shop.Domain.OrderRepository
  
  alias Shop.Repo
  alias Shop.Domain.{Order, OrderItem, Email, Money, Quantity}
  alias Shop.Infrastructure.OrderSchema
  import Ecto.Query

  @impl true
  def save(%Order{} = order) do
    schema = to_schema(order)
    
    case Repo.get(OrderSchema, order.id) do
      nil ->
        # Insert new order
        schema
        |> OrderSchema.changeset(%{})
        |> Repo.insert()
        |> handle_result()
        
      existing ->
        # Update existing order
        existing
        |> OrderSchema.changeset(schema_to_map(schema))
        |> Repo.update()
        |> handle_result()
    end
  end

  @impl true
  def find_by_id(id) do
    case Repo.get(OrderSchema, id) do
      nil -> {:error, :not_found}
      schema -> {:ok, to_domain(schema)}
    end
  end

  @impl true
  def find_by_customer(customer_email) do
    query = from o in OrderSchema,
      where: o.customer_email == ^customer_email,
      order_by: [desc: o.order_date]
    
    orders = 
      query
      |> Repo.all()
      |> Enum.map(&to_domain/1)
    
    {:ok, orders}
  end

  @impl true
  def find_by_status(status) do
    status_string = Atom.to_string(status)
    
    query = from o in OrderSchema,
      where: o.status == ^status_string,
      order_by: [desc: o.order_date]
    
    orders = 
      query
      |> Repo.all()
      |> Enum.map(&to_domain/1)
    
    {:ok, orders}
  end

  @impl true
  def delete(id) do
    case Repo.get(OrderSchema, id) do
      nil -> {:error, :not_found}
      schema ->
        case Repo.delete(schema) do
          {:ok, _} -> :ok
          {:error, reason} -> {:error, reason}
        end
    end
  end

  # Private conversion functions

  @doc false
  @spec to_schema(Order.t()) :: OrderSchema.t()
  defp to_schema(%Order{} = order) do
    %OrderSchema{
      id: order.id,
      customer_email: Email.to_string(order.customer_email),
      order_date: order.order_date,
      status: Atom.to_string(order.status),
      items: Enum.map(order.items, &item_to_map/1)
    }
  end

  @doc false
  @spec to_domain(OrderSchema.t()) :: Order.t()
  defp to_domain(%OrderSchema{} = schema) do
    {:ok, email} = Email.new(schema.customer_email)
    
    items = Enum.map(schema.items, &map_to_item/1)
    
    Order.reconstitute(
      schema.id,
      email,
      schema.order_date,
      items,
      String.to_existing_atom(schema.status)
    )
  end

  defp item_to_map(%OrderItem{} = item) do
    %{
      "product_id" => item.product_id,
      "unit_price" => Money.amount(item.unit_price),
      "currency" => Money.currency(item.unit_price),
      "quantity" => Quantity.to_integer(item.quantity)
    }
  end

  defp map_to_item(map) do
    {:ok, unit_price} = Money.new(map["unit_price"], String.to_atom(map["currency"]))
    {:ok, quantity} = Quantity.new(map["quantity"])
    {:ok, item} = OrderItem.new(map["product_id"], unit_price, quantity)
    item
  end

  defp schema_to_map(%OrderSchema{} = schema) do
    Map.take(schema, [:customer_email, :order_date, :status, :items])
  end

  defp handle_result({:ok, schema}), do: {:ok, to_domain(schema)}
  defp handle_result({:error, changeset}), do: {:error, changeset}
end
```

**Key aspects:**

- **`to_schema/1`**: Converts domain `Order` to Ecto `OrderSchema`
- **`to_domain/1`**: Converts Ecto `OrderSchema` to domain `Order`
- **Explicit mapping**: Value objects like `Email`, `Money`, `Quantity` are explicitly converted
- **Handles inserts and updates**: Check if order exists, then insert or update
- **Error handling**: Returns `{:ok, order}` or `{:error, reason}` consistently

## In-Memory Implementation

For testing, we create an Agent-based in-memory repository that implements the same behavior:

```elixir
# lib/shop/infrastructure/in_memory_order_repository.ex
defmodule Shop.Infrastructure.InMemoryOrderRepository do
  @moduledoc """
  In-memory implementation of OrderRepository using Agent.
  Perfect for fast unit tests without database dependencies.
  """
  
  use Agent
  @behaviour Shop.Domain.OrderRepository
  
  alias Shop.Domain.Order

  # Public API

  def start_link(_opts \\ []) do
    Agent.start_link(fn -> %{} end, name: __MODULE__)
  end

  def clear do
    Agent.update(__MODULE__, fn _ -> %{} end)
  end

  @impl true
  def save(%Order{} = order) do
    Agent.update(__MODULE__, fn orders ->
      Map.put(orders, order.id, order)
    end)
    {:ok, order}
  end

  @impl true
  def find_by_id(id) do
    case Agent.get(__MODULE__, fn orders -> Map.get(orders, id) end) do
      nil -> {:error, :not_found}
      order -> {:ok, order}
    end
  end

  @impl true
  def find_by_customer(customer_email_string) do
    orders = Agent.get(__MODULE__, fn orders ->
      orders
      |> Map.values()
      |> Enum.filter(fn order ->
        Email.to_string(order.customer_email) == customer_email_string
      end)
      |> Enum.sort_by(& &1.order_date, {:desc, DateTime})
    end)
    
    {:ok, orders}
  end

  @impl true
  def find_by_status(status) do
    orders = Agent.get(__MODULE__, fn orders ->
      orders
      |> Map.values()
      |> Enum.filter(& &1.status == status)
      |> Enum.sort_by(& &1.order_date, {:desc, DateTime})
    end)
    
    {:ok, orders}
  end

  @impl true
  def delete(id) do
    result = Agent.get_and_update(__MODULE__, fn orders ->
      case Map.has_key?(orders, id) do
        true -> {:ok, Map.delete(orders, id)}
        false -> {{:error, :not_found}, orders}
      end
    end)
    
    case result do
      :ok -> :ok
      {:error, reason} -> {:error, reason}
    end
  end

  # Helper for tests
  def count do
    Agent.get(__MODULE__, fn orders -> map_size(orders) end)
  end
  
  def all do
    Agent.get(__MODULE__, fn orders -> Map.values(orders) end)
  end
end
```

**Benefits:**

- **Fast**: No database I/O, runs in memory
- **Isolated**: Each test can clear the repository
- **Simple**: Uses Agent for state management
- **Same behavior**: Implements same contract as Ecto version
- **Helper methods**: `clear/0`, `count/0`, `all/0` for test setup

## Phoenix Controller Integration

With repositories in place, controllers become thin HTTP adapters:

### Before: Controller with Mixed Concerns

We saw this earlier—controllers mixing validation, queries, business logic, etc.

### After: Clean Controller Using Repository

```elixir
# lib/shop_web/controllers/order_controller.ex - AFTER
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  
  alias Shop.Application.PlaceOrderCommand
  alias Shop.Application.PlaceOrderService

  @doc """
  POST /api/orders
  
  Place a new order using the application service.
  Controller only handles HTTP concerns.
  """
  def create(conn, %{"order" => order_params}) do
    # 1. Parse command from HTTP params
    command = %PlaceOrderCommand{
      customer_email: order_params["customer_email"],
      product_id: order_params["product_id"],
      quantity: order_params["quantity"]
    }

    # 2. Execute use case via application service
    case PlaceOrderService.execute(command) do
      {:ok, order_id} ->
        conn
        |> put_status(:created)
        |> put_resp_header("location", ~p"/api/orders/#{order_id}")
        |> json(%{order_id: order_id})

      {:error, :product_not_found} ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Product not found"})

      {:error, :insufficient_stock} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: "Insufficient stock"})

      {:error, :invalid_email} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Invalid email address"})

      {:error, reason} ->
        conn
        |> put_status(:internal_server_error)
        |> json(%{error: inspect(reason)})
    end
  end

  @doc """
  GET /api/orders/:id
  """
  def show(conn, %{"id" => id}) do
    order_repo = get_order_repo()
    
    case order_repo.find_by_id(id) do
      {:ok, order} ->
        conn
        |> json(%{
          id: order.id,
          customer_email: Email.to_string(order.customer_email),
          status: order.status,
          total: Money.amount(Order.calculate_total(order)),
          items: Enum.map(order.items, &format_item/1)
        })

      {:error, :not_found} ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Order not found"})
    end
  end

  defp get_order_repo do
    Application.get_env(:shop, :order_repository)
  end

  defp format_item(item) do
    %{
      product_id: item.product_id,
      quantity: Quantity.to_integer(item.quantity),
      unit_price: Money.amount(item.unit_price),
      subtotal: Money.amount(OrderItem.subtotal(item))
    }
  end
end
```

**What improved:**

- ✅ **Thin controller**: Only HTTP translation, no business logic
- ✅ **Clear separation**: Controller → Service → Domain → Repository
- ✅ **Testable**: Can test controller with mock repository
- ✅ **Reusable**: Business logic lives in service, not controller
- ✅ **Configurable**: Repository selected via configuration

## Configuration-Based Dependency Injection

Elixir's configuration system provides elegant dependency injection:

### Application Config

```elixir
# config/config.exs
import Config

config :shop,
  order_repository: Shop.Infrastructure.EctoOrderRepository

# Other environments inherit this
```

### Test Config

```elixir
# config/test.exs
import Config

# Use in-memory repository for fast tests
config :shop,
  order_repository: Shop.Infrastructure.InMemoryOrderRepository

# Use sandbox mode for Ecto when needed
config :shop, Shop.Repo,
  pool: Ecto.Adapters.SQL.Sandbox
```

### Retrieving Repository

```elixir
# In application service or controller
defmodule Shop.Application.PlaceOrderService do
  @order_repo Application.compile_env(:shop, :order_repository)

  def execute(command) do
    # Use @order_repo throughout
    @order_repo.save(order)
  end
end

# Or at runtime (more flexible but slightly slower)
defp get_order_repo do
  Application.get_env(:shop, :order_repository)
end
```

### Application Supervisor

For Agent-based repositories, start them in your supervision tree:

```elixir
# lib/shop/application.ex
defmodule Shop.Application do
  use Application

  def start(_type, _args) do
    children = [
      Shop.Repo,
      ShopWeb.Endpoint,
      # Start in-memory repository if configured
      {Shop.Infrastructure.InMemoryOrderRepository, []}
    ]

    opts = [strategy: :one_for_one, name: Shop.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

**Benefits:**

- **Environment-specific**: Different repositories for dev/test/prod
- **Compile-time safety**: `Application.compile_env/2` checks at compile time
- **Easy testing**: Automatically use in-memory repository in tests
- **Flexible**: Can swap implementations without code changes

## Domain ↔ Ecto Mapping

The heart of the repository pattern is explicit mapping. Let's see a complete example:

### Domain Model with Value Objects

```elixir
# Domain model uses rich value objects
%Order{
  id: "order-123",
  customer_email: %Email{address: "customer@example.com"},
  items: [
    %OrderItem{
      product_id: "prod-1",
      unit_price: %Money{amount: 2999, currency: :USD},
      quantity: %Quantity{value: 2}
    }
  ],
  status: :pending,
  order_date: ~U[2024-01-15 10:30:00Z]
}
```

### Ecto Schema (Database Representation)

```elixir
# Ecto schema uses primitives
%OrderSchema{
  id: "order-123",
  customer_email: "customer@example.com",  # String, not Email
  items: [
    %{
      "product_id" => "prod-1",
      "unit_price" => 2999,     # Integer, not Money
      "currency" => "USD",
      "quantity" => 2           # Integer, not Quantity
    }
  ],
  status: "pending",            # String, not atom
  order_date: ~U[2024-01-15 10:30:00Z]
}
```

### Conversion: Domain → Ecto

```elixir
defp to_schema(%Order{} = order) do
  %OrderSchema{
    id: order.id,
    # Convert Email value object to string
    customer_email: Email.to_string(order.customer_email),
    order_date: order.order_date,
    # Convert status atom to string
    status: Atom.to_string(order.status),
    # Convert each OrderItem to a map
    items: Enum.map(order.items, fn item ->
      %{
        "product_id" => item.product_id,
        "unit_price" => Money.amount(item.unit_price),
        "currency" => Atom.to_string(Money.currency(item.unit_price)),
        "quantity" => Quantity.to_integer(item.quantity)
      }
    end)
  }
end
```

### Conversion: Ecto → Domain

```elixir
defp to_domain(%OrderSchema{} = schema) do
  # Reconstruct Email value object
  {:ok, email} = Email.new(schema.customer_email)
  
  # Convert each item map to OrderItem
  items = Enum.map(schema.items, fn item_map ->
    {:ok, unit_price} = Money.new(
      item_map["unit_price"],
      String.to_existing_atom(item_map["currency"])
    )
    {:ok, quantity} = Quantity.new(item_map["quantity"])
    {:ok, item} = OrderItem.new(
      item_map["product_id"],
      unit_price,
      quantity
    )
    item
  end)
  
  # Reconstitute the domain model
  Order.reconstitute(
    schema.id,
    email,
    schema.order_date,
    items,
    String.to_existing_atom(schema.status)
  )
end
```

**Key points:**

- **Bidirectional**: Domain → Ecto for saving, Ecto → Domain for loading
- **Explicit**: Each transformation is visible and intentional
- **Value objects**: Rich domain types (Email, Money, Quantity) vs primitives
- **Validation**: Value object constructors validate during reconstitution
- **Fail fast**: Errors in conversion are caught early

## Transaction Handling

Repositories fit naturally into transaction boundaries defined by application services:

### Simple Transaction

```elixir
defmodule Shop.Application.PlaceOrderService do
  alias Shop.Repo
  alias Shop.Domain.{Order, OrderRepository, ProductRepository}

  def execute(command) do
    Repo.transaction(fn ->
      with {:ok, product} <- ProductRepository.find_by_id(command.product_id),
           {:ok, order} <- create_order(command, product),
           {:ok, saved_order} <- OrderRepository.save(order),
           :ok <- reserve_inventory(product.id, order.quantity) do
        saved_order.id
      else
        {:error, reason} -> Repo.rollback(reason)
      end
    end)
  end

  defp create_order(command, product) do
    {:ok, email} = Email.new(command.customer_email)
    {:ok, quantity} = Quantity.new(command.quantity)
    
    with {:ok, order} <- Order.new(order_id(), email),
         {:ok, order} <- Order.add_item(order, product.id, product.price, quantity) do
      {:ok, order}
    end
  end

  defp reserve_inventory(product_id, quantity) do
    # Update inventory within transaction
    :ok
  end

  defp order_id, do: Ecto.UUID.generate()
end
```

### Complex Transaction with Multiple Repositories

```elixir
defmodule Shop.Application.CheckoutService do
  alias Shop.Repo
  alias Ecto.Multi

  def execute(command) do
    Multi.new()
    |> Multi.run(:fetch_product, fn _repo, _changes ->
      ProductRepository.find_by_id(command.product_id)
    end)
    |> Multi.run(:create_order, fn _repo, %{fetch_product: product} ->
      create_and_save_order(command, product)
    end)
    |> Multi.run(:reserve_inventory, fn _repo, %{create_order: order} ->
      InventoryRepository.reserve(command.product_id, order.quantity)
    end)
    |> Multi.run(:process_payment, fn _repo, %{create_order: order} ->
      PaymentRepository.charge(command.payment_method, order.total)
    end)
    |> Multi.run(:send_notification, fn _repo, %{create_order: order} ->
      # Fire and forget - already committed
      Task.start(fn -> NotificationService.send_confirmation(order) end)
      {:ok, :sent}
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{create_order: order}} -> {:ok, order.id}
      {:error, _step, reason, _changes} -> {:error, reason}
    end
  end
end
```

**Best practices:**

- **Define boundaries**: Transactions in application services, not repositories
- **Atomic operations**: All database changes succeed or fail together
- **Side effects outside**: Send emails/events after transaction commits
- **Error handling**: Use `Repo.rollback/1` to abort transaction
- **Ecto.Multi**: For complex multi-step workflows

## Testing with ExUnit

Repositories make testing dramatically easier:

### Testing with In-Memory Repository

```elixir
# test/shop/application/place_order_service_test.exs
defmodule Shop.Application.PlaceOrderServiceTest do
  use ExUnit.Case, async: true

  alias Shop.Application.{PlaceOrderService, PlaceOrderCommand}
  alias Shop.Infrastructure.InMemoryOrderRepository
  alias Shop.Domain.{Order, Email, Money}

  setup do
    # Start in-memory repository
    {:ok, _pid} = InMemoryOrderRepository.start_link()
    InMemoryOrderRepository.clear()
    :ok
  end

  describe "execute/1" do
    test "successfully places an order" do
      command = %PlaceOrderCommand{
        customer_email: "customer@example.com",
        product_id: "prod-123",
        quantity: 2
      }

      assert {:ok, order_id} = PlaceOrderService.execute(command)
      assert is_binary(order_id)

      # Verify order was saved
      {:ok, order} = InMemoryOrderRepository.find_by_id(order_id)
      assert Email.to_string(order.customer_email) == "customer@example.com"
      assert length(order.items) == 1
    end

    test "fails with invalid email" do
      command = %PlaceOrderCommand{
        customer_email: "invalid",
        product_id: "prod-123",
        quantity: 2
      }

      assert {:error, :invalid_email} = PlaceOrderService.execute(command)
      
      # Verify no order was created
      assert InMemoryOrderRepository.count() == 0
    end

    test "fails with zero quantity" do
      command = %PlaceOrderCommand{
        customer_email: "customer@example.com",
        product_id: "prod-123",
        quantity: 0
      }

      assert {:error, :invalid_quantity} = PlaceOrderService.execute(command)
    end
  end
end
```

### Testing Repository Implementation

```elixir
# test/shop/infrastructure/ecto_order_repository_test.exs
defmodule Shop.Infrastructure.EctoOrderRepositoryTest do
  use Shop.DataCase, async: true

  alias Shop.Infrastructure.EctoOrderRepository
  alias Shop.Domain.{Order, Email, Money, Quantity, OrderItem}

  describe "save/1 and find_by_id/1" do
    test "saves and retrieves order correctly" do
      # Create domain model
      {:ok, email} = Email.new("customer@example.com")
      {:ok, order} = Order.new("order-123", email)
      {:ok, price} = Money.new(2999, :USD)
      {:ok, quantity} = Quantity.new(2)
      {:ok, order} = Order.add_item(order, "prod-1", price, quantity)

      # Save via repository
      assert {:ok, saved_order} = EctoOrderRepository.save(order)
      assert saved_order.id == "order-123"

      # Retrieve via repository
      assert {:ok, retrieved_order} = EctoOrderRepository.find_by_id("order-123")
      assert retrieved_order.id == "order-123"
      assert Email.to_string(retrieved_order.customer_email) == "customer@example.com"
      assert length(retrieved_order.items) == 1
    end

    test "returns error when order not found" do
      assert {:error, :not_found} = EctoOrderRepository.find_by_id("nonexistent")
    end

    test "updates existing order" do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, order} = Order.new("order-456", email)
      
      # Save initial order
      {:ok, _} = EctoOrderRepository.save(order)
      
      # Modify and save again
      {:ok, price} = Money.new(1999, :USD)
      {:ok, quantity} = Quantity.new(1)
      {:ok, updated_order} = Order.add_item(order, "prod-2", price, quantity)
      {:ok, _} = EctoOrderRepository.save(updated_order)
      
      # Verify update
      {:ok, retrieved} = EctoOrderRepository.find_by_id("order-456")
      assert length(retrieved.items) == 1
    end
  end

  describe "find_by_customer/1" do
    test "returns all orders for customer" do
      {:ok, email1} = Email.new("customer1@example.com")
      {:ok, email2} = Email.new("customer2@example.com")
      
      {:ok, order1} = Order.new("order-1", email1)
      {:ok, order2} = Order.new("order-2", email1)
      {:ok, order3} = Order.new("order-3", email2)
      
      EctoOrderRepository.save(order1)
      EctoOrderRepository.save(order2)
      EctoOrderRepository.save(order3)
      
      {:ok, customer1_orders} = EctoOrderRepository.find_by_customer("customer1@example.com")
      assert length(customer1_orders) == 2
      
      {:ok, customer2_orders} = EctoOrderRepository.find_by_customer("customer2@example.com")
      assert length(customer2_orders) == 1
    end
  end

  describe "find_by_status/1" do
    test "returns orders with given status" do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, order1} = Order.new("order-1", email)
      {:ok, order2} = Order.new("order-2", email)
      {:ok, order2} = Order.confirm(order2)
      
      EctoOrderRepository.save(order1)
      EctoOrderRepository.save(order2)
      
      {:ok, pending_orders} = EctoOrderRepository.find_by_status(:pending)
      assert length(pending_orders) == 1
      
      {:ok, confirmed_orders} = EctoOrderRepository.find_by_status(:confirmed)
      assert length(confirmed_orders) == 1
    end
  end

  describe "delete/1" do
    test "deletes order successfully" do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, order} = Order.new("order-to-delete", email)
      EctoOrderRepository.save(order)
      
      assert :ok = EctoOrderRepository.delete("order-to-delete")
      assert {:error, :not_found} = EctoOrderRepository.find_by_id("order-to-delete")
    end

    test "returns error when deleting non-existent order" do
      assert {:error, :not_found} = EctoOrderRepository.delete("nonexistent")
    end
  end
end
```

**Testing benefits:**

- **Fast**: In-memory tests run in milliseconds
- **Isolated**: Each test is independent
- **No database setup**: In-memory repository requires no migrations
- **Integration tests**: Test Ecto implementation with real database
- **Comprehensive**: Test all repository operations

## Common Pitfalls

### ❌ Pitfall 1: Using Ecto Schemas as Domain Models

```elixir
# ❌ Bad: Ecto schema is the domain model
defmodule Shop.Order do
  use Ecto.Schema
  import Ecto.Changeset

  schema "orders" do
    field :customer_email, :string
    field :status, :string
    field :total, :decimal
    timestamps()
  end

  # Business logic mixed with database schema
  def confirm(%__MODULE__{status: "pending"} = order) do
    changeset(order, %{status: "confirmed"})
  end
end

# Used directly in controller:
order = Repo.get!(Shop.Order, id)
changeset = Shop.Order.confirm(order)
Repo.update!(changeset)
```

**Problems:**
- Domain logic mixed with persistence concerns
- Can't test without database
- Business rules tied to database representation
- Hard to evolve domain independently

```elixir
# ✅ Good: Separate domain model and Ecto schema
defmodule Shop.Domain.Order do
  defstruct [:id, :customer_email, :status, :items]
  
  def confirm(%__MODULE__{status: :pending} = order) do
    {:ok, %{order | status: :confirmed}}
  end
end

defmodule Shop.Infrastructure.OrderSchema do
  use Ecto.Schema
  schema "orders" do
    field :customer_email, :string
    field :status, :string
    timestamps()
  end
end

# Repository handles conversion
```

### ❌ Pitfall 2: Leaking Changesets to Domain

```elixir
# ❌ Bad: Returning Ecto.Changeset from repository
def save(order) do
  order
  |> to_schema()
  |> OrderSchema.changeset()
  |> Repo.insert()
  # Returns {:ok, schema} or {:error, changeset}
  # Changeset is Ecto concept!
end

# ✅ Good: Return domain model or simple error
def save(order) do
  case order |> to_schema() |> Repo.insert() do
    {:ok, schema} -> {:ok, to_domain(schema)}
    {:error, _changeset} -> {:error, :invalid_order}
  end
end
```

### ❌ Pitfall 3: Business Logic in Repository

```elixir
# ❌ Bad: Business logic in repository
def save(order) do
  # Don't do calculations here!
  total = calculate_order_total(order)
  
  # Don't validate business rules here!
  if total < 0 do
    {:error, :invalid_total}
  else
    order
    |> Map.put(:total, total)
    |> to_schema()
    |> Repo.insert()
  end
end

# ✅ Good: Business logic in domain model
defmodule Order do
  def calculate_total(%__MODULE__{items: items}) do
    # Business logic here
  end
end

def save(order) do
  # Repository just persists
  order |> to_schema() |> Repo.insert()
end
```

### ❌ Pitfall 4: Not Handling Errors Consistently

```elixir
# ❌ Bad: Different error formats
def find_by_id(id) do
  case Repo.get(OrderSchema, id) do
    nil -> nil  # Sometimes returns nil
    schema -> to_domain(schema)  # Sometimes returns domain model
  end
end

def save(order) do
  Repo.insert!(to_schema(order))  # Sometimes raises!
end

# ✅ Good: Consistent tagged tuples
def find_by_id(id) do
  case Repo.get(OrderSchema, id) do
    nil -> {:error, :not_found}
    schema -> {:ok, to_domain(schema)}
  end
end

def save(order) do
  case Repo.insert(to_schema(order)) do
    {:ok, schema} -> {:ok, to_domain(schema)}
    {:error, _changeset} -> {:error, :save_failed}
  end
end
```

### ❌ Pitfall 5: Forgetting to Map Value Objects

```elixir
# ❌ Bad: Storing value object structs directly
defp to_schema(order) do
  %OrderSchema{
    customer_email: order.customer_email  # %Email{} struct stored as-is!
  }
end

# This will fail - Ecto doesn't know how to serialize custom structs

# ✅ Good: Extract primitives from value objects
defp to_schema(order) do
  %OrderSchema{
    customer_email: Email.to_string(order.customer_email)
  }
end
```

### ❌ Pitfall 6: Coupling Tests to Database

```elixir
# ❌ Bad: All tests use Ecto repository
test "order service creates order" do
  # Setup database
  {:ok, product} = Repo.insert(%Product{...})
  
  # Test service
  {:ok, order_id} = OrderService.create(...)
  
  # Query database to verify
  order = Repo.get!(Order, order_id)
end

# ✅ Good: Unit tests use in-memory repository
test "order service creates order" do
  # Setup in-memory data
  InMemoryProductRepo.save(product)
  
  # Test service (no database)
  {:ok, order_id} = OrderService.create(...)
  
  # Verify via repository
  {:ok, order} = InMemoryOrderRepo.find_by_id(order_id)
end
```

## Ecto Schemas vs Domain Models

A common question: When should I separate Ecto schemas from domain models?

### When to Keep Simple (Use Ecto Schemas Directly)

**Use Ecto schemas as domain models when:**

- ✅ Simple CRUD operations with minimal business logic
- ✅ Database structure mirrors domain closely
- ✅ Few or no value objects needed
- ✅ Validation is primarily data format checking
- ✅ Small, simple application

**Example:**

```elixir
# Simple app: Blog posts with basic fields
defmodule Blog.Post do
  use Ecto.Schema
  import Ecto.Changeset

  schema "posts" do
    field :title, :string
    field :content, :text
    field :published, :boolean, default: false
    timestamps()
  end

  def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :content, :published])
    |> validate_required([:title, :content])
  end
  
  # Minimal business logic
  def publish(post), do: change(post, published: true)
end
```

### When to Separate (Use Repository Pattern)

**Separate domain models and Ecto schemas when:**

- ✅ Complex business rules and invariants
- ✅ Rich value objects (Email, Money, Quantity)
- ✅ Domain model structure differs from database
- ✅ Need to test business logic without database
- ✅ Multiple persistence strategies (DB, cache, API)
- ✅ Domain logic will evolve independently

**Example:**

```elixir
# Complex app: E-commerce with rich domain
defmodule Shop.Domain.Order do
  # Rich domain model with value objects
  defstruct [:id, :customer_email, :items, :status]
  
  def add_item(%__MODULE__{status: :pending} = order, product, qty) do
    # Complex business rules
    with :ok <- validate_stock(product, qty),
         {:ok, item} <- OrderItem.new(product.id, product.price, qty) do
      {:ok, %{order | items: [item | order.items]}}
    end
  end
  
  def calculate_total(%__MODULE__{items: items}) do
    # Domain calculation using value objects
  end
end

# Separate Ecto schema for persistence
defmodule Shop.Infrastructure.OrderSchema do
  use Ecto.Schema
  
  schema "orders" do
    field :customer_email, :string
    field :items, {:array, :map}
    field :status, :string
  end
end

# Repository handles conversion
```

### Decision Matrix

| Factor | Use Ecto Directly | Separate with Repository |
|--------|-------------------|-------------------------|
| Business complexity | Low | High |
| Value objects | None/Few | Many |
| Testing needs | DB tests OK | Need fast unit tests |
| Domain evolution | Stable | Frequent changes |
| Multiple storage | No | Yes |
| Team size | Small | Large |

## Phoenix Contexts vs Repositories

Understanding the difference between Phoenix Contexts and Repositories is crucial:

### Phoenix Contexts

**What they are:**
- Application layer boundaries
- Group related functionality
- Expose public API for features
- Orchestrate use cases

**Example:**

```elixir
defmodule Shop.Orders do
  @moduledoc """
  Context for order management.
  Defines the public API for order-related operations.
  """
  
  def create_order(attrs) do
    # Orchestrate the use case
  end
  
  def get_order(id) do
    # Fetch and return
  end
  
  def list_orders_for_customer(customer_id) do
    # Query and return
  end
end
```

### Repositories

**What they are:**
- Infrastructure layer boundaries
- Handle persistence only
- Return domain models
- Hide database details

**Example:**

```elixir
defmodule Shop.Infrastructure.OrderRepository do
  @moduledoc """
  Repository for Order persistence.
  Converts between domain models and database schemas.
  """
  
  def save(order) do
    # Convert and persist
  end
  
  def find_by_id(id) do
    # Fetch and convert to domain
  end
end
```

### How They Work Together

```elixir
# Context (Application Layer)
defmodule Shop.Orders do
  alias Shop.Domain.{Order, OrderRepository}
  alias Shop.Application.PlaceOrderService
  
  @doc """
  Public API - Create a new order
  """
  def create_order(attrs) do
    # Context orchestrates the use case
    PlaceOrderService.execute(attrs)
  end
  
  @doc """
  Public API - Get order by ID
  """
  def get_order(id) do
    # Context delegates to repository
    order_repo().find_by_id(id)
  end
  
  defp order_repo do
    Application.get_env(:shop, :order_repository)
  end
end

# Application Service
defmodule Shop.Application.PlaceOrderService do
  def execute(command) do
    with {:ok, order} <- create_domain_order(command),
         {:ok, saved_order} <- OrderRepository.save(order) do
      {:ok, saved_order.id}
    end
  end
end

# Repository (Infrastructure Layer)
defmodule Shop.Infrastructure.EctoOrderRepository do
  @behaviour Shop.Domain.OrderRepository
  
  def save(order) do
    # Handle persistence details
  end
end
```

### Comparison Table

| Aspect | Phoenix Context | Repository |
|--------|----------------|------------|
| **Layer** | Application | Infrastructure |
| **Purpose** | Orchestrate use cases | Handle persistence |
| **Exposes** | Public API | Data access methods |
| **Contains** | Business workflows | DB conversion logic |
| **Depends on** | Domain + Repositories | Only domain models |
| **Used by** | Controllers, LiveView | Contexts, Services |
| **Returns** | Results of use cases | Domain models |

### Example: Complete Flow

```elixir
# 1. Controller calls Context
defmodule ShopWeb.OrderController do
  def create(conn, params) do
    case Shop.Orders.create_order(params) do
      {:ok, order_id} -> json(conn, %{order_id: order_id})
      {:error, reason} -> json(conn, %{error: reason})
    end
  end
end

# 2. Context calls Application Service
defmodule Shop.Orders do
  def create_order(params) do
    command = PlaceOrderCommand.new(params)
    PlaceOrderService.execute(command)
  end
end

# 3. Service uses Domain Models and Repository
defmodule Shop.Application.PlaceOrderService do
  def execute(command) do
    with {:ok, order} <- Order.new(command),
         {:ok, saved} <- @order_repo.save(order) do
      {:ok, saved.id}
    end
  end
end

# 4. Repository handles persistence
defmodule Shop.Infrastructure.EctoOrderRepository do
  def save(order) do
    order |> to_schema() |> Repo.insert()
  end
end
```

**Key takeaway:** Contexts are about **use case orchestration** (application layer), while repositories are about **data persistence** (infrastructure layer). They complement each other.

## Conclusion

The Repository pattern in Elixir provides a functional approach to data persistence that keeps your domain models pure and your code testable. By creating an explicit boundary between domain logic and infrastructure, you gain flexibility, clarity, and maintainability.

### Key Takeaways

**Repository Pattern in Elixir:**
- ✅ Use behaviors to define repository contracts
- ✅ Implement with Ecto for production, Agent for testing
- ✅ Explicitly map between domain models and Ecto schemas
- ✅ Return domain models with value objects, not schemas
- ✅ Configure repository implementations via config files
- ✅ Define transaction boundaries in application services
- ✅ Keep business logic in domain models, not repositories

**When to Use:**
- Complex domain logic with value objects
- Need fast unit tests without database
- Domain structure differs from database
- Multiple teams working on different layers
- Domain will evolve independently

**When to Skip:**
- Simple CRUD operations
- Database structure matches domain closely
- Small application with minimal business logic
- Tight deadline with limited resources

### Remember

The Repository pattern is not about adding layers for the sake of architecture—it's about **separating concerns** so your domain logic stays clean, testable, and independent of infrastructure decisions.

Start with Ecto schemas if your app is simple. Introduce repositories when you feel pain from mixed concerns, complex business logic, or difficult testing. Refactor incrementally as your application grows.

### What's Next?

Now that you understand repositories in Elixir:

- **Review [Chapter 4: Domain Models in Elixir](04-domain-models-elixir.md)** for rich domain model patterns
- **Explore [Chapter 7: Application Services](07-application-services.md)** to see how services orchestrate repositories
- **Practice**: Refactor one Phoenix context to use repositories
- **Test**: Experience the joy of fast, database-free domain tests

### Further Reading

- [Ecto Documentation](https://hexdocs.pm/ecto/) - Official Ecto guide
- [Domain-Driven Design](https://www.domainlanguage.com/ddd/) by Eric Evans
- [Implementing Domain-Driven Design](https://vaughnvernon.com/) by Vaughn Vernon
- [Designing Elixir Systems with OTP](https://pragprog.com/titles/jgotp/) by James Edward Gray II

---

*This chapter is part of the Web Patterns documentation series. Continue your journey with clean architecture in Elixir!*

**Previous**: [Chapter 5: Repository Pattern in React](05-repository-pattern-react.md)  
**Next**: [Chapter 7: Application Services](07-application-services.md)
