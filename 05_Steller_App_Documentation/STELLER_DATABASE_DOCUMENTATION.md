# Steller Database Documentation

This document provides comprehensive information about the Steller database architecture, schema, and best practices based on a complete database expert analysis.

## Database Overview

The Steller system uses PostgreSQL as its primary database with Entity Framework Core as the ORM. The database is designed to support a card ordering and management platform with multi-tenant capabilities.

### Database Version and Technology Stack
- **Database**: PostgreSQL 18
- **ORM**: Entity Framework Core with Npgsql provider
- **Connection**: Connection pooling with timeout configuration
- **Migration**: EF Core migrations for schema management

## Complete Schema Reference

### Core Transaction Tables

#### 1. Orders Table
- **Id** (uuid): Primary key, globally unique for distributed systems
- **OrderNumber** (integer): Sequential order number for business reference
- **OrderId** (integer): Internal order ID
- **RequestId** (uuid): Request tracking ID
- **PartnerId** (integer): Reference to partner placing the order
- **SaleTotal** (numeric(18,4)): Total sale amount with precision
- **Total** (numeric(18,4)): Order total amount
- **OrderStatusId** (integer): Current status reference
- **Status** (text): Human-readable status
- **CreateDate** (timestamp with time zone): Order creation time
- **ErrorMessage** (text, nullable): Error details if order failed
- **OrderType** (text): Type classification
- **Currency** (text): Currency code
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

#### 2. OrderItems Table
- **Id** (uuid): Primary key for distributed tracking
- **ServerOrderId** (integer): Server-specific order ID
- **OrderId** (uuid): Reference to parent order
- **BrandCode** (text): Brand identifier
- **ProductId** (integer): Reference to product
- **SaleAmount** (numeric(18,4)): Sale amount for item
- **ProductFaceValue** (numeric(18,4)): Card face value
- **Quantity** (integer): Quantity ordered
- **PictureUrl** (text): Product image URL
- **CountryCode** (text): Country for transaction
- **CurrencyCode** (text): Currency code
- **CurrencyCodeBambo** (text): Bamboo system currency code
- **Status** (text): Item status
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

#### 3. OrderCards Table (High Security Risk)
- **Id** (uuid): Primary key
- **OrderItemId** (uuid): Reference to order item
- **OrderCardId** (integer): Internal card ID
- **SerialNumber** (uuid): Card serial number
- **CardCode** (text, nullable): Card access code
- **Pin** (text, nullable): Card PIN (HIGH SECURITY RISK - stored in plain text)
- **ExpirationDate** (timestamp with time zone): Card expiration
- **Status** (text): Card status
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

### User and Partner Management

#### 4. Users Table
- **Id** (integer): Auto-incrementing primary key
- **Name** (text): User's full name
- **Email** (text): User email (unique)
- **Phone** (text): User phone number
- **RoleId** (integer, nullable): Role reference
- **Address** (text): User address
- **PartnerId** (integer, nullable): Associated partner
- **Password** (text): Hashed password
- **CreatedAt** (timestamp with time zone): Account creation
- **UpdatedAt** (timestamp with time zone): Account update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference
- **IsDeleted** (boolean, nullable): Soft delete flag

#### 5. Partners Table
- **Id** (integer): Auto-incrementing primary key
- **BusinessName** (text): Partner business name
- **RegistrationNumber** (text): Business registration
- **Logo** (text): Logo URL
- **Address** (text): Business address
- **Phone** (text): Business phone
- **Email** (text): Business email (unique)
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference
- **IsDeleted** (boolean, nullable): Soft delete flag

#### 6. PartnerPersonContact Table
- **Id** (integer): Auto-incrementing primary key
- **PartnerId** (integer): Reference to partner
- **Name** (text): Contact person name
- **Email** (text): Contact email
- **Phone** (text): Contact phone
- **Position** (text): Contact position
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

### Product and Brand Management

#### 7. Products Table
- **Id** (integer): Auto-incrementing primary key
- **BrandInternalId** (integer): Brand-specific ID
- **BrandId** (integer): Reference to brand
- **Name** (text): Product name
- **Description** (text): Product description
- **MinFaceValue** (numeric): Minimum face value
- **MaxFaceValue** (numeric): Maximum face value
- **Count** (integer, nullable): Available count
- **ModifiedDate** (timestamp with time zone): Product modification
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

#### 8. Brands Table
- **Id** (integer): Auto-incrementing primary key
- **InternalId** (text, nullable): Internal brand ID
- **Name** (text): Brand name
- **CountryCode** (text): Brand country
- **CurrencyCode** (text): Brand currency
- **Description** (text, nullable): Brand description
- **Disclaimer** (text, nullable): Legal disclaimer
- **RedemptionInstructions** (text, nullable): Redemption details
- **Terms** (text, nullable): Terms and conditions
- **LogoUrl** (text, nullable): Logo URL
- **ModifiedDate** (timestamp with time zone): Modification time
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

### Financial and Wallet Tables

#### 9. Wallets Table
- **Id** (uuid): Primary key for distributed systems
- **PartnerId** (integer): Reference to partner
- **AvailableBalance** (numeric(18,4)): Current balance
- **CurrencyId** (integer): Reference to currency
- **WalletNumber** (text): Wallet identifier
- **WalletTypeId** (integer): Wallet type reference
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

#### 10. WalletHistories Table
- **Id** (uuid): Primary key
- **WalletId** (uuid): Reference to wallet
- **Description** (text): Transaction description
- **OriginalTransactionId** (text): Original transaction reference
- **TransactionTypeId** (integer): Transaction type reference
- **Amount** (numeric(18,4)): Transaction amount
- **BalanceBefore** (numeric(18,4)): Balance before transaction
- **BalanceAfter** (numeric(18,4)): Balance after transaction
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

### Security and Authentication Tables

#### 11. ApiClientSecrets Table
- **Id** (integer): Auto-incrementing primary key
- **PartnerId** (integer): Reference to partner
- **Token** (text): API access token (SECURITY RISK - stored in plain text)
- **RefreshToken** (text): API refresh token (SECURITY RISK - stored in plain text)
- **ExpireDate** (timestamp with time zone): Token expiration
- **IsRevoked** (boolean): Token revocation status
- **CreatedAt** (timestamp with time zone): Record creation
- **UpdatedAt** (timestamp with time zone): Record update
- **CreatedBy** (integer, nullable): Creator reference
- **UpdatedBy** (integer, nullable): Updater reference

#### 12. PasswordResets Table
- **Id** (integer): Auto-incrementing primary key
- **Email** (character varying(100)): User email
- **Token** (character varying(100)): Reset token (SECURITY RISK - stored in plain text)
- **Created_at** (timestamp with time zone): Token creation

## Relationship Diagram

The database follows a normalized design with the following key relationships:

- `Users` ↔ `Partners` (Users can belong to a Partner)
- `Orders` → `Partners` (Orders are placed by Partners)
- `Orders` ↔ `OrderStatuses` (Orders have a status)
- `OrderItems` → `Orders` (OrderItems belong to an Order) [CASCADE DELETE]
- `OrderItems` → `Products` (OrderItems reference a Product) [CASCADE DELETE]
- `OrderCards` → `OrderItems` (OrderCards belong to an OrderItem) [CASCADE DELETE]
- `Products` → `Brands` (Products belong to a Brand) [CASCADE DELETE]
- `Wallets` → `Partners` (Wallets belong to a Partner) [CASCADE DELETE]
- `WalletHistories` → `Wallets` (WalletHistories belong to a Wallet) [CASCADE DELETE]

## Security Considerations

### High-Risk Data
The following fields contain sensitive information:
- `OrderCards.Pin`: Card PIN codes stored in plain text
- `OrderCards.CardCode`: Card access codes
- `ApiClientSecrets.Token`: API access tokens
- `ApiClientSecrets.RefreshToken`: API refresh tokens
- `PasswordResets.Token`: Password reset tokens
- Personal information in `Users`, `Partners`, and `PartnerPersonContact` tables

### Security Recommendations
1. **Immediate Action Required**: Implement encryption at rest for sensitive fields
2. **Token Management**: Use proper token hashing instead of plain text storage
3. **SSL Configuration**: Review `TrustServerCertificate=true` setting for production
4. **Access Controls**: Implement database-level row-level security
5. **Audit Logging**: Enable PostgreSQL audit logging for sensitive operations

## Performance Optimization

### Indexes
- Foreign key columns are automatically indexed by EF Core
- Email fields on `Users` and `Partners` have unique indexes
- Self-referencing foreign keys (CreatedBy/UpdatedBy) are indexed

### Optimization Recommendations
- Consider partitioning `Orders` and `OrderItems` tables by date
- Implement read replicas for reporting queries
- Monitor query performance for complex joins
- Regular database maintenance and statistics updates

## Migration Strategy

### Current Migration History
1. **Initial Migration** (2025-04-02): Complete schema creation (36 tables)
2. **Rate Table** (2025-05-20): Added currency exchange rates
3. **Password Reset** (2025-07-05): Added password reset functionality
4. **Soft Delete** (2025-11-01): Added IsDeleted columns to Users and Partners
5. **Maintenance** (2025-12-06): Empty migration for deployment purposes

### Migration Best Practices
- All migrations are wrapped in transactions
- Proper rollback procedures are implemented
- Foreign key constraints are maintained during schema changes
- Data migration is handled within each migration where needed

## Connection and Configuration

### Connection String Format
The application constructs connection strings from environment variables:
```
Host={DB_HOST};Port={DB_PORT};Database={DB_NAME};Username={DB_USERNAME};Password={DB_PASSWORD};TrustServerCertificate=true;Command Timeout=60;Timeout=30
```

### Connection Management
- EF Core manages connection pooling automatically
- Command timeout set to 60 seconds for complex operations
- Connection timeout set to 30 seconds for resiliency
- Environment-based configuration prevents hardcoded credentials

## Development and Testing

### Seeding Strategy
- No built-in EF Core seeding in migrations
- Runtime seeding handled by application logic
- Development data created through sanitization script
- Sample data created for development environments only

### Testing Considerations
- Use sanitized data for all development environments
- Never copy production data without sanitization
- Regular verification of sanitization effectiveness
- Automated testing with clean, consistent datasets