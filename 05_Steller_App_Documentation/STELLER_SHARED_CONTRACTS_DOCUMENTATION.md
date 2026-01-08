# Steller Shared Contracts Documentation

## Overview

The Shared Contracts project serves as the critical communication layer between the StellerConsumer API (publisher) and Steller API (consumer) services. It defines the message contract used for inter-service communication via RabbitMQ.

## Project Structure

```
SharedContracts/
├── OrderCreated.cs          # Primary message contract
├── SharedContracts.csproj   # Project configuration
├── SharedContracts.sln      # Solution file
└── .git/                    # Version control
```

### Technology Stack
- **Framework**: .NET 9.0
- **Language**: C# 11+ (records support)
- **Pattern**: Message Contract / Data Transfer Object
- **Purpose**: Cross-service communication interface

## Message Contract Definition

### OrderCreated Record
```csharp
namespace SharedContracts
{
    // Current version: Simplified for minimal data transfer
    public record OrderCreated(Guid OrderId);
    
    // Previous version (commented out):
    // public record OrderCreated(Guid OrderId, string ProductName, int Quantity);
}
```

### Contract Analysis

#### Current Contract (Simplified)
- **Properties**: 1 (`Guid OrderId`)
- **Purpose**: Minimal identifier for order processing
- **Benefits**: Smaller message size, reduced complexity, faster serialization

#### Historical Contract (Previously Used)
- **Properties**: 3 (`Guid OrderId, string ProductName, int Quantity`)
- **Evolution**: Major simplification with breaking change

## Purpose and Functionality

### Messaging Pattern
- **Pattern**: Publisher-Consumer via RabbitMQ
- **Publisher**: StellerConsumer API (accepts orders and publishes messages)
- **Consumer**: Steller API (consumes messages and processes orders)
- **Transport**: MassTransit with RabbitMQ transport

### Data Flow
1. **Publisher Side**: StellerConsumer API creates `OrderCreated` message with OrderId
2. **Message Queue**: RabbitMQ stores and routes the message 
3. **Consumer Side**: Steller API consumes message and retrieves additional data from database
4. **Processing**: Complete order fulfillment using external Bamboo API

### Contract Usage
The `OrderCreated` record serves as the message payload that:
- Identifies the order to be processed
- Provides minimal data needed for message routing
- Enables loose coupling between services
- Reduces message size and network overhead

## Service Integration Points

### StellerConsumer API (Publisher)
- **Reference**: Depends on SharedContracts for message creation
- **Usage**: Creates and publishes `OrderCreated` messages via MassTransit
- **Timing**: After successful order validation and storage in database

### Steller API (Consumer)
- **Reference**: Depends on SharedContracts for message consumption
- **Usage**: Receives and processes `OrderCreated` messages
- **Handling**: Uses OrderId to retrieve full order details from database

## Versioning and Evolution

### Contract Evolution Strategy
The contract shows evidence of intentional evolution from a more complex to a simpler form:

**Before Evolution:**
```csharp
public record OrderCreated(Guid OrderId, string ProductName, int Quantity);
```

**After Evolution:**
```csharp
public record OrderCreated(Guid OrderId);
```

### Evolution Impact
- **Breaking Change**: Major breaking change requiring coordinated deployment
- **Justification**: Reduced complexity and improved maintainability
- **Data Retrieval**: Services now retrieve additional data from database using OrderId
- **Migration**: Coordinated deployment of both services required

### Versioning Approach
- **Strategy**: Breaking changes allowed with coordinated deployments
- **Documentation**: Changes documented through code comments
- **Backward Compatibility**: Not maintained (intentional simplification)

## Implementation Details

### Record Features
- **Immutability**: Records provide immutable data structures by default
- **Value-Based Equality**: Records use value-based equality semantics
- **Compact Syntax**: Concise definition with implicit getter methods
- **Type Safety**: Strong typing with `Guid` for order identification

### Serialization
- **Format**: JSON serialization via MassTransit
- **Mechanism**: Automatic serialization/deserialization
- **Performance**: Optimized for minimal message overhead
- **Compatibility**: Cross-platform compatibility maintained

## Cross-Service Compatibility

### Binary Compatibility
- **Assembly**: Compiled as .NET 9.0 compatible assembly
- **Reference**: Both services reference the same SharedContracts version
- **Deployment**: Coordinated deployments when breaking changes occur

### Message Format Compatibility
- **Structure**: Fixed contract structure ensures compatibility
- **Serialization**: JSON format ensures cross-platform compatibility
- **Field Addition**: Forward-compatible (new fields can be added safely)
- **Field Removal**: Breaking change (requires coordinated deployment)

## Security Considerations

### Data Sensitivity
- **Minimal Data**: Contract contains only essential identifier
- **No PII**: No personally identifiable information in the contract
- **Security**: Reduces risk of sensitive data exposure in message queues

### Validation
- **Type Safety**: Guid type provides basic validation
- **Runtime Validation**: Performed by consuming services
- **Input Sanitization**: Assumed to be handled by consuming services

## Performance Characteristics

### Message Size
- **Small Payload**: Minimal data reduces network overhead
- **Serialization Speed**: Compact structure optimizes serialization
- **Queue Efficiency**: Smaller messages improve queue performance

### Processing Efficiency
- **Fast Deserialization**: Simple structure enables quick processing
- **Low Memory Usage**: Minimal memory footprint per message
- **Network Efficiency**: Reduced bandwidth usage

## Best Practices Applied

### ✅ Current Implementation
- **Simplicity**: Minimal contract reduces complexity
- **Clear Responsibility**: Single purpose (message contract)
- **Modern Syntax**: Uses C# records for immutable data
- **Type Safety**: Strong typing with Guid
- **Separation**: Dedicated project for contracts

### 📚 Recommended Improvements
- **Documentation**: Add XML documentation comments
- **Validation**: Consider data annotations for basic validation
- **Versioning**: Formalize versioning strategy for breaking changes
- **Testing**: Add serialization/deserialization tests
- **Monitoring**: Add message processing metrics

## Integration Patterns

### Messaging Strategy
- **Async Communication**: Event-driven, non-blocking communication
- **Loose Coupling**: Services can evolve independently
- **Scalability**: Multiple consumers can process messages
- **Reliability**: Message persistence ensures delivery

### Data Retrieval Strategy
- **On-Demand**: Additional data retrieved by consumers as needed
- **Database**: Full order details stored in PostgreSQL
- **Efficiency**: Reduces message size while maintaining flexibility

## Future Considerations

### Potential Enhancements
- **Version Negotiation**: Handle multiple contract versions simultaneously
- **Schema Registry**: Formal contract management system
- **Validation Framework**: Enhanced validation capabilities
- **Error Handling**: Improved error handling for malformed messages

### Evolution Guidelines
- **Breaking Changes**: Coordinate deployments when making breaking changes
- **Backwards Compatibility**: Consider backwards compatibility for minor changes
- **Migration Strategy**: Plan migration paths for contract updates
- **Testing**: Thorough testing of both services during contract changes

## Deployment Considerations

### Deployment Strategy
- **Synchronized Deployment**: Both services must use same contract version
- **Breaking Changes**: Require coordinated deployment of publisher and consumer
- **Rollback Plan**: Plan for coordinated rollbacks if needed
- **Monitoring**: Monitor message processing after contract changes