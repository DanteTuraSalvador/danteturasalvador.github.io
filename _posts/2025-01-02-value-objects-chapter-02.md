---
layout: post
title: "Chapter 2: Solution - Value Object Pattern"
date: 2025-01-02
tags:
  - ddd
  - value-objects
  - beginner
series: value-objects
chapter: 2
prerequisites: "Chapter 1"
estimated_time: "20 minutes"
prev_title: "Chapter 1: Problem - Primitive Obsession"
prev: "/2025/01/01/value-objects-chapter-01.html"
next_title: "Chapter 3: Implementation - Your First Value Object"
next: "/2025/01/03/value-objects-chapter-03.html"
---

# Chapter 2: Solution - Value Object Pattern

## Learning Objectives

By the end of this chapter, you will understand:
- What is a Value Object
- Core principles of Value Objects
- When to use Value Objects
- Benefits of Value Objects

---

## What is a Value Object?

A **Value Object** is an object that represents a descriptive aspect of domain with **no conceptual identity**. It's defined by its **attributes**, not by a unique identifier.

### Key Principles

1. **Immutability** - Once created, cannot be modified
2. **Value-based equality** - Equal if all attributes are equal
3. **Self-validation** - Ensures valid state at creation
4. **No identity** - Not identified by ID, only by values

---

## Value Object vs Entity

| Aspect | Value Object | Entity |
|--------|--------------|--------|
| Identity | No (defined by values) | Yes (defined by ID) |
| Mutability | Immutable | Mutable |
| Equality | Value-based | Identity-based |
| Example | Email, Money, Address | Customer, Order, Product |

---

## Example: Email Value Object

### Implementation

```csharp
public class Email : ValueObject
{
    public string Value { get; }

    private Email(string value)
    {
        Value = value.ToLowerInvariant();
    }

    public static Result<Email> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return Result.Failure<Email>("Email cannot be empty");

        if (!IsValidFormat(email))
            return Result.Failure<Email>("Invalid email format");

        return Result.Success(new Email(email));
    }

    private static bool IsValidFormat(string email)
    {
        // Email validation logic
        return Regex.IsMatch(email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
    }
}
```

### Usage

```csharp
// Create email with validation
var result = Email.Create("user@example.com");

if (result.IsSuccess)
{
    var email = result.Value;
    Console.WriteLine($"Email: {email.Value}");
}
else
{
    Console.WriteLine($"Error: {result.Error}");
}
```

---

## Benefits of Value Objects

### 1. Type Safety

```csharp
// Compiler prevents mixing types
var email = Email.Create("user@example.com");
var phone = PhoneNumber.Create("+1-555-1234");

// Cannot accidentally pass phone where email is expected
void ProcessEmail(Email email) { }
// ProcessEmail(phone); // Compiler error!
```

### 2. Self-Validation

```csharp
// Validation happens at creation
var email = Email.Create("invalid-email");
// Result is Failure, no invalid email can exist
```

### 3. Self-Documenting

```csharp
// Clear intent
public class Customer
{
    public Email Email { get; }         // Clear: it's an email
    public PhoneNumber Phone { get; }   // Clear: it's a phone
    public Money Balance { get; }      // Clear: it's money
}
```

### 4. Value Semantics

```csharp
var email1 = Email.Create("test@example.com").Value;
var email2 = Email.Create("test@example.com").Value;

// They are equal (same values)
Console.WriteLine(email1 == email2);  // True
```

---

## When to Use Value Objects

**Use Value Objects when:**
- The object has no identity
- Equality is based on attributes
- The object is small and simple
- You need type safety
- You need self-validation

**Examples:**
- Email, PhoneNumber, Address
- Money, Currency, Price
- DateRange, TimeRange
- Color, Size, Weight

---

## When NOT to Use Value Objects

**Don't use when:**
- The object needs identity
- The object is large or complex
- Performance is critical and immutability is expensive
- You need to track lifecycle changes

---

## Real-World Integration

```csharp
public class Customer
{
    public CustomerId Id { get; }
    public Email Email { get; }
    public PhoneNumber Phone { get; }
    public Address ShippingAddress { get; }
    public Money Balance { get; }
}

// Type-safe operations
public void UpdateEmail(Email newEmail)
{
    // Compiler ensures it's an Email
    Email = newEmail;
}

// No parameter confusion possible
// customer.UpdateEmail(phoneNumber); // Compiler error!
```

---

## Key Takeaways

- Value Objects are immutable, value-based objects
- They provide type safety and self-validation
- Use when identity doesn't matter, only values do
- Next: Implement your first Value Object in Chapter 3

---

## References

- [Source Code: TestNest.ValueObjects](https://github.com/DanteTuraSalvador/TestNest.ValueObjects)
- [Eric Evans: Domain-Driven Design](https://domainlanguage.com/ddd/)

{% include tutorial-nav.html %}
