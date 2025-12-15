# Web Architecture Patterns
## A Practical Guide to Clean Architecture in React & Elixir

*Separating Core Business Logic from Infrastructure Code*

---

## Welcome 👋

This guide teaches you how to build maintainable, testable, and flexible web applications by separating your core business logic from infrastructure concerns.

Whether you're working with **React/TypeScript** on the frontend or **Elixir/Phoenix** on the backend, the principles remain the same: keep your domain logic pure, isolated, and independent of frameworks and external systems.

## The Problem This Solves

Have you ever:
- 😰 Been afraid to change code because you might break something unexpected?
- 🐌 Found it difficult to write tests without setting up databases and APIs?
- 🔄 Copy-pasted business logic between controllers because it was too tangled to extract?
- 📚 Spent hours understanding what a piece of code actually *does* vs *how* it does it?

**This guide will help you solve these problems.**

## Who This Guide Is For

- **React developers** who want to write cleaner, more testable frontend code
- **Elixir/Phoenix developers** looking to apply DDD and clean architecture principles
- **Full-stack developers** working across both ecosystems
- **Teams** wanting to establish better architectural patterns
- **Anyone** tired of "spaghetti code" and looking for a better way

## What Makes This Guide Different

✅ **Dual-stack perspective:** See the same patterns applied to both frontend and backend  
✅ **Practical, not theoretical:** Real code examples you can use immediately  
✅ **Progressive learning:** Start simple, build up to advanced patterns  
✅ **Before/after comparisons:** See exactly what improves  
✅ **Testing-focused:** Learn how to make your code actually testable  
✅ **No over-engineering:** Patterns scaled appropriately to problem size  

## What You'll Learn

### Part 1: Foundation
- How to identify core logic vs infrastructure code
- The two rules that define clean architecture
- Why separation matters for long-term maintainability

### Part 2: Domain Modeling
- Creating rich domain models in TypeScript classes
- Domain modeling with Elixir structs and pure functions
- Entities, Value Objects, and business rules
- Validation and invariants

### Part 3: Patterns
- **Repository Pattern:** Abstracting data access in both React and Elixir
- **Application Services:** Orchestrating use cases
- **Command Pattern:** Representing user intentions
- **Ports & Adapters:** The hexagonal architecture approach

### Part 4: Architecture
- Layered architecture (Domain → Application → Infrastructure)
- Dependency inversion and why it matters
- Testing strategies for each layer
- State management that respects boundaries

### Part 5: Practice
- Refactoring legacy code step-by-step
- Real-world example: E-commerce order system
- Common pitfalls and how to avoid them
- When to use (and not use) these patterns

## Guide Structure

### 📖 Core Concepts
- [Chapter 1: The Problem with Mixed Code](docs/01-the-problem.md)
- [Chapter 2: Defining Core vs Infrastructure Code](docs/02-core-vs-infrastructure.md)

### 🎯 Domain Modeling
- [Chapter 3: Domain Models in TypeScript](docs/03-domain-models-typescript.md)
- [Chapter 4: Domain Models in Elixir](docs/04-domain-models-elixir.md)

### 🔧 Patterns & Practices
- [Chapter 5: Repository Pattern in React](docs/05-repository-pattern-react.md)
- [Chapter 6: Repository Pattern in Elixir](docs/06-repository-pattern-elixir.md)
- [Chapter 7: Application Services](docs/07-application-services.md)
- [Chapter 8: Command Pattern](docs/08-command-pattern.md)

### 🏗️ Architecture
- [Chapter 9: Layered Architecture](docs/09-layered-architecture.md)
- [Chapter 10: Hexagonal Architecture (Ports & Adapters)](docs/10-hexagonal-architecture.md)
- [Chapter 11: Testing Strategies](docs/11-testing-strategies.md)

### 🔄 Refactoring
- Chapter 12: Refactoring React Code *(coming soon)*
- Chapter 13: Refactoring Elixir Code *(coming soon)*

### 📚 Reference
- Architecture Diagrams *(coming soon)*
- Decision Matrix *(coming soon)*
- Recommended Libraries *(coming soon)*
- Common Pitfalls *(coming soon)*

## How to Use This Guide

**Sequential Reading:** Start from Chapter 1 and work your way through. Each chapter builds on previous concepts.

**Reference Guide:** Jump to specific chapters when you need to understand a particular pattern.

**Team Learning:** Review chapters as a team and discuss how to apply them to your codebase.

**Gradual Adoption:** You don't need to refactor everything at once. Apply these patterns incrementally as you work on new features.

## Code Examples

All code examples in this guide are:
- ✅ Complete and runnable (not just snippets)
- ✅ Typed (TypeScript on frontend, specs on backend)
- ✅ Tested (includes test examples)
- ✅ Practical (based on real-world scenarios)

The recurring example throughout the guide is an **e-commerce order system**, showing how the same concepts apply across different layers and technologies.

## Philosophy

This guide follows these principles:

**1. Core logic should be pure:** Business rules live in plain TypeScript classes or Elixir modules with no framework dependencies.

**2. Infrastructure is interchangeable:** Databases, APIs, and UI frameworks can change without touching core logic.

**3. Tests should be fast:** Core logic can be tested in milliseconds without external dependencies.

**4. Code should tell a story:** Reading your code should reveal *what* the business does, not just *how* it's implemented.

**5. Patterns serve purpose:** Don't add layers or abstractions without a clear benefit.

## Inspiration & Prior Art

This guide is inspired by:
- **Domain-Driven Design** (Eric Evans)
- **Clean Architecture** (Robert C. Martin)
- **Hexagonal Architecture** (Alistair Cockburn)
- **"Web App Architecture" Vietnamese ebook** by Huy Nguyen

It adapts these timeless principles for modern React and Elixir development.

## About the Author

Created by developers who've experienced the pain of tangled codebases and the joy of clean architecture. This guide represents years of learning, mistakes, and refinements across multiple projects and teams.

## Contributing

Found a typo? Have a suggestion? Want to add an example?

Contributions are welcome! Please:
- Open an issue for discussion
- Submit PRs for improvements
- Share your experiences applying these patterns

## Get Started

Ready to write cleaner code? **[Start with Chapter 1: The Problem →](docs/01-the-problem.md)**

---

*Remember: The best architecture is the one that makes your code easy to understand, test, and change. Start simple, refactor as needed, and always keep your domain logic at the center.*

Happy coding! ✨