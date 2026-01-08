# Steller Platform - Background Services Documentation

## Overview

The Steller platform includes two critical background services that operate continuously to maintain data synchronization and processing. These services are the backbone of the platform's reliability and ensure consistent data flow between internal systems and external APIs.

## BrandBackgroundService

### Purpose
The BrandBackgroundService serves as the synchronization engine that keeps the internal brand and category data in sync with the external Bamboo Card Portal API. This ensures that the latest product catalog information is always available within the system.

### Architecture
- **Implementation**: Inherits from .NET `BackgroundService` 
- **Hosting**: Registered as `IHostedService` via `AddHostedService<BrandBackgroundService>()`
- **Execution Model**: Continuous execution with scheduled intervals
- **Lifecycle**: Respects `CancellationToken` for graceful shutdown

### Execution Schedule
- **Frequency**: Every 30 minutes
- **Reasoning**: Aligns with Bamboo API rate limits for catalog endpoint (2 requests per hour)
- **Execution Pattern**: Runs both categories and brands sync in each cycle

### Core Operations

#### Categories Synchronization (`FetchCategoriesFromBambo`)
- **Endpoint**: `{BaseUrlV2}/categories`
- **Rate Limiting**: Uses `bamboo_categories` limiter
- **Processing**:
  - Fetches all categories from external API
  - Creates/updates categories in internal database
  - Uses system user ID (1) for background operations
- **Error Handling**: Individual try-catch with continued execution

#### Brands Synchronization (`FetchBrandsFromBambo`)
- **Endpoint**: `{BaseUrlV1}/catalog`
- **Rate Limiting**: Uses `bamboo_catalog` limiter (2 req/hour)
- **Processing**:
  - Fetches complete catalog from external API
  - Updates existing brands or creates new ones
  - Synchronizes associated products, pricing, and configurations
  - Links brands to categories
- **Error Handling**: Individual try-catch with continued execution

### Error Handling Strategy
1. **Top-Level Protection**: Service continues running despite operation failures
2. **Granular Handling**: Each sync operation (categories vs brands) handled separately
3. **Rate Limit Adaptation**: Parses "wait for X seconds" messages to adjust rate limits dynamically
4. **Continued Execution**: If categories fail, brands sync still proceeds
5. **Comprehensive Logging**: Detailed error information for troubleshooting

### Data Synchronization Pattern
- **Upsert Logic**: Creates new records or updates existing ones based on InternalId
- **Full Catalog Refresh**: Processes entire catalog on each execution
- **Relationship Preservation**: Maintains links between brands, products, categories, and pricing
- **Incremental Updates**: Only processes changes compared to stored data

### Performance Considerations
- **Batch Processing**: Processes all data in single operations
- **Bulk Database Operations**: Uses EF Core's AddRange/UpdateRange
- **Efficient Lookup**: Uses Dictionary for O(1) matching operations
- **Memory Management**: Clears lists between executions to prevent memory growth
- **Connection Efficiency**: Leverages EF Core connection pooling

### Dependencies
- `IBrandServiceFactory`: For creating brand service instances with proper scoping
- `ExternalApiService`: For making external API calls with authentication
- `RateLimiterService`: For respecting API rate limits
- `ILogger`: For comprehensive logging
- `IServiceProvider`: For creating scoped services during execution

## OrderQueueBackgroundService

### Purpose
The OrderQueueBackgroundService ensures that orders which were accepted but may not have been immediately processed are eventually queued for fulfillment. This provides a critical safety net in the event-driven architecture.

### Architecture
- **Implementation**: Inherits from .NET `BackgroundService`
- **Hosting**: Registered as `IHostedService` via `AddHostedService<OrderQueueBackgroundService>()`
- **Execution Model**: Continuous execution with scheduled intervals
- **Lifecycle**: Respects `CancellationToken` for graceful shutdown

### Execution Schedule
- **Frequency**: Every 1 minute
- **Reasoning**: Ensures timely order processing and minimal latency
- **Execution Pattern**: Continues operation despite individual failures

### Core Operations

#### Order Discovery (`OrderQueueService`)
- **Primary Query**: Orders from last 24 hours without Bamboo OrderId
- **Secondary Query**: All pending orders (fallback if no recent orders)
- **Filtering Conditions**:
  - `o.OrderId == 0` (no Bamboo OrderId yet)
  - Status not "failed" or "succeeded"
  - Within specified date range

#### Message Publishing
- **Destination**: RabbitMQ via MassTransit `IPublishEndpoint`
- **Message Type**: `OrderCreated(RequestId)` from SharedContracts
- **Reliability**: Each order published individually with error handling
- **Batch Processing**: Multiple orders processed per execution cycle

### Error Handling Strategy
1. **Service Protection**: Entire operation wrapped in try-catch
2. **Individual Item Handling**: Each order queued in separate try-catch
3. **Continued Operation**: If one order fails, others continue processing
4. **Fallback Logic**: If recent orders not found, processes all pending orders
5. **Comprehensive Logging**: Detailed logs for troubleshooting

### Data Flow Process
1. **Query Execution**: Database query for unprocessed orders
2. **Order Validation**: Verify order status and eligibility for queuing
3. **Message Creation**: Create `OrderCreated` messages for eligible orders
4. **Message Publishing**: Publish messages to RabbitMQ via MassTransit
5. **Logging**: Record successful and failed operations

### Performance Considerations
- **Time-based Filtering**: Efficient date range queries to limit result set
- **Asynchronous Operations**: All operations async to prevent thread blocking
- **Scoped Services**: Fresh service instances per execution cycle
- **Connection Efficiency**: Leverages existing EF Core and MassTransit connection pools
- **Memory Management**: Proper disposal of scoped services

### Dependencies
- `OrderQueueService`: Core business logic for order queuing operations
- `ILogger`: For comprehensive logging
- `TimeSpan`: Configuration for execution interval (1 minute)

## Integration with Overall Architecture

### Event-Driven Architecture
- **OrderQueueBackgroundService** bridges the gap between synchronous order acceptance and async processing
- Provides reliability in the publisher-consumer model
- Ensures no orders are lost due to temporary processing issues

### API Integration
- **BrandBackgroundService** maintains data consistency with external Bamboo API
- Both services use the same RateLimiterService for API compliance
- Respect external API rate limits and retry-after policies

### Database Integration
- Both services use EF Core with PostgreSQL
- Proper transaction management for data consistency
- Efficient querying and bulk operations

### Messaging Integration
- OrderQueueBackgroundService publishes to RabbitMQ for Steller API consumption
- Uses MassTransit for reliable message delivery
- Integrates with shared `OrderCreated` contract

## Monitoring and Observability

### Logging Strategy
- **Structured Logging**: All operations logged with Serilog
- **Context Information**: Rich context for debugging and monitoring
- **Error Classification**: Differentiated error types for alerting
- **Performance Metrics**: Operation duration and success/failure rates

### Health Indicators
- **Execution Monitoring**: Regular execution confirmation
- **Error Rate Tracking**: Monitoring for increasing failure rates
- **Processing Time**: Tracking for performance degradation
- **API Response Times**: Monitoring external API performance

## Best Practices Implemented

### Resilience
- **Graceful Degrading**: Services continue despite partial failures
- **Rate Limit Respect**: Prevents API flooding and blocking
- **Recovery Mechanisms**: Automatic recovery from temporary failures
- **Circuit Protection**: Prevents cascading failures

### Performance
- **Async Patterns**: Non-blocking operations throughout
- **Batch Processing**: Efficient bulk operations
- **Connection Optimization**: Proper connection pool usage
- **Memory Management**: Efficient memory usage patterns

### Maintainability
- **Separation of Concerns**: Clear service responsibilities
- **Dependency Injection**: Proper service lifecycle management
- **Configurable Intervals**: Potential for configuration adjustments
- **Comprehensive Logging**: Rich debugging information

## Operational Considerations

### Configuration Management
- **Scheduled Intervals**: Currently hard-coded but could be configurable
- **Rate Limits**: Dynamically adjusted based on API responses
- **Error Thresholds**: Monitoring for failure rate limits

### Scaling Considerations
- **Single Instance**: Currently designed as single instance services
- **Idempotent Operations**: Safe for multiple executions
- **Database Load**: Consider impact of frequent queries on database

### Security
- **Authentication**: Uses same API keys as other services
- **Data Access**: Proper database permission scoping
- **Logging**: Avoid logging sensitive information

These background services are critical components that ensure the reliability and consistency of the Steller platform, acting as the reliability mechanisms that make the event-driven architecture robust and dependable.