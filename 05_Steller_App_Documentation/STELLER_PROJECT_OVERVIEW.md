# Steller Platform - Project Overview

## Executive Summary

The Steller platform is a comprehensive order management system that integrates with external services (specifically Bamboo Card Portal) to process card orders, manage inventory, and provide administrative and consumer dashboards. The system utilizes a microservice architecture with event-driven messaging patterns.

## Architecture Overview

### High-Level Architecture
```
┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Admin Dashboard│    │Consumer Dashboard│    │External Systems  │
│    (Vue.js)     │    │   (Vue.js)       │    │ (Bamboo Portal)  │
└─────────┬───────┘    └─────────┬────────┘    └─────────┬────────┘
          │                      │                       │
          │                      │                       │
          │              ┌───────▼─────────┐             │
          │              │   RabbitMQ      │             │
          │              │  (Message Queue)│             │
          │              └───────┬─────────┘             │
          │                      │                       │
    ┌─────▼──────────┐    ┌──────▼──────────┐            │
    │   Steller.API  │    │StellerConsumer  │            │
    │   (Consumer)   │◄──►│     (Publisher) │◄───────────┤
    └────────────────┘    └─────────────────┘            │
                               │                         │
                               │                         │
                        ┌──────▼─────────────┐           │
                        │   PostgreSQL DB    │◄──────────┤
                        │   (Data Storage)   │           │
                        └────────────────────┘           │
                                                          │
                                                ┌─────────▼────────┐
                                                │External APIs     │
                                                │(Bamboo CardPortal)│
                                                └──────────────────┘
```

## System Components

### 1. Steller.API (Consumer Service)
- **Technology Stack**: ASP.NET Core 9.0, Entity Framework, PostgreSQL
- **Role**: Message consumer that processes orders from the queue and integrates with external APIs
- **Key Features**:
  - Consumes messages from RabbitMQ queue
  - Processes orders asynchronously
  - Interacts with Bamboo Card Portal API
  - Maintains order status and tracking
  - Implements retry mechanisms for failed orders
  - Background services for data synchronization

### 2. StellerConsumer.API (Publisher Service)
- **Technology Stack**: ASP.NET Core 9.0, Entity Framework, PostgreSQL
- **Role**: Order acceptance and message publishing service
- **Key Features**:
  - Validates and stores orders in database
  - Deducts funds from partner's wallet during order creation
  - Handles multi-currency orders with real-time conversion
  - Publishes `OrderCreated` messages to RabbitMQ after successful order storage
  - Implements partner isolation for multi-tenant architecture

### 3. Admin Dashboard (Web Interface)
- **Technology Stack**: Vue.js 2.6, Bootstrap-Vue, Firebase integration
- **Role**: Administrative interface for managing partners, orders, and system configurations
- **Features**: Order tracking, reporting, partner management, product catalogs

### 4. Consumer Dashboard (Web Interface)
- **Technology Stack**: Vue.js 2.6, Bootstrap-Vue, Firebase integration
- **Role**: Partner/consumer-facing interface for placing and tracking orders
- **Features**: Order placement, tracking, account management

### 5. Shared Contracts
- **Technology Stack**: .NET Standard
- **Role**: Contains shared data contracts between services
- **Current Contract**: `OrderCreated` record with OrderId

### 6. Infrastructure Services
- **PostgreSQL**: Primary database for storing orders, partners, products, and system data
- **RabbitMQ**: Message broker for asynchronous order processing using MassTransit
  - Implements Publisher-Consumer pattern with `order-created` queue
  - Uses `OrderCreated(Guid OrderId)` message contract
  - Configured with 5 concurrent message processing limit
  - Includes 3-retry policy with 30-second intervals for transient failures

## Hybrid Development Architecture

The platform uses a **Hybrid Strategy**:
- **Infrastructure**: Dockerized (PostgreSQL & RabbitMQ)
- **Applications**: Run locally on the host machine (hot-reload enabled)

### Service Map
- **PostgreSQL**: Port 5432 (Docker)
- **RabbitMQ**: Port 5672/15672 (Docker)
- **Steller API**: http://localhost:5091
- **Consumer API**: http://localhost:5092
- **Admin Dashboard**: http://localhost:8080
- **Partner Dashboard**: http://localhost:8081

## Development Setup

### Prerequisites
- Docker Desktop with Compose plugin
- .NET 9 SDK
- Node.js 16+
- Vue CLI 4+
- Git

### Infrastructure Setup
```bash
# Start Shared Database (PostgreSQL 18)
docker-compose -f postgresql/docker-compose.yml up -d

# Start Message Broker (RabbitMQ)
docker-compose -f rabbitmq/docker-compose.yml up -d
```

### Running Services
```bash
# 1. Run backend services directly with .NET (for debugging)
cd Steller/Steller.Api
dotnet run

cd StellerConsumer/StellerConsumer.Api
dotnet run

# 2. Run frontend dashboards with hot reload
cd steller-admin-dashboard
npm run serve

cd steller-consumer-dashboard
npm run serve
```

## Database Schema

### Core Entities
- **Users**: User management with role-based access
- **Partners**: Business partner information
- **PartnerPersonContact**: Contact information for partners
- **Orders**: Order tracking and management
- **OrderItems**: Individual items within orders
- **OrderCards**: Card-specific information for orders
- **Products**: Product catalog
- **Brands**: Brand information
- **Wallets**: Partner wallet information
- **Categories**: Product categories
- **OrderStatuses**: Order status definitions

## Message Queue Architecture

The Steller system implements an event-driven architecture using RabbitMQ as the message broker to decouple order acceptance from fulfillment processes.

### Message Flow
1. Partner/consumer submits order via Consumer Dashboard
2. Consumer Dashboard calls StellerConsumer.API's `/api/order` endpoint
3. StellerConsumer.API validates the order and saves to database
4. StellerConsumer.API publishes `OrderCreated` message to RabbitMQ
5. Steller.API consumes the message via `OrderCreatedConsumer`
6. Steller.API processes the order with external Bamboo Card Portal API
7. Order status is updated in the database
8. Progress is tracked and available through both dashboards

## Security Features
- JWT-based authentication and authorization
- API key validation (middleware implementation)
- Rate limiting to prevent abuse of external APIs
- Secure credential management via environment variables
- HTTPS redirection in production

## External Integrations
- **Bamboo Card Portal API** (v1.0 and v2.0):
  - Base URLs configured in environment variables
  - OAuth client credentials for authentication
  - Used for card fulfillment and tracking

## Deployment Architecture
- Dockerized microservices
- External/shared PostgreSQL and RabbitMQ containers
- Environment-specific configurations
- Volume mounting for persistent data
- Cross-service networking using Docker networks

## Monitoring & Observability
- Structured logging with Serilog
- HTTP request/response timing
- Business metrics and operation tracking
- Error tracking with correlation IDs

## Development Patterns
- Repository Pattern with generic base repository
- Service Layer pattern for business logic
- Consumer-Publisher pattern using RabbitMQ
- Background services for periodic tasks
- Dependency injection throughout the application
- Separation of concerns with dedicated layers (API, Core, EF, Contracts)