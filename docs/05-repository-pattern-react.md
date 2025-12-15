# Chapter 5: Repository Pattern in React

## Introduction

The Repository pattern is a powerful abstraction that sits between your application's business logic and data access layer. In React applications, it provides a clean separation between your UI components and the various data sources they depend on—whether that's a REST API, GraphQL endpoint, browser storage, or mock data during testing.

By introducing this layer of abstraction, you gain significant benefits:

- **Testability**: Components can be tested with mock repositories without network calls
- **Flexibility**: Swap data sources without changing component code
- **Clarity**: Components focus on UI logic, not data fetching details
- **Reusability**: Share data access logic across components
- **Type Safety**: Strong TypeScript contracts for data operations

This chapter explores how to implement the Repository pattern in React/TypeScript applications, showing practical examples that you can apply immediately to your codebase.

## Table of Contents

1. [The Problem: Scattered Data Access](#the-problem-scattered-data-access)
2. [The Solution: Repository Pattern](#the-solution-repository-pattern)
3. [Defining Repository Interfaces](#defining-repository-interfaces)
4. [Implementation Examples](#implementation-examples)
5. [React Integration](#react-integration)
6. [Testing with Repositories](#testing-with-repositories)
7. [Advanced Patterns](#advanced-patterns)
8. [Common Pitfalls](#common-pitfalls)
9. [When NOT to Use This Pattern](#when-not-to-use-this-pattern)
10. [Comparison with Data Fetching Libraries](#comparison-with-data-fetching-libraries)
11. [Conclusion](#conclusion)

## The Problem: Scattered Data Access

Without the Repository pattern, React components often become tightly coupled to specific data sources. Let's look at a typical example:

```typescript
// OrderDetails.tsx - Tightly coupled to fetch API
import React, { useState, useEffect } from 'react';

interface Props {
  orderId: string;
}

function OrderDetails({ orderId }: Props) {
  const [order, setOrder] = useState<any | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setLoading(true);
    fetch(`/api/orders/${orderId}`)
      .then(response => {
        if (!response.ok) {
          throw new Error('Failed to fetch order');
        }
        return response.json();
      })
      .then(data => {
        // Converting raw API response to domain model inline
        setOrder({
          id: data.id,
          customerEmail: data.customer_email,
          items: data.line_items.map((item: any) => ({
            productId: item.product_id,
            quantity: item.qty,
            price: item.unit_price
          })),
          total: data.total_amount
        });
        setLoading(false);
      })
      .catch(err => {
        setError(err);
        setLoading(false);
      });
  }, [orderId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!order) return <div>Order not found</div>;

  return (
    <div>
      <h2>Order {order.id}</h2>
      <p>Customer: {order.customerEmail}</p>
      <ul>
        {order.items.map((item: any) => (
          <li key={item.productId}>
            Product {item.productId}: {item.quantity} × ${item.price}
          </li>
        ))}
      </ul>
      <p>Total: ${order.total}</p>
    </div>
  );
}
```

**Problems with this approach:**

1. **Hard to test**: Testing requires mocking `fetch` globally or using MSW
2. **Tight coupling**: Component knows about API structure, endpoints, and transformations
3. **Duplication**: Every component that needs orders duplicates this logic
4. **Hard to change**: Switching from REST to GraphQL requires changing every component
5. **Mixed concerns**: UI rendering mixed with data fetching and transformation
6. **Type safety**: Using `any` types because API structure is inline

## The Solution: Repository Pattern

The Repository pattern provides an abstraction layer that:

1. **Defines contracts** for data operations via TypeScript interfaces
2. **Hides implementation details** of how data is fetched or stored
3. **Returns domain models** not raw API responses
4. **Enables dependency injection** for testing and flexibility
5. **Centralizes data access logic** for reusability

Here's how it looks conceptually:

```
┌─────────────────┐
│  UI Components  │
└────────┬────────┘
         │ uses
         ▼
┌─────────────────┐
│   Repository    │ ◄─── Interface (contract)
│   Interface     │
└────────┬────────┘
         │ implements
         ▼
┌─────────────────────────────────────┐
│  RestOrderRepository                │
│  GraphQLOrderRepository             │
│  LocalStorageOrderRepository        │
│  InMemoryOrderRepository (testing)  │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  Data Sources   │ (API, LocalStorage, Memory)
└─────────────────┘
```

Components depend on the interface, not the implementation. This is the essence of the Dependency Inversion Principle.

## Defining Repository Interfaces

Let's start by defining clear contracts for our data operations. We'll use the Order domain from Chapter 3.

### Order Domain Model

First, recall our domain model from Chapter 3:

```typescript
// domain/Order.ts
export enum OrderStatus {
  Pending = 'PENDING',
  Confirmed = 'CONFIRMED',
  Shipped = 'SHIPPED',
  Delivered = 'DELIVERED',
  Cancelled = 'CANCELLED'
}

export class OrderItem {
  constructor(
    public readonly productId: string,
    public readonly productName: string,
    public readonly quantity: number,
    public readonly unitPrice: number
  ) {
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    if (unitPrice < 0) {
      throw new Error('Unit price cannot be negative');
    }
  }

  get subtotal(): number {
    return this.quantity * this.unitPrice;
  }
}

export class Order {
  private constructor(
    private readonly _id: string,
    private readonly _customerId: string,
    private readonly _customerEmail: string,
    private readonly _items: OrderItem[],
    private _status: OrderStatus,
    private readonly _createdAt: Date
  ) {
    this.validate();
  }

  static create(
    id: string,
    customerId: string,
    customerEmail: string,
    items: OrderItem[]
  ): Order {
    return new Order(
      id,
      customerId,
      customerEmail,
      items,
      OrderStatus.Pending,
      new Date()
    );
  }

  static reconstitute(
    id: string,
    customerId: string,
    customerEmail: string,
    items: OrderItem[],
    status: OrderStatus,
    createdAt: Date
  ): Order {
    return new Order(id, customerId, customerEmail, items, status, createdAt);
  }

  // Getters
  get id(): string {
    return this._id;
  }

  get customerId(): string {
    return this._customerId;
  }

  get customerEmail(): string {
    return this._customerEmail;
  }

  get items(): readonly OrderItem[] {
    return this._items;
  }

  get status(): OrderStatus {
    return this._status;
  }

  get createdAt(): Date {
    return this._createdAt;
  }

  get total(): number {
    return this._items.reduce((sum, item) => sum + item.subtotal, 0);
  }

  // Business methods
  confirm(): void {
    if (this._status !== OrderStatus.Pending) {
      throw new Error('Only pending orders can be confirmed');
    }
    if (this._items.length === 0) {
      throw new Error('Cannot confirm order with no items');
    }
    this._status = OrderStatus.Confirmed;
  }

  cancel(): void {
    if (this._status === OrderStatus.Delivered) {
      throw new Error('Cannot cancel delivered order');
    }
    this._status = OrderStatus.Cancelled;
  }

  private validate(): void {
    if (!this._id) {
      throw new Error('Order ID is required');
    }
    if (!this._customerId) {
      throw new Error('Customer ID is required');
    }
    if (!this._customerEmail || !this._customerEmail.includes('@')) {
      throw new Error('Valid customer email is required');
    }
    if (this._items.length === 0) {
      throw new Error('Order must have at least one item');
    }
  }
}
```

### Repository Interface

Now let's define the repository interface:

```typescript
// repositories/OrderRepository.ts
import { Order } from '../domain/Order';

/**
 * Repository interface for Order persistence operations.
 * 
 * This interface defines the contract for accessing Order data.
 * Implementations can use REST APIs, GraphQL, LocalStorage, or any other data source.
 */
export interface OrderRepository {
  /**
   * Find an order by its unique identifier
   * @throws Error if order is not found
   */
  findById(id: string): Promise<Order>;

  /**
   * Save a new order or update an existing one
   * @returns The ID of the saved order
   */
  save(order: Order): Promise<string>;

  /**
   * Find all orders for a specific customer
   * @returns Array of orders, empty if none found
   */
  findByCustomer(customerId: string): Promise<Order[]>;

  /**
   * Find orders matching specific status
   * @returns Array of orders, empty if none found
   */
  findByStatus(status: OrderStatus): Promise<Order[]>;

  /**
   * Delete an order by ID
   * @returns true if deleted, false if not found
   */
  delete(id: string): Promise<boolean>;
}
```

**Key principles:**

- **Return domain models**: Methods return `Order` objects, not raw API data
- **Promise-based**: All operations are async (even in-memory for consistency)
- **Clear contracts**: Each method has a single, well-defined purpose
- **Error handling**: Document when exceptions should be thrown
- **Simple signatures**: Avoid complex query objects initially

## Implementation Examples

Now let's implement this interface for different data sources.

### REST API Implementation

```typescript
// repositories/RestOrderRepository.ts
import { Order, OrderItem, OrderStatus } from '../domain/Order';
import { OrderRepository } from './OrderRepository';

interface ApiOrderItem {
  product_id: string;
  product_name: string;
  qty: number;
  unit_price: number;
}

interface ApiOrder {
  id: string;
  customer_id: string;
  customer_email: string;
  line_items: ApiOrderItem[];
  status: string;
  created_at: string;
}

export class RestOrderRepository implements OrderRepository {
  constructor(
    private readonly baseUrl: string,
    private readonly authToken?: string
  ) {}

  async findById(id: string): Promise<Order> {
    const response = await fetch(`${this.baseUrl}/orders/${id}`, {
      headers: this.getHeaders()
    });

    if (!response.ok) {
      if (response.status === 404) {
        throw new Error(`Order ${id} not found`);
      }
      throw new Error(`Failed to fetch order: ${response.statusText}`);
    }

    const data: ApiOrder = await response.json();
    return this.toDomainModel(data);
  }

  async save(order: Order): Promise<string> {
    const payload = this.toApiFormat(order);
    const response = await fetch(`${this.baseUrl}/orders`, {
      method: 'POST',
      headers: {
        ...this.getHeaders(),
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(payload)
    });

    if (!response.ok) {
      throw new Error(`Failed to save order: ${response.statusText}`);
    }

    const result = await response.json();
    return result.id;
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    const response = await fetch(
      `${this.baseUrl}/orders?customer_id=${customerId}`,
      {
        headers: this.getHeaders()
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to fetch orders: ${response.statusText}`);
    }

    const data: ApiOrder[] = await response.json();
    return data.map(item => this.toDomainModel(item));
  }

  async findByStatus(status: OrderStatus): Promise<Order[]> {
    const response = await fetch(
      `${this.baseUrl}/orders?status=${status}`,
      {
        headers: this.getHeaders()
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to fetch orders: ${response.statusText}`);
    }

    const data: ApiOrder[] = await response.json();
    return data.map(item => this.toDomainModel(item));
  }

  async delete(id: string): Promise<boolean> {
    const response = await fetch(`${this.baseUrl}/orders/${id}`, {
      method: 'DELETE',
      headers: this.getHeaders()
    });

    if (response.status === 404) {
      return false;
    }

    if (!response.ok) {
      throw new Error(`Failed to delete order: ${response.statusText}`);
    }

    return true;
  }

  // Private helper methods

  private getHeaders(): Record<string, string> {
    const headers: Record<string, string> = {};
    if (this.authToken) {
      headers['Authorization'] = `Bearer ${this.authToken}`;
    }
    return headers;
  }

  private toDomainModel(apiOrder: ApiOrder): Order {
    const items = apiOrder.line_items.map(
      item => new OrderItem(
        item.product_id,
        item.product_name,
        item.qty,
        item.unit_price
      )
    );

    return Order.reconstitute(
      apiOrder.id,
      apiOrder.customer_id,
      apiOrder.customer_email,
      items,
      this.mapStatus(apiOrder.status),
      new Date(apiOrder.created_at)
    );
  }

  private toApiFormat(order: Order): any {
    return {
      id: order.id,
      customer_id: order.customerId,
      customer_email: order.customerEmail,
      line_items: order.items.map(item => ({
        product_id: item.productId,
        product_name: item.productName,
        qty: item.quantity,
        unit_price: item.unitPrice
      })),
      status: order.status,
      created_at: order.createdAt.toISOString()
    };
  }

  private mapStatus(apiStatus: string): OrderStatus {
    const statusMap: Record<string, OrderStatus> = {
      'PENDING': OrderStatus.Pending,
      'CONFIRMED': OrderStatus.Confirmed,
      'SHIPPED': OrderStatus.Shipped,
      'DELIVERED': OrderStatus.Delivered,
      'CANCELLED': OrderStatus.Cancelled
    };

    const status = statusMap[apiStatus.toUpperCase()];
    if (!status) {
      throw new Error(`Unknown order status: ${apiStatus}`);
    }
    return status;
  }
}
```

**Key aspects:**

- **Translation layer**: Converts between API format and domain models
- **Error handling**: Provides meaningful errors for different scenarios
- **Encapsulation**: API structure is hidden from consumers
- **Type safety**: Defines API types separately from domain types

### In-Memory Implementation (for Testing)

```typescript
// repositories/InMemoryOrderRepository.ts
import { Order, OrderStatus } from '../domain/Order';
import { OrderRepository } from './OrderRepository';

export class InMemoryOrderRepository implements OrderRepository {
  private orders = new Map<string, Order>();

  async findById(id: string): Promise<Order> {
    const order = this.orders.get(id);
    if (!order) {
      throw new Error(`Order ${id} not found`);
    }
    return order;
  }

  async save(order: Order): Promise<string> {
    this.orders.set(order.id, order);
    return order.id;
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    return Array.from(this.orders.values())
      .filter(order => order.customerId === customerId);
  }

  async findByStatus(status: OrderStatus): Promise<Order[]> {
    return Array.from(this.orders.values())
      .filter(order => order.status === status);
  }

  async delete(id: string): Promise<boolean> {
    return this.orders.delete(id);
  }

  // Test helper methods
  clear(): void {
    this.orders.clear();
  }

  count(): number {
    return this.orders.size;
  }

  getAll(): Order[] {
    return Array.from(this.orders.values());
  }
}
```

**Benefits for testing:**

- **Fast**: No network calls, synchronous logic
- **Isolated**: No external dependencies
- **Controllable**: Helper methods for test setup
- **Simple**: Minimal implementation complexity

### LocalStorage Implementation

For offline capabilities or client-side caching:

```typescript
// repositories/LocalStorageOrderRepository.ts
import { Order, OrderItem, OrderStatus } from '../domain/Order';
import { OrderRepository } from './OrderRepository';

export class LocalStorageOrderRepository implements OrderRepository {
  private readonly storageKey = 'orders';

  async findById(id: string): Promise<Order> {
    const orders = this.loadOrders();
    const order = orders.find(o => o.id === id);
    if (!order) {
      throw new Error(`Order ${id} not found`);
    }
    return this.deserialize(order);
  }

  async save(order: Order): Promise<string> {
    const orders = this.loadOrders();
    const index = orders.findIndex(o => o.id === order.id);
    
    const serialized = this.serialize(order);
    
    if (index >= 0) {
      orders[index] = serialized;
    } else {
      orders.push(serialized);
    }
    
    this.saveOrders(orders);
    return order.id;
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    const orders = this.loadOrders();
    return orders
      .filter(o => o.customerId === customerId)
      .map(o => this.deserialize(o));
  }

  async findByStatus(status: OrderStatus): Promise<Order[]> {
    const orders = this.loadOrders();
    return orders
      .filter(o => o.status === status)
      .map(o => this.deserialize(o));
  }

  async delete(id: string): Promise<boolean> {
    const orders = this.loadOrders();
    const index = orders.findIndex(o => o.id === id);
    
    if (index < 0) {
      return false;
    }
    
    orders.splice(index, 1);
    this.saveOrders(orders);
    return true;
  }

  // Private serialization helpers

  private loadOrders(): any[] {
    try {
      const data = localStorage.getItem(this.storageKey);
      return data ? JSON.parse(data) : [];
    } catch (error) {
      console.error('Failed to load orders from localStorage', error);
      return [];
    }
  }

  private saveOrders(orders: any[]): void {
    try {
      localStorage.setItem(this.storageKey, JSON.stringify(orders));
    } catch (error) {
      console.error('Failed to save orders to localStorage', error);
      throw new Error('Failed to save order');
    }
  }

  private serialize(order: Order): any {
    return {
      id: order.id,
      customerId: order.customerId,
      customerEmail: order.customerEmail,
      items: order.items.map(item => ({
        productId: item.productId,
        productName: item.productName,
        quantity: item.quantity,
        unitPrice: item.unitPrice
      })),
      status: order.status,
      createdAt: order.createdAt.toISOString()
    };
  }

  private deserialize(data: any): Order {
    const items = data.items.map(
      (item: any) => new OrderItem(
        item.productId,
        item.productName,
        item.quantity,
        item.unitPrice
      )
    );

    return Order.reconstitute(
      data.id,
      data.customerId,
      data.customerEmail,
      items,
      data.status as OrderStatus,
      new Date(data.createdAt)
    );
  }
}
```

## React Integration

Now let's integrate repositories into React components using Context API and custom hooks.

### Repository Context

```typescript
// context/OrderRepositoryContext.tsx
import React, { createContext, useContext, ReactNode } from 'react';
import { OrderRepository } from '../repositories/OrderRepository';

const OrderRepositoryContext = createContext<OrderRepository | null>(null);

interface Props {
  repository: OrderRepository;
  children: ReactNode;
}

export function OrderRepositoryProvider({ repository, children }: Props) {
  return (
    <OrderRepositoryContext.Provider value={repository}>
      {children}
    </OrderRepositoryContext.Provider>
  );
}

export function useOrderRepository(): OrderRepository {
  const repository = useContext(OrderRepositoryContext);
  if (!repository) {
    throw new Error(
      'useOrderRepository must be used within OrderRepositoryProvider'
    );
  }
  return repository;
}
```

### Application Setup

```typescript
// App.tsx
import React from 'react';
import { OrderRepositoryProvider } from './context/OrderRepositoryContext';
import { RestOrderRepository } from './repositories/RestOrderRepository';
import { OrderDetails } from './components/OrderDetails';

const orderRepository = new RestOrderRepository(
  process.env.REACT_APP_API_URL || 'http://localhost:3000/api',
  localStorage.getItem('auth_token') || undefined
);

function App() {
  return (
    <OrderRepositoryProvider repository={orderRepository}>
      <div className="App">
        <OrderDetails orderId="123" />
      </div>
    </OrderRepositoryProvider>
  );
}

export default App;
```

### Component Using Repository

Now let's see the **AFTER** version of our component:

```typescript
// components/OrderDetails.tsx
import React, { useState, useEffect } from 'react';
import { Order } from '../domain/Order';
import { useOrderRepository } from '../context/OrderRepositoryContext';

interface Props {
  orderId: string;
}

export function OrderDetails({ orderId }: Props) {
  const orderRepo = useOrderRepository();
  const [order, setOrder] = useState<Order | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let isMounted = true;

    setLoading(true);
    setError(null);

    orderRepo
      .findById(orderId)
      .then(order => {
        if (isMounted) {
          setOrder(order);
          setLoading(false);
        }
      })
      .catch(err => {
        if (isMounted) {
          setError(err);
          setLoading(false);
        }
      });

    return () => {
      isMounted = false;
    };
  }, [orderId, orderRepo]);

  if (loading) {
    return <div className="loading">Loading order...</div>;
  }

  if (error) {
    return <div className="error">Error: {error.message}</div>;
  }

  if (!order) {
    return <div className="not-found">Order not found</div>;
  }

  return (
    <div className="order-details">
      <h2>Order {order.id}</h2>
      <div className="order-info">
        <p><strong>Customer:</strong> {order.customerEmail}</p>
        <p><strong>Status:</strong> {order.status}</p>
        <p><strong>Created:</strong> {order.createdAt.toLocaleDateString()}</p>
      </div>
      
      <h3>Items</h3>
      <ul className="order-items">
        {order.items.map((item, index) => (
          <li key={index}>
            <span>{item.productName}</span>
            <span>{item.quantity} × ${item.unitPrice.toFixed(2)}</span>
            <span>${item.subtotal.toFixed(2)}</span>
          </li>
        ))}
      </ul>
      
      <div className="order-total">
        <strong>Total: ${order.total.toFixed(2)}</strong>
      </div>
    </div>
  );
}
```

**What improved:**

- ✅ No direct `fetch` calls
- ✅ No API structure knowledge
- ✅ Works with domain models (`Order`)
- ✅ Easy to test (inject mock repository)
- ✅ Type-safe throughout
- ✅ Single responsibility (UI only)

### Custom Hook for Data Fetching

For more reusable logic, create a custom hook:

```typescript
// hooks/useOrder.ts
import { useState, useEffect } from 'react';
import { Order } from '../domain/Order';
import { useOrderRepository } from '../context/OrderRepositoryContext';

interface UseOrderResult {
  order: Order | null;
  loading: boolean;
  error: Error | null;
  refresh: () => void;
}

export function useOrder(orderId: string): UseOrderResult {
  const orderRepo = useOrderRepository();
  const [order, setOrder] = useState<Order | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [refreshKey, setRefreshKey] = useState(0);

  useEffect(() => {
    let isMounted = true;

    setLoading(true);
    setError(null);

    orderRepo
      .findById(orderId)
      .then(order => {
        if (isMounted) {
          setOrder(order);
          setLoading(false);
        }
      })
      .catch(err => {
        if (isMounted) {
          setError(err);
          setLoading(false);
        }
      });

    return () => {
      isMounted = false;
    };
  }, [orderId, orderRepo, refreshKey]);

  const refresh = () => {
    setRefreshKey(prev => prev + 1);
  };

  return { order, loading, error, refresh };
}
```

**Simplified component:**

```typescript
// components/OrderDetails.tsx
import React from 'react';
import { useOrder } from '../hooks/useOrder';

interface Props {
  orderId: string;
}

export function OrderDetails({ orderId }: Props) {
  const { order, loading, error, refresh } = useOrder(orderId);

  if (loading) return <div>Loading order...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!order) return <div>Order not found</div>;

  return (
    <div className="order-details">
      <h2>Order {order.id}</h2>
      <p>Customer: {order.customerEmail}</p>
      <p>Total: ${order.total.toFixed(2)}</p>
      <button onClick={refresh}>Refresh</button>
    </div>
  );
}
```

## Testing with Repositories

The Repository pattern makes testing significantly easier.

### Component Test

```typescript
// components/__tests__/OrderDetails.test.tsx
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import { OrderDetails } from '../OrderDetails';
import { OrderRepositoryProvider } from '../../context/OrderRepositoryContext';
import { InMemoryOrderRepository } from '../../repositories/InMemoryOrderRepository';
import { Order, OrderItem, OrderStatus } from '../../domain/Order';

describe('OrderDetails', () => {
  let mockRepo: InMemoryOrderRepository;

  beforeEach(() => {
    mockRepo = new InMemoryOrderRepository();
  });

  it('displays order information', async () => {
    // Arrange
    const testOrder = Order.create(
      'order-123',
      'customer-1',
      'customer@example.com',
      [
        new OrderItem('prod-1', 'Widget', 2, 10.00),
        new OrderItem('prod-2', 'Gadget', 1, 25.00)
      ]
    );
    await mockRepo.save(testOrder);

    // Act
    render(
      <OrderRepositoryProvider repository={mockRepo}>
        <OrderDetails orderId="order-123" />
      </OrderRepositoryProvider>
    );

    // Assert
    await waitFor(() => {
      expect(screen.getByText('Order order-123')).toBeInTheDocument();
    });
    expect(screen.getByText(/customer@example\.com/)).toBeInTheDocument();
    expect(screen.getByText(/Widget/)).toBeInTheDocument();
    expect(screen.getByText(/Gadget/)).toBeInTheDocument();
    expect(screen.getByText(/Total: \$45\.00/)).toBeInTheDocument();
  });

  it('displays error when order not found', async () => {
    // Act
    render(
      <OrderRepositoryProvider repository={mockRepo}>
        <OrderDetails orderId="non-existent" />
      </OrderRepositoryProvider>
    );

    // Assert
    await waitFor(() => {
      expect(screen.getByText(/Error:/)).toBeInTheDocument();
    });
  });

  it('displays loading state initially', () => {
    // Act
    render(
      <OrderRepositoryProvider repository={mockRepo}>
        <OrderDetails orderId="order-123" />
      </OrderRepositoryProvider>
    );

    // Assert
    expect(screen.getByText(/Loading/)).toBeInTheDocument();
  });
});
```

### Testing with Mock Repository

```typescript
// repositories/__tests__/MockOrderRepository.ts
import { Order, OrderStatus } from '../../domain/Order';
import { OrderRepository } from '../OrderRepository';

export class MockOrderRepository implements OrderRepository {
  public findByIdCalls: string[] = [];
  public saveCalls: Order[] = [];

  constructor(
    private mockedOrders: Order[] = [],
    private shouldFail: boolean = false
  ) {}

  async findById(id: string): Promise<Order> {
    this.findByIdCalls.push(id);

    if (this.shouldFail) {
      throw new Error('Mock error');
    }

    const order = this.mockedOrders.find(o => o.id === id);
    if (!order) {
      throw new Error(`Order ${id} not found`);
    }
    return order;
  }

  async save(order: Order): Promise<string> {
    this.saveCalls.push(order);

    if (this.shouldFail) {
      throw new Error('Mock error');
    }

    return order.id;
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    if (this.shouldFail) {
      throw new Error('Mock error');
    }
    return this.mockedOrders.filter(o => o.customerId === customerId);
  }

  async findByStatus(status: OrderStatus): Promise<Order[]> {
    if (this.shouldFail) {
      throw new Error('Mock error');
    }
    return this.mockedOrders.filter(o => o.status === status);
  }

  async delete(id: string): Promise<boolean> {
    if (this.shouldFail) {
      throw new Error('Mock error');
    }
    const index = this.mockedOrders.findIndex(o => o.id === id);
    if (index >= 0) {
      this.mockedOrders.splice(index, 1);
      return true;
    }
    return false;
  }

  // Test helpers
  reset(): void {
    this.findByIdCalls = [];
    this.saveCalls = [];
  }

  wasCalledWith(id: string): boolean {
    return this.findByIdCalls.includes(id);
  }
}
```

**Usage in tests:**

```typescript
it('calls repository with correct ID', async () => {
  const mockRepo = new MockOrderRepository([testOrder]);
  
  render(
    <OrderRepositoryProvider repository={mockRepo}>
      <OrderDetails orderId="order-123" />
    </OrderRepositoryProvider>
  );

  await waitFor(() => {
    expect(mockRepo.wasCalledWith('order-123')).toBe(true);
  });
});
```

## Advanced Patterns

### Repository Factory

Create repositories based on environment or configuration:

```typescript
// repositories/RepositoryFactory.ts
import { OrderRepository } from './OrderRepository';
import { RestOrderRepository } from './RestOrderRepository';
import { InMemoryOrderRepository } from './InMemoryOrderRepository';
import { LocalStorageOrderRepository } from './LocalStorageOrderRepository';

export type RepositoryType = 'rest' | 'localStorage' | 'inMemory';

export interface RepositoryConfig {
  type: RepositoryType;
  apiUrl?: string;
  authToken?: string;
}

export class RepositoryFactory {
  static createOrderRepository(config: RepositoryConfig): OrderRepository {
    switch (config.type) {
      case 'rest':
        if (!config.apiUrl) {
          throw new Error('API URL required for REST repository');
        }
        return new RestOrderRepository(config.apiUrl, config.authToken);

      case 'localStorage':
        return new LocalStorageOrderRepository();

      case 'inMemory':
        return new InMemoryOrderRepository();

      default:
        throw new Error(`Unknown repository type: ${config.type}`);
    }
  }

  static createForEnvironment(): OrderRepository {
    const env = process.env.NODE_ENV;

    if (env === 'test') {
      return new InMemoryOrderRepository();
    }

    if (env === 'development') {
      return new RestOrderRepository(
        process.env.REACT_APP_API_URL || 'http://localhost:3000/api'
      );
    }

    return new RestOrderRepository(
      process.env.REACT_APP_API_URL || 'https://api.production.com'
    );
  }
}
```

**Usage:**

```typescript
// App.tsx
const orderRepository = RepositoryFactory.createForEnvironment();

function App() {
  return (
    <OrderRepositoryProvider repository={orderRepository}>
      {/* app content */}
    </OrderRepositoryProvider>
  );
}
```

### Caching Decorator

Add caching without changing the core repository:

```typescript
// repositories/CachedOrderRepository.ts
import { Order, OrderStatus } from '../domain/Order';
import { OrderRepository } from './OrderRepository';

interface CacheEntry<T> {
  data: T;
  timestamp: number;
}

export class CachedOrderRepository implements OrderRepository {
  private cache = new Map<string, CacheEntry<Order>>();
  private readonly ttl: number; // Time to live in milliseconds

  constructor(
    private readonly inner: OrderRepository,
    ttlSeconds: number = 60
  ) {
    this.ttl = ttlSeconds * 1000;
  }

  async findById(id: string): Promise<Order> {
    const cached = this.cache.get(id);
    
    if (cached && Date.now() - cached.timestamp < this.ttl) {
      console.log(`Cache hit for order ${id}`);
      return cached.data;
    }

    console.log(`Cache miss for order ${id}`);
    const order = await this.inner.findById(id);
    
    this.cache.set(id, {
      data: order,
      timestamp: Date.now()
    });

    return order;
  }

  async save(order: Order): Promise<string> {
    const id = await this.inner.save(order);
    
    // Invalidate cache for this order
    this.cache.delete(order.id);
    
    return id;
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    // Cache key for customer queries
    const cacheKey = `customer:${customerId}`;
    const cached = this.cache.get(cacheKey);

    if (cached && Date.now() - cached.timestamp < this.ttl) {
      return cached.data as any; // Type assertion needed for array
    }

    const orders = await this.inner.findByCustomer(customerId);
    
    this.cache.set(cacheKey, {
      data: orders as any,
      timestamp: Date.now()
    });

    return orders;
  }

  async findByStatus(status: OrderStatus): Promise<Order[]> {
    return this.inner.findByStatus(status);
  }

  async delete(id: string): Promise<boolean> {
    const result = await this.inner.delete(id);
    this.cache.delete(id);
    return result;
  }

  // Cache management
  clearCache(): void {
    this.cache.clear();
  }

  invalidate(id: string): void {
    this.cache.delete(id);
  }
}
```

**Usage:**

```typescript
const baseRepo = new RestOrderRepository(apiUrl, authToken);
const cachedRepo = new CachedOrderRepository(baseRepo, 60); // 60 second TTL

<OrderRepositoryProvider repository={cachedRepo}>
  {/* components */}
</OrderRepositoryProvider>
```

### Optimistic Updates

Handle optimistic UI updates with repositories:

```typescript
// hooks/useOptimisticOrder.ts
import { useState } from 'react';
import { Order } from '../domain/Order';
import { useOrderRepository } from '../context/OrderRepositoryContext';

export function useOptimisticOrder(initialOrder: Order | null) {
  const orderRepo = useOrderRepository();
  const [order, setOrder] = useState<Order | null>(initialOrder);
  const [isSaving, setIsSaving] = useState(false);

  const updateOrder = async (updatedOrder: Order) => {
    // Optimistic update
    const previousOrder = order;
    setOrder(updatedOrder);
    setIsSaving(true);

    try {
      await orderRepo.save(updatedOrder);
      setIsSaving(false);
    } catch (error) {
      // Rollback on error
      setOrder(previousOrder);
      setIsSaving(false);
      throw error;
    }
  };

  return { order, updateOrder, isSaving };
}
```

### Error Handling Decorator

Add centralized error handling:

```typescript
// repositories/ErrorHandlingOrderRepository.ts
import { Order, OrderStatus } from '../domain/Order';
import { OrderRepository } from './OrderRepository';

export class ErrorHandlingOrderRepository implements OrderRepository {
  constructor(
    private readonly inner: OrderRepository,
    private readonly onError?: (error: Error, operation: string) => void
  ) {}

  async findById(id: string): Promise<Order> {
    try {
      return await this.inner.findById(id);
    } catch (error) {
      this.handleError(error, 'findById', { id });
      throw error;
    }
  }

  async save(order: Order): Promise<string> {
    try {
      return await this.inner.save(order);
    } catch (error) {
      this.handleError(error, 'save', { orderId: order.id });
      throw error;
    }
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    try {
      return await this.inner.findByCustomer(customerId);
    } catch (error) {
      this.handleError(error, 'findByCustomer', { customerId });
      throw error;
    }
  }

  async findByStatus(status: OrderStatus): Promise<Order[]> {
    try {
      return await this.inner.findByStatus(status);
    } catch (error) {
      this.handleError(error, 'findByStatus', { status });
      throw error;
    }
  }

  async delete(id: string): Promise<boolean> {
    try {
      return await this.inner.delete(id);
    } catch (error) {
      this.handleError(error, 'delete', { id });
      throw error;
    }
  }

  private handleError(error: unknown, operation: string, context: any): void {
    const err = error instanceof Error ? error : new Error(String(error));
    
    // Log error
    console.error(`Repository error in ${operation}:`, err, context);
    
    // Call custom error handler if provided
    if (this.onError) {
      this.onError(err, operation);
    }
  }
}
```

## Common Pitfalls

### ❌ Pitfall 1: Query Explosion

**Problem:**

```typescript
interface OrderRepository {
  findById(id: string): Promise<Order>;
  findByCustomerId(customerId: string): Promise<Order[]>;
  findByCustomerIdAndStatus(customerId: string, status: OrderStatus): Promise<Order[]>;
  findByCustomerIdAndDateRange(customerId: string, start: Date, end: Date): Promise<Order[]>;
  findByStatusAndDateRange(status: OrderStatus, start: Date, end: Date): Promise<Order[]>;
  // This explodes quickly!
}
```

**Solution:** Use query objects or keep methods general:

```typescript
interface OrderQuery {
  customerId?: string;
  status?: OrderStatus;
  dateRange?: { start: Date; end: Date };
}

interface OrderRepository {
  findById(id: string): Promise<Order>;
  save(order: Order): Promise<string>;
  find(query: OrderQuery): Promise<Order[]>; // Flexible
}
```

### ❌ Pitfall 2: Returning Raw API Responses

**Problem:**

```typescript
async findById(id: string): Promise<any> {
  const response = await fetch(`/api/orders/${id}`);
  return response.json(); // Returns API structure!
}
```

**Solution:** Always return domain models:

```typescript
async findById(id: string): Promise<Order> {
  const response = await fetch(`/api/orders/${id}`);
  const data = await response.json();
  return this.toDomainModel(data); // Convert to Order
}
```

### ❌ Pitfall 3: Business Logic in Repository

**Problem:**

```typescript
async confirmOrder(id: string): Promise<void> {
  const order = await this.findById(id);
  
  // Business logic in repository!
  if (order.items.length === 0) {
    throw new Error('Cannot confirm empty order');
  }
  
  order.confirm();
  await this.save(order);
}
```

**Solution:** Keep repositories focused on data access:

```typescript
// Repository only handles persistence
async findById(id: string): Promise<Order>;
async save(order: Order): Promise<string>;

// Business logic in domain model
order.confirm(); // Throws if invalid
await repository.save(order);
```

### ❌ Pitfall 4: Not Handling Errors Properly

**Problem:**

```typescript
async findById(id: string): Promise<Order> {
  const response = await fetch(`/api/orders/${id}`);
  return this.toDomainModel(await response.json()); // No error handling!
}
```

**Solution:** Handle different error scenarios:

```typescript
async findById(id: string): Promise<Order> {
  try {
    const response = await fetch(`/api/orders/${id}`);
    
    if (!response.ok) {
      if (response.status === 404) {
        throw new Error(`Order ${id} not found`);
      }
      throw new Error(`Failed to fetch order: ${response.statusText}`);
    }
    
    const data = await response.json();
    return this.toDomainModel(data);
  } catch (error) {
    if (error instanceof Error) {
      throw error;
    }
    throw new Error('An unexpected error occurred');
  }
}
```

### ✅ Best Practices

1. **Keep repositories simple**: Focus on CRUD operations
2. **Return domain models**: Never expose API structures
3. **Use interfaces**: Enable dependency injection and testing
4. **Handle errors gracefully**: Provide meaningful error messages
5. **Be consistent**: All methods return Promises, even if sync
6. **Document contracts**: Use JSDoc to clarify behavior
7. **Avoid over-abstraction**: Don't add repositories for simple cases
8. **Use composition**: Decorators for cross-cutting concerns

## When NOT to Use This Pattern

The Repository pattern isn't always the right choice. Avoid it when:

### 1. Simple Applications

If your app is small and has a single data source:

```typescript
// Overkill for simple app
interface UserRepository {
  findById(id: string): Promise<User>;
}

// Just use fetch directly
function useUser(id: string) {
  return useQuery(['user', id], () =>
    fetch(`/api/users/${id}`).then(r => r.json())
  );
}
```

### 2. Using GraphQL with Normalized Cache

Apollo Client or Relay already provide abstraction:

```typescript
// Apollo Client handles this
const { data } = useQuery(GET_ORDER, {
  variables: { id: orderId }
});

// No need for repository layer
```

### 3. Simple CRUD with No Business Logic

If you're just passing data through:

```typescript
// Unnecessary abstraction
function OrderList() {
  const repo = useOrderRepository();
  const [orders, setOrders] = useState([]);
  
  useEffect(() => {
    repo.findAll().then(setOrders);
  }, []);
  
  // Just mapping API data to UI
}

// Better: Use React Query directly
function OrderList() {
  const { data: orders } = useQuery('orders', fetchOrders);
  // ...
}
```

### 4. Team Unfamiliarity

If your team isn't familiar with the pattern:

- **Start simpler**: Direct API calls
- **Educate gradually**: Introduce pattern with examples
- **Show benefits**: Refactor one component as proof
- **Get buy-in**: Ensure team understands value

## Comparison with Data Fetching Libraries

### Repository Pattern vs React Query

**React Query** excels at:
- Caching and cache invalidation
- Background refetching
- Optimistic updates
- Request deduplication

**Repository Pattern** excels at:
- Abstracting data source details
- Domain model transformation
- Swappable implementations
- Business logic separation

### Can They Work Together?

**Yes!** Use repositories inside React Query:

```typescript
// repositories/OrderRepository.ts
export class RestOrderRepository implements OrderRepository {
  async findById(id: string): Promise<Order> {
    // Implementation
  }
}

// hooks/useOrder.ts
import { useQuery } from 'react-query';
import { useOrderRepository } from '../context/OrderRepositoryContext';

export function useOrder(orderId: string) {
  const orderRepo = useOrderRepository();
  
  return useQuery(
    ['order', orderId],
    () => orderRepo.findById(orderId),
    {
      staleTime: 60000, // React Query's caching
      retry: 3
    }
  );
}

// Component
function OrderDetails({ orderId }: Props) {
  const { data: order, isLoading, error } = useOrder(orderId);
  
  // Clean and powerful!
}
```

**Benefits of combining:**
- Repository handles data source abstraction
- React Query handles caching and state management
- Best of both worlds
- Easy to test (mock repository)

### Repository Pattern vs SWR

Similar story with SWR:

```typescript
import useSWR from 'swr';
import { useOrderRepository } from '../context/OrderRepositoryContext';

export function useOrder(orderId: string) {
  const orderRepo = useOrderRepository();
  
  return useSWR(
    ['order', orderId],
    () => orderRepo.findById(orderId)
  );
}
```

### Decision Matrix

| Use Case | Solution |
|----------|----------|
| Simple app, single data source | Direct fetch or React Query |
| Multiple data sources (REST, GraphQL, LocalStorage) | Repository Pattern |
| Complex domain logic | Repository Pattern + Domain Models |
| Need advanced caching/refetching | React Query or SWR |
| Need to swap implementations (test/prod) | Repository Pattern |
| Best of both worlds | Repository + React Query/SWR |

## Conclusion

The Repository pattern brings clean separation between your React components and data access logic. By defining clear interfaces and hiding implementation details, you create applications that are:

- **Testable**: Components can be tested with mock repositories
- **Flexible**: Swap data sources without changing components  
- **Maintainable**: Data access logic is centralized and reusable
- **Type-safe**: Strong contracts enforced through TypeScript
- **Clear**: Components focus on UI, repositories focus on data

### Key Takeaways

1. **Define clear interfaces**: Use TypeScript interfaces for repository contracts
2. **Return domain models**: Never expose raw API responses
3. **Multiple implementations**: REST, InMemory, LocalStorage for different needs
4. **React integration**: Use Context API and custom hooks for dependency injection
5. **Compose with libraries**: Repositories work great with React Query/SWR
6. **Test easily**: InMemory repositories make testing fast and simple
7. **Start simple**: Don't over-engineer, add abstraction when it provides value

### When to Use

✅ Multiple data sources (REST, GraphQL, offline)  
✅ Complex domain models with business rules  
✅ Need to test components in isolation  
✅ Planning to swap implementations  
✅ Large team needing clear boundaries  

### When to Skip

❌ Simple CRUD with no domain logic  
❌ Using GraphQL with normalized cache  
❌ Small app with single data source  
❌ Team unfamiliar with pattern  

### Next Steps

Now that you understand the Repository pattern in React, you're ready to:

1. **Apply to your codebase**: Start with one entity and refactor incrementally
2. **Explore Chapter 6**: Learn how to implement repositories in Elixir
3. **Read Chapter 7**: Discover Application Services that orchestrate repositories
4. **Practice testing**: Build confidence with the InMemory pattern

The Repository pattern is a foundational piece of clean architecture. Combined with domain models (Chapter 3) and application services (Chapter 7), you'll have a complete toolkit for building maintainable React applications.

### Further Reading

- **Domain-Driven Design** by Eric Evans (Repository pattern origins)
- **Patterns of Enterprise Application Architecture** by Martin Fowler
- **Clean Architecture** by Robert C. Martin
- [React Query Documentation](https://tanstack.com/query/)
- [SWR Documentation](https://swr.vercel.app/)

---

*This guide is part of the Web Patterns documentation series. Continue your journey with [Chapter 6: Repository Pattern in Elixir](06-repository-pattern-elixir.md) or explore [Application Services in Chapter 7](07-application-services.md).*
