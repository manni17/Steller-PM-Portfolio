# Steller System Architecture & Business Context Documentation

## Executive Summary
Steller is a comprehensive order management system that integrates with external services (specifically Bamboo Card Portal) to process card orders, manage inventory, and provide administrative and consumer dashboards. The system utilizes a microservice architecture with event-driven messaging patterns.

## Business Context
The Steller platform appears to be a card ordering and management system designed for businesses that need to issue cards (likely gift cards, employee cards, or similar). It connects to the Bamboo Card Portal API to process orders and manages the entire lifecycle from creation to fulfillment.

## System Architecture Overview

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
    │   (Producer)   │◄──►│     (Consumer)  │◄───────────┤
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

### Component Breakdown

#### 1. Steller.API (Primary Service)
- **Technology Stack**: ASP.NET Core 9.0, Entity Framework, PostgreSQL
- **Role**: The main order processing API that acts as a message producer
- **Key Features**:
  - RESTful API endpoints for order management
  - JWT Authentication & Authorization  
  - Rate limiting for external API calls
  - Event publishing to RabbitMQ queues
  - Integration with Bamboo Card Portal API
  - Email notification service
  - Background services for data synchronization

#### 2. StellerConsumer.API (Consumer Service)
- **Technology Stack**: ASP.NET Core 9.0, Entity Framework, PostgreSQL
- **Role**: Message consumer that processes orders from the queue
- **Key Features**:
  - Consumes messages from RabbitMQ queue
  - Processes orders asynchronously
  - Interacts with Bamboo Card Portal API
  - Maintains order status and tracking
  - Implements retry mechanisms for failed orders

#### 3. Admin Dashboard (Web Interface)
- **Technology Stack**: Vue.js 2.6, Bootstrap-Vue, Firebase integration
- **Role**: Administrative interface for managing partners, orders, and system configurations
- **Features**: Order tracking, reporting, partner management, product catalogs

#### 4. Consumer Dashboard (Web Interface)
- **Technology Stack**: Vue.js 2.6, Bootstrap-Vue, Firebase integration
- **Role**: Partner/consumer-facing interface for placing and tracking orders
- **Features**: Order placement, tracking, account management

#### 5. Shared Contracts
- **Technology Stack**: .NET Standard
- **Role**: Contains shared data contracts between services
- **Current Contract**: `OrderCreated` record with OrderId

#### 6. Infrastructure Services
- **PostgreSQL**: Primary database for storing orders, partners, products, and system data
- **RabbitMQ**: Message broker for asynchronous order processing using MassTransit
  - Implements Publisher-Consumer pattern with `order-created` queue
  - Uses `OrderCreated(Guid OrderId)` message contract
  - Configured with 5 concurrent message processing limit
  - Includes 3-retry policy with 30-second intervals for transient failures
  - Currently uses default guest/guest credentials (requires security hardening for production)
- **Redis**: (Potentially used for caching - not explicitly visible in configs)

## StellerConsumer API (Publisher)

The StellerConsumer API serves as the order entry point in the Steller platform. It implements a publisher-only architecture that accepts orders from clients/partners and publishes asynchronous messages for processing.

### Architecture and Responsibilities
- **Role**: Order acceptance and message publishing service
- **Framework**: ASP.NET Core 9.0 with Entity Framework Core
- **Database**: PostgreSQL for storing orders and related data
- **Messaging**: Publisher-only (publishes to RabbitMQ, doesn't consume)
- **Authentication**: Dual authentication with JWT tokens and API keys

### Key Features
- **Order Processing**: Validates and stores orders in database
- **Wallet Integration**: Deducts funds from partner's wallet during order creation
- **Currency Conversion**: Handles multi-currency orders with real-time conversion
- **Message Publishing**: Publishes `OrderCreated` messages to RabbitMQ after successful order storage
- **Partner Isolation**: Multi-tenant architecture with partner-specific data access

### API Endpoints
- **POST /api/order**: Create new orders (publishes to RabbitMQ)
- **GET /api/order**: Search and filter orders
- **GET /api/order/{orderId}**: Retrieve specific order
- Additional endpoints for partners, brands, products, and other entities

### External Integration
- **Note**: Unlike the Steller API (consumer), the StellerConsumer API does NOT make direct external API calls
- Instead, it publishes messages to RabbitMQ which are later consumed by Steller API for external API integration
- This decouples order acceptance from external API processing

### Security Implementation
- **API Key Authentication**: "X-Api-Key" header with database validation
- **JWT Authentication**: Fallback to JWT tokens with "partner" claim
- **Partner Isolation**: Each request scoped to specific partner's data
- **Input Validation**: Comprehensive validation for order data

## Steller API (Consumer and External API Processor)

The Steller API serves as the consumer and external API processing service in the Steller platform. It consumes messages from RabbitMQ and processes them by calling external Bamboo Card Portal API.

### Architecture and Responsibilities
- **Role**: Message consumption and external API integration service
- **Framework**: ASP.NET Core 9.0 with Entity Framework Core
- **Database**: PostgreSQL for storing order status and synchronization data
- **Messaging**: Consumer-only (consumes from RabbitMQ, doesn't publish)
- **External Integration**: Direct calls to Bamboo Card Portal API
- **Authentication**: JWT tokens for internal API access, Basic Auth for external API

### Key Features
- **Message Consumption**: Consumes `OrderCreated` messages from RabbitMQ
- **External API Integration**: Calls Bamboo Card Portal API for card fulfillment
- **Rate Limiting**: Sophisticated rate limiting system to respect Bamboo API limits
- **Data Synchronization**: Updates local database with external API responses
- **Error Handling**: Comprehensive failure management to prevent infinite retries
- **Database Sync**: Maintains local database consistency with external API

### API Endpoints
- **GET /api/order/{id}**: Get order by ID from external API
- **POST /api/order/placeOrder**: Create order with external API call
- Additional endpoints for system administration and monitoring

### External API Integration
- **Primary Integration**: Bamboo Card Portal API for card processing
- **Authentication Method**: Basic Authentication with Client ID/Secret
- **Rate Limits**:
  - Place Order: 2 requests per second
  - Get Order: 120 requests per minute
  - Catalog: 2 requests per hour
  - Exchange Rate: 20 requests per minute
- **Error Handling**: Parses "retry-after" responses and dynamically adjusts rate limits

### Message Processing Architecture
- **Consumer**: MassTransit consumer for `order-created` queue
- **Concurrency**: Limited to 5 concurrent message processing to respect external API limits
- **Retry Policy**: 3 retry attempts with 30-second intervals for transient failures
- **Failure Management**: Updates order status to prevent infinite re-queuing

### Rate Limiting System
- **Sliding Window Algorithm**: Implements proper rate limiting with time windows
- **Multiple Endpoints**: Different rate limits for different API operations
- **Dynamic Adjustment**: Adapts to "retry-after" responses from external API
- **Thread Safety**: Uses ConcurrentDictionary for safe multi-threaded operations

### Security Implementation
- **Internal Authentication**: JWT Bearer tokens for internal endpoints
- **External Authentication**: Basic Authentication with client credentials for Bamboo API
- **Credential Storage**: Environment-based configuration for sensitive data
- **API Key Management**: Client ID/Secret for external API access

## Monitoring and Observability

The Steller platform implements comprehensive monitoring and observability using Serilog for structured logging and various operational patterns.

### Serilog Configuration
- **Output Sinks**: Console and file (rolling daily with 7-day retention)
- **Structure**: JSON-formatted structured logs with property enrichment
- **Context Enrichment**: RequestHost and UserAgent automatically added to HTTP requests
- **File Pattern**: `Logs/errors-.log` with daily rolling intervals
- **Format**: Structured JSON for log aggregation and analysis

### Request Monitoring
- **HTTP Request Logging**: Automatic request/response timing via `UseSerilogRequestLogging`
- **Context Information**: Host and User-Agent automatically logged
- **Performance Timing**: Request duration tracking included
- **Correlation IDs**: For tracking related operations across services

### Service Monitoring
- **Background Services**: Comprehensive lifecycle logging (start, operations, stop)
- **Operation Tracking**: Detailed logging of business operations with counts and metrics
- **External API Monitoring**: Bamboo API integration with rate limiting and error logging
- **Database Operations**: Data access and operation logging with performance indicators

### Error Tracking
- **Global Exception Handling**: Centralized exception middleware with reference IDs
- **Reference IDs**: Unique GUIDs for error correlation and support
- **Comprehensive Logging**: Full exception details with stack traces preserved
- **Security Conscious**: User-friendly error messages, detailed internal logs
- **Service Continuity**: Services continue operating despite individual operation failures

### Security Considerations
- **PII Protection**: No sensitive data exposed in logs
- **Information Sanitization**: Proper masking of credentials and tokens
- **Audit Safety**: Business-relevant events logged without exposure

## DevOps and Deployment Architecture

The Steller platform implements a comprehensive container-based deployment architecture with DevOps best practices.

### Container Orchestration
- **Docker Compose**: Multi-service orchestration for all platform components
- **Multi-stage Builds**: Optimized Docker images with separate build and runtime stages
- **Environment Configuration**: Externalized configuration via environment variables and .env files
- **Service Networks**: Isolated networks for secure inter-service communication

### Deployment Strategy
- **Infrastructure-First**: PostgreSQL and RabbitMQ deployed as shared external services
- **Service Independence**: Each service (APIs, Dashboards) deployed independently
- **Migration Service**: Separate service for database migrations with Docker Compose profiles
- **Configuration Management**: Environment-specific configurations via Docker Compose overrides

### Build Process
- **Multi-stage Builds**: Optimized Docker builds with dependency caching
- **Layer Caching**: Efficient build caching by copying dependencies first
- **Docker Compose Profiles**: Tool services (migrations) managed via profiles
- **Volume Management**: Log persistence and data volumes for critical services

### Development Workflows
- **Component Isolation**: Individual services can be developed independently
- **Hot Reload**: Support for direct execution with hot-reload during development
- **Environment Parity**: Consistent environments across dev, staging, and production
- **Infrastructure Sharing**: Shared infrastructure services across development instances

## Performance Architecture

The Steller platform implements a performance-optimized event-driven architecture designed to handle external API rate limits while maintaining system responsiveness and reliability.

### Performance Optimization Architecture
- **Event-Driven Design**: Asynchronous processing with RabbitMQ to decouple operations from external API limitations
- **Rate Limiting Integration**: Built-in rate limiter service respecting Bamboo API constraints (2 req/sec for placing orders)
- **Caching Strategy**: IMemoryCache registered but underutilized (significant optimization opportunity)
- **Database Optimization**: EF Core with async patterns for efficient data access
- **Connection Management**: Proper connection pooling for HTTP and database connections

### Current Performance Characteristics
- **Order Acceptance**: Fast (under 200ms) since it only stores in database and publishes to queue
- **Order Processing**: Variable (2-60 seconds) depending on external Bamboo API response
- **Catalog Synchronization**: Runs every 30 minutes via background service
- **Throughput Limit**: Capped at ~120 orders per minute due to external API limits

### Major Performance Constraints
1. **External API Rate Limits**: Bamboo Place Order API limited to 2 requests/second (primary bottleneck)
2. **Message Processing**: Consumer concurrency limited to 5 to respect external limits
3. **Database Query Patterns**: Some N+1 query issues in order processing
4. **No Response Caching**: All data access goes directly to database

## Security Architecture

The Steller platform implements multiple security layers to protect sensitive financial and personal data.

### Authentication and Authorization
- **JWT Bearer Authentication**: Standard JWT token authentication with issuer, audience, and lifetime validation
- **API Key Authentication**: Custom X-Api-Key header validation for partner-specific services
- **Partner Isolation**: All data access filtered by PartnerId to prevent cross-partner data access
- **Role-Based Access Control**: User roles and permissions management

### API Security
- **Rate Limiting**: Built-in rate limiting to prevent abuse and protect external APIs
- **Input Validation**: Comprehensive model validation with Data Annotations
- **Parameter Validation**: Custom validation logic for business rules
- **Secure Headers**: Proper CORS configuration with origin restrictions

### Secrets Management
- **Environment Variables**: All sensitive data stored in environment variables (no hardcoded credentials)
- **Externalized Configuration**: Database credentials, API keys, and JWT settings loaded from .env files
- **Security Concern**: All credentials stored in plain text (critical vulnerability requiring remediation)

### Transport Security
- **HTTPS Enforcement**: Uses UseHttpsRedirection middleware
- **External API Communication**: HTTPS for Bamboo Card Portal API calls
- **Database SSL**: TrustServerCertificate=true (security vulnerability requiring remediation)
- **Message Queue Security**: No SSL/TLS configuration for RabbitMQ (security risk)

### Message Queue Security
- **Authentication**: RabbitMQ connection using username/password authentication
- **Message Content**: Minimal sensitive data in messages (only GUIDs)
- **Access Control**: Queue access limited to internal services
- **Security**: No message encryption implemented

### Data Protection
- **Database Security**: EF Core protections against SQL injection
- **Message Security**: Messages transmitted as JSON with minimal sensitive data
- **Personal Data**: No apparent PII encryption visible in codebase
- **Partner Data**: Proper isolation by PartnerId in all operations

### Potential Security Vulnerabilities Identified
- **Critical**: Plain text credentials in .env files (requires immediate remediation)
- **High**: Default RabbitMQ credentials (guest/guest) - security risk for production
- **High**: Weak JWT signing key with predictable content
- **Medium**: No transport security for internal communications between services
- **Medium**: TrustServerCertificate=true for database connections allows MITM attacks
- **Low**: No explicit container security contexts or root user restrictions

## Message Queue Architecture

The Steller system implements an event-driven architecture using RabbitMQ as the message broker to decouple order acceptance from fulfillment processes.

### Optimization Opportunities
- **Caching Implementation**: Redis caching for brand/catalog data and API responses
- **Database Indexing**: Add indexes on frequently queried fields (Orders.CreateDate, foreign keys)
- **Pagination**: Implement pagination for large dataset endpoints
- **Batch Processing**: Process multiple orders from queue in batches
- **Connection Pooling**: Optimize database and HTTP connection pool sizes

### Scalability Considerations
- **Horizontal Scaling**: Possible but limited by external API constraints
- **Database Scaling**: Single PostgreSQL instance currently (read replicas could be considered)
- **Message Queue**: RabbitMQ handles multiple consumer instances well
- **Background Services**: Single-threaded sync processes are scaling bottlenecks

## Message Queue Architecture

The Steller system implements an event-driven architecture using RabbitMQ as the message broker to decouple order acceptance from fulfillment processes.

## Frontend Dashboards

The Steller platform includes two Vue.js 2.6 dashboards built from the same Vuexy template:

### Steller Admin Dashboard
- **Purpose**: Administrative functions and system management
- **API Endpoint**: Connects to Steller API at port 5091
- **App Name**: "Steller"
- **Features**: Partner management, user administration, system reporting, order tracking
- **Authentication**: JWT tokens stored in secure cookies
- **Technology**: Vue.js 2.6, BootstrapVue, Vuexy template

### Steller Consumer Dashboard
- **Purpose**: Partner/consumer functions and order placement
- **API Endpoint**: Connects to StellerConsumer API at port 5092
- **App Name**: "StellerConsumer"
- **Features**: Product catalog browsing, shopping cart, order placement with API key authentication
- **Special Feature**: "Make Order" flow with product selection and cart management
- **Technology**: Vue.js 2.6, BootstrapVue, Vuexy template

### Common Frontend Architecture
- **Authentication System**: JWT token-based with cookie storage and route guards
- **API Integration**: Axios with interceptors for automatic header attachment
- **State Management**: Vuex for centralized state management
- **Routing**: Vue Router with authentication protection
- **UI Components**: BootstrapVue with custom styling
- **Form Validation**: VeeValidate for user input validation
- **Security**: Secure token handling and partner-specific API key integration

### Key Frontend Flows
1. **Admin Flow**: Authentication → Dashboard → Partner Management → Order Reporting
2. **Consumer Flow**: Authentication → Product Catalog → Cart Management → Order Placement
3. **API Key Flow**: Partner authentication → API key retrieval → Order placement with X-Api-Key header

### Shared Contracts
- **Project**: SharedContracts (.NET 9.0 class library)
- **Purpose**: Defines message contracts for inter-service communication
- **Main Contract**: `OrderCreated(Guid OrderId)` record for RabbitMQ messaging
- **Evolution**: Simplified from previous version (OrderId, ProductName, Quantity) to minimal form
- **Integration**: Referenced by both StellerConsumer API (publisher) and Steller API (consumer)
- **Pattern**: Enables publisher-consumer pattern with loose coupling
- **Serialization**: JSON serialization via MassTransit for RabbitMQ transport
- **Versioning**: Breaking changes require coordinated deployment of both services

### Build and Deployment
- **Build System**: Vue CLI with optimized production builds
- **Containerization**: Docker with nginx for production serving
- **Environment Configuration**: Separate .env files for different deployment targets
- **SPA Configuration**: History mode routing with proper fallback handling

### Configuration and Performance
- **Transport**: MassTransit with RabbitMQ transport adapter
- **Concurrency**: Limited to 5 concurrent message consumers to prevent overwhelming external APIs
- **Retry Policy**: 3 retry attempts with 30-second intervals for handling transient failures
- **Message Persistence**: All messages are persistent for reliability
- **Connection**: Environment-based configuration for host, username, and password

### Security Considerations
- **Current State**: Uses default guest/guest credentials (security risk for production)
- **Encryption**: No SSL/TLS configuration visible (requires implementation)
- **Recommendation**: Implement strong authentication and TLS encryption for production

## Database Schema

### Core Entities

#### 1. Users Table
- **Id** (integer): Primary key, auto-incrementing
- **Name** (text): User's full name
- **Email** (text): User's email address (unique)
- **Phone** (text): User's phone number
- **RoleId** (integer, nullable): Reference to user's role
- **Address** (text): User's address
- **PartnerId** (integer, nullable): Reference to associated partner
- **Password** (text): Hashed password
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references to Users table
- **IsDeleted** (boolean, nullable): Soft delete flag

#### 2. Partners Table
- **Id** (integer): Primary key, auto-incrementing
- **BusinessName** (text): Name of the business partner
- **RegistrationNumber** (text): Business registration number
- **Logo** (text): URL to partner's logo
- **Address** (text): Partner's address
- **Phone** (text): Partner's phone number
- **Email** (text): Partner's email address (unique)
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references
- **IsDeleted** (boolean, nullable): Soft delete flag

#### 3. PartnerPersonContact Table
- **Id** (integer): Primary key, auto-incrementing
- **PartnerId** (integer): Reference to partner
- **Name** (text): Contact person's name
- **Email** (text): Contact person's email
- **Phone** (text): Contact person's phone
- **Position** (text): Contact person's position
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references

#### 4. Orders Table
- **Id** (uuid): Primary key (GUID)
- **OrderNumber** (integer): Sequential order number
- **OrderId** (integer): Internal order ID
- **RequestId** (uuid): Request ID (GUID)
- **PartnerId** (integer): Reference to partner
- **SaleTotal** (numeric(18,4)): Sale total amount
- **Total** (numeric(18,4)): Total amount
- **OrderStatusId** (integer): Reference to order status
- **Status** (text): Current status string
- **CreateDate** (timestamp with time zone): Order creation date
- **ErrorMessage** (text, nullable): Error message if order failed
- **OrderType** (text): Type of order
- **Currency** (text): Currency code
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references

#### 5. OrderItems Table
- **Id** (uuid): Primary key
- **ServerOrderId** (integer): Server order ID
- **OrderId** (uuid): Reference to order
- **BrandCode** (text): Brand code
- **ProductId** (integer): Reference to product
- **SaleAmount** (numeric(18,4)): Sale amount
- **ProductFaceValue** (numeric(18,4)): Product face value
- **Quantity** (integer): Quantity
- **PictureUrl** (text): URL to product picture
- **CountryCode** (text): Country code
- **CurrencyCode** (text): Currency code
- **CurrencyCodeBambo** (text): Bamboo-specific currency code
- **Status** (text): Item status
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references

#### 6. OrderCards Table
- **Id** (uuid): Primary key
- **OrderItemId** (uuid): Reference to order item
- **OrderCardId** (integer): Order card ID
- **SerialNumber** (uuid): Serial number
- **CardCode** (text, nullable): Card code
- **Pin** (text, nullable): PIN (sensitive)
- **ExpirationDate** (timestamp with time zone): Expiration date
- **Status** (text): Card status
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references

#### 7. Products Table
- **Id** (integer): Primary key, auto-incrementing
- **BrandInternalId** (integer): Internal brand ID
- **BrandId** (integer): Reference to brand
- **Name** (text): Product name
- **Description** (text): Product description
- **MinFaceValue** (numeric): Minimum face value
- **MaxFaceValue** (numeric): Maximum face value
- **Count** (integer, nullable): Count available
- **ModifiedDate** (timestamp with time zone): Modification date
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references

#### 8. Brands Table
- **Id** (integer): Primary key, auto-incrementing
- **InternalId** (text, nullable): Internal brand ID
- **Name** (text): Brand name
- **CountryCode** (text): Country code
- **CurrencyCode** (text): Currency code
- **Description** (text, nullable): Brand description
- **Disclaimer** (text, nullable): Disclaimer text
- **RedemptionInstructions** (text, nullable): Redemption instructions
- **Terms** (text, nullable): Terms and conditions
- **LogoUrl** (text, nullable): URL to brand logo
- **ModifiedDate** (timestamp with time zone): Modification date
- **CreatedAt/UpdatedAt** (timestamp with time zone): Timestamps
- **CreatedBy/UpdatedBy** (integer, nullable): Foreign key references

#### 9. Additional Tables
- **ApiClientSecrets**: Stores API client tokens and refresh tokens
- **Categories**: Product categories
- **CommissionTypes**: Commission type definitions
- **Currencies**: Currency definitions
- **OrderStatuses**: Order status definitions
- **Permissions**: System permissions
- **ProductPricingConfigurations**: Product pricing configurations
- **ProductPricings**: Product pricing
- **Roles**: User roles
- **RoleBases**: Base roles
- **TransactionTypes**: Transaction type definitions
- **Wallets**: Wallet information
- **WalletHistories**: Wallet transaction histories
- **WalletTypes**: Wallet type definitions
- **UserPermissions**: User permission assignments
- **UserPermissionGroups**: User permission groups
- **Rates**: Currency exchange rates (from RateTable migration)
- **PasswordResets**: Password reset tokens (from PasswordReset migration)

## Data Flow

### Order Creation Process
1. Partner/consumer submits order via Consumer Dashboard
2. Consumer Dashboard calls StellerConsumer.API's `/api/order` endpoint
3. StellerConsumer.API validates the order and saves to database
4. StellerConsumer.API publishes `OrderCreated` message to RabbitMQ
5. Steller.API consumes the message via `OrderCreatedConsumer`
6. Steller.API processes the order with external Bamboo Card Portal API
7. Order status is updated in the database
8. Progress is tracked and available through both dashboards

### Messaging Pattern
- **Publisher**: StellerConsumer.API publishes `OrderCreated` messages
- **Consumer**: Steller.API consumes `OrderCreated` messages
- **Contract**: `OrderCreated(Guid OrderId)` defined in SharedContracts project
- **Queue Name**: `order-created`
- **Retry Policy**: 3 retries with 30-second intervals
- **Concurrency Limit**: 5 concurrent messages
- **Serialization**: JSON via MassTransit with automatic serialization from publisher to consumer
- **Cross-Service**: Contract enables loose coupling between services with coordinated deployment for breaking changes
- **OrderQueueBackgroundService**: Background service that monitors database for unprocessed orders and publishes them to the queue (every 1 minute)
- **Fallback Processing**: Ensures orders not immediately processed get queued eventually

## Background Services

The Steller platform includes two critical background services that operate continuously to maintain data synchronization and processing:

### BrandBackgroundService
- **Purpose**: Synchronizes brand and category data from external Bamboo API
- **Schedule**: Executes every 30 minutes
- **Operations**:
  - Fetches categories from `/{BaseUrlV2}/categories` endpoint
  - Fetches brands from `/{BaseUrlV1}/catalog` endpoint
  - Updates internal database with new/changed information
- **Rate Limiting**: Respects Bamboo API rate limits through RateLimiterService
- **Error Handling**: Continues operation despite partial failures
- **Data Management**: Uses upsert pattern to update or create records

### OrderQueueBackgroundService
- **Purpose**: Ensures orders are properly queued for processing
- **Schedule**: Executes every 1 minute
- **Operations**:
  - Finds orders from last 24 hours without Bamboo OrderId
  - Queues these orders via `OrderCreated` message to RabbitMQ
  - Fallback to all pending orders if no recent orders found
- **Reliability**: Provides safety net for order processing
- **Message Publishing**: Uses MassTransit IPublishEndpoint for reliable messaging

### Service Architecture
- **Implementation**: Both services inherit from .NET BackgroundService
- **Hosting**: Registered as hosted services in Program.cs
- **Cancellation**: Properly handle cancellation tokens for graceful shutdown
- **Logging**: Comprehensive structured logging with Serilog

## Technical Dependencies

### Backend Technologies
- **Language**: C# (.NET 9.0)
- **Framework**: ASP.NET Core 9.0
- **ORM**: Entity Framework with PostgreSQL provider
- **Message Bus**: MassTransit with RabbitMQ transport
- **Authentication**: JWT Bearer tokens
- **Logging**: Serilog with file and console outputs
- **Object Mapping**: AutoMapper
- **Configuration**: Environment variables and appsettings

### Frontend Technologies
- **Framework**: Vue.js 2.6
- **UI Library**: Bootstrap-Vue
- **Build System**: Vue CLI
- **HTTP Client**: Axios
- **State Management**: Vuex

### Infrastructure
- **Database**: PostgreSQL 18
- **Message Broker**: RabbitMQ 3-management
- **Container Orchestration**: Docker + Docker Compose
- **Reverse Proxy**: NGINX (implied by nginx.conf in dashboards)

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

## Development Patterns
- Repository Pattern with generic base repository
- Service Layer pattern for business logic
- Consumer-Producer pattern using RabbitMQ
- Background services for periodic tasks
- Dependency injection throughout the application
- Separation of concerns with dedicated layers (API, Core, EF, Contracts)

## Monitoring & Observability
- Structured logging with Serilog
- Request tracking and correlation
- Error logging with context
- Console and file-based log outputs

---

This architecture enables scalable, resilient order processing with proper separation of concerns between order acceptance and fulfillment, while maintaining a responsive user experience through asynchronous processing.

## Database Expert Analysis

### Database Creation and Migration Process

The Steller system uses Entity Framework Core with PostgreSQL for database management. The migration strategy includes:

- **20250402192732_Initial.cs**: Creates the core schema with 36 tables
- **20250520150948_RateTable.cs**: Adds exchange rates table
- **20250705104925_PasswordReset.cs**: Adds password reset functionality
- **20251101180000_AddIsDeletedColumn.cs**: Adds soft delete functionality
- **20251206112026_AddIsDeletedToUsers.cs**: Empty migration for deployment purposes

The migrations use PostgreSQL-specific features:
- Identity columns with `NpgsqlValueGenerationStrategy.IdentityByDefaultColumn`
- UUID fields for transactional data to support distributed systems
- Decimal precision `numeric(18,4)` for financial accuracy
- Proper foreign key relationships with cascade deletes where appropriate

### Connection Management and Security

The application implements secure database connection management:

```csharp
// Environment-based connection string construction
var dbHost = Environment.GetEnvironmentVariable("DB_HOST") ?? "localhost";
var dbPort = Environment.GetEnvironmentVariable("DB_PORT") ?? "5432";
var dbName = Environment.GetEnvironmentVariable("DB_NAME") ?? "stellerDB";
var dbUsername = Environment.GetEnvironmentVariable("DB_USERNAME") ?? "postgres";
var dbPassword = Environment.GetEnvironmentVariable("DB_PASSWORD") ?? "password";

var connectionString = $"Host={dbHost};Port={dbPort};Database={dbName};Username={dbUsername};Password={dbPassword};TrustServerCertificate=true;Command Timeout=60;Timeout=30";
```

Security features include:
- Environment-based credential management
- No hardcoded connections in source code
- Connection pooling via EF Core
- Command and connection timeout settings for resiliency

### Data Integrity and Constraints

The database maintains integrity through:
- Comprehensive primary key constraints using both identity and UUID keys appropriately
- Foreign key relationships with proper cascading rules
- Unique constraints on email fields for Users and Partners
- Self-referencing foreign keys for audit trails (CreatedBy/UpdatedBy)
- Check constraints with appropriate data types for financial accuracy

### Performance Characteristics

Optimization strategies implemented:
- Proper indexing on foreign key columns
- Appropriate data types for efficient storage and retrieval
- UUID keys for distributed transaction tracking
- Decimal precision for financial data integrity

Potential improvements:
- Consider table partitioning for large transaction tables (Orders, OrderItems)
- Implement read replicas for reporting queries
- Add query optimization for complex joins

### Security Measures and Sensitive Data Handling

Current security measures:
- Password hashing in Users table
- Soft delete functionality with IsDeleted flags
- Audit fields (CreatedAt, UpdatedAt) on all tables
- User tracking through CreatedBy/UpdatedBy relationships

Security concerns requiring attention:
- Sensitive card data (PINs) stored in plain text in OrderCards table
- API tokens stored in ApiClientSecrets table
- SSL configuration uses `TrustServerCertificate=true` (should be reviewed for production)

### Best Practices Recommendations

1. **Security Enhancements**:
   - Implement field-level encryption for PINs, tokens, and sensitive data
   - Replace `TrustServerCertificate=true` with proper certificate validation in production
   - Add row-level security for multi-tenant data isolation

2. **Performance Optimization**:
   - Consider table partitioning for large transaction tables
   - Implement read replicas for reporting and analytics
   - Add database performance monitoring and alerting

3. **Operational Excellence**:
   - Implement automated backup strategies with point-in-time recovery
   - Create data archival processes for old orders
   - Establish formal change management for database schema updates
   - Add database connectivity health checks

4. **Development Practices**:
   - Implement proper EF Core seeding for essential lookup data
   - Automate sanitization pipeline for development environments
   - Maintain schema documentation alongside code changes