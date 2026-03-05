---
layout: post
title: "Part 3: Implementation - Project Structure"
date: 2025-01-18
category: ddd
thumbnail-img: "https://images.pexels.com/photos/943096/pexels-photo-943096.jpeg?auto=compress&cs=tinysrgb&w=400&h=200&fit=crop"
tags:
  - ddd
  - clean-architecture
  - implementation
series: clean-architecture
chapter: 3
prerequisites: "Part 2"
estimated_time: "30 minutes"
prev_title: "Part 2: Solution - Clean Architecture"
prev_url: "/ddd/2025/01/17/clean-architecture-chapter-02.html"
next_title: "Part 4: Domain Layer Implementation"
next_url: "/ddd/2025/01/19/clean-architecture-chapter-04.html"
---

# Part 3: Implementation - Project Structure

## Learning Objectives

By the end of this chapter, you will be able to:
- Set up a Clean Architecture project structure
- Configure dependency injection
- Implement cross-cutting concerns
- Create DTOs and mappers
- Set up project references and dependencies

---

## Implementation: Step by Step

### Step 1: Create Solution Structure

```bash
# Create solution folders
mkdir src
cd src
mkdir Domain Application Infrastructure Presentation Tests
```

### Step 2: Add Projects to Solution

```bash
# Create .NET projects
dotnet new sln -n TestNest.CleanArchitecture
cd TestNest.CleanArchitecture

# Domain project
dotnet new classlib -n TestNest.CleanArchitecture.Domain

# Application project  
dotnet new classlib -n TestNest.CleanArchitecture.Application

# Infrastructure project
dotnet new classlib -n TestNest.CleanArchitecture.Infrastructure

# Presentation project
dotnet new webapi -n TestNest.CleanArchitecture.API

# Tests project
dotnet new xunit -n TestNest.CleanArchitecture.Tests
```

### Step 3: Configure Project References

```bash
# Domain references (none - core layer)
# No project references

# Application references
cd Application
dotnet add reference ../Domain

# Infrastructure references
cd Infrastructure
dotnet add reference ../Domain

# Presentation references
cd ../Presentation/API
dotnet add reference ../Application
dotnet add reference ../Domain

# Tests references
cd ../Tests
dotnet add reference ../Domain
dotnet add reference ../Application
dotnet add reference ../Infrastructure
```

### Step 4: Add Required Packages

```bash
# Domain packages
cd ../Domain
dotnet add package FluentValidation

# Application packages
cd ../Application
dotnet add package MediatR
dotnet add package AutoFixture
dotnet add package Moq

# Infrastructure packages
cd ../Infrastructure
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Dapper
dotnet add package Serilog.AspNetCore
dotnet add package StackExchange.Redis

# Presentation packages
cd ../Presentation/API
dotnet add package Swashbuckle.AspNetCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Serilog.AspNetCore
```

---

### Step 5: Configure Dependency Injection

**Program.cs**
```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using TestNest.CleanArchitecture.Application.Services;
using TestNest.CleanArchitecture.Infrastructure.Persistence;

var builder = WebApplication.CreateBuilder(args);

// Add services to DI container
builder.Services.AddApplicationServices();

// Add infrastructure
builder.Services.AddInfrastructureServices(builder.Configuration);

// Add Serilog
builder.Host.UseSerilog((context, services, configuration) =>
{
    Serilog.UseSerilogRequestLogging();
    WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties}{NewLine}{Exception}")
    ).CreateLogger();
});

var app = builder.Build();

app.MapControllers();
app.Run();
```

---

## Domain Layer Implementation

### Entities

```csharp
namespace TestNest.CleanArchitecture.Domain.Entities;

public class Customer
{
    public CustomerId Id { get; }
    public string Name { get; private set; }
    public Email Email { get; private set; }
    public PhoneNumber Phone { get; private set; }
    public Address Address { get; private set; }
    public DateOnly CreatedAt { get; private set; }

    public Customer(CustomerId id, string name, Email email, 
                   PhoneNumber phone, Address address)
    {
        Id = id;
        Name = name;
        Email = email;
        Phone = phone;
        Address = address;
        CreatedAt = DateOnly.FromDateTime(DateTime.UtcNow);
    }

    public void ChangeEmail(Email newEmail)
    {
        Email = newEmail;
    }

    public void ChangePhone(PhoneNumber newPhone)
    {
        Phone = newPhone;
    }

    public void ChangeAddress(Address newAddress)
    {
        Address = newAddress;
    }
}

public class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public Money TotalAmount { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public OrderStatus Status { get; private set; }

    public Order(OrderId id, CustomerId customerId, Money totalAmount)
    {
        Id = id;
        CustomerId = customerId;
        TotalAmount = totalAmount;
        CreatedAt = DateTime.UtcNow;
        Status = OrderStatus.Created;
    }

    public void AddItem(OrderItem item)
    {
        TotalAmount = TotalAmount.Add(item.LineTotal);
    }

    public void MarkAsPaid()
    {
        Status = OrderStatus.Paid;
    }
}

public enum OrderStatus
{
    Created,
    PaymentProcessing,
    Paid,
    Shipped,
    Delivered,
    Canceled
}
```

---

## Application Layer Implementation

### Services

```csharp
namespace TestNest.CleanArchitecture.Application.Services;

public class CustomerService : ICustomerService
{
    private readonly ICustomerRepository _repository;
    private readonly IEmailService _emailService;

    public CustomerService(ICustomerRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }

    public async Task<Result<Customer>> CreateCustomerAsync(CreateCustomerCommand command)
    {
        // Validate email
        var emailResult = Email.Create(command.Email);
        if (emailResult.IsFailure)
            return Result<Customer>.Failure(emailResult.Error);

        // Validate phone
        var phoneResult = PhoneNumber.Create(command.Phone);
        if (phoneResult.IsFailure)
            return Result<Customer>.Failure(phoneResult.Error);

        // Check for existing customer
        var existing = await _repository.GetByEmailAsync(command.Email);
        if (existing != null)
            return Result<Customer>.Failure(Error.Create(
                "Customer with this email already exists",
                ErrorType.Conflict));

        // Create customer entity
        var customer = new Customer(
            CustomerId.NewGuid(),
            command.Name,
            emailResult.Value,
            phoneResult.Value,
            Address.Create(command.Street, command.City, 
                           command.State, command.PostalCode, 
                           command.Country).Value
        );

        // Persist
        var saved = await _repository.AddAsync(customer);

        // Send welcome email
        await _emailService.SendWelcomeEmailAsync(saved);

        return Result<Customer>.Success(saved);
    }
}
```

### Use Cases (Commands)

```csharp
namespace TestNest.CleanArchitecture.Application.UseCases;

public record CreateCustomerCommand
{
    public string Name { get; init; }
    public string Email { get; init; }
    public string Phone { get; init; }
    public string Street { get; init; }
    public string City { get; init; }
    public string State { get; init; }
    public string PostalCode { get; init; }
    public string Country { get; init; }
}

public interface ICreateCustomerHandler
    : IRequestHandler<CreateCustomerCommand, Result<Customer>>
{
}

public class CreateCustomerHandler : ICreateCustomerHandler
{
    public async Task<Result<Customer>> Handle(CreateCustomerCommand command, 
                                                          CancellationToken cancellationToken)
    {
        var customerService = _serviceProvider.GetService<IEmailService>();
        return await customerService.CreateCustomerAsync(command);
    }
}
```

---

## Infrastructure Layer Implementation

### Repository Interface (in Domain)

```csharp
namespace TestNest.CleanArchitecture.Domain.Interfaces;

public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(CustomerId id);
    Task<Customer?> GetByEmailAsync(string email);
    Task<List<Customer>> GetAllAsync();
    Task<Customer> AddAsync(Customer customer);
    Task UpdateAsync(Customer customer);
    Task<bool> ExistsByEmailAsync(string email);
}
```

### Repository Implementation

```csharp
namespace TestNest.CleanArchitecture.Infrastructure.Persistence;

public class CustomerRepository : ICustomerRepository
{
    private readonly DbContext _context;

    public CustomerRepository(DbContext context)
    {
        _context = context;
    }

    public async Task<Customer?> GetByIdAsync(CustomerId id)
    {
        return await _context.Customers.FindAsync(id.Value);
    }

    public async Task<Customer?> GetByEmailAsync(string email)
    {
        return await _context.Customers
            .FirstOrDefaultAsync(c => c.Email.Value == email);
    }

    public async Task<List<Customer>> GetAllAsync()
    {
        return await _context.Customers.ToListAsync();
    }

    public async Task<Customer> AddAsync(Customer customer)
    {
        await _context.Customers.AddAsync(customer);
        await _context.SaveChangesAsync();
        return customer;
    }

    public async Task<bool> ExistsByEmailAsync(string email)
    {
        return await _context.Customers
            .AnyAsync(c => c.Email.Value == email);
    }
}
```

---

## Presentation Layer Implementation

### Controller

```csharp
namespace TestNest.CleanArchitecture.API.Controllers;

[ApiController]
[Route("api/[controller]")]
public class CustomerController : ControllerBase
{
    private readonly IMediator _mediator;

    public CustomerController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    [Route("customers")]
    public async Task<IActionResult> CreateCustomer(CreateCustomerCommand command)
    {
        var result = await _mediator.Send(command);

        if (result.IsFailure)
        {
            return BadRequest(result.Error);
        }

        return Ok(new CustomerDto(result.Value));
    }

    [HttpGet]
    [Route("customers/{id:guid}")]
    public async Task<IActionResult> GetCustomer(Guid id)
    {
        var query = new GetCustomerQuery(id);
        var result = await _mediator.Send(query);

        if (result.IsFailure)
        {
            return NotFound(result.Error.Message);
        }

        return Ok(result.Value);
    }
}
```

---

## Cross-Cutting Concerns

### Logging

```csharp
public static class LoggingExtensions
{
    public static IServiceCollection AddLogging(this IServiceCollection services)
    {
        return services.AddSerilog()
            .EnrichFromContext()
            .ReadFrom.Configuration()
            .CreateLogger();
    }
}
```

### Caching

```csharp
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null);
    Task RemoveAsync(string key);
}

public class RedisCacheService : ICacheService
{
    private readonly IConnectionMultiplexer _redis;

    public RedisCacheService(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        var db = _redis.GetDatabase();
        var value = await db.StringGetAsync(key);
        
        if (value.IsNullOrEmpty)
            return default;

        return JsonSerializer.Deserialize<T>(value);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry)
    {
        var serialized = JsonSerializer.Serialize(value);
        var db = _redis.GetDatabase();
        
        if (expiry.HasValue)
            await db.StringSetAsync(key, serialized, expiry.Value);
        else
            await db.StringSetAsync(key, serialized);
    }
}
```

---

## Testing Setup

### Unit Tests with Moq

```csharp
public class CustomerServiceTests
{
    private readonly Mock<ICustomerRepository> _mockRepository;
    private readonly Mock<IEmailService> _mockEmailService;
    private readonly CustomerService _service;

    public CustomerServiceTests()
    {
        _mockRepository = new Mock<ICustomerRepository>();
        _mockEmailService = new Mock<IEmailService>();
        _service = new CustomerService(_mockRepository.Object, _mockEmailService.Object);
    }

    [Fact]
    public async Task CreateCustomer_ValidData_ReturnsSuccess()
    {
        // Arrange
        var command = new CreateCustomerCommand("John Doe", "john@example.com", "555-1234",
                                                      "123 Main St", "Springfield", "IL", "62701", "USA");
        _mockRepository.Setup(r => r.GetByEmailAsync(It.IsAny<string>()))
                  .ReturnsAsync((Customer?)null);
        _mockEmailService.Setup(e => e.SendWelcomeEmailAsync(It.IsAny<Customer>()))
                  .Returns(Task.CompletedTask);

        // Act
        var result = await _service.CreateCustomerAsync(command);

        // Assert
        Assert.True(result.IsSuccess);
        _mockRepository.Verify(r => r.AddAsync(It.IsAny<Customer>()), Times.Once);
        _mockEmailService.Verify(e => e.SendWelcomeEmailAsync(It.IsAny<Customer>()), Times.Once);
    }
}
```

---

## Complete Source Code

**Full Project:**
[View on GitHub →](https://github.com/DanteTuraSalvador/MinimalAPICleanArchitecture)

**Key Files:**
- Domain/Entities/Customer.cs
- Application/Services/CustomerService.cs
- Infrastructure/Persistence/CustomerRepository.cs
- Presentation/API/Controllers/CustomerController.cs

---

## Key Takeaways

- Organize projects by layer (Domain, Application, Infrastructure, Presentation)
- Configure DI to enforce dependency rules
- Implement interfaces for testability
- Use cross-cutting concerns for infrastructure
- Set up proper project references
- Next: Implement domain layer details in Part 4

---

## References

- [Complete Source Code: MinimalAPICleanArchitecture](https://github.com/DanteTuraSalvador/MinimalAPICleanArchitecture)
- [Microsoft: Minimal APIs](https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-web-api)
- [MediatR: Simple mediator pattern](https://github.com/jbogard/MediatR)

