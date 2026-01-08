# Local Development Deployment Guide (Code-Based)

This guide is derived strictly from the analysis of the `Steller` and `StellerConsumer` codebase, specifically `Program.cs`, `docker-compose.yml`, `launchSettings.json`, and `.csproj` files.

## 1. Prerequisites (Inferred from Code)
-   **.NET 9.0 SDK**: Required by `<TargetFramework>net9.0</TargetFramework>` in `.csproj` files.
-   **Docker Desktop**: Required for `postgresql` (v18) and `rabbitmq` (v3-management) containers defined in `docker-compose.yml`.

## 2. Infrastructure Setup
The code expects external services to be running on specific ports.

```bash
# 1. Start Shared Database (PostgreSQL 18)
# Source: postgresql/docker-compose.yml
docker-compose -f postgresql/docker-compose.yml up -d

# 2. Start Message Broker (RabbitMQ)
# Source: rabbitmq/docker-compose.yml
docker-compose -f rabbitmq/docker-compose.yml up -d
```

## 3. Configuration
The application uses `DotNetEnv` to load configuration from `.env` files. You must create these files in the project roots.

**Steller API (`Steller/.env`):**
```ini
DB_HOST=localhost
DB_PORT=5432
DB_NAME=stellerDB
DB_USERNAME=postgres
DB_PASSWORD=password
RABBITMQ_HOST=localhost
JWT_KEY=steller@verysecret!serverkey123
ASPNETCORE_ENVIRONMENT=Development
APP_URL=http://localhost:5091
```

**Steller Consumer API (`StellerConsumer/.env`):**
```ini
DB_HOST=localhost
DB_PORT=5432
DB_NAME=stellerDB
DB_USERNAME=postgres
DB_PASSWORD=password
RABBITMQ_HOST=localhost
JWT_KEY=steller@verysecret!serverkey123
ASPNETCORE_ENVIRONMENT=Development
APP_URL=http://localhost:5092
```

## 4. Database Initialization
The code contains EF Core migrations in `Steller/Steller.EF/Migrations`. You must apply them to the running database.

```bash
# Apply Migrations
cd Steller
dotnet ef database update --project ../Steller.EF/Steller.EF.csproj --startup-project ../Steller.Api/Steller.Api.csproj
```
*(Alternatively, use the `steller-migrations` container defined in `Steller/docker-compose.yml` if you prefer Docker-based migration).*

## 5. Running the Services
The `launchSettings.json` files define the local execution profiles.

**Backend API (Port 5091):**
```bash
cd Steller/Steller.Api
dotnet run --launch-profile http
```
*   **Swagger UI**: `http://localhost:5091/swagger`
*   **Health Check**: Code references `Steller.Api.Controllers`

**Consumer API (Port 5092):**
```bash
cd StellerConsumer/StellerConsumer.Api
dotnet run --launch-profile http
```
*   **Swagger UI**: `http://localhost:5092/swagger`

## 6. Verification (Code-Based)
-   **Dependencies**: Ensure `SharedContracts` is compiled (`dotnet build SharedContracts`).
-   **Authentication**: The `Program.cs` enforces `JwtBearer` auth. You will need to generate a token using the `/Account/Login` endpoint (visible in `AccountController.cs`) to access protected routes.
-   **Async Messaging**: Order placement (`OrderController.cs`) publishes to RabbitMQ. Ensure the `order-created` queue is visible in RabbitMQ Management (`http://localhost:15672`).
