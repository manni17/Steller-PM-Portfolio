# Steller RabbitMQ Documentation

This document provides comprehensive information about the RabbitMQ messaging system implementation in the Steller platform based on a complete messaging expert analysis.

## Overview

The Steller system uses RabbitMQ as its message broker to implement an event-driven architecture that decouples order acceptance from order fulfillment. This pattern allows the system to handle external API rate limits and provides resilience against temporary service failures.

### Technology Stack
- **Message Broker**: RabbitMQ 3-management
- **Client Library**: MassTransit for .NET
- **Transport**: RabbitMQ transport adapter
- **Protocol**: AMQP 0.9.1
- **Port Configuration**: 5672 (AMQP), 15672 (Management UI)

## Architecture and Messaging Pattern

### Event-Driven Architecture
The system implements a **Publisher-Consumer** pattern where:

1. **StellerConsumer.API** (Publisher): Accepts orders from customers/partners and publishes events
2. **RabbitMQ** (Message Broker): Stores and routes messages between services
3. **Steller.API** (Consumer): Processes orders asynchronously with external API integration

### Message Flow
```
[Consumer Dashboard] → [StellerConsumer.API/OrderController] → [RabbitMQ Queue: order-created] → [Steller.API/OrderCreatedConsumer] → [Bamboo Card Portal API]
```

### Communication Pattern
- **Pattern**: Fire-and-forget asynchronous messaging
- **Direction**: One-way from consumer service to processing service
- **Purpose**: Decouple order acceptance from processing to handle rate limits and external dependencies
- **Message Contract**: `OrderCreated(Guid OrderId)` in SharedContracts

## Configuration and Setup

### Docker Configuration
```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    container_name: shared_rabbitmq
    ports:
      - "5672:5672"   # AMQP protocol port
      - "15672:15672" # Management UI port
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_DEFAULT_USER}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_DEFAULT_PASS}
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    restart: always
    networks:
      - rabbitmq_network
```

### Application Configuration
Both applications use environment-based configuration:

**RabbitMQ Connection Settings**:
- Host: Configured via `RABBITMQ_HOST` environment variable (defaults to localhost)
- Username: Configured via `RABBITMQ_USERNAME` environment variable (defaults to guest)
- Password: Configured via `RABBITMQ_PASSWORD` environment variable (defaults to guest)

## Publisher Implementation (StellerConsumer.API)

### MassTransit Configuration
```csharp
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

### Message Publishing
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

**Key Features**:
- Uses `IPublishEndpoint` for message publishing
- Publishes after successful order validation
- Uses GUID-based RequestId for message correlation
- Handles validation before message publishing

## Consumer Implementation (Steller.API)

### MassTransit Configuration
```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();  // Register consumer

    x.UsingRabbitMq((context, cfg) =>
    {
        // Configure RabbitMQ connection
        var rabbitMqConfig = builder.Configuration.GetSection("RabbitMQ");
        var host = rabbitMqConfig["Host"] ?? "localhost";
        var username = rabbitMqConfig["Username"] ?? "guest";
        var password = rabbitMqConfig["Password"] ?? "guest";

        cfg.Host(host, h =>
        {
            h.Username(username);
            h.Password(password);
        });

        // Configure receive endpoint
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

### Consumer Implementation
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
                AccountId = 1256,
            };
            
            var orderResult = await _orderService.AddOrder(orderFromDataBaseDto);

            if (orderResult.Status)
            {
                _logger.LogInformation($"Successfully processed order {orderId}. Bamboo order id: {orderResult.Data}");
            }
            else
            {
                _logger.LogError($"Failed to process order {orderId}. Error: {orderResult.Message}, ErrorCode: {orderResult.ErrorCode}");
                await UpdateOrderStatusToFailed(orderId, orderResult.Message ?? "Unknown error");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Exception occurred while processing order {orderId}");
            await UpdateOrderStatusToFailed(orderId, $"Exception: {ex.Message}");
            throw; // Re-throw to trigger MassTransit retry logic
        }
    }
}
```

**Key Features**:
- Implements proper error handling with failure status updates
- Uses MassTransit retry mechanism for transient failures
- Prevents infinite re-queuing through failure status tracking
- Handles external API communication (Bamboo Card Portal)

## Queue Structure and Routing

### Queue Configuration
- **Queue Name**: `order-created` - dedicated queue for order processing
- **Durable**: Yes (persistent messages survive broker restart)
- **Auto-delete**: No (queue persists after application restart)
- **Exclusive**: No (multiple applications can connect)

### Routing Configuration
- **Exchange Type**: Default direct exchange (MassTransit convention)
- **Routing Key**: Auto-generated based on message type (`OrderCreated`)
- **Binding**: Automatic binding between publisher and consumer via MassTransit conventions
- **Message Persistence**: Messages are persistent by default

### Receive Endpoint Configuration
```csharp
cfg.ReceiveEndpoint("order-created", e =>
{
    e.ConfigureConsumer<OrderCreatedConsumer>(context);
    
    // Retry policy
    e.UseMessageRetry(r => r.Interval(3, TimeSpan.FromSeconds(30)));
    
    // Concurrency control
    e.ConcurrentMessageLimit = 5;
});
```

### Retry Configuration
- **Retry Count**: 3 retries for transient failures
- **Retry Interval**: 30-second intervals between retries
- **Backoff Strategy**: Fixed interval retry
- **Failure Handling**: After retries exhausted, message behavior depends on MassTransit default configuration

## Security Considerations

### Current Security State
- **Credentials**: Uses default RabbitMQ credentials (guest/guest) - security risk for production
- **Connection Security**: No TLS/SSL configuration visible in current setup
- **Network Isolation**: Docker network isolation between services
- **Environment-based**: Credentials loaded from environment variables

### Security Vulnerabilities
1. **Default Credentials**: Uses default guest/guest credentials
2. **No SSL/TLS**: Connection between applications and RabbitMQ is unencrypted
3. **Plain Text Credentials**: Credentials stored in environment files
4. **Management UI**: Accessible with default credentials

### Security Recommendations
1. **Strong Credentials**: Replace default credentials with strong, unique passwords
2. **TLS/SSL Encryption**: Implement SSL/TLS for RabbitMQ connections
3. **User Management**: Create specific users with limited permissions
4. **Network Security**: Restrict RabbitMQ port access with firewall rules
5. **Environment Security**: Encrypt environment files containing credentials

## Performance and Scaling

### Current Performance Configuration
- **Concurrency Limit**: 5 concurrent messages processed simultaneously
- **Retry Policy**: 3 retries with 30-second intervals for transient failures
- **Message Persistence**: All messages are persistent by default
- **Connection Pooling**: MassTransit manages connection pooling automatically

### Performance Characteristics
- **Throughput**: Limited to 5 concurrent order processing operations
- **Retry Strategy**: Prevents message loss during temporary failures  
- **Rate Limiting**: Consumer concurrency limits apply backpressure to prevent overwhelming external APIs
- **Error Handling**: Failed messages are retried with backoff before being moved to error queue

### Scaling Considerations
- **Horizontal Scaling**: Consumer service can be scaled horizontally (multiple instances)
- **Message Backlog**: Queue can accumulate messages during processing delays
- **Resource Utilization**: Concurrency limit helps manage resource usage
- **External API Integration**: Rate limiting considerations for Bamboo Card Portal API

### Performance Optimization Recommendations
1. **Connection Management**: Implement proper connection lifecycle management
2. **Message Acknowledgment**: Use manual acknowledgment mode for better control
3. **Queue Configuration**: Optimize queue settings based on message volume patterns
4. **Consumer Scaling**: Monitor and scale consumer instances based on queue depth
5. **Monitoring**: Implement RabbitMQ metrics collection and alerting

## Management and Monitoring

### Management Interface
- **Port**: 15672 for Management UI
- **Authentication**: Basic authentication (currently with default credentials)
- **Features**: Queue monitoring, message inspection, connection tracking

### Monitoring Considerations
- **Queue Depth**: Monitor message queue lengths
- **Consumer Health**: Track consumer processing rates
- **Error Rates**: Monitor message processing failures
- **Connection Status**: Track broker connectivity

## Operational Guidelines

### Deployment
1. Start RabbitMQ service before applications
2. Verify RabbitMQ connectivity before starting services
3. Monitor queue depths during peak loads
4. Use environment-specific RabbitMQ configurations

### Maintenance
1. Regular monitoring of queue metrics
2. Log analysis for troubleshooting
3. Performance tuning based on usage patterns
4. Security updates for RabbitMQ and MassTransit

### Troubleshooting
1. Check RabbitMQ logs for connectivity issues
2. Monitor queue depths for potential backlogs
3. Verify consumer health and processing rates
4. Validate message schemas for compatibility

## Best Practices

### Development Best Practices
- Use sanitized data for development environments
- Implement proper error handling in consumers
- Test message flow under various failure conditions
- Maintain message schema compatibility

### Production Best Practices
- Use strong authentication and encryption
- Implement comprehensive monitoring
- Regular backup and disaster recovery procedures
- Performance testing under load conditions
- Security hardening of RabbitMQ instance