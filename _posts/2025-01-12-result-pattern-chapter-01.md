---
layout: post
title: "Part 1: Problem - Exception-Driven Error Handling"
date: 2025-01-12
category: ddd
thumbnail-img: "https://images.unsplash.com/photo-1504639725590-34d0984388bd?w=400&h=200&fit=crop"
tags:
  - ddd
  - result-pattern
  - beginner
series: result-pattern
chapter: 1
prerequisites: None
estimated_time: "15 minutes"
prev_title: "Back to Smart Enums Series"
prev_url: "/ddd/2025/01/11/smart-enums-chapter-04.html"
next_title: "Part 2: Solution - Result Pattern"
next_url: "/ddd/2025/01/13/result-pattern-chapter-02.html"
---

# Part 1: Problem - Exception-Driven Error Handling

## Learning Objectives

By the end of this chapter, you will understand:
- What is exception-driven error handling
- Why exceptions are problematic for domain operations
- Real-world examples of error handling issues
- The impact on code quality and maintainability

---

## What is Exception-Driven Error Handling?

Exception-driven error handling occurs when:
- Exceptions are used for control flow
- Business logic failures throw exceptions
- Validation errors throw exceptions
- Expected failures throw exceptions
- Error messages lost in exception stacks
- Exceptions cross abstraction boundaries inappropriately

### Example: The Wrong Way

```csharp
public class CustomerService
{
    public Customer CreateCustomer(string email, string phone)
    {
        // Throws exceptions for validation
        if (string.IsNullOrWhiteSpace(email))
            throw new ArgumentException("Email cannot be empty");

        if (string.IsNullOrWhiteSpace(phone))
            throw new ArgumentException("Phone cannot be empty");

        // Throws exception for business rule
        var emailResult = Email.Create(email);
        if (emailResult.IsFailure)
            throw new InvalidOperationException(emailResult.Error);

        return new Customer(email, phone);
    }

    public Order CreateOrder(Guid customerId, List<Guid> productIds)
    {
        // Throws exception if customer not found
        var customer = _repository.GetCustomer(customerId)
            ?? throw new NotFoundException("Customer not found");

        return new Order(customer, productIds);
    }
}
```

**Problems:**

1. **Control flow via exceptions** - Exceptions used for validation instead of control flow
2. **Business logic as exceptions** - Expected failures throw unexpected exceptions
3. **Error information loss** - Exception type doesn't describe error
4. **Stack trace pollution** - Exception stacks become long and confusing
5. **Cross-boundary issues** - Domain exceptions leak to application layer
6. **Hard to test** - Need to test for specific exceptions
7. **Performance overhead** - Exception throwing and catching is expensive
8. **No error aggregation** - Can't collect multiple errors at once

---

## Why This Matters

**Imagine a codebase with:**
- 100+ exception types for different scenarios
- Validation throws exceptions everywhere
- Business rules throw exceptions
- Error messages hidden in exception types
- Complex exception handling logic
- Difficult to distinguish between expected and unexpected errors

---

## The Cost of Exception-Driven Error Handling

| Problem | Impact |
|---------|--------|
| Control flow violations | Confusing code flow |
| Business logic in exceptions | Inappropriate exception usage |
| Error information loss | Hard to debug errors |
| Stack trace pollution | Performance and memory issues |
| Cross-boundary leaks | Architecture violations |
| Hard to test | Complex test setup |
| Performance overhead | Slower application |
| No error aggregation | Can't combine errors |
| Inconsistent error handling | Different patterns everywhere |

---

## Real-World Impact

### Example 1: Registration Flow

```csharp
try
{
    var customer = _customerService.CreateCustomer(email, phone);
    var account = _accountService.CreateAccount(customer);

    try
    {
        var order = _orderService.CreateOrder(customer.Id, products);
        var payment = _paymentService.ProcessPayment(order);

        _emailService.SendConfirmation(customer.Email);
    }
    catch (ValidationException ex)
    {
        _logger.LogError(ex);
        return RedirectToAction("ValidationError");
    }
    catch (BusinessRuleException ex)
    {
        _logger.LogError(ex);
        return RedirectToAction("BusinessError");
    }
    catch (NotFoundException ex)
    {
        _logger.LogError(ex);
        return RedirectToAction("NotFound");
    }
    catch (Exception ex) // Catches everything!
    {
        _logger.LogError(ex);
        return RedirectToAction("Error");
    }
}
```

**Problems:**
- Multiple catch blocks
- Exceptions used for control flow
- Hard to distinguish expected vs unexpected errors
- Error messages lost in exception type
- Logging mixed with control flow
- Hard to test different scenarios

### Example 2: API Error Responses

```csharp
[HttpGet]
public IActionResult GetCustomer(Guid id)
{
    try
    {
        var customer = _repository.GetCustomer(id)
            ?? throw new NotFoundException("Customer not found");

        return Ok(customer);
    }
    catch (Exception ex)
    {
        // All errors return 500
        return StatusCode(500, new { error = "An error occurred" });
        // Specific error information lost!
    }
}
```

**Problems:**
- Validation errors (404) return 500 (Internal Server Error)
- Business rule violations return 500
- Clients can't distinguish error types
- Meaningless error messages
- Poor API design

---

## Common Exception Anti-Patterns

### Anti-Pattern 1: Throwing Exceptions for Validation

```csharp
// WRONG: Validation throws exception
public Email Create(string email)
{
    if (!IsValidEmail(email))
        throw new ValidationException("Invalid email");
}

// RIGHT: Validation returns Result
public Result<Email> Create(string email)
{
    if (!IsValidEmail(email))
        return Result.Failure<Email>("Invalid email");

    return Result.Success(new Email(email));
}
```

### Anti-Pattern 2: Throwing for Expected Conditions

```csharp
// WRONG: Expected business rules throw exceptions
if (customer.CreditLimit < order.Amount)
    throw new InsufficientCreditException();

// RIGHT: Return Result for expected conditions
if (customer.CreditLimit < order.Amount)
    return Result.Failure<InsufficientCredit>("Insufficient credit");
```

### Anti-Pattern 3: Exception for Control Flow

```csharp
// WRONG: Using exceptions for control flow
try
{
    var customer = _repository.GetCustomer(id);
    if (customer == null)
        throw new NotFoundException();

    return Ok(customer);
}
catch (NotFoundException)
{
    return NotFound();
}

// RIGHT: Use Result pattern
var result = _repository.GetCustomer(id);
if (result.IsFailure)
    return NotFound(result.Error);

return Ok(result.Value);
```

---

## What We Need

We need a way to:
1. **Represent** success and failure clearly
2. **Avoid** throwing exceptions for expected failures
3. **Aggregate** multiple validation errors
3. **Chain** operations gracefully
4. **Propagate** errors through call stack
5. **Distinguish** expected vs unexpected errors
6. **Keep** error information available
7. **Maintain** control flow without exceptions
8. **Support** both synchronous and async operations
9. **Enable** functional-style error handling
10. **Improve** testability

---

## The Solution: Result Pattern

In [Part 2](/2025/01/13/result-pattern-chapter-02.html), we'll explore the **Result pattern** and how it solves all these error handling problems.

---

## Key Takeaways

- Exception-driven error handling causes many problems
- Exceptions should be for truly unexpected errors
- Business validation should not throw exceptions
- Error information is often lost in exceptions
- The solution is Result pattern
- Next: Learn about Result pattern in Part 2

---

## References

- [Source Code: TestNest.ResultPattern](https://github.com/DanteTuraSalvador/TestNest.ResultPattern)
- [Wikipedia: Result Type](https://en.wikipedia.org/wiki/Result_type)

