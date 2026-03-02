---
layout: post
title: "Chapter 3: Implementation - Your First Value Object"
date: 2025-01-03
tags:
  - ddd
  - value-objects
  - implementation
series: value-objects
chapter: 3
prerequisites: "Chapter 2"
estimated_time: "30 minutes"
prev_title: "Chapter 2: Solution - Value Object Pattern"
prev_url: "/2025/01/02/value-objects-chapter-02.html"
next_title: "Chapter 4: Advanced - Composing Value Objects"
next_url: "/2025/01/04/value-objects-chapter-04.html"
---

# Chapter 3: Implementation - Your First Value Object

## Learning Objectives

By the end of this chapter, you will be able to:
- Create a Value Object base class
- Implement a concrete Value Object (Email)
- Use Value Objects in your domain
- Test Value Objects

---

## Implementation: Step by Step

### Step 1: Create Value Object Base Class

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object> GetAtomicValues();

    public override bool Equals(object obj)
    {
        if (obj == null || obj.GetType() != GetType())
            return false;

        var other = (ValueObject)obj;
        return GetAtomicValues().SequenceEqual(other.GetAtomicValues());
    }

    public override int GetHashCode()
    {
        return GetAtomicValues()
            .Select(x => x?.GetHashCode() ?? 0)
            .Aggregate((x, y) => x ^ y);
    }
}
```

**What this does:**
- Provides value-based equality
- Compares all atomic values
- Implements GetHashCode correctly

---

### Step 2: Implement Email Value Object

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
        try
        {
            var addr = new MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return Value;
    }
}
```

---

### Step 3: Use Email in Domain

```csharp
public class Customer
{
    public CustomerId Id { get; }
    public Email Email { get; private set; }

    public Result ChangeEmail(string newEmail)
    {
        var emailResult = Email.Create(newEmail);

        if (emailResult.IsFailure)
            return Result.Failure(emailResult.Error);

        Email = emailResult.Value;
        return Result.Success();
    }
}
```

---

## Complete Implementation

### With Full Code Link

**Base Class:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Domain/ValueObjects/Common/ValueObject.cs)

**Email Implementation:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Domain/ValueObjects/Email.cs)

---

## Testing Your Value Object

```csharp
[Test]
public void Create_ValidEmail_ReturnsSuccess()
{
    var result = Email.Create("test@example.com");

    Assert.IsTrue(result.IsSuccess);
    Assert.AreEqual("test@example.com", result.Value.Value);
}

[Test]
public void Create_InvalidEmail_ReturnsFailure()
{
    var result = Email.Create("invalid");

    Assert.IsTrue(result.IsFailure);
    Assert.AreEqual("Invalid email format", result.Error);
}

[Test]
public void EqualEmails_AreEqual()
{
    var email1 = Email.Create("test@example.com").Value;
    var email2 = Email.Create("test@example.com").Value;

    Assert.AreEqual(email1, email2);
}
```

---

## Common Pitfalls

### 1. Mutable Value Objects

```csharp
// WRONG: Mutable
public class Email : ValueObject
{
    public string Value { get; set; }  // Should be { get; }
}
```

### 2. Incomplete Equality

```csharp
// WRONG: Missing atomic values
protected override IEnumerable<object> GetAtomicValues()
{
    yield return Value;
    // Missing other properties!
}
```

### 3. Exposing Constructor Directly

```csharp
// WRONG: Bypasses validation
var email = new Email("invalid-email");  // Should use Create()
```

---

## Best Practices

1. **Always use factory methods** (Create, Parse, TryParse)
2. **Make properties private set** or read-only
3. **Implement GetAtomicValues completely**
4. **Normalize values** (e.g., lowercase emails)
5. **Provide meaningful error messages**
6. **Write comprehensive tests**

---

## Next Steps

You've implemented your first Value Object! In [Chapter 4](/2025/01/04/value-objects-chapter-04.html), we'll explore advanced concepts like composing Value Objects and using Money/Price.

---

## Key Takeaways

- Implement base ValueObject class for equality
- Create factory methods with validation
- Use Value Objects in domain entities
- Test thoroughly
- Next: Advanced Value Object composition

---

## References

- [Complete Source Code: TestNest.ValueObjects](https://github.com/DanteTuraSalvador/TestNest.ValueObjects)
- [Tests: EmailTests.cs](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Tests/EmailTests.cs)
