# Steller Development Setup Guide

This guide provides instructions for setting up the Steller system on your local development environment.

## Prerequisites

### System Requirements
- Windows 10/11, macOS, or Linux
- Docker Desktop (with Docker Compose)
- .NET 9.0 SDK
- Node.js 16+ (for dashboard development)
- Git

### Required Software
1. **Docker Desktop**: Download from [Docker website](https://www.docker.com/products/docker-desktop)
2. **.NET 9.0 SDK**: Download from [Microsoft website](https://dotnet.microsoft.com/download/dotnet/9.0)
3. **Node.js**: Download from [Node.js website](https://nodejs.org/)
4. **Git**: Download from [Git website](https://git-scm.com/)

## Getting Started

### 1. Clone the Repository
```bash
git clone <your-repository-url>
cd <repository-root>
```

### 2. Project Structure
The Steller system consists of multiple components:

```
Steller/
├── Steller.Api/           # Main API service (producer)
├── StellerConsumer/       # Consumer service
├── steller-admin-dashboard/    # Admin dashboard
├── steller-consumer-dashboard/ # Consumer dashboard
├── postgresql/           # PostgreSQL configuration
├── rabbitmq/             # RabbitMQ configuration
└── SharedContracts/      # Shared data contracts
```

## Infrastructure Setup

### 1. Start PostgreSQL Service
```bash
cd postgresql
docker-compose up -d
```

### 2. Start RabbitMQ Service
```bash
cd rabbitmq
docker-compose up -d
```

## Backend Services Setup

### 1. Steller API (Producer Service)

#### Environment Variables
Create a `.env` file in the `Steller/` directory:
```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=steller_dB
DB_USERNAME=leader
DB_PASSWORD=amokashi@2030
EXTERNAL_API_BASE_URL_V1=https://api.bamboocardportal.com/api/integration/v1.0
EXTERNAL_API_BASE_URL_V2=https://api.bamboocardportal.com/api/integration/v2.0
EXTERNAL_API_CLIENT_ID=STELLER-TECHNOLOGY-FOR-COMMUNICATIONS-INFORMATION---CLIENT-SANDBOX
EXTERNAL_API_CLIENT_SECRET=c0e00ff8-8058-4968-9e57-796c851b6f25
JWT_KEY=steller@verysecret!serverkey123 , this is a custom Secret key for authentication
JWT_ISSUER=Steller Dashboard Management
JWT_AUDIENCE=Steller Admins
RABBITMQ_HOST=localhost
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest
EMAIL_SMTP_SERVER=smtp.gmail.com
EMAIL_SMTP_PORT=587
EMAIL_SENDER_EMAIL=osman.ahmed.dev@gmail.com
EMAIL_SENDER_PASSWORD=hjvj buwc qhdu omsa
EMAIL_SENDER_NAME=Steller
EMAIL_ENABLE_SSL=true
APP_URL=http://localhost:8080/reset-password
```

#### Run the Service
```bash
cd Steller
# Using Docker
docker-compose -f docker-compose.dev.yml up -d

# Or run directly with .NET (for development)
cd Steller.Api
dotnet run
```

### 2. StellerConsumer API (Consumer Service)

#### Environment Variables
Create a `.env` file in the `StellerConsumer/` directory:
```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=steller_dB
DB_USERNAME=leader
DB_PASSWORD=amokashi@2030
EXTERNAL_API_BASE_URL_V1=https://api.bamboocardportal.com/api/integration/v1.0
EXTERNAL_API_BASE_URL_V2=https://api.bamboocardportal.com/api/integration/v2.0
EXTERNAL_API_CLIENT_ID=STELLER-TECHNOLOGY-FOR-COMMUNICATIONS-INFORMATION---CLIENT-SANDBOX
EXTERNAL_API_CLIENT_SECRET=0fea3fab-21eb-4350-b222-6f95911384f7
JWT_KEY=steller@verysecret!serverkey123 , this is a custom Secret key for authentication
JWT_ISSUER=Steller Dashboard Management
JWT_AUDIENCE=Steller Admins
RABBITMQ_HOST=localhost
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest
EMAIL_SMTP_SERVER=smtp.gmail.com
EMAIL_SMTP_PORT=587
EMAIL_SENDER_EMAIL=osman.ahmed.dev@gmail.com
EMAIL_SENDER_PASSWORD=hjvj buwc qhdu omsa
EMAIL_SENDER_NAME=Steller
EMAIL_ENABLE_SSL=true
APP_URL=http://localhost:8080/reset-password
```

#### Run the Service
```bash
cd StellerConsumer
# Using Docker
docker-compose up -d

# Or run directly with .NET (for development)
cd StellerConsumer.Api
dotnet run
```

## Frontend Dashboards Setup

### 1. Admin Dashboard

#### Install Dependencies
```bash
cd steller-admin-dashboard
npm install
```

#### Environment Variables
Create a `.env` file in the `steller-admin-dashboard/` directory:
```env
# Add any specific environment variables if needed
# Most configuration is handled in the Vue app itself
```

#### Run the Development Server
```bash
# Development server with hot reload
npm run serve

# Build for production
npm run build
```

#### Using Docker
```bash
# From the steller-admin-dashboard directory
docker-compose up -d
```

### 2. Consumer Dashboard

#### Install Dependencies
```bash
cd steller-consumer-dashboard
npm install
```

#### Environment Variables
Create a `.env` file in the `steller-consumer-dashboard/` directory:
```env
# Add any specific environment variables if needed
```

#### Run the Development Server
```bash
# Development server with hot reload
npm run serve

# Build for production
npm run build
```

#### Using Docker
```bash
# From the steller-consumer-dashboard directory
docker-compose up -d
```

## Complete System Startup

### Using Docker Compose (Recommended)
To start the entire system with one command:

#### Start Infrastructure
```bash
# Terminal 1 - Start PostgreSQL
cd postgresql
docker-compose up -d

# Wait for PostgreSQL to be ready
sleep 10

# Terminal 2 - Start RabbitMQ
cd rabbitmq
docker-compose up -d

# Wait for RabbitMQ to be ready
sleep 10
```

#### Database Sanitization (For Development)
```bash
# Terminal 3 - Apply database sanitization for development data
# If you have production data, apply sanitization:
# psql -h localhost -p 5432 -d steller_dB -U leader -f STELLER_DATABASE_SANITIZATION.sql

# Or run the sanitization script to create clean sample data:
# (This assumes you have the sanitize_db.sql script from the sanitization documentation)
# psql -h localhost -p 5432 -d steller_dB -U leader -f sanitize_db.sql

# For initial setup with sample data, you can also create the sanitized tables directly:
# 1. First, you would need to run migrations (if not already done):
# cd Steller
# docker-compose -f docker-compose.yml --profile tools up steller-migrations

# 2. Then apply the sanitization script (create sanitize_db.sql first based on the documentation)
# psql -h localhost -p 5432 -d steller_dB -U leader -f sanitize_db.sql
```

#### Start Backend Services
```bash
# Terminal 4 - Start Steller API
cd Steller
docker-compose -f docker-compose.dev.yml up -d

# Terminal 5 - Start Steller Consumer
cd StellerConsumer
docker-compose up -d
```

#### Start Frontend Dashboards
```bash
# Terminal 6 - Start Admin Dashboard
cd steller-admin-dashboard
docker-compose up -d

# Terminal 7 - Start Consumer Dashboard
cd steller-consumer-dashboard
docker-compose up -d
```

### Alternative: Development Mode with Live Reload
For active development with live reload:

```bash
# 1. Start infrastructure (PostgreSQL and RabbitMQ) with Docker
cd postgresql
docker-compose up -d

cd ../rabbitmq
docker-compose up -d

# 2. Run backend services directly with .NET (for debugging)
# Terminal 1: Run Steller API directly
cd Steller/Steller.Api
dotnet run

# Terminal 2: Run StellerConsumer API directly
cd StellerConsumer/StellerConsumer.Api
dotnet run

# 3. Run frontend dashboards with hot reload
# Terminal 3: Admin dashboard
cd steller-admin-dashboard
npm run serve

# Terminal 4: Consumer dashboard
cd steller-consumer-dashboard
npm run serve
```

## Service URLs

After starting the services, the following URLs will be available:

- **Steller API**: http://localhost:5091/swagger
- **StellerConsumer API**: http://localhost:5092/swagger
- **Admin Dashboard**: http://localhost:8080
- **Consumer Dashboard**: http://localhost:8081 (if configured separately) or use same port as admin
- **RabbitMQ Management**: http://localhost:15672 (guest/guest)
- **PostgreSQL**: localhost:5432 (connect using your preferred DB client)

## RabbitMQ Configuration for Development

### Default Settings
- **Host**: localhost (or shared_rabbitmq in Docker)
- **Username**: guest
- **Password**: guest
- **Port**: 5672 (AMQP), 15672 (Management UI)

### Message Flow
1. **Publisher**: StellerConsumer.API publishes `OrderCreated` messages
2. **Queue**: Messages stored in `order-created` queue
3. **Consumer**: Steller.API consumes and processes messages
4. **Concurrency**: Limited to 5 concurrent messages for rate limiting

### Development Best Practices
- Monitor the RabbitMQ management UI for queue depth and performance
- Check consumer processing rates in the logs
- Verify message delivery by examining both application logs
- For debugging, you can inspect messages directly in the management interface
- Use environment variables to configure RabbitMQ settings for different environments

## Database Migrations and Sanitization

### Run Migrations for Steller
```bash
cd Steller
docker-compose -f docker-compose.yml --profile tools up steller-migrations
```

### Run Migrations for StellerConsumer
```bash
cd StellerConsumer
# Manually run migrations or add a migration service to docker-compose.yml
dotnet ef database update --project StellerConsumer.EF --startup-project StellerConsumer.Api
```

### Database Schema and Migration Best Practices

Based on the database expert analysis, consider these points when working with the Steller database:

#### Migration Management:
- All migrations are stored in the `Steller.EF\Migrations` and `StellerConsumer.EF\Migrations` directories
- Migrations use PostgreSQL-specific features like identity columns and UUID fields
- Current migrations include: Initial schema, Rate table, PasswordReset functionality, and soft delete capabilities

#### Connection Configuration:
- Database connection parameters are loaded from environment variables
- Connection string includes command and connection timeout settings for resiliency
- For development environments, ensure your .env files contain the correct database settings

### Database Sanitization for Development

When working with a copy of production data, it's important to sanitize the database to protect sensitive information. Use the sanitization scripts documented in STELLER_DATABASE_SANITIZATION.md:

1. **Generate a sanitized database backup:**
   ```bash
   # Create sanitized data from production backup
   psql -h localhost -p 5432 -d steller_dB -U leader -f sanitize_db.sql
   ```

2. **For initial development setup with sample data:**
   - First run migrations to create the schema
   - Then apply the sanitization script that includes sample data instead of real data

3. **Verify sanitization:**
   ```bash
   # Check that sensitive data has been replaced
   psql -h localhost -p 5432 -d steller_dB -U leader -c "
   SELECT COUNT(*) as users_count FROM public.\"Users\" WHERE \"Email\" LIKE '%example.com';
   "
   ```

### Database Security Considerations

The database contains sensitive information that requires special attention:

- **Sensitive Card Data**: PIN codes in the `OrderCards` table are stored in plain text and are sanitized by the sanitization script
- **API Tokens**: API client secrets are stored in the `ApiClientSecrets` table
- **SSL Configuration**: The application uses `TrustServerCertificate=true` which should be reviewed for production environments

### Performance Optimization for Development

For optimal development performance:

- Large transaction tables (Orders, OrderItems) are indexed appropriately
- Consider using smaller datasets for development to improve performance
- The sanitization script helps maintain a clean, optimized dataset for development

## Development Workflow

### Backend Development
1. Make changes to the code
2. If using direct .NET execution: The service will auto-reload
3. If using Docker: Rebuild the container with `docker-compose up --build`

### Frontend Development
1. Make changes to Vue components
2. Changes will auto-reload in the browser
3. Use `npm run serve` for development with hot reload

### Environment Configuration
- Use `.env` files for different environments (dev, staging, prod)
- Never commit sensitive information to version control
- Use Docker secrets for production environments

## Troubleshooting

### Common Issues

1. **Port Already in Use**
   - Check if PostgreSQL (5432), RabbitMQ (5672, 15672), or API ports (5091, 5092) are busy
   - Use `netstat -an | findstr :<port>` on Windows or `lsof -i :<port>` on macOS/Linux

2. **Database Connection Issues**
   - Ensure PostgreSQL is running
   - Verify credentials in `.env` files match PostgreSQL configuration
   - Check that the database name exists

3. **RabbitMQ Connection Issues**
   - Ensure RabbitMQ is running
   - Verify RabbitMQ credentials in `.env` files
   - Check that services can reach RabbitMQ host

4. **Docker Network Issues**
   - Ensure all services are on the same network
   - Check that external networks (postgres_network, rabbitmq_network) exist
   - Use `docker network ls` to verify network existence

### Useful Commands

```bash
# Check running containers
docker ps

# View container logs
docker logs <container_name>

# Execute commands in containers
docker exec -it <container_name> bash

# Stop all Steller services
docker-compose down

# Clean up volumes (resets data)
docker-compose down -v

# Check network connectivity between services
docker network ls
docker inspect <container_name>
```

## Production Considerations

### Security
- Use strong, unique passwords for all services
- Implement HTTPS in production
- Use environment-appropriate JWT keys
- Enable authentication for RabbitMQ management UI
- Regularly update dependencies

### Performance
- Configure appropriate resource limits for containers
- Monitor database performance and add indexes as needed
- Configure proper RabbitMQ queues and exchanges for scale
- Implement caching where appropriate

### Monitoring
- Set up health checks for all services
- Implement comprehensive logging
- Monitor message queue lengths
- Set up alerts for critical failures