---
layout: post
title: "Chapter 4: Advanced - Chaining, Mapping, and Async Support"
date: 2025-01-15
category: ddd
tags:
  - ddd
  - result-pattern
  - advanced
series: result-pattern
chapter: 4
prerequisites: "Chapter 3"
estimated_time: "25 minutes"
prev_title: "Chapter 3: Implementation - Result and Result<T>"
prev_url: "/ddd/2025/01/14/result-pattern-chapter-03.html"
---

# Chapter 4: Advanced - Chaining, Mapping, and Async Support

## Learning Objectives

By the end of this chapter, you will understand:
- How to chain operations with Bind
- How to transform values with Map
- How to handle asynchronous operations
- Pattern matching with Match
- Performance optimizations
- Real-world complex scenarios

---

## Advanced: Chaining Operations

### Bind - Chain Operations Returning Result

```csharp
public static class ResultExtensions
{
    public static Result<TResult> Bind<T, TResult>(
        this Result<T> result,
        Func<T, Result<TResult>> bind)
    {
        if (result.IsFailure)
            return Result<TResult>.Failure(result.Error!);

        return bind(result.Value!);
    }
}
```

### Example: Validation Chain

```csharp
public Result<Order> CreateOrder(Guid customerId, List<Guid> productIds)
{
    // Chain validation steps
    return Result.Success(customerId)
        .Bind(id => ValidateCustomer(id))
        .Bind(customer => ValidateProducts(productIds))
        .Bind(products => CreateOrderEntity(customer, products))
        .Bind(order => SaveOrder(order));
}

private Result<Customer> ValidateCustomer(Guid id)
{
    var customer = _repository.GetCustomer(id);
    if (customer == null)
        return Result<Customer>.Failure(Error.Create(
            "Customer not found",
            ErrorType.NotFound));

    return Result<Customer>.Success(customer);
}

private Result<List<Product>> ValidateProducts(List<Guid> productIds)
{
    var products = _repository.GetProducts(productIds);

    // Collect all validation errors
    var errors = new List<Error>();
    foreach (var product in products)
    {
        if (!product.IsActive)
            errors.Add(Error.Create($"Product {product.Id} is not active", ErrorType.Validation));
    }

    if (errors.Any())
        return Result<List<Product>>.Failure(Error.CreateAggregate(errors));

    return Result<List<Product>>.Success(products);
}

private Result<Order> CreateOrderEntity(Customer customer, List<Product> products)
{
    // Business rule: Maximum 10 items
    if (products.Count > 10)
        return Result<Order>.Failure(Error.Create(
            "Cannot create order with more than 10 items",
            ErrorType.Validation));

    // Business rule: Minimum total value
    var total = products.Sum(p => p.Price.Amount);
    if (total < 10)
        return Result<Order>.Failure(Error.Create(
            "Minimum order value is $10",
            ErrorType.Validation));

    var order = new Order(customer, products);
    return Result<Order>.Success(order);
}
```

---

## Advanced: Mapping Operations

### Map - Transform Values on Success

```csharp
public static class ResultExtensions
{
    public static Result<TResult> Map<T, TResult>(
        this Result<T> result,
        Func<T, TResult> map)
    {
        if (result.IsFailure)
            return Result<TResult>.Failure(result.Error!);

        var mappedValue = map(result.Value!);
        return Result<TResult>.Success(mappedValue);
    }
}
```

### Example: Transform Customer to DTO

```csharp
public Result<CustomerDto> GetCustomerDto(Guid id)
{
    return _repository.GetById(id)
        .Map(customer => new CustomerDto
        {
            Id = customer.Id,
            Name = customer.FullName,
            Email = customer.Email.Value,
            CreatedAt = customer.CreatedAt
        });
}

public record CustomerDto
{
    public Guid Id { get; init; }
    public string Name { get; init; }
    public string Email { get; init; }
    public DateTime CreatedAt { get; init; }
}
```

### Example: Complex Transformations

```csharp
public Result<OrderSummary> CreateOrderSummary(Guid orderId)
{
    return _repository.GetOrder(orderId)
        .Bind(order => ValidateOrderForSummary(order))
        .Map(order => new OrderSummary
        {
            OrderId = order.Id,
            Customer = order.Customer.FullName,
            Total = CalculateTotal(order.Items),
            Status = DetermineStatus(order)
        });
}

private Result<Order> ValidateOrderForSummary(Order order)
{
    if (order.Items.Count == 0)
        return Result<Order>.Failure(Error.Create(
            "Order has no items",
            ErrorType.Validation));

    return Result<Order>.Success(order);
}
```

---

## Advanced: Asynchronous Operations

### BindAsync - Chain Async Operations

```csharp
public static class ResultExtensions
{
    public static async Task<Result<TResult>> BindAsync<T, TResult>(
        this Result<T> result,
        Func<T, Task<Result<TResult>>> bindAsync)
    {
        if (result.IsFailure)
            return Task.FromResult(Result<TResult>.Failure(result.Error!));

        return await bindAsync(result.Value!);
    }
}
```

### Example: Async Service Chain

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IPaymentService _paymentService;
    private readonly INotificationService _notificationService;

    public async Task<Result<Order>> CreateOrderAsync(Guid customerId, List<Guid> productIds)
    {
        return await Result.Success(customerId)
            .BindAsync(async id => await GetCustomerAsync(id))
            .BindAsync(async customer => await ValidateProductsAsync(productIds))
            .BindAsync(async products => await CreateOrderEntityAsync(customer, products))
            .BindAsync(async order => await SaveOrderAsync(order))
            .BindAsync(async order => await ProcessPaymentAsync(order))
            .BindAsync(async order => await SendNotificationAsync(order));
    }

    private Task<Result<Customer>> GetCustomerAsync(Guid id)
    {
        var customer = await _repository.GetByIdAsync(id);
        if (customer == null)
            return Task.FromResult(Result<Customer>.Failure(Error.Create(
                "Customer not found",
                ErrorType.NotFound)));

        return Task.FromResult(Result<Customer>.Success(customer));
    }

    private Task<Result<List<Product>>> ValidateProductsAsync(List<Guid> productIds)
    {
        var products = await _repository.GetProductsAsync(productIds);
        var errors = new List<Error>();

        foreach (var product in products)
        {
            if (!await IsProductAvailableAsync(product))
            {
                errors.Add(Error.Create(
                    $"Product {product.Id} is not available",
                    ErrorType.Validation));
            }
        }

        if (errors.Any())
            return Task.FromResult(Result<List<Product>>.Failure(Error.CreateAggregate(errors)));

        return Task.FromResult(Result<List<Product>>.Success(products));
    }
}
```

### MapAsync - Transform Async Results

```csharp
public static class ResultExtensions
{
    public static async Task<Result<TResult>> MapAsync<T, TResult>(
        this Result<T> result,
        Func<T, Task<TResult>> mapAsync)
    {
        if (result.IsFailure)
            return Task.FromResult(Result<TResult>.Failure(result.Error!));

        var mappedValue = await mapAsync(result.Value!);
        return Task.FromResult(Result<TResult>.Success(mappedValue));
    }
}
```

---

## Advanced: Pattern Matching with Match

### Match Expression for Branching

```csharp
// C# 7+ pattern matching
public static class ResultExtensions
{
    public static TResult Match<TResult>(this Result result,
        Func<TResult> onSuccess,
        Func<Error, TResult> onFailure)
    {
        return result.IsSuccess ? onSuccess() : onFailure(result.Error!);
    }
}
```

### Example: Branching Logic

```csharp
public IActionResult GetCustomer(Guid id)
{
    var result = _repository.GetById(id);

    return result.Match(
        onSuccess: customer => Ok(new CustomerDto(customer)),
        onFailure: error => error.Type switch
        {
            ErrorType.NotFound => NotFound(error.Message),
            ErrorType.Unauthorized => Unauthorized(error.Message),
            ErrorType.Validation => BadRequest(error.Message),
            _ => StatusCode(500, error.Message)
        }
    );
}
```

### Example: Deconstruction with Pattern Matching

```csharp
public void ProcessOrder(Guid orderId)
{
    var result = _repository.GetOrder(orderId);

    var (isSuccess, order, error) = result;

    if (isSuccess)
    {
        Console.WriteLine($"Processing order: {order.Id}");
        ProcessOrder(order);
    }
    else
    {
        Console.WriteLine($"Failed: {error.Message}");
        HandleError(error);
    }
}
```

---

## Advanced: Error Aggregation

### Collect Multiple Errors

```csharp
public static class ResultExtensions
{
    public static Result<T> CollectErrors<T>(this Result<T> result, params Result<T>[] additionalResults)
    {
        var allResults = new[] { result }.Concat(additionalResults).ToArray();
        return Result.Combine(allResults);
    }
}
```

### Example: Form Validation

```csharp
public Result<Customer> RegisterCustomer(
    string email,
    string password,
    string confirmPassword,
    DateOnly birthDate)
{
    var results = new List<Result<object>>();

    // Validate email
    var emailResult = Email.Create(email);
    results.Add(emailResult);

    // Validate password
    var passwordResult = Password.Validate(password);
    results.Add(passwordResult);

    // Validate confirm password
    if (password != confirmPassword)
    {
        results.Add(Result<object>.Failure(Error.Create(
            "Passwords do not match",
            ErrorType.Validation)));
    }
    else
    {
        results.Add(Result<object>.Success(null));
    }

    // Validate age
    var age = CalculateAge(birthDate);
    if (age < 18)
    {
        results.Add(Result<object>.Failure(Error.Create(
            "Must be at least 18 years old",
            ErrorType.Validation)));
    }
    else
    {
        results.Add(Result<object>.Success(age));
    }

    // Combine all validation results
    var combined = Result.Combine(results.ToArray());

    if (combined.IsSuccess)
    {
        var customer = CreateCustomer(email, password, age);
        return Result<Customer>.Success(customer);
    }

    return Result<Customer>.Failure(combined.Error!);
}
```

---

## Performance Optimizations

### Optimization 1: Reduce Allocations

```csharp
// Cache frequently used errors
public static class CommonErrors
{
    public static readonly Error NotFound = Error.Create("Not found", ErrorType.NotFound);
    public static readonly Error Unauthorized = Error.Create("Unauthorized", ErrorType.Unauthorized);
    public static readonly Error ValidationError = Error.Create("Validation failed", ErrorType.Validation);
}

public Result<Customer> GetCustomer(Guid id)
{
    var customer = _repository.GetCustomer(id);
    if (customer == null)
        return Result<Customer>.Failure(CommonErrors.NotFound);

    return Result<Customer>.Success(customer);
}
```

### Optimization 2: Avoid Boxed Comparisons

```csharp
public abstract class Result<T>
{
    // Direct value comparison without boxing
    public override bool Equals(object? obj)
    {
        return obj is Result<T> other && Value!.Equals(other.Value);
    }

    // Efficient hash code
    public override int GetHashCode()
    {
        return Value?.GetHashCode() ?? 0;
    }
}
```

### Performance Benchmarks

From TestNest.ResultPattern benchmarking:

| Operation | Time | Notes |
|-----------|------|-------|
| Success(value) | 0.05 μs | Object creation |
| Bind (success path) | 0.15 μs | Extra Result object |
| Map (success path) | 0.12 μs | Extra Result object + delegate call |
| Combine (all success) | 0.20 μs | Depends on array size |
| Match (success path) | 0.08 μs | Pattern matching |
| Match (failure path) | 0.10 μs | Pattern matching |

**Overhead is minimal** for typical applications.

---

## Real-World: Complete API Endpoint

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    // Validate request
    var validationResult = ValidateRequest(request);
    if (validationResult.IsFailure)
        return BadRequest(validationResult.Error);

    // Chain all operations
    var result = await Result.Success(request.CustomerId)
        .BindAsync(async id => await GetCustomerAsync(id))
        .BindAsync(async customer => await ValidateInventoryAsync(request.ProductIds))
        .BindAsync(async inventory => await CreateOrderAsync(customer, inventory, request.ShippingAddress))
        .BindAsync(async order => await ProcessPaymentAsync(order))
        .BindAsync(async order => await SendConfirmationAsync(order));

    // Return appropriate response
    return result.Match(
        onSuccess: order => Ok(new OrderDto(order)),
        onFailure: error => error.Type switch
        {
            ErrorType.NotFound => NotFound(error.Message),
            ErrorType.Validation => BadRequest(error.Message),
            ErrorType.Conflict => Conflict(error.Message),
            _ => StatusCode(500, error.Message)
        }
    );
}

private Result<object> ValidateRequest(CreateOrderRequest request)
{
    // Validate email
    var emailResult = Email.Create(request.Email);
    if (emailResult.IsFailure)
        return Result<object>.Failure(emailResult.Error);

    // Validate product IDs
    if (request.ProductIds == null || !request.ProductIds.Any())
        return Result<object>.Failure(Error.Create(
            "At least one product is required",
            ErrorType.Validation));

    // Validate shipping address
    if (request.ShippingAddress == null)
        return Result<object>.Failure(Error.Create(
            "Shipping address is required",
            ErrorType.Validation));

    return Result<object>.Success(null);
}
```

---

## Testing Advanced Result Patterns

```csharp
[Test]
public async Task BindAsync_ChainExecutesInOrder()
{
    var result = await CreateOrderAsync(customerId, productIds);

    Assert.IsTrue(result.IsSuccess);
    // Verify order was saved to database
    Assert.IsNotNull(_repository.GetOrder(result.Value.Id));
}

[Test]
public void BindAsync_Failure_StopsChain()
{
    var mockRepo = new MockCustomerRepository();
    mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
                  .ReturnsAsync((Result<Customer>.Failure(CommonErrors.NotFound)));

    var result = await CreateOrderAsyncWithMock(mockRepo);

    Assert.IsTrue(result.IsFailure);
    Assert.AreEqual(ErrorType.NotFound, result.Error!.Type);
}

[Test]
public void Map_TransformsValueOnSuccess()
{
    var result = _repository.GetById(customerId);

    var mapped = result.Map(customer => customer.FullName);

    Assert.IsTrue(mapped.IsSuccess);
    Assert.AreEqual(customer.FullName, mapped.Value);
}

[Test]
public void Map_PreservesFailure()
{
    var result = _repository.GetById(nonExistentId);

    var mapped = result.Map(customer => customer.FullName);

    Assert.IsTrue(mapped.IsFailure);
    Assert.AreEqual(result.Error, mapped.Error);
}

[Test]
public void Match_SuccessBranch_ReturnsCorrectResult()
{
    var result = Result<int>.Success(42);

    var branchResult = result.Match(
        onSuccess: value => Result<string>.Success($"Value is {value}"),
        onFailure: error => Result<string>.Failure(error.Message)
    );

    Assert.IsTrue(branchResult.IsSuccess);
    Assert.AreEqual("Value is 42", branchResult.Value);
}
```

---

## Best Practices for Advanced Results

1. **Use Bind** for chaining Result-returning operations
2. **Use Map** for transforming values on success
3. **Use Match** for branching logic based on success/failure
4. **Use async variants** (BindAsync, MapAsync) for async chains
5. **Collect errors** for comprehensive validation
6. **Cache common errors** to reduce allocations
7. **Use deconstruction** for pattern matching
8. **Keep operations short** - avoid deep nesting
9. **Provide rich error metadata**
10. **Test all paths** - success, failure, and edge cases

---

## Complete Source Code

**Full Implementation:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.ResultPattern)

**Extensions:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.ResultPattern/blob/main/TestNest.ResultPattern.Domain/Extensions/ResultExtensions.cs)

**Tests:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.ResultPattern/blob/main/TestNest.ResultPattern.Tests/ResultExtensionsTests.cs)

---

## Key Takeaways

- Bind enables chaining Result-returning operations
- Map transforms values while preserving Result
- Async variants support asynchronous workflows
- Match provides clean pattern matching
- Error aggregation is built-in
- Test all success and failure paths
- Series complete! Next: Clean Architecture

---

## What's Next?

Congratulations on completing Result Pattern series!

Continue your DDD journey with:
- [Clean Architecture](/2025/01/16/clean-architecture-chapter-01.html) - Architectural patterns
- [CQRS](#) - Command Query Separation

