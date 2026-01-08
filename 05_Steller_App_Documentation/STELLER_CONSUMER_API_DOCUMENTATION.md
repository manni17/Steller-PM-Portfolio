# StellerConsumer API Documentation

This document provides comprehensive information about the StellerConsumer API, which serves as the order entry point in the Steller platform. The API implements an event-driven architecture that decouples order acceptance from fulfillment processes.

## Overview

The StellerConsumer API is a .NET 9.0 Web API that:
- Accepts orders from clients/partners
- Performs validation and business logic
- Stores orders in the database
- Publishes messages to RabbitMQ for asynchronous processing
- Implements multi-tenant security with partner isolation

### Technology Stack
- **Framework**: ASP.NET Core 9.0
- **ORM**: Entity Framework Core with PostgreSQL
- **Messaging**: MassTransit with RabbitMQ transport
- **Authentication**: JWT Bearer tokens and API keys
- **Database**: PostgreSQL
- **Architecture**: Event-driven with Publisher-Consumer pattern

## Architecture and Structure

### Project Structure
```
StellerConsumer/
├── StellerConsumer.Api/      # Main API application
├── StellerConsumer.Core/     # Business logic, models, DTOs
├── StellerConsumer.EF/       # Data access layer and repositories
└── SharedContracts/          # Shared message contracts
```

### Key Architecture Patterns
1. **Event-Driven Architecture**: Orders published asynchronously via RabbitMQ
2. **Repository Pattern**: Generic repository implementation for data access
3. **Service Layer**: Business logic separated from controllers
4. **Dependency Injection**: All services managed by built-in DI container
5. **Separation of Concerns**: Clear separation between API, Core, and Data layers

## API Endpoints

**Note**: Complete API endpoint documentation with request/response examples is available in [STELLER_COMPREHENSIVE_API_DOCUMENTATION.md](STELLER_COMPREHENSIVE_API_DOCUMENTATION.md). This document focuses on architecture and implementation details.

## Message Publishing Implementation

### RabbitMQ Configuration
The API implements a publisher-only architecture:

```csharp
// Program.cs - MassTransit Configuration
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        var rabbitMqConfig = builder.Configuration.GetSection("RabbitMQ");
        var host = rabbitMqConfig["Host"] ?? "localhost";
        var username = rabbitMqConfig["Username"] ?? "guest";
        var password = rabbitMqConfig["Password"] ?? "guest";

        cfg.Host(host, h =>
        {
            h.Username(username);
            h.Password(password);
        });
    });
});
```

### Message Publishing in OrderController
```csharp
[HttpPost]
public async Task<IActionResult> PlaceOrder(OrderDto orderDto)
{
    // ... validation code ...
    
    var orderResult = await _orderService.AddOrder(orderDto, partenerId);
    
    if (!orderResult.Status)
    {
        return BadRequest(orderResult);
    }

    var orderCreated = new OrderCreated(orderResult.Data!.RequestId);
    await _publishEndpoint.Publish(orderCreated);  // Publish to RabbitMQ
    return Ok(orderResult);
}
```

### Message Contract
```csharp
// In SharedContracts
public record OrderCreated(Guid OrderId);
```

## Service Layer Implementation

### Core Services
1. **OrderService**: Main business logic for order processing
   - Handles order validation and creation
   - Manages currency conversion
   - Integrates with wallet service
   - Links orders to partners

2. **HelperService**: Authentication and context utility
   - Extracts partner ID from API key or JWT
   - Provides cross-cutting concerns

3. **WalletService**: Financial transaction management
   - Handles fund withdrawal for orders
   - Manages wallet balances

4. **CurrencyService**: Exchange rate management
   - Handles currency conversion for orders

### Service Architecture
- **Dependency Injection**: All services registered with appropriate lifetimes
- **Repository Pattern**: Services use repositories for data access
- **AutoMapper**: Used for DTO to entity transformations
- **Entity Framework Core**: Primary data access mechanism

## Data Access Layer

### Repository Pattern
The `BaseRepository<T>` provides:
- CRUD operations with full async support
- Querying with expression support and inclusion
- Paging and sorting capabilities
- Bulk operations (AddRange, DeleteRange)

### Data Models
- **Base Classes**: Models inherit from BaseEntity, BaseEntityGuid, etc.
- **Navigation Properties**: Proper relationships with collections and references
- **Precision**: Decimal(18,4) for financial data
- **JSON Handling**: Attributes to prevent circular reference serialization

## Security and Authentication

### Authentication Methods
1. **JWT Authentication**: Standard JWT Bearer token validation
2. **API Key Authentication**: Custom API key header ("X-Api-Key") validation

For complete authentication details and security considerations, refer to [STELLER_COMPREHENSIVE_API_DOCUMENTATION.md](STELLER_COMPREHENSIVE_API_DOCUMENTATION.md).

### Security Flow
```csharp
public int GetPartnerId()
{
    string headerValue = GetHeaderValue(API_KEY_HEADER_NAME);

    if (!string.IsNullOrWhiteSpace(headerValue))
    {
        var partnerIdFromKey = _apiKeyRepository.GetPartnerId(headerValue);
        if (partnerIdFromKey != 0)
        {
            return partnerIdFromKey;
        }
    }

    // Fallback to JWT claim "partner"
    var partnerClaim = _httpContextAccessor.HttpContext?.User?.FindFirst("partner")?.Value;
    if (int.TryParse(partnerClaim, out var partnerIdFromJwt))
    {
        return partnerIdFromJwt;
    }

    return 0;
}
```

### Security Features
- **Dual Authentication**: Supports both API key and JWT token
- **Partner Isolation**: Each request is associated with a specific partner
- **Token Validation**: JWT tokens validated with standard security parameters
- **API Key Validation**: Verifies API key exists and is not revoked

## Business Logic

### Order Processing Flow
1. **Validation**: Order data validated through model binding and custom checks
2. **Currency Conversion**: Automatic conversion based on brand currencies
3. **Database Persistence**: Order saved to PostgreSQL database
4. **Wallet Deduction**: Funds deducted from partner's wallet
5. **Message Publishing**: OrderCreated message published to RabbitMQ
6. **Response**: Result returned to client

### Currency Handling
- **Multi-Currency Support**: Orders can be made in different currencies
- **Automatic Conversion**: Real-time conversion using exchange rates
- **Brand Currency**: Each brand has its own currency setting

## Performance and Scalability

### Architecture-Level Scalability
- **Event-Driven Design**: Asynchronous processing allows independent scaling
- **Publisher-Consumer Pattern**: Order acceptance and processing scale independently
- **Horizontal Scaling**: Multiple instances can run behind load balancer

### Performance Features
- **Connection Pooling**: EF Core and MassTransit manage connection pools
- **AutoMapper**: Efficient object-to-object mapping
- **Asynchronous Operations**: Full async/await pattern throughout

### Resource Management
- **Dependency Injection**: Proper lifetime management
- **HTTP Context Accessor**: Request-scoped access to context
- **Efficient Queries**: Use of Include/ThenInclude for related data

## External API Considerations

### Integration Architecture
The StellerConsumer API itself does NOT make direct external API calls. Instead:
- **StellerConsumer API**: Accepts orders, stores in DB, publishes to RabbitMQ
- **Steller API**: Consumes from RabbitMQ, makes external Bamboo Card Portal API calls
- This decouples order acceptance from external API processing

### Configuration Settings
```csharp
// External API configuration (used by Steller API, not StellerConsumer)
ExternalApi:BaseUrlV1 = Environment.GetEnvironmentVariable("EXTERNAL_API_BASE_URL_V1")
ExternalApi:BaseUrlV2 = Environment.GetEnvironmentVariable("EXTERNAL_API_BASE_URL_V2")
ExternalApi:MyClientId = Environment.GetEnvironmentVariable("EXTERNAL_API_CLIENT_ID")
ExternalApi:MyClientSecret = Environment.GetEnvironmentVariable("EXTERNAL_API_CLIENT_SECRET")
```

## Configuration and Environment

### Environment Variables
The application uses comprehensive environment-based configuration:
- **Database Settings**: Host, port, name, username, password
- **RabbitMQ Settings**: Host, username, password
- **JWT Settings**: Key, issuer, audience
- **External API Settings**: Base URLs and credentials
- **Email Settings**: SMTP configuration

### Connection String Construction
```csharp
var connectionString = $"Host={dbHost};Port={dbPort};Database={dbName};Username={dbUsername};Password={dbPassword};TrustServerCertificate=true;Command Timeout=60;Timeout=30";
```

## Best Practices

### Code Quality
- **Separation of Concerns**: Clear separation between API, Core, and Data layers
- **Dependency Injection**: Proper use of built-in DI container
- **Async/Await**: Consistent use of asynchronous patterns
- **Error Handling**: Consistent error handling with ServiceResponse wrapper following the standard response format (see [STELLER_COMPREHENSIVE_API_DOCUMENTATION.md](STELLER_COMPREHENSIVE_API_DOCUMENTATION.md) for details)

## Response Format
All API responses follow the standard format as detailed in [STELLER_COMPREHENSIVE_API_DOCUMENTATION.md](STELLER_COMPREHENSIVE_API_DOCUMENTATION.md).

### Security
- **Input Validation**: Comprehensive validation in controllers
- **Authentication**: Dual approach with API keys and JWT
- **Partner Isolation**: Proper multi-tenant security
- **Secure Configuration**: Environment-based secrets management

### Performance
- **Asynchronous Operations**: Non-blocking operations throughout
- **Connection Pooling**: Efficient resource management
- **Efficient Queries**: Proper use of Include/ThenInclude
- **Object Mapping**: AutoMapper for efficient transformations

### Scalability
- **Event-Driven Architecture**: Decoupled processing
- **Horizontal Scaling**: Multiple instance support
- **Message Reliability**: Persistent messaging with RabbitMQ
- **Database Optimization**: Proper indexing and query optimization

## Operational Considerations

### Deployment
- **Dockerization**: Container-ready with Dockerfile
- **Docker Compose**: Multi-container orchestration
- **Environment Configuration**: Flexible deployment environments
- **Health Checks**: Built-in health check infrastructure

### Monitoring and Logging
- **Structured Logging**: Serilog for structured logging
- **Request Logging**: Comprehensive request tracking
- **Error Tracking**: Detailed error information
- **Performance Monitoring**: Available through standard metrics

### Security Hardening
- **CORS Configuration**: Currently allows any origin (may need restriction)
- **Authentication**: Dual authentication approach
- **Data Validation**: Comprehensive input validation
- **Credentials Management**: Environment-based credential storage