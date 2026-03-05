---
layout: post
title: "Chapter 2: Solution - StronglyTypedId Pattern"
date: 2025-01-06
tags:
  - ddd
  - strongly-typed-ids
  - beginner
series: strongly-typed-ids
chapter: 2
prerequisites: "Chapter 1"
estimated_time: "20 minutes"
prev_title: "Chapter 1: Problem - ID Type Confusion"
prev_url: "/2025/01/05/strongly-typed-ids-chapter-01.html"
next_title: "Chapter 3: Integration with Value Objects"
next_url: "/2025/01/07/strongly-typed-ids-chapter-03.html"
---

# Chapter 2: Solution - StronglyTypedId Pattern

## Learning Objectives

By the end of this chapter, you will understand:
- What is a StronglyTypedId
- Core principles of StronglyTypedId
- When to use StronglyTypedId
- Benefits of StronglyTypedId

---

## What is a StronglyTypedId?

A **StronglyTypedId** is a wrapper around a primitive ID type (Guid, int, string) that provides type safety for domain identifiers. Each domain entity has its own dedicated ID type that cannot be confused with other ID types.

### Key Principles

1. **Type safety** - Each entity has its own ID type
2. **Value semantics** - IDs compared by value, not reference
3. **Immutability** - Once created, cannot be modified
4. **Primitive compatibility** - Works with existing systems (databases, APIs)
5. **Factory methods** - Controlled creation (New, Empty, Create, Parse)

---

## StronglyTypedId vs Primitive IDs

| Aspect | StronglyTypedId | Primitive ID |
|--------|----------------|-------------|
| Type safety | Entity-specific types | Shared type (Guid, int) |
| Compiler enforcement | Prevents parameter swapping | No protection |
| Self-documentation | Clear entity type | Generic "id" |
| Value semantics | Value-based equality | Reference-based equality |
| Database compatible | Yes (implicit conversion) | Yes (native) |

---

## Example: StronglyTypedId Implementation

### Base Class

```csharp
public abstract class StronglyTypedId<TValue, TDerived>
    where TValue : notnull
    where TDerived : StronglyTypedId<TValue, TDerived>, new()
{
    public TValue Value { get; }

    protected StronglyTypedId(TValue value)
    {
        Value = value;
    }

    protected StronglyTypedId()
    {
        Value = default(TValue)!;
    }

    public static TDerived New()
    {
        return new TDerived();
    }

    public static TDerived New(TValue value)
    {
        return new TDerived(value);
    }

    public static TDerived Empty => new TDerived();

    public static TDerived Create(TValue value)
    {
        if (value == null || EqualityComparer<TValue>.Default.Equals(value, default!))
            throw new StronglyTypedIdException("Value cannot be null or empty");

        return new TDerived(value);
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

    public static implicit operator TValue(StronglyTypedId<TValue, TDerived> id)
    {
        return id.Value!;
    }

    public static bool operator ==(StronglyTypedId<TValue, TDerived> left, StronglyTypedId<TValue, TDerived> right)
    {
        if (left is null || right is null)
            return false;

        return left.Value!.Equals(right.Value!);
    }

    public static bool operator !=(StronglyTypedId<TValue, TDerived> left, StronglyTypedId<TValue, TDerived> right)
    {
        return !(left == right);
    }

    public override string ToString() => Value!.ToString()!;
}
```

### Concrete Implementations

```csharp
public class CustomerId : StronglyTypedId<Guid, CustomerId>
{
    public CustomerId(Guid value) : base(value) { }
    public CustomerId() : base() { }
}

public class OrderId : StronglyTypedId<Guid, OrderId>
{
    public OrderId(Guid value) : base(value) { }
    public OrderId() : base() { }
}

public class GuestId : StronglyTypedId<Guid, GuestId>
{
    public GuestId(Guid value) : base(value) { }
    public GuestId() : base() { }
}

public class ProductId : StronglyTypedId<Guid, ProductId>
{
    public ProductId(Guid value) : base(value) { }
    public ProductId() : base() { }
}

public class VisitId : StronglyTypedId<Guid, VisitId>
{
    public VisitId(Guid value) : base(value) { }
    public VisitId() : base() { }
}
```

---

## Benefits of StronglyTypedId

### 1. Type Safety

```csharp
// Compiler prevents mixing types
var customerId = CustomerId.New();
var orderId = OrderId.New();

void ProcessOrder(OrderId id)
{
    // Order must use OrderId
}

// Cannot accidentally pass Customer where OrderId expected
// ProcessOrder(customerId); // Compiler error!
```

### 2. Self-Documentation

```csharp
// Clear intent
public class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public ProductId ProductId { get; }

    public void AssignCustomer(CustomerId newCustomerId)
    {
        // Compiler ensures it's a CustomerId
        CustomerId = newCustomerId;
    }

    // No parameter confusion possible
    // AssignCustomer(orderId); // Compiler error!
}
```

### 3. Value Semantics

```csharp
var id1 = CustomerId.NewGuid();
var id2 = CustomerId.Create(id1.Value);

// They are equal (same values)
Console.WriteLine(id1 == id2);  // True

// Different types are never equal
var customerId = CustomerId.NewGuid();
var orderId = OrderId.NewGuid();
Console.WriteLine(customerId == orderId);  // False - different types
```

### 4. Database Compatibility

```csharp
// Implicit conversion to Guid for database
var customerId = CustomerId.NewGuid();

// Works with Entity Framework, Dapper, etc.
_context.Customers.Add(new Customer { Id = customerId });
_context.SaveChanges(); // Uses Guid value
```

### 5. Factory Methods

```csharp
// New - Creates new ID
var id = CustomerId.NewGuid(); // New Guid

// Empty - Singleton for empty/default
var empty = CustomerId.Empty;

// Create - Validates non-null
var id = CustomerId.Create(Guid.NewGuid()); // Validated

// Parse - From string
var id = CustomerId.Parse("d72b0e9-9c4e-4f3f-a8db-3c0c5d3e8f1");
```

---

## When to Use StronglyTypedId

**Use StronglyTypedId when:**
- You have domain entities with IDs
- Multiple entity types could be confused
- You need type safety for IDs
- You want compile-time error prevention
- You're working with existing database (Guid, int)

**Examples:**
- CustomerId, OrderId, ProductId
- GuestId, VisitId, UserId
- Any domain entity identifier

---

## When NOT to Use StronglyTypedId

**Don't use when:**
- Performance is critical and wrapper overhead matters
- You don't have multiple entity types
- The domain doesn't have identifiable entities
- Working with external systems only (no domain model)

---

## Real-World Integration

```csharp
public class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public List<OrderItem> Items { get; }

    public Order(OrderId id, CustomerId customerId, List<OrderItem> items)
    {
        Id = id;
        CustomerId = customerId;
        Items = items;
    }
}

public class OrderService
{
    public void CreateOrder(CustomerId customerId, List<ProductId> productIds)
    {
        // Type-safe operations
        var orderId = OrderId.NewGuid();
        var order = new Order(orderId, customerId, CreateItems(productIds));
        _repository.Save(order);
    }

    // No parameter confusion possible
    // CreateOrder(customerId, orderIds); // Compiler error!
}
```

---

## Key Takeaways

- StronglyTypedId provides type safety for domain identifiers
- They prevent parameter swapping at compile-time
- They're database compatible and self-documenting
- Use when you have multiple entity types with IDs
- Next: Integrate StronglyTypedId with Value Objects in Chapter 3

---

## References

- [Source Code: TestNest.StronglyTypedId](https://github.com/DanteTuraSalvador/TestNest.StronglyTypedId)
- [Microsoft: Domain-Driven Design](https://docs.microsoft.com/en-us/previous-versions/dotnet/articles/net-msdn-patterns-ddd)

---

## What's Next?

In [Chapter 3](/2025/01/07/strongly-typed-ids-chapter-03.html), you'll learn how to integrate StronglyTypedId with Value Objects for a completely type-safe domain model!

