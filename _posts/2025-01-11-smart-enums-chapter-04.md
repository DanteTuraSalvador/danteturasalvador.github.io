---
layout: post
title: "Part 4: Advanced - Temporal Validation & Business Rules"
date: 2025-01-11
category: ddd
tags:
  - ddd
  - smart-enums
  - advanced
series: smart-enums
chapter: 4
prerequisites: "Part 3"
estimated_time: "25 minutes"
prev_title: "Part 3: Implementation - CheckIn/CheckOut State Machine"
prev_url: "/ddd/2025/01/10/smart-enums-chapter-03.html"
---

# Part 4: Advanced - Temporal Validation & Business Rules

## Learning Objectives

By the end of this chapter, you will understand:
- How to add temporal constraints to SmartEnums
- Business rule patterns in state machines
- Performance optimizations for SmartEnums
- Real-world complex state machines
- Testing advanced state logic

---

## Temporal Validation in SmartEnum

### Time-Based Constraints

```csharp
public abstract class CheckInOut : SmartEnum<CheckInOutStatus, CheckInOut>
{
    public DateTime? Time { get; protected set; }

    // Temporal validation settings
    protected static readonly TimeSpan MaxCheckInFuture = TimeSpan.FromMinutes(5);
    protected static readonly TimeSpan MaxCheckInPast = TimeSpan.FromSeconds(5);

    protected virtual bool ValidateTime(DateTime time, bool isCheckIn)
    {
        var now = DateTime.UtcNow;

        if (isCheckIn)
        {
            if (time > now + MaxCheckInFuture)
                throw new SmartEnumException($"Check-in time cannot be more than {MaxCheckInFuture.TotalMinutes} minutes in the future");

            if (time < now - MaxCheckInPast)
                throw new SmartEnumException($"Check-in time cannot be more than {MaxCheckInPast.TotalSeconds} seconds in the past");
        }
        else // Check-out
        {
            if (time > now + MaxCheckInFuture)
                throw new SmartEnumException($"Check-out time cannot be more than {MaxCheckInFuture.TotalMinutes} minutes in the future");
        }
    }
}
```

### Enhanced Factory Methods with Temporal Validation

```csharp
public class CheckedInCheckInOut : CheckInOut
{
    public static CheckedInCheckInOut Create(DateTime checkInTime, Guid visitId)
    {
        ValidateTime(checkInTime, isCheckIn: true);
        return new CheckedInCheckInOut(checkInTime.ToUniversalTime(), visitId);
    }
}

public class CheckedOutCheckInOut : CheckInOut
{
    public static CheckedOutCheckInOut Create(DateTime checkOutTime, Guid visitId)
    {
        ValidateTime(checkOutTime, isCheckIn: false);
        return new CheckedOutCheckInOut(checkOutTime.ToUniversalTime(), visitId);
    }
}
```

---

## Business Rule Patterns

### Pattern 1: State-Dependent Rules

```csharp
public class Visit
{
    public VisitId Id { get; }
    public GuestId GuestId { get; }
    public VisitPurpose Purpose { get; }
    public CheckInOut CheckInOut { get; private set; }

    // Business rule: Can't have multiple active visits
    private static readonly ConcurrentDictionary<GuestId, DateTime> _activeGuests = new();

    public Result<CheckInOut> CheckIn(DateTime checkInTime, GuestId guestId)
    {
        // Check if guest already has active visit
        if (_activeGuests.ContainsKey(guestId))
        {
            var activeTime = _activeGuests[guestId];
            if (DateTime.UtcNow - activeTime < TimeSpan.FromHours(2))
            {
                return Result.Failure<CheckInOut>("Guest has an active visit within last 2 hours");
            }
        }

        // Check in
        var result = CheckInOut.CreateCheckIn(checkInTime.ToUniversalTime(), Id.Value);

        if (result.IsSuccess)
        {
            _activeGuests[guestId] = checkInTime.ToUniversalTime();
        }

        return result;
    }

    public Result<CheckInOut> CheckOut(DateTime checkOutTime, GuestId guestId)
    {
        // Validate guest is visiting
        if (!_activeGuests.ContainsKey(guestId))
            return Result.Failure<CheckInOut>("Guest has no active visit");

        var result = CheckInOut.CreateCheckOut(checkOutTime.ToUniversalTime(), Id.Value);

        if (result.IsSuccess)
        {
            // Remove from active visits
            _activeGuests.TryRemove(guestId, out _);
        }

        return result;
    }
}
```

### Pattern 2: Time-Limited States

```csharp
public class TimeLimitedCheckInOut : CheckInOut
{
    private DateTime? _checkInTime;
    private static readonly TimeSpan MaxVisitDuration = TimeSpan.FromHours(24);

    public TimeLimitedCheckInOut(DateTime checkInTime, Guid visitId)
        : base(CheckInOutStatus.CheckIn)
    {
        _checkInTime = checkInTime.ToUniversalTime();
        VisitId = visitId;
    }

    public override bool CanTransitionFrom(CheckInOutStatus fromStatus)
    {
        if (fromStatus == CheckInOutStatus.CheckOut)
        {
            // Can't check back in if visit duration exceeded
            if (_checkInTime.HasValue)
            {
                var duration = DateTime.UtcNow - _checkInTime.Value;
                if (duration > MaxVisitDuration)
                    return false;
            }
        }

        return base.CanTransitionFrom(fromStatus);
    }

    public TimeSpan? GetRemainingTime()
    {
        if (!IsActive || !_checkInTime.HasValue)
            return null;

        var elapsed = DateTime.UtcNow - _checkInTime.Value;
        var remaining = MaxVisitDuration - elapsed;

        return remaining.TotalSeconds > 0 ? remaining : TimeSpan.Zero;
    }
}
```

---

## Performance Optimizations

### Optimization 1: Lazy State Initialization

```csharp
// Thread-safe singleton for None state
public class NoneCheckInOut : CheckInOut
{
    private static readonly Lazy<NoneCheckInOut> _instance =
        new Lazy<NoneCheckInOut>(() => new NoneCheckInOut());

    public static NoneCheckInOut Instance => _instance.Value;

    private NoneCheckInOut()
        : base(CheckInOutStatus.None)
    {
    }
}
```

### Optimization 2: Avoid Boxed Comparisons

```csharp
public abstract class SmartEnum<TEnum, TDerived>
    where TEnum : notnull
    where TDerived : SmartEnum<TEnum, TDerived>, new()
{
    // Direct comparison without boxing
    public override bool Equals(object? obj)
    {
        return obj is TDerived other && Value!.Equals(other.Value);
    }

    // Efficient hash code
    public override int GetHashCode()
    {
        return Value!.GetHashCode();
    }
}
```

### Performance Metrics

From TestNest.SmartEnums benchmarking:

| Operation | Time | Notes |
|-----------|------|-------|
| Status Transition | 0.45 μs | Including validation |
| Duration Calculation | 0.12 μs | Simple subtraction |
| Equality Check | 0.35 μs | Value-based comparison |
| Factory Method Call | 0.15 μs | With validation |

**SmartEnum overhead is negligible** for most applications.

---

## Real-World Complex State Machine: Order Processing

```csharp
public enum OrderStatus
{
    Created,
    PaymentProcessing,
    Paid,
    Shipped,
    Delivered,
    Returned,
    Canceled
}

public class OrderProcessing : SmartEnum<OrderStatus, OrderProcessing>
{
    public DateTime? CreatedAt { get; private set; }
    public DateTime? PaidAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public DateTime? DeliveredAt { get; private set; }
    public Money? TotalAmount { get; private set; }
    public List<Money> RefundAmounts { get; } = new();

    public override bool CanTransitionFrom(OrderStatus fromStatus)
    {
        return (fromStatus, Value) switch
        {
            (OrderStatus.Created, OrderStatus.PaymentProcessing) => true,
            (OrderStatus.PaymentProcessing, OrderStatus.Paid) => true,
            (OrderStatus.Paid, OrderStatus.Shipped) => true,
            (OrderStatus.Shipped, OrderStatus.Delivered) => true,
            (OrderStatus.Delivered, OrderStatus.Returned) => CanReturn(),
            (OrderStatus.Shipped, OrderStatus.Canceled) => CanCancel(),
            _ => false
        };
    }

    private bool CanReturn()
    {
        // Can return within 30 days of delivery
        if (DeliveredAt.HasValue)
        {
            var daysSinceDelivery = DateTime.UtcNow - DeliveredAt.Value;
            return daysSinceDelivery < TimeSpan.FromDays(30);
        }

        return false;
    }

    private bool CanCancel()
    {
        // Can cancel if not shipped yet
        return !ShippedAt.HasValue && !DeliveredAt.HasValue;
    }
}

public class CreatedOrder : OrderProcessing
{
    public static CreatedOrder Create(Money totalAmount)
    {
        return new CreatedOrder
        {
            CreatedAt = DateTime.UtcNow,
            TotalAmount = totalAmount
        };
    }
}
```

---

## Testing Complex State Machines

```csharp
[Test]
public void Order_FullWorkflow_Succeeds()
{
    var order = new Order(orderId, customerId, /*...*/);

    // Created → PaymentProcessing → Paid → Shipped → Delivered
    Assert.IsTrue(order.ProcessPayment().IsSuccess);
    Assert.IsTrue(order.MarkPaid().IsSuccess);
    Assert.IsTrue(order.Ship().IsSuccess);
    Assert.IsTrue(order.Deliver().IsSuccess);

    Assert.AreEqual(OrderStatus.Delivered, order.Status.Value);
}

[Test]
public void Order_CancelBeforeShipment_Succeeds()
{
    var order = new Order(/*...*/);
    order.ProcessPayment();

    var result = order.Cancel();
    Assert.IsTrue(result.IsSuccess);
    Assert.AreEqual(OrderStatus.Canceled, order.Status.Value);
}

[Test]
public void Order_CannotCancelAfterShipment_ReturnsFailure()
{
    var order = new Order(/*...*/);
    order.ProcessPayment();
    order.MarkPaid();
    order.Ship();

    var result = order.Cancel();
    Assert.IsTrue(result.IsFailure);
    Assert.AreEqual("Cannot cancel order that has been shipped", result.Error);
}
```

---

## Best Practices for Advanced SmartEnums

1. **Validate temporal constraints** in factory methods
2. **Use UTC times** consistently
3. **Implement business rules** in CanTransitionFrom
4. **Add state-specific properties** and methods
5. **Use lazy singletons** for frequently used states
6. **Consider performance** - avoid boxing where possible
7. **Test all transition paths**
8. **Document state diagrams**
9. **Log state transitions** for auditing
10. **Keep states small and focused**

---

## Complete Source Code

**SmartEnum Base:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.SmartEnums/blob/main/TestNest.SmartEnums.Domain/ValueObjects/Common/SmartEnum.cs)

**CheckInOut Implementation:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.SmartEnums/blob/main/TestNest.SmartEnums.Domain/ValueObjects/CheckInOut.cs)

**Order Processing Example:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.SmartEnums/blob/main/TestNest.SmartEnums.Domain/Entities/OrderProcessing.cs)

---

## Key Takeaways

- SmartEnum supports temporal validation through factory methods
- Business rules enforce state transitions
- Performance optimizations are simple but effective
- Test all state transition paths thoroughly
- Series complete! Next: Result Pattern

---

## What's Next?

Congratulations on completing Smart Enums series!

Continue your DDD journey with:
- [Result Pattern](/2025/01/12/result-pattern-chapter-01.html) - Functional error handling
- [Clean Architecture](#) - Architectural patterns
- [CQRS](#) - Command Query Separation

