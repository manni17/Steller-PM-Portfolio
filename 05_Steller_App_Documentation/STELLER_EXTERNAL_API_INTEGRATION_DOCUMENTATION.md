# Steller Platform - External API Integration Documentation (Bamboo Card Portal API)

## Overview

This document provides comprehensive documentation about the Steller platform's integration with the Bamboo Card Portal API, which serves as the external card fulfillment service for the platform.

## API Integration Architecture

### Core Integration Components

#### 1. ExternalApiService
- **Purpose**: Centralized HTTP client for all external API communications
- **Location**: `Steller.EF.Services.ExternalApiService` and `StellerConsumer.EF.Services.ExternalApiService`
- **Authentication**: HTTP Basic Authentication with client ID and client secret
- **Pattern**: Wrapper service with consistent error handling and response formatting

#### 2. RateLimiterService
- **Purpose**: Prevents exceeding Bamboo API rate limits
- **Location**: `Steller.Api.Services.RateLimiterService`
- **Strategy**: Sliding window rate limiting per API endpoint
- **Dynamic Adjustment**: Parses retry-after responses to adjust limits dynamically

#### 3. Message Queue Integration
- **Pattern**: Publisher-Consumer via RabbitMQ with MassTransit
- **Publisher**: StellerConsumer API publishes order messages
- **Consumer**: Steller API processes orders and calls Bamboo API
- **Decoupling**: Prevents synchronous blocking during external API calls

## API Endpoints and Rate Limits

### Bamboo API Endpoints
1. **Order Checkout**: `POST {BaseUrlV1}/orders/checkout`
   - **Rate Limit**: 2 requests per second (critical bottleneck)
   - **Purpose**: Create new card orders
   - **Authentication**: Basic Auth with client credentials

2. **Get Order**: `GET {BaseUrlV1}/orders/{orderId}`
   - **Rate Limit**: 120 requests per minute (2 req/sec average)
   - **Purpose**: Retrieve order status and details
   - **Authentication**: Basic Auth with client credentials

3. **Catalog Sync**: `GET {BaseUrlV1}/catalog`
   - **Rate Limit**: 2 requests per hour (extremely restrictive)
   - **Purpose**: Retrieve brand and product catalog
   - **Authentication**: Basic Auth with client credentials

4. **Categories Sync**: `GET {BaseUrlV2}/categories`
   - **Rate Limit**: Unknown (managed by rate limiter service)
   - **Purpose**: Retrieve category information
   - **Authentication**: Basic Auth with client credentials

### Rate Limiting Configuration
```csharp
// RateLimiterService.cs
_rateLimits["bamboo_place_order"] = (2, TimeSpan.FromSeconds(1)); // 2 requests per second
_rateLimits["bamboo_get_order"] = (120, TimeSpan.FromMinutes(1)); // 120 requests per minute  
_rateLimits["bamboo_catalog"] = (2, TimeSpan.FromHours(1)); // 2 requests per hour
_rateLimits["bamboo_exchange_rate"] = (20, TimeSpan.FromMinutes(1)); // 20 requests per minute
_rateLimits["bamboo_transactions"] = (4, TimeSpan.FromHours(1)); // 4 requests per hour
```

## Authentication and Authorization

### HTTP Basic Authentication
```csharp
// In ExternalApiService constructor
var authToken = Convert.ToBase64String(Encoding.UTF8.GetBytes($"{_clientId}:{_clientSecret}"));
_httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic", authToken);
```

### Authentication Details:
- **Credentials**: Loaded from environment variables (not hardcoded)
- **Encoding**: Base64 encoded before transmission
- **Transmission**: Sent in Authorization header for all requests
- **Security**: Should be used only over HTTPS connections

### Partner Authentication
- **API Key Header**: `X-Api-Key` for partner-specific operations
- **Partner Isolation**: Each request is scoped to specific partner's data
- **Database Validation**: API keys validated against database records

## Data Synchronization Patterns

### 1. Order Creation Flow (Publisher-Consumer Pattern)
```
[Consumer Dashboard] → [StellerConsumer.API] → [Save to DB] → [Publish OrderCreated msg] → [RabbitMQ] → [Steller.API] → [Call Bamboo API] → [Process Response] → [Update DB]
```

### 2. Catalog Synchronization (Background Service)
```
[BrandBackgroundService] → [Every 30 minutes] → [GET Bamboo Catalog] → [Save to DB] → [Update Local Cache]
```

### 3. Order Status Synchronization (On-Demand)
```
[UI Request] → [Steller.API] → [GET Bamboo Order] → [Parse Response] → [Update Local DB] → [Return to UI]
```

## Error Handling and Retry Mechanisms

### Rate Limiting Response Handling
```csharp
// Dynamic rate limit adjustment based on server response
if (result.StatusCode == HttpStatusCode.TooManyRequests && !string.IsNullOrEmpty(result.Message))
{
    var retrySeconds = ExtractRetrySeconds(result.Message);
    if (retrySeconds > 0)
    {
        _logger.LogWarning($"Rate limited by Bamboo API. Setting retry-after to {retrySeconds} seconds");
        _rateLimiter.SetRetryAfter("bamboo_place_order", retrySeconds);
    }
}
```

### Retry Pattern Implementation
1. **MassTransit Retry**: 3 retries with 30-second intervals for consumer service
2. **Rate Limiter Retry**: Dynamic adjustment based on HTTP 429 responses
3. **HTTP Client Retry**: Automatic retry for transient network failures
4. **Database Retry**: EF Core handles connection retries automatically

### Error Response Parsing
- **Retry Seconds Extraction**: Parses "wait for X seconds" from error messages
- **Status Code Handling**: Proper HTTP status code interpretation (429, 401, 500, etc.)
- **Message Parsing**: Attempts JSON error response parsing before falling back to raw response

## Security Considerations

### Data Protection
- **API Credentials**: Stored in environment variables, not in source code
- **HTTPS Transport**: All external communication uses HTTPS
- **Rate Limiting**: Prevents API abuse and protects against DoS attacks
- **Partner Isolation**: Each partner's data is isolated by PartnerId

### Security Vulnerabilities
- **Plain Text Credentials**: Environment files with credentials need better protection
- **TrustServerCertificate**: Database connection uses TrustServerCertificate=true (security risk)
- **API Key Storage**: API keys stored in database without apparent encryption
- **PIN Data**: Card PINs stored in database without explicit encryption

## Performance and Scalability

### Current Performance Characteristics
- **Order Throughput**: Limited to 2 orders per second due to Bamboo API limits
- **Catalog Updates**: Every 30 minutes due to 2 req/hour catalog limit
- **Concurrent Processing**: Max 5 concurrent message consumers to respect API limits
- **External Dependency**: System performance tightly coupled to Bamboo API performance

### Performance Limitations
- **Bamboo API Rate Limits**: Most significant performance bottleneck
- **No Caching**: All API calls hit Bamboo directly without caching
- **Synchronous Processing**: Order fulfillment blocked by slow external API calls
- **No Bulk Operations**: Individual API calls for each operation

## Integration Best Practices Applied

### ✅ Applied Best Practices
1. **Environment Configuration**: No hardcoded credentials in source code
2. **Rate Limiting**: Proactive approach to prevent rate limit breaches
3. **Async Operations**: Full async/await pattern throughout
4. **Error Handling**: Comprehensive error handling with graceful degradation
5. **Service Isolation**: Proper separation between internal and external services
6. **Structured Logging**: Serilog for comprehensive debugging and monitoring
7. **Message Queue**: Asynchronous processing decouples external API dependency

### 📉 Areas for Improvement
1. **Response Caching**: No caching of API responses (major optimization opportunity)
2. **Circuit Breaker**: No explicit circuit breaker implementation
3. **Monitoring**: Limited external API monitoring and alerting
4. **Webhook Support**: No real-time notifications from Bamboo API

## Data Models and Mapping

### External API Request Models
- **OrderDto**: Contains RequestId, AccountId, and Products list
- **ProductData**: Individual product with ProductId, Quantity, Value, CurrencyCode
- **OrderFromDataBaseDto**: Internal database model for processing

### External API Response Models  
- **OrderBambo**: Complete order information from Bamboo API
- **OrderItemBambo**: Individual order item details
- **OrderCardBambo**: Card-specific information (serial, code, PIN, expiration)
- **ApiErrorResponse**: Standardized error response format

### Data Transformation
- **AutoMapper**: Used for mapping between internal and external models
- **DTO Pattern**: Proper separation between external API models and internal models
- **Validation**: Input validation before external API calls
- **Transformation**: Data transformation to meet external API expectations

## Monitoring and Observability

### Logging Implementation
- **Serilog**: Structured logging with contextual information
- **Correlation IDs**: RequestId used for tracking across services
- **Performance Timing**: Request timing and response information
- **Error Tracking**: Comprehensive error information with stack traces

### Key Metrics to Monitor
1. **API Call Success/Failure Rates**: Track external API success rates
2. **Rate Limit Hit Count**: Monitor frequency of hitting rate limits
3. **Response Times**: External API response time tracking
4. **Message Processing Rates**: Track message queue processing throughput
5. **Order Processing Success Rates**: Track successful order fulfillment rates

## Security Recommendations

### Immediate Actions
1. **Implement Secrets Management**: Move from environment files to secure vault
2. **Strengthen JWT Keys**: Use high-entropy random keys for authentication
3. **Fix SSL Configuration**: Remove TrustServerCertificate=true
4. **Enable Transport Security**: Implement SSL/TLS for all internal communications

### Security Enhancements
1. **API Key Encryption**: Encrypt stored API keys in database
2. **Card Data Protection**: Implement proper encryption for PIN and card data
3. **Input Sanitization**: Add XSS and injection protection for all inputs
4. **Request Validation**: Implement more thorough request validation

## Performance Recommendations

### Caching Strategy
1. **Response Caching**: Cache Bamboo API responses with appropriate TTL
2. **Catalog Caching**: Cache brand/product catalogs with longer TTL
3. **Rate Limit Caching**: Cache rate limit status to avoid unnecessary delays
4. **Redis Integration**: Use distributed caching for multi-instance support

### Bulk Operations
1. **Batch Order Processing**: Implement batch order creation where Bamboo supports it
2. **Bulk Status Checks**: Check multiple order statuses in single operations
3. **Optimized Catalog Sync**: Reduce frequency of catalog updates
4. **Connection Optimization**: Configure optimal connection pool sizes

## Operational Considerations

### Deployment Strategies
1. **Blue-Green Deployment**: Maintain availability during deployments
2. **Gradual Rollouts**: Roll out changes gradually to protect external API
3. **Configuration Management**: Proper environment management for different stages
4. **Monitoring Integration**: Alerting for API failures and performance degradations

### Maintenance Procedures
1. **Regular Updates**: Monitor Bamboo API changes and updates
2. **Credential Rotation**: Periodic rotation of API credentials
3. **Rate Limit Monitoring**: Regular monitoring of rate limit usage
4. **Performance Tuning**: Continuous optimization based on usage patterns

## Future Enhancements

### Integration Improvements
1. **Webhook Support**: Implement webhook integration for real-time updates
2. **Event Streaming**: Consider event streaming for better real-time integration
3. **GraphQL API**: If Bamboo supports more advanced API patterns
4. **OpenAPI Specification**: Better documentation of external API contracts

### Architecture Enhancements
1. **API Gateway**: Consider adding API gateway for centralized management
2. **Circuit Breaker**: Implement circuit breaker for better resilience
3. **Retry Strategies**: More sophisticated retry patterns with exponential backoff
4. **Health Monitoring**: Real-time monitoring of external API availability

## Conclusion

The Steller platform shows a solid foundation for external API integration with proper rate limiting, error handling, and security measures. The event-driven architecture with RabbitMQ effectively decouples order acceptance from fulfillment, and the rate limiting implementation properly respects Bamboo API constraints.

However, significant opportunities exist for performance optimization through caching, improved error handling with circuit breakers, and enhanced security through proper secrets management. The system should be able to better handle the severe rate limits imposed by Bamboo API and provide better user experience during external API outages.