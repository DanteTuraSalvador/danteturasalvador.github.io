---
layout: post
title: "Part 2: Solution - CQRS Pattern"
date: 2025-01-22
category: ddd
thumbnail-img: "https://images.pexels.com/photos/1181673/pexels-photo-1181673.jpeg?auto=compress&cs=tinysrgb&w=400&h=200&fit=crop"
tags:
  - ddd
  - cqrs
  - beginner
series: cqrs
chapter: 2
prerequisites: "Part 1"
estimated_time: "20 minutes"
prev_title: "Part 1: Problem - CRUD Complexity"
prev_url: "/ddd/2025/01/21/cqrs-chapter-01.html"
next_title: "Part 3: Implementation - Commands and Queries"
next_url: "/ddd/2025/01/23/cqrs-chapter-03.html"
---

# Part 2: Solution - CQRS Pattern

## Learning Objectives

By the end of this chapter, you will understand:
- What is CQRS
- Core principles of CQRS
- Difference between commands and queries
- When to use CQRS
- Benefits of CQRS over traditional CRUD

---

## What is CQRS?

**CQRS** (Command Query Responsibility Segregation) is a pattern that:
- **Separates commands** from queries - Different models and operations
- **Optimizes** each for its specific use case
- **Enables scaling** - Scale reads and writes independently
- **Improves performance** - Optimize queries with caching
- **Simplifies testing** - Test commands/queries independently
- **Clears responsibilities** - Commands = writes, Queries = reads

### Core Principles

1. **Separate models** - Different for commands and queries
2. **Separate operations** - Different handlers for each
3. **Optimization opportunities** - Cache queries separately
4. **Business logic in domain** - Not in application
5. **Event-driven architecture** - Optional but powerful
6. **Event sourcing** - Immutable command log
7. **Write model** = Commands only write
8. **Read model** = Queries only read
9. **Single responsibility** - One job per handler
10. **Eventual consistency** - Through events

---

## CQRS vs Traditional CRUD

| Aspect | CQRS | Traditional CRUD |
|--------|----------------|---------------|
| Models | Separate (Command/Query) | Single model |
| Operations | Different handlers | Same endpoints |
| Performance | Optimizable | Mixed and slow |
| Scalability | Independent | Coupled |
| Testing | Easy | Complex |
| Caching | Read-optimized | Difficult |
| Business logic | In domain | In controllers |
| Write model | Commands only | Mixed |

---

## Command Side

### What are Commands?

**Commands** are operations that:
- **Change state** - Create, Update, Delete
- **No return value** - Task-based or fire-and-forget
- **Validation** - Business rules enforced
- **Domain events** - Generate events on success
- **Transaction boundaries** - Unit of work

### Example Commands

```csharp
public record CreateCustomerCommand
{
    public string Name { get; init; }
    public string Email { get; init; }
    public string Phone { get; init; }
    public Address Address { get; init; }
}

public record UpdateCustomerAddressCommand
{
    public CustomerId CustomerId { get; init; }
    public Address Address { get; init; }
}

public record DeleteCustomerCommand
{
    public CustomerId CustomerId { get; init; }
    public string Reason { get; init; }
}

public record PlaceOrderCommand
{
    public CustomerId CustomerId { get; init; }
    public List<CreateOrderItemCommand> Items { get; init; }
    public string ShippingAddress { get; init; }
}

public record CancelOrderCommand
{
    public OrderId OrderId { get; init; }
    public string Reason { get; init; }
}

public record ProcessPaymentCommand
{
    public OrderId OrderId { get; init; }
    public Money Amount { get; init; }
    public string PaymentMethod { get; init; }
}
```

---

## Query Side

### What are Queries?

**Queries** are operations that:
- **Read data** - Retrieve information
- **Return values** - Data transfer objects (DTOs)
- **No state change** - Pure read operations
- **Optimized for performance** - Can be cached
- **Complex queries** - Include aggregations
- **Validation filters** - Specification pattern support

### Example Queries

```csharp
public record GetCustomerByIdQuery
{
    public CustomerId CustomerId { get; init; }
}

public record GetOrdersByCustomerQuery
{
    public CustomerId CustomerId { get; init; }
    public OrderStatus? Status { get; init; }
    public DateTime? FromDate { get; init; }
    public DateTime? ToDate { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
}

public record GetOrderDetailsQuery
{
    public OrderId OrderId { get; init; }
}

public record SearchOrdersQuery
{
    public string? SearchTerm { get; init; }
    public OrderStatus? Status { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
}

public record GetDashboardStatsQuery
{
    public CustomerId CustomerId { get; init; }
    public DateTime FromDate { get; init; }
    public DateTime ToDate { get; init; }
}
```

---

## CQRS Architecture Diagram

```
┌─────────────────────────────────────┐
│       Presentation Layer (API)      │
├──────────┬──────────────────────────┤
│ Commands │        Queries           │
└────┬─────┴──────────┬───────────────┘
     │                │
     ▼                ▼
┌─────────────────────────────────────┐
│        Application Layer            │
├──────────┬──────────────────────────┤
│ Command  │       Query              │
│ Handlers │       Handlers           │
└────┬─────┴──────────┬───────────────┘
     │                │
     ▼                ▼
┌─────────────────────────────────────┐
│        Domain Layer (Core)          │
├──────────┬──────────────────────────┤
│ Entities │   Value Objects          │
│ Events   │   Specifications        │
└────┬─────┴──────────┬───────────────┘
     │                │
     ▼                ▼
┌─────────────────────────────────────┐
│       Infrastructure Layer          │
├──────────┬──────────────────────────┤
│  Write   │       Read               │
│  Repos   │       Repos + Cache      │
└──────────┴──────────────────────────┘
```

**Data Flow:**
- API → Commands/Queries
- Application Handlers → Domain Models
- Domain → Infrastructure Implementation
- Commands → Write to database
- Queries → Read from database (can be cached)

---

## Benefits of CQRS

### 1. Performance Optimization

```csharp
// Queries can be optimized independently
public class OrderQueryHandler : IQueryHandler<GetOrdersByCustomerQuery, List<OrderDto>>
{
    private readonly IOrderReadRepository _repository;
    private readonly ICacheService _cache;

    public OrderQueryHandler(IOrderReadRepository repository, ICacheService cache)
    {
        _repository = repository;
        _cache = cache;
    }

    public async Task<List<OrderDto>> Handle(GetOrdersByCustomerQuery query, 
                                                  CancellationToken token)
    {
        // Check cache first
        var cacheKey = $"orders:{query.CustomerId.Value}:{query.Status}:{query.Page}:{query.PageSize}";

        var cached = await _cache.GetAsync<List<OrderDto>>(cacheKey);
        if (cached != null)
            return cached;

        // Fetch from database with optimized queries
        var orders = await _repository.GetByCustomerAsync(
            query.CustomerId,
            query.Status,
            query.FromDate,
            query.ToDate,
            query.Page,
            query.PageSize);

        // Cache for future requests
        await _cache.SetAsync(cacheKey, orders, TimeSpan.FromMinutes(5));

        // Map to DTOs
        return orders.Select(o => ToDto(o)).ToList();
    }
}
```

### 2. Simplified Command Handlers

```csharp
public class CreateOrderCommandHandler : ICommandHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IOrderWriteRepository _repository;
    private readonly IEventBus _eventBus;

    public CreateOrderCommandHandler(IOrderWriteRepository repository, IEventBus eventBus)
    {
        _repository = repository;
        _eventBus = eventBus;
    }

    public async Task<Result<OrderId>> Handle(CreateOrderCommand command,
                                           CancellationToken token)
    {
        // Validate command
        var validationResult = await ValidateCommand(command);
        if (validationResult.IsFailure)
            return Result<OrderId>.Failure(validationResult.Error);

        // Create order entity
        var order = new Order(/*...*/);
        
        // Persist
        var saved = await _repository.AddAsync(order);
        
        // Emit domain event
        await _eventBus.PublishAsync(new OrderCreatedEvent(saved.Id));

        return Result<OrderId>.Success(saved.Id);
    }
}
```

### 3. Clear Responsibilities

```csharp
// Command handlers only handle commands
public class OrderCommandService : IOrderCommandService
{
    // BAD: Query mixed with commands
    public async Task<Result<List<Order>>> GetOrdersByCustomer(CustomerId id)
    {
        // This should be in query handler!
        return await _queryHandler.Handle(new GetOrdersByCustomerQuery(id));
    }
}

// Query handlers only handle queries
public class OrderQueryService : IOrderQueryService
{
    // BAD: Commands mixed with queries
    public async Task<Result> DeleteOrder(OrderId id)
    {
        // This should be in command handler!
        return await commandHandler.Handle(new DeleteOrderCommand(id));
    }
}
```

---

## When to Use CQRS

**Use CQRS when:**
- Read and write have different performance needs
- System requires high read scalability
- Complex business logic
- Multiple data sources
- Event-driven architecture desired
- Testing read/write separately
- Microservices architecture planned

**Examples:**
- High-traffic applications
- E-commerce platforms
- Social media platforms
- Real-time dashboards
- Systems with complex queries

---

## When NOT to Use CQRS

**Don't use when:**
- Simple CRUD application
- Low traffic (no performance issues)
- Small team learning curve too steep
- Simple domain model
- Traditional layered architecture works fine
- Only one data source

**Use traditional CRUD when:**
- Simple applications
- Small team or solo developer
- Fast time-to-market
- Low complexity requirements
- Performance not critical

---

## Real-World Examples

### E-Commerce: Read Optimization

```csharp
// Query handler with caching
public class ProductCatalogQueryHandler : IQueryHandler<SearchProductsQuery, List<ProductDto>>
{
    private readonly IProductReadRepository _repository;
    private readonly ICacheService _cache;

    public async Task<List<ProductDto>> Handle(SearchProductsQuery query, 
                                             CancellationToken token)
    {
        // Check cache
        var cacheKey = $"search:{query.SearchTerm}:{query.Category}";

        var cached = await _cache.GetAsync<List<ProductDto>>(cacheKey);
        if (cached != null)
            return cached;

        // Fetch with optimized search
        var products = await _repository.SearchAsync(query.SearchTerm, query.Category);

        // Cache for 5 minutes
        await _cache.SetAsync(cacheKey, products, TimeSpan.FromMinutes(5));

        return products.Select(p => ToDto(p)).ToList();
    }
}
```

### Financial: Transaction Boundaries

```csharp
// Command handler with transactions
public class PaymentCommandHandler : ICommandHandler<ProcessPaymentCommand, Result<PaymentId>>
{
    private readonly IOrderWriteRepository _orderRepository;
    private readonly IPaymentGateway _gateway;

    public PaymentCommandHandler(IOrderWriteRepository repository, IPaymentGateway gateway)
    {
        _orderRepository = repository;
        _gateway = gateway;
    }

    public async Task<Result<PaymentId>> Handle(ProcessPaymentCommand command,
                                                  CancellationToken token)
    {
        var order = await _orderRepository.GetByIdAsync(command.OrderId);
        if (order == null)
            return Result<PaymentId>.Failure("Order not found");

        // Validate payment amount
        var validationResult = ValidatePayment(command, order);
        if (validationResult.IsFailure)
            return Result<PaymentId>.Failure(validationResult.Error);

        // Process payment
        var paymentResult = await _gateway.ProcessAsync(command);

        if (!paymentResult.Success)
        return Result<PaymentId>.Failure(paymentResult.Error);

        // Update order status
        order.MarkAsPaid();
        await _orderRepository.UpdateAsync(order);

        return Result<PaymentId>.Success(paymentResult.PaymentId);
    }
}
```

---

## Key Takeaways

- CQRS separates commands and queries
- Commands change state, queries return data
- Performance can be optimized independently
- Business logic remains in domain
- Testing becomes simpler
- Scalability improves significantly
- Next: Implement Commands and Queries in Part 3

---

## References

- [Source Code: CQRSCleanApi](https://github.com/DanteTuraSalvador/CQRSCleanApi)
- [Greg Young: CQRS and Event Sourcing](https://www.youtube.com/watch?v=8hkXbE1Rg)
- [Microsoft: CQRS](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)

