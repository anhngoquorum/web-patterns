# Chapter 13: Refactoring Elixir Code

## Introduction

You've learned clean architecture patterns in Elixir—domain models, repositories, services, and hexagonal architecture. But how do you actually refactor existing Phoenix code to follow these patterns?

This chapter provides a practical, step-by-step guide for transforming messy Phoenix controllers into clean, maintainable architecture. You'll learn an incremental approach that lets you refactor in deployable steps, avoiding the "big bang" rewrite that often fails. We'll work through a realistic example, showing exactly how to extract domain logic, create repositories, build services, and update controllers—all while keeping your application working at every step.


### Why Refactor Elixir/Phoenix to Clean Architecture?

Clean architecture isn't just for object-oriented languages—it's equally powerful in functional languages like Elixir. The combination of **functional programming** and **clean architecture** gives you unique advantages:

**1. Pure functions + domain models = Fearless refactoring**
- Business logic as pure functions is easier to test than OOP methods
- Immutability prevents hidden state bugs
- Pattern matching makes edge cases explicit
- `with` pipelines make complex workflows clear

**2. Elixir's strengths enhance clean architecture**
- Pattern matching replaces conditionals with clear business rules
- Result tuples (`{:ok, value}` / `{:error, reason}`) make error handling explicit
- Behaviours define clear contracts for ports
- Immutability prevents accidental coupling

**3. Phoenix pushes you toward clean separation**
- Contexts already suggest grouping by domain
- LiveView makes testing valuable (test business logic without browser)
- Umbrellaapps encourage clear boundaries
- Ecto separates schemas from domain (when used correctly)

### The Problem with Messy Phoenix Controllers

Before we refactor, we need to understand what we're fixing:

**Typical problems in Phoenix controllers:**
- Validation logic mixed with HTTP concerns
- Business calculations scattered across controllers
- Direct `Repo` calls instead of abstractions
- Ecto schemas used as domain models
- External API calls (Stripe, email) directly in controllers
- Hard to test without full Phoenix setup
- Cannot reuse logic in other contexts (CLI, admin panel, background jobs)

**Common excuses:**
- "Phoenix Contexts already solve this" (they don't—they mix concerns)
- "It's just CRUD, why complicate it?" (until it's not just CRUD anymore)
- "Elixir is functional, we don't need layers" (functional != no architecture)
- "Our code is simple" (until you need to add that one feature...)

### When to Refactor (and When NOT to)

**✅ Refactor when:**
- Business logic is duplicated across multiple controllers or contexts
- Testing requires spinning up the entire Phoenix stack
- The same code causes repeated bugs
- You need to reuse logic in LiveView, GraphQL, and REST simultaneously
- Team velocity is slowing because changes require touching many files
- You're adding similar features and could extract shared patterns

**❌ Don't refactor when:**
- Simple CRUD operations that will stay simple
- Prototypes or proof-of-concept code
- Features scheduled for deletion
- During production incidents
- You don't understand the business domain yet
- Business priorities require shipping features immediately

### The Incremental Approach

The key to successful refactoring is making small, deployable changes:

**Not this:** Rewrite entire Phoenix app over 3 weeks, pray tests pass, big deploy.

**Do this:** Extract one controller this week, deploy, validate, repeat.

**The strategy:**
1. Each refactoring step takes hours, not days
2. Each step is deployable and testable independently
3. Old and new code coexist during transition
4. Risk is minimized at every stage
5. You can stop anytime and have improved code

Let's see this in action with a realistic example.

## The Starting Point: Messy Phoenix Controller

Here's a realistic 150-line Phoenix controller with all the classic problems. This is the kind of code we're refactoring:

```elixir
# lib/shop_web/controllers/order_controller.ex - BEFORE refactoring
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  
  alias Shop.Repo
  alias Shop.Order
  alias Shop.OrderItem
  alias Shop.Product
  
  # 150+ lines of mixed concerns
  def create(conn, %{"email" => email, "items" => items}) do
    # Validation directly in controller
    cond do
      !String.contains?(email, "@") ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Invalid email address"})
      
      email == "" ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Email is required"})
      
      String.length(email) > 255 ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Email too long"})
      
      true ->
        # Business logic in controller
        order_id = "ORD-#{System.unique_integer([:positive])}"
        
        # Direct Ecto insert
        case Repo.insert(%Order{
          id: order_id,
          customer_email: email,
          status: "pending",
          inserted_at: DateTime.utc_now(),
          updated_at: DateTime.utc_now()
        }) do
          {:ok, order} ->
            # More business logic: process items
            case process_order_items(order, items) do
              {:ok, total} ->
                # Calculation in controller
                tax = total * 0.08
                final_total = total + tax
                
                # External API call directly
                case Stripe.charge(
                  amount: round(final_total * 100),
                  currency: "usd",
                  customer_email: email
                ) do
                  {:ok, charge} ->
                    # Update order
                    Repo.update!(Ecto.Changeset.change(order, 
                      stripe_charge_id: charge.id,
                      total: total,
                      tax: tax,
                      final_total: final_total
                    ))
                    
                    # Send email directly
                    send_confirmation_email(email, order_id, final_total)
                    
                    json(conn, %{
                      order_id: order_id,
                      total: final_total,
                      status: "pending"
                    })
                  
                  {:error, _} ->
                    # Rollback is manual and error-prone
                    Repo.delete(order)
                    
                    conn
                    |> put_status(:payment_required)
                    |> json(%{error: "Payment failed"})
                end
              
              {:error, reason} ->
                Repo.delete(order)
                
                conn
                |> put_status(:unprocessable_entity)
                |> json(%{error: reason})
            end
          
          {:error, changeset} ->
            conn
            |> put_status(:unprocessable_entity)
            |> json(%{error: "Failed to create order"})
        end
    end
  end
  
  def confirm(conn, %{"id" => order_id}) do
    # Direct database query
    order = Repo.get(Order, order_id)
    
    cond do
      is_nil(order) ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Order not found"})
      
      # Business rule in controller
      order.status != "pending" ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: "Cannot confirm order - invalid status"})
      
      # More business rules
      order.final_total <= 0 ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: "Cannot confirm order with zero total"})
      
      true ->
        # State transition directly
        case Repo.update(Ecto.Changeset.change(order, 
          status: "confirmed",
          confirmed_at: DateTime.utc_now()
        )) do
          {:ok, updated_order} ->
            # More external calls
            notify_warehouse(updated_order)
            send_confirmation_email(updated_order.customer_email, order_id, updated_order.final_total)
            
            json(conn, %{
              order_id: updated_order.id,
              status: updated_order.status
            })
          
          {:error, _} ->
            conn
            |> put_status(:internal_server_error)
            |> json(%{error: "Failed to confirm order"})
        end
    end
  end
  
  # Helper functions with business logic
  defp process_order_items(order, items) do
    results = Enum.reduce_while(items, {:ok, 0}, fn item, {:ok, acc_total} ->
      # Fetch product
      product = Repo.get(Product, item["product_id"])
      
      cond do
        is_nil(product) ->
          {:halt, {:error, "Product not found: #{item["product_id"]}"}}
        
        # Stock validation in helper
        product.stock_quantity < item["quantity"] ->
          {:halt, {:error, "Insufficient stock for product: #{product.name}"}}
        
        # Price validation in helper
        item["quantity"] <= 0 ->
          {:halt, {:error, "Invalid quantity"}}
        
        true ->
          # Create order item
          Repo.insert!(%OrderItem{
            order_id: order.id,
            product_id: product.id,
            quantity: item["quantity"],
            unit_price: product.price
          })
          
          # Update product stock directly
          Repo.update!(Ecto.Changeset.change(product,
            stock_quantity: product.stock_quantity - item["quantity"]
          ))
          
          # Calculate subtotal
          item_total = product.price * item["quantity"]
          {:cont, {:ok, acc_total + item_total}}
      end
    end)
    
    results
  end
  
  defp send_confirmation_email(email, order_id, total) do
    # Direct email sending
    Shop.Mailer.deliver(
      to: email,
      subject: "Order Confirmation",
      body: "Your order #{order_id} for $#{total} has been confirmed."
    )
  end
  
  defp notify_warehouse(order) do
    # Direct HTTP call
    HTTPoison.post("https://warehouse.example.com/api/orders", 
      Jason.encode!(%{order_id: order.id, items: order.items})
    )
  end
end
```

### What's Wrong with This Code?

Let's count the problems:

**1. Mixed Concerns (7 different responsibilities)**
- ✗ HTTP request/response handling
- ✗ Validation
- ✗ Business calculations (tax, totals)
- ✗ Database operations (Repo.insert, Repo.update)
- ✗ External API calls (Stripe)
- ✗ Email sending
- ✗ Warehouse notifications

**2. Untestable Business Logic**
- Cannot test order confirmation rules without Phoenix
- Cannot test tax calculation without database
- Cannot test without Stripe API
- Cannot test without email service

**3. Ecto Schema Used as Domain Model**
- `Order` schema is just a database mapping
- No business behavior (can't call `Order.confirm/1`)
- Business rules scattered across controllers

**4. No Error Recovery**
- Manual rollback (error-prone)
- No transaction boundaries
- Partial failures leave inconsistent state

**5. Hard to Change**
- Want to change tax rate? Must find all calculations
- Want to add discount logic? Must modify controller
- Want to reuse in GraphQL? Copy-paste all this logic

**6. Not Reusable**
- Cannot use order logic in background jobs
- Cannot use in admin panel without duplicating
- Cannot use in CLI tools

Now let's fix it, step by step.


## The 6-Step Refactoring Strategy

Here's our incremental refactoring plan:

```
Step 1: Extract Domain Models (2-3 hours)
  ↓ Deploy & Test
  
Step 2: Create Repository Behaviour (2-3 hours)
  ↓ Deploy & Test
  
Step 3: Extract Application Service (2-3 hours)
  ↓ Deploy & Test
  
Step 4: Update Controller (2 hours)
  ↓ Deploy & Test
  
Step 5: Add ExUnit Tests (2-3 hours)
  ↓ Deploy & Test
  
Step 6: Iterate & Refine (ongoing)
```

**Key points:**
- Each step is independently deployable
- Old and new code coexist during transition
- You can stop after any step and have improved code
- Total time: 1-2 weeks of incremental work
- Risk: LOW (small changes, tested at each step)

Let's implement each step.

## Step 1: Extract Domain Models

First, we create domain models that encapsulate business logic as pure Elixir modules with structs and functions.

### Order Domain Model

```elixir
# lib/shop/domain/order.ex
defmodule Shop.Domain.Order do
  @moduledoc """
  Order aggregate root with business logic.
  Pure Elixir - no dependencies on Ecto, Phoenix, or external services.
  """
  
  alias Shop.Domain.{Email, Money, OrderItem}
  
  @enforce_keys [:id, :customer_email, :items, :status]
  defstruct [:id, :customer_email, :items, :status, :confirmed_at, :inserted_at]
  
  @type status :: :pending | :confirmed | :shipped | :cancelled
  
  @type t :: %__MODULE__{
    id: String.t(),
    customer_email: Email.t(),
    items: [OrderItem.t()],
    status: status(),
    confirmed_at: DateTime.t() | nil,
    inserted_at: DateTime.t()
  }
  
  @spec new(String.t(), Email.t(), [OrderItem.t()]) :: 
    {:ok, t()} | {:error, atom()}
  def new(id, %Email{} = email, items) when is_list(items) do
    with :ok <- validate_items(items) do
      {:ok, %__MODULE__{
        id: id,
        customer_email: email,
        items: items,
        status: :pending,
        confirmed_at: nil,
        inserted_at: DateTime.utc_now()
      }}
    end
  end
  
  @spec calculate_subtotal(t()) :: Money.t()
  def calculate_subtotal(%__MODULE__{items: items}) do
    Enum.reduce(items, Money.new(0, :USD), fn item, acc ->
      item_total = OrderItem.calculate_total(item)
      Money.add(acc, item_total)
    end)
  end
  
  @spec calculate_tax(t()) :: Money.t()
  def calculate_tax(order) do
    subtotal = calculate_subtotal(order)
    Money.multiply(subtotal, 0.08)  # 8% tax rate
  end
  
  @spec calculate_total(t()) :: Money.t()
  def calculate_total(order) do
    subtotal = calculate_subtotal(order)
    tax = calculate_tax(order)
    Money.add(subtotal, tax)
  end
  
  @spec can_confirm?(t()) :: boolean()
  def can_confirm?(%__MODULE__{status: :pending, items: items}) when length(items) > 0 do
    total = calculate_total(%__MODULE__{status: :pending, items: items, id: "", customer_email: nil, inserted_at: nil})
    Money.positive?(total)
  end
  def can_confirm?(_), do: false
  
  @spec confirm(t()) :: {:ok, t()} | {:error, :invalid_status | :empty_order}
  def confirm(%__MODULE__{status: :pending, items: items} = order) when length(items) > 0 do
    if can_confirm?(order) do
      {:ok, %{order | status: :confirmed, confirmed_at: DateTime.utc_now()}}
    else
      {:error, :invalid_total}
    end
  end
  def confirm(%__MODULE__{status: :pending, items: []}), do: {:error, :empty_order}
  def confirm(_), do: {:error, :invalid_status}
  
  @spec cancel(t()) :: {:ok, t()} | {:error, :invalid_status}
  def cancel(%__MODULE__{status: status} = order) when status in [:pending, :confirmed] do
    {:ok, %{order | status: :cancelled}}
  end
  def cancel(_), do: {:error, :invalid_status}
  
  # Private validation
  defp validate_items([]), do: {:error, :empty_items}
  defp validate_items(_items), do: :ok
end
```

### Email Value Object

```elixir
# lib/shop/domain/email.ex
defmodule Shop.Domain.Email do
  @moduledoc """
  Email value object with validation.
  """
  
  @enforce_keys [:address]
  defstruct [:address]
  
  @type t :: %__MODULE__{address: String.t()}
  
  @email_regex ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/
  
  @spec new(String.t()) :: {:ok, t()} | {:error, :invalid_email}
  def new(address) when is_binary(address) do
    normalized = address |> String.trim() |> String.downcase()
    
    cond do
      byte_size(normalized) == 0 ->
        {:error, :invalid_email}
      
      byte_size(normalized) > 255 ->
        {:error, :invalid_email}
      
      not Regex.match?(@email_regex, normalized) ->
        {:error, :invalid_email}
      
      true ->
        {:ok, %__MODULE__{address: normalized}}
    end
  end
  def new(_), do: {:error, :invalid_email}
  
  @spec to_string(t()) :: String.t()
  def to_string(%__MODULE__{address: address}), do: address
end
```

### Money Value Object

```elixir
# lib/shop/domain/money.ex
defmodule Shop.Domain.Money do
  @moduledoc """
  Money value object representing an amount in cents.
  All operations return new Money instances (immutable).
  """
  
  @enforce_keys [:amount, :currency]
  defstruct [:amount, :currency]
  
  @type t :: %__MODULE__{
    amount: integer(),  # Amount in cents
    currency: atom()
  }
  
  @spec new(integer(), atom()) :: t()
  def new(amount, currency \\ :USD) when is_integer(amount) and is_atom(currency) do
    %__MODULE__{amount: amount, currency: currency}
  end
  
  @spec add(t(), t()) :: t()
  def add(%__MODULE__{amount: a1, currency: c}, %__MODULE__{amount: a2, currency: c}) do
    %__MODULE__{amount: a1 + a2, currency: c}
  end
  
  @spec multiply(t(), float()) :: t()
  def multiply(%__MODULE__{amount: amount, currency: c}, factor) when is_number(factor) do
    %__MODULE__{amount: round(amount * factor), currency: c}
  end
  
  @spec positive?(t()) :: boolean()
  def positive?(%__MODULE__{amount: amount}), do: amount > 0
  
  @spec to_dollars(t()) :: float()
  def to_dollars(%__MODULE__{amount: amount}), do: amount / 100.0
  
  @spec format(t()) :: String.t()
  def format(%__MODULE__{amount: amount, currency: :USD}) do
    "$#{:erlang.float_to_binary(amount / 100.0, decimals: 2)}"
  end
end
```

### OrderItem Value Object

```elixir
# lib/shop/domain/order_item.ex
defmodule Shop.Domain.OrderItem do
  @moduledoc """
  Represents a single line item in an order.
  """
  
  alias Shop.Domain.Money
  
  @enforce_keys [:product_id, :product_name, :unit_price, :quantity]
  defstruct [:product_id, :product_name, :unit_price, :quantity]
  
  @type t :: %__MODULE__{
    product_id: String.t(),
    product_name: String.t(),
    unit_price: Money.t(),
    quantity: pos_integer()
  }
  
  @spec new(String.t(), String.t(), Money.t(), pos_integer()) :: 
    {:ok, t()} | {:error, atom()}
  def new(product_id, product_name, %Money{} = unit_price, quantity) 
    when is_binary(product_id) and is_binary(product_name) and is_integer(quantity) and quantity > 0 do
    
    if Money.positive?(unit_price) do
      {:ok, %__MODULE__{
        product_id: product_id,
        product_name: product_name,
        unit_price: unit_price,
        quantity: quantity
      }}
    else
      {:error, :invalid_price}
    end
  end
  def new(_, _, _, _), do: {:error, :invalid_order_item}
  
  @spec calculate_total(t()) :: Money.t()
  def calculate_total(%__MODULE__{unit_price: price, quantity: qty}) do
    Money.multiply(price, qty)
  end
end
```

### Why This Step Matters

**Before:** Business logic scattered in controller
```elixir
# Calculation buried in controller
tax = total * 0.08
final_total = total + tax

# Validation in controller
if order.status != "pending" do
  {:error, "Cannot confirm"}
end
```

**After:** Business logic in domain model
```elixir
# Self-documenting, testable, reusable
Order.calculate_total(order)
Order.can_confirm?(order)
Order.confirm(order)
```

**Benefits:**
- ✅ Business logic is testable without Phoenix, Ecto, or database
- ✅ Validation is centralized in one place
- ✅ Calculations are consistent everywhere
- ✅ Rules are explicit and type-safe


## Step 2: Create Repository Behaviour

Next, we abstract data access behind a repository behaviour. This separates "what" we persist from "how" we persist it.

### Define Repository Behaviour

```elixir
# lib/shop/domain/order_repository.ex
defmodule Shop.Domain.OrderRepository do
  @moduledoc """
  Repository behaviour for Order aggregate.
  Defines the contract for persisting and retrieving orders.
  """
  
  alias Shop.Domain.Order
  
  @callback save(Order.t()) :: {:ok, Order.t()} | {:error, term()}
  @callback find_by_id(String.t()) :: {:ok, Order.t()} | {:error, :not_found}
  @callback delete(String.t()) :: :ok | {:error, term()}
end
```

### Implement Ecto Repository

```elixir
# lib/shop/infrastructure/ecto_order_repository.ex
defmodule Shop.Infrastructure.EctoOrderRepository do
  @moduledoc """
  Ecto implementation of OrderRepository.
  Maps between domain models and Ecto schemas.
  """
  
  @behaviour Shop.Domain.OrderRepository
  
  alias Shop.Domain.{Order, Email, Money, OrderItem}
  alias Shop.Repo
  alias Shop.Infrastructure.OrderSchema
  
  @impl true
  def save(%Order{} = order) do
    schema = to_schema(order)
    
    case Repo.insert(schema) do
      {:ok, saved_schema} -> 
        {:ok, to_domain(saved_schema)}
      
      {:error, changeset} -> 
        {:error, changeset}
    end
  end
  
  @impl true
  def find_by_id(id) when is_binary(id) do
    case Repo.get(OrderSchema, id) do
      nil -> 
        {:error, :not_found}
      
      schema -> 
        {:ok, to_domain(schema)}
    end
  end
  
  @impl true
  def delete(id) when is_binary(id) do
    case Repo.get(OrderSchema, id) do
      nil -> {:error, :not_found}
      schema -> 
        case Repo.delete(schema) do
          {:ok, _} -> :ok
          {:error, changeset} -> {:error, changeset}
        end
    end
  end
  
  # Explicit domain ↔ Ecto mapping
  defp to_schema(%Order{} = order) do
    %OrderSchema{
      id: order.id,
      customer_email: Email.to_string(order.customer_email),
      status: Atom.to_string(order.status),
      confirmed_at: order.confirmed_at,
      inserted_at: order.inserted_at,
      # Note: items would be handled separately in a real implementation
      items: Enum.map(order.items, &order_item_to_map/1)
    }
  end
  
  defp to_domain(%OrderSchema{} = schema) do
    {:ok, email} = Email.new(schema.customer_email)
    items = Enum.map(schema.items, &map_to_order_item/1)
    
    %Order{
      id: schema.id,
      customer_email: email,
      items: items,
      status: String.to_existing_atom(schema.status),
      confirmed_at: schema.confirmed_at,
      inserted_at: schema.inserted_at
    }
  end
  
  defp order_item_to_map(%OrderItem{} = item) do
    %{
      "product_id" => item.product_id,
      "product_name" => item.product_name,
      "unit_price" => item.unit_price.amount,
      "quantity" => item.quantity
    }
  end
  
  defp map_to_order_item(%{"product_id" => id, "product_name" => name, 
                           "unit_price" => price, "quantity" => qty}) do
    {:ok, item} = OrderItem.new(id, name, Money.new(price, :USD), qty)
    item
  end
end
```

### Ecto Schema (Separate from Domain)

```elixir
# lib/shop/infrastructure/order_schema.ex
defmodule Shop.Infrastructure.OrderSchema do
  @moduledoc """
  Ecto schema for orders table.
  This is ONLY for database mapping, NOT a domain model.
  """
  
  use Ecto.Schema
  
  @primary_key {:id, :string, autogenerate: false}
  schema "orders" do
    field :customer_email, :string
    field :status, :string
    field :confirmed_at, :utc_datetime
    field :items, {:array, :map}  # Embedded JSON for simplicity
    
    timestamps(type: :utc_datetime, updated_at: false)
  end
end
```

### Why This Step Matters

**Before:** Direct Repo calls in controller
```elixir
# Direct database access
order = Repo.get(Order, id)
Repo.insert(%Order{...})
Repo.update(changeset)
```

**After:** Repository abstracts persistence
```elixir
# Repository abstraction
{:ok, order} = OrderRepository.find_by_id(id)
{:ok, saved} = OrderRepository.save(order)
```

**Benefits:**
- ✅ Can swap Ecto for Mnesia, ETS, or external API without changing domain
- ✅ Can mock repository for testing services
- ✅ Clear boundary between domain and infrastructure
- ✅ Domain models stay pure (no Ecto dependencies)

## Step 3: Extract Application Service

Now we create an application service to orchestrate the "confirm order" use case. Services coordinate between repositories and domain models.

### Confirm Order Service

```elixir
# lib/shop/application/confirm_order_service.ex
defmodule Shop.Application.ConfirmOrderService do
  @moduledoc """
  Application service for confirming an order.
  Orchestrates domain logic, repository, and external services.
  """
  
  alias Shop.Domain.{Order, OrderRepository}
  
  # Inject repository via application config
  @order_repo Application.compile_env(:shop, :order_repository, Shop.Infrastructure.EctoOrderRepository)
  
  @spec execute(String.t()) :: {:ok, Order.t()} | {:error, term()}
  def execute(order_id) when is_binary(order_id) do
    with {:ok, order} <- @order_repo.find_by_id(order_id),
         {:ok, confirmed_order} <- Order.confirm(order),
         {:ok, saved_order} <- @order_repo.save(confirmed_order) do
      {:ok, saved_order}
    end
  end
end
```

### Create Order Service

```elixir
# lib/shop/application/create_order_service.ex
defmodule Shop.Application.CreateOrderService do
  @moduledoc """
  Application service for creating orders.
  Handles order creation workflow including validation and persistence.
  """
  
  alias Shop.Domain.{Order, Email, OrderItem, Money, OrderRepository}
  alias Shop.Domain.ProductRepository
  
  @order_repo Application.compile_env(:shop, :order_repository, Shop.Infrastructure.EctoOrderRepository)
  @product_repo Application.compile_env(:shop, :product_repository, Shop.Infrastructure.EctoProductRepository)
  
  @spec execute(String.t(), list(map())) :: {:ok, Order.t()} | {:error, term()}
  def execute(email_string, items_data) when is_binary(email_string) and is_list(items_data) do
    with {:ok, email} <- Email.new(email_string),
         {:ok, items} <- build_order_items(items_data),
         {:ok, order} <- Order.new(generate_order_id(), email, items),
         {:ok, saved_order} <- @order_repo.save(order) do
      {:ok, saved_order}
    end
  end
  
  defp build_order_items(items_data) do
    items_data
    |> Enum.reduce_while({:ok, []}, fn item_data, {:ok, acc} ->
      case build_single_item(item_data) do
        {:ok, item} -> {:cont, {:ok, [item | acc]}}
        {:error, reason} -> {:halt, {:error, reason}}
      end
    end)
    |> case do
      {:ok, items} -> {:ok, Enum.reverse(items)}
      error -> error
    end
  end
  
  defp build_single_item(%{"product_id" => product_id, "quantity" => quantity}) do
    with {:ok, product} <- @product_repo.find_by_id(product_id),
         true <- product.stock_quantity >= quantity || {:error, :insufficient_stock},
         {:ok, item} <- OrderItem.new(
           product.id,
           product.name,
           Money.new(product.price_cents, :USD),
           quantity
         ) do
      {:ok, item}
    else
      {:error, reason} -> {:error, reason}
      false -> {:error, :insufficient_stock}
    end
  end
  
  defp generate_order_id do
    "ORD-#{System.unique_integer([:positive])}"
  end
end
```

### Why This Step Matters

**Before:** Business workflow in controller
```elixir
def confirm(conn, %{"id" => id}) do
  order = Repo.get(Order, id)
  # Validation
  # Update
  # External calls
  # Error handling
  # All mixed together
end
```

**After:** Service encapsulates workflow
```elixir
case ConfirmOrderService.execute(order_id) do
  {:ok, order} -> # handle success
  {:error, reason} -> # handle error
end
```

**Benefits:**
- ✅ Workflow is testable without Phoenix
- ✅ Can reuse in Phoenix controller, LiveView, GraphQL, background jobs
- ✅ Transaction boundaries are clear
- ✅ Error handling is consistent
- ✅ Easy to add cross-cutting concerns (logging, metrics)


## Step 4: Refactored Controller

Now we update the controller to use our domain models, repository, and services. The controller becomes a thin HTTP layer.

### Refactored Controller

```elixir
# lib/shop_web/controllers/order_controller.ex - AFTER refactoring
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  
  alias Shop.Application.{CreateOrderService, ConfirmOrderService}
  alias Shop.Domain.Order
  
  # Clean controller - only HTTP concerns
  def create(conn, %{"email" => email, "items" => items}) do
    case CreateOrderService.execute(email, items) do
      {:ok, order} ->
        total = Order.calculate_total(order)
        
        conn
        |> put_status(:created)
        |> json(%{
          order_id: order.id,
          total: Money.to_dollars(total),
          status: order.status
        })
      
      {:error, :invalid_email} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Invalid email address"})
      
      {:error, :insufficient_stock} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: "Insufficient stock for one or more items"})
      
      {:error, :empty_items} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Order must contain at least one item"})
      
      {:error, reason} ->
        conn
        |> put_status(:internal_server_error)
        |> json(%{error: "Failed to create order: #{inspect(reason)}"})
    end
  end
  
  def confirm(conn, %{"id" => order_id}) do
    case ConfirmOrderService.execute(order_id) do
      {:ok, order} ->
        json(conn, %{
          order_id: order.id,
          status: order.status,
          confirmed_at: order.confirmed_at
        })
      
      {:error, :not_found} ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Order not found"})
      
      {:error, :invalid_status} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: "Cannot confirm order in current status"})
      
      {:error, :empty_order} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: "Cannot confirm empty order"})
      
      {:error, reason} ->
        conn
        |> put_status(:internal_server_error)
        |> json(%{error: "Failed to confirm order: #{inspect(reason)}"})
    end
  end
end
```

## Before/After Metrics

Let's look at the concrete improvements:

### Metrics Table

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Lines in OrderController** | 150+ | 45 | **70% reduction** |
| **Responsibilities** | 7 (HTTP, validation, business logic, DB, Stripe, email, warehouse) | 1 (HTTP only) | **86% reduction** |
| **Testable without Phoenix** | 0% | 95% | **∞ improvement** |
| **Testable without database** | 0% | 95% | **∞ improvement** |
| **Reusable modules** | 0 | 7 (Order, Email, Money, OrderItem, 2 Services, Repository) | **7 new reusable parts** |
| **Duplicated validation** | Multiple places | 1 place (domain) | **100% reduction** |
| **Test coverage possible** | ~20% | ~95% | **+75%** |
| **Lines of pure business logic** | 0 | 120 | **Pure, testable code** |

### Code Comparison

**Before:**
```elixir
# Mixed concerns - all in controller
def create(conn, params) do
  # Validation
  if !String.contains?(params["email"], "@") do
    # Error handling
  else
    # Database
    order = Repo.insert!(%Order{...})
    # Business logic
    tax = total * 0.08
    # External API
    Stripe.charge(...)
    # More database
    Repo.update!(...)
  end
end
```

**After:**
```elixir
# Clean separation
def create(conn, params) do
  case CreateOrderService.execute(params["email"], params["items"]) do
    {:ok, order} -> json(conn, serialize(order))
    {:error, reason} -> handle_error(conn, reason)
  end
end
```

### File Structure

**Before:**
```
lib/
  shop_web/
    controllers/
      order_controller.ex (150+ lines, does everything)
  shop/
    order.ex (Ecto schema, no business logic)
```

**After:**
```
lib/
  shop/
    domain/
      order.ex (domain model, 80 lines)
      email.ex (value object, 30 lines)
      money.ex (value object, 40 lines)
      order_item.ex (value object, 35 lines)
      order_repository.ex (behaviour, 10 lines)
    
    application/
      create_order_service.ex (service, 45 lines)
      confirm_order_service.ex (service, 20 lines)
    
    infrastructure/
      ecto_order_repository.ex (adapter, 60 lines)
      order_schema.ex (Ecto schema, 15 lines)
  
  shop_web/
    controllers/
      order_controller.ex (HTTP only, 45 lines)
```

**Total lines:** ~380 lines (vs 150 before)

**Wait, more code?** Yes! But:
- ✅ Each module has ONE clear responsibility
- ✅ 95% is testable without Phoenix or database
- ✅ Business logic is reusable everywhere (LiveView, GraphQL, CLI, jobs)
- ✅ Controller is simple and focused on HTTP
- ✅ Adding features is now easier and safer

## Testing Examples

Now that code is separated, testing is straightforward and fast.

### Domain Model Tests

```elixir
# test/shop/domain/order_test.exs
defmodule Shop.Domain.OrderTest do
  use ExUnit.Case, async: true
  
  alias Shop.Domain.{Order, Email, Money, OrderItem}
  
  describe "new/3" do
    test "creates valid order" do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, item} = OrderItem.new("P1", "Laptop", Money.new(100_000, :USD), 1)
      
      assert {:ok, order} = Order.new("ORD-1", email, [item])
      assert order.status == :pending
      assert length(order.items) == 1
    end
    
    test "rejects empty items list" do
      {:ok, email} = Email.new("customer@example.com")
      
      assert {:error, :empty_items} = Order.new("ORD-1", email, [])
    end
  end
  
  describe "calculate_total/1" do
    test "calculates total with tax" do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, item} = OrderItem.new("P1", "Laptop", Money.new(100_000, :USD), 2)
      {:ok, order} = Order.new("ORD-1", email, [item])
      
      # Subtotal: $2000.00
      # Tax (8%): $160.00
      # Total: $2160.00 = 216000 cents
      total = Order.calculate_total(order)
      assert total.amount == 216_000
    end
  end
  
  describe "confirm/1" do
    setup do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, item} = OrderItem.new("P1", "Laptop", Money.new(100_000, :USD), 1)
      {:ok, order} = Order.new("ORD-1", email, [item])
      
      %{order: order}
    end
    
    test "confirms pending order", %{order: order} do
      assert {:ok, confirmed} = Order.confirm(order)
      assert confirmed.status == :confirmed
      assert confirmed.confirmed_at != nil
    end
    
    test "rejects confirming already confirmed order", %{order: order} do
      {:ok, confirmed} = Order.confirm(order)
      
      assert {:error, :invalid_status} = Order.confirm(confirmed)
    end
    
    test "rejects confirming empty order" do
      {:ok, email} = Email.new("customer@example.com")
      # Create order without going through new/3 to bypass validation
      empty_order = %Order{
        id: "ORD-2",
        customer_email: email,
        items: [],
        status: :pending,
        inserted_at: DateTime.utc_now()
      }
      
      assert {:error, :empty_order} = Order.confirm(empty_order)
    end
  end
  
  describe "can_confirm?/1" do
    test "returns true for valid pending order" do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, item} = OrderItem.new("P1", "Laptop", Money.new(100_000, :USD), 1)
      {:ok, order} = Order.new("ORD-1", email, [item])
      
      assert Order.can_confirm?(order) == true
    end
    
    test "returns false for confirmed order" do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, item} = OrderItem.new("P1", "Laptop", Money.new(100_000, :USD), 1)
      {:ok, order} = Order.new("ORD-1", email, [item])
      {:ok, confirmed} = Order.confirm(order)
      
      assert Order.can_confirm?(confirmed) == false
    end
  end
end
```

### Service Tests

```elixir
# test/shop/application/confirm_order_service_test.exs
defmodule Shop.Application.ConfirmOrderServiceTest do
  use ExUnit.Case, async: true
  
  alias Shop.Application.ConfirmOrderService
  alias Shop.Domain.{Order, Email, Money, OrderItem}
  alias Shop.Test.Support.InMemoryOrderRepository
  
  setup do
    # Use in-memory repository for testing
    Application.put_env(:shop, :order_repository, InMemoryOrderRepository)
    
    {:ok, email} = Email.new("customer@example.com")
    {:ok, item} = OrderItem.new("P1", "Laptop", Money.new(100_000, :USD), 1)
    {:ok, order} = Order.new("test-order", email, [item])
    
    # Save order to in-memory repo
    InMemoryOrderRepository.save(order)
    
    on_exit(fn ->
      InMemoryOrderRepository.clear()
      Application.delete_env(:shop, :order_repository)
    end)
    
    %{order_id: order.id}
  end
  
  test "confirms order successfully", %{order_id: order_id} do
    assert {:ok, confirmed_order} = ConfirmOrderService.execute(order_id)
    assert confirmed_order.status == :confirmed
    assert confirmed_order.confirmed_at != nil
  end
  
  test "returns error when order not found" do
    assert {:error, :not_found} = ConfirmOrderService.execute("nonexistent")
  end
  
  test "returns error when order cannot be confirmed", %{order_id: order_id} do
    # Confirm once
    {:ok, _} = ConfirmOrderService.execute(order_id)
    
    # Try to confirm again
    assert {:error, :invalid_status} = ConfirmOrderService.execute(order_id)
  end
end
```

### In-Memory Test Repository

```elixir
# test/support/in_memory_order_repository.ex
defmodule Shop.Test.Support.InMemoryOrderRepository do
  @moduledoc """
  In-memory repository for testing.
  Implements OrderRepository behaviour without database.
  """
  
  @behaviour Shop.Domain.OrderRepository
  
  use Agent
  
  alias Shop.Domain.Order
  
  def start_link(_opts \\ []) do
    Agent.start_link(fn -> %{} end, name: __MODULE__)
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
  def delete(id) do
    Agent.update(__MODULE__, fn orders -> Map.delete(orders, id) end)
    :ok
  end
  
  def clear do
    Agent.update(__MODULE__, fn _ -> %{} end)
  end
end
```

### Controller Tests

```elixir
# test/shop_web/controllers/order_controller_test.exs
defmodule ShopWeb.OrderControllerTest do
  use ShopWeb.ConnCase, async: true
  
  alias Shop.Domain.{Order, Email, Money, OrderItem}
  alias Shop.Test.Support.InMemoryOrderRepository
  
  setup do
    Application.put_env(:shop, :order_repository, InMemoryOrderRepository)
    InMemoryOrderRepository.start_link()
    
    on_exit(fn ->
      InMemoryOrderRepository.clear()
      Application.delete_env(:shop, :order_repository)
    end)
    
    :ok
  end
  
  describe "POST /api/orders" do
    test "creates order successfully", %{conn: conn} do
      params = %{
        "email" => "customer@example.com",
        "items" => [
          %{"product_id" => "P1", "quantity" => 2}
        ]
      }
      
      conn = post(conn, "/api/orders", params)
      
      assert %{
        "order_id" => _order_id,
        "status" => "pending",
        "total" => _total
      } = json_response(conn, 201)
    end
    
    test "returns error for invalid email", %{conn: conn} do
      params = %{
        "email" => "invalid-email",
        "items" => [%{"product_id" => "P1", "quantity" => 1}]
      }
      
      conn = post(conn, "/api/orders", params)
      
      assert %{"error" => "Invalid email address"} = json_response(conn, 400)
    end
  end
  
  describe "POST /api/orders/:id/confirm" do
    setup %{conn: conn} do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, item} = OrderItem.new("P1", "Laptop", Money.new(100_000, :USD), 1)
      {:ok, order} = Order.new("test-order", email, [item])
      InMemoryOrderRepository.save(order)
      
      %{conn: conn, order_id: order.id}
    end
    
    test "confirms order successfully", %{conn: conn, order_id: order_id} do
      conn = post(conn, "/api/orders/#{order_id}/confirm")
      
      assert %{
        "order_id" => ^order_id,
        "status" => "confirmed"
      } = json_response(conn, 200)
    end
    
    test "returns 404 for nonexistent order", %{conn: conn} do
      conn = post(conn, "/api/orders/nonexistent/confirm")
      
      assert %{"error" => "Order not found"} = json_response(conn, 404)
    end
  end
end
```

### Test Coverage Summary

**Domain Layer:** ~95% coverage, tests run in **milliseconds**
- Order creation and validation
- Business calculations (totals, tax)
- State transitions
- Value objects (Email, Money, OrderItem)

**Application Layer:** ~90% coverage, tests run in **milliseconds**
- Service workflows
- Error handling
- Repository interactions (with in-memory impl)

**Controller Layer:** ~75% coverage, tests run in **seconds**
- HTTP request/response handling
- Error code mapping
- Integration with services

**Total:** ~85% meaningful coverage with **fast, reliable tests**

## Common Refactoring Patterns

Let's look at common patterns you'll encounter when refactoring Phoenix code to clean architecture.

### Pattern 1: Extract Validation from Controller

**Before: Validation in controller**
```elixir
def create(conn, %{"email" => email}) do
  cond do
    !String.contains?(email, "@") ->
      json(conn, %{error: "Invalid email"})
    
    String.length(email) > 255 ->
      json(conn, %{error: "Email too long"})
    
    true ->
      # Process...
  end
end
```

**After: Validation in domain**
```elixir
# Domain layer
defmodule Email do
  def new(address) do
    cond do
      !String.contains?(address, "@") -> {:error, :invalid_email}
      byte_size(address) > 255 -> {:error, :invalid_email}
      true -> {:ok, %Email{address: address}}
    end
  end
end

# Controller
def create(conn, %{"email" => email_string}) do
  case Email.new(email_string) do
    {:ok, email} -> # Process...
    {:error, _} -> json(conn, %{error: "Invalid email"})
  end
end
```

### Pattern 2: Extract Business Logic

**Before: Calculation in controller**
```elixir
def create(conn, %{"items" => items}) do
  total = Enum.reduce(items, 0, fn item, acc ->
    acc + (item["price"] * item["quantity"])
  end)
  
  tax = total * 0.08
  final_total = total + tax
  
  json(conn, %{total: final_total})
end
```

**After: Calculation in domain**
```elixir
# Domain layer
defmodule Order do
  def calculate_total(order) do
    subtotal = calculate_subtotal(order)
    tax = Money.multiply(subtotal, 0.08)
    Money.add(subtotal, tax)
  end
end

# Controller
def create(conn, params) do
  {:ok, order} = CreateOrderService.execute(params)
  total = Order.calculate_total(order)
  
  json(conn, %{total: Money.to_dollars(total)})
end
```

### Pattern 3: Extract Ecto Queries

**Before: Repo calls in controller**
```elixir
def show(conn, %{"id" => id}) do
  order = Repo.get(Order, id)
  |> Repo.preload(:items)
  
  if order do
    json(conn, serialize(order))
  else
    send_resp(conn, 404, "Not found")
  end
end
```

**After: Repository abstraction**
```elixir
# Repository
defmodule OrderRepository do
  def find_by_id(id) do
    case Repo.get(OrderSchema, id) do
      nil -> {:error, :not_found}
      schema -> {:ok, to_domain(schema)}
    end
  end
end

# Controller
def show(conn, %{"id" => id}) do
  case OrderRepository.find_by_id(id) do
    {:ok, order} -> json(conn, serialize(order))
    {:error, :not_found} -> send_resp(conn, 404, "Not found")
  end
end
```

## Phoenix Contexts vs Clean Architecture

**Phoenix Contexts are NOT the same as clean architecture.** Understanding the difference is crucial:

### Phoenix Context (Default Approach)

```elixir
# lib/shop/orders.ex
defmodule Shop.Orders do
  import Ecto.Query
  alias Shop.Repo
  alias Shop.Orders.Order
  
  # Mix of domain logic, queries, and infrastructure
  def create_order(attrs) do
    %Order{}
    |> Order.changeset(attrs)
    |> Repo.insert()
  end
  
  def confirm_order(id) do
    order = Repo.get!(Order, id)
    
    if order.status == "pending" do
      order
      |> Ecto.Changeset.change(status: "confirmed")
      |> Repo.update()
    else
      {:error, "Invalid status"}
    end
  end
end
```

**Problems with Phoenix Contexts:**
- Mixes domain logic with Ecto queries
- Uses Ecto schemas as domain models
- Business rules scattered across context functions
- Hard to test without database

### Clean Architecture Approach

```elixir
# lib/shop/domain/order.ex - Pure business logic
defmodule Shop.Domain.Order do
  defstruct [:id, :status, :items]
  
  def confirm(%__MODULE__{status: :pending} = order) do
    {:ok, %{order | status: :confirmed}}
  end
  def confirm(_), do: {:error, :invalid_status}
end

# lib/shop/infrastructure/order_repository.ex - Data access
defmodule Shop.Infrastructure.OrderRepository do
  def save(order), do: # Ecto operations here
  def find_by_id(id), do: # Ecto operations here
end

# lib/shop/application/confirm_order_service.ex - Use case
defmodule Shop.Application.ConfirmOrderService do
  def execute(order_id) do
    with {:ok, order} <- OrderRepository.find_by_id(order_id),
         {:ok, confirmed} <- Order.confirm(order),
         {:ok, saved} <- OrderRepository.save(confirmed) do
      {:ok, saved}
    end
  end
end
```

**You can keep Phoenix Contexts** as application services:

```elixir
# Context becomes application layer
defmodule Shop.Orders do
  alias Shop.Domain.Order
  alias Shop.Infrastructure.OrderRepository
  
  # Application service methods
  def create_order(params) do
    CreateOrderService.execute(params)
  end
  
  def confirm_order(id) do
    ConfirmOrderService.execute(id)
  end
end
```

## Refactoring Large Controllers

Strategy for controllers with 300+ lines:

### 1. Identify Bounded Contexts

Look for natural groupings:
```elixir
# OrderController might actually be:
# - OrderCreationController
# - OrderConfirmationController
# - OrderShippingController
```

### 2. Extract Domain Models First

Start with the most complex business logic:
```elixir
# Priority 1: Order state machine
Order.confirm/1
Order.ship/1
Order.cancel/1

# Priority 2: Calculations
Order.calculate_total/1
Order.calculate_tax/1

# Priority 3: Validation
Email.new/1
OrderItem.new/4
```

### 3. Create Repositories

One repository per aggregate:
```elixir
OrderRepository
ProductRepository
CustomerRepository
```

### 4. Extract Services

One service per use case:
```elixir
CreateOrderService
ConfirmOrderService
CancelOrderService
ShipOrderService
```

### 5. Split Controllers

Split by responsibility:
```elixir
# Before: OrderController (350 lines)
# After:
OrderCreationController (50 lines)
OrderManagementController (60 lines)
OrderQueryController (40 lines)
```

## Refactoring Priorities

What should you refactor first? Prioritize by impact and risk:

### Priority 1: High-Change Areas (Highest ROI)

Features that change frequently get the most benefit from clean architecture.

**Identify:**
- Controllers modified in 50%+ of PRs
- Code with frequent merge conflicts
- Features that change every sprint

**Why first:**
- Immediate productivity improvement
- Reduces bug introduction rate
- Makes future changes faster

**Example:** If pricing rules change monthly, extract `PricingService` first.

### Priority 2: Critical Business Logic (Highest Risk)

Logic that must be correct and affects money/security.

**Identify:**
- Payment processing
- Authorization rules
- Financial calculations
- Inventory management

**Why first:**
- Bugs here are expensive
- Testing improves confidence
- Audit trail is easier

**Example:** Extract order total calculation before tax law changes.

### Priority 3: Hard-to-Test Code (Biggest Pain)

Code that developers avoid testing.

**Identify:**
- Controllers with <20% test coverage
- Code requiring extensive mocking
- Tests marked with `@tag :skip`

**Why first:**
- Enables TDD going forward
- Catches bugs earlier
- Reduces regression risk

**Example:** Extract validation from controller to enable unit testing.

### Priority 4: Code with Many Bugs (Quality Issues)

Controllers with recurring issues.

**Identify:**
- Check bug tracker for repeat issues
- Controllers mentioned in >5 bug reports
- Code with "FIXME" or "TODO" comments

**Why first:**
- Directly reduces bugs
- Prevents regression
- Improves team morale

**Example:** Refactor checkout flow if it has 10 open bugs.

## When NOT to Refactor

Refactoring isn't always the right choice. Be pragmatic:

### ❌ Don't Refactor: Simple CRUD Operations

**Scenario:** Basic create/read/update/delete with minimal business logic.

**Why not:**
- Over-engineering simple operations
- Maintenance cost outweighs benefit
- Phoenix/Ecto already handle this well

**Instead:** Use Phoenix generators and Ecto changesets directly.

### ❌ Don't Refactor: Prototypes

**Scenario:** Validating a product idea or building a demo.

**Why not:**
- Requirements will change completely
- Code might be thrown away
- Time better spent learning from users

**Instead:** Ship fast, gather feedback, refactor if validated.

### ❌ Don't Refactor: Code About to be Deleted

**Scenario:** Feature is deprecated or being replaced.

**Why not:**
- Wasted effort
- Won't improve codebase long-term
- Opportunity cost is high

**Instead:** Focus on the replacement code.

### ❌ Don't Refactor: During Production Outages

**Scenario:** Site is down or critical bug in production.

**Why not:**
- Risk making things worse
- Team stress is already high
- Focus should be on immediate fix

**Instead:** Fix the immediate issue, schedule refactoring after postmortem.

### ❌ Don't Refactor: When You Don't Understand the Domain

**Scenario:** You're new to codebase or business domain.

**Why not:**
- Risk breaking working logic
- Wrong abstractions are worse than duplication
- Missing context leads to bad design

**Instead:** Learn the domain first (pair program, read docs, ask questions), refactor later.

## Team Strategies

Refactoring is easier with team coordination.

### Strategy 1: Start with Pilot Controller

**Approach:**
1. Choose one controller to refactor completely
2. Document the process and patterns discovered
3. Get team feedback in code review
4. Iterate on approach based on feedback
5. Then scale pattern to rest of codebase

**Benefits:**
- Low risk (only one controller)
- Establishes team patterns
- Team learns together
- Can abandon if not working

**Example:** Refactor `OrderController`, document steps in ADR, then apply to `ProductController`.

### Strategy 2: Pair Programming

**Approach:**
- Pair experienced with junior developers
- One drives refactoring, other asks questions
- Swap roles regularly

**Benefits:**
- Knowledge transfer
- Better decisions (two perspectives)
- Fewer mistakes
- Team alignment

**When to use:** First few refactorings, complex domains, onboarding new team members.

### Strategy 3: Code Review Focus

**Approach:**
- Add architecture checklist to PR template
- Review for separation of concerns
- Suggest refactoring opportunities
- Share best practices in comments

**Benefits:**
- Continuous improvement
- Team learns from each other
- Prevents regression to old patterns
- Creates shared vocabulary

**Checklist for reviewers:**
- ✅ Is business logic in domain modules?
- ✅ Are Ecto schemas separate from domain models?
- ✅ Can domain logic be tested without database?
- ✅ Is the code reusable outside this controller?
- ✅ Are state transitions explicit in domain?

### Strategy 4: Gradual Migration

**Approach:**
- New features follow clean architecture
- Old code refactored opportunistically ("Scout Rule")
- Create coexistence layers when needed
- Don't require 100% migration

**Benefits:**
- No "big bang" rewrite
- Delivers value continuously
- Low risk
- Can stop anytime with improved code

### Strategy 5: Architecture Decision Records (ADRs)

**Approach:**
- Document why specific patterns chosen
- Record tradeoffs considered
- Update as you learn more

**Benefits:**
- New team members understand decisions
- Can revisit choices with context
- Creates team alignment
- Reduces repeated debates

**Example ADR:**
```markdown
# ADR 002: Separate Domain Models from Ecto Schemas

## Context
Ecto schemas mix persistence concerns with business logic.
Hard to test business rules without database.

## Decision
Create pure Elixir modules for domain models.
Use Ecto schemas only for database mapping.
Repository layer converts between domain and schemas.

## Consequences
+ Business logic testable without database
+ Domain models reusable in LiveView, GraphQL, CLI
+ Clear separation of concerns
- More mapping code in repository
- Team needs to learn pattern
- Initial development slower

## Status
Accepted - Applied to Order aggregate first
```

## Common Pitfalls

Learn from these common mistakes when refactoring Elixir code:

### ❌ Pitfall 1: Refactoring Everything at Once

**The mistake:**
```
Week 1: Start rewriting entire Phoenix app
Week 2: Still refactoring, no PRs merged, conflicts everywhere
Week 3: Lost track of what's changed, team confused
Week 4: Give up, revert everything, months of work lost
```

**The fix:**
```
Day 1: Extract Order domain model, merge PR
Day 2: Create OrderRepository, merge PR
Day 3: Extract CreateOrderService, merge PR
...
Week 4: 5 controllers refactored, all working, team confident
```

**Lesson:** Small, incremental changes win every time.

### ❌ Pitfall 2: Using Ecto Schemas as Domain Models

**The mistake:**
```elixir
# Using Ecto schema as domain model
defmodule Shop.Order do
  use Ecto.Schema
  
  schema "orders" do
    field :status, :string
    # Ecto concerns mixed with domain
  end
  
  # Business logic directly on schema
  def confirm(order) do
    if order.status == "pending" do
      Ecto.Changeset.change(order, status: "confirmed")
      |> Repo.update()
    end
  end
end
```

**The fix:**
```elixir
# Pure domain model
defmodule Shop.Domain.Order do
  defstruct [:id, :status]
  
  def confirm(%__MODULE__{status: :pending} = order) do
    {:ok, %{order | status: :confirmed}}
  end
  def confirm(_), do: {:error, :invalid_status}
end

# Separate Ecto schema
defmodule Shop.Infrastructure.OrderSchema do
  use Ecto.Schema
  
  schema "orders" do
    field :status, :string
  end
end

# Repository maps between them
defmodule Shop.Infrastructure.OrderRepository do
  def save(order) do
    order |> to_schema() |> Repo.insert()
  end
end
```

**Lesson:** Ecto schemas are for persistence, domain models are for business logic.

### ❌ Pitfall 3: Not Writing Tests During Refactoring

**The mistake:**
```elixir
# Refactor first, test later
def refactor_everything do
  # Move tons of code around
  # Change abstractions
  # Hope nothing breaks
  # "I'll add tests after it works"
end
```

**The fix:**
```elixir
# Test first, then refactor
describe "Order.confirm/1" do
  test "confirms pending order" do
    order = %Order{status: :pending}
    assert {:ok, confirmed} = Order.confirm(order)
    assert confirmed.status == :confirmed
  end
end

# Now refactor with confidence - tests catch regressions
```

**Lesson:** Tests enable fearless refactoring. Write them first.

### ❌ Pitfall 4: Over-Engineering Simple Operations

**The mistake:**
```elixir
# Too many layers for simple CRUD
defmodule PostDomain do
  defstruct [:id, :title]
end

defmodule PostRepository do
  @behaviour PostRepositoryBehaviour
end

defmodule CreatePostService do
  def execute(params) do
    # 50 lines for simple insert
  end
end

defmodule PostFactory do
  # ...
end

# When all you needed was:
def create_post(params) do
  %Post{} |> Post.changeset(params) |> Repo.insert()
end
```

**The fix:**
```elixir
# Use clean architecture only when complexity warrants it
# Simple CRUD? Use Phoenix generators and contexts
mix phx.gen.context Blog Post posts title:string body:text

# Complex business logic? Use clean architecture
```

**Lesson:** Don't use a sledgehammer to crack a nut.

### ❌ Pitfall 5: Wrong Abstractions

**The mistake:**
```elixir
# Generic "manager" or "handler" antipattern
defmodule OrderManager do
  def handle_order(:create, params), do: # ...
  def handle_order(:update, params), do: # ...
  def handle_order(:delete, params), do: # ...
  def handle_order(:send_email, params), do: # Wait, what?
  def handle_order(:calculate_tax, params), do: # This doesn't belong here!
  # Kitchen sink of order-related stuff
end
```

**The fix:**
```elixir
# Specific, focused modules
defmodule Order do
  # Domain model with behavior
  def confirm(order), do: # ...
  def cancel(order), do: # ...
end

defmodule CreateOrderService do
  # One use case
  def execute(params), do: # ...
end

defmodule OrderTaxCalculator do
  # One responsibility
  def calculate(order), do: # ...
end
```

**Lesson:** Specific abstractions > Generic abstractions.

### ❌ Pitfall 6: Ignoring Pattern Matching

**The mistake:**
```elixir
# Not using pattern matching for business rules
def ship_order(order) do
  if order.status == :confirmed do
    if length(order.items) > 0 do
      if has_shipping_address?(order) do
        # Finally do the work
      else
        {:error, "No shipping address"}
      end
    else
      {:error, "No items"}
    end
  else
    {:error, "Invalid status"}
  end
end
```

**The fix:**
```elixir
# Use pattern matching - it's Elixir's superpower!
def ship_order(%Order{status: :confirmed, items: items, shipping_address: addr}) 
  when length(items) > 0 and not is_nil(addr) do
  {:ok, %{order | status: :shipped}}
end

def ship_order(%Order{status: status}) when status != :confirmed do
  {:error, :invalid_status}
end

def ship_order(%Order{items: []}), do: {:error, :no_items}

def ship_order(%Order{shipping_address: nil}), do: {:error, :no_address}
```

**Lesson:** Pattern matching makes business rules explicit and clear.

### ✅ Success Checklist

Before considering a refactoring done, verify:

- [ ] All tests pass (including old tests)
- [ ] Code coverage maintained or improved
- [ ] Business logic is in pure Elixir modules (domain layer)
- [ ] Domain modules have no Phoenix, Ecto, or external dependencies
- [ ] Controllers are <50 lines each
- [ ] Services have single responsibility
- [ ] Repository abstracts all database operations
- [ ] Can test domain logic without database
- [ ] Team understands the changes
- [ ] Documentation/ADRs updated
- [ ] Code is deployed and working in production

## Real-World Example: Complete Refactoring

Let's see a complete example with nested resources showing the full transformation.

### Before: Monolithic OrderContext

```elixir
# lib/shop/orders.ex - 400+ lines, mixed concerns
defmodule Shop.Orders do
  import Ecto.Query
  alias Shop.Repo
  alias Shop.Orders.{Order, OrderItem}
  alias Shop.Products
  
  def create_order(user_id, attrs) do
    # Validation
    with {:ok, validated_attrs} <- validate_order_attrs(attrs),
         # Check inventory
         {:ok, _} <- check_inventory(validated_attrs["items"]),
         # Calculate totals
         {subtotal, tax, total} <- calculate_totals(validated_attrs["items"]),
         # Create order
         {:ok, order} <- insert_order(user_id, total, subtotal, tax),
         # Create items
         {:ok, _items} <- insert_order_items(order.id, validated_attrs["items"]),
         # Update inventory
         :ok <- update_inventory(validated_attrs["items"]),
         # Charge payment
         {:ok, payment} <- charge_payment(order, total),
         # Update order with payment
         {:ok, order} <- update_order_payment(order, payment),
         # Send emails
         :ok <- send_confirmation_email(order) do
      {:ok, load_full_order(order.id)}
    end
  end
  
  defp validate_order_attrs(attrs) do
    # 50 lines of validation
  end
  
  defp check_inventory(items) do
    # 30 lines checking stock
  end
  
  defp calculate_totals(items) do
    # 40 lines of calculations
  end
  
  defp insert_order(user_id, total, subtotal, tax) do
    %Order{
      user_id: user_id,
      total: total,
      subtotal: subtotal,
      tax: tax,
      status: "pending"
    }
    |> Repo.insert()
  end
  
  defp insert_order_items(order_id, items) do
    # 50 lines creating items
  end
  
  defp update_inventory(items) do
    # 40 lines updating stock
  end
  
  defp charge_payment(order, total) do
    # 60 lines Stripe integration
  end
  
  # ... 200 more lines ...
end
```

### After: Clean Architecture with Nested Resources

```elixir
# Domain Layer - Pure business logic
# lib/shop/domain/order.ex
defmodule Shop.Domain.Order do
  @enforce_keys [:id, :user_id, :items, :status]
  defstruct [:id, :user_id, :items, :status, :payment_id, :totals, :confirmed_at]
  
  alias Shop.Domain.{OrderItem, OrderTotals}
  
  def new(id, user_id, items) do
    with :ok <- validate_items(items) do
      totals = OrderTotals.calculate(items)
      
      {:ok, %__MODULE__{
        id: id,
        user_id: user_id,
        items: items,
        status: :pending,
        totals: totals
      }}
    end
  end
  
  def add_payment(%__MODULE__{status: :pending} = order, payment_id) do
    {:ok, %{order | payment_id: payment_id}}
  end
  
  def confirm(%__MODULE__{status: :pending, payment_id: pid} = order) when not is_nil(pid) do
    {:ok, %{order | status: :confirmed, confirmed_at: DateTime.utc_now()}}
  end
  def confirm(_), do: {:error, :cannot_confirm}
  
  defp validate_items([]), do: {:error, :empty_items}
  defp validate_items(_), do: :ok
end

# lib/shop/domain/order_item.ex
defmodule Shop.Domain.OrderItem do
  @enforce_keys [:product_id, :quantity, :unit_price]
  defstruct [:product_id, :product_name, :quantity, :unit_price]
  
  def new(product_id, product_name, quantity, unit_price) do
    with :ok <- validate_quantity(quantity),
         :ok <- validate_price(unit_price) do
      {:ok, %__MODULE__{
        product_id: product_id,
        product_name: product_name,
        quantity: quantity,
        unit_price: unit_price
      }}
    end
  end
  
  def total(%__MODULE__{quantity: qty, unit_price: price}) do
    Money.multiply(price, qty)
  end
  
  defp validate_quantity(qty) when is_integer(qty) and qty > 0, do: :ok
  defp validate_quantity(_), do: {:error, :invalid_quantity}
  
  defp validate_price(%Money{amount: amt}) when amt > 0, do: :ok
  defp validate_price(_), do: {:error, :invalid_price}
end

# lib/shop/domain/order_totals.ex
defmodule Shop.Domain.OrderTotals do
  defstruct [:subtotal, :tax, :total]
  
  @tax_rate 0.08
  
  def calculate(items) do
    subtotal = items
    |> Enum.map(&OrderItem.total/1)
    |> Enum.reduce(Money.zero(), &Money.add/2)
    
    tax = Money.multiply(subtotal, @tax_rate)
    total = Money.add(subtotal, tax)
    
    %__MODULE__{
      subtotal: subtotal,
      tax: tax,
      total: total
    }
  end
end

# Application Layer - Orchestration
# lib/shop/application/create_order_service.ex
defmodule Shop.Application.CreateOrderService do
  alias Shop.Domain.{Order, OrderItem}
  alias Shop.Infrastructure.{OrderRepository, ProductRepository, InventoryService, PaymentService}
  
  def execute(user_id, items_params) do
    Repo.transaction(fn ->
      with {:ok, items} <- build_order_items(items_params),
           :ok <- check_inventory(items),
           {:ok, order} <- create_order(user_id, items),
           {:ok, payment_id} <- process_payment(order),
           {:ok, order_with_payment} <- add_payment_to_order(order, payment_id),
           {:ok, confirmed_order} <- confirm_order(order_with_payment),
           {:ok, saved_order} <- OrderRepository.save(confirmed_order),
           :ok <- reduce_inventory(items) do
        # Send email asynchronously
        Task.start(fn -> send_confirmation_email(saved_order) end)
        
        {:ok, saved_order}
      else
        {:error, reason} ->
          Repo.rollback(reason)
      end
    end)
  end
  
  defp build_order_items(items_params) do
    items_params
    |> Enum.reduce_while({:ok, []}, fn item_params, {:ok, acc} ->
      case build_single_item(item_params) do
        {:ok, item} -> {:cont, {:ok, [item | acc]}}
        error -> {:halt, error}
      end
    end)
    |> case do
      {:ok, items} -> {:ok, Enum.reverse(items)}
      error -> error
    end
  end
  
  defp build_single_item(%{"product_id" => id, "quantity" => qty}) do
    with {:ok, product} <- ProductRepository.find(id),
         {:ok, item} <- OrderItem.new(id, product.name, qty, product.price) do
      {:ok, item}
    end
  end
  
  defp check_inventory(items) do
    InventoryService.check_availability(items)
  end
  
  defp create_order(user_id, items) do
    Order.new(generate_id(), user_id, items)
  end
  
  defp process_payment(order) do
    PaymentService.charge(order)
  end
  
  defp add_payment_to_order(order, payment_id) do
    Order.add_payment(order, payment_id)
  end
  
  defp confirm_order(order) do
    Order.confirm(order)
  end
  
  defp reduce_inventory(items) do
    InventoryService.reduce_stock(items)
  end
  
  defp send_confirmation_email(order) do
    # Send email...
  end
  
  defp generate_id, do: "ORD-#{System.unique_integer([:positive])}"
end

# Infrastructure Layer - External concerns
# lib/shop/infrastructure/order_repository.ex
defmodule Shop.Infrastructure.OrderRepository do
  alias Shop.Domain.Order
  alias Shop.Infrastructure.OrderSchema
  
  def save(%Order{} = order) do
    schema = to_schema(order)
    
    case Repo.insert(schema) do
      {:ok, saved} -> {:ok, to_domain(saved)}
      error -> error
    end
  end
  
  def find(id) do
    case Repo.get(OrderSchema, id) |> Repo.preload(:items) do
      nil -> {:error, :not_found}
      schema -> {:ok, to_domain(schema)}
    end
  end
  
  defp to_schema(%Order{} = order) do
    %OrderSchema{
      id: order.id,
      user_id: order.user_id,
      status: Atom.to_string(order.status),
      subtotal: order.totals.subtotal.amount,
      tax: order.totals.tax.amount,
      total: order.totals.total.amount,
      payment_id: order.payment_id,
      confirmed_at: order.confirmed_at,
      items: Enum.map(order.items, &item_to_schema/1)
    }
  end
  
  defp to_domain(%OrderSchema{} = schema) do
    items = Enum.map(schema.items, &schema_to_item/1)
    
    %Order{
      id: schema.id,
      user_id: schema.user_id,
      items: items,
      status: String.to_existing_atom(schema.status),
      payment_id: schema.payment_id,
      totals: %OrderTotals{
        subtotal: Money.new(schema.subtotal),
        tax: Money.new(schema.tax),
        total: Money.new(schema.total)
      },
      confirmed_at: schema.confirmed_at
    }
  end
  
  # ... item mapping functions ...
end

# lib/shop/infrastructure/payment_service.ex
defmodule Shop.Infrastructure.PaymentService do
  alias Shop.Domain.Order
  
  def charge(%Order{totals: totals}) do
    case Stripe.charge(
      amount: totals.total.amount,
      currency: "usd"
    ) do
      {:ok, charge} -> {:ok, charge.id}
      {:error, _} -> {:error, :payment_failed}
    end
  end
end

# Controller - Thin HTTP layer
# lib/shop_web/controllers/order_controller.ex
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  
  alias Shop.Application.CreateOrderService
  
  def create(conn, %{"items" => items}) do
    user_id = conn.assigns.current_user.id
    
    case CreateOrderService.execute(user_id, items) do
      {:ok, order} ->
        conn
        |> put_status(:created)
        |> json(serialize_order(order))
      
      {:error, :empty_items} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: "Order must contain items"})
      
      {:error, :insufficient_stock} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: "Insufficient stock"})
      
      {:error, :payment_failed} ->
        conn
        |> put_status(:payment_required)
        |> json(%{error: "Payment failed"})
      
      {:error, reason} ->
        conn
        |> put_status(:internal_server_error)
        |> json(%{error: "Failed to create order"})
    end
  end
  
  defp serialize_order(order) do
    %{
      id: order.id,
      status: order.status,
      subtotal: Money.to_dollars(order.totals.subtotal),
      tax: Money.to_dollars(order.totals.tax),
      total: Money.to_dollars(order.totals.total),
      items: Enum.map(order.items, &serialize_item/1)
    }
  end
  
  defp serialize_item(item) do
    %{
      product_id: item.product_id,
      product_name: item.product_name,
      quantity: item.quantity,
      unit_price: Money.to_dollars(item.unit_price)
    }
  end
end
```

**Benefits of the refactored version:**
- ✅ Domain logic is pure and testable in isolation
- ✅ Clear separation: Domain → Application → Infrastructure → Controller
- ✅ Each module has one responsibility
- ✅ Easy to test each layer independently
- ✅ Business rules are explicit in domain models
- ✅ Services coordinate workflows with clear transaction boundaries
- ✅ Repository abstracts database operations
- ✅ Controller is thin - just HTTP translation

## Conclusion

Refactoring Elixir/Phoenix code to clean architecture is a journey of incremental improvement. The key insights:

### 1. Functional Programming + Clean Architecture = Powerful Combination

- Pure functions make business logic fearlessly testable
- Immutability prevents hidden coupling
- Pattern matching makes business rules explicit
- Result tuples make error handling clear

### 2. Incremental Refactoring Wins

- Small, deployable steps reduce risk
- Each step adds value independently
- Old and new code can coexist
- Team learns gradually
- Can stop anytime with improved code

### 3. Domain First, Always

- Extract domain models before anything else
- Make domain pure (no Ecto, no Phoenix, no side effects)
- Everything else follows naturally from this
- Test domain logic without any infrastructure

### 4. Separate Concerns Explicitly

- **Domain:** Pure business logic (structs + functions)
- **Application:** Use case orchestration (services)
- **Infrastructure:** External concerns (Ecto, APIs, email)
- **Controller:** HTTP translation only

### 5. Pragmatic, Not Perfect

- Don't refactor simple CRUD
- Don't over-engineer
- Use Phoenix Contexts as application services
- Apply clean architecture where complexity warrants it
- Simple solutions usually win

### The Transformation

**Before Refactoring:**
```
OrderController (150+ lines)
├─ HTTP handling
├─ Validation
├─ Business logic
├─ Calculations
├─ Database operations
├─ External API calls
├─ Email sending
└─ Everything else
```

**After Refactoring:**
```
Domain (Pure Business Logic)
├─ Order (domain model)
├─ Email (value object)
├─ Money (value object)
└─ OrderItem (value object)

Application (Use Case Orchestration)
├─ CreateOrderService
└─ ConfirmOrderService

Infrastructure (External Concerns)
├─ EctoOrderRepository
├─ PaymentService
└─ EmailService

Controller (HTTP Translation)
└─ OrderController (45 lines)
```

### Your Action Plan

**This week:**
1. Identify one controller to refactor
2. List its current responsibilities (should be 5+)
3. Extract one domain model with tests
4. Deploy and validate

**This month:**
5. Create repository behaviour and implementation
6. Extract one service
7. Update controller to use service
8. Get team feedback on the pattern

**This quarter:**
9. Apply pattern to 10 controllers
10. Document patterns in ADRs
11. Train team members on approach
12. Measure improvement (test coverage, bugs, velocity)

### Remember

- Every refactoring makes the next one easier
- Start small and stay consistent
- Tests enable fearless change
- Pure domain models are your foundation
- Pattern matching is your superpower
- Result tuples make errors explicit
- Your codebase will thank you

### Next Steps

- **Previous**: [Chapter 12: Refactoring React Code](12-refactoring-react-code.md)
- **Next**: Chapter 14: Advanced Topics *(coming soon)*

---

*Refactoring is not about perfection—it's about making tomorrow easier than today. In Elixir, we have the tools to make our business logic pure, explicit, and fearlessly testable. Use them wisely.* ✨
