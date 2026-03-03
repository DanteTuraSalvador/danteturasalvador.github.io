---
layout: post
title: "Chapter 3: Implementation - Commands and Queries"
date: 2025-01-23
tags:
  - ddd
  - cqrs
  - implementation
series: cqrs
chapter: 3
prerequisites: "Chapter 2"
estimated_time: "30 minutes"
prev_title: "Chapter 2: Solution - CQRS Pattern"
prev: "/2025/01/22/cqrs-chapter-02.html"
next_title: "Chapter 4: Advanced - Event Sourcing"
next: "/2025/01/24/cqrs-chapter-04.html"
---

# Chapter 3: Implementation - Commands and Queries

## Learning Objectives

By the end of this chapter, you will be able to:
- Implement command handlers
- Implement query handlers
- Create CQRS models (commands and queries)
- Use Result pattern with CQRS
- Implement caching for queries
- Handle events

---

## Implementation: Step by Step

### Step 1: CQRS Models

```csharp
namespace TestNest.CleanArchitecture.CQRS.Models;

// Write Model - Commands only modify state
public class Order
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; set; }
    public Money TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ShippedAt { get; set; }
    public DateTime? DeliveredAt { get; set; }
    public List<OrderItem> Items { get; } = new();
}

    public void AddItem(OrderItem item)
    {
        Items.Add(item);
        TotalAmount = TotalAmount.Add(item.LineTotal);
    }

    public void MarkAsPaid()
    {
        Status = OrderStatus.Paid;
    }

    public void MarkAsShipped()
    {
        if (Status != OrderStatus.Paid)
            throw new DomainException("Order must be paid before shipping");

        Status = OrderStatus.Shipped;
        ShippedAt = DateTime.UtcNow;
    }

    public bool CanCancel()
    {
        return Status == OrderStatus.Created || 
               Status == OrderStatus.PaymentProcessing;
    }

    public bool CanShip()
    {
        return Status == OrderStatus.Paid;
    }

    public void Cancel()
    {
        if (!CanCancel())
            throw new DomainException("Cannot cancel order in this state");
        Status = OrderStatus.Canceled;
    }
}

// Read Model - DTOs for queries
public class OrderDto
{
    public Guid Id { get; init; }
    public string CustomerName { get; init; }
    public Money TotalAmount { get; init; }
    public string Status { get; init; }
    public DateTime CreatedAt { get; init; }
    public List<OrderItemDto> Items { get; init; }
}

public class OrderItemDto
{
    public Guid Id { get; init; }
    public string ProductName { get; init; }
    public Money UnitPrice { get; init; }
    public int Quantity { get; init; }
    public Money LineTotal { get; init; }
}

public class CustomerSummaryDto
{
    public Guid Id { get; init; }
    public string Name { get; init; }
    public string Email { get; init; }
    public Money TotalSpent { get; init; }
    public int OrderCount { get; init; }
    public Money CreditLimit { get; init; }
}
```

---

### Step 2: Command Handlers

```csharp
namespace TestNest.CleanArchitecture.CQRS.Commands;

public interface ICommandHandler<TCommand, TResult>
{
    Task<TResult> Handle(TCommand command, CancellationToken cancellationToken);
}

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
                                           CancellationToken cancellationToken)
    {
        // Validate command
        var validation = await ValidateCommand(command);
        if (validation.IsFailure)
            return Result<OrderId>.Failure(validation.Error);

        // Get customer
        var customerResult = await _customerService.GetByIdAsync(command.CustomerId);
        if (customerResult.IsFailure)
            return Result<OrderId>.Failure(customerResult.Error);

        var customer = customerResult.Value;

        // Check credit
        var creditCheckResult = await CheckCreditLimit(customer, command);
        if (creditCheckResult.IsFailure)
            return Result<OrderId>.Failure(creditCheckResult.Error);

        // Validate stock
        var stockResult = await ValidateStock(command.Items);
        if (stockResult.IsFailure)
            return Result<OrderId>.Failure(stockResult.Error);

        // Create order
        var order = new Order(customer.Id, /*...*/);

        // Persist
        var saved = await _repository.AddAsync(order);

        // Emit event
        await _eventBus.PublishAsync(new OrderCreatedEvent(saved.Id));

        return Result<OrderId>.Success(saved.Id);
    }
}
```

### Step 3: Query Handlers

```csharp
namespace TestNest.CleanArchitecture.CQRS.Queries;

public interface IQueryHandler<TQuery, TResult>
{
    Task<TResult> Handle(TQuery query, CancellationToken cancellationToken);
}

public class GetOrderByIdQueryHandler : IQueryHandler<GetOrderByIdQuery, OrderDto>
{
    private readonly IOrderReadRepository _repository;

    public GetOrderByIdQueryHandler(IOrderReadRepository repository)
    {
        _repository = repository;
    }

    public async Task<OrderDto?> Handle(GetOrderByIdQuery query,
                                           CancellationToken cancellationToken)
    {
        var order = await _repository.GetByIdAsync(query.OrderId);
        
        if (order == null)
            return null;

        return new OrderDto
        {
            Id = order.Id.Value,
            CustomerName = order.Customer.Name,
            TotalAmount = order.TotalAmount,
            Status = order.Status.ToString(),
            CreatedAt = order.CreatedAt,
            Items = order.Items.Select(i => new OrderItemDto
            {
                Id = i.Id.Value,
                ProductName = i.Product.Name,
                UnitPrice = i.UnitPrice,
                Quantity = i.Quantity,
                LineTotal = i.LineTotal
            }).ToList()
        };
    }
}

public class GetOrdersByCustomerQueryHandler : IQueryHandler<GetOrdersByCustomerQuery, List<OrderDto>>
{
    private readonly IOrderReadRepository _repository;
    private readonly ICacheService _cache;

    public GetOrdersByCustomerQueryHandler(IOrderReadRepository repository, ICacheService cache)
    {
        _repository = repository;
        _cache = cache;
    }

    public async Task<List<OrderDto>> Handle(GetOrdersByCustomerQuery query,
                                                CancellationToken cancellationToken)
    {
        // Cache key
        var cacheKey = $"orders:{query.CustomerId.Value}:{query.Status}:{query.Page}:{query.PageSize}";

        // Try cache first
        var cached = await _cache.GetAsync<List<OrderDto>>(cacheKey);
        if (cached != null)
            return cached;

        // Fetch from database
        var orders = await _repository.GetByCustomerAsync(
            query.CustomerId,
            query.Status,
            query.Page,
            query.PageSize);

        // Cache for future requests
        await _cache.SetAsync(cacheKey, orders, TimeSpan.FromMinutes(5));

        return orders.Select(o => ToDto(o)).ToList();
    }
}
```

---

### Step 4: Event Definitions

```csharp
namespace TestNest.CleanArchitecture.CQRS.Events;

public record OrderCreatedEvent(OrderId OrderId);
public record OrderUpdatedEvent(OrderId OrderId);
public record OrderPaidEvent(OrderId OrderId, Money Amount);
public record OrderShippedEvent(OrderId OrderId);
public record OrderDeliveredEvent(OrderId OrderId);
public record OrderCanceledEvent(OrderId OrderId, string Reason);
```

---

## Complete Implementation

### File Structure

```
TestNest.CleanArchitecture.CQRS/
├── Commands/
│   ├── Handlers/
│   │   ├── CreateOrderCommandHandler.cs
│   │   ├── UpdateCustomerCommandHandler.cs
│   │   └── CancelOrderCommandHandler.cs
│   └── Models/
│       └── CreateOrderCommand.cs
├── Queries/
│   ├── Handlers/
│   │   ├── GetOrderByIdQueryHandler.cs
│   │   ├── GetOrdersByCustomerQueryHandler.cs
│   │   ├── SearchProductsQueryHandler.cs
│   │   └── GetDashboardStatsQueryHandler.cs
│   └── Models/
│       └── GetOrderByIdQuery.cs
└── Events/
    ├── OrderCreatedEvent.cs
    ├── OrderPaidEvent.cs
    ├── OrderShippedEvent.cs
    ├── OrderDeliveredEvent.cs
    └── OrderCanceledEvent.cs
```

---

## Testing Command Handlers

```csharp
public class CreateOrderCommandHandlerTests
{
    [Fact]
    public async Task Handle_ValidCommand_ReturnsOrderId()
    {
        var handler = new CreateOrderCommandHandler(_repository, _eventBus);
        var command = new CreateOrderCommand(/*...*/);

        var result = await handler.Handle(command, CancellationToken.None);

        Assert.IsTrue(result.IsSuccess);
        Assert.True(result.IsSuccess);
    }

    [Fact]
    public async Task Handle_InvalidCustomer_ReturnsFailure()
    {
        var handler = new CreateOrderHandler(_repository, _eventBus);
        var command = new CreateOrderCommand(/*non-existent customer*/);

        var result = await handler.Handle(command, CancellationToken.None);

        Assert.IsTrue(result.IsFailure);
        Assert.Contains("Customer not found", result.Error!.Message);
    }

    [Fact]
    public async Task Handle_InsufficientCredit_ReturnsFailure()
    {
        var handler = new CreateOrderCommandHandler(_repository, _eventBus);
        var command = new CreateOrderCommand(/*...high amount*/);

        var result = await handler.Handle(command, CancellationToken.None);

        Assert.IsTrue(result.IsFailure);
        Assert.Contains("Insufficient credit", result.Error!.Message);
    }
}
```

## Testing Query Handlers

```csharp
public class GetOrdersByCustomerQueryHandlerTests
{
    [Fact]
    public async Task Handle_FirstRequest_CachesResult()
    {
        var handler = new GetOrdersByCustomerQueryHandler(_repository, _cache);
        var query = new GetOrdersByCustomerQuery(/*...*/);

        var result1 = await handler.Handle(query, CancellationToken.None);
        Assert.NotNull(result1);

        // Second call should use cache
        var result2 = await handler.Handle(query, CancellationToken.None);
        Assert.NotNull(result2);
    }

    [Fact]
    public async Task Handle_Pagination_ReturnsCorrectPage()
    {
        var query = new GetOrdersByCustomerQuery(/*...*/);

        var result = await handler.Handle(query, CancellationToken.None);

        Assert.NotNull(result);
        Assert.Equal(10, result.Count);
        Assert.All(result.Select(r => r.Items.Any()));
    }
}
```

---

## Key Takeaways

- Command handlers handle state changes
- Query handlers return data
- Models separated for each concern
- Result pattern for error handling
- Caching can be added to queries
- Events provide decoupling
- Next: Advanced Event Sourcing in Chapter 4

---

## References

- [Source Code: CQRSCleanApi](https://github.com/DanteTuraSalvador/CQRSCleanApi)
- [MediatR: Mediator for CQRS](https://github.com/jbogard/MediatR)
- [Jimmy Bogard: CQRS pattern](https://www.jimmybogard.com/tags/cqrs/)

{% include tutorial-nav.html %}
