---
layout: post
title: "Chapter 2: Solution - SmartEnum Pattern"
date: 2025-01-09
category: ddd
tags:
  - ddd
  - smart-enums
  - beginner
series: smart-enums
chapter: 2
prerequisites: "Chapter 1"
estimated_time: "20 minutes"
prev_title: "Chapter 1: Problem - State Management"
prev_url: "/ddd/2025/01/08/smart-enums-chapter-01.html"
next_title: "Chapter 3: Implementation - CheckIn/CheckOut State Machine"
next_url: "/ddd/2025/01/10/smart-enums-chapter-03.html"
---

# Chapter 2: Solution - SmartEnum Pattern

## Learning Objectives

By the end of this chapter, you will understand:
- What is a SmartEnum
- Core principles of SmartEnum
- When to use SmartEnum
- Benefits of SmartEnum over primitive enums

---

## What is a SmartEnum?

A **SmartEnum** is a pattern that extends regular enums to provide:
- **State transition validation** - Only allow valid transitions
- **Business rule enforcement** - Encapsulate state-specific rules
- **Behavior encapsulation** - State-specific methods and properties
- **Type safety** - Strongly typed states
- **Immutability** - Once created, state cannot be changed
- **Extensibility** - Easy to add new states without breaking existing code

### Key Principles

1. **Type-safe states** - Each state is a specific type
2. **Valid transitions** - Only allowed state changes
3. **State-specific behavior** - Methods and properties per state
4. **Encapsulated validation** - Rules live with the state
5. **Immutable state changes** - Use factory methods for transitions
6. **Thread-safe** - No shared mutable state
7. **Self-documenting** - Clear what states exist and are valid

---

## SmartEnum vs Primitive Enum

| Aspect | SmartEnum | Primitive Enum |
|--------|-----------|---------------|
| State validation | Enforced | None |
| Business rules | Encapsulated | Scattered |
| Behavior | Per state | None |
| Type safety | Strong types | Shared enum |
| Transitions | Controlled | Unrestricted |
| Temporal constraints | Enforced | Manual |
| Immutability | Enforced | Optional |
| Extensibility | Easy | Breaks code |

---

## Example: SmartEnum Base Class

### Generic Implementation

```csharp
public abstract class SmartEnum<TEnum, TDerived>
    where TEnum : notnull
    where TDerived : SmartEnum<TEnum, TDerived>, new()
{
    public TEnum Value { get; }

    protected SmartEnum(TEnum value)
    {
        Value = value;
    }

    protected SmartEnum()
    {
        Value = default(TEnum)!;
    }

    public override bool Equals(object? obj)
    {
        if (obj is not TDerived other)
            return false;

        return Value!.Equals(other.Value);
    }

    public override int GetHashCode()
    {
        return Value!.GetHashCode();
    }

    public static bool operator ==(SmartEnum<TEnum, TDerived> left, SmartEnum<TEnum, TDerived> right)
    {
        if (left is null || right is null)
            return false;

        return left.Value!.Equals(right.Value!);
    }

    public static bool operator !=(SmartEnum<TEnum, TDerived> left, SmartEnum<TEnum, TDerived> right)
    {
        return !(left == right);
    }

    public override string ToString() => Value!.ToString()!;
}
```

---

## Example: CheckIn/CheckOut SmartEnum

### State Definition

```csharp
public enum CheckInOutStatus
{
    None,
    CheckIn,
    CheckOut
}

public class CheckInOut : SmartEnum<CheckInOutStatus, CheckInOut>
{
    // Factory methods
    public static readonly CheckInOut None = new NoneCheckInOut();
    public static readonly CheckInOut CheckedIn = new CheckedInCheckInOut();
    public static readonly CheckInOut CheckedOut = new CheckedOutCheckInOut();

    // State properties
    public DateTime? Time { get; private set; }
    public Guid? VisitId { get; private set; }

    // State-specific behaviors
    public bool IsActive => Value == CheckInOutStatus.CheckIn;
    public bool IsCompleted => Value == CheckInOutStatus.CheckOut;
    public bool CanTransitionTo(CheckInOutStatus newStatus)
    {
        // Validate transitions
        return Value switch
        {
            CheckInOutStatus.None => newStatus == CheckInOutStatus.CheckIn,
            CheckInOutStatus.CheckIn => newStatus == CheckInOutStatus.CheckOut || newStatus == CheckInOutStatus.CheckIn,
            CheckInOutStatus.CheckOut => newStatus == CheckInOutStatus.CheckOut,
            _ => false
        };
    }

    // Transition factory methods
    public static CheckInOut CreateCheckIn(DateTime checkInTime, Guid visitId)
    {
        return new CheckedInCheckInOut(checkInTime, visitId);
    }

    public static CheckInOut CreateCheckOut(DateTime checkOutTime, Guid visitId)
    {
        return new CheckedOutCheckInOut(checkOutTime, visitId);
    }

    // Duration calculation
    public TimeSpan? GetDuration()
    {
        if (Time == null)
            return null;

        if (CheckedOut.Time == null)
            return null;

        return CheckedOut.Time.Value - Time.Value;
    }
}
```

### State Implementations

```csharp
// None state
private class NoneCheckInOut : CheckInOut
{
    public NoneCheckInOut()
    {
    }
}

// CheckedIn state
private class CheckedInCheckInOut : CheckInOut
{
    public CheckedInCheckInOut(DateTime checkInTime, Guid visitId)
        : base(CheckInOutStatus.CheckIn)
    {
        Time = checkInTime;
        VisitId = visitId;
    }
}

// CheckedOut state
private class CheckedOutCheckInOut : CheckInOut
{
    public CheckedOutCheckInOut(DateTime checkOutTime, Guid visitId)
        : base(CheckInOutStatus.CheckOut)
    {
        Time = checkOutTime;
        VisitId = visitId;
    }
}
```

---

## Benefits of SmartEnum

### 1. State Transition Validation

```csharp
// Only valid transitions allowed
var checkIn = CheckInOut.CreateCheckIn(DateTime.Now, visitId);
var checkOut = CheckInOut.CreateCheckOut(DateTime.Now, visitId);

// Can only transition CheckIn → CheckOut or CheckIn → CheckIn
checkIn.CanTransitionTo(CheckInOutStatus.CheckOut); // True
checkIn.CanTransitionTo(CheckInOutStatus.None); // False
```

### 2. Business Rule Enforcement

```csharp
// Business rules live with the state
public bool CanTransitionTo(CheckInOutStatus newStatus)
{
    return Value switch
    {
        CheckInOutStatus.None => newStatus == CheckInOutStatus.CheckIn,
        CheckInOutStatus.CheckIn => newStatus == CheckInOutStatus.CheckOut || newStatus == CheckInOutStatus.CheckIn,
        CheckInOutStatus.CheckOut => newStatus == CheckInOutStatus.CheckOut,
        _ => false
    };
}
```

### 3. Behavior Encapsulation

```csharp
// State-specific methods
var checkIn = CheckInOut.CreateCheckIn(DateTime.Now, visitId);

// Only CheckIn state has IsActive
Console.WriteLine(checkIn.IsActive); // True

// Duration only available after CheckOut
var duration = checkIn.GetDuration(); // null

var checkOut = CheckInOut.CreateCheckOut(DateTime.Now, visitId);
var visitDuration = checkOut.GetDuration(); // TimeSpan
```

### 4. Type Safety

```csharp
// Can't mix different SmartEnums
void ProcessCheckIn(CheckInOut checkIn) { }

// Cannot accidentally pass different SmartEnum type
// ProcessCheckIn(someOtherSmartEnum); // Compiler error!
```

---

## When to Use SmartEnum

**Use SmartEnum when:**
- You have complex state management
- Multiple states with transition rules
- Business rules vary by state
- You need state-specific behavior
- Temporal constraints matter
- Audit trail of state changes needed

**Examples:**
- Order status (Created → Paid → Shipped → Delivered)
- Visit tracking (CheckIn → CheckOut)
- Appointment status (Requested → Confirmed → Completed)
- Document status (Draft → Review → Approved → Published)

---

## When NOT to Use SmartEnum

**Don't use when:**
- Simple enum without business rules
- No state transitions
- No state-specific behavior
- Performance is critical (minimal overhead needed)
- Only 2-3 static states with no logic

**Use primitive enum instead when:**
- Just need type safety
- No business rules
- No transitions
- Simple state list

---

## Real-World Integration

```csharp
public class Visit
{
    public VisitId Id { get; }
    public GuestId GuestId { get; }
    public VisitPurpose Purpose { get; }
    public CheckInOut CheckInOut { get; private set; }

    public Visit(VisitId id, GuestId guestId, VisitPurpose purpose, CheckInOut checkInOut)
    {
        Id = id;
        GuestId = guestId;
        Purpose = purpose;
        CheckInOut = checkInOut;
    }

    public Result<CheckInOut> CheckIn(DateTime checkInTime)
    {
        var checkInOut = CheckInOut.CreateCheckIn(checkInTime, Id.Value);

        if (!CheckInOut.CanTransitionTo(CheckInOutStatus.CheckIn))
            return Result.Failure<CheckInOut>("Invalid transition: Not checked out yet");

        CheckInOut = checkInOut;
        return Result.Success(CheckInOut);
    }

    public Result<CheckInOut> CheckOut(DateTime checkOutTime)
    {
        if (!CheckInOut.IsActive)
            return Result.Failure<CheckInOut>("Must be checked in first");

        var checkOut = CheckInOut.CreateCheckOut(checkOutTime, Id.Value);
        CheckInOut = checkOut;
        return Result.Success(CheckInOut);
    }
}
```

---

## Key Takeaways

- SmartEnum provides state transition validation
- They encapsulate business rules
- They enable state-specific behavior
- Use when you have complex state management
- Next: Implement a real CheckIn/CheckOut SmartEnum in Chapter 3

---

## References

- [Source Code: TestNest.SmartEnums](https://github.com/DanteTuraSalvador/TestNest.SmartEnums)
- [Martin Fowler: State Pattern](https://martinfowler.com/eaaCatalog/state.html)

