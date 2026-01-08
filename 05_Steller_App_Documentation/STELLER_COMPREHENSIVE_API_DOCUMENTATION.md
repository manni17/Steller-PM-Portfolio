# Steller Platform - Comprehensive API Documentation

## IMPORTANT NOTICE

**This is the PRIMARY and AUTHORITATIVE source for all Steller platform API documentation. All other documentation files should be considered supplementary.**

## Overview

This document serves as the central reference for all Steller platform APIs. The platform implements an event-driven architecture with two main services:

1. **StellerConsumer API** - Partner-facing endpoints and order entry point
2. **Steller API** - Backend service for external API integration and order processing

## API Segmentation

### Partner-facing APIs
- StellerConsumer API handles partner authentication, order submission, and data retrieval
- Endpoints available with API key or JWT authentication
- Partner isolation ensures data security

### Administrative APIs
- Steller API provides system-wide admin access for management operations
- JWT authentication required for all admin endpoints
- Full system oversight capabilities

### Internal Services
- RabbitMQ-based messaging system for asynchronous processing
- Background services for external API synchronization

---

## AUTHENTICATION & AUTHORIZATION

### Partner Authentication
Partners can authenticate using:
- **API Key Header**: `X-Api-Key: {partner_api_key}`
- **JWT Bearer Token**: `Authorization: Bearer {jwt_token}`

### Admin Authentication
Administrators use:
- **JWT Bearer Token**: `Authorization: Bearer {jwt_token}`

**Note**: Partner accounts have restricted access and cannot access admin endpoints.

---

## PARTNER API ENDPOINTS

All partner endpoints are accessible through the StellerConsumer API service.

### Authentication

#### POST /api/account/login
Authenticate partner user and receive JWT token

**Description**: Authenticates a partner user and returns a JWT token for subsequent API calls.

**Request Body**:
```json
{
  "email": "partner@steller.com",
  "password": "secure_password"
}
```

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "token": "jwt_token_here",
    "user": {
      "id": 1,
      "name": "Partner User",
      "email": "partner@steller.com",
      "partnerId": 1,
      "roleId": 2
    }
  },
  "message": "Login successful",
  "errorCode": null,
  "statusCode": 200
}
```

### Order Management

#### GET /api/order
Retrieve paginated list of partner's orders

**Description**: Fetches orders for the authenticated partner with optional filtering and pagination.

**Security**: JWT or API Key Required

**Query Parameters**:
- **PageNumber** (integer, optional): Page number (default: 1)
- **PageSize** (integer, optional): Items per page (default: 20)
- **SearchTerm** (string, optional): Search term for filtering
- **StartDate** (string, optional): Filter by start date (ISO 8601)
- **EndDate** (string, optional): Filter by end date (ISO 8601)
- **Status** (string, optional): Filter by order status

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "items": [
      {
        "orderId": "12345678-1234-1234-1234-123456789012",
        "requestId": "12345678-1234-1234-1234-123456789012",
        "partnerId": 1,
        "orderStatusId": 1,
        "status": "Processing",
        "createdDate": "2025-01-01T00:00:00Z",
        "errorMessage": null,
        "orderType": "Standard",
        "currency": "USD",
        "total": 100.00,
        "items": [
          {
            "productId": 1,
            "quantity": 1,
            "productFaceValue": 100.00,
            "saleAmount": 100.00,
            "countryCode": "US",
            "currencyCode": "USD",
            "status": "Pending"
          }
        ]
      }
    ],
    "totalCount": 5,
    "pageSize": 20,
    "currentPage": 1,
    "totalPages": 1,
    "hasNext": false,
    "hasPrevious": false
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### GET /api/order/{orderId}
Get specific order by ID

**Description**: Retrieves a specific order for the authenticated partner.

**Security**: JWT or API Key Required

**Parameters**:
- **orderId** (string): Order ID (GUID format)

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "orderId": "12345678-1234-1234-1234-123456789012",
    "requestId": "12345678-1234-1234-1234-123456789012",
    "partnerId": 1,
    "orderStatusId": 1,
    "status": "Processing",
    "createdDate": "2025-01-01T00:00:00Z",
    "errorMessage": null,
    "orderType": "Standard",
    "currency": "USD",
    "total": 100.00,
    "items": [
      {
        "productId": 1,
        "quantity": 1,
        "productFaceValue": 100.00,
        "saleAmount": 100.00,
        "countryCode": "US",
        "currencyCode": "USD",
        "status": "Pending",
        "cards": [
          {
            "cardCode": "ABC123XYZ789",
            "pin": "****",
            "expirationDate": "2026-01-01T00:00:00Z",
            "status": "Active"
          }
        ]
      }
    ]
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### POST /api/order
Create new order

**Description**: Submits a new order for processing. The order will be queued for asynchronous processing.

**Security**: JWT or API Key Required

**Request Body**:
```json
{
  "items": [
    {
      "productId": 1,
      "quantity": 1,
      "faceValue": 100.00,
      "countryCode": "US",
      "currencyCode": "USD"
    }
  ],
  "currency": "USD",
  "totalAmount": 100.00,
  "partnerReferenceId": "ORD-001-2025"
}
```

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "orderId": "12345678-1234-1234-1234-123456789012",
    "requestId": "12345678-1234-1234-1234-123456789012",
    "status": "Queued",
    "message": "Order submitted successfully and is being processed"
  },
  "message": "Order submitted successfully",
  "errorCode": null,
  "statusCode": 200
}
```

**Response (400 Bad Request)**:
```json
{
  "status": false,
  "data": null,
  "message": "Insufficient funds in wallet",
  "errorCode": "INSUFFICIENT_FUNDS",
  "statusCode": 400
}
```

### Product and Brand Information

#### GET /api/brand
Get available brands and products

**Description**: Retrieves all available brands and their associated products for the authenticated partner.

**Security**: JWT or API Key Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "name": "Sample Brand",
      "internalId": "BRAND001",
      "countryCode": "US",
      "currencyCode": "USD",
      "description": "Brand Description",
      "disclaimer": "Legal Disclaimer",
      "redemptionInstructions": "How to redeem...",
      "terms": "Terms and Conditions",
      "logoUrl": "https://example.com/brand-logo.png",
      "modifiedDate": "2025-01-01T00:00:00Z",
      "createdAt": "2025-01-01T00:00:00Z",
      "updatedAt": "2025-01-01T00:00:00Z",
      "products": [
        {
          "id": 1,
          "name": "Product Name",
          "description": "Product Description",
          "minFaceValue": 10.00,
          "maxFaceValue": 100.00,
          "count": 100,
          "modifiedDate": "2025-01-01T00:00:00Z"
        }
      ]
    }
  ],
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### GET /api/product
Get available products by brand

**Description**: Retrieves products filtered by brand for the authenticated partner.

**Security**: JWT or API Key Required

**Query Parameters**:
- **brandId** (integer, optional): Filter by specific brand ID
- **categoryId** (integer, optional): Filter by specific category ID

**Response (200 OK)**:
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "name": "Product Name",
      "description": "Product Description",
      "minFaceValue": 10.00,
      "maxFaceValue": 100.00,
      "count": 100,
      "brandId": 1,
      "categoryId": 1,
      "modifiedDate": "2025-01-01T00:00:00Z"
    }
  ],
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

### Partner Information and Contact

#### GET /api/partner/{id}
Get specific partner information

**Security**: JWT or API Key Required

**Parameters**:
- **id** (integer): Partner ID (must match authenticated user's partnerId)

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "id": 1,
    "businessName": "Partner Company Inc.",
    "registrationNumber": "REG001",
    "logo": "https://example.com/logo.png",
    "address": "123 Business St",
    "phone": "+15551234567",
    "email": "contact@partner.com",
    "createdAt": "2025-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### GET /api/partnerpersoncontact
Get partner contacts

**Security**: JWT or API Key Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "partnerId": 1,
      "firstName": "John",
      "lastName": "Doe",
      "phone": "+15551234567",
      "email": "john.doe@partner.com",
      "isPrimary": true,
      "position": "Manager",
      "isActive": true,
      "createdAt": "2025-01-01T00:00:00Z",
      "updatedAt": "2025-01-01T00:00:00Z"
    }
  ],
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

### Financial Management

#### GET /api/wallet
Get partner wallet balance

**Description**: Retrieves the current wallet balance for the authenticated partner.

**Security**: JWT or API Key Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "partnerId": 1,
    "availableBalance": 5000.00,
    "reservedBalance": 500.00,
    "currency": "USD",
    "lastUpdated": "2025-01-01T00:00:00Z"
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

### Dashboard

#### GET /api/dashboard
Get partner dashboard metrics

**Description**: Retrieves key metrics and recent activity for the authenticated partner.

**Security**: JWT or API Key Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "totalOrdersThisMonth": 15,
    "successfulOrders": 14,
    "failedOrders": 1,
    "recentOrders": [
      {
        "orderId": "12345678-1234-1234-1234-123456789012",
        "status": "Completed",
        "date": "2025-01-01T00:00:00Z",
        "total": 100.00
      }
    ],
    "walletBalance": {
      "available": 5000.00,
      "currency": "USD"
    },
    "upcomingExpenses": 250.00
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

---

## ADMIN API ENDPOINTS

Administrative endpoints provide system-wide access for platform administrators and management operations.

### Administrative Authentication

#### POST /api/account/login
Authenticate admin user and receive JWT token

**Description**: Authenticates an administrator user and returns a JWT token for subsequent API calls.

**Request Body**:
```json
{
  "email": "admin@steller.com",
  "password": "secure_password"
}
```

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "token": "jwt_token_here",
    "user": {
      "id": 1,
      "name": "Admin User",
      "email": "admin@steller.com",
      "partnerId": null,
      "roleId": 1
    }
  },
  "message": "Login successful",
  "errorCode": null,
  "statusCode": 200
}
```

### Partner Management

#### GET /api/partner
Get all partners (paginated)

**Description**: Retrieves a paginated list of all partners in the system.

**Security**: JWT Admin Required

**Query Parameters**:
- **PageNumber** (integer, optional): Page number (default: 1)
- **PageSize** (integer, optional): Items per page (default: 20)
- **SearchTerm** (string, optional): Search term for filtering
- **Status** (string, optional): Filter by partner status

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "items": [
      {
        "id": 1,
        "businessName": "Partner Company Inc.",
        "registrationNumber": "REG001",
        "logo": "https://example.com/logo.png",
        "address": "123 Business St",
        "phone": "+15551234567",
        "email": "contact@partner.com",
        "status": "Active",
        "createdAt": "2025-01-01T00:00:00Z",
        "updatedAt": "2025-01-01T00:00:00Z"
      }
    ],
    "totalCount": 10,
    "pageSize": 20,
    "currentPage": 1,
    "totalPages": 1,
    "hasNext": false,
    "hasPrevious": false
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### GET /api/partner/{id}
Get specific partner by ID

**Security**: JWT Admin Required

**Parameters**:
- **id** (integer): Partner ID

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "id": 1,
    "businessName": "Partner Company Inc.",
    "registrationNumber": "REG001",
    "logo": "https://example.com/logo.png",
    "address": "123 Business St",
    "phone": "+15551234567",
    "email": "contact@partner.com",
    "status": "Active",
    "walletBalance": 5000.00,
    "currency": "USD",
    "createdAt": "2025-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### POST /api/partner/createdPartner
Create new partner

**Security**: JWT Admin Required

**Request Body**:
```json
{
  "businessName": "New Partner Business",
  "registrationNumber": "NEWREG123",
  "logo": "https://example.com/new-logo.png",
  "address": "456 New Partner Ave",
  "phone": "+15559876543",
  "email": "newpartner@contact.com",
  "status": "Active"
}
```

**Response (201 Created)**:
```json
{
  "status": true,
  "data": {
    "id": 2,
    "businessName": "New Partner Business",
    "registrationNumber": "NEWREG123",
    "logo": "https://example.com/new-logo.png",
    "address": "456 New Partner Ave",
    "phone": "+15559876543",
    "email": "newpartner@contact.com",
    "status": "Active",
    "createdAt": "2025-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  },
  "message": "Partner created successfully",
  "errorCode": null,
  "statusCode": 201
}
```

#### PUT /api/partner/{id}
Update existing partner

**Security**: JWT Admin Required

**Parameters**:
- **id** (integer): Partner ID

**Request Body**:
```json
{
  "businessName": "Updated Partner Business",
  "registrationNumber": "UPDATEREG123",
  "logo": "https://example.com/updated-logo.png",
  "address": "789 Updated Street",
  "phone": "+15555555555",
  "email": "updated@partner.com",
  "status": "Active"
}
```

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "id": 2,
    "businessName": "Updated Partner Business",
    "registrationNumber": "UPDATEREG123",
    "logo": "https://example.com/updated-logo.png",
    "address": "789 Updated Street",
    "phone": "+15555555555",
    "email": "updated@partner.com",
    "status": "Active",
    "createdAt": "2025-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  },
  "message": "Partner updated successfully",
  "errorCode": null,
  "statusCode": 200
}
```

### Brand Management

#### GET /api/brand
Get all brands and their products (admin access)

**Security**: JWT Admin Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "name": "Brand Name",
      "internalId": "BRAND001",
      "countryCode": "US",
      "currencyCode": "USD",
      "description": "Brand description",
      "disclaimer": "Legal disclaimer",
      "redemptionInstructions": "How to redeem...",
      "terms": "Terms and conditions",
      "logoUrl": "https://example.com/brand-logo.png",
      "modifiedDate": "2025-01-01T00:00:00Z",
      "createdAt": "2025-01-01T00:00:00Z",
      "updatedAt": "2025-01-01T00:00:00Z",
      "products": [
        {
          "id": 1,
          "name": "Product Name",
          "description": "Product description",
          "minFaceValue": 10.00,
          "maxFaceValue": 100.00,
          "count": 100,
          "modifiedDate": "2025-01-01T00:00:00Z"
        }
      ]
    }
  ],
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### POST /api/brand
Create new brand

**Security**: JWT Admin Required

**Request Body**:
```json
{
  "name": "New Brand",
  "internalId": "BRAND_NEW",
  "countryCode": "CA",
  "currencyCode": "CAD",
  "description": "New brand description",
  "disclaimer": "Legal disclaimer",
  "redemptionInstructions": "Redemption instructions",
  "terms": "Terms and conditions",
  "logoUrl": "https://example.com/new-brand-logo.png"
}
```

**Response (201 Created)**:
```json
{
  "status": true,
  "data": {
    "id": 2,
    "name": "New Brand",
    "internalId": "BRAND_NEW",
    "countryCode": "CA",
    "currencyCode": "CAD",
    "description": "New brand description",
    "disclaimer": "Legal disclaimer",
    "redemptionInstructions": "Redemption instructions",
    "terms": "Terms and conditions",
    "logoUrl": "https://example.com/new-brand-logo.png",
    "modifiedDate": "2025-01-01T00:00:00Z",
    "createdAt": "2025-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  },
  "message": "Brand created successfully",
  "errorCode": null,
  "statusCode": 201
}
```

### Category Management

#### GET /api/category
Get all categories (admin access)

**Security**: JWT Admin Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "name": "Electronics",
      "description": "Electronic products",
      "brandId": 1,
      "createdAt": "2025-01-01T00:00:00Z",
      "updatedAt": "2025-01-01T00:00:00Z"
    }
  ],
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### POST /api/category
Create new category

**Security**: JWT Admin Required

**Request Body**:
```json
{
  "name": "Clothing",
  "description": "Apparel and accessories",
  "brandId": 1
}
```

**Response (201 Created)**:
```json
{
  "status": true,
  "data": {
    "id": 2,
    "name": "Clothing",
    "description": "Apparel and accessories",
    "brandId": 1,
    "createdAt": "2025-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  },
  "message": "Category created successfully",
  "errorCode": null,
  "statusCode": 201
}
```

### Order Management (Admin)

#### GET /api/order/{orderId}
Get specific order by ID (admin access)

**Security**: JWT Admin Required

**Parameters**:
- **orderId** (string): Order ID (GUID format)

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "orderId": 12345,
    "requestId": "12345678-1234-1234-1234-123456789012",
    "partnerId": 1,
    "partnerName": "Partner Company Inc.",
    "orderStatusId": 1,
    "status": "Processing",
    "createdDate": "2025-01-01T00:00:00Z",
    "errorMessage": null,
    "orderType": "Standard",
    "currency": "USD",
    "total": 100.00,
    "items": [
      {
        "productId": 1,
        "quantity": 1,
        "productFaceValue": 100.00,
        "saleAmount": 100.00,
        "countryCode": "US",
        "currencyCode": "USD",
        "status": "Pending",
        "cards": [
          {
            "cardCode": "ABC123",
            "pin": "hidden_for_security",
            "expirationDate": "2026-01-01T00:00:00Z",
            "status": "Active"
          }
        ]
      }
    ]
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

#### GET /api/order
Get all orders (admin access - paginated)

**Security**: JWT Admin Required

**Query Parameters**:
- **PageNumber** (integer, optional): Page number (default: 1)
- **PageSize** (integer, optional): Items per page (default: 20)
- **SearchTerm** (string, optional): Search term for filtering
- **StartDate** (string, optional): Filter by start date (ISO 8601)
- **EndDate** (string, optional): Filter by end date (ISO 8601)
- **Status** (string, optional): Filter by order status
- **PartnerId** (integer, optional): Filter by specific partner

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "items": [
      {
        "orderId": 12345,
        "requestId": "12345678-1234-1234-1234-123456789012",
        "partnerId": 1,
        "partnerName": "Partner Company Inc.",
        "orderStatusId": 1,
        "status": "Processing",
        "createdDate": "2025-01-01T00:00:00Z",
        "errorMessage": null,
        "orderType": "Standard",
        "currency": "USD",
        "total": 100.00
      }
    ],
    "totalCount": 150,
    "pageSize": 20,
    "currentPage": 1,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": false
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

### Administrative Dashboard

#### GET /api/dashboard
Get administrative dashboard metrics

**Security**: JWT Admin Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "totalPartners": 10,
    "totalOrdersToday": 5,
    "totalOrdersThisMonth": 150,
    "successfulOrders": 145,
    "failedOrders": 5,
    "walletBalances": [
      {
        "partnerId": 1,
        "partnerName": "Partner Company Inc.",
        "availableBalance": 1000.00,
        "currency": "USD"
      }
    ],
    "systemMetrics": {
      "messageQueueDepth": 12,
      "averageProcessingTime": 2.5,
      "apiResponseTime": 120
    }
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

### System Configuration

#### GET /api/system/config
Get system configuration

**Security**: JWT Admin Required

**Response (200 OK)**:
```json
{
  "status": true,
  "data": {
    "version": "1.0.0",
    "environment": "production",
    "rabbitmq": {
      "host": "rabbitmq.host",
      "port": 5672,
      "queues": {
        "order_created": "order-created",
        "order_status_updated": "order-status-updated"
      }
    },
    "externalApi": {
      "baseUrl": "https://api.bamboocard.com"
    },
    "rateLimits": {
      "placeOrder": "2 per second",
      "getOrder": "120 per minute",
      "catalogSync": "2 per hour"
    }
  },
  "message": null,
  "errorCode": null,
  "statusCode": 200
}
```

---

## ARCHITECTURE & IMPLEMENTATION

### Event-Driven Architecture
The Steller platform implements an event-driven architecture where:
- Partners submit orders to StellerConsumer API
- StellerConsumer API validates and stores orders in the database
- StellerConsumer API publishes messages to RabbitMQ
- Steller API consumes messages and processes orders with external systems
- Status updates flow back to the database for partner consumption

### Message Publishing Implementation
The StellerConsumer API implements a publisher-only architecture:

```csharp
// Program.cs - MassTransit Configuration
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

### Message Consumption Implementation
The Steller API implements a consumer-only architecture:

```csharp
// Program.cs - MassTransit Configuration
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();

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

### Rate Limiting Implementation
The system implements advanced rate limiting for Bamboo API endpoints:

```csharp
public RateLimiterService()
{
    // Configure rate limits based on Bamboo API documentation
    _rateLimits["bamboo_place_order"] = (2, TimeSpan.FromSeconds(1)); // 2 requests per second
    _rateLimits["bamboo_get_order"] = (120, TimeSpan.FromMinutes(1)); // 120 requests per minute
    _rateLimits["bamboo_catalog"] = (2, TimeSpan.FromHours(1)); // 2 requests per hour
    _rateLimits["bamboo_exchange_rate"] = (20, TimeSpan.FromMinutes(1)); // 20 requests per minute
    _rateLimits["bamboo_transactions"] = (4, TimeSpan.FromHours(1)); // 4 requests per hour
    _rateLimits["bamboo_other"] = (1, TimeSpan.FromSeconds(1)); // 1 request per second for other APIs
}
```

---

## SECURITY CONSIDERATIONS

### Authentication and Authorization
- JWT tokens expire after 24 hours by default
- API keys provide alternative authentication path for partners
- Role-based access controls ensure admin endpoints are not accessible to partners
- Partner isolation prevents access to other partners' data

### Rate Limiting
- Administrative endpoints: Not subject to external API rate limits
- Partner endpoints: Protected by system-wide rate limiting to prevent abuse
- External API calls: Rate limited per Bamboo Card Portal specifications

### Data Protection
- Sensitive data like card pins are masked in API responses
- All API traffic should be served over HTTPS
- Database connections encrypted with SSL
- Credential storage follows industry best practices

## RESPONSE FORMAT

All API responses follow a consistent format:

```json
{
  "status": true/false,           // Indicates success (true) or failure (false)
  "data": {...},                  // Response payload (null if error)
  "message": "Success message",   // Human-readable message
  "errorCode": "ERROR_CODE",      // Machine-readable error code (null if success)
  "statusCode": 200              // HTTP status code
}
```

## ERROR CODES

Common error codes returned by the API:

| Error Code | Description |
|------------|-------------|
| INVALID_CREDENTIALS | Provided credentials are invalid |
| INSUFFICIENT_FUNDS | Wallet balance insufficient for the operation |
| ORDER_NOT_FOUND | Requested order does not exist |
| ACCESS_DENIED | Insufficient privileges for the requested operation |
| VALIDATION_ERROR | Request data validation failed |
| SYSTEM_ERROR | Unexpected system error occurred |

## VERSIONING

The Steller platform follows semantic versioning. API endpoints are versioned as part of the URL:

- Current version: `/api/v1/` (default)
- Planned future versioning: `/api/v2/` for major changes

## HEALTH CHECKS

For testing purposes, the system provides health check endpoints:

#### GET /health
System health check

**Response (200 OK)**:
```json
{
  "status": "Healthy",
  "checks": [
    {
      "name": "Database Check",
      "status": "Healthy"
    },
    {
      "name": "RabbitMQ Check",
      "status": "Healthy"
    }
  ]
}
```
