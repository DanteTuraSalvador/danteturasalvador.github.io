---
layout: post
title: "Chapter 1: Problem - ID Type Confusion"
date: 2025-01-05
tags:
  - ddd
  - strongly-typed-ids
  - beginner
series: strongly-typed-ids
chapter: 1
prerequisites: None
estimated_time: "15 minutes"
prev_title: "Back to Value Objects Series"
prev: "/2025/01/04/value-objects-chapter-04.html"
next_title: "Chapter 2: Solution - StronglyTypedId Pattern"
next: "/2025/01/06/strongly-typed-ids-chapter-02.html"
---

# Chapter 1: Problem - ID Type Confusion

## Learning Objectives

By the end of this chapter, you will understand:
- What is ID type confusion
- Why it's a problem in domain modeling
- Real-world examples of type confusion
- The impact on code safety

---

## What is ID Type Confusion?

ID type confusion occurs when domain identifiers are represented using primitive types (int, Guid, string), leading to:
- Compiler doesn't catch parameter mixing
- IDs can be accidentally swapped
- No type safety for domain identifiers
- Bugs that compile but fail at runtime

### Example: The Wrong Way

```csharp
public class Customer
{
    public Guid Id { get; set; }
}

public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
}

// Oops! Wrong ID passed
var order = new Order
{
    Id = Guid.NewGuid(),
    CustomerId = orderId // Bug: Order ID passed as Customer ID
};

// No compiler error, no runtime error - silent data corruption
```

---

## Why This Matters

**Imagine a codebase with:**
- 100+ Guid IDs scattered everywhere
- Parameter confusion between different entity IDs
- No compile-time safety for ID matching
- Bugs from mixing up IDs
- Hard to understand what each ID represents

---

## The Cost of ID Type Confusion

| Problem | Impact |
|---------|--------|
| No type safety | Runtime bugs |
| Parameter swapping | Wrong data in wrong fields |
| No self-documentation | What ID type is expected? |
| Mix-up errors | Silent data corruption |
| Hard to refactor | Changes ripple through codebase |

---

## Real-World Impact

### Example 1: Repository Method

```csharp
public interface ICustomerRepository
{
    Customer GetById(Guid id);
    Order GetOrderById(Guid id);
    Product GetProductById(Guid id);
}

// Can accidentally call:
var order = _customerRepository.GetById(orderId);
// Compiles! But Customer ID expected
```

### Example 2: Domain Services

```csharp
public class OrderService
{
    public void ProcessOrder(Guid orderId, Guid customerId, Guid productId)
    {
        // What if these get swapped?
        Process(orderId, productId, customerId); // Oops!
    }
}
```

---

## What We Need

We need a way to:
1. **Type-safe IDs** that can't be confused
2. **Self-documentation** - Clear what entity type
3. **Compiler enforcement** - Prevent parameter swapping
4. **Value semantics** - IDs compared by value, not reference
5. **Compatibility** - Work with existing systems (Guid, int)

---

## The Solution: Strongly Typed IDs

In [Chapter 2](/2025/01/06/strongly-typed-ids-chapter-02.html), we'll explore the **StronglyTypedId pattern** and how it solves all these problems.

---

## Key Takeaways

- ID type confusion uses primitive types for domain identifiers
- It causes type safety, parameter swapping, and maintainability issues
- The solution is StronglyTypedId pattern
- Next: Learn about StronglyTypedId in Chapter 2

---

## References

- [Source Code: TestNest.StronglyTypedId](https://github.com/DanteTuraSalvador/TestNest.StronglyTypeId)
- [Martin Fowler: Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/identityField.html)

{% include tutorial-nav.html %}
