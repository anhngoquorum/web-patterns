# Chapter 4: Domain Models in Elixir

## Introduction

Domain models in Elixir take a fundamentally different approach from object-oriented languages. Instead of classes with methods and mutable state, we use **structs for data** and **pure functions for behavior**. This functional approach brings powerful advantages: immutability by default, explicit data flow, and exceptional testability.

This chapter explores how to build rich, expressive domain models in Elixir that capture business rules clearly while remaining pure, composable, and easy to test. If you've read Chapter 3 on TypeScript domain models, you'll see how the same domain concepts translate beautifully into a functional paradigm—often with even greater clarity.

### Why Domain Models in Elixir?

In Elixir, a domain model is not a single entity but a collection of:
- **Structs** that define the shape of your data
- **Pure functions** that implement business rules
- **Pattern matching** for validation and flow control
- **Typespecs** for documentation and static analysis

The power of this approach lies in its simplicity and explicitness. Every transformation is visible, every decision point uses pattern matching, and side effects are pushed to the edges of your system.

### The Functional Advantage

Coming from OOP, you might wonder: "Without methods on objects, how do I organize behavior?" The answer is liberating: **behavior is just functions that transform data**. This leads to:

- **No hidden state**: All inputs are explicit function parameters
- **No side effects**: Pure functions always return the same output for the same input
- **Easy testing**: Test functions with plain data—no mocking needed
- **Clear composition**: Functions compose naturally using pipes and pattern matching

Let's see how to build domain models that embody these principles.

## Table of Contents

1. [Domain Entities in Elixir](#domain-entities-in-elixir)
2. [Value Objects](#value-objects)
3. [Validation & Invariants](#validation--invariants)
4. [Complete Example: Order Domain](#complete-example-order-domain)
5. [Comparison with OOP](#comparison-with-oop)
6. [Testing Domain Models](#testing-domain-models)
7. [Best Practices](#best-practices)
8. [Common Pitfalls](#common-pitfalls)

## Domain Entities in Elixir

### What Are Entities?

Entities are domain objects with a unique identity that persists over time. In Elixir, we represent entities as **structs** with **constructor functions** that enforce invariants.

### Defining an Entity

Here's a Product entity with proper structure:

```elixir
defmodule Shop.Domain.Product do
  @moduledoc """
  Represents a product in the shop catalog.
  Products have identity (id), price, and inventory.
  """
  
  @enforce_keys [:id, :name, :price]
  defstruct [
    :id,
    :name,
    :description,
    :price,
    :stock_quantity,
    active: true
  ]
  
  @type t :: %__MODULE__{
    id: String.t(),
    name: String.t(),
    description: String.t() | nil,
    price: Money.t(),
    stock_quantity: non_neg_integer(),
    active: boolean()
  }
  
  @type error_reason :: :invalid_name | :invalid_price | :invalid_stock
  
  # Smart constructor - returns {:ok, product} or {:error, reason}
  @spec new(String.t(), String.t(), Money.t(), non_neg_integer()) ::
    {:ok, t()} | {:error, error_reason()}
  def new(id, name, price, stock_quantity) do
    with :ok <- validate_name(name),
         :ok <- validate_price(price),
         :ok <- validate_stock(stock_quantity) do
      product = %__MODULE__{
        id: id,
        name: String.trim(name),
        description: nil,
        price: price,
        stock_quantity: stock_quantity,
        active: true
      }
      {:ok, product}
    end
  end
  
  # Business logic: check if product can fulfill an order
  @spec can_fulfill?(t(), non_neg_integer()) :: boolean()
  def can_fulfill?(%__MODULE__{active: true, stock_quantity: stock}, quantity) 
    when quantity > 0 and stock >= quantity do
    true
  end
  def can_fulfill?(_product, _quantity), do: false
  
  # Business logic: deduct stock
  @spec deduct_stock(t(), non_neg_integer()) :: 
    {:ok, t()} | {:error, :insufficient_stock}
  def deduct_stock(%__MODULE__{stock_quantity: stock} = product, quantity) 
    when stock >= quantity do
    {:ok, %{product | stock_quantity: stock - quantity}}
  end
  def deduct_stock(_product, _quantity) do
    {:error, :insufficient_stock}
  end
  
  # Business logic: add stock
  @spec add_stock(t(), non_neg_integer()) :: {:ok, t()}
  def add_stock(%__MODULE__{stock_quantity: stock} = product, quantity) 
    when quantity > 0 do
    {:ok, %{product | stock_quantity: stock + quantity}}
  end
  
  # Business logic: activate/deactivate
  @spec activate(t()) :: t()
  def activate(product), do: %{product | active: true}
  
  @spec deactivate(t()) :: t()
  def deactivate(product), do: %{product | active: false}
  
  # Private validation functions
  defp validate_name(name) when is_binary(name) and byte_size(name) > 0 and byte_size(name) <= 200 do
    :ok
  end
  defp validate_name(_), do: {:error, :invalid_name}
  
  defp validate_price(%Money{amount: amount}) when amount > 0, do: :ok
  defp validate_price(_), do: {:error, :invalid_price}
  
  defp validate_stock(quantity) when is_integer(quantity) and quantity >= 0, do: :ok
  defp validate_stock(_), do: {:error, :invalid_stock}
end
```

### Key Patterns in Entity Design

**1. `@enforce_keys`**: Ensures required fields must be provided when creating a struct.

**2. Smart Constructors**: The `new/4` function validates inputs and returns `{:ok, struct}` or `{:error, reason}`.

**3. Typespecs**: Document expected types and enable Dialyzer to catch errors.

**4. Pure Functions**: Every function transforms data without side effects—no database calls, no IO.

**5. Pattern Matching**: Used for validation and control flow (e.g., `can_fulfill?/2`).

### Creating and Using Entities

```elixir
# Create a product
case Product.new("P123", "Gaming Laptop", Money.new(1200, :USD), 10) do
  {:ok, product} ->
    IO.puts("Created product: #{product.name}")
    
  {:error, reason} ->
    IO.puts("Failed to create product: #{reason}")
end

# Use product in business logic
with {:ok, product} <- Product.new("P123", "Laptop", Money.new(1200, :USD), 10),
     true <- Product.can_fulfill?(product, 2),
     {:ok, updated_product} <- Product.deduct_stock(product, 2) do
  IO.puts("Stock remaining: #{updated_product.stock_quantity}")
else
  false -> IO.puts("Cannot fulfill order")
  {:error, reason} -> IO.puts("Error: #{reason}")
end
```

## Value Objects

Value objects are immutable objects defined by their values, not their identity. Two value objects with the same values are considered equal.

### Email Value Object

```elixir
defmodule Shop.Domain.Email do
  @moduledoc """
  Email value object with validation.
  Emails are immutable and validated on creation.
  """
  
  @email_regex ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/
  
  defstruct [:address]
  
  @type t :: %__MODULE__{address: String.t()}
  
  @spec new(String.t()) :: {:ok, t()} | {:error, :invalid_email}
  def new(address) when is_binary(address) do
    normalized = address |> String.trim() |> String.downcase()
    
    cond do
      String.length(normalized) == 0 ->
        {:error, :invalid_email}
      
      String.length(normalized) > 254 ->
        {:error, :invalid_email}
      
      not Regex.match?(@email_regex, normalized) ->
        {:error, :invalid_email}
      
      true ->
        {:ok, %__MODULE__{address: normalized}}
    end
  end
  def new(_), do: {:error, :invalid_email}
  
  @spec domain(t()) :: String.t()
  def domain(%__MODULE__{address: address}) do
    address |> String.split("@") |> List.last()
  end
  
  @spec local_part(t()) :: String.t()
  def local_part(%__MODULE__{address: address}) do
    address |> String.split("@") |> List.first()
  end
  
  @spec to_string(t()) :: String.t()
  def to_string(%__MODULE__{address: address}), do: address
end
```

### Money Value Object

```elixir
defmodule Shop.Domain.Money do
  @moduledoc """
  Money value object representing an amount in a specific currency.
  All operations return new Money instances (immutable).
  """
  
  defstruct [:amount, :currency]
  
  @type t :: %__MODULE__{
    amount: integer(),  # Amount in smallest unit (cents)
    currency: atom()
  }
  
  @spec new(integer(), atom()) :: {:ok, t()} | {:error, :invalid_money}
  def new(amount, currency) when is_integer(amount) and is_atom(currency) do
    {:ok, %__MODULE__{amount: amount, currency: currency}}
  end
  def new(_, _), do: {:error, :invalid_money}
  
  @spec zero(atom()) :: t()
  def zero(currency \\ :USD) do
    %__MODULE__{amount: 0, currency: currency}
  end
  
  # Arithmetic operations
  @spec add(t(), t()) :: {:ok, t()} | {:error, :currency_mismatch}
  def add(%__MODULE__{currency: c1} = m1, %__MODULE__{currency: c2}) when c1 != c2 do
    {:error, :currency_mismatch}
  end
  def add(%__MODULE__{amount: a1, currency: c}, %__MODULE__{amount: a2}) do
    {:ok, %__MODULE__{amount: a1 + a2, currency: c}}
  end
  
  @spec subtract(t(), t()) :: {:ok, t()} | {:error, :currency_mismatch}
  def subtract(%__MODULE__{currency: c1}, %__MODULE__{currency: c2}) when c1 != c2 do
    {:error, :currency_mismatch}
  end
  def subtract(%__MODULE__{amount: a1, currency: c}, %__MODULE__{amount: a2}) do
    {:ok, %__MODULE__{amount: a1 - a2, currency: c}}
  end
  
  @spec multiply(t(), number()) :: t()
  def multiply(%__MODULE__{amount: amount, currency: c}, factor) do
    %__MODULE__{amount: round(amount * factor), currency: c}
  end
  
  # Comparison operations
  @spec compare(t(), t()) :: :lt | :eq | :gt | {:error, :currency_mismatch}
  def compare(%__MODULE__{currency: c1}, %__MODULE__{currency: c2}) when c1 != c2 do
    {:error, :currency_mismatch}
  end
  def compare(%__MODULE__{amount: a1}, %__MODULE__{amount: a2}) do
    cond do
      a1 < a2 -> :lt
      a1 > a2 -> :gt
      true -> :eq
    end
  end
  
  @spec positive?(t()) :: boolean()
  def positive?(%__MODULE__{amount: amount}), do: amount > 0
  
  @spec negative?(t()) :: boolean()
  def negative?(%__MODULE__{amount: amount}), do: amount < 0
  
  @spec zero?(t()) :: boolean()
  def zero?(%__MODULE__{amount: 0}), do: true
  def zero?(_), do: false
  
  # Formatting
  @spec format(t()) :: String.t()
  def format(%__MODULE__{amount: amount, currency: currency}) do
    dollars = div(amount, 100)
    cents = rem(amount, 100)
    "#{currency} $#{dollars}.#{String.pad_leading(Integer.to_string(cents), 2, "0")}"
  end
end
```

### Quantity Value Object

```elixir
defmodule Shop.Domain.Quantity do
  @moduledoc """
  Represents a quantity with constraints.
  Quantities must be positive integers.
  """
  
  defstruct [:value]
  
  @type t :: %__MODULE__{value: pos_integer()}
  
  @spec new(integer()) :: {:ok, t()} | {:error, :invalid_quantity}
  def new(value) when is_integer(value) and value > 0 do
    {:ok, %__MODULE__{value: value}}
  end
  def new(_), do: {:error, :invalid_quantity}
  
  @spec add(t(), t()) :: t()
  def add(%__MODULE__{value: v1}, %__MODULE__{value: v2}) do
    %__MODULE__{value: v1 + v2}
  end
  
  @spec to_integer(t()) :: pos_integer()
  def to_integer(%__MODULE__{value: value}), do: value
end
```

## Validation & Invariants

Validation in Elixir uses a combination of pattern matching, guard clauses, and the `with` construct for complex validation chains.

### Pattern Matching Validation

```elixir
defmodule Shop.Domain.OrderItem do
  defstruct [:product_id, :unit_price, :quantity]
  
  @type t :: %__MODULE__{
    product_id: String.t(),
    unit_price: Money.t(),
    quantity: Quantity.t()
  }
  
  # Pattern match ensures correct types and validates in guards
  @spec new(String.t(), Money.t(), Quantity.t()) :: 
    {:ok, t()} | {:error, atom()}
  def new(product_id, %Money{} = unit_price, %Quantity{} = quantity)
    when is_binary(product_id) and byte_size(product_id) > 0 do
    
    if Money.positive?(unit_price) do
      item = %__MODULE__{
        product_id: product_id,
        unit_price: unit_price,
        quantity: quantity
      }
      {:ok, item}
    else
      {:error, :price_must_be_positive}
    end
  end
  def new(_, _, _), do: {:error, :invalid_order_item}
  
  @spec subtotal(t()) :: Money.t()
  def subtotal(%__MODULE__{unit_price: price, quantity: qty}) do
    Money.multiply(price, Quantity.to_integer(qty))
  end
end
```

### Using `with` for Validation Chains

The `with` construct is perfect for validating multiple conditions:

```elixir
defmodule Shop.Domain.Order do
  # ... struct definition ...
  
  @spec add_item(t(), Product.t(), pos_integer()) :: 
    {:ok, t()} | {:error, atom()}
  def add_item(%__MODULE__{status: :pending} = order, product, quantity) do
    with {:ok, qty} <- Quantity.new(quantity),
         true <- Product.can_fulfill?(product, quantity),
         {:ok, item} <- OrderItem.new(product.id, product.price, qty) do
      updated_order = %{order | items: [item | order.items]}
      {:ok, updated_order}
    else
      {:error, reason} -> {:error, reason}
      false -> {:error, :cannot_fulfill_order}
    end
  end
  def add_item(_order, _product, _quantity) do
    {:error, :order_not_modifiable}
  end
end
```

### Guard Clauses for Pre-conditions

```elixir
defmodule Shop.Domain.Order do
  # Only pending orders can be modified
  def add_item(%__MODULE__{status: :pending} = order, product, quantity) do
    # ... implementation ...
  end
  def add_item(_order, _product, _quantity) do
    {:error, :order_not_modifiable}
  end
  
  # Only confirmed orders can be shipped
  def ship(%__MODULE__{status: :confirmed, items: items} = order) 
    when length(items) > 0 do
    {:ok, %{order | status: :shipped}}
  end
  def ship(%__MODULE__{status: status}) when status != :confirmed do
    {:error, :invalid_status_transition}
  end
  def ship(%__MODULE__{items: []}) do
    {:error, :cannot_ship_empty_order}
  end
end
```

## Complete Example: Order Domain

Let's build a complete order management domain that ties everything together.

### Order Entity

```elixir
defmodule Shop.Domain.Order do
  @moduledoc """
  Represents a customer order.
  Orders are aggregates that manage order items and status transitions.
  """
  
  alias Shop.Domain.{OrderItem, Product, Email, Money, Quantity}
  
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
  
  @spec new(String.t(), Email.t()) :: {:ok, t()}
  def new(id, %Email{} = customer_email) when is_binary(id) and byte_size(id) > 0 do
    order = %__MODULE__{
      id: id,
      customer_email: customer_email,
      order_date: DateTime.utc_now(),
      items: [],
      status: :pending
    }
    {:ok, order}
  end
  
  @spec add_item(t(), Product.t(), pos_integer()) :: 
    {:ok, t()} | {:error, atom()}
  def add_item(%__MODULE__{status: :pending} = order, product, quantity) do
    with {:ok, qty} <- Quantity.new(quantity),
         true <- Product.can_fulfill?(product, quantity),
         {:ok, item} <- OrderItem.new(product.id, product.price, qty) do
      # Check if item already exists, if so increase quantity
      updated_items = merge_or_add_item(order.items, item)
      updated_order = %{order | items: updated_items}
      {:ok, updated_order}
    else
      {:error, reason} -> {:error, reason}
      false -> {:error, :cannot_fulfill_order}
    end
  end
  def add_item(_order, _product, _quantity) do
    {:error, :order_not_modifiable}
  end
  
  @spec remove_item(t(), String.t()) :: {:ok, t()} | {:error, atom()}
  def remove_item(%__MODULE__{status: :pending, items: items} = order, product_id) do
    case Enum.reject(items, fn item -> item.product_id == product_id end) do
      ^items -> {:error, :item_not_found}
      updated_items -> {:ok, %{order | items: updated_items}}
    end
  end
  def remove_item(_order, _product_id) do
    {:error, :order_not_modifiable}
  end
  
  @spec calculate_total(t()) :: Money.t()
  def calculate_total(%__MODULE__{items: items}) do
    Enum.reduce(items, Money.zero(:USD), fn item, acc ->
      subtotal = OrderItem.subtotal(item)
      case Money.add(acc, subtotal) do
        {:ok, new_total} -> new_total
        _ -> acc
      end
    end)
  end
  
  @spec item_count(t()) :: non_neg_integer()
  def item_count(%__MODULE__{items: items}) do
    Enum.reduce(items, 0, fn item, acc ->
      acc + Quantity.to_integer(item.quantity)
    end)
  end
  
  # Status transitions
  @spec confirm(t()) :: {:ok, t()} | {:error, atom()}
  def confirm(%__MODULE__{status: :pending, items: items} = order) 
    when length(items) > 0 do
    {:ok, %{order | status: :confirmed}}
  end
  def confirm(%__MODULE__{status: status}) when status != :pending do
    {:error, :invalid_status_transition}
  end
  def confirm(%__MODULE__{items: []}) do
    {:error, :cannot_confirm_empty_order}
  end
  
  @spec ship(t()) :: {:ok, t()} | {:error, atom()}
  def ship(%__MODULE__{status: :confirmed} = order) do
    {:ok, %{order | status: :shipped}}
  end
  def ship(_order), do: {:error, :invalid_status_transition}
  
  @spec deliver(t()) :: {:ok, t()} | {:error, atom()}
  def deliver(%__MODULE__{status: :shipped} = order) do
    {:ok, %{order | status: :delivered}}
  end
  def deliver(_order), do: {:error, :invalid_status_transition}
  
  @spec cancel(t()) :: {:ok, t()} | {:error, atom()}
  def cancel(%__MODULE__{status: :delivered}) do
    {:error, :cannot_cancel_delivered_order}
  end
  def cancel(%__MODULE__{status: :cancelled}) do
    {:error, :already_cancelled}
  end
  def cancel(order) do
    {:ok, %{order | status: :cancelled}}
  end
  
  # Private helpers
  defp merge_or_add_item(items, new_item) do
    case Enum.find_index(items, fn item -> 
      item.product_id == new_item.product_id 
    end) do
      nil ->
        [new_item | items]
      
      index ->
        existing = Enum.at(items, index)
        combined_qty = Quantity.add(existing.quantity, new_item.quantity)
        updated = %{existing | quantity: combined_qty}
        List.replace_at(items, index, updated)
    end
  end
end
```

### Complete Usage Example

```elixir
# Create domain objects
{:ok, email} = Email.new("customer@example.com")
{:ok, order} = Order.new("ORD-001", email)

{:ok, laptop_price} = Money.new(120_000, :USD)  # $1,200.00
{:ok, laptop} = Product.new("P001", "Gaming Laptop", laptop_price, 5)

{:ok, mouse_price} = Money.new(5_000, :USD)  # $50.00
{:ok, mouse} = Product.new("P002", "Gaming Mouse", mouse_price, 20)

# Add items to order
with {:ok, order} <- Order.add_item(order, laptop, 1),
     {:ok, order} <- Order.add_item(order, mouse, 2) do
  
  # Check order details
  total = Order.calculate_total(order)
  count = Order.item_count(order)
  
  IO.puts("Order #{order.id}")
  IO.puts("Items: #{count}")
  IO.puts("Total: #{Money.format(total)}")  # USD $1300.00
  
  # Process order through its lifecycle
  with {:ok, order} <- Order.confirm(order),
       {:ok, order} <- Order.ship(order),
       {:ok, order} <- Order.deliver(order) do
    IO.puts("Order delivered successfully!")
    {:ok, order}
  end
else
  {:error, reason} -> 
    IO.puts("Order processing failed: #{reason}")
    {:error, reason}
end
```

### Domain Service Example

Sometimes behavior doesn't belong to a single entity. Use domain services:

```elixir
defmodule Shop.Domain.PricingService do
  @moduledoc """
  Domain service for pricing calculations that involve multiple entities.
  """
  
  alias Shop.Domain.{Order, Money}
  
  @spec apply_discount(Order.t(), String.t()) :: 
    {:ok, {Order.t(), Money.t()}} | {:error, atom()}
  def apply_discount(order, discount_code) do
    discount_amount = calculate_discount(order, discount_code)
    {:ok, {order, discount_amount}}
  end
  
  @spec calculate_discount(Order.t(), String.t()) :: Money.t()
  defp calculate_discount(order, "FIRSTORDER") do
    total = Order.calculate_total(order)
    Money.multiply(total, 0.15)  # 15% off
  end
  defp calculate_discount(order, "SAVE20") do
    total = Order.calculate_total(order)
    Money.multiply(total, 0.20)  # 20% off
  end
  defp calculate_discount(_order, _code) do
    Money.zero(:USD)
  end
end
```

## Comparison with OOP

Understanding how functional domain modeling differs from OOP helps clarify design decisions.

### Data and Behavior Separation

**OOP Approach:**
```typescript
class Order {
  private items: OrderItem[] = [];
  
  addItem(product: Product, quantity: number): void {
    // Method modifies internal state
    this.items.push(new OrderItem(product, quantity));
  }
  
  calculateTotal(): Money {
    // Method accesses internal state
    return this.items.reduce(...);
  }
}
```

**Elixir Approach:**
```elixir
defmodule Order do
  # Data structure
  defstruct [:id, :items, :status]
  
  # Pure function takes and returns data
  def add_item(order, product, quantity) do
    item = OrderItem.new(product.id, product.price, quantity)
    %{order | items: [item | order.items]}
  end
  
  # Pure function transforms data
  def calculate_total(order) do
    Enum.reduce(order.items, Money.zero(), &Money.add/2)
  end
end
```

**Key Differences:**
- **OOP**: Behavior is attached to objects (methods). State is hidden and mutable.
- **Elixir**: Behavior is separate (functions). Data is explicit and immutable.

### Validation Strategies

**OOP Approach:**
```typescript
class Product {
  constructor(
    private name: string,
    private price: number
  ) {
    this.validate();  // Validation in constructor
  }
  
  private validate(): void {
    if (this.price <= 0) {
      throw new Error('Price must be positive');
    }
  }
}
```

**Elixir Approach:**
```elixir
defmodule Product do
  def new(name, price) do
    # Validation returns result type
    with :ok <- validate_name(name),
         :ok <- validate_price(price) do
      {:ok, %Product{name: name, price: price}}
    end
  end
  
  defp validate_price(price) when price > 0, do: :ok
  defp validate_price(_), do: {:error, :invalid_price}
end
```

**Key Differences:**
- **OOP**: Validation throws exceptions. Invalid objects cannot exist.
- **Elixir**: Validation returns `{:ok, value}` or `{:error, reason}`. Explicit error handling.

### State Changes

**OOP Approach:**
```typescript
class Order {
  private status: OrderStatus = 'pending';
  
  confirm(): void {
    if (this.status !== 'pending') {
      throw new Error('Invalid transition');
    }
    this.status = 'confirmed';  // Mutation
  }
}
```

**Elixir Approach:**
```elixir
defmodule Order do
  def confirm(%Order{status: :pending} = order) do
    {:ok, %{order | status: :confirmed}}  # New struct
  end
  def confirm(_order) do
    {:error, :invalid_transition}
  end
end
```

**Key Differences:**
- **OOP**: Methods mutate object state in place.
- **Elixir**: Functions return new data structures. Original is unchanged.

### Why This Matters

The functional approach offers several advantages:

1. **Predictability**: Pure functions always produce the same output for the same input
2. **Testability**: No setup required—just call functions with data
3. **Debugging**: Easy to trace data transformations through the pipeline
4. **Concurrency**: Immutable data eliminates race conditions
5. **Time Travel**: Keep old versions of data for free (useful for undo, audit logs)

## Testing Domain Models

Testing domain models in Elixir is exceptionally straightforward because they're pure functions operating on plain data.

### Basic Entity Tests

```elixir
defmodule Shop.Domain.ProductTest do
  use ExUnit.Case, async: true
  
  alias Shop.Domain.{Product, Money}
  
  describe "new/4" do
    test "creates valid product" do
      {:ok, price} = Money.new(10_000, :USD)
      {:ok, product} = Product.new("P1", "Laptop", price, 10)
      
      assert product.id == "P1"
      assert product.name == "Laptop"
      assert product.price == price
      assert product.stock_quantity == 10
      assert product.active == true
    end
    
    test "trims product name" do
      {:ok, price} = Money.new(10_000, :USD)
      {:ok, product} = Product.new("P1", "  Laptop  ", price, 10)
      
      assert product.name == "Laptop"
    end
    
    test "rejects empty name" do
      {:ok, price} = Money.new(10_000, :USD)
      assert {:error, :invalid_name} = Product.new("P1", "", price, 10)
    end
    
    test "rejects name longer than 200 characters" do
      {:ok, price} = Money.new(10_000, :USD)
      long_name = String.duplicate("a", 201)
      assert {:error, :invalid_name} = Product.new("P1", long_name, price, 10)
    end
    
    test "rejects invalid price" do
      {:ok, zero_price} = Money.new(0, :USD)
      assert {:error, :invalid_price} = Product.new("P1", "Laptop", zero_price, 10)
    end
    
    test "rejects negative stock" do
      {:ok, price} = Money.new(10_000, :USD)
      assert {:error, :invalid_stock} = Product.new("P1", "Laptop", price, -1)
    end
  end
  
  describe "can_fulfill?/2" do
    setup do
      {:ok, price} = Money.new(10_000, :USD)
      {:ok, product} = Product.new("P1", "Laptop", price, 10)
      {:ok, product: product}
    end
    
    test "returns true when stock is sufficient and active", %{product: product} do
      assert Product.can_fulfill?(product, 5) == true
      assert Product.can_fulfill?(product, 10) == true
    end
    
    test "returns false when stock is insufficient", %{product: product} do
      assert Product.can_fulfill?(product, 11) == false
    end
    
    test "returns false when product is inactive", %{product: product} do
      inactive_product = Product.deactivate(product)
      assert Product.can_fulfill?(inactive_product, 5) == false
    end
    
    test "returns false for zero or negative quantity", %{product: product} do
      assert Product.can_fulfill?(product, 0) == false
      assert Product.can_fulfill?(product, -1) == false
    end
  end
  
  describe "deduct_stock/2" do
    setup do
      {:ok, price} = Money.new(10_000, :USD)
      {:ok, product} = Product.new("P1", "Laptop", price, 10)
      {:ok, product: product}
    end
    
    test "deducts stock correctly", %{product: product} do
      {:ok, updated} = Product.deduct_stock(product, 3)
      assert updated.stock_quantity == 7
    end
    
    test "returns error when insufficient stock", %{product: product} do
      assert {:error, :insufficient_stock} = Product.deduct_stock(product, 11)
    end
    
    test "does not modify original product", %{product: product} do
      {:ok, _updated} = Product.deduct_stock(product, 3)
      assert product.stock_quantity == 10  # Original unchanged
    end
  end
end
```

### Value Object Tests

```elixir
defmodule Shop.Domain.EmailTest do
  use ExUnit.Case, async: true
  
  alias Shop.Domain.Email
  
  describe "new/1" do
    test "creates valid email" do
      {:ok, email} = Email.new("user@example.com")
      assert email.address == "user@example.com"
    end
    
    test "normalizes to lowercase" do
      {:ok, email} = Email.new("User@Example.COM")
      assert email.address == "user@example.com"
    end
    
    test "trims whitespace" do
      {:ok, email} = Email.new("  user@example.com  ")
      assert email.address == "user@example.com"
    end
    
    test "rejects invalid formats" do
      assert {:error, :invalid_email} = Email.new("invalid")
      assert {:error, :invalid_email} = Email.new("@example.com")
      assert {:error, :invalid_email} = Email.new("user@")
      assert {:error, :invalid_email} = Email.new("")
    end
    
    test "rejects emails longer than 254 characters" do
      long_email = String.duplicate("a", 250) <> "@example.com"
      assert {:error, :invalid_email} = Email.new(long_email)
    end
  end
  
  describe "domain/1" do
    test "extracts domain from email" do
      {:ok, email} = Email.new("user@example.com")
      assert Email.domain(email) == "example.com"
    end
  end
  
  describe "local_part/1" do
    test "extracts local part from email" do
      {:ok, email} = Email.new("user@example.com")
      assert Email.local_part(email) == "user"
    end
  end
end
```

### Order Lifecycle Tests

```elixir
defmodule Shop.Domain.OrderTest do
  use ExUnit.Case, async: true
  
  alias Shop.Domain.{Order, Product, Email, Money, Quantity}
  
  describe "order lifecycle" do
    setup do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, order} = Order.new("O1", email)
      
      {:ok, price} = Money.new(100_000, :USD)
      {:ok, product} = Product.new("P1", "Laptop", price, 10)
      
      {:ok, order: order, product: product}
    end
    
    test "creates order in pending status", %{order: order} do
      assert order.status == :pending
      assert order.items == []
    end
    
    test "transitions through valid statuses", %{order: order, product: product} do
      # Add item and confirm
      {:ok, order} = Order.add_item(order, product, 1)
      {:ok, order} = Order.confirm(order)
      assert order.status == :confirmed
      
      # Ship
      {:ok, order} = Order.ship(order)
      assert order.status == :shipped
      
      # Deliver
      {:ok, order} = Order.deliver(order)
      assert order.status == :delivered
    end
    
    test "rejects invalid status transitions", %{order: order} do
      # Cannot ship before confirming
      assert {:error, :invalid_status_transition} = Order.ship(order)
      
      # Cannot deliver before shipping
      assert {:error, :invalid_status_transition} = Order.deliver(order)
    end
    
    test "cannot modify non-pending orders", %{order: order, product: product} do
      {:ok, order} = Order.add_item(order, product, 1)
      {:ok, order} = Order.confirm(order)
      
      # Cannot add items after confirmation
      assert {:error, :order_not_modifiable} = Order.add_item(order, product, 1)
    end
    
    test "cannot confirm empty order", %{order: order} do
      assert {:error, :cannot_confirm_empty_order} = Order.confirm(order)
    end
  end
  
  describe "item management" do
    setup do
      {:ok, email} = Email.new("customer@example.com")
      {:ok, order} = Order.new("O1", email)
      
      {:ok, price1} = Money.new(100_000, :USD)
      {:ok, product1} = Product.new("P1", "Laptop", price1, 10)
      
      {:ok, price2} = Money.new(5_000, :USD)
      {:ok, product2} = Product.new("P2", "Mouse", price2, 20)
      
      {:ok, order: order, product1: product1, product2: product2}
    end
    
    test "adds items to order", %{order: order, product1: product1} do
      {:ok, order} = Order.add_item(order, product1, 2)
      assert length(order.items) == 1
      assert hd(order.items).product_id == "P1"
    end
    
    test "combines quantities for same product", %{order: order, product1: product1} do
      {:ok, order} = Order.add_item(order, product1, 2)
      {:ok, order} = Order.add_item(order, product1, 3)
      
      assert length(order.items) == 1
      item = hd(order.items)
      assert Quantity.to_integer(item.quantity) == 5
    end
    
    test "calculates total correctly", 
      %{order: order, product1: product1, product2: product2} do
      {:ok, order} = Order.add_item(order, product1, 1)  # $1000
      {:ok, order} = Order.add_item(order, product2, 2)  # $50 * 2 = $100
      
      total = Order.calculate_total(order)
      assert total.amount == 110_000  # $1100.00
      assert total.currency == :USD
    end
    
    test "counts items correctly", %{order: order, product1: product1, product2: product2} do
      {:ok, order} = Order.add_item(order, product1, 1)
      {:ok, order} = Order.add_item(order, product2, 2)
      
      assert Order.item_count(order) == 3
    end
    
    test "removes items from order", %{order: order, product1: product1} do
      {:ok, order} = Order.add_item(order, product1, 2)
      {:ok, order} = Order.remove_item(order, "P1")
      
      assert order.items == []
    end
    
    test "returns error when removing non-existent item", %{order: order} do
      assert {:error, :item_not_found} = Order.remove_item(order, "P999")
    end
  end
end
```

### Testing with Property-Based Testing

For more thorough testing, use StreamData:

```elixir
defmodule Shop.Domain.MoneyPropertyTest do
  use ExUnit.Case, async: true
  use ExUnitProperties
  
  alias Shop.Domain.Money
  
  property "adding money is commutative" do
    check all amount1 <- positive_integer(),
              amount2 <- positive_integer() do
      {:ok, m1} = Money.new(amount1, :USD)
      {:ok, m2} = Money.new(amount2, :USD)
      
      {:ok, result1} = Money.add(m1, m2)
      {:ok, result2} = Money.add(m2, m1)
      
      assert result1.amount == result2.amount
    end
  end
  
  property "multiplying by 0 gives zero money" do
    check all amount <- positive_integer() do
      {:ok, money} = Money.new(amount, :USD)
      result = Money.multiply(money, 0)
      
      assert Money.zero?(result)
    end
  end
end
```

## Best Practices

### 1. Use Smart Constructors

Always validate in constructor functions, returning result tuples:

```elixir
# ✅ Good: Validation with result type
def new(name, price) do
  with :ok <- validate_name(name),
       :ok <- validate_price(price) do
    {:ok, %Product{name: name, price: price}}
  end
end

# ❌ Bad: No validation
def new(name, price) do
  %Product{name: name, price: price}
end
```

### 2. Enforce Keys for Required Fields

Use `@enforce_keys` to catch missing data at compile time:

```elixir
# ✅ Good: Required fields enforced
@enforce_keys [:id, :name, :price]
defstruct [:id, :name, :price, :description]

# ❌ Bad: No enforcement
defstruct [:id, :name, :price, :description]
```

### 3. Use Typespecs Everywhere

Typespecs document your code and enable Dialyzer:

```elixir
# ✅ Good: Comprehensive types
@type t :: %__MODULE__{
  id: String.t(),
  name: String.t(),
  price: Money.t()
}

@spec new(String.t(), String.t(), Money.t()) :: 
  {:ok, t()} | {:error, atom()}

# ❌ Bad: No types
def new(id, name, price) do
  # ...
end
```

### 4. Keep Functions Pure

Domain functions should not have side effects:

```elixir
# ✅ Good: Pure function
def calculate_total(order) do
  Enum.reduce(order.items, Money.zero(), fn item, acc ->
    Money.add(acc, item_subtotal(item))
  end)
end

# ❌ Bad: Side effects in domain
def calculate_total(order) do
  Logger.info("Calculating total")  # Side effect!
  DB.insert(order)  # Side effect!
  # ...
end
```

### 5. Use Pattern Matching for Validation

Pattern matching is clearer than conditionals:

```elixir
# ✅ Good: Pattern matching
def ship(%Order{status: :confirmed} = order) do
  {:ok, %{order | status: :shipped}}
end
def ship(_order), do: {:error, :invalid_status}

# ❌ Bad: Nested conditionals
def ship(order) do
  if order.status == :confirmed do
    {:ok, %{order | status: :shipped}}
  else
    {:error, :invalid_status}
  end
end
```

### 6. Use `with` for Complex Validation

Chain validations clearly with `with`:

```elixir
# ✅ Good: Clear validation chain
def create_order(customer_email, items) do
  with {:ok, email} <- Email.new(customer_email),
       {:ok, order} <- Order.new(generate_id(), email),
       {:ok, order} <- add_items(order, items),
       {:ok, order} <- Order.confirm(order) do
    {:ok, order}
  end
end

# ❌ Bad: Nested case statements
def create_order(customer_email, items) do
  case Email.new(customer_email) do
    {:ok, email} ->
      case Order.new(generate_id(), email) do
        {:ok, order} ->
          # More nesting...
      end
  end
end
```

### 7. Make Invariants Explicit

Express business rules clearly in code:

```elixir
# ✅ Good: Explicit invariants
def add_item(%Order{status: :pending} = order, product, qty) 
  when qty > 0 do
  # Add item
end

# ❌ Bad: Implicit constraints
def add_item(order, product, qty) do
  if order.status == :pending and qty > 0 do
    # Add item
  end
end
```

### 8. Return Result Tuples Consistently

Always use `{:ok, value}` or `{:error, reason}`:

```elixir
# ✅ Good: Consistent result types
@spec deduct_stock(t(), pos_integer()) :: 
  {:ok, t()} | {:error, :insufficient_stock}

# ❌ Bad: Raises exceptions
def deduct_stock(product, qty) do
  if product.stock < qty do
    raise "Insufficient stock"  # Exception in domain!
  end
  # ...
end
```

## Common Pitfalls

### ❌ 1. Putting Database Queries in Domain Modules

```elixir
# ❌ Bad: Database in domain
defmodule Shop.Domain.Order do
  def add_item(order, product_id, qty) do
    product = Repo.get!(Product, product_id)  # Database query!
    # ...
  end
end

# ✅ Good: Pass product as parameter
defmodule Shop.Domain.Order do
  def add_item(order, product, qty) do
    # Pure domain logic
  end
end
```

### ❌ 2. Using Ecto Schemas as Domain Models Directly

```elixir
# ❌ Bad: Ecto schema is domain model
defmodule Shop.Order do
  use Ecto.Schema
  
  schema "orders" do
    field :status, :string
    # Mixing persistence and domain concerns
  end
end

# ✅ Good: Separate domain and persistence
defmodule Shop.Domain.Order do
  defstruct [:id, :status]
  # Pure domain model
end

defmodule Shop.Infrastructure.OrderSchema do
  use Ecto.Schema
  # Database mapping
end
```

### ❌ 3. Mutable State Thinking

```elixir
# ❌ Bad: Trying to mutate
def process_order(order) do
  order.status = :confirmed  # Won't work in Elixir!
end

# ✅ Good: Return new value
def process_order(order) do
  %{order | status: :confirmed}
end
```

### ❌ 4. Over-Engineering Simple Validations

```elixir
# ❌ Bad: Over-engineered
defmodule EmailValidator do
  defmodule Strategy do
    @callback validate(String.t()) :: boolean()
  end
  # ...unnecessary complexity...
end

# ✅ Good: Simple is better
defmodule Email do
  def new(address) when is_binary(address) do
    if Regex.match?(~r/@/, address) do
      {:ok, %Email{address: address}}
    else
      {:error, :invalid_email}
    end
  end
end
```

### ❌ 5. Not Using Result Tuples

```elixir
# ❌ Bad: Nil on error
def find_product(id) do
  # Returns nil on error - loses error information
  products[id]
end

# ✅ Good: Explicit error handling
def find_product(id) do
  case products[id] do
    nil -> {:error, :not_found}
    product -> {:ok, product}
  end
end
```

### ❌ 6. Ignoring Typespecs

```elixir
# ❌ Bad: No types
def calculate_total(order) do
  # What does this return? What does it accept?
end

# ✅ Good: Clear contract
@spec calculate_total(Order.t()) :: Money.t()
def calculate_total(%Order{} = order) do
  # Clear expectations
end
```

### ❌ 7. Mixing Ecto.Changeset with Pure Domain Validation

```elixir
# ❌ Bad: Ecto in domain
defmodule Order do
  def validate(attrs) do
    %Order{}
    |> Ecto.Changeset.cast(attrs, [:status])  # Ecto dependency!
    |> Ecto.Changeset.validate_required([:status])
  end
end

# ✅ Good: Pure validation, Ecto at boundary
defmodule Order do
  def new(status) when is_atom(status) do
    {:ok, %Order{status: status}}
  end
  def new(_), do: {:error, :invalid_status}
end

# Use Ecto.Changeset in infrastructure layer
defmodule OrderSchema do
  use Ecto.Schema
  import Ecto.Changeset
  
  def changeset(schema, attrs) do
    # Ecto validation here
  end
end
```

## Before/After Comparison

Let's see how domain modeling improves a messy Phoenix controller.

### ❌ Before: Mixed Concerns in Controller

```elixir
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  alias Shop.Repo
  alias Shop.Product
  
  def create(conn, %{"email" => email, "items" => items}) do
    # Email validation in controller
    if not String.match?(email, ~r/@/) do
      conn
      |> put_status(:bad_request)
      |> json(%{error: "Invalid email"})
    else
      # Create order directly with Repo
      order = Repo.insert!(%Order{
        customer_email: email,
        status: "pending",
        order_date: DateTime.utc_now()
      })
      
      # Process items with inline logic
      total = Enum.reduce(items, 0, fn item, acc ->
        product = Repo.get!(Product, item["product_id"])
        
        # Stock check inline
        if product.stock < item["quantity"] do
          # Problem: Already created order but now failing!
          raise "Insufficient stock"
        end
        
        # Create order item
        Repo.insert!(%OrderItem{
          order_id: order.id,
          product_id: product.id,
          quantity: item["quantity"],
          unit_price: product.price
        })
        
        # Update stock directly
        Repo.update!(Ecto.Changeset.change(product, 
          stock: product.stock - item["quantity"]
        ))
        
        acc + (product.price * item["quantity"])
      end)
      
      json(conn, %{
        order_id: order.id,
        total: total,
        message: "Order created"
      })
    end
  end
end
```

**Problems:**
- Validation mixed with controller logic
- Database operations interleaved with business logic
- No transaction safety (order created but items might fail)
- Hard to test without database
- No domain model expressing business rules
- Error handling is messy

### ✅ After: Clean Separation with Domain Models

```elixir
# Domain Layer - Pure business logic
defmodule Shop.Domain.Order do
  defstruct [:id, :customer_email, :items, :status, :order_date]
  
  alias Shop.Domain.{Email, OrderItem, Product}
  
  def new(id, email) do
    {:ok, %__MODULE__{
      id: id,
      customer_email: email,
      items: [],
      status: :pending,
      order_date: DateTime.utc_now()
    }}
  end
  
  def add_item(order, product, quantity) do
    with true <- Product.can_fulfill?(product, quantity),
         {:ok, item} <- OrderItem.new(product.id, product.price, quantity) do
      {:ok, %{order | items: [item | order.items]}}
    else
      false -> {:error, :insufficient_stock}
      error -> error
    end
  end
  
  def calculate_total(order) do
    Enum.reduce(order.items, Money.zero(:USD), fn item, acc ->
      {:ok, total} = Money.add(acc, OrderItem.subtotal(item))
      total
    end)
  end
end

# Application Layer - Orchestrates use case
defmodule Shop.Application.OrderService do
  alias Shop.Domain.{Order, Email, Product}
  alias Shop.Infrastructure.{OrderRepo, ProductRepo}
  
  def create_order(email_string, items_data) do
    with {:ok, email} <- Email.new(email_string),
         {:ok, order} <- Order.new(generate_id(), email),
         {:ok, order} <- add_items_to_order(order, items_data),
         {:ok, order} <- OrderRepo.save(order),
         :ok <- update_product_stock(items_data) do
      {:ok, order}
    end
  end
  
  defp add_items_to_order(order, items_data) do
    Enum.reduce_while(items_data, {:ok, order}, fn item_data, {:ok, acc_order} ->
      case ProductRepo.find(item_data["product_id"]) do
        {:ok, product} ->
          case Order.add_item(acc_order, product, item_data["quantity"]) do
            {:ok, updated_order} -> {:cont, {:ok, updated_order}}
            error -> {:halt, error}
          end
        error -> {:halt, error}
      end
    end)
  end
  
  defp update_product_stock(items_data) do
    # Update stock in repository layer
    # ...
  end
  
  defp generate_id do
    "ORD-#{System.unique_integer([:positive])}"
  end
end

# Infrastructure Layer - Database access
defmodule Shop.Infrastructure.OrderRepo do
  alias Shop.Domain.Order
  alias Shop.Infrastructure.OrderSchema
  
  def save(order) do
    # Convert domain model to Ecto schema and save
    # ...
    {:ok, order}
  end
end

# Controller - Thin HTTP layer
defmodule ShopWeb.OrderController do
  use ShopWeb, :controller
  alias Shop.Application.OrderService
  
  def create(conn, %{"email" => email, "items" => items}) do
    case OrderService.create_order(email, items) do
      {:ok, order} ->
        total = Order.calculate_total(order)
        
        conn
        |> put_status(:created)
        |> json(%{
          order_id: order.id,
          total: Money.format(total),
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
      
      {:error, reason} ->
        conn
        |> put_status(:internal_server_error)
        |> json(%{error: "Failed to create order: #{reason}"})
    end
  end
end
```

**Benefits:**
- ✅ Domain logic is pure and testable without database
- ✅ Clear separation: Domain → Application → Infrastructure → Controller
- ✅ Explicit error handling with result tuples
- ✅ Business rules in domain model, not scattered in controller
- ✅ Easy to test each layer independently
- ✅ Can add transaction logic in application layer
- ✅ Controller is thin—just HTTP translation

## Conclusion

Domain modeling in Elixir embraces functional programming principles to create clear, testable, and maintainable business logic. By using structs for data, pure functions for behavior, and pattern matching for control flow, we build domain models that are:

- **Explicit**: All data flow is visible
- **Immutable**: No hidden state changes
- **Testable**: Pure functions test without infrastructure
- **Composable**: Functions combine naturally
- **Resilient**: Pattern matching catches edge cases

### Key Takeaways

1. **Structs + Functions**: Data structures are separate from behavior
2. **Smart Constructors**: Always validate and return `{:ok, value}` or `{:error, reason}`
3. **Pattern Matching**: Use for validation, control flow, and expressing rules
4. **Typespecs**: Document contracts and enable static analysis
5. **Pure Functions**: Keep domain logic free of side effects
6. **Result Tuples**: Explicit error handling is better than exceptions
7. **Separation**: Domain never touches database, HTTP, or framework

### When to Use These Patterns

**Use domain modeling when:**
- ✅ Business rules are complex
- ✅ Multiple validations must be coordinated
- ✅ Logic will be reused across contexts
- ✅ Team needs clear documentation of business rules
- ✅ Code will evolve over time

**Simple CRUD might not need it:**
- Basic forms with simple validation can use changesets directly
- Not every module needs domain modeling
- Start simple, refactor when complexity warrants it

### Next Steps

Now that you understand domain modeling in Elixir:
- **Chapter 6**: Learn the Repository Pattern to persist domain models
- **Chapter 7**: Explore Application Services for orchestrating use cases
- **Chapter 11**: Discover testing strategies for each layer

Remember: The goal is not to add layers for the sake of architecture, but to make your code express business intent clearly and remain maintainable as requirements evolve.

## Further Reading

- [Elixir Documentation](https://elixir-lang.org/docs.html)
- [Domain-Driven Design](https://www.domainlanguage.com/ddd/) by Eric Evans
- [Designing Elixir Systems with OTP](https://pragprog.com/titles/jgotp/designing-elixir-systems-with-otp/) by James Edward Gray II and Bruce Tate
- [Programming Phoenix](https://pragprog.com/titles/phoenix14/programming-phoenix-1-4/) by Chris McCord, Bruce Tate, and José Valim

---

*This chapter is part of the Web Patterns documentation series. For more patterns and practices, see the other chapters in this guide.*

**Previous**: [Chapter 3: Domain Models in TypeScript](03-domain-models-typescript.md)  
**Next**: Chapter 5: Repository Pattern in React *(coming soon)*
