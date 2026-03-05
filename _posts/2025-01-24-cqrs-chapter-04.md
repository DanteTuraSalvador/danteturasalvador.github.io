---
layout: post
title: "Part 4: Advanced - Event Sourcing"
date: 2025-01-24
category: ddd
thumbnail-img: "https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=400&h=200&fit=crop"
tags:
  - ddd
  - cqrs
  - advanced
series: cqrs
chapter: 4
prerequisites: "Part 3"
estimated_time: "25 minutes"
prev_title: "Part 3: Implementation - Commands and Queries"
prev_url: "/ddd/2025/01/23/cqrs-chapter-03.html"
---

# Part 4: Advanced - Event Sourcing

## Learning Objectives

By the end of this chapter, you will understand:
- What is Event Sourcing
- Event Sourcing principles
- How Event Sourcing extends CQRS
- Command and event stores vs traditional persistence
- When to use Event Sourcing
- Rebuilding state from events

---

## What is Event Sourcing?

Event Sourcing is a pattern that:
- **Captures state changes** as events
- **Persists events** as authoritative source of truth
- **Rebuilds state** by replaying events
- **Provides audit trail** - Complete history
- **Enables temporal queries** - Any point in time
- **Supports eventual consistency** - System can be made distributed
- **Enables snapshotting** - Performance optimization

### Key Principles

1. **Event is immutable** - Events can't be changed
2. **Events are append-only** - No updates or deletes
3. **State derived from events** - Not stored directly
4. **Event log is source of truth** - Single source of truth
5. **Snapshots** provide checkpoints
6. **Optimistic concurrency** - Event logs handle conflicts
7. **Temporal data** - Built-in time-travel capability

---

## Event Sourcing vs Traditional Persistence

| Aspect | Event Sourcing | Traditional |
|--------|----------------|-------------|
| State storage | Event log only | Database tables |
| Audit trail | Built-in | Manual |
| Temporal queries | Supported | Difficult |
| Consistency | Eventual | Strong |
| Concurrency | Optimistic | Pessimistic |
| Scalability | Better | Limited |
| Performance | Slower writes | Faster |
| Complexity | Higher | Lower |
| Learning curve | Steep | Shallow |

---

## Event Store Architecture

```
┌─────────────────────────────────────┐
│          Command Side               │
├──────────────────┬──────────────────┤
│     Commands     │    Validation    │
└────────┬─────────┴──────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│           Event Store               │
├──────────────────┬──────────────────┤
│    Commands      │    Queries       │
│    Events        │    Snapshots     │
└────────┬─────────┴────────┬─────────┘
         │                  │
         ▼                  ▼
┌──────────────────┐ ┌────────────────┐
│   Write Model    │ │   Read Model   │
│  (Projections)   │ │    (Views)     │
└────────┬─────────┘ └────────┬───────┘
         │                    │
         ▼                    ▼
┌─────────────────────────────────────┐
│          Event Stream               │
│   ├─ Domain Events                  │
│   ├─ Integration Events             │
│   └─ System Events                  │
└─────────────────────────────────────┘
```

**Data Flow:**
- Commands → Event Store → Write Model (Append only)
- Queries → Event Store → Read Model (Projections)
- Event Stream → Read Model

---

## Event Implementation

### Domain Events

```csharp
namespace TestNest.CleanArchitecture.Domain.Events;

public abstract record DomainEvent
{
    public DateTime OccurredAt { get; }
    public Guid AggregateId { get; }
}

public record CustomerCreatedEvent : DomainEvent
{
    public CustomerId CustomerId { get; }
    public string Name { get; }
    public Email Email { get; }
    public DateTime CreatedAt { get; } = DateTime.UtcNow;
}

public record CustomerEmailChangedEvent : DomainEvent
{
    public CustomerId CustomerId { get; }
    public Email OldEmail { get; }
    public Email NewEmail { get; }
}

public record OrderCreatedEvent : DomainEvent
{
    public OrderId OrderId { get; }
    public CustomerId CustomerId { get; }
    public Money TotalAmount { get; }
}
```

### Entity with Event Generation

```csharp
public class Order
{
    private readonly List<DomainEvent> _events = new();

    public Order(OrderId id, CustomerId customerId, Money totalAmount)
    {
        Id = id;
        CustomerId = customerId;
        TotalAmount = totalAmount;
        Status = OrderStatus.Created;
        CreatedAt = DateTime.UtcNow;

        // Emit event on creation
        _events.Add(new OrderCreatedEvent(Id, CustomerId, totalAmount));
    }

    public void MarkAsPaid(Money paymentAmount)
    {
        if (Status != OrderStatus.PaymentProcessing)
            throw new DomainException("Order is not in payment processing");

        Status = OrderStatus.Paid;
        PaidAt = DateTime.UtcNow;

        _events.Add(new OrderPaidEvent(Id, paymentAmount));
    }

    public void MarkAsShipped()
    {
        if (Status != OrderStatus.Paid)
            throw new DomainException("Order must be paid before shipping");

        Status = OrderStatus.Shipped;
        ShippedAt = DateTime.UtcNow;

        _events.Add(new OrderShippedEvent(Id));
    }
}
```

---

## Event Store Implementation

### Event Store Interface

```csharp
public interface IEventStore
{
    Task<Guid> SaveEventAsync<TEvent>(TEvent @event);
    Task<TEvent?> GetEventAsync<TEvent>(Guid eventId);
    Task<List<TEvent>> GetEventsAsync<TEvent>(
        Guid aggregateId,
        int fromVersion,
        int toVersion);
    Task<int> GetVersionAsync(Guid aggregateId);
}
```

### In-Memory Event Store (Demo)

```csharp
public class InMemoryEventStore : IEventStore
{
    private readonly ConcurrentDictionary<Type, List<object>> _events = new();

    public async Task<Guid> SaveEventAsync<TEvent>(TEvent @event)
    {
        var eventType = typeof(TEvent);
        var events = _events.GetOrAdd(eventType);

        var eventId = Guid.NewGuid();
        events.Add(new EventWrapper(eventId, eventId, @event));

        return eventId;
    }

    public async Task<TEvent?> GetEventAsync<TEvent>(Guid eventId)
    {
        var eventType = typeof(TEvent);
        if (!_events.TryGetValue(eventType, out var eventsList))
            return default;

        var wrapper = eventsList.OfType<EventWrapper>()
                               .FirstOrDefault(w => w.EventId == eventId);

        return wrapper?.Event;
    }

    public async Task<List<TEvent>> GetEventsAsync<TEvent>(
        Guid aggregateId,
        int fromVersion = 0,
        int toVersion = int.MaxValue)
    {
        var eventType = typeof(TEvent);
        if (!_events.TryGetValue(eventType, out var eventsList))
            return new List<TEvent>();

        return eventsList.OfType<EventWrapper>()
                     .Where(w => w.AggregateId == aggregateId)
                     .Where(w => w.Version >= fromVersion && w.Version <= toVersion)
                     .Select(w => w.Event)
                     .Cast<TEvent>()
                     .ToList();
    }

    public async Task<int> GetVersionAsync(Guid aggregateId)
    {
        var allEvents = await GetAllEventsAsync(aggregateId);
        if (!allEvents.Any())
            return 0;

        return allEvents.Max(e => e.Version);
    }

    private record EventWrapper
    {
        public Guid AggregateId { get; }
        public Guid EventId { get; }
        public int Version { get; }
        public object Event { get; }
        public DateTime OccurredAt { get; }
    }
}
```

---

## Event Projection Implementation

### Write Model (Projections)

```csharp
public class OrderReadModel
{
    public Guid Id { get; set; }
    public CustomerId CustomerId { get; set; }
    public string CustomerName { get; set; }
    public Money TotalAmount { get; set; }
    public string Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? PaidAt { get; set; }
    public DateTime? ShippedAt { get; set; }
    public DateTime? DeliveredAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public List<OrderItemReadModel> Items { get; } = new();

    // Computed from events
    public int TotalItems => Items.Sum(i => i.Quantity);
}
```

### Projection from Events

```csharp
public class OrderProjection
{
    public static OrderReadModel Project(IEnumerable<DomainEvent> events)
    {
        var orderEvents = events.Where(e => e is OrderCreatedEvent
                                      || e is OrderPaidEvent
                                      || e is OrderShippedEvent
                                      || e is OrderDeliveredEvent);

        var firstEvent = orderEvents.First();
        var order = new OrderReadModel
        {
            Id = firstEvent.AggregateId,
            CreatedAt = firstEvent.OccurredAt,
            UpdatedAt = orderEvents.Last().OccurredAt
        };

        // Apply events in order
        foreach (var e in orderEvents.Skip(1))
        {
            switch (e)
            {
                case OrderPaidEvent paid:
                    order.PaidAt = e.OccurredAt;
                    order.TotalAmount = order.TotalAmount + paid.Amount;
                    order.Status = OrderStatus.Paid.ToString();
                    break;

                case OrderShippedEvent shipped:
                    order.ShippedAt = e.OccurredAt;
                    order.Status = OrderStatus.Shipped.ToString();
                    break;

                case OrderDeliveredEvent delivered:
                    order.DeliveredAt = e.OccurredAt;
                    order.Status = OrderStatus.Delivered.ToString();
                    break;
            }
        }

        return order;
    }
}
```

---

## Event Sourcing with CQRS

### Command Handler with Event Sourcing

```csharp
public class CreateOrderCommandHandler : ICommandHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IEventStore _eventStore;

    public CreateOrderCommandHandler(IEventStore eventStore)
    {
        _eventStore = eventStore;
    }

    public async Task<Result<OrderId>> Handle(CreateOrderCommand command,
                                           CancellationToken cancellationToken)
    {
        // Validate command
        var validation = await ValidateCommand(command);
        if (validation.IsFailure)
            return Result<OrderId>.Failure(validation.Error);

        // Create order entity
        var order = new Order(/*...*/);

        // Save event (source of truth)
        var eventId = await _eventStore.SaveEventAsync(
            new OrderCreatedEvent(order.Id, order.CustomerId, order.TotalAmount));

        // Persist projection
        var orderModel = OrderProjection.Project(
            new[] { new OrderCreatedEvent(order.Id, order.CustomerId, order.TotalAmount) });

        await _writeRepository.AddAsync(order);

        return Result<OrderId>.Success(order.Id);
    }
}
```

### Query Handler with Event Sourcing

```csharp
public class GetOrdersQueryHandler : IQueryHandler<GetOrdersByCustomerQuery, List<OrderReadModel>>
{
    private readonly IEventStore _eventStore;

    public GetOrdersQueryHandler(IEventStore eventStore)
    {
        _eventStore = eventStore;
    }

    public async Task<List<OrderReadModel>> Handle(GetOrdersByCustomerQuery query,
                                                CancellationToken cancellationToken)
    {
        // Get events for customer's orders
        var events = await _eventStore.GetEventsAsync<DomainEvent>(
            query.CustomerId, 0, int.MaxValue);

        // Project to read model
        var orderModels = OrderProjection.Project(events);

        return orderModels;
    }
}
```

---

## Testing Event Sourcing

```csharp
[Test]
public void EventSourcing_RebuildsStateCorrectly()
{
    var events = new List<DomainEvent>
    {
        new OrderCreatedEvent(orderId, customerId, totalAmount),
        new OrderPaidEvent(orderId, paymentAmount),
        new OrderShippedEvent(orderId),
        new OrderDeliveredEvent(orderId)
    };

    var store = new InMemoryEventStore();
    await store.SaveEventAsync(events[0]);
    await store.SaveEventAsync(events[1]);
    await store.SaveEventAsync(events[2]);
    await store.SaveEventAsync(events[3]);

    var readModel = OrderProjection.Project(events);

    Assert.Equal(OrderStatus.Delivered, readModel.Status);
    Assert.Equal(deliveredAt, readModel.UpdatedAt);
    Assert.Equal(expectedTotal, readModel.TotalAmount);
}
```

---

## Event Store Snapshots

```csharp
public interface ISnapshotStore
{
    Task<Guid> CreateSnapshotAsync<T>(T aggregate, string snapshotData);
    Task RestoreSnapshotAsync<T>(T aggregate, Guid snapshotId);
    Task<List<Guid>> GetSnapshotsAsync<T>(Guid aggregateId);
}

public class Snapshot<T>
{
    public Guid Id { get; }
    public T Aggregate { get; }
    public string SnapshotData { get; }
    public DateTime CreatedAt { get; }
}
```

### Creating Snapshots

```csharp
public class SnapshotService : ISnapshotStore
{
    private readonly IEventStore _eventStore;

    public SnapshotService(IEventStore eventStore)
    {
        _eventStore = eventStore;
    }

    public async Task<Guid> CreateSnapshotAsync<T>(T aggregate, string description)
    {
        var eventId = await _eventStore.SaveEventAsync(
            new SnapshotCreatedEvent<T>(aggregate.Id, description));

        return eventId;
    }

    public async Task RestoreSnapshotAsync<T>(T aggregate, Guid snapshotId)
    {
        // Get snapshot event
        var snapshotEvent = await _eventStore.GetEventAsync<SnapshotCreatedEvent<T>>(
            snapshotId);

        if (snapshotEvent == null)
            throw new DomainException("Snapshot not found");

        // Restore from snapshot
        aggregate = snapshotEvent.Aggregate;
        aggregate.RestoredAt = snapshotEvent.CreatedAt;

        return aggregate.Id;
    }
}
```

---

## Real-World: Complete CQRS + Event Sourcing

### Order Lifecycle with Events

```csharp
// Order created
POST /api/orders (command)
→ EventStore: OrderCreatedEvent
→ WriteModel: OrderCreated
→ CustomerService: Email notification
→ Repository: Persist order projection

// Order paid
POST /api/orders/{id}/pay (command)
→ EventStore: OrderPaidEvent
→ UpdateModel: OrderPaid
→ Notification: Payment confirmation

// Order shipped
POST /api/orders/{id}/ship (command)
→ EventStore: OrderShippedEvent
→ UpdateModel: OrderShipped
→ Notification: Shipping confirmation

// Order delivered
POST /api/orders/{id}/deliver (command)
→ EventStore: OrderDeliveredEvent
→ UpdateModel: OrderDelivered
→ Notification: Delivery confirmation

// Query orders by customer
GET /api/customers/{id}/orders (query)
→ EventStore: Get events
→ Project to read model
→ Return: List<OrderReadModel>
```

### Optimistic Concurrency

```csharp
public class OrderCommandHandler : ICommandHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IEventStore _eventStore;
    private readonly IWriteLock _writeLock;

    public OrderCommandHandler(IEventStore eventStore, IWriteLock writeLock)
    {
        _eventStore = eventStore;
        _writeLock = writeLock;
    }

    public async Task<Result<OrderId>> Handle(CreateOrderCommand command,
                                           CancellationToken cancellationToken)
    {
        // Validate command
        var validation = await ValidateCommand(command);
        if (validation.IsFailure)
            return Result<OrderId>.Failure(validation.Error);

        // Check for duplicate order
        var existingOrder = await CheckForDuplicateOrder(command);

        // Load current state
        var currentState = await _eventStore.GetEventsAsync<OrderCreatedEvent>(
            command.CustomerId, 0, int.MaxValue);

        // Generate new state
        var eventId = await _eventStore.SaveEventAsync(
            new OrderCreatedEvent(orderId, command.CustomerId, command.TotalAmount));

        return Result<OrderId>.Success(orderId);
    }

    private async Task<Result<OrderId>> CheckForDuplicateOrder(CreateOrderCommand command)
    {
        var recentEvents = await _eventStore.GetEventsAsync<DomainEvent>(
            command.CustomerId, 0, int.MaxValue);

        // Check for recent OrderCreatedEvent (within 5 minutes)
        var recentCreated = recentEvents
            .Where(e => e is OrderCreatedEvent)
            .Where(e => DateTime.UtcNow - e.OccurredAt < TimeSpan.FromMinutes(5));

        var hasRecentOrder = recentCreated
            .Any(e => ((OrderCreatedEvent)e).CustomerId == command.CustomerId &&
                        ((OrderCreatedEvent)e).TotalAmount == command.TotalAmount);

        if (hasRecentOrder)
            return Result<OrderId>.Failure("Duplicate recent order");

        return Result<OrderId>.Success(default);
    }
}
```

---

## Key Takeaways

- Event sourcing provides complete audit trail
- State is immutable and derived from events
- Any point in time can be queried
- CQRS + Event Sourcing is powerful combination
- Enables temporal queries
- Performance can be optimized separately
- Testing involves event replay
- Series complete with 20 chapters! All patterns covered

---

## References

- [Source Code: CQRSCleanApi](https://github.com/DanteTuraSalvador/CQRSCleanApi)
- [Martin Fowler: Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing)
- [Greg Young: CQRS + Event Sourcing](https://www.youtube.com/watch?v=8hkXbE1Rg)
- [Microsoft: Event Sourcing with EF Core](https://docs.microsoft.com/en-us/ef/core/modeling/draft/design/event-sourcing)

