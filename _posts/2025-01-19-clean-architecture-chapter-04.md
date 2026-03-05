---
layout: post
title: "Chapter 4: Domain Layer Implementation"
date: 2025-01-19
tags:
  - ddd
  - clean-architecture
  - implementation
series: clean-architecture
chapter: 4
prerequisites: "Chapter 3"
estimated_time: "25 minutes"
prev_title: "Chapter 3: Implementation - Project Structure"
prev_url: "/2025/01/18/clean-architecture-chapter-03.html"
next_title: "Chapter 5: Integration and Deployment"
next_url: "/2025/01/20/clean-architecture-chapter-05.html"
---

# Chapter 4: Domain Layer Implementation

## Learning Objectives

By the end of this chapter, you will be able to:
- Implement rich domain entities
- Create domain services for complex business logic
- Define domain events for loose coupling
- Implement specifications for queries
- Use value objects in domain modeling

---

## Domain Layer Principles

**Core Domain Layer Rules:**

1. **No infrastructure dependencies** - Pure business logic
2. **No framework dependencies** - No EF Core, no ASP.NET
3. **Rich domain model** - Behavior in entities, not anemic models
4. **Business rules encapsulated** - Invariants enforced
5. **Value objects for concepts** - No primitive obsession
6. **Domain events for changes** - Event-driven architecture
7. **Invariants at boundaries** - Entities always valid

---

## Rich Domain Entities

### Example: Customer Entity with Behavior

```csharp
namespace TestNest.CleanArchitecture.Domain.Entities;

public class Customer
{
    private readonly List<Order> _orders = new();

    public CustomerId Id { get; }
    public string Name { get; private set; }
    public Email Email { get; private set; }
    public PhoneNumber Phone { get; private set; }
    public Address Address { get; private set; }
    public DateOnly CreatedAt { get; private set; }
    public Money CreditLimit { get; private set; }

    public Customer(CustomerId id, string name, Email email, 
                   PhoneNumber phone, Address address)
    {
        Id = id;
        Name = name;
        Email = email;
        Phone = phone;
        Address = address;
        CreatedAt = DateOnly.FromDateTime(DateTime.UtcNow);
        CreditLimit = Money.Create(1000, Currency.USD).Value;
    }

    public void ChangeEmail(Email newEmail)
    {
        // Domain rule: Cannot change to existing email
        var existingCustomerWithSameEmail = CheckForDuplicateEmail(newEmail);
        if (existingCustomerWithSameEmail != null)
        {
            throw new DomainException("Email already in use");
        }

        Email = newEmail;
    }

    public void ChangePhone(PhoneNumber newPhone)
    {
        Phone = newPhone;
    }

    public void UpdateCreditLimit(Money newLimit)
    {
        if (newLimit.Amount > CreditLimit.Amount)
        throw new DomainException("New limit cannot exceed current limit");

        CreditLimit = newLimit;
    }

    public IReadOnlyList<Order> Orders => _orders.AsReadOnly();

    public void AddOrder(Order order)
    {
        // Domain rule: Check credit limit
        if (order.TotalAmount > CreditLimit)
            throw new DomainException("Order exceeds credit limit");

        _orders.Add(order);
    }

    public Money GetAvailableCredit()
    {
        var usedCredit = _orders.Sum(o => o.TotalAmount);
        return CreditLimit.Subtract(usedCredit);
    }
}
```

### Example: Order Entity with Business Logic

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();

    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public Money TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public DateTime? DeliveredAt { get; private set; }

    public Order(OrderId id, CustomerId customerId)
    {
        Id = id;
        CustomerId = customerId;
        TotalAmount = Money.Create(0, Currency.USD).Value;
        CreatedAt = DateTime.UtcNow;
        Status = OrderStatus.Created;
    }

    public void AddItem(OrderItem item)
    {
        if (Status != OrderStatus.Created && Status != OrderStatus.PaymentProcessing)
            throw new DomainException("Cannot add items to this order");

        _items.Add(item);
        RecalculateTotal();
    }

    public void MarkAsPaid()
    {
        if (Status != OrderStatus.PaymentProcessing)
            throw new DomainException("Order is not in payment processing");

        Status = OrderStatus.Paid;
    }

    public void Ship()
    {
        if (Status != OrderStatus.Paid)
            throw new DomainException("Order must be paid before shipping");

        Status = OrderStatus.Shipped;
        ShippedAt = DateTime.UtcNow;
    }

    public void Deliver()
    {
        if (Status != OrderStatus.Shipped)
            throw new DomainException("Order must be shipped before delivery");

        Status = OrderStatus.Delivered;
        DeliveredAt = DateTime.UtcNow;
    }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
            throw new DomainException("Cannot cancel shipped or delivered order");

        Status = OrderStatus.Canceled;
    }

    public bool CanCancel()
    {
        return Status == OrderStatus.Created || 
               Status == OrderStatus.PaymentProcessing;
    }

    private void RecalculateTotal()
    {
        TotalAmount = _items.Aggregate(
            Money.Create(0, Currency.USD).Value,
            (acc, item) => acc.Add(item.LineTotal));
    }
}
```

---

## Domain Services

### Interface for Complex Business Logic

```csharp
namespace TestNest.CleanArchitecture.Domain.Services;

public interface IPricingDomainService
{
    Money CalculateDiscount(Money originalPrice, decimal percentage);
    Money ApplyTax(Money amount, string stateCode);
    bool IsEligibleForDiscount(Customer customer, Money totalAmount);
}
```

### Implementation

```csharp
public class PricingDomainService : IPricingDomainService
{
    public Money CalculateDiscount(Money price, decimal percentage)
    {
        if (percentage < 0 || percentage > 50)
            throw new DomainException("Discount percentage must be between 0 and 50");

        var discountAmount = price.Amount * (percentage / 100);
        var discountedPrice = price.Subtract(new Money(discountAmount, price.Currency));

        return discountedPrice;
    }

    public Money ApplyTax(Money amount, string stateCode)
    {
        var taxRate = GetTaxRate(stateCode);
        var taxAmount = amount.Amount * (taxRate / 100);
        var taxedAmount = amount.Add(new Money(taxAmount, amount.Currency));

        return taxedAmount;
    }

    private decimal GetTaxRate(string stateCode)
    {
        return stateCode switch
        {
            "CA" => 0.0725m, // California
            "NY" => 0.08m,     // New York
            "TX" => 0.0625m,  // Texas
            "FL" => 0.06m,    // Florida
            _ => 0m          // No tax
        };
    }

    public bool IsEligibleForDiscount(Customer customer, Money totalAmount)
    {
        // Business rule: Premium customers get discount
        if (customer.MembershipLevel == MembershipLevel.Regular)
            return totalAmount.Amount > 100;

        return totalAmount.Amount > 50;
    }
}
```

---

## Domain Events

### Event Definitions

```csharp
namespace TestNest.CleanArchitecture.Domain.Events;

public record CustomerCreatedEvent(CustomerId CustomerId, string Name, DateTime CreatedAt);
public record CustomerEmailChangedEvent(CustomerId CustomerId, Email OldEmail, Email NewEmail, DateTime ChangedAt);
public record OrderPlacedEvent(OrderId OrderId, CustomerId CustomerId, Money TotalAmount, DateTime PlacedAt);
public record OrderPaidEvent(OrderId OrderId, Money Amount, DateTime PaidAt);
public record OrderShippedEvent(OrderId OrderId, DateTime ShippedAt);
public record OrderDeliveredEvent(OrderId OrderId, DateTime DeliveredAt);
```

### Using Events in Entities

```csharp
public class Customer
{
    // ... properties ...

    public event EventHandler<DomainEvent>? OnDomainEvent;

    public Customer(CustomerId id, /*...*/)
    {
        // ...

        // Create event when customer is created
        OnDomainEvent?.Invoke(new CustomerCreatedEvent(id, Name, CreatedAt));
    }
}
```

---

## Domain Specifications

### Customer Specifications

```csharp
namespace TestNest.CleanArchitecture.Domain.Specifications;

public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    Expression<Func<T, bool>> ToExpression();
}

public class CustomerHasMinimumOrdersSpecification : ISpecification<Customer>
{
    private readonly int _minimumOrders;

    public CustomerHasMinimumOrdersSpecification(int minimumOrders)
    {
        _minimumOrders = minimumOrders;
    }

    public bool IsSatisfiedBy(Customer customer)
    {
        return customer.Orders.Count >= _minimumOrders;
    }

    public Expression<Func<Customer, bool>> ToExpression()
    {
        return c => c.Orders.Count >= _minimumOrders;
    }
}

// Usage
var spec = new CustomerHasMinimumOrdersSpecification(5);
var qualifiedCustomers = customers.Where(spec.ToExpression().Compile());
```

---

## Domain Testing

### Unit Tests Without Infrastructure

```csharp
public class CustomerTests
{
    [Fact]
    public void ChangeEmail_ValidEmail_UpdatesEmail()
    {
        var customer = new Customer(
            CustomerId.NewGuid(),
            "John Doe",
            Email.Create("john@example.com").Value,
            PhoneNumber.Create("555-1234").Value,
            Address.Create("123 Main St", "Springfield", "IL", "62701", "USA").Value,
            Money.Create(1000, Currency.USD).Value);

        var newEmail = Email.Create("john.doe@example.com").Value;
        customer.ChangeEmail(newEmail);

        Assert.AreEqual(newEmail, customer.Email);
    }

    [Fact]
    public void PlaceOrder_ExceedsCreditLimit_ThrowsException()
    {
        var customer = new Customer(
            CustomerId.NewGuid(),
            "John Doe",
            Email.Create("john@example.com").Value,
            PhoneNumber.Create("555-1234").Value,
            Address.Create("123 Main St", "Springfield", "IL", "62701", "USA").Value,
            Money.Create(100, Currency.USD).Value);

        var order = new Order(OrderId.NewGuid(), customer.Id, Money.Create(150, Currency.USD).Value);

        Assert.Throws<DomainException>(() => customer.AddOrder(order));
    }
}
```

---

## Best Practices for Domain Layer

1. **Keep domain pure** - No infrastructure references
2. **Rich domain model** - Behavior in entities
3. **Use value objects** - No primitive obsession
4. **Enforce invariants** - Entities always valid
5. **Business rules encapsulated** - In domain services
6. **Domain events** - Decoupled communication
7. **Specifications** - Reusable business rules
8. **Test independently** - No infrastructure mocks needed
9. **Self-documenting** - Clear domain concepts
10. **Avoid anemic models** - Behavior, not just data

---

## Key Takeaways

- Domain layer is pure business logic
- Rich entities with behavior
- Domain services for complex logic
- Events for decoupled communication
- Specifications for reusable rules
- Test without infrastructure dependencies
- Next: Integration and deployment in Chapter 5

---

## References

- [Complete Source Code: MinimalAPICleanArchitecture](https://github.com/DanteTuraSalvador/MinimalAPICleanArchitecture)
- [DDD: Domain-Driven Design](https://domainlanguage.com/ddd/)
- [Evans: Domain-Driven Design](https://domainlanguage.com/ddd/evans.html)

