# Steller Platform - DevOps Processes and Deployment Documentation

## Overview

The Steller platform implements a containerized, event-driven architecture with a sophisticated DevOps approach. This document provides comprehensive guidance on development workflows, build processes, CI/CD considerations, and deployment strategies.

## Build and Development Infrastructure

### Project Structure
```
Steller Platform/
├── Steller/                    # Main API (Order Processing Consumer)
│   ├── Steller.Api/           # API Application
│   ├── Steller.Core/          # Business Logic & Models  
│   ├── Steller.EF/            # Entity Framework Layer
├── StellerConsumer/           # Consumer API (Order Acceptance Publisher)
│   ├── StellerConsumer.Api/   # API Application
│   ├── StellerConsumer.Core/  # Business Logic & Models
│   ├── StellerConsumer.EF/    # Entity Framework Layer
├── SharedContracts/           # Shared Message Contracts
├── steller-admin-dashboard/   # Admin Dashboard (Vue.js)
├── steller-consumer-dashboard/ # Consumer Dashboard (Vue.js)
└── postgresql/                # External Infrastructure Service
└── rabbitmq/                  # External Infrastructure Service
```

### Build Process

#### .NET Applications
All .NET applications use multi-stage Docker builds:

1. **Build Stage**: Uses SDK image to compile and publish
2. **Runtime Stage**: Uses minimal ASP.NET Core runtime image
3. **Layer Caching**: Dependencies copied and restored first for optimal caching
4. **Optimization**: Separate Dockerfiles with environment-specific configurations

#### Frontend Applications
Vue.js applications use:
1. **Build Stage**: Node.js image for compilation and bundling
2. **Runtime Stage**: Nginx for serving static assets
3. **Optimization**: Production-ready bundles with compression

### Docker Configuration

#### Multi-Stage Builds
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /app

# Copy project files first for better caching
COPY SharedContracts/*.csproj ./SharedContracts/
COPY Steller/Steller.sln ./
COPY Steller/Steller.Api/*.csproj ./Steller/Steller.Api/
# ... restore dependencies first ...

# Copy source code
COPY SharedContracts/. ./SharedContracts/
COPY Steller/.

# Build and publish
RUN dotnet publish -c Release -o /app/publish

# Runtime stage  
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Steller.Api.dll"]
```

#### Build Optimization Strategies
1. **Dependency Caching**: .csproj files copied before source code
2. **Sequential Restoration**: Dependencies restored in dependency order
3. **No Cache Builds**: Explicit no-cache flags for deterministic builds
4. **Multi-platform**: Compatible with different architectures

## Development Workflows

### Local Development Setup

#### Prerequisites
- Docker Desktop with Compose plugin
- .NET 9 SDK
- Node.js 16+ (for frontend development)
- Git

#### Component-Isolated Development
```bash
# 1. Start shared infrastructure (separate terminals)
cd postgresql && docker-compose up -d
cd ../rabbitmq && docker-compose up -d

# 2. Develop individual services with hot-reload
cd Steller/Steller.Api
dotnet watch run

cd StellerConsumer/StellerConsumer.Api  
dotnet watch run

# 3. Frontend development
cd steller-admin-dashboard
npm run serve

cd steller-consumer-dashboard
npm run serve
```

#### Full Platform Deployment
```bash
# For complete platform testing
cd postgresql && docker-compose up -d
cd ../rabbitmq && docker-compose up -d
cd ../Steller && docker-compose up -d
cd ../StellerConsumer && docker-compose up -d
cd ../steller-admin-dashboard && docker-compose up -d
cd ../steller-consumer-dashboard && docker-compose up -d
```

### Environment Management

#### Configuration Strategy
1. **Development**: .env files with localhost endpoints
2. **Docker**: Service names for inter-container communication
3. **Production**: Externalized secrets and secure endpoints

#### Environment Variables
- **Database**: DB_HOST, DB_NAME, DB_USERNAME, DB_PASSWORD
- **RabbitMQ**: RABBITMQ_HOST, RABBITMQ_USERNAME, RABBITMQ_PASSWORD
- **External APIs**: Base URLs and authentication credentials
- **JWT**: Keys, issuers, audiences
- **Email**: SMTP settings and sender information

### Team Development Patterns
1. **Component Ownership**: Teams can work on individual services independently
2. **Shared Infrastructure**: Common PostgreSQL and RabbitMQ services
3. **Environment Parity**: Consistent environments across development, staging, and production
4. **Integration Points**: Shared contracts in SharedContracts project

## Deployment Strategies

### Container-Based Deployment

#### Service Architecture
```
┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Admin Dashboard│    │Consumer Dashboard│    │Infrastructure    │
│    (Nginx)      │    │   (Nginx)        │    │ (PostgreSQL,    │
└─────────┬───────┘    └─────────┬────────┘    │  RabbitMQ)      │
          │                      │               └─────────┬──────┘
          │              ┌───────▼─────────┐               │
          │              │   RabbitMQ      │               │
          │              │  (Message Queue)  ◄─────────────┤
          │              └───────┬─────────┘               │
          │                      │                         │
    ┌─────▼──────────┐    ┌──────▼──────────┐              │
    │   Steller.API  │    │StellerConsumer  │              │
    │   (Consumer)   │◄───┤     (Publisher) │              │
    └────────────────┘    └─────────────────┘              │
                                                            │
                                                  ┌───────▼───────┐
                                                  │External APIs  │
                                                  │(Bamboo Portal)│
                                                  └───────────────┘
```

#### Deployment Patterns
1. **Infrastructure-First**: PostgreSQL and RabbitMQ deployed separately
2. **Service-Independent**: Each service can be deployed independently
3. **Configuration-Driven**: Environment variables enable different deployment targets
4. **Network-Isolated**: Services use Docker networks for secure communication

### Migration Strategy

#### Database Migrations
- **Dedicated Service**: Separate Dockerfile for migration execution
- **Profile-Based**: `--profile tools` for migration service orchestration
- **Environment-Aware**: Same configuration as main applications
- **Safe Execution**: Transactional migrations with rollback capabilities

```bash
# Run migrations separately
docker-compose --profile tools up steller-migrations

# Or run as part of full deployment
docker-compose --profile tools run --rm steller-migrations
```

### Scaling Strategies
1. **Horizontal Scaling**: Service instances can be scaled independently
2. **Resource Allocation**: Docker resource constraints for predictable performance
3. **Load Distribution**: Container orchestrators can handle traffic distribution
4. **Queue Processing**: Consumer services can be scaled based on queue depth

## CI/CD Considerations

### Current State
The platform is designed for container-based deployments but lacks formal CI/CD pipeline definitions:

#### Built-In Capabilities
1. **Container-Ready**: All services have Dockerfiles for consistent builds
2. **Environment Configurable**: All settings externalized for different environments
3. **Multi-Stage Builds**: Optimized for both development and production
4. **Migration Support**: Database migrations as separate deployable units

#### Suggested CI/CD Implementation

#### GitHub Actions Pipeline Example
```yaml
name: Build and Deploy

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 9.0.x
        
    - name: Restore dependencies
      run: dotnet restore
      
    - name: Build
      run: dotnet build --no-restore
      
    - name: Test
      run: dotnet test --no-build --verbosity normal
      
    - name: Publish Artifacts
      run: |
        dotnet publish -c Release -o ./publish Steller/Steller.Api
        dotnet publish -c Release -o ./publish StellerConsumer/StellerConsumer.Api
        
    - name: Build Docker Images
      run: |
        docker build -f ./Steller/Dockerfile -t steller-api:${{ github.sha }} .
        docker build -f ./StellerConsumer/Dockerfile -t steller-consumer:${{ github.sha }} .
        
    - name: Push to Registry
      run: |
        docker tag steller-api:${{ github.sha }} registry/repo/steller-api:${{ github.sha }}
        docker push registry/repo/steller-api:${{ github.sha }}
```

#### Deployment Pipelines
1. **Development**: Automatic builds on commits to feature branches
2. **Staging**: Deploy on merge to develop branch with automated testing
3. **Production**: Manual approval process for main branch deployment
4. **Rollback**: Automated rollback capability if health checks fail

## Testing Configuration

### Current State
- **No Unit Tests Visible**: No test project files detected in current analysis
- **Integration Testing**: Limited to manual API and UI testing
- **Component Testing**: Individual service testing possible with container isolation

### Recommended Testing Strategy
1. **Unit Tests**: NUnit/xUnit for business logic testing
2. **Integration Tests**: Testcontainers for database and RabbitMQ integration
3. **Contract Testing**: Verify SharedContracts compatibility between services
4. **End-to-End Tests**: UI testing for dashboard functionality
5. **Performance Tests**: Load testing for expected transaction volumes

### Test Configuration Patterns
```csharp
// Example test configuration
[TestClass]
public class OrderServiceTests
{
    [TestMethod]
    public async Task AddOrder_ValidOrder_ShouldPublishOrderCreated()
    {
        // Arrange
        var mockPublishEndpoint = new Mock<IPublishEndpoint>();
        var orderService = new OrderService(mockPublishEndpoint.Object);
        
        // Act
        var result = await orderService.AddOrder(validOrder);
        
        // Assert
        Assert.IsTrue(result.Status);
        mockPublishEndpoint.Verify(p => p.Publish(It.IsAny<OrderCreated>()), Times.Once);
    }
}
```

## Multi-Environment Support

### Environment Types
1. **Development**: Local development with hot-reload
2. **Testing**: Isolated test environment with test data
3. **Staging**: Production-like environment for validation
4. **Production**: Live environment with security hardening

### Configuration Management
- **.env Files**: Different environment configurations
- **Docker Compose Overrides**: Environment-specific service configurations
- **Environment Variables**: Runtime configuration through deployment tools
- **Secrets Management**: Integration ready for Azure Key Vault, AWS Secrets Manager, or HashiCorp Vault

### Deployment Environment Patterns
```bash
# Development
docker-compose -f docker-compose.yml up -d

# Staging with overrides
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d

# Production with security hardening
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Security Considerations

### Current Security Measures
1. **Container Isolation**: Services isolated in separate containers
2. **Network Segmentation**: Docker networks for secure service communication
3. **Credential Externalization**: No hardcoded credentials in source code
4. **API Key Authentication**: Partner-specific access control

### Security Enhancement Recommendations
1. **Image Scanning**: Integrate security scanning for Docker images
2. **Runtime Security**: Implement runtime security monitoring
3. **Secrets Management**: Implement enterprise secrets management solution
4. **Network Policies**: Implement network policies for additional isolation
5. **SSL/TLS**: Require encrypted connections in production
6. **Security Updates**: Automated dependency and base image updates

## Performance and Scaling

### Current Performance Features
1. **Message Queue**: RabbitMQ for asynchronous processing
2. **Rate Limiting**: Built-in rate limiters for external API calls
3. **Connection Pooling**: EF Core and HttpClient connection pooling
4. **Background Services**: Continuous synchronization services

### Scaling Considerations
1. **Horizontal Scaling**: Services can be scaled independently based on load
2. **Queue Depth Monitoring**: Monitor RabbitMQ queue depths for scaling decisions
3. **Database Connections**: Scale database connection pools with service instances
4. **External API Limits**: Respect Bamboo API rate limits during scaling

## Operational Excellence

### Monitoring and Observability
1. **Application Logging**: Serilog with structured logging
2. **Request Monitoring**: HTTP request/response timing
3. **Error Tracking**: Global exception handling with correlation IDs
4. **Business Metrics**: Order processing and performance indicators

### Health Checking
- **Service Health**: (Not implemented but could be added)
- **Database Connectivity**: Database connection health
- **Message Broker**: RabbitMQ connectivity health
- **External API**: Bamboo API availability health

### Disaster Recovery
1. **Database Backups**: PostgreSQL backup strategies
2. **Message Persistence**: RabbitMQ message durability
3. **Service Recovery**: Docker restart policies
4. **Data Recovery**: Database migration rollback capabilities

## Deployment Commands Reference

### Local Development
```bash
# Complete platform startup
./setup-local.sh  # If available, or run steps manually

# Individual service deployment
cd Steller && docker-compose up -d
cd StellerConsumer && docker-compose up -d

# Database migrations
cd Steller && docker-compose --profile tools up steller-migrations
```

### Production Deployment (Conceptual)
```bash
# Pull latest images
docker-compose pull

# Deploy with zero-downtime
docker-compose up -d --no-deps --force-recreate <service-name>

# Health check
curl http://<service-host>/health  # If implemented

# Rollback
docker-compose down
docker-compose up -d
```

## Best Practices Summary

### Development Best Practices
- ✅ Use container-based development for environment consistency
- ✅ Externalize all configuration via environment variables
- ✅ Implement proper service separation and independence
- ✅ Use multi-stage builds for optimized container images
- ✅ Maintain shared infrastructure for multiple services

### Deployment Best Practices
- ✅ Implement automated CI/CD pipelines
- ✅ Use environment-specific configuration management
- ✅ Implement comprehensive testing at all levels
- ✅ Monitor application and infrastructure health
- ✅ Plan for disaster recovery and rollback scenarios

### Security Best Practices
- ✅ Never hardcode sensitive information
- ✅ Use secrets management for production deployments
- ✅ Implement proper network segmentation
- ✅ Scan container images for vulnerabilities
- ✅ Use minimal runtime images

## Future Enhancements

### CI/CD Improvements
1. **Automated Testing**: Integration of comprehensive test suites
2. **Security Scanning**: Image vulnerability scanning in pipeline
3. **Performance Testing**: Automated performance validation
4. **Deployment Gates**: Automated promotion between environments

### Infrastructure as Code
1. **Terraform**: Infrastructure provisioning automation
2. **Helm Charts**: Kubernetes deployment packaging
3. **ARM Templates**: Azure resource provisioning
4. **Configuration Management**: Infrastructure configuration versioning

### Observability Enhancements
1. **Metrics Collection**: Application Performance Monitoring
2. **Distributed Tracing**: Request correlation across services
3. **Alerting**: Automated alerting for critical issues
4. **Dashboarding**: Operational dashboards for system health

This DevOps approach provides a solid foundation for the Steller platform while leaving room for further automation and operational excellence improvements.