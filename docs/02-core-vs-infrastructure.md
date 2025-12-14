# Chapter 2: Core Code vs Infrastructure Code

## Overview

One of the most fundamental architectural decisions in web application development is distinguishing between **core code** and **infrastructure code**. This separation is crucial for building maintainable, testable, and scalable applications. Understanding this distinction helps teams organize their codebase, make better design decisions, and improve long-term code quality.

## What is Core Code?

**Core code** (also known as domain code or business logic) represents the heart of your application—the unique value proposition that makes your application different from others.

### Characteristics of Core Code

- **Business Rules**: Encodes the specific rules and workflows of your domain
- **Domain Models**: Represents the key concepts and entities in your business
- **Framework-Independent**: Should not depend on specific web frameworks, databases, or external services
- **Highly Testable**: Can be tested in isolation without spinning up servers or databases
- **Long-Lived**: Changes less frequently as it represents stable business concepts
- **Pure Logic**: Focuses on transformations, calculations, and decisions

### Examples of Core Code

```javascript
// User validation logic
class UserValidator {
  validateEmail(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }

  validateAge(birthDate) {
    const age = this.calculateAge(birthDate);
    return age >= 18;
  }

  calculateAge(birthDate) {
    const today = new Date();
    const birth = new Date(birthDate);
    let age = today.getFullYear() - birth.getFullYear();
    const monthDiff = today.getMonth() - birth.getMonth();
    
    if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birth.getDate())) {
      age--;
    }
    
    return age;
  }
}

// Order processing logic
class OrderProcessor {
  calculateTotal(items, discountCode = null) {
    const subtotal = items.reduce((sum, item) => {
      return sum + (item.price * item.quantity);
    }, 0);

    const discount = this.applyDiscount(subtotal, discountCode);
    const tax = this.calculateTax(subtotal - discount);
    
    return {
      subtotal,
      discount,
      tax,
      total: subtotal - discount + tax
    };
  }

  applyDiscount(amount, code) {
    // Business rule: discount codes
    const discounts = {
      'SAVE10': 0.10,
      'SAVE20': 0.20,
      'FIRSTORDER': 0.15
    };

    return code && discounts[code] 
      ? amount * discounts[code] 
      : 0;
  }

  calculateTax(amount) {
    // Business rule: tax rate
    const TAX_RATE = 0.08;
    return amount * TAX_RATE;
  }
}
```

## What is Infrastructure Code?

**Infrastructure code** handles the technical concerns of communicating with the outside world—databases, APIs, file systems, message queues, and other external systems.

### Characteristics of Infrastructure Code

- **External Communication**: Manages interactions with databases, APIs, file systems, etc.
- **Framework-Dependent**: Often tied to specific frameworks, libraries, or platforms
- **I/O Operations**: Handles input/output operations that are often slow and error-prone
- **Configuration-Heavy**: Requires environment-specific settings
- **Changes Frequently**: May need updates as technology evolves
- **Side Effects**: Performs actions that change state outside the application

### Examples of Infrastructure Code

```javascript
// Database repository
class UserRepository {
  constructor(database) {
    this.db = database;
  }

  async findById(userId) {
    const query = 'SELECT * FROM users WHERE id = ?';
    const result = await this.db.query(query, [userId]);
    return result[0] || null;
  }

  async save(user) {
    const query = `
      INSERT INTO users (id, email, name, created_at)
      VALUES (?, ?, ?, ?)
      ON DUPLICATE KEY UPDATE
        email = VALUES(email),
        name = VALUES(name)
    `;
    
    await this.db.query(query, [
      user.id,
      user.email,
      user.name,
      user.createdAt
    ]);
  }

  async delete(userId) {
    const query = 'DELETE FROM users WHERE id = ?';
    await this.db.query(query, [userId]);
  }
}

// HTTP Controller
class OrderController {
  constructor(orderService, logger) {
    this.orderService = orderService;
    this.logger = logger;
  }

  async createOrder(req, res) {
    try {
      const { items, discountCode } = req.body;
      const userId = req.user.id;

      const order = await this.orderService.createOrder(
        userId,
        items,
        discountCode
      );

      this.logger.info(`Order created: ${order.id}`);
      
      res.status(201).json({
        success: true,
        data: order
      });
    } catch (error) {
      this.logger.error('Order creation failed:', error);
      
      res.status(400).json({
        success: false,
        error: error.message
      });
    }
  }
}

// External API client
class PaymentGateway {
  constructor(apiKey, baseUrl) {
    this.apiKey = apiKey;
    this.baseUrl = baseUrl;
  }

  async processPayment(amount, cardToken) {
    const response = await fetch(`${this.baseUrl}/charges`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        amount: amount * 100, // Convert to cents
        currency: 'usd',
        source: cardToken
      })
    });

    if (!response.ok) {
      throw new Error('Payment processing failed');
    }

    return await response.json();
  }
}
```

## The Dependency Rule

A crucial principle when separating core and infrastructure code is the **Dependency Rule**:

> **Core code should never depend on infrastructure code. Infrastructure code should depend on core code.**

This means:
- ✅ Core code defines interfaces/contracts
- ✅ Infrastructure code implements those interfaces
- ✅ Core code receives dependencies via dependency injection
- ❌ Core code should NOT import infrastructure modules directly
- ❌ Core code should NOT know about HTTP, databases, or frameworks

### Visualizing the Dependency Rule

```
┌─────────────────────────────────────┐
│     Infrastructure Layer            │
│  (Controllers, Repositories, APIs)  │
│            ↓ depends on             │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│        Core/Domain Layer            │
│   (Business Logic, Entities)        │
│       ↓ defines interfaces          │
└─────────────────────────────────────┘
```

## Practical Example: User Registration

Let's see how to properly separate concerns in a user registration feature.

### ❌ Bad Example (Mixed Concerns)

```javascript
// Everything mixed together
class UserService {
  async registerUser(email, password) {
    // Validation mixed with infrastructure
    if (!email.includes('@')) {
      throw new Error('Invalid email');
    }

    // Direct database access in business logic
    const existing = await db.query(
      'SELECT * FROM users WHERE email = ?',
      [email]
    );

    if (existing.length > 0) {
      throw new Error('User already exists');
    }

    // Password hashing
    const hashedPassword = await bcrypt.hash(password, 10);

    // More direct database access
    await db.query(
      'INSERT INTO users (email, password) VALUES (?, ?)',
      [email, hashedPassword]
    );

    // Direct email service call
    await sendGrid.send({
      to: email,
      subject: 'Welcome!',
      body: 'Thanks for registering'
    });

    return { success: true };
  }
}
```

### ✅ Good Example (Separated Concerns)

```javascript
// Core Code: Domain Model
class User {
  constructor(id, email, passwordHash) {
    this.id = id;
    this.email = email;
    this.passwordHash = passwordHash;
    this.createdAt = new Date();
  }

  static create(id, email, passwordHash) {
    if (!User.isValidEmail(email)) {
      throw new Error('Invalid email format');
    }
    return new User(id, email, passwordHash);
  }

  static isValidEmail(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
}

// Core Code: Registration Service (pure business logic)
class UserRegistrationService {
  constructor(userRepository, passwordHasher, emailService) {
    this.userRepository = userRepository;
    this.passwordHasher = passwordHasher;
    this.emailService = emailService;
  }

  async registerUser(email, password) {
    // Business logic: check if user exists
    const existingUser = await this.userRepository.findByEmail(email);
    if (existingUser) {
      throw new Error('User already exists');
    }

    // Business logic: create user
    const passwordHash = await this.passwordHasher.hash(password);
    const user = User.create(
      this.generateId(),
      email,
      passwordHash
    );

    // Business logic: save and notify
    await this.userRepository.save(user);
    await this.emailService.sendWelcomeEmail(user.email);

    return user;
  }

  generateId() {
    return Math.random().toString(36).substr(2, 9);
  }
}

// Infrastructure Code: Database Repository
class SqlUserRepository {
  constructor(database) {
    this.db = database;
  }

  async findByEmail(email) {
    const result = await this.db.query(
      'SELECT * FROM users WHERE email = ?',
      [email]
    );
    
    if (result.length === 0) return null;
    
    const row = result[0];
    return new User(row.id, row.email, row.password_hash);
  }

  async save(user) {
    await this.db.query(
      'INSERT INTO users (id, email, password_hash, created_at) VALUES (?, ?, ?, ?)',
      [user.id, user.email, user.passwordHash, user.createdAt]
    );
  }
}

// Infrastructure Code: Password Hasher
class BcryptPasswordHasher {
  async hash(password) {
    return await bcrypt.hash(password, 10);
  }

  async verify(password, hash) {
    return await bcrypt.compare(password, hash);
  }
}

// Infrastructure Code: Email Service
class SendGridEmailService {
  constructor(apiKey) {
    this.client = sendGrid(apiKey);
  }

  async sendWelcomeEmail(email) {
    await this.client.send({
      to: email,
      from: 'noreply@example.com',
      subject: 'Welcome!',
      text: 'Thanks for registering'
    });
  }
}

// Infrastructure Code: HTTP Controller
class UserController {
  constructor(registrationService) {
    this.registrationService = registrationService;
  }

  async register(req, res) {
    try {
      const { email, password } = req.body;
      const user = await this.registrationService.registerUser(email, password);
      
      res.status(201).json({
        success: true,
        data: { id: user.id, email: user.email }
      });
    } catch (error) {
      res.status(400).json({
        success: false,
        error: error.message
      });
    }
  }
}
```

## Benefits of This Separation

### 1. **Testability**

Core code can be tested without any infrastructure:

```javascript
describe('UserRegistrationService', () => {
  it('should register a new user', async () => {
    // Mock infrastructure dependencies
    const mockRepo = {
      findByEmail: jest.fn().mockResolvedValue(null),
      save: jest.fn()
    };
    const mockHasher = {
      hash: jest.fn().mockResolvedValue('hashed_password')
    };
    const mockEmailService = {
      sendWelcomeEmail: jest.fn()
    };

    const service = new UserRegistrationService(
      mockRepo,
      mockHasher,
      mockEmailService
    );

    // Test pure business logic
    const user = await service.registerUser('test@example.com', 'password123');

    expect(user.email).toBe('test@example.com');
    expect(mockRepo.save).toHaveBeenCalled();
    expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith('test@example.com');
  });

  it('should reject invalid email', async () => {
    const service = new UserRegistrationService(null, null, null);

    await expect(
      service.registerUser('invalid-email', 'password123')
    ).rejects.toThrow('Invalid email format');
  });
});
```

### 2. **Flexibility**

Easy to swap infrastructure implementations:

```javascript
// Use PostgreSQL in production
const prodRepo = new SqlUserRepository(postgresDb);

// Use MongoDB for a specific feature
const mongoRepo = new MongoUserRepository(mongoDb);

// Use in-memory storage for testing
const testRepo = new InMemoryUserRepository();

// Same business logic works with all of them
const service = new UserRegistrationService(prodRepo, hasher, emailService);
```

### 3. **Maintainability**

Changes to infrastructure don't affect business logic:

```javascript
// Switch from SendGrid to AWS SES
class SesEmailService {
  async sendWelcomeEmail(email) {
    // Different implementation, same interface
    await ses.sendEmail({ /* ... */ });
  }
}

// No changes needed to UserRegistrationService!
```

### 4. **Understanding**

Business logic is clear and readable without infrastructure noise:

```javascript
// Easy to understand what the business does
async registerUser(email, password) {
  const existingUser = await this.userRepository.findByEmail(email);
  if (existingUser) {
    throw new Error('User already exists');
  }

  const passwordHash = await this.passwordHasher.hash(password);
  const user = User.create(this.generateId(), email, passwordHash);

  await this.userRepository.save(user);
  await this.emailService.sendWelcomeEmail(user.email);

  return user;
}
```

## Common Mistakes to Avoid

### 1. **Leaking Infrastructure into Core**

```javascript
// ❌ Bad: Database concerns in domain model
class User {
  async save() {
    await db.query('INSERT INTO users...');
  }
}

// ✅ Good: Separate repository
class UserRepository {
  async save(user) {
    await db.query('INSERT INTO users...');
  }
}
```

### 2. **Mixing Validation with Infrastructure**

```javascript
// ❌ Bad: HTTP validation in controller only
app.post('/users', (req, res) => {
  if (!req.body.email) {
    return res.status(400).json({ error: 'Email required' });
  }
  // ...
});

// ✅ Good: Core validation, HTTP mapping in controller
class User {
  static create(email) {
    if (!email) {
      throw new ValidationError('Email required');
    }
    // ...
  }
}

app.post('/users', (req, res) => {
  try {
    const user = User.create(req.body.email);
    // ...
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

### 3. **Over-Abstracting Too Early**

```javascript
// ❌ Bad: Unnecessary abstraction
class IUserRepositoryFactoryProvider {
  // Too complex for simple needs
}

// ✅ Good: Start simple, refactor when needed
class UserRepository {
  // Simple, direct implementation
}
```

## Decision Framework

When deciding if code is core or infrastructure, ask:

| Question | Core | Infrastructure |
|----------|------|----------------|
| Does it represent a business rule? | ✅ | ❌ |
| Can it be tested without external systems? | ✅ | ❌ |
| Would it change if we switched databases? | ❌ | ✅ |
| Would it change if we switched frameworks? | ❌ | ✅ |
| Does it perform I/O operations? | ❌ | ✅ |
| Does it depend on external libraries? | ❌ | ✅ |
| Would domain experts understand it? | ✅ | ❌ |

## Summary

- **Core code** = Business logic, domain models, pure functions
- **Infrastructure code** = Databases, HTTP, APIs, external services
- **Dependency Rule**: Core never depends on infrastructure
- **Benefits**: Testability, flexibility, maintainability, clarity
- **Key Practice**: Use dependency injection to provide infrastructure to core

In the next chapter, we'll explore specific architectural patterns that build upon this foundational separation, starting with the **Layered Architecture** pattern.

---

**Next**: [Chapter 3: Layered Architecture Pattern](03-layered-architecture.md)  
**Previous**: [Chapter 1: Introduction to Web Architecture Patterns](01-introduction.md)
