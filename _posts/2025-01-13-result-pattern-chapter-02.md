---
layout: post
title: "Chapter 2: Solution - Result Pattern"
date: 2025-01-13
tags:
  - ddd
  - result-pattern
  - beginner
series: result-pattern
chapter: 2
prerequisites: "Chapter 1"
estimated_time: "20 minutes"
prev_title: "Chapter 1: Problem - Exception-Driven Error Handling"
prev: "/2025/01/12/result-pattern-chapter-01.html"
next_title: "Chapter 3: Implementation - Result and Result<T>"
next: "/2025/01/14/result-pattern-chapter-03.html"
---

# Chapter 2: Solution - Result Pattern

## Learning Objectives

By the end of this chapter, you will understand:
- What is the Result pattern
- Core principles of Result pattern
- When to use Result pattern
- Benefits of Result over exceptions

---

## What is the Result Pattern?

The **Result pattern** is a functional programming pattern that:
- **Wraps** operation outcomes in a Result type
- **Distinguishes** success from failure
- **Carries error information** without exceptions
- **Supports chaining** operations gracefully
- **Enables** functional-style error handling
- **Avoids** control flow via exceptions
- **Supports** error aggregation

### Key Principles

1. **Explicit success/failure** - No implicit exceptions
2. **Error information** - Rich error details
3. **Value propagation** - Pass data through successful results
4. **Error propagation** - Pass errors through failures
5. **Functional composition** - Chain operations cleanly
6. **Type safety** - Compile-time error handling
7. **Testability** - Easy to test error paths
8. **No stack pollution** - Keeps stack traces clean

---

## Result Pattern vs Exceptions

| Aspect | Result Pattern | Exceptions |
|--------|----------------|------------|
| Success/Failure | Explicit | Implicit (try/catch) |
| Error information | Rich error types | Message in exception |
| Error aggregation | Supported easily | Requires manual effort |
| Composition | Built-in (Bind/Map) | Manual propagation |
| Type safety | Compile-time enforced | Runtime checking |
| Control flow | Functional style | Disrupted by exceptions |
| Performance | Minimal overhead | Significant overhead |
| Stack pollution | None | Significant pollution |
| Testability | Easy to test | Complex setup |

---

## Example: Result Types

### Non-Generic Result (No Return Value)

```csharp
public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error? Error { get; }

    private Result()
    {
        IsSuccess = true;
        Error = null;
    }

    private Result(Error error)
    {
        IsSuccess = false;
        Error = error;
    }

    // Factory methods
    public static Result Success()
    {
        return new Result();
    }

    public static Result Failure(string message)
    {
        return new Result(new Error(message));
    }

    public static Result Failure(Error error)
    {
        return new Result(error);
    }
}
```

### Generic Result<T> (With Return Value)

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T? Value { get; }
    public Error? Error { get; }

    private Result(T value)
    {
        IsSuccess = true;
        Value = value;
        Error = null;
    }

    private Result(Error error)
    {
        IsSuccess = false;
        Value = default;
        Error = error;
    }

    // Factory methods
    public static Result<T> Success(T value)
    {
        return new Result<T>(value);
    }

    public static Result<T> Failure(string message)
    {
        return new Result<T>(new Error(message));
    }

    public static Result<T> Failure(Error error)
    {
        return new Result<T>(error);
    }

    // Implicit conversion from T to Result<T>
    public static implicit operator Result<T>(T value)
    {
        return Success(value);
    }
}
```

### Error Type

```csharp
public record Error
{
    public string Message { get; init; }
    public ErrorType Type { get; init; }
    public string? Details { get; init; }

    public static Error Create(string message)
    {
        return new Error
        {
            Message = message,
            Type = ErrorType.Validation
        };
    }

    public static Error Create(string message, ErrorType type)
    {
        return new Error
        {
            Message = message,
            Type = type
        };
    }
}

public enum ErrorType
{
    Validation,
    NotFound,
    Unauthorized,
    Conflict,
    Internal,
    Aggregate,
    Invalid
}
```

---

## Benefits of Result Pattern

### 1. Explicit Success/Failure

```csharp
// Clear intention
var result = Email.Create("user@example.com");

if (result.IsSuccess)
{
    Console.WriteLine($"Email: {result.Value}");
}
else
{
    Console.WriteLine($"Error: {result.Error.Message}");
}
```

### 2. Rich Error Information

```csharp
public class Error
{
    public string Message { get; }
    public ErrorType Type { get; }
    public string? Field { get; }
    public object? Metadata { get; }
}

// Rich error details
var error = Error.Create("Invalid email", ErrorType.Validation)
{
    Field = "Email",
    Metadata = new { AllowedDomains = new[] { "example.com", "test.com" }
};

var result = Result<Email>.Failure(error);
```

### 3. Error Aggregation

```csharp
// Combine multiple errors
public static Result<T> Combine<T>(params Result<T>[] results)
{
    var errors = results.Where(r => r.IsFailure)
                              .Select(r => r.Error!)
                              .ToList();

    if (errors.Any())
        return Result<T>.Failure(Error.CreateAggregate(errors));

    return results.First();
}

// Usage
var emailResult = Email.Create("user@example.com");
var phoneResult = PhoneNumber.Create("invalid-phone");
var combined = Result.Combine(emailResult, phoneResult);

if (combined.IsFailure)
{
    foreach (var error in combined.Error!.Errors)
    {
        Console.WriteLine($"Error: {error.Message}");
    }
}
```

---

## When to Use Result Pattern

**Use Result when:**
- Operations can succeed or fail
- Validation is needed
- Multiple validation errors may occur
- You need rich error information
- You want to chain operations
- Performance matters (no exception overhead)
- Clear separation between success/failure paths

**Examples:**
- Domain operations (create, update, delete)
- Validation logic
- Repository operations
- Service layer operations
- API responses

---

## When NOT to Use Result

**Don't use when:**
- Operation should always succeed
- Only one type of error possible (use exceptions instead)
- Performance is critical (Result has minimal but non-zero overhead)
- Framework already provides similar pattern (e.g., Task<T>)

**Use exceptions when:**
- Truly unexpected errors (system failures)
- Framework-level errors (I/O, network)
- Errors that should crash the application
- Asynchronous operations where Task<T> is more appropriate

---

## Real-World Example: Customer Service with Result

```csharp
public class CustomerService
{
    private readonly ICustomerRepository _repository;

    public Result<Customer> CreateCustomer(string email, string phone)
    {
        // Validate email
        var emailResult = Email.Create(email);
        if (emailResult.IsFailure)
            return Result<Customer>.Failure(emailResult.Error);

        // Validate phone
        var phoneResult = PhoneNumber.Create(phone);
        if (phoneResult.IsFailure)
            return Result<Customer>.Failure(phoneResult.Error);

        // Check for existing customer
        var existing = _repository.FindByEmail(email);
        if (existing != null)
            return Result<Customer>.Failure(Error.Create(
                "Customer with this email already exists",
                ErrorType.Conflict));

        // Create customer
        var customer = new Customer(email, phone);
        var saved = _repository.Add(customer);

        return Result<Customer>.Success(saved);
    }

    public Result<Customer> UpdateCustomer(CustomerId id, string newEmail)
    {
        // Validate new email
        var emailResult = Email.Create(newEmail);
        if (emailResult.IsFailure)
            return Result<Customer>.Failure(emailResult.Error);

        // Find customer
        var customer = _repository.GetById(id);
        if (customer == null)
            return Result<Customer>.Failure(Error.Create(
                "Customer not found",
                ErrorType.NotFound));

        // Check for duplicate email
        var existing = _repository.FindByEmail(newEmail);
        if (existing != null && existing.Id != id)
            return Result<Customer>.Failure(Error.Create(
                "Email already in use",
                ErrorType.Conflict));

        // Update customer
        var updated = customer with { Email = emailResult.Value };
        _repository.Update(updated);

        return Result<Customer>.Success(updated);
    }
}
```

---

## Key Takeaways

- Result pattern provides explicit success/failure handling
- Rich error information improves debugging
- Error aggregation is built-in
- Functional composition is supported
- Use when operations can succeed or fail
- Next: Implement Result and Result<T> in Chapter 3

---

## References

- [Source Code: TestNest.ResultPattern](https://github.com/DanteTuraSalvador/TestNest.ResultPattern)
- [Language Ext: Result Type](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types)

{% include tutorial-nav.html %}
