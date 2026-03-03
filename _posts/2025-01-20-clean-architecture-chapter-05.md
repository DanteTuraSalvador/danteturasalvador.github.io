---
layout: post
title: "Chapter 5: Integration and Deployment"
date: 2025-01-20
tags:
  - ddd
  - clean-architecture
  - advanced
series: clean-architecture
chapter: 5
prerequisites: "Chapter 4"
estimated_time: "25 minutes"
prev_title: "Chapter 4: Domain Layer Implementation"
prev: "/2025/01/19/clean-architecture-chapter-04.html"
---

# Chapter 5: Integration and Deployment

## Learning Objectives

By the end of this chapter, you will be able to:
- Integrate all layers in ASP.NET Core
- Configure dependency injection
- Set up Entity Framework Core
- Implement API endpoints
- Configure Swagger documentation
- Deploy to production

---

## Application Integration

### Service Registration

```csharp
namespace TestNest.CleanArchitecture.Application;

public static class ServiceRegistration
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddScoped<ICustomerRepository, CustomerRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IEmailService, EmailService>();
        services.AddScoped<IPaymentService, PaymentGatewayService>();

        // Application services
        services.AddScoped<ICustomerService, CustomerService>();
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IPricingDomainService, PricingDomainService>();

        // MediatR for CQRS
        services.AddMediatR(typeof(Domain.Assembly));
        services.AddMediatR(typeof(Application.Assembly));
    }
}
```

### Program.cs Setup

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add application services
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
app.UseSwagger();
app.UseHttpsRedirection();
app.Run();
```

---

## Infrastructure Integration

### DbContext Configuration

```csharp
namespace TestNest.CleanArchitecture.Infrastructure.Persistence;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configure relationships
        modelBuilder.Entity<Order>()
            .HasOne(o => o.Customer)
            .WithMany(o => o.Items)
            .HasForeignKey(o => o.CustomerId);

        // Configure value objects
        modelBuilder.Entity<Customer>()
            .OwnsOne(c => c.Email)
            .OwnsOne(c => c.Phone)
            .OwnsOne(c => c.Address);

        modelBuilder.Entity<Customer>()
            .Ignore(c => c.DomainEvents);

        modelBuilder.Entity<Order>()
            .OwnsOne(o => o.Items)
            .OwnsOne(o => o.DomainEvents);

        modelBuilder.Entity<OrderItem>()
            .OwnsOne(oi => oi.Order)
            .HasOne(oi => oi.Product);

        // Apply global query filters (soft delete)
        modelBuilder.Entity<Customer>()
            .HasQueryFilter(c => c.IsDeleted)
            .HasQueryFilter(c => c.IsArchived);

        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.IsDeleted)
            .HasQueryFilter(o => o.IsArchived);
    }
}
```

### Repository with EF Core

```csharp
public class CustomerRepository : ICustomerRepository
{
    private readonly AppDbContext _context;

    public CustomerRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Customer?> GetByIdAsync(CustomerId id)
    {
        return await _context.Customers
            .AsNoTracking()
            .FirstOrDefaultAsync(c => c.Id == id.Value);
    }

    public async Task<Customer?> GetByEmailAsync(string email)
    {
        return await _context.Customers
            .AsNoTracking()
            .FirstOrDefaultAsync(c => c.Email.Value == email);
    }

    public async Task<List<Customer>> GetAllAsync()
    {
        return await _context.Customers
            .AsNoTracking()
            .Where(c => !c.IsArchived)
            .OrderBy(c => c.CreatedAt)
            .ToListAsync();
    }

    public async Task<Customer> AddAsync(Customer customer)
    {
        await _context.Customers.AddAsync(customer);
        await _context.SaveChangesAsync();

        // Raise domain event
        // OnDomainEvent?.Invoke(new CustomerCreatedEvent(...));

        return customer;
    }

    public async Task<bool> ExistsByEmailAsync(string email)
    {
        return await _context.Customers
            .AsNoTracking()
            .AnyAsync(c => c.Email.Value == email);
    }
}
```

---

## Presentation Layer Integration

### Minimal API Endpoints

```csharp
namespace TestNest.CleanArchitecture.API.Endpoints;

public static class CustomerEndpoints
{
    public static IEndpointRouteBuilder MapCustomerEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/customers");

        group.MapPost("/", CreateCustomer)
             .WithName("CreateCustomer")
             .Accepts<CustomerCreateRequest>();

        group.MapGet("/{id:guid}", GetCustomer)
             .WithName("GetCustomer");

        group.MapGet("/", GetAllCustomers)
             .WithName("GetAllCustomers");

        group.MapPut("/{id:guid}", UpdateCustomer)
             .WithName("UpdateCustomer");

        return group;
    }

    private static async Task<IResult> CreateCustomer(ICustomerService service, CustomerCreateRequest request)
    {
        var command = new CreateCustomerCommand(request.Name, request.Email, request.Phone,
                                                      request.Street, request.City,
                                                      request.State, request.PostalCode,
                                                      request.Country);

        var result = await service.CreateCustomerAsync(command);

        return result.IsSuccess
            ? Results.Ok(new CustomerDto(result.Value))
            : Results.BadRequest(result.Error);
    }

    private static async Task<IResult> GetCustomer(ICustomerService service, Guid id)
    {
        var result = await service.GetByIdAsync(id);

        return result.IsSuccess
            ? Results.Ok(new CustomerDto(result.Value))
            : Results.NotFound(result.Error.Message);
    }
}
```

### Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class CustomerController : ControllerBase
{
    private readonly ICustomerService _service;

    public CustomerController(ICustomerService service)
    {
        _service = service;
    }

    // No controller logic - delegate to endpoints
    public CustomerController(IEndpointRouteBuilder app)
    {
        app.MapCustomerEndpoints(_service);
    }

    public IActionResult Options()
    {
        return NoContent();
    }
}
```

---

## DTOs and Mapping

### DTO Definition

```csharp
namespace TestNest.CleanArchitecture.Application.DTOs;

public record CustomerDto
{
    public Guid Id { get; }
    public string Name { get; init; }
    public string Email { get; init; }
    public string Phone { get; init; }
    public AddressDto Address { get; init; }
    public DateTime CreatedAt { get; init; }
    public Money CreditLimit { get; init; }
}

public record CustomerCreateRequest
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
```

### Mapper Implementation

```csharp
namespace TestNest.CleanArchitecture.Application.Mappers;

public static class CustomerMapper
{
    public static CustomerDto ToDto(Customer customer)
    {
        return new CustomerDto
        {
            Id = customer.Id.Value,
            Name = customer.Name,
            Email = customer.Email.Value,
            Phone = customer.Phone.Value,
            Address = AddressMapper.ToDto(customer.Address),
            CreatedAt = customer.CreatedAt.ToDateTime(),
            CreditLimit = customer.CreditLimit
        };
    }

    public static Customer ToDomain(CustomerDto dto)
    {
        return new Customer(
            new CustomerId(dto.Id),
            dto.Name,
            Email.Create(dto.Email).Value,
            PhoneNumber.Create(dto.Phone).Value,
            AddressMapper.ToDomain(dto.Address),
            DateOnly.FromDateTime(dto.CreatedAt),
            dto.CreditLimit
        );
    }
}
```

---

## Cross-Cutting Concerns Setup

### Global Exception Handler

```csharp
public class GlobalExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        _logger.LogError(exception, "An unhandled exception occurred");

        var response = context.Response;
        response.ContentType = "application/problem+json";

        var problem = new ProblemDetails
        {
            Type = exception.GetType().Name,
            Title = "An error occurred",
            Status = (int)HttpStatusCode.InternalServerError,
            Detail = exception.Message,
            Instance = context.Request.Path
        };

        await response.WriteAsJsonAsync(problem);
    }
}

public record ProblemDetails
{
    public string Type { get; init; }
    public string Title { get; init; }
    public int Status { get; init; }
    public string Detail { get; init; }
    public string Instance { get; init; }
}
```

### Logging Configuration

```csharp
public static class LoggingConfiguration
{
    public static void Configure(string connectionString)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .Enrich.FromProperty("Application")
            .Enrich.FromProperty("MachineName", Environment.MachineName)
            .WriteTo.File(connectionString)
            .WriteTo.Seq("Application")
            .CreateLogger();
    }
}
```

---

## Deployment Configuration

### appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=CleanArchitectureDb;Trusted_Connection=True;",
    "ProductionConnection": "Server=tcp:production-server;Database=CleanArchitectureDb;User Id=cleanarch;Password=YourPassword;"
  },
  "Serilog": {
    "MinimumLevel": "Information",
    "WriteTo": {
      "Name": "CleanArchitecture",
      "Args": { "Serilog:Path=C:\\Logs\\clean-architecture-.log",
              "OutputTemplateTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Properties}{NewLine}{Exception}" }
    },
    "AllowedHosts": "development,production"
  },
  "Jwt": {
    "Secret": "Your-Super-Secret-Key",
    "Issuer": "CleanArchitecture",
    "Audience": "CleanArchitecture.Users"
  }
}
```

### Docker Configuration

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY ["TestNest.CleanArchitecture.API/TestNest.CleanArchitecture.API.csproj"]
COPY ["TestNest.CleanArchitecture.Domain/TestNest.CleanArchitecture.Domain.csproj"]
COPY ["TestNest.CleanArchitecture.Application/TestNest.CleanArchitecture.Application.csproj"]
COPY ["TestNest.CleanArchitecture.Infrastructure/TestNest.CleanArchitecture.Infrastructure.csproj"]
COPY ["TestNest.CleanArchitecture.Domain/Entities/TestNest.CleanArchitecture.Domain.Entities.csproj"]
COPY ["TestNest.CleanArchitecture.Domain/TestNest.CleanArchitecture.Domain.csproj"]
COPY ["TestNest.CleanArchitecture.Domain/ValueObjects/TestNest.CleanArchitecture.Domain.ValueObjects.csproj"]
COPY ["TestNest.CleanArchitecture.Domain/Interfaces/TestNest.CleanArchitecture.Domain.Interfaces.csproj"]
COPY ["TestNest.CleanArchitecture.Domain/Services/TestNest.CleanArchitecture.Domain.Services.csproj"]

RUN dotnet restore
RUN dotnet build TestNest.CleanArchitecture.Domain
RUN dotnet build TestNest.CleanArchitecture.Infrastructure
RUN dotnet build TestNest.CleanArchitecture.Application

ENV ASPNETCORE_URLS=http://+:80
ENV ASPNETCORE_ENVIRONMENT=Development

EXPOSE 80
ENTRYPOINT ["dotnet", "TestNest.CleanArchitecture.API.dll"]
```

---

## Testing the Integration

### Integration Tests

```csharp
[CollectionDefinition]
public class IntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private readonly WebApplicationFactory<Program> _factory;

    public IntegrationTests(WebApplicationFactory<Program> factory, HttpClient client)
    {
        _client = client;
        _factory = factory;
    }

    [Fact]
    public async Task CreateCustomer_ShouldPersistAndReturnDto()
    {
        // Arrange
        var request = new CustomerCreateRequest
        {
            Name = "John Doe",
            Email = "john@example.com",
            Phone = "555-1234",
            Street = "123 Main St",
            City = "Springfield",
            State: "IL",
            PostalCode: "62701",
            Country: "USA"
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/customers", request);

        // Assert
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        Assert.True(response.IsSuccessStatusCode);

        var dto = await response.Content.ReadFromJsonAsync<CustomerDto>();
        Assert.Equal("John Doe", dto.Name);
        Assert.Equal("john@example.com", dto.Email);
    }

    [Fact]
    public async Task GetCustomer_ShouldReturn404ForNonExistent()
    {
        // Arrange
        var nonExistentId = Guid.NewGuid();

        // Act
        var response = await _client.GetAsync($"/api/customers/{nonExistentId}");

        // Assert
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

---

## Real-World Production Considerations

### Performance Optimization

```csharp
// Caching frequently accessed data
public class CachedCustomerRepository : ICustomerRepository
{
    private readonly ICustomerRepository _repository;
    private readonly ICacheService _cache;

    public CachedCustomerRepository(ICustomerRepository repository, ICacheService cache)
    {
        _repository = repository;
        _cache = cache;
    }

    public async Task<Customer?> GetByIdAsync(CustomerId id)
    {
        var cacheKey = $"customer:{id}";

        // Try cache first
        var cached = await _cache.GetAsync<Customer>(cacheKey);
        if (cached != null)
            return cached;

        // Not in cache, fetch from database
        var customer = await _repository.GetByIdAsync(id);

        // Cache for next time
        await _cache.SetAsync(cacheKey, customer, TimeSpan.FromMinutes(5));

        return customer;
    }
}
```

### Monitoring and Observability

```csharp
// Application Insights
public class ApplicationInsightsMiddleware
{
    private readonly ILogger<ApplicationInsightsMiddleware> _logger;

    public ApplicationInsightsMiddleware(ILogger<ApplicationInsightsMiddleware> logger)
    {
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // Log request details
        var watch = Stopwatch.StartNew();
        var response = await next(context);

        watch.Stop();

        _logger.LogInformation(
            "Request: {Method} {Path} completed in {ElapsedMilliseconds}ms",
            context.Request.Method,
            context.Request.Path,
            response.StatusCode
        );
    }
}
```

### Security Configuration

```csharp
// JWT Authentication
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true
        };

        options.TokenValidationParameters.IssuerSigningKey = new SymmetricSecurityKey(
            configuration["Jwt:Secret"],
            SecurityAlgorithms.HmacSha256);

        options.TokenValidationParameters.Audience = configuration["Jwt:Audience"];
        options.TokenValidationParameters.Expiration = TimeSpan.FromDays(7);
    });

// Authorization Policy
services.AddAuthorization(options =>
{
    options.AddPolicy("CustomerPolicy", policy =>
        policy.RequireRole("Customer")
        .RequireClaim(ClaimTypes.Role, "CustomerRole")
    );
});
```

---

## Key Takeaways

- Integrate all layers through dependency injection
- Configure Entity Framework Core for data access
- Set up Minimal API with clean endpoints
- Configure Swagger for API documentation
- Implement cross-cutting concerns (logging, caching)
- Use DTOs for layer boundaries
- Test integration between layers
- Configure deployment options
- Series complete! Next: CQRS

---

## References

- [Complete Source Code: MinimalAPICleanArchitecture](https://github.com/DanteTuraSalvador/MinimalAPICleanArchitecture)
- [Docker: Containerized deployment](https://docs.docker.com/engine/examples/dotnetcore/)
- [MediatR: Mediator pattern implementation](https://github.com/jbogard/MediatR)
- [OWASP API Security: Secure APIs](https://github.com/OWASP/ASP.NET)

{% include tutorial-nav.html %}
