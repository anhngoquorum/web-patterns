# Chapter 3: Domain Models in TypeScript

## Introduction

Domain models are the heart of your business logic. They represent the core concepts, rules, and behaviors of your application domain. In TypeScript, we can leverage strong typing, classes, and modern language features to create rich, expressive domain models that are both maintainable and reliable.

This guide explores how to build robust domain models in TypeScript, focusing on practical patterns and best practices that help you express business rules clearly and prevent invalid states.

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Building Entities](#building-entities)
3. [Value Objects](#value-objects)
4. [Domain Model Examples](#domain-model-examples)
5. [Testing Strategies](#testing-strategies)
6. [Best Practices](#best-practices)

## Core Concepts

### What are Domain Models?

Domain models are object-oriented representations of your business domain. They encapsulate:

- **State**: The data that defines an entity or value object
- **Behavior**: The operations that can be performed
- **Invariants**: The rules that must always be true
- **Business Logic**: The domain-specific rules and workflows

### Entities vs Value Objects

**Entities** have:
- Unique identity that persists over time
- Mutable state (though controlled)
- Lifecycle management
- Examples: User, Order, Product

**Value Objects** have:
- No unique identity
- Immutable state
- Equality based on values, not identity
- Examples: Money, Email, Address

## Building Entities

### Product Entity

Let's start with a Product entity that demonstrates key patterns:

```typescript
export class Product {
  private constructor(
    private readonly _id: string,
    private _name: string,
    private _description: string,
    private _price: Money,
    private _stockQuantity: number,
    private _isActive: boolean = true
  ) {
    this.validate();
  }

  // Factory method for creating new products
  static create(
    id: string,
    name: string,
    description: string,
    price: Money,
    stockQuantity: number
  ): Product {
    return new Product(id, name, description, price, stockQuantity);
  }

  // Factory method for reconstituting from persistence
  static reconstitute(
    id: string,
    name: string,
    description: string,
    price: Money,
    stockQuantity: number,
    isActive: boolean
  ): Product {
    return new Product(id, name, description, price, stockQuantity, isActive);
  }

  // Getters
  get id(): string {
    return this._id;
  }

  get name(): string {
    return this._name;
  }

  get description(): string {
    return this._description;
  }

  get price(): Money {
    return this._price;
  }

  get stockQuantity(): number {
    return this._stockQuantity;
  }

  get isActive(): boolean {
    return this._isActive;
  }

  get isInStock(): boolean {
    return this._stockQuantity > 0;
  }

  // Business methods
  updatePrice(newPrice: Money): void {
    if (newPrice.amount <= 0) {
      throw new Error('Price must be greater than zero');
    }
    this._price = newPrice;
  }

  updateName(newName: string): void {
    if (!newName || newName.trim().length === 0) {
      throw new Error('Product name cannot be empty');
    }
    if (newName.length > 200) {
      throw new Error('Product name cannot exceed 200 characters');
    }
    this._name = newName.trim();
  }

  addStock(quantity: number): void {
    if (quantity <= 0) {
      throw new Error('Quantity to add must be positive');
    }
    this._stockQuantity += quantity;
  }

  removeStock(quantity: number): void {
    if (quantity <= 0) {
      throw new Error('Quantity to remove must be positive');
    }
    if (quantity > this._stockQuantity) {
      throw new Error(`Insufficient stock. Available: ${this._stockQuantity}, Requested: ${quantity}`);
    }
    this._stockQuantity -= quantity;
  }

  deactivate(): void {
    this._isActive = false;
  }

  activate(): void {
    this._isActive = true;
  }

  canFulfillOrder(quantity: number): boolean {
    return this._isActive && this._stockQuantity >= quantity;
  }

  // Self-validation
  private validate(): void {
    if (!this._id || this._id.trim().length === 0) {
      throw new Error('Product ID is required');
    }
    if (!this._name || this._name.trim().length === 0) {
      throw new Error('Product name is required');
    }
    if (this._name.length > 200) {
      throw new Error('Product name cannot exceed 200 characters');
    }
    if (this._price.amount <= 0) {
      throw new Error('Product price must be greater than zero');
    }
    if (this._stockQuantity < 0) {
      throw new Error('Stock quantity cannot be negative');
    }
  }
}
```

### Order Entity

The Order entity demonstrates aggregates and managing collections:

```typescript
export class Order {
  private _items: OrderItem[] = [];
  private _status: OrderStatus;

  private constructor(
    private readonly _id: string,
    private readonly _customerId: string,
    private readonly _orderDate: Date,
    items: OrderItem[] = [],
    status: OrderStatus = OrderStatus.Pending
  ) {
    this._items = [...items];
    this._status = status;
    this.validate();
  }

  static create(id: string, customerId: string): Order {
    return new Order(id, customerId, new Date());
  }

  static reconstitute(
    id: string,
    customerId: string,
    orderDate: Date,
    items: OrderItem[],
    status: OrderStatus
  ): Order {
    return new Order(id, customerId, orderDate, items, status);
  }

  // Getters
  get id(): string {
    return this._id;
  }

  get customerId(): string {
    return this._customerId;
  }

  get orderDate(): Date {
    return new Date(this._orderDate);
  }

  get items(): readonly OrderItem[] {
    return [...this._items];
  }

  get status(): OrderStatus {
    return this._status;
  }

  get totalAmount(): Money {
    const total = this._items.reduce(
      (sum, item) => sum + item.subtotal.amount,
      0
    );
    return Money.fromAmount(total, this.getCurrency());
  }

  get itemCount(): number {
    return this._items.reduce((sum, item) => sum + item.quantity, 0);
  }

  // Business methods
  addItem(product: Product, quantity: number): void {
    this.ensureOrderIsModifiable();

    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }

    if (!product.canFulfillOrder(quantity)) {
      throw new Error(`Cannot fulfill order for product ${product.name}`);
    }

    const existingItem = this._items.find(item => item.productId === product.id);
    
    if (existingItem) {
      existingItem.updateQuantity(existingItem.quantity + quantity);
    } else {
      this._items.push(OrderItem.create(product.id, product.price, quantity));
    }
  }

  removeItem(productId: string): void {
    this.ensureOrderIsModifiable();
    
    const index = this._items.findIndex(item => item.productId === productId);
    if (index === -1) {
      throw new Error(`Product ${productId} not found in order`);
    }
    
    this._items.splice(index, 1);
  }

  updateItemQuantity(productId: string, newQuantity: number): void {
    this.ensureOrderIsModifiable();
    
    const item = this._items.find(item => item.productId === productId);
    if (!item) {
      throw new Error(`Product ${productId} not found in order`);
    }
    
    if (newQuantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    
    item.updateQuantity(newQuantity);
  }

  confirm(): void {
    if (this._status !== OrderStatus.Pending) {
      throw new Error('Only pending orders can be confirmed');
    }
    if (this._items.length === 0) {
      throw new Error('Cannot confirm order with no items');
    }
    this._status = OrderStatus.Confirmed;
  }

  ship(): void {
    if (this._status !== OrderStatus.Confirmed) {
      throw new Error('Only confirmed orders can be shipped');
    }
    this._status = OrderStatus.Shipped;
  }

  deliver(): void {
    if (this._status !== OrderStatus.Shipped) {
      throw new Error('Only shipped orders can be delivered');
    }
    this._status = OrderStatus.Delivered;
  }

  cancel(): void {
    if (this._status === OrderStatus.Delivered) {
      throw new Error('Cannot cancel delivered orders');
    }
    if (this._status === OrderStatus.Cancelled) {
      throw new Error('Order is already cancelled');
    }
    this._status = OrderStatus.Cancelled;
  }

  private ensureOrderIsModifiable(): void {
    if (this._status !== OrderStatus.Pending) {
      throw new Error('Cannot modify order that is not in pending status');
    }
  }

  private getCurrency(): string {
    if (this._items.length === 0) {
      return 'USD'; // Default currency
    }
    return this._items[0].unitPrice.currency;
  }

  private validate(): void {
    if (!this._id || this._id.trim().length === 0) {
      throw new Error('Order ID is required');
    }
    if (!this._customerId || this._customerId.trim().length === 0) {
      throw new Error('Customer ID is required');
    }
  }
}

export enum OrderStatus {
  Pending = 'PENDING',
  Confirmed = 'CONFIRMED',
  Shipped = 'SHIPPED',
  Delivered = 'DELIVERED',
  Cancelled = 'CANCELLED'
}
```

### OrderItem Entity

```typescript
export class OrderItem {
  private constructor(
    private readonly _productId: string,
    private readonly _unitPrice: Money,
    private _quantity: number
  ) {
    this.validate();
  }

  static create(productId: string, unitPrice: Money, quantity: number): OrderItem {
    return new OrderItem(productId, unitPrice, quantity);
  }

  get productId(): string {
    return this._productId;
  }

  get unitPrice(): Money {
    return this._unitPrice;
  }

  get quantity(): number {
    return this._quantity;
  }

  get subtotal(): Money {
    return this._unitPrice.multiply(this._quantity);
  }

  updateQuantity(newQuantity: number): void {
    if (newQuantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    this._quantity = newQuantity;
  }

  private validate(): void {
    if (!this._productId || this._productId.trim().length === 0) {
      throw new Error('Product ID is required');
    }
    if (this._quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    if (this._unitPrice.amount <= 0) {
      throw new Error('Unit price must be positive');
    }
  }
}
```

## Value Objects

Value objects are immutable objects defined by their values, not their identity. They're perfect for concepts like Money, Email, and Address.

### Money Value Object

```typescript
export class Money {
  private constructor(
    private readonly _amount: number,
    private readonly _currency: string
  ) {
    this.validate();
  }

  static fromAmount(amount: number, currency: string = 'USD'): Money {
    return new Money(amount, currency);
  }

  static zero(currency: string = 'USD'): Money {
    return new Money(0, currency);
  }

  get amount(): number {
    return this._amount;
  }

  get currency(): string {
    return this._currency;
  }

  // Arithmetic operations
  add(other: Money): Money {
    this.ensureSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other: Money): Money {
    this.ensureSameCurrency(other);
    return new Money(this._amount - other._amount, this._currency);
  }

  multiply(factor: number): Money {
    return new Money(this._amount * factor, this._currency);
  }

  divide(divisor: number): Money {
    if (divisor === 0) {
      throw new Error('Cannot divide by zero');
    }
    return new Money(this._amount / divisor, this._currency);
  }

  // Comparison operations
  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  isGreaterThan(other: Money): boolean {
    this.ensureSameCurrency(other);
    return this._amount > other._amount;
  }

  isLessThan(other: Money): boolean {
    this.ensureSameCurrency(other);
    return this._amount < other._amount;
  }

  isZero(): boolean {
    return this._amount === 0;
  }

  isPositive(): boolean {
    return this._amount > 0;
  }

  isNegative(): boolean {
    return this._amount < 0;
  }

  // Formatting
  toString(): string {
    return `${this._currency} ${this._amount.toFixed(2)}`;
  }

  format(locale: string = 'en-US'): string {
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency: this._currency
    }).format(this._amount);
  }

  private ensureSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new Error(`Cannot operate on different currencies: ${this._currency} and ${other._currency}`);
    }
  }

  private validate(): void {
    if (!this._currency || this._currency.trim().length === 0) {
      throw new Error('Currency is required');
    }
    if (this._currency.length !== 3) {
      throw new Error('Currency must be a 3-letter ISO code');
    }
    if (!Number.isFinite(this._amount)) {
      throw new Error('Amount must be a finite number');
    }
  }
}
```

### Email Value Object

```typescript
export class Email {
  private static readonly EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

  private constructor(private readonly _value: string) {
    this.validate();
  }

  static create(email: string): Email {
    return new Email(email.toLowerCase().trim());
  }

  get value(): string {
    return this._value;
  }

  get domain(): string {
    return this._value.split('@')[1];
  }

  get localPart(): string {
    return this._value.split('@')[0];
  }

  equals(other: Email): boolean {
    return this._value === other._value;
  }

  toString(): string {
    return this._value;
  }

  private validate(): void {
    if (!this._value || this._value.trim().length === 0) {
      throw new Error('Email address is required');
    }
    if (!Email.EMAIL_REGEX.test(this._value)) {
      throw new Error('Invalid email address format');
    }
    if (this._value.length > 254) {
      throw new Error('Email address is too long');
    }
  }
}
```

### Address Value Object

```typescript
export class Address {
  private constructor(
    private readonly _street: string,
    private readonly _city: string,
    private readonly _state: string,
    private readonly _postalCode: string,
    private readonly _country: string
  ) {
    this.validate();
  }

  static create(
    street: string,
    city: string,
    state: string,
    postalCode: string,
    country: string
  ): Address {
    return new Address(
      street.trim(),
      city.trim(),
      state.trim(),
      postalCode.trim(),
      country.trim()
    );
  }

  get street(): string {
    return this._street;
  }

  get city(): string {
    return this._city;
  }

  get state(): string {
    return this._state;
  }

  get postalCode(): string {
    return this._postalCode;
  }

  get country(): string {
    return this._country;
  }

  equals(other: Address): boolean {
    return (
      this._street === other._street &&
      this._city === other._city &&
      this._state === other._state &&
      this._postalCode === other._postalCode &&
      this._country === other._country
    );
  }

  toString(): string {
    return `${this._street}, ${this._city}, ${this._state} ${this._postalCode}, ${this._country}`;
  }

  private validate(): void {
    if (!this._street || this._street.length === 0) {
      throw new Error('Street address is required');
    }
    if (!this._city || this._city.length === 0) {
      throw new Error('City is required');
    }
    if (!this._state || this._state.length === 0) {
      throw new Error('State is required');
    }
    if (!this._postalCode || this._postalCode.length === 0) {
      throw new Error('Postal code is required');
    }
    if (!this._country || this._country.length === 0) {
      throw new Error('Country is required');
    }
  }
}
```

## Domain Model Examples

### Complete E-commerce Example

```typescript
// Customer entity
export class Customer {
  private _addresses: Address[] = [];

  private constructor(
    private readonly _id: string,
    private _email: Email,
    private _firstName: string,
    private _lastName: string,
    addresses: Address[] = []
  ) {
    this._addresses = [...addresses];
    this.validate();
  }

  static create(
    id: string,
    email: Email,
    firstName: string,
    lastName: string
  ): Customer {
    return new Customer(id, email, firstName, lastName);
  }

  get id(): string {
    return this._id;
  }

  get email(): Email {
    return this._email;
  }

  get fullName(): string {
    return `${this._firstName} ${this._lastName}`;
  }

  get addresses(): readonly Address[] {
    return [...this._addresses];
  }

  addAddress(address: Address): void {
    if (this._addresses.some(a => a.equals(address))) {
      throw new Error('Address already exists for this customer');
    }
    this._addresses.push(address);
  }

  updateEmail(newEmail: Email): void {
    this._email = newEmail;
  }

  private validate(): void {
    if (!this._id || this._id.trim().length === 0) {
      throw new Error('Customer ID is required');
    }
    if (!this._firstName || this._firstName.trim().length === 0) {
      throw new Error('First name is required');
    }
    if (!this._lastName || this._lastName.trim().length === 0) {
      throw new Error('Last name is required');
    }
  }
}

// Shopping Cart example
export class ShoppingCart {
  private _items: Map<string, CartItem> = new Map();

  private constructor(
    private readonly _customerId: string,
    private readonly _createdAt: Date
  ) {
    this.validate();
  }

  static create(customerId: string): ShoppingCart {
    return new ShoppingCart(customerId, new Date());
  }

  get customerId(): string {
    return this._customerId;
  }

  get items(): CartItem[] {
    return Array.from(this._items.values());
  }

  get totalAmount(): Money {
    const items = Array.from(this._items.values());
    if (items.length === 0) {
      return Money.zero();
    }
    return items.reduce(
      (total, item) => total.add(item.subtotal),
      Money.zero(items[0].unitPrice.currency)
    );
  }

  get itemCount(): number {
    return Array.from(this._items.values()).reduce(
      (sum, item) => sum + item.quantity,
      0
    );
  }

  addItem(product: Product, quantity: number): void {
    if (!product.canFulfillOrder(quantity)) {
      throw new Error(`Cannot add product ${product.name} to cart`);
    }

    const existingItem = this._items.get(product.id);
    if (existingItem) {
      existingItem.updateQuantity(existingItem.quantity + quantity);
    } else {
      this._items.set(
        product.id,
        CartItem.create(product.id, product.name, product.price, quantity)
      );
    }
  }

  removeItem(productId: string): void {
    if (!this._items.has(productId)) {
      throw new Error(`Product ${productId} not found in cart`);
    }
    this._items.delete(productId);
  }

  updateQuantity(productId: string, newQuantity: number): void {
    const item = this._items.get(productId);
    if (!item) {
      throw new Error(`Product ${productId} not found in cart`);
    }
    if (newQuantity <= 0) {
      this.removeItem(productId);
    } else {
      item.updateQuantity(newQuantity);
    }
  }

  clear(): void {
    this._items.clear();
  }

  isEmpty(): boolean {
    return this._items.size === 0;
  }

  checkout(): Order {
    if (this.isEmpty()) {
      throw new Error('Cannot checkout empty cart');
    }

    const order = Order.create(
      this.generateOrderId(),
      this._customerId
    );

    // This would typically involve fetching full product details
    // For now, we'll assume we have them
    this.items.forEach(cartItem => {
      // In real scenario, fetch product and verify availability
      // order.addItem(product, cartItem.quantity);
    });

    return order;
  }

  private generateOrderId(): string {
    return `ORD-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  private validate(): void {
    if (!this._customerId || this._customerId.trim().length === 0) {
      throw new Error('Customer ID is required');
    }
  }
}

class CartItem {
  private constructor(
    private readonly _productId: string,
    private readonly _productName: string,
    private readonly _unitPrice: Money,
    private _quantity: number
  ) {
    this.validate();
  }

  static create(
    productId: string,
    productName: string,
    unitPrice: Money,
    quantity: number
  ): CartItem {
    return new CartItem(productId, productName, unitPrice, quantity);
  }

  get productId(): string {
    return this._productId;
  }

  get productName(): string {
    return this._productName;
  }

  get unitPrice(): Money {
    return this._unitPrice;
  }

  get quantity(): number {
    return this._quantity;
  }

  get subtotal(): Money {
    return this._unitPrice.multiply(this._quantity);
  }

  updateQuantity(newQuantity: number): void {
    if (newQuantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    this._quantity = newQuantity;
  }

  private validate(): void {
    if (!this._productId) {
      throw new Error('Product ID is required');
    }
    if (this._quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
  }
}
```

## Testing Strategies

### Unit Testing Domain Models

Domain models should be thoroughly tested as they contain your core business logic.

```typescript
import { describe, it, expect } from '@jest/globals';

describe('Product', () => {
  describe('creation', () => {
    it('should create a valid product', () => {
      const price = Money.fromAmount(99.99, 'USD');
      const product = Product.create('P1', 'Laptop', 'Gaming laptop', price, 10);

      expect(product.id).toBe('P1');
      expect(product.name).toBe('Laptop');
      expect(product.price.amount).toBe(99.99);
      expect(product.stockQuantity).toBe(10);
      expect(product.isActive).toBe(true);
    });

    it('should reject invalid product names', () => {
      const price = Money.fromAmount(99.99, 'USD');
      
      expect(() => {
        Product.create('P1', '', 'Description', price, 10);
      }).toThrow('Product name is required');

      expect(() => {
        Product.create('P1', 'A'.repeat(201), 'Description', price, 10);
      }).toThrow('Product name cannot exceed 200 characters');
    });

    it('should reject negative stock quantities', () => {
      const price = Money.fromAmount(99.99, 'USD');
      
      expect(() => {
        Product.create('P1', 'Laptop', 'Description', price, -1);
      }).toThrow('Stock quantity cannot be negative');
    });
  });

  describe('stock management', () => {
    it('should add stock correctly', () => {
      const price = Money.fromAmount(99.99, 'USD');
      const product = Product.create('P1', 'Laptop', 'Gaming laptop', price, 10);

      product.addStock(5);
      expect(product.stockQuantity).toBe(15);
    });

    it('should remove stock correctly', () => {
      const price = Money.fromAmount(99.99, 'USD');
      const product = Product.create('P1', 'Laptop', 'Gaming laptop', price, 10);

      product.removeStock(3);
      expect(product.stockQuantity).toBe(7);
    });

    it('should throw error when removing more stock than available', () => {
      const price = Money.fromAmount(99.99, 'USD');
      const product = Product.create('P1', 'Laptop', 'Gaming laptop', price, 10);

      expect(() => {
        product.removeStock(15);
      }).toThrow('Insufficient stock');
    });
  });

  describe('canFulfillOrder', () => {
    it('should return true when product is active and has sufficient stock', () => {
      const price = Money.fromAmount(99.99, 'USD');
      const product = Product.create('P1', 'Laptop', 'Gaming laptop', price, 10);

      expect(product.canFulfillOrder(5)).toBe(true);
      expect(product.canFulfillOrder(10)).toBe(true);
    });

    it('should return false when insufficient stock', () => {
      const price = Money.fromAmount(99.99, 'USD');
      const product = Product.create('P1', 'Laptop', 'Gaming laptop', price, 10);

      expect(product.canFulfillOrder(11)).toBe(false);
    });

    it('should return false when product is inactive', () => {
      const price = Money.fromAmount(99.99, 'USD');
      const product = Product.create('P1', 'Laptop', 'Gaming laptop', price, 10);
      product.deactivate();

      expect(product.canFulfillOrder(5)).toBe(false);
    });
  });
});

describe('Money', () => {
  describe('arithmetic operations', () => {
    it('should add money correctly', () => {
      const money1 = Money.fromAmount(10, 'USD');
      const money2 = Money.fromAmount(5, 'USD');
      const result = money1.add(money2);

      expect(result.amount).toBe(15);
      expect(result.currency).toBe('USD');
    });

    it('should subtract money correctly', () => {
      const money1 = Money.fromAmount(10, 'USD');
      const money2 = Money.fromAmount(3, 'USD');
      const result = money1.subtract(money2);

      expect(result.amount).toBe(7);
    });

    it('should multiply money correctly', () => {
      const money = Money.fromAmount(10, 'USD');
      const result = money.multiply(3);

      expect(result.amount).toBe(30);
    });

    it('should throw error when operating on different currencies', () => {
      const usd = Money.fromAmount(10, 'USD');
      const eur = Money.fromAmount(10, 'EUR');

      expect(() => {
        usd.add(eur);
      }).toThrow('Cannot operate on different currencies');
    });
  });

  describe('comparison operations', () => {
    it('should compare money values correctly', () => {
      const money1 = Money.fromAmount(10, 'USD');
      const money2 = Money.fromAmount(5, 'USD');

      expect(money1.isGreaterThan(money2)).toBe(true);
      expect(money2.isLessThan(money1)).toBe(true);
      expect(money1.equals(money2)).toBe(false);
    });

    it('should identify zero, positive, and negative values', () => {
      const zero = Money.zero('USD');
      const positive = Money.fromAmount(10, 'USD');
      const negative = Money.fromAmount(-5, 'USD');

      expect(zero.isZero()).toBe(true);
      expect(positive.isPositive()).toBe(true);
      expect(negative.isNegative()).toBe(true);
    });
  });
});

describe('Email', () => {
  it('should create valid email', () => {
    const email = Email.create('user@example.com');
    expect(email.value).toBe('user@example.com');
    expect(email.domain).toBe('example.com');
    expect(email.localPart).toBe('user');
  });

  it('should normalize email to lowercase', () => {
    const email = Email.create('User@Example.COM');
    expect(email.value).toBe('user@example.com');
  });

  it('should reject invalid email formats', () => {
    expect(() => Email.create('invalid')).toThrow('Invalid email address format');
    expect(() => Email.create('@example.com')).toThrow('Invalid email address format');
    expect(() => Email.create('user@')).toThrow('Invalid email address format');
    expect(() => Email.create('')).toThrow('Email address is required');
  });
});

describe('Order', () => {
  describe('order lifecycle', () => {
    it('should create order in pending status', () => {
      const order = Order.create('O1', 'C1');
      expect(order.status).toBe(OrderStatus.Pending);
    });

    it('should transition through valid statuses', () => {
      const order = Order.create('O1', 'C1');
      const product = Product.create(
        'P1',
        'Laptop',
        'Gaming laptop',
        Money.fromAmount(999, 'USD'),
        10
      );

      order.addItem(product, 1);
      
      order.confirm();
      expect(order.status).toBe(OrderStatus.Confirmed);

      order.ship();
      expect(order.status).toBe(OrderStatus.Shipped);

      order.deliver();
      expect(order.status).toBe(OrderStatus.Delivered);
    });

    it('should reject invalid status transitions', () => {
      const order = Order.create('O1', 'C1');

      expect(() => order.ship()).toThrow('Only confirmed orders can be shipped');
      expect(() => order.deliver()).toThrow('Only shipped orders can be delivered');
    });

    it('should not allow modifying non-pending orders', () => {
      const order = Order.create('O1', 'C1');
      const product = Product.create(
        'P1',
        'Laptop',
        'Gaming laptop',
        Money.fromAmount(999, 'USD'),
        10
      );

      order.addItem(product, 1);
      order.confirm();

      expect(() => {
        order.addItem(product, 1);
      }).toThrow('Cannot modify order that is not in pending status');
    });
  });

  describe('order items management', () => {
    it('should add items to order', () => {
      const order = Order.create('O1', 'C1');
      const product = Product.create(
        'P1',
        'Laptop',
        'Gaming laptop',
        Money.fromAmount(999, 'USD'),
        10
      );

      order.addItem(product, 2);
      expect(order.items.length).toBe(1);
      expect(order.items[0].quantity).toBe(2);
    });

    it('should combine quantities for same product', () => {
      const order = Order.create('O1', 'C1');
      const product = Product.create(
        'P1',
        'Laptop',
        'Gaming laptop',
        Money.fromAmount(999, 'USD'),
        10
      );

      order.addItem(product, 2);
      order.addItem(product, 3);

      expect(order.items.length).toBe(1);
      expect(order.items[0].quantity).toBe(5);
    });

    it('should calculate total correctly', () => {
      const order = Order.create('O1', 'C1');
      const product1 = Product.create(
        'P1',
        'Laptop',
        'Gaming laptop',
        Money.fromAmount(1000, 'USD'),
        10
      );
      const product2 = Product.create(
        'P2',
        'Mouse',
        'Gaming mouse',
        Money.fromAmount(50, 'USD'),
        20
      );

      order.addItem(product1, 1);
      order.addItem(product2, 2);

      expect(order.totalAmount.amount).toBe(1100); // 1000 + (50 * 2)
    });
  });
});
```

### Integration Testing

```typescript
describe('E-commerce Integration', () => {
  it('should complete full order flow', () => {
    // Create customer
    const customer = Customer.create(
      'C1',
      Email.create('customer@example.com'),
      'John',
      'Doe'
    );

    // Create products
    const laptop = Product.create(
      'P1',
      'Laptop',
      'Gaming laptop',
      Money.fromAmount(1200, 'USD'),
      5
    );

    const mouse = Product.create(
      'P2',
      'Mouse',
      'Gaming mouse',
      Money.fromAmount(50, 'USD'),
      20
    );

    // Create shopping cart
    const cart = ShoppingCart.create(customer.id);
    cart.addItem(laptop, 1);
    cart.addItem(mouse, 2);

    // Verify cart
    expect(cart.itemCount).toBe(3);
    expect(cart.totalAmount.amount).toBe(1300);

    // Create order from cart
    const order = Order.create('O1', customer.id);
    order.addItem(laptop, 1);
    order.addItem(mouse, 2);

    // Process order
    order.confirm();
    
    // Update product stock
    laptop.removeStock(1);
    mouse.removeStock(2);

    // Ship and deliver
    order.ship();
    order.deliver();

    // Verify final state
    expect(order.status).toBe(OrderStatus.Delivered);
    expect(laptop.stockQuantity).toBe(4);
    expect(mouse.stockQuantity).toBe(18);
  });
});
```

## Best Practices

### 1. Self-Validation

Always validate your domain objects in their constructors:

```typescript
class Product {
  private constructor(/* ... */) {
    this.validate(); // Validate immediately
  }

  private validate(): void {
    if (!this._name) {
      throw new Error('Name is required');
    }
    // More validations...
  }
}
```

**Benefits:**
- Invalid objects cannot exist
- Validation logic is centralized
- Errors are caught early

### 2. Immutability for Value Objects

Value objects should be immutable:

```typescript
class Money {
  private constructor(
    private readonly _amount: number, // readonly!
    private readonly _currency: string
  ) {}

  add(other: Money): Money {
    // Return new instance instead of modifying
    return new Money(this._amount + other._amount, this._currency);
  }
}
```

**Benefits:**
- Thread-safe
- Easier to reason about
- No unexpected mutations

### 3. Use Factory Methods

Provide clear factory methods instead of exposing constructors:

```typescript
class Order {
  private constructor(/* ... */) {}

  // Clear intent: creating new order
  static create(id: string, customerId: string): Order {
    return new Order(id, customerId, new Date(), [], OrderStatus.Pending);
  }

  // Clear intent: loading from database
  static reconstitute(id: string, /* ... */): Order {
    return new Order(/* ... */);
  }
}
```

**Benefits:**
- Clear intent
- Can have multiple creation methods
- Encapsulates creation logic

### 4. Express Business Rules Explicitly

Make business rules explicit in your domain model:

```typescript
class Order {
  canBeCancelled(): boolean {
    return this._status !== OrderStatus.Delivered &&
           this._status !== OrderStatus.Cancelled;
  }

  cancel(): void {
    if (!this.canBeCancelled()) {
      throw new Error('Cannot cancel this order');
    }
    this._status = OrderStatus.Cancelled;
  }
}
```

**Benefits:**
- Business rules are testable
- Clear documentation
- Reusable logic

### 5. Protect Invariants

Use encapsulation to protect invariants:

```typescript
class Order {
  private _items: OrderItem[] = []; // Private!

  // Don't expose the array directly
  get items(): readonly OrderItem[] {
    return [...this._items]; // Return copy
  }

  // Provide controlled methods for modification
  addItem(product: Product, quantity: number): void {
    // Validate and enforce business rules
    this.ensureOrderIsModifiable();
    // Add item...
  }
}
```

**Benefits:**
- Invariants cannot be broken from outside
- Changes go through validation
- Internal state is protected

### 6. Meaningful Method Names

Use domain language in method names:

```typescript
// Good
product.deactivate();
order.confirm();
customer.updateEmail(email);

// Bad
product.setActive(false);
order.setStatus(OrderStatus.Confirmed);
customer.setEmail(email);
```

**Benefits:**
- Reads like business logic
- Clear intent
- Better documentation

### 7. Aggregate Boundaries

Define clear aggregate boundaries:

```typescript
// Order is the aggregate root
class Order {
  private _items: OrderItem[] = [];

  // Order items can only be modified through Order
  addItem(/* ... */): void { }
  removeItem(/* ... */): void { }
}

// OrderItem is part of Order aggregate
class OrderItem {
  // No public methods that bypass Order's control
}
```

**Benefits:**
- Consistency boundaries
- Simpler transactions
- Clear ownership

### 8. Rich Domain Models Over Anemic Models

Prefer rich domain models:

```typescript
// Good: Rich domain model
class Order {
  confirm(): void {
    if (this._items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    this._status = OrderStatus.Confirmed;
  }
}

// Bad: Anemic domain model
class Order {
  items: OrderItem[];
  status: OrderStatus;
}

// Business logic elsewhere
function confirmOrder(order: Order): void {
  if (order.items.length === 0) {
    throw new Error('Cannot confirm empty order');
  }
  order.status = OrderStatus.Confirmed;
}
```

**Benefits:**
- Logic is co-located with data
- Easier to maintain
- Better encapsulation

### 9. Handle Edge Cases

Consider and handle edge cases:

```typescript
class Money {
  divide(divisor: number): Money {
    if (divisor === 0) {
      throw new Error('Cannot divide by zero');
    }
    if (!Number.isFinite(divisor)) {
      throw new Error('Divisor must be finite');
    }
    return new Money(this._amount / divisor, this._currency);
  }
}
```

### 10. Use TypeScript Features

Leverage TypeScript's type system:

```typescript
// Use enums for fixed values
enum OrderStatus {
  Pending = 'PENDING',
  Confirmed = 'CONFIRMED',
  // ...
}

// Use readonly for immutable properties
class Product {
  private constructor(
    private readonly _id: string, // Cannot be changed
    private _name: string // Can be changed through methods
  ) {}
}

// Use readonly arrays for collections
get items(): readonly OrderItem[] {
  return this._items;
}
```

## Conclusion

Building robust domain models in TypeScript requires careful attention to:

1. **Clear boundaries** between entities and value objects
2. **Self-validation** to prevent invalid states
3. **Immutability** where appropriate
4. **Explicit business rules** encoded in methods
5. **Protected invariants** through encapsulation
6. **Rich behavior** not just data holders
7. **Comprehensive testing** of business logic

By following these patterns and practices, you'll create domain models that are:
- **Maintainable**: Easy to understand and modify
- **Reliable**: Business rules are enforced consistently
- **Testable**: Logic is isolated and testable
- **Expressive**: Code reads like business requirements

Remember: Your domain models are the heart of your application. Invest time in designing them well, and you'll reap the benefits throughout your application's lifecycle.

## Further Reading

- **Domain-Driven Design** by Eric Evans
- **Implementing Domain-Driven Design** by Vaughn Vernon
- **Patterns of Enterprise Application Architecture** by Martin Fowler
- **Clean Architecture** by Robert C. Martin

---

*This guide is part of the Web Patterns documentation series. For more patterns and practices, see the other chapters in this series.*
