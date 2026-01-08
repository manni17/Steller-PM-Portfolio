# Steller Backend Integration Service Documentation

## Overview

The Steller Backend Integration Service is a .NET 9.0 Web API that:
- Consumes messages from RabbitMQ queue
- Processes orders by calling external Bamboo Card Portal API
- Updates local database with external API responses
- Implements sophisticated rate limiting for external API calls
- Provides administrative functions for system oversight

**Note**: For complete API endpoint documentation, please refer to the [STELLER_COMPREHENSIVE_API_DOCUMENTATION.md](STELLER_COMPREHENSIVE_API_DOCUMENTATION.md) file.

## Architecture and Structure

### Project Structure
```
Steller/
├── Steller.Api/      # Main API application (consumer)
├── Steller.Core/     # Business logic, models, DTOs
├── Steller.EF/       # Data access layer and repositories
└── SharedContracts/  # Shared message contracts
```

### Key Architecture Patterns
1. **Event-Driven Architecture**: Consumer processes messages from RabbitMQ
2. **Repository Pattern**: Generic repository implementation for data access
3. **Service Layer**: Business logic separated from controllers
4. **Dependency Injection**: All services managed by built-in DI container
5. **External API Integration**: Calls to Bamboo Card Portal API with rate limiting

## Message Consumption Implementation

### MassTransit Configuration
The API implements a consumer-only architecture:

```csharp
// Program.cs - MassTransit Configuration
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();

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

        cfg.ReceiveEndpoint("order-created", e =>
        {
            e.ConfigureConsumer<OrderCreatedConsumer>(context);

            // Add retry policy for handling transient failures
            e.UseMessageRetry(r => r.Interval(3, TimeSpan.FromSeconds(30))); // Retry 3 times with 30 second intervals

            // Configure concurrency to prevent overwhelming the rate limiter
            e.ConcurrentMessageLimit = 5; // Process max 5 messages concurrently
        });
    });
});
```

### OrderCreated Consumer Implementation
```csharp
public class OrderCreatedConsumer : IConsumer<OrderCreated>
{
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        var orderId = context.Message.OrderId;
        _logger.LogInformation($"Received Order: {orderId}");

        try
        {
            OrderFromDataBaseDto orderFromDataBaseDto = new OrderFromDataBaseDto()
            {
                RequestId = orderId,
                AccountId = 1256, // TODO: This should be dynamic based on the order
            };

            var orderResult = await _orderService.AddOrder(orderFromDataBaseDto);

            if (orderResult.Status)
            {
                _logger.LogInformation($"Successfully processed order {orderId}. Bamboo order id: {orderResult.Data}");
            }
            else
            {
                _logger.LogError($"Failed to process order {orderId}. Error: {orderResult.Message}, ErrorCode: {orderResult.ErrorCode}");

                // Update the order status to indicate processing failure
                // This prevents infinite re-queuing of failed orders
                await UpdateOrderStatusToFailed(orderId, orderResult.Message ?? "Unknown error");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Exception occurred while processing order {orderId}");

            // Update the order status to indicate processing failure
            await UpdateOrderStatusToFailed(orderId, $"Exception: {ex.Message}");

            // Re-throw to let MassTransit handle retry logic if configured
            throw;
        }
    }
}
```

### Message Contract
```csharp
// In SharedContracts
public record OrderCreated(Guid OrderId);
```

## External API Integration

### Bamboo Card Portal API Integration
The Steller API integrates with the Bamboo Card Portal API for:
- **Order Creation**: POST to `/orders/checkout` endpoint
- **Order Retrieval**: GET from `/orders/{orderId}` endpoint
- **Brand and Category Sync**: Background sync for product catalogs
- **Currency Exchange Rates**: Retrieval of exchange rates

### External API Service
```csharp
// External API Authentication with Basic Auth
public ExternalApiService(HttpClient httpClient, IConfiguration configuration)
{
    _httpClient = httpClient;
    _clientId = configuration["ExternalApi:MyClientId"]!;
    _clientSecret = configuration["ExternalApi:MyClientSecret"]!;

    // Add Basic Authentication header
    _authToken = Convert.ToBase64String(Encoding.UTF8.GetBytes($"{_clientId}:{_clientSecret}"));
    _httpClient.DefaultRequestHeaders.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue("Basic", _authToken);
}
```

## Rate Limiting Implementation

### Sophisticated Rate Limiting System
The system implements advanced rate limiting for Bamboo API endpoints:

```csharp
public RateLimiterService()
{
    // Configure rate limits based on Bamboo API documentation
    _rateLimits["bamboo_place_order"] = (2, TimeSpan.FromSeconds(1)); // 2 requests per second
    _rateLimits["bamboo_get_order"] = (120, TimeSpan.FromMinutes(1)); // 120 requests per minute
    _rateLimits["bamboo_catalog"] = (2, TimeSpan.FromHours(1)); // 2 requests per hour
    _rateLimits["bamboo_exchange_rate"] = (20, TimeSpan.FromMinutes(1)); // 20 requests per minute
    _rateLimits["bamboo_transactions"] = (4, TimeSpan.FromHours(1)); // 4 requests per hour
    _rateLimits["bamboo_other"] = (1, TimeSpan.FromSeconds(1)); // 1 request per second for other APIs
}
```

### Dynamic Rate Adjustments
- **Retry-After Support**: Adjusts rate limits based on API responses
- **Sliding Window Algorithm**: Implements proper rate limiting
- **Thread-Safe Operations**: Uses ConcurrentDictionary for safety

## Service Layer Implementation

### Core Services
1. **OrderService**: Primary business logic for order processing
   - External API integration with Bamboo Card Portal
   - Rate limiting integration for API calls
   - Database synchronization with external responses
   - Error handling and failure management

2. **ExternalApiService**: HTTP client wrapper
   - Basic Authentication implementation
   - Comprehensive error handling
   - JSON serialization/deserialization

3. **RateLimiterService**: Rate limiting implementation
   - Multiple endpoint-specific limits
   - Dynamic adjustment based on responses
   - Thread-safe operations

4. **BrandBackgroundService**: Background synchronization
   - Periodic brand and category updates
   - Runs as hosted service

### Service Architecture
- **Dependency Injection**: All services registered with appropriate lifetimes
- **Repository Pattern**: Services use repositories for data access
- **External API Integration**: Proper integration with rate limiting
- **Error Handling**: Comprehensive error handling with status codes

## Data Access Layer

### Repository Pattern
Same pattern as StellerConsumer:
- **CRUD Operations**: Full set of Create, Read, Update, Delete operations
- **Query Support**: Expression-based querying with Include/ThenInclude
- **Bulk Operations**: Support for AddRange, DeleteRange operations

### Data Synchronization
- **Two-Way Sync**: Local database synchronized with external API
- **Status Tracking**: Local status updated from external API responses
- **Error Handling**: ErrorMessage field for tracking external API errors

## Security and Authentication

### Internal API Security
- **JWT Authentication**: Standard JWT Bearer token validation
- **Proper Configuration**: Token validation parameters set correctly

For complete authentication details and security considerations, refer to [STELLER_COMPREHENSIVE_API_DOCUMENTATION.md](STELLER_COMPREHENSIVE_API_DOCUMENTATION.md).

### External API Security
- **Basic Authentication**: Client ID/Secret for Bamboo API
- **Secure Storage**: Credentials managed via environment variables
- **Header Management**: Authentication headers set per request

### Security Considerations
- **CORS Policy**: Currently allows any origin (may need restriction)
- **Sensitive Data**: PINs and card codes stored in database (needs encryption)
- **Credential Management**: Environment-based secure configuration

## Performance and Scalability

### Message Processing Performance
- **Asynchronous Processing**: Non-blocking message consumption
- **Concurrency Control**: Limited to 5 concurrent consumers to respect external limits
- **MassTransit Integration**: Efficient message processing with connection management

### Rate Limiting Benefits
- **API Compliance**: Respects external API rate limits
- **Dynamic Adjustment**: Adjusts based on external API responses
- **Performance Optimization**: Prevents API throttling

### Scalability Features
- **Horizontal Scaling**: Multiple consumer instances can process messages
- **Event-Driven Architecture**: Decoupled processing allows independent scaling
- **Connection Pooling**: Efficient resource management

## Error Handling and Resilience

### Comprehensive Error Handling
- **External API Errors**: Proper HTTP status code mapping
- **Rate Limiting**: Dynamic adjustment based on API responses
- **Message Processing**: Prevents infinite re-queuing of failed messages
- **Database Sync**: Handles external API failures gracefully

### Retry and Recovery
- **MassTransit Retries**: Built-in retry mechanism for transient failures
- **Error Tracking**: Failed orders marked to prevent infinite retries
- **Logging**: Comprehensive logging for debugging

## Response Format
All API responses follow the standard format as detailed in [STELLER_COMPREHENSIVE_API_DOCUMENTATION.md](STELLER_COMPREHENSIVE_API_DOCUMENTATION.md).

## Operational Considerations

### Deployment
- **Dockerization**: Container-ready with Dockerfile
- **Docker Compose**: Multi-container orchestration
- **Environment Configuration**: Flexible deployment environments

### Monitoring and Logging
- **Structured Logging**: Serilog for detailed logging
- **Performance Monitoring**: Available through standard metrics
- **Error Tracking**: Detailed error information with context

### Security Hardening
- **CORS Configuration**: Currently allows any origin (needs restriction)
- **Authentication**: Dual approach (internal JWT, external Basic Auth)
- **Data Protection**: Sensitive data encryption needed
- **Credentials Management**: Environment-based secure storage

## Integration Patterns
- Asynchronous message processing
- Circuit breaker pattern for external API calls
- Bulk data synchronization for catalogs
- Event-driven architecture principles