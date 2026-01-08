# Steller System - Quick Reference Guide

## Overview
Steller is a card order management system with a microservice architecture that integrates with Bamboo Card Portal API.

## Component Architecture
- **Steller.API**: Order processing API (Producer)
- **StellerConsumer.API**: Order fulfillment service (Consumer)  
- **Admin Dashboard**: Administrative interface (Vue.js)
- **Consumer Dashboard**: Partner-facing interface (Vue.js)
- **PostgreSQL**: Data storage
- **RabbitMQ**: Asynchronous message queue
- **SharedContracts**: Data contracts between services

## Core Technology Stack
- Backend: .NET 9.0, ASP.NET Core, Entity Framework
- Frontend: Vue.js 2.6, Bootstrap-Vue
- Database: PostgreSQL
- Message Queue: RabbitMQ with MassTransit
- Containerization: Docker & Docker Compose

## Key Features
- JWT Authentication & Authorization
- Event-driven architecture with RabbitMQ
- Rate limiting for external API calls
- Background processing for data sync
- Email notifications
- Real-time order tracking

## Critical Endpoints
- Steller API: http://localhost:5091/swagger
- StellerConsumer API: http://localhost:5092/swagger
- Admin Dashboard: http://localhost:8080
- RabbitMQ Management: http://localhost:15672

## Development Setup
1. Start PostgreSQL: `cd postgresql && docker-compose up -d`
2. Start RabbitMQ: `cd rabbitmq && docker-compose up -d`
3. Start Steller API: `cd Steller && docker-compose -f docker-compose.dev.yml up -d`
4. Start StellerConsumer: `cd StellerConsumer && docker-compose up -d`
5. Start Dashboards: `cd steller-admin-dashboard && npm run serve`

## Key Configuration Files
- All `.env` files contain sensitive configuration
- Docker Compose files define service orchestration
- Program.cs files contain service configuration
- Controllers define API endpoints

## Order Processing Flow
1. Order created via Consumer Dashboard → StellerConsumer API
2. Order validated and saved to database
3. OrderCreated message published to RabbitMQ queue
4. Steller API consumes message
5. Order processed with Bamboo Card Portal API
6. Results stored and available in dashboards

## External Dependencies
- Bamboo Card Portal API (v1.0 and v2.0)
- SMTP server for email notifications
- PostgreSQL database
- RabbitMQ message broker

## Security Considerations
- JWT-based authentication
- API key validation
- Environment-based configuration
- Credential management via environment variables