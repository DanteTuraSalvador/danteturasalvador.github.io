---
layout: post
title: "Part 2: Solution - Clean Architecture"
date: 2025-01-17
category: ddd
thumbnail-img: "https://images.unsplash.com/photo-1487058792275-0ad4aaf24ca7?w=400&h=200&fit=crop"
tags:
  - ddd
  - clean-architecture
  - beginner
series: clean-architecture
chapter: 2
prerequisites: "Part 1"
estimated_time: "20 minutes"
prev_title: "Part 1: Problem - Layered Architecture Issues"
prev_url: "/ddd/2025/01/16/clean-architecture-chapter-01.html"
next_title: "Part 3: Implementation - Project Structure"
next_url: "/ddd/2025/01/18/clean-architecture-chapter-03.html"
---

# Part 2: Solution - Clean Architecture

## Learning Objectives

By the end of this chapter, you will understand:
- What is Clean Architecture
- Core principles of Clean Architecture
- How dependencies should flow
- The layers and their responsibilities
- Benefits over traditional layered architecture

---

## What is Clean Architecture?

Clean Architecture is an architectural pattern that:
- **Separates concerns** into distinct layers
- **Enforces dependency inversion** - Depends on abstractions, not implementations
- **Controls dependencies** - Dependencies flow inward
- **Isolates domain** - No infrastructure dependencies
- **Enables testing** - Easy to test each layer independently
- **Supports flexibility** - Easy to swap implementations
- **Follows dependency rule** - Depends on nothing more stable than you

### Core Principles

1. **Dependency Rule** - Dependencies point inward
2. **SOLID Principles** - Single responsibility, Open-closed, Liskov substitution, Interface segregation, Dependency inversion
3. **Domain-Centric** - Business logic in domain layer
4. **Infrastructure as plugin** - Swap implementations easily
5. **Cross-cutting concerns** - Handle logging, caching separately
6. **Clear boundaries** - Layers have well-defined responsibilities

---

## Clean Architecture Layers

### 1. Domain Layer (Core)

**Responsibility:** Core business rules and entities

**Should NOT have dependencies on:**
- Infrastructure (EF Core, Dapper, etc.)
- Application layer
- Presentation layer
- External services (unless abstractions)

**Can depend on:**
- Nothing external - Core domain is the innermost layer

**Examples:**
- Entities (Customer, Order, Product)
- Value Objects (Email, Money, Address)
- Domain Services (interfaces for domain concepts)
- Specifications (domain-specific interfaces)
- Events (domain events)

### 2. Application Layer (Use Cases)

**Responsibility:** Orchestrate domain objects to fulfill use cases

**Should NOT have dependencies on:**
- Infrastructure implementations
- Presentation layer
- Framework-specific technologies

**Can depend on:**
- Domain layer (entities, value objects, domain services)
- Application services (interfaces)
- Infrastructure interfaces (IRepository, IEmailService)

**Examples:**
- Application Services (CustomerService, OrderService)
- Command Handlers (CreateCustomerCommandHandler)
- Query Handlers (GetOrdersQueryHandler)
- Use Cases (RegisterCustomer, PlaceOrder)
- DTOs for communication between layers
- Mappers (domain ↔ DTO conversion)

### 3. Infrastructure Layer

**Responsibility:** Technical implementations and external integrations

**Can depend on:**
- Domain layer (entities, value objects)
- Application layer interfaces
- External libraries (EF Core, Serilog, HttpClient)
- Framework-specific libraries

**Should NOT have dependencies on:**
- Application layer implementations
- Presentation layer
- Domain logic (only provides implementations)

**Examples:**
- Repository implementations (EntityFrameworkRepository)
- External service clients (EmailServiceClient)
- Background services (HostedService)
- Database migrations (DbContext)
- Caching implementations (RedisCache)
- File storage implementations (AzureBlobStorage)

### 4. Presentation Layer

**Responsibility:** Handle HTTP requests and responses

**Can depend on:**
- Application layer (interfaces)
- DTOs for data transfer
- Domain entities (for responses)
- Cross-cutting concerns (logging, authentication)

**Should NOT have dependencies on:**
- Infrastructure implementations
- Domain layer (only use interfaces)
- Business logic (only pass through data)

**Examples:**
- Controllers (CustomerController, OrderController)
- Minimal API endpoints
- SignalR hubs
- GraphQL resolvers
- API clients (HttpClient wrappers)

### 5. Cross-Cutting Concerns

**Responsibility:** Infrastructure concerns spanning all layers

**Can depend on:**
- All layers (via interfaces)
- Domain entities (for logging, etc.)
- Infrastructure implementations

**Examples:**
- Logging (Serilog, NLog)
- Caching (Redis, MemoryCache)
- Authentication (JwtTokenService, IdentityServer)
- Authorization (Policy-based authorization)
- Validation (FluentValidation, custom validators)
- Exception handling (Global exception middleware)

---

## Dependency Flow Diagram

```
┌─────────────────────────────────────┐
│     Presentation Layer           │
│   (Controllers, API)           │
└───────────────┬────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│     Application Layer           │
│   (Use Cases, Services)         │
└───────────────┬────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│       Domain Layer (Core)         │
│   (Entities, VOs, Events)        │
└─────────────────────────────────────┘

                ▼
┌─────────────────────────────────────┐
│   Infrastructure Layer            │
│   (Repos, External Services)     │
└─────────────────────────────────────┘
```

**Dependencies Flow:**
- Presentation → Application interfaces
- Application → Domain layer
- Application → Infrastructure interfaces
- Infrastructure → External services
- Domain → No dependencies (pure)
- Infrastructure → Domain implementations

---

## Benefits of Clean Architecture

### 1. Testability

```csharp
// Test domain without infrastructure
[Test]
public void Customer_ChangeEmail_ValidData_ReturnsSuccess()
{
    var customerId = CustomerId.NewGuid();
    var customer = new Customer(customerId);
    var newEmail = Email.Create("new@example.com").Value;
    
    customer.ChangeEmail(newEmail);
    
    Assert.AreEqual(newEmail, customer.Email);
}

// Test application with mocked infrastructure
[Test]
public async Task CustomerService_CreateCustomer_WithMockRepository_Success()
{
    var mockRepo = new Mock<ICustomerRepository>();
    var service = new CustomerService(mockRepo, _emailService);
    
    mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
                  .ReturnsAsync((Customer?)null);
    
    var request = new CreateCustomerRequest("John Doe", "john@example.com");
    var result = await service.CreateCustomerAsync(request);
    
    Assert.IsTrue(result.IsSuccess);
    mockRepo.Verify(r => r.AddAsync(It.IsAny<Customer>()), Times.Once);
}
```

### 2. Flexibility

```csharp
// Easy to swap implementations
public class CustomerService
{
    private readonly ICustomerRepository _repository;
    private readonly IEmailService _emailService;
    
    // Constructor injection of interfaces
    public CustomerService(ICustomerRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }
}

// Swap EF Core for Dapper
// No changes to CustomerService!
// Just register different implementation in DI container
```

### 3. Maintainability

```csharp
// Clear separation of concerns
public class Customer
{
    // Domain concerns only
    public CustomerId Id { get; }
    public Email Email { get; private set; }
    public string Name { get; private set; }
    
    public void ChangeEmail(Email newEmail)
    {
        // Business logic in domain
        Email = newEmail;
    }
}

public class CustomerRepository : ICustomerRepository
{
    // Infrastructure concerns only
    private readonly DbContext _context;
    
    public CustomerRepository(DbContext context)
    {
        _context = context;
    }
    
    public async Task<Customer?> GetByIdAsync(Guid id)
    {
        // Persistence only, no business logic
        return await _context.Customers.FindAsync(id);
    }
}
```

---

## When to Use Clean Architecture

**Use Clean Architecture when:**
- You have complex business logic
- Multiple implementations possible (different databases)
- Testability is important
- Long-term maintenance is a concern
- Team size is growing
- Business rules are complex
- You need clear boundaries between layers

**Examples:**
- Enterprise applications
- Systems with multiple data sources
- APIs that need to be easily tested
- Applications requiring flexibility

---

## When NOT to Use Clean Architecture

**Don't use when:**
- Simple CRUD application (traditional layered is fine)
- Single developer project
- Quick prototype (over-engineering)
- Small domain with simple rules
- Performance is critical (has overhead)

**Use traditional layered when:**
- Application is simple CRUD
- Team is small and experienced
- Speed of development is priority
- Learning curve is too steep

---

## Real-World: Clean Architecture Structure

```
Project.Solution/
├── Domain/
│   ├── Entities/
│   │   ├── Customer.cs
│   │   ├── Order.cs
│   │   └── Product.cs
│   ├── ValueObjects/
│   │   ├── Email.cs
│   │   ├── Money.cs
│   │   └── Address.cs
│   ├── Services/
│   │   ├── ICustomerDomainService.cs
│   │   └── IOrderDomainService.cs
│   ├── Events/
│   │   ├── CustomerCreatedEvent.cs
│   │   └── OrderPlacedEvent.cs
│   └── Specifications/
│       ├── ICustomerSpecification.cs
│       └── IOrderSpecification.cs
│
├── Application/
│   ├── Services/
│   │   ├── CustomerService.cs
│   │   └── OrderService.cs
│   ├── UseCases/
│   │   ├── CreateCustomerUseCase.cs
│   │   └── PlaceOrderUseCase.cs
│   ├── DTOs/
│   │   ├── CustomerDto.cs
│   │   └── OrderDto.cs
│   ├── Interfaces/
│   │   ├── ICustomerRepository.cs
│   │   ├── IOrderRepository.cs
│   │   └── IEmailService.cs
│   └── Mappers/
│       ├── CustomerMapper.cs
│       └── OrderMapper.cs
│
├── Infrastructure/
│   ├── Persistence/
│   │   ├── EntityFramework/
│   │   │   ├── CustomerRepository.cs
│   │   │   └── OrderRepository.cs
│   │   └── DbContext.cs
│   │   └── Dapper/
│   │       └── CustomerRepository.cs
│   ├── ExternalServices/
│   │   ├── EmailService.cs
│   │   └── PaymentGateway.cs
│   └── Caching/
│       ├── RedisCache.cs
│       └── MemoryCache.cs
│
└── Presentation/
    ├── Controllers/
    │   ├── CustomerController.cs
    │   └── OrderController.cs
    └── Minimal API/
        ├── Endpoints/
        │   ├── CustomerEndpoints.cs
        └── OrderEndpoints.cs
```

---

## Key Takeaways

- Clean Architecture enforces dependency inversion
- Domain is isolated from infrastructure
- Each layer has clear responsibilities
- Testing is significantly easier
- Implementations are swappable
- Use for complex business applications
- Next: Implement Clean Architecture project structure in Part 3

---

## References

- [Source Code: MinimalAPICleanArchitecture](https://github.com/DanteTuraSalvador/MinimalAPICleanArchitecture)
- [Robert C. Martin: Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Jason Taylor: Clean Architecture](https://jasontaylor.dev/2012/08/13/the-clean-architecture.html)
- [Microsoft: Architecture Patterns](https://docs.microsoft.com/en-us/dotnet/standard/architecture/)

