# Chapter 4: Domain Models in Elixir

## Introduction

In Elixir, domain modeling takes a functional approach rather than object-oriented. Instead of classes with methods and mutable state, we use structs for data and modules with pure functions for behavior. This chapter shows how to build rich domain models in Elixir that are immutable, composable, and easy to test.

## Core Concepts

### Structs vs Classes

In Elixir, we use structs to define domain entities and value objects:

```elixir
defmodule Shop.Domain.Product do
  defstruct [:id, :name, :description, :price, :stock_quantity, :active]
  
  @type t :: %__MODULE__{
    id: String.t(),
    name: String.t(),
    description: String.t(),
    price: Money.t(),
    stock_quantity: non_neg_integer(),
    active: boolean()
  }
end
```

### Pattern Matching for Validation

Elixir uses pattern matching and guard clauses for validation:

```elixir
def create(name, price, stock) when is_binary(name) and price > 0 and stock >= 0 do
  {:ok, %Product{name: name, price: price, stock: stock}}
end

def create(_name, _price, _stock) do
  {:error, :invalid_product_data}
end
```

[Continue with the full chapter content as provided above...]
