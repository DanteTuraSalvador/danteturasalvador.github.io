---
layout: post
title: "Part 1: Problem - Layered Architecture Issues"
date: 2025-01-16
category: ddd
thumbnail-img: "https://images.pexels.com/photos/1181677/pexels-photo-1181677.jpeg?auto=compress&cs=tinysrgb&w=400&h=200&fit=crop"
tags:
  - ddd
  - clean-architecture
  - beginner
series: clean-architecture
chapter: 1
prerequisites: None
estimated_time: "15 minutes"
prev_title: "Back to Result Pattern Series"
prev_url: "/ddd/2025/01/15/result-pattern-chapter-04.html"
next_title: "Part 2: Solution - Clean Architecture"
next_url: "/ddd/2025/01/17/clean-architecture-chapter-02.html"
---

# Part 1: Problem - Layered Architecture Issues

## Learning Objectives

By the end of this chapter, you will understand:
- What is layered architecture
- Common problems with traditional layered approaches
- Why dependencies create tight coupling
- How layer violations lead to maintenance nightmares
- The impact on testability and flexibility

---

## What is Layered Architecture?

Layered architecture organizes code into horizontal layers:
- **Presentation Layer** - Controllers, API endpoints
- **Application Layer** - Business logic, use cases
- **Domain Layer** - Core business rules, entities
- **Infrastructure Layer** - Database, external services
- **Data Access Layer** - Repositories (sometimes part of infrastructure)

### Traditional Layered Architecture Issues

In many implementations, layered architecture has problems:
- **Dependencies flow inward** - Presentation → Application → Domain → Infrastructure
- **Domain depends on infrastructure** - Entities reference repositories directly
- **Circular dependencies** - Hard to manage
- **Tight coupling** - Layers can't be easily swapped
- **Testing difficulties** - Mocking infrastructure is hard
- **Business logic leaks** - Domain rules in wrong layers

---

## Why This Matters

**Imagine maintaining a codebase with:**
- Domain entities directly calling repositories
- Application layer mixed with infrastructure concerns
- Tests requiring complex mocking setups
- Business rules scattered across layers
- Hard to understand code flow
- Fear of breaking code when changing layers

---

## The Cost of Bad Layered Architecture

| Problem | Impact |
|---------|--------|
| Domain-Infrastructure coupling | Hard to test domain in isolation |
| Circular dependencies | Build complexity and errors |
| Tight coupling | Changes ripple through layers |
| Business logic leakage | Inconsistent rules |
| Testing complexity | Requires mocking of entire stack |
| Maintenance nightmare | Simple changes break multiple layers |
| Flexibility issues | Can't swap implementations |
| Reusability issues | Can't use domain in different contexts |

---

## Real-World Impact

### Example 1: Domain Entity with Repository Dependency

```csharp
public class Customer
{
    public CustomerId Id { get; set; }
    public string Name { get; set; }
    public Email Email { get; set; }
    
    // BAD: Direct dependency on infrastructure
    private readonly ICustomerRepository _repository;
    
    public Customer(IEmailService emailService, ICustomerRepository repository)
    {
        Email = Email.Create("test@example.com").Value;
        _repository = repository;  // Domain depends on infrastructure!
    }
    
    public void Save()
    {
        _repository.Add(this); // Domain knows about persistence!
    _repository.SaveChanges();
    }
}
```

**Problems:**
- Domain entity depends on repository
- Can't test domain logic without database
- Business rules mixed with persistence logic
- Domain not reusable in different contexts

### Example 2: Application Layer with Database Concerns

```csharp
public class CustomerService
{
    private readonly ICustomerRepository _repository;
    private readonly IEmailService _emailService;
    
    public CustomerService(ICustomerRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }
    
    public async Task<Customer> CreateCustomerAsync(string name, string email)
    {
        // BAD: Application knows about database entities
        var customer = new Customer(_emailService, _repository);
        customer.Name = name;
        customer.Email = Email.Create(email).Value;
        
        // Application directly manipulating infrastructure
        await _repository.AddAsync(customer);
        await _repository.SaveChangesAsync();
        
        return customer;
    }
    
    public async Task<List<Customer>> GetActiveCustomersAsync()
    {
        // BAD: Application layer doing database queries
        return await _repository.Query(c => c.IsActive)
            .ToListAsync();
    }
}
```

**Problems:**
- Application layer knows about database entities
- Business rules in wrong place
- Can't swap infrastructure easily
- Testing requires full database setup

### Example 3: Repository Exposing Domain

```csharp
public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(Guid id);
    Task<List<Customer>> GetAllAsync();
    Task<Customer> AddAsync(Customer customer);
    Task SaveChangesAsync();
}

public class CustomerRepository : ICustomerRepository
{
    private readonly DbContext _context;
    
    public CustomerRepository(DbContext context)
    {
        _context = context;
    }
    
    public async Task<Customer?> GetByIdAsync(Guid id)
    {
        // BAD: Returns domain entity directly
        return await _context.Customers.FindAsync(id);
    }
    
    public async Task<List<Customer>> GetAllAsync()
    {
        // BAD: Repository knows about EF Core entities
        return await _context.Customers.ToListAsync();
    }
}
```

**Problems:**
- Repository exposes infrastructure entities
- Domain depends on EF Core types
- Tight coupling to ORM
- Hard to test without database
- Can't swap data access implementation

---

## Common Architecture Violations

### Violation 1: Domain Depends on Infrastructure

```csharp
public class Order
{
    // BAD: Domain knows about repository
    private readonly IOrderRepository _repository;
    
    public void AddItem(OrderItem item)
    {
        // Domain logic depends on persistence
        _repository.AddItem(item);
    }
}
```

### Violation 2: Infrastructure in Domain

```csharp
public class Customer : Entity
{
    // BAD: Database concerns in domain
    [Key]
    public int Id { get; set; }
    [Column(TypeName = "varchar(255)")]
    public string Name { get; set; }
}
```

### Violation 3: Cross-Layer Calls

```csharp
// BAD: Presentation calls infrastructure directly
public class CustomerController
{
    private readonly DbContext _context;
    
    public IActionResult GetCustomer(Guid id)
    {
        // Skips all layers
        var customer = _context.Customers.Find(id);
        return Ok(customer);
    }
}
```

### Violation 4: Business Logic in Wrong Layer

```csharp
// BAD: Business rules in application layer
public class CustomerService
{
    public bool CanCreateCustomer(string email)
    {
        // Business rule in wrong place
        if (string.IsNullOrWhiteSpace(email))
            return false;
        
        if (email.Contains("@"))
            return false;
            
        return true;
    }
}
```

---

## What We Need

We need an architecture that:
1. **Isolates domain** from infrastructure
2. **Separates concerns** between layers
3. **Enforces dependency rules** - Dependencies flow inward only
4. **Prevents circular dependencies**
5. **Enables testing** - Test domain without infrastructure
6. **Makes domain reusable** - Use in any context
7. **Keeps business logic** where it belongs
8. **Supports multiple implementations** - Easy to swap layers
9. **Follows dependency inversion** - Depends on abstractions
10. **Enforces clean boundaries** - Layers have clear responsibilities

---

## The Solution: Clean Architecture

In [Part 2](/2025/01/17/clean-architecture-chapter-02.html), we'll explore the **Clean Architecture pattern** and how it solves all these architectural problems.

---

## Key Takeaways

- Traditional layered architecture has dependency problems
- Domain depends on infrastructure in bad implementations
- Tight coupling makes maintenance difficult
- Testing is complex due to dependencies
- The solution is Clean Architecture
- Next: Learn about Clean Architecture in Part 2

---

## References

- [Source Code: MinimalAPICleanArchitecture](https://github.com/DanteTuraSalvador/MinimalAPICleanArchitecture)
- [Robert C. Martin: Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Microsoft: Clean Architecture with ASP.NET Core](https://docs.microsoft.com/en-us/dotnet/standard/architecture/clean-architecture)

