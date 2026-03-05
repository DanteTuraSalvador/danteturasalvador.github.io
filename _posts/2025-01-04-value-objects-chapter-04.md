---
layout: post
title: "Chapter 4: Advanced - Composing Value Objects"
date: 2025-01-04
category: ddd
tags:
  - ddd
  - value-objects
  - advanced
series: value-objects
chapter: 4
prerequisites: "Chapter 3"
estimated_time: "25 minutes"
prev_title: "Chapter 3: Implementation - Your First Value Object"
prev_url: "/2025/01/03/value-objects-chapter-03.html"
---

# Chapter 4: Advanced - Composing Value Objects

## Learning Objectives

By the end of this chapter, you will understand:
- How to compose complex Value Objects
- Money and Currency implementation
- Price with standard/peak rates
- Real-world composition examples

---

## Composing Value Objects

Value Objects can be composed of other Value Objects to create rich domain concepts.

---

## Example 1: Address Value Object

```csharp
public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string State { get; }
    public string PostalCode { get; }
    public string Country { get; }

    public Address(string street, string city, string state,
                   string postalCode, string country)
    {
        Street = street;
        City = city;
        State = state;
        PostalCode = postalCode;
        Country = country;
    }

    public static Result<Address> Create(string street, string city,
                                         string state, string postalCode,
                                         string country)
    {
        if (string.IsNullOrWhiteSpace(street))
            return Result.Failure<Address>("Street cannot be empty");

        if (string.IsNullOrWhiteSpace(city))
            return Result.Failure<Address>("City cannot be empty");

        // More validations...

        return Result.Success(new Address(street, city, state,
                                          postalCode, country));
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return Street;
        yield return City;
        yield return State;
        yield return PostalCode;
        yield return Country;
    }
}
```

---

## Example 2: Money Value Object

```csharp
public class Money : ValueObject
{
    public decimal Amount { get; }
    public Currency Currency { get; }

    public Money(decimal amount, Currency currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Result<Money> Create(decimal amount, Currency currency)
    {
        if (amount < 0)
            return Result.Failure<Money>("Amount cannot be negative");

        return Result.Success(new Money(amount, currency));
    }

    // Arithmetic operations
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");

        return new Money(Amount + other.Amount, Currency);
    }

    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot subtract different currencies");

        return new Money(Amount - other.Amount, Currency);
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

### Usage

```csharp
var usd = Currency.USD;
var money1 = Money.Create(100, usd).Value;
var money2 = Money.Create(50, usd).Value;

var total = money1.Add(money2);
// Amount: 150, Currency: USD
```

---

## Example 3: Currency Value Object

```csharp
public class Currency : ValueObject
{
    public string Code { get; }
    public string Symbol { get; }
    public string Name { get; }

    private Currency(string code, string symbol, string name)
    {
        Code = code;
        Symbol = symbol;
        Name = name;
    }

    // Predefined currencies (Singleton pattern)
    public static Currency USD { get; } = new Currency("USD", "$", "US Dollar");
    public static Currency EUR { get; } = new Currency("EUR", "€", "Euro");
    public static Currency GBP { get; } = new Currency("GBP", "£", "British Pound");
    public static Currency JPY { get; } = new Currency("JPY", "¥", "Japanese Yen");
    public static Currency PHP { get; } = new Currency("PHP", "₱", "Philippine Peso");

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return Code;
    }
}
```

---

## Example 4: Price Value Object

```csharp
public class Price : ValueObject
{
    public Money StandardPrice { get; }
    public Money? PeakPrice { get; }

    public Price(Money standardPrice, Money? peakPrice = null)
    {
        StandardPrice = standardPrice;
        PeakPrice = peakPrice;
    }

    public bool IsPeak { get; private set; }

    public Money GetCurrentPrice()
    {
        return IsPeak && PeakPrice != null ? PeakPrice : StandardPrice;
    }

    public void SetPeak(bool isPeak)
    {
        IsPeak = isPeak;
    }

    public Money ApplyDiscount(decimal percentage)
    {
        var currentPrice = GetCurrentPrice();
        var discountAmount = currentPrice.Amount * (percentage / 100);
        return new Money(currentPrice.Amount - discountAmount,
                        currentPrice.Currency);
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return StandardPrice;
        yield return PeakPrice;
    }
}
```

### Usage

```csharp
var standardPrice = Money.Create(100, Currency.USD).Value;
var peakPrice = Money.Create(120, Currency.USD).Value;

var price = new Price(standardPrice, peakPrice);

price.SetPeak(true);
var currentPrice = price.GetCurrentPrice();
// Amount: 120, Currency: USD

var discounted = price.ApplyDiscount(10);
// Amount: 108 (10% off 120)
```

---

## Real-World Composition

```csharp
public class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public Address ShippingAddress { get; }    // Composed Value Object
    public Email CustomerEmail { get; }          // Value Object
    public PhoneNumber CustomerPhone { get; }   // Value Object
    public List<OrderItem> Items { get; }
    public Money Total { get; }                // Composed Value Object
    public Price FinalPrice { get; }           // Composed Value Object
}

public class OrderItem
{
    public ProductId ProductId { get; }
    public string ProductName { get; }
    public Quantity Quantity { get; }           // Value Object
    public Money UnitPrice { get; }             // Value Object
}
```

---

## Benefits of Composition

1. **Rich Domain Model** - Express complex concepts
2. **Type Safety** - Compiler catches errors
3. **Self-Validation** - Each VO validates itself
4. **Reusable** - Use across different entities
5. **Testable** - Easy to unit test

---

## Common Patterns

### 1. Single Property VO
```csharp
public class Email : ValueObject { /* ... */ }
```

### 2. Multi-Property VO
```csharp
public class Address : ValueObject { /* ... */ }
```

### 3. Composed VO
```csharp
public class Money : ValueObject
{
    public decimal Amount { get; }
    public Currency Currency { get; }  // Another VO
}
```

### 4. Behavior-Rich VO
```csharp
public class Price : ValueObject
{
    public Money GetCurrentPrice() { /* ... */ }
    public Money ApplyDiscount(decimal pct) { /* ... */ }
}
```

---

## Complete Source Code

[View on GitHub →](https://github.com/DanteTuraSalvador/TestNest.ValueObjects)

**Key Files:**
- [ValueObject.cs](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Domain/ValueObjects/Common/ValueObject.cs)
- [Email.cs](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Domain/ValueObjects/Email.cs)
- [Money.cs](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Domain/ValueObjects/Money.cs)
- [Currency.cs](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Domain/ValueObjects/Currency.cs)
- [Price.cs](https://github.com/DanteTuraSalvador/TestNest.ValueObjects/blob/main/TestNest.ValueObjects.Domain/ValueObjects/Price.cs)

---

## Key Takeaways

- Value Objects can be composed of other Value Objects
- Money, Currency, Price are powerful examples
- Composition creates rich domain models
- Test each VO independently
- Series complete! Next: Strongly Typed IDs

---

## References

- [Complete Source Code: TestNest.ValueObjects](https://github.com/DanteTuraSalvador/TestNest.ValueObjects)
- [DDD Reference: Value Objects](https://www.domainlanguage.com/ddd/reference/)

---

## What's Next?

Congratulations on completing Value Objects series!

Continue your DDD journey with:
- [Strongly Typed IDs](#) - Type-safe identifiers
- [Smart Enums](#) - State machines
- [Result Pattern](#) - Functional error handling
