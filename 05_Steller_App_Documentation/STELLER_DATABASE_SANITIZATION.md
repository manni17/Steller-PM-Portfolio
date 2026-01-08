# Steller Database Sanitization Script

This script provides instructions and commands to sanitize the Steller database for development use, replacing sensitive production data with safe, sanitized sample data.

## Why Sanitize the Database?
- Remove sensitive customer information
- Protect API keys, credentials, and personal data
- Provide consistent test data for development
- Ensure compliance with data privacy regulations
- Prevent accidental exposure of production data

## Sanitization Process

### 1. Database Backup (Production)

Before performing any sanitization, ensure you have a backup of the production database:
```bash
# Backup command (adjust host, username, and database name as needed)
pg_dump -h <prod_host> -U <username> -d <prod_db_name> > production_backup.sql
```

### 2. Extract Production Database for Development
```bash
# Copy production data to development database
pg_dump -h <prod_host> -U <prod_user> -d <prod_db_name> | psql -h localhost -U leader -d steller_dB
```

### 3. Apply Sanitization Scripts

Create and run these sanitization scripts to clean sensitive data:

#### 3.1 Sanitize User Data
Based on the actual database schema from migration files:
```sql
-- Sanitize user information (Users table has Name, Email, Phone, Address, Password fields)
UPDATE public."Users"
SET
    "Name" = CONCAT('User', "Id"),
    "Email" = CONCAT('user', "Id", '@example.com'),
    "Phone" = CONCAT('+1555', LPAD("Id"::text, 7, '0')),
    "Address" = CONCAT('Test Address ', "Id"),
    "Password" = '$2a$11$gXHmA35MlCvPz3a2zJdE6uF7v8w9x0y1z2A3B4C5D6E7F8G9H0I1J'
WHERE "Email" NOT LIKE '%example.com';
```

#### 3.2 Sanitize Partner Information
```sql
-- Sanitize partner data (Partners table has Email, Phone, Address, BusinessName, RegistrationNumber)
UPDATE public."Partners"
SET
    "Email" = CONCAT('partner', "Id", '@example.com'),
    "Phone" = CONCAT('+1555', LPAD("Id"::text, 7, '0')),
    "Address" = CONCAT('Test Address ', "Id"),
    "BusinessName" = CONCAT('Test Business ', "Id"),
    "RegistrationNumber" = CONCAT('REG', LPAD("Id"::text, 8, '0'))
WHERE "Email" NOT LIKE '%example.com';
```

#### 3.3 Sanitize Partner Person Contact Information
```sql
-- Sanitize contact information (PartnerPersonContact table has Name, Email, Phone, Position)
UPDATE public."PartnerPersonContact"
SET
    "Name" = CONCAT('Contact', "Id"),
    "Email" = CONCAT('contact', "Id", '@example.com'),
    "Phone" = CONCAT('+1555', LPAD("Id"::text, 7, '0')),
    "Position" = 'Test Position'
WHERE "Email" NOT LIKE '%example.com';
```

#### 3.4 Sanitize API Client Secrets
```sql
-- Sanitize API tokens and refresh tokens (ApiClientSecrets table has Token, RefreshToken fields)
UPDATE public."ApiClientSecrets"
SET
    "Token" = CONCAT('dev_token_', "Id"),
    "RefreshToken" = CONCAT('dev_refresh_', "Id");
```

#### 3.5 Sanitize Orders Information
```sql
-- Sanitize order data (Orders table may have sensitive error messages)
UPDATE public."Orders"
SET
    "ErrorMessage" = NULL  -- Clear error messages that might contain sensitive information
WHERE "ErrorMessage" IS NOT NULL;
```

#### 3.6 Sanitize Order Cards (Sensitive Information)
```sql
-- Sanitize card information (OrderCards table has potentially sensitive card codes and PINs)
UPDATE public."OrderCards"
SET
    "CardCode" = CONCAT('SANITIZED_', "Id"),
    "Pin" = 'SANITIZED_PIN',
    "SerialNumber" = gen_random_uuid();  -- Replace with random UUID
WHERE "Pin" IS NOT NULL;
```

#### 3.7 Sanitize Wallet Information
```sql
-- Sanitize wallet numbers (Wallets table has WalletNumber field)
UPDATE public."Wallets"
SET
    "WalletNumber" = CONCAT('WALLET_SANITIZED_', "Id");
```

#### 3.8 Sanitize Wallet History Information
```sql
-- Sanitize wallet transaction history (WalletHistories table may have transaction details)
UPDATE public."WalletHistories"
SET
    "Description" = CONCAT('Sanitized transaction for testing ', "Id"),
    "OriginalTransactionId" = CONCAT('TX_SANITIZED_', "Id");
```

#### 3.9 Sanitize Brand Information
```sql
-- Sanitize brand descriptions that might contain sensitive information
UPDATE public."Brands"
SET
    "Description" = CONCAT('Sanitized Brand Description ', "Id"),
    "Disclaimer" = 'This is a sanitized disclaimer for development use',
    "RedemptionInstructions" = 'These are sanitized redemption instructions',
    "Terms" = 'These are sanitized terms and conditions',
    "LogoUrl" = 'https://example.com/logo-placeholder.png';
```

#### 3.10 Sanitize Product Information
```sql
-- Sanitize product descriptions that might contain sensitive information
UPDATE public."Products"
SET
    "Description" = CONCAT('Sanitized Product Description ', "Id");
```

#### 3.11 Sanitize Password Reset Tokens
```sql
-- Sanitize if PasswordResets table exists (from PasswordReset migration)
DO $$
BEGIN
    IF EXISTS (SELECT FROM information_schema.tables WHERE table_name = 'passwordresets') THEN
        UPDATE public."PasswordResets"
        SET
            "Email" = CONCAT('reset', "Id", '@example.com'),
            "Token" = CONCAT('reset_token_', "Id");
    END IF;
END $$;
```

### 4. Insert Sample Data

After sanitization, add sample data to ensure the system has proper test data:

#### 4.1 Sample Partners
Based on the actual Partners table schema:
```sql
INSERT INTO public."Partners" (
    "Id", "BusinessName", "RegistrationNumber", "Logo", "Address",
    "Phone", "Email", "CreatedAt", "UpdatedAt"
) VALUES
(1001, 'Sample Partner Inc.', 'REG0001001', 'https://example.com/logo.png', '1001 Sample St', '+15550001001', 'partner1001@example.com', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
(1002, 'Demo Company LLC', 'REG0001002', 'https://example.com/demo-logo.png', '1002 Demo Ave', '+15550001002', 'partner1002@example.com', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

#### 4.2 Sample Orders
Based on the actual Orders table schema:
```sql
INSERT INTO public."Orders" (
    "Id", "OrderNumber", "OrderId", "RequestId", "PartnerId",
    "SaleTotal", "Total", "OrderStatusId", "Status", "CreateDate",
    "ErrorMessage", "OrderType", "Currency", "CreatedAt", "UpdatedAt"
) VALUES
('11111111-1111-1111-1111-111111111111'::uuid, 1001, 2001, '11111111-1111-1111-1111-111111111111'::uuid, 1001, 100.00, 100.00, 1, 'Processing', CURRENT_TIMESTAMP, NULL, 'Standard', 'USD', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
('22222222-2222-2222-2222-222222222222'::uuid, 1002, 2002, '22222222-2222-2222-2222-222222222222'::uuid, 1002, 75.50, 75.50, 2, 'Completed', CURRENT_TIMESTAMP, NULL, 'Standard', 'USD', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

#### 4.3 Sample Products
Based on the actual Products table schema:
```sql
INSERT INTO public."Products" (
    "Id", "BrandInternalId", "BrandId", "Name", "Description",
    "MinFaceValue", "MaxFaceValue", "Count", "ModifiedDate",
    "CreatedAt", "UpdatedAt"
) VALUES
(3001, 1, 1, 'Sample Gift Card', 'Sample gift card product', 25.00, 250.00, 100, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
(3002, 2, 1, 'Demo Card Product', 'Demo card product', 50.00, 500.00, 50, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

#### 4.4 Sample Users
Based on the actual Users table schema:
```sql
INSERT INTO public."Users" (
    "Id", "Name", "Email", "Phone", "Address", "PartnerId",
    "Password", "CreatedAt", "UpdatedAt"
) VALUES
(4001, 'Admin User', 'admin@example.com', '+15550004001', 'Admin Address', 1001, '$2a$11$gXHmA35MlCvPz3a2zJdE6uF7v8w9x0y1z2A3B4C5D6E7F8G9H0I1J', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
(4002, 'Test User', 'user@example.com', '+15550004002', 'Test Address', 1002, '$2a$11$gXHmA35MlCvPz3a2zJdE6uF7v8w9x0y1z2A3B4C5D6E7F8G9H0I1J', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

#### 4.5 Sample Order Statuses
```sql
INSERT INTO public."OrderStatuses" (
    "Id", "Name", "CreatedAt", "UpdatedAt"
) VALUES
(1, 'Processing', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
(2, 'Completed', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
(3, 'Failed', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

#### 4.6 Sample Brands
```sql
INSERT INTO public."Brands" (
    "Id", "InternalId", "Name", "CountryCode", "CurrencyCode",
    "Description", "Disclaimer", "RedemptionInstructions",
    "Terms", "LogoUrl", "ModifiedDate", "CreatedAt", "UpdatedAt"
) VALUES
(1, 'BRAND001', 'Sample Brand', 'US', 'USD', 'Sample brand description', 'Sample disclaimer', 'Sample instructions', 'Sample terms', 'https://example.com/brand-logo.png', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

### 5. Database Sanitization Script (Full Process)

Create a complete sanitization script `sanitize_db.sql`:

```sql
-- Steller Database Sanitization Script
-- This script sanitizes all sensitive data and provides sample data for development

BEGIN;

-- Sanitize Users Table
UPDATE public."Users"
SET 
    "Email" = CONCAT('user', "Id", '@example.com'),
    "FirstName" = CONCAT('User', "Id"),
    "LastName" = 'Test',
    "PhoneNumber" = CONCAT('+1555', LPAD("Id"::text, 7, '0'))
WHERE "Email" NOT LIKE '%example.com';

-- Sanitize Partners Table
UPDATE public."Partners" 
SET 
    "ContactEmail" = CONCAT('partner', "Id", '@example.com'),
    "ContactPhone" = CONCAT('+1555', LPAD("Id"::text, 7, '0')),
    "Address" = CONCAT('Test Address ', "Id"),
    "City" = 'Test City',
    "State" = 'TS',
    "ZipCode" = CONCAT('902', LPAD("Id"::text, 3, '0'))
WHERE "ContactEmail" NOT LIKE '%example.com';

-- Sanitize Orders Table
UPDATE public."Orders"
SET 
    "CustomerEmail" = CONCAT('customer', "Id", '@example.com'),
    "CustomerPhone" = CONCAT('+1555', LPAD(("Id" % 10000)::text, 7, '0')),
    "ShippingAddress" = CONCAT('Test Shipping ', "Id"),
    "BillingAddress" = CONCAT('Test Billing ', "Id");

-- Sanitize Accounts Table
UPDATE public."Accounts"
SET 
    "AccountNumber" = CONCAT('ACC', LPAD("Id"::text, 8, '0')),
    "ExternalAccountId" = CONCAT('EXT', LPAD("Id"::text, 10, '0'));

-- Sanitize API Keys
UPDATE public."ApiKeys"
SET 
    "KeyValue" = CONCAT('dev_key_', "Id");

-- Sanitize Partner Person Contacts
UPDATE public."PartnerPersonContacts"
SET 
    "Email" = CONCAT('contact', "Id", '@example.com'),
    "PhoneNumber" = CONCAT('+1555', LPAD("Id"::text, 7, '0')),
    "FirstName" = CONCAT('Contact', "Id"),
    "LastName" = 'Test'
WHERE "Email" NOT LIKE '%example.com';

-- Insert Sample Data if needed
INSERT INTO public."Partners" ("Id", "Name", "ContactEmail", "ContactPhone", "Address", "City", "State", "ZipCode", "CreatedAt", "UpdatedAt", "IsActive")
SELECT 1001, 'Sample Partner Inc.', 'partner1001@example.com', '+15550001001', '1001 Sample St', 'Sample City', 'CA', '90210', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, true
WHERE NOT EXISTS (SELECT 1 FROM public."Partners" WHERE "Id" = 1001);

INSERT INTO public."Orders" ("Id", "PartnerId", "ExternalOrderId", "Status", "CustomerEmail", "CustomerPhone", "ShippingAddress", "CreatedAt", "UpdatedAt", "TotalAmount", "Currency", "AccountId")
SELECT 2001, 1001, 'ORD001', 'Processing', 'customer2001@example.com', '+15550002001', '2001 Test St', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 100.00, 'USD', 1256
WHERE NOT EXISTS (SELECT 1 FROM public."Orders" WHERE "Id" = 2001);

INSERT INTO public."Users" ("Id", "Email", "PasswordHash", "FirstName", "LastName", "PhoneNumber", "IsEmailConfirmed", "CreatedAt", "UpdatedAt")
SELECT 4001, 'admin@example.com', '$2a$11$gXHmA35MlCvPz3a2zJdE6uF7v8w9x0y1z2A3B4C5D6E7F8G9H0I1J', 'Admin', 'User', '+15550004001', true, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP
WHERE NOT EXISTS (SELECT 1 FROM public."Users" WHERE "Email" = 'admin@example.com');

COMMIT;

-- Verify sanitization
SELECT 
    COUNT(*) as total_users,
    COUNT(*) FILTER (WHERE "Email" LIKE '%example.com') as sanitized_users
FROM public."Users";

SELECT 
    COUNT(*) as total_partners,
    COUNT(*) FILTER (WHERE "ContactEmail" LIKE '%example.com') as sanitized_partners
FROM public."Partners";
```

### 6. Automation Script

Create a shell script `sanitize_db.sh` to automate the process:

```bash
#!/bin/bash
# Steller Database Sanitization Script

# Configuration
DB_HOST=${DB_HOST:-localhost}
DB_PORT=${DB_PORT:-5432}
DB_NAME=${DB_NAME:-steller_dB}
DB_USER=${DB_USER:-leader}

echo "Starting Steller database sanitization..."

# Check if database is accessible
if ! pg_isready -h $DB_HOST -p $DB_PORT -d $DB_NAME -U $DB_USER; then
    echo "Error: Cannot connect to the database. Please ensure PostgreSQL is running."
    exit 1
fi

echo "Connected to database. Proceeding with sanitization..."

# Apply sanitization script
psql -h $DB_HOST -p $DB_PORT -d $DB_NAME -U $DB_USER -f sanitize_db.sql

if [ $? -eq 0 ]; then
    echo "Database sanitization completed successfully!"
    echo "The database now contains sanitized data suitable for development."
else
    echo "Error occurred during database sanitization."
    exit 1
fi

# Verify sanitization
echo "Verifying sanitization results..."
psql -h $DB_HOST -p $DB_PORT -d $DB_NAME -U $DB_USER -c "
SELECT 
    (SELECT COUNT(*) FROM public.\"Users\") as user_count,
    (SELECT COUNT(*) FROM public.\"Partners\") as partner_count,
    (SELECT COUNT(*) FROM public.\"Orders\") as order_count;
"
```

### 7. Integration with Development Setup

Update the development setup process to include database sanitization:

```bash
# 1. Start PostgreSQL infrastructure
cd postgresql
docker-compose up -d

# Wait for PostgreSQL to be ready
sleep 10

# 2. Run sanitization script to prepare development database
psql -h localhost -p 5432 -d steller_dB -U leader -f sanitize_db.sql

# 3. Continue with the rest of the setup...
```

### 8. Security Considerations

#### Primary Security Measures
- **Never** run sanitization scripts on production databases
- Always ensure backups are available before sanitization
- Use environment variables for database credentials
- Store sanitization scripts securely and audit access
- Implement proper access controls to prevent unauthorized sanitization

#### Sensitive Data Handling
Based on the database expert analysis, the Steller database contains several types of sensitive information that require sanitization:

1. **Card Information**:
   - PIN codes stored in the `OrderCards` table (Pin column)
   - Card codes in the `OrderCards` table (CardCode column)
   - Serial numbers in the `OrderCards` table (SerialNumber column)

2. **Authentication Data**:
   - API tokens in the `ApiClientSecrets` table (Token column)
   - Refresh tokens in the `ApiClientSecrets` table (RefreshToken column)
   - Password reset tokens in the `PasswordResets` table (Token column)

3. **Personal Information**:
   - User emails, phones, and addresses in the `Users` table
   - Partner contact information in the `Partners` and `PartnerPersonContact` tables
   - Error messages in the `Orders` table that may contain sensitive information

#### Production Security Recommendations
- **Encryption at Rest**: Implement field-level encryption for sensitive data like PINs
- **SSL Configuration**: Replace `TrustServerCertificate=true` with proper certificate validation
- **Database Auditing**: Enable PostgreSQL audit logging for sensitive data access
- **Row-Level Security**: Implement RLS for multi-tenant data isolation
- **Access Monitoring**: Regular monitoring of database access patterns for anomalies

#### Development Security
- Always use sanitized data for development environments
- Never copy production data without proper sanitization
- Regularly review and update sanitization scripts as the schema evolves
- Implement automated checks to verify sanitization completeness

This sanitization process will ensure that development environments have clean, safe data while maintaining the database structure and relationships needed for proper application testing.