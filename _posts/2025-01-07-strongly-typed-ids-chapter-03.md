---
layout: post
title: "Chapter 3: Integration with Value Objects"
date: 2025-01-07
tags:
  - ddd
  - strongly-typed-ids
  - integration
series: strongly-typed-ids
chapter: 3
prerequisites: "Chapter 2"
estimated_time: "30 minutes"
prev_title: "Chapter 2: Solution - StronglyTypedId Pattern"
prev: "/2025/01/06/strongly-typed-ids-chapter-02.html"
---

# Chapter 3: Integration with Value Objects

## Learning Objectives

By the end of this chapter, you will be able to:
- Combine StronglyTypedId with Value Objects
- Create rich domain entities
- Use both patterns together effectively
- Test integrated solutions

---

## Combining Patterns Together

StronglyTypedId and Value Objects work perfectly together to create type-safe, self-documenting domain models.

**StronglyTypedId** provides type-safe identifiers
**Value Objects** provide type-safe domain values

Together, they create a **completely type-safe domain**.

---

## Example: Rich Customer Entity

### Using Both Patterns

```csharp
public class Customer
{
    public CustomerId Id { get; }
    public Email Email { get; }
    public PhoneNumber Phone { get; }
    public Address BillingAddress { get; }
    public Address ShippingAddress { get; }
    public Money CreditLimit { get; }

    public Customer(CustomerId id, Email email, PhoneNumber phone,
                 Address billingAddress, Address shippingAddress,
                 Money creditLimit)
    {
        Id = id;
        Email = email;
        Phone = phone;
        BillingAddress = billingAddress;
        ShippingAddress = shippingAddress;
        CreditLimit = creditLimit;
    }

    public Result<Money> UpdateEmail(string newEmail)
    {
        var emailResult = Email.Create(newEmail);

        if (emailResult.IsFailure)
            return Result.Failure<Money>(emailResult.Error);

        Email = emailResult.Value;
        return Result.Success(CreditLimit);
    }

    public Result<Address> UpdateShippingAddress(string street, string city,
                                              string state, string postalCode,
                                              string country)
    {
        var addressResult = Address.Create(street, city, state,
                                         postalCode, country);

        if (addressResult.IsFailure)
            return Result.Failure<Address>(addressResult.Error);

        ShippingAddress = addressResult.Value;
        return Result.Success(ShippingAddress);
    }
}
```

---

## Example: Order Entity with Full Integration

```csharp
public class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public Address ShippingAddress { get; }
    public Email CustomerEmail { get; }
    public PhoneNumber CustomerPhone { get; }
    public List<OrderItem> Items { get; }
    public Money Total { get; private set; }
    public Price FinalPrice { get; private set; }

    public Order(OrderId id, CustomerId customerId, Address shippingAddress,
               List<OrderItem> items)
    {
        Id = id;
        CustomerId = customerId;
        ShippingAddress = shippingAddress;
        Items = items;
        CalculateTotal();
    }

    private void CalculateTotal()
    {
        Total = Items.Aggregate(Money.Create(0, Currency.USD).Value,
                                  (acc, item) => acc.Add(item.LineTotal));
    }

    public Result<Order> ApplyDiscount(decimal percentage)
    {
        var discountedPrice = FinalPrice.ApplyDiscount(percentage);
        FinalPrice = discountedPrice;
        return Result.Success(this);
    }
}

public class OrderItem
{
    public ProductId ProductId { get; }
    public string ProductName { get; }
    public Quantity Quantity { get; }
    public Money UnitPrice { get; }
    public Money LineTotal { get; }

    public OrderItem(ProductId productId, string productName,
                   Quantity quantity, Money unitPrice)
    {
        ProductId = productId;
        ProductName = productName;
        Quantity = quantity;
        UnitPrice = unitPrice;
        LineTotal = unitPrice.Multiply(quantity.Value);
    }
}
```

---

## Benefits of Combined Patterns

### 1. Complete Type Safety

```csharp
// Every parameter is type-safe
var customer = new Customer(
    customerId,
    email,
    phoneNumber,
    billingAddress,
    shippingAddress,
    creditLimit
);

// Compiler prevents any type confusion
// No way to pass OrderId where CustomerId expected
```

### 2. Self-Validation Everywhere

```csharp
// Value Objects validate themselves
var emailResult = Email.Create("invalid-email");
if (emailResult.IsFailure)
{
    // Cannot create invalid email
    return Result.Failure(emailResult.Error);
}

// StronglyTypedId validates creation
var id = CustomerId.Create(Guid.Empty); // Throws exception
```

### 3. Rich Domain Model

```csharp
// Domain concepts are explicit
public class Order
{
    public OrderId Id { get; }              // StronglyTypedId
    public CustomerId CustomerId { get; }      // StronglyTypedId
    public Address ShippingAddress { get; }    // Value Object
    public Email CustomerEmail { get; }          // Value Object
    public Money Total { get; }                // Value Object
}

// No ambiguity about what each property represents
```

### 4. Compiler Enforcement

```csharp
public interface IOrderRepository
{
    Order GetById(OrderId id);
    Customer GetCustomerById(CustomerId id);
}

// Cannot accidentally swap IDs
// var order = _repo.GetCustomerById(orderId); // Compiler error!
```

---

## Real-World Repository Example

```csharp
public interface ICustomerRepository
{
    Result<Customer> GetById(CustomerId id);
    Result<List<Customer>> GetAll();
    Result<Customer> Save(Customer customer);
}

public class CustomerRepository : ICustomerRepository
{
    private readonly DbContext _context;

    public Result<Customer> GetById(CustomerId id)
    {
        var customer = _context.Customers.Find(id);
        if (customer == null)
            return Result.Failure<Customer>("Customer not found");

        return Result.Success(customer);
    }

    public Result<Customer> Save(Customer customer)
    {
        _context.Customers.Add(customer);
        _context.SaveChanges();
        return Result.Success(customer);
    }
}

// Usage
var customerId = CustomerId.NewGuid();
var email = Email.Create("customer@example.com").Value;
var customer = new Customer(customerId, email, /* ... */);

var result = _repository.Save(customer);
```

---

## Testing Integrated Solutions

```csharp
[Test]
public void CreateCustomer_ValidData_ReturnsSuccess()
{
    var customerId = CustomerId.NewGuid();
    var email = Email.Create("test@example.com").Value;
    var phone = PhoneNumber.Create("+1-555-1234").Value;
    var address = Address.Create("123 Main St", "City", "CA", "90210", "USA").Value;
    var creditLimit = Money.Create(1000, Currency.USD).Value;

    var customer = new Customer(customerId, email, phone, address, address, creditLimit);

    Assert.AreEqual(customerId, customer.Id);
    Assert.AreEqual(email, customer.Email);
}

[Test]
public void CreateCustomer_InvalidEmail_ReturnsFailure()
{
    var customerId = CustomerId.NewGuid();
    var emailResult = Email.Create("invalid");

    Assert.IsTrue(emailResult.IsFailure);
}

[Test]
public void CannotMixIdTypes()
{
    var customerId = CustomerId.NewGuid();
    var orderId = OrderId.NewGuid();

    // These should never be equal
    Assert.AreNotEqual(customerId, orderId);
}
```

---

## Best Practices for Combined Patterns

1. **Use StronglyTypedId** for all entity identifiers
2. **Use Value Objects** for domain values
3. **Self-validate** at creation boundaries
4. **Factory methods** for controlled creation
5. **Result pattern** for operations that can fail
6. **Immutability** throughout
7. **Value semantics** for equality
8. **Test thoroughly** - each pattern independently and together

---

## Performance Considerations

StronglyTypedId and Value Objects have minimal overhead:

- **StronglyTypedId**: ~0.1μs overhead over Guid
- **Value Objects**: Immutable, no allocation on read
- **Equality**: Fast value comparison
- **Garbage collection**: Reduced due to immutability

**Benefits outweigh costs** for most applications.

---

## Complete Source Code

**Base Class:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.StronglyTypeId/blob/main/TestNest.StronglyTypeId/Common/StronglyTypedId.cs)

**Concrete IDs:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.StronglyTypeId/blob/main/TestNest.StronglyTypeId/StronglyTypeIds)

**Value Objects:**
[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.ValueObjects)

---

## Key Takeaways

- StronglyTypedId and Value Objects work together perfectly
- Combined, they create completely type-safe domains
- Each pattern provides different benefits
- Test each pattern independently and together
- Series complete! Next: Smart Enums

---

## References

- [Complete Source Code: TestNest.StronglyTypedId](https://github.com/DanteTuraSalvador/TestNest.StronglyTypeId)
- [Complete Source Code: TestNest.ValueObjects](https://github.com/DanteTuraSalvador/TestNest.ValueObjects)
- [DDD Reference: Patterns](https://www.domainlanguage.com/ddd/reference/)

{% include tutorial-nav.html %}
