---
layout: post
title: "Chapter 1: Problem - Primitive Obsession"
date: 2025-01-01
tags:
  - ddd
  - value-objects
  - beginner
series: value-objects
chapter: 1
prerequisites: None
estimated_time: "15 minutes"
prev_title: "Back to Homepage"
prev: "/"
next_title: "Chapter 2: Solution - Value Object Pattern"
next: "/2025/01/02/value-objects-chapter-02.html"
---

# Chapter 1: Problem - Primitive Obsession

## Learning Objectives

By the end of this chapter, you will understand:
- What is primitive obsession
- Why it's a problem in domain modeling
- Real-world examples of primitive obsession
- The impact on code maintainability

---

## What is Primitive Obsession?

Primitive obsession is a code smell where basic primitive types (string, int, decimal) are used to represent domain concepts instead of creating dedicated types.

### Example: The Wrong Way

```csharp
public class Customer
{
    public string Email { get; set; }
    public string PhoneNumber { get; set; }
    public decimal Balance { get; set; }
}
```

**Problems:**

1. **No validation** - Invalid emails can be set
2. **Scattered validation** - Validation logic spread everywhere
3. **Type confusion** - Email could be used as phone number
4. **No self-documentation** - What format is expected?
5. **Reference equality bugs** - Two emails with same value are different objects

### Real-World Impact

```csharp
// Bug: Mixing up email and phone
var customer = new Customer
{
    Email = "555-1234",              // Oops! Phone in email field
    PhoneNumber = "user@example.com" // Oops! Email in phone field
};

// No compiler error, no runtime error - silent data corruption
```

---

## Why This Matters

**Imagine maintaining a codebase with:**
- 100+ places validating email format
- 50+ places checking phone number format
- Inconsistent validation rules
- Bugs from parameter confusion
- Hard to read and modify

---

## The Cost of Primitive Obsession

| Problem | Impact |
|---------|--------|
| No type safety | Runtime bugs |
| Scattered validation | Duplicated code |
| Parameter confusion | Wrong data in wrong fields |
| No encapsulation | Business logic leaks everywhere |
| Hard to test | Complex test setup |

---

## What We Need

We need a way to:
1. **Validate** data at creation
2. **Encapsulate** validation rules
3. **Prevent** parameter confusion
4. **Self-document** our domain
5. **Ensure** value semantics

---

## The Solution: Value Objects

In [Chapter 2](/2025/01/02/value-objects-chapter-02.html), we'll explore the **Value Object pattern** and how it solves all these problems.

---

## Key Takeaways

- Primitive obsession uses basic types for domain concepts
- It causes validation, type safety, and maintainability issues
- The solution is Value Object pattern
- Next: Learn about Value Objects in Chapter 2

---

## References

- [Source Code: TestNest.ValueObjects](https://github.com/DanteTuraSalvador/TestNest.ValueObjects)
- [Wikipedia: Primitive Obsession](https://en.wikipedia.org/wiki/Primitive_obsession)

{% include tutorial-nav.html %}
