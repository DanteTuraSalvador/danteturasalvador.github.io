---
layout: post
title: "Chapter 3: Implementation - CheckIn/CheckOut State Machine"
date: 2025-01-10
tags:
  - ddd
  - smart-enums
  - implementation
series: smart-enums
chapter: 3
prerequisites: "Chapter 2"
estimated_time: "30 minutes"
prev_title: "Chapter 2: Solution - SmartEnum Pattern"
prev: "/2025/01/09/smart-enums-chapter-02.html"
next_title: "Chapter 4: Advanced - Temporal Validation & Business Rules"
next: "/2025/01/11/smart-enums-chapter-04.html"
---

# Chapter 3: Implementation - CheckIn/CheckOut State Machine

## Learning Objectives

By the end of this chapter, you will be able to:
- Implement a SmartEnum base class
- Create a concrete CheckIn/CheckOut state machine
- Add state transition validation
- Implement state-specific behaviors
- Test your SmartEnum implementation

---

## Implementation: Step by Step

### Step 1: Create SmartEnum Base Class

```csharp
public abstract class SmartEnum<TEnum, TDerived>
    where TEnum : notnull
    where TDerived : SmartEnum<TEnum, TDerived>, new()
{
    private static readonly Lazy<Dictionary<string, TEnum>> _allValues =
        new Lazy<Dictionary<string, TEnum>>(() =>
        {
            var enumValues = Enum.GetValues(typeof(TEnum));
            var dict = new Dictionary<string, TEnum>();

            foreach (TEnum value in enumValues)
            {
                dict[value.ToString()!] = value;
            }

            return dict;
        });

    public TEnum Value { get; }

    protected SmartEnum(TEnum value)
    {
        Value = value;
    }

    public override string ToString() => Value.ToString()!;

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

    public override int GetHashCode()
    {
        return Value!.GetHashCode();
    }

    public static TDerived FromValue(TEnum value)
    {
        var stateType = typeof(TDerived);
        var constructors = stateType.GetConstructors(BindingFlags.NonPublic | BindingFlags.Instance);

        foreach (var ctor in constructors)
        {
            var parameters = ctor.GetParameters();
            if (parameters.Length == 1 && parameters[0].ParameterType == typeof(TEnum))
            {
                var instance = (TDerived)ctor.Invoke(new object[] { value });
                return instance;
            }
        }

        throw new InvalidOperationException($"No constructor found for {typeof(TDerived).Name} with {typeof(TEnum).Name} parameter");
    }

    public static IEnumerable<TDerived> GetAll()
    {
        return Enum.GetValues(typeof(TEnum))
            .Select(FromValue);
    }
}
```

---

### Step 2: Define CheckInOutStatus Enum

```csharp
public enum CheckInOutStatus
{
    None,
    CheckIn,
    CheckOut
}
```

---

### Step 3: Implement CheckInOut SmartEnum

```csharp
public abstract class CheckInOut : SmartEnum<CheckInOutStatus, CheckInOut>
{
    public DateTime? Time { get; protected set; }
    public Guid? VisitId { get; protected set; }

    protected CheckInOut(CheckInOutStatus value) : base(value)
    {
    }

    public abstract bool IsActive { get; }
    public abstract bool IsCompleted { get; }
    public abstract bool CanTransitionFrom(CheckInOutStatus fromStatus);

    public TimeSpan? GetDuration()
    {
        if (Time == null)
            return null;

        if (!IsCompleted)
            return null;

        return DateTime.UtcNow - Time;
    }
}
```

---

### Step 4: Implement Specific States

```csharp
public class NoneCheckInOut : CheckInOut
{
    public NoneCheckInOut()
        : base(CheckInOutStatus.None)
    {
    }

    public override bool IsActive => false;
    public override bool IsCompleted => false;

    public override bool CanTransitionFrom(CheckInOutStatus fromStatus)
    {
        // Can only start from None state
        return fromStatus == CheckInOutStatus.None;
    }
}

public class CheckedInCheckInOut : CheckInOut
{
    public CheckedInCheckInOut(DateTime checkInTime, Guid visitId)
        : base(CheckInOutStatus.CheckIn)
    {
        Time = checkInTime.ToUniversalTime();
        VisitId = visitId;
    }

    public override bool IsActive => true;
    public override bool IsCompleted => false;

    public override bool CanTransitionFrom(CheckInOutStatus fromStatus)
    {
        // Can transition from None or CheckIn (re-check-in)
        return fromStatus == CheckInOutStatus.None || fromStatus == CheckInOutStatus.CheckIn;
    }

    // Factory method
    public static CheckedInCheckInOut Create(DateTime checkInTime, Guid visitId)
    {
        return new CheckedInCheckInOut(checkInTime, visitId);
    }
}

public class CheckedOutCheckInOut : CheckInOut
{
    public CheckedOutCheckInOut(DateTime checkOutTime, Guid visitId)
        : base(CheckInOutStatus.CheckOut)
    {
        Time = checkOutTime.ToUniversalTime();
        VisitId = visitId;
    }

    public override bool IsActive => false;
    public override bool IsCompleted => true;

    public override bool CanTransitionFrom(CheckInOutStatus fromStatus)
    {
        // Can only transition from CheckIn or CheckOut (re-check-out)
        return fromStatus == CheckInOutStatus.CheckIn || fromStatus == CheckInOutStatus.CheckOut;
    }

    // Factory method
    public static CheckedOutCheckInOut Create(DateTime checkOutTime, Guid visitId)
    {
        return new CheckedOutCheckInOut(checkOutTime, visitId);
    }
}
```

---

### Step 5: Use CheckInOut in Domain Entity

```csharp
public class Visit
{
    public VisitId Id { get; }
    public GuestId GuestId { get; }
    public VisitPurpose Purpose { get; }
    public CheckInOut CheckInOut { get; private set; }

    public Visit(VisitId id, GuestId guestId, VisitPurpose purpose)
    {
        Id = id;
        GuestId = guestId;
        Purpose = purpose;
        CheckInOut = CheckInOut.None;
    }

    public Result<CheckInOut> CheckIn(DateTime checkInTime)
    {
        // Validate time is not too far in future
        if (checkInTime > DateTime.UtcNow.AddMinutes(5))
            return Result.Failure<CheckInOut>("Check-in time cannot be more than 5 minutes in the future");

        if (checkInTime < DateTime.UtcNow.AddSeconds(-5))
            return Result.Failure<CheckInOut>("Check-in time cannot be in the past");

        // Validate transition
        if (!CheckInOut.CanTransitionFrom(CheckInOut.Value))
            return Result.Failure<CheckInOut>($"Invalid transition from {CheckInOut.Value}");

        // Create checked-in state
        var checkedIn = CheckedInCheckInOut.Create(checkInTime.ToUniversalTime(), Id.Value);
        CheckInOut = checkedIn;

        return Result.Success(CheckInOut);
    }

    public Result<CheckInOut> CheckOut(DateTime checkOutTime)
    {
        // Validate check-out is after check-in
        if (!CheckInOut.IsActive)
            return Result.Failure<CheckInOut>("Guest must be checked in first");

        // Validate time is not before check-in
        if (checkOutTime < CheckInOut.Time)
            return Result.Failure<CheckInOut>("Check-out time cannot be before check-in time");

        // Validate check-out is not too far in future
        if (checkOutTime > DateTime.UtcNow.AddMinutes(5))
            return Result.Failure<CheckInOut>("Check-out time cannot be more than 5 minutes in the future");

        // Validate transition
        if (!CheckInOut.CanTransitionFrom(CheckInOut.Value))
            return Result.Failure<CheckInOut>($"Invalid transition from {CheckInOut.Value}");

        // Create checked-out state
        var checkedOut = CheckedOutCheckInOut.Create(checkOutTime.ToUniversalTime(), Id.Value);
        CheckInOut = checkedOut;

        return Result.Success(CheckInOut);
    }
}
```

---

## Complete Implementation

### With Full Code Link

**Base Class:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.SmartEnums/blob/main/TestNest.SmartEnums.Domain/ValueObjects/Common/SmartEnum.cs)

**CheckInOut Implementation:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.SmartEnums/blob/main/TestNest.SmartEnums.Domain/ValueObjects/CheckInOut.cs)

---

## Testing Your SmartEnum

```csharp
[Test]
public void CreateCheckIn_ValidTime_ReturnsSuccess()
{
    var visitId = VisitId.NewGuid();
    var checkInTime = DateTime.UtcNow;

    var result = new Visit(visitId, GuestId.NewGuid(), VisitPurpose.Create("Purpose").Value)
        .CheckIn(checkInTime);

    Assert.IsTrue(result.IsSuccess);
    Assert.IsTrue(result.Value.IsActive);
}

[Test]
public void CreateCheckIn_FutureTime_ReturnsFailure()
{
    var visitId = VisitId.NewGuid();
    var checkInTime = DateTime.UtcNow.AddMinutes(10);

    var result = new Visit(visitId, GuestId.NewGuid(), VisitPurpose.Create("Purpose").Value)
        .CheckIn(checkInTime);

    Assert.IsTrue(result.IsFailure);
    Assert.AreEqual("Check-in time cannot be more than 5 minutes in the future", result.Error);
}

[Test]
public void CreateCheckOut_BeforeCheckIn_ReturnsFailure()
{
    var visitId = VisitId.NewGuid();
    var checkOutTime = DateTime.UtcNow.AddHours(-1);

    var result = new Visit(visitId, GuestId.NewGuid(), VisitPurpose.Create("Purpose").Value)
        .CheckOut(checkOutTime);

    Assert.IsTrue(result.IsFailure);
    Assert.AreEqual("Guest must be checked in first", result.Error);
}

[Test]
public void Duration_CalculatesCorrectly()
{
    var checkInTime = DateTime.UtcNow.AddHours(-2);
    var checkOutTime = DateTime.UtcNow.AddHours(-1);

    var checkedIn = CheckedInCheckInOut.Create(checkInTime.ToUniversalTime(), Guid.NewGuid());
    var checkedOut = CheckedOutCheckInOut.Create(checkOutTime.ToUniversalTime(), Guid.NewGuid());

    var duration = checkedOut.GetDuration();

    Assert.IsNotNull(duration);
    Assert.AreEqual(TimeSpan.FromHours(1), duration);
}
```

---

## Common Pitfalls

### 1. Mutable State Properties

```csharp
// WRONG: Mutable time
public abstract class CheckInOut : SmartEnum<CheckInOutStatus, CheckInOut>
{
    public DateTime? Time { get; set; }  // Should be protected set
}
```

### 2. Missing Transition Validation

```csharp
// WRONG: No validation
public override bool CanTransitionFrom(CheckInOutStatus fromStatus)
{
    return true;  // Allows any transition
}
```

### 3. Incorrect Factory Methods

```csharp
// WRONG: Wrong state type
public static CheckedInCheckInOut Create(DateTime time)
{
    return new CheckedOutCheckInOut(time);  // Returns wrong state!
}
```

### 4. Not Using Factory Methods

```csharp
// WRONG: Creating state directly
var checkIn = new CheckedInCheckInOut(time, visitId);  // Bypasses validation
```

---

## Best Practices

1. **Always use factory methods** (CreateCheckIn, CreateCheckOut)
2. **Make state properties protected set** or read-only
3. **Implement transition validation** in CanTransitionFrom
4. **Use UTC times** to avoid timezone issues
5. **Enforce temporal constraints** in factory methods
6. **Provide meaningful error messages**
7. **Test all transition paths**
8. **Write comprehensive unit tests**
9. **Document state transitions** with diagrams
10. **Keep state small and focused**

---

## State Transition Diagram

```
    ┌─────────┐
    │   None   │
    └────┬────┘
         │
         │ Valid
         ▼
    ┌─────────┐
    │ Check-In │◄────┐
    └────┬────┘     │
         │ Valid      │ Optional
         ▼           │ (re-check-in)
    ┌─────────┐      │
    │Check-Out │◀─────┘
    └─────────┘
```

**Valid Transitions:**
- None → CheckIn
- CheckIn → CheckOut
- CheckIn → CheckIn (re-check-in)
- CheckOut → CheckOut (re-check-out)

**Invalid Transitions:**
- None → CheckOut (can't check out without checking in)
- CheckIn → None (can't undo)
- CheckOut → None (can't undo)

---

## Next Steps

You've implemented your first SmartEnum with state transitions! In [Chapter 4](/2025/01/11/smart-enums-chapter-04.html), we'll explore advanced concepts like temporal validation, business rules, and state machine performance.

---

## Key Takeaways

- Implement SmartEnum base class for type-safe states
- Create factory methods for state transitions
- Validate transitions with CanTransitionFrom
- Encapsulate state-specific behavior
- Test all transition paths thoroughly
- Next: Advanced temporal validation in Chapter 4

---

## References

- [Complete Source Code: TestNest.SmartEnums](https://github.com/DanteTuraSalvador/TestNest.SmartEnums)
- [Tests: CheckInOutTests.cs](https://github.com/DanteTuraSalvador/TestNest.SmartEnums/blob/main/TestNest.SmartEnums.Tests/CheckInOutTests.cs)

{% include tutorial-nav.html %}
