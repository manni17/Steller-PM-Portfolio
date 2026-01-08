# Steller Platform - Deployment Infrastructure Documentation

## Overview

The Steller platform uses a sophisticated Docker-based deployment infrastructure that supports both production and development environments. The architecture is modular with separate infrastructure services that can be reused across multiple application deployments.

## Infrastructure Architecture

### Multi-Layer Deployment Model
The platform follows a three-layer deployment model:

1. **Infrastructure Layer**: PostgreSQL and RabbitMQ as shared services
2. **Application Layer**: Steller API and StellerConsumer API services  
3. **Presentation Layer**: Admin and Consumer Dashboards

### Container Network Topology

#### External Networks (Shared Infrastructure)
- `postgres_network`: External network for PostgreSQL connectivity
- `rabbitmq_network`: External network for RabbitMQ connectivity

#### Application Networks (Service-Specific)
- `steller-network`: Internal network for Steller API components
- `consumer-network`: Internal network for StellerConsumer API components

### Network Security
- Isolated bridge networks provide container isolation
- Services communicate using container hostnames
- Infrastructure services deployed as external networks for reusability

## Docker Configuration

### Build Strategy
- **Multi-stage builds**: Build-time optimization with separate build and runtime stages
- **Layer caching**: Optimized Docker builds with strategic layer ordering
- **SDK images**: Build stage uses .NET SDK, runtime stage uses ASP.NET Core runtime

### Dockerfile Patterns
#### Steller API Dockerfile
- Base: `mcr.microsoft.com/dotnet/sdk:9.0` for build phase
- Runtime: `mcr.microsoft.com/dotnet/aspnet:9.0` for execution
- Port: Exposes 5091
- Cache optimization: Strategic package restoration before source copying

#### StellerConsumer API Dockerfile
- Base: `mcr.microsoft.com/dotnet/sdk:9.0` for build phase  
- Runtime: `mcr.microsoft.com/dotnet/aspnet:9.0` for execution
- Port: Exposes 5092
- Additional ports: 5672, 15672, 443 for direct RabbitMQ access

## Docker Compose Orchestration

### Service Organization
The platform uses a distributed Docker Compose approach:

#### Infrastructure Services
Located in separate directories with their own compose files:
- `postgresql/docker-compose.yml`: PostgreSQL database service
- `rabbitmq/docker-compose.yml`: RabbitMQ message broker

#### Application Services  
Located in their respective service directories:
- `Steller/docker-compose.yml`: Steller API and migrations
- `StellerConsumer/docker-compose.yml`: StellerConsumer API
- `steller-admin-dashboard/docker-compose.yml`: Admin dashboard
- `steller-consumer-dashboard/docker-compose.yml`: Consumer dashboard

### Multi-Compose Orchestration
- **External networks**: Infrastructure services deployed separately and referenced as external networks
- **Service linking**: Applications connect to infrastructure via network references
- **Profile-based services**: Migration jobs run with `--profile tools`
- **Volume mapping**: Strategic volume mounts for persistent data and logs

### Orchestration Patterns

#### Infrastructure Independence
- PostgreSQL and RabbitMQ deployed as external services
- Reusable across multiple application deployments
- Shared by both Steller and StellerConsumer APIs

#### Service Dependencies
- Implicit dependencies via network connectivity
- Applications designed with resilience to temporary service unavailability
- MassTransit provides automatic reconnection to RabbitMQ
- EF Core handles database connection retries

## Environment Configuration Management

### Configuration Sources
1. **Local .env files**: Per-service configuration
2. **Docker Compose env_file**: References local environment files
3. **Environment overrides**: Docker Compose files can override .env values
4. **Default fallbacks**: Default values provided using `${VARNAME:-default}` pattern

### Sensitive Data Handling
- All credentials stored in environment variables
- No hardcoded secrets in source code
- Default development credentials provided in .env files
- Container-isolated environment variables

### Configuration Strategy
- **Service-specific**: Each service maintains its own configuration
- **Infrastructure-shared**: Common infrastructure settings (DB_HOST, etc.)
- **Development-friendly**: Safe defaults for local development
- **Production-ready**: Environment variable overrides for production

## Volume Management and Persistence

### Infrastructure Persistence
#### PostgreSQL Volumes
- Named volume: `postgres_data_v18`
- Mount point: `/var/lib/postgresql`
- Purpose: Database data persistence across container restarts

#### RabbitMQ Volumes
- Named volume: `rabbitmq_data` 
- Mount point: `/var/lib/rabbitmq`
- Purpose: Message broker state and configuration persistence

### Application Volumes
#### Log Volumes
- Mount type: Bind mount
- Source: `./Logs` from host
- Destination: `/app/Logs` in container
- Purpose: Access to application logs for monitoring and debugging

### Volume Strategy
- **Named volumes**: Used for critical infrastructure data
- **Bind mounts**: Used for application logs (development access)
- **No application persistence**: Applications are stateless and rebuilt on changes
- **Data durability**: Critical data survives container lifecycle events

## Deployment Strategies

### Complete Platform Deployment
1. Deploy infrastructure services (PostgreSQL, RabbitMQ)
2. Run database migrations
3. Deploy application services
4. Deploy frontend dashboards

### Selective Deployment
- Individual service deployment for focused development
- Infrastructure reuse across different deployment scenarios
- Hot-reload development setups available

### Profile-Based Deployments
- `--profile tools`: For migration and utility jobs
- Selective service activation based on profiles

## Service Dependencies and Startup Order

### Dependency Hierarchy
1. **Infrastructure** (PostgreSQL, RabbitMQ) - Foundation services
2. **Migrations** (Optional) - Database schema preparation
3. **Application services** (StellerConsumer, Steller) - Functional services
4. **Frontend services** - Presentation layer

### Resilience Patterns
- Applications built with retry logic for temporary service unavailability
- MassTransit provides automatic message broker reconnection
- Connection pool management in EF Core
- Graceful degradation when dependent services are temporarily unavailable

### Startup Coordination
- No explicit `depends_on` requirements (applications handle temporary unavailability)
- Infrastructure typically deployed separately and always available
- Applications implement connection retry logic

## Development Infrastructure Features

### Developer Productivity
- **Quick restart**: Individual services can be restarted without affecting others
- **Isolated testing**: Services can be tested in isolation
- **Environment parity**: Dev/prod environments are structurally similar
- **Log access**: Easy access to application logs via volume mounts

### Infrastructure Reusability
- **Multi-project**: Infrastructure services shared across multiple applications
- **Environment sharing**: Same infrastructure for multiple developer environments
- **Configuration flexibility**: Environment variables allow for different configurations

## Production Considerations

### Scalability
- **Horizontal scaling**: Services can be scaled independently
- **Network isolation**: Proper network segmentation for security
- **Load balancing**: Ready for reverse proxy and load balancer integration

### Security
- **Network security**: Proper container network isolation
- **Secret management**: Environment variables for sensitive data
- **Access controls**: Infrastructure services accessible only within networks

### Monitoring and Observability
- **Log aggregation**: Logs accessible via mounted volumes
- **Service health**: Designed for health check integration
- **Performance**: Optimized Docker images for efficient resource usage

## Deployment Commands Reference

### Infrastructure Setup
```bash
# Start PostgreSQL
cd postgresql && docker-compose up -d

# Start RabbitMQ  
cd ../rabbitmq && docker-compose up -d
```

### Application Deployment
```bash
# Deploy Steller API with migrations
cd Steller
docker-compose --profile tools up steller-migrations  # Optional
docker-compose up -d

# Deploy StellerConsumer API
cd ../StellerConsumer
docker-compose up -d
```

### Complete Platform
```bash
# Infrastructure
cd postgresql && docker-compose up -d && cd ../rabbitmq && docker-compose up -d

# Applications
cd ../Steller && docker-compose up -d
cd ../StellerConsumer && docker-compose up -d

# Frontends
cd ../steller-admin-dashboard && docker-compose up -d
cd ../steller-consumer-dashboard && docker-compose up -d
```

## Summary

The Steller platform's deployment infrastructure is designed with modularity, scalability, and developer productivity in mind. The architecture separates infrastructure from application services, provides robust resilience patterns, and supports flexible deployment strategies for both development and production environments.