# PowerSuite - New Company Creation Guide

## Overview
This document provides step-by-step instructions and SQL queries for creating a new company in the PowerSuite system, including setting up users, branches, and copying chart of accounts and posting rules from an existing company.

## Prerequisites
- Database access with INSERT and UPDATE privileges
- Existing company code to copy settings from (default: 9876)
- Password hashing method used by the application

## Company Creation Process

### Step 1: Create New Company

Replace the placeholder values with your actual company information:

```sql
INSERT INTO companies (code, name, address, phone, email, licence_no, ntn)
VALUES (
    '1002',        -- Unique company code (e.g., 'ABC123')
    'GTB Travels',   -- Full company name
    'Office # 708 7th Floor Iconic Business Centre, Jamaluddin Afghani Road, BMCHS, Block-3, Sharfabad, Karachi.', -- Company address
    '+92 21 34892650 - 51',    -- Phone number
    'enquire@gtb.com.pk',    -- Email address
    'LIC-2024-001',       -- License number
    'NTN-123456'          -- NTN (Tax number)
);
```

### Step 2: Create Administrative User

Create the first admin user for the new company:

```sql
INSERT INTO users (
    company_code,
    username,
    password,
    user_group_id,
    first_name,
    last_name,
    email,
    branch_id,
    sales_person,
    inactive,
    admin,
    res,
    acct,
    accounts
)
VALUES (
    'COMPANY_CODE',                       -- Same as Step 1
    'admin_username',                     -- Unique username
    'HASHED_PASSWORD',                    -- Use bcrypt or your app's hashing
    NULL,                                 -- User group (optional)
    'Admin',                              -- First name
    'User',                               -- Last name
    'admin@company.com',                  -- Email
    NULL,                                 -- Will be updated after branch creation
    1,                                    -- Sales person (1=yes, 0=no)
    0,                                    -- Inactive (0=active, 1=inactive)
    1,                                    -- Admin privileges
    1,                                    -- Reservation access
    1,                                    -- Accounting access
    1                                     -- Accounts access
);

-- Store the user ID for later use
SET @new_user_id = LAST_INSERT_ID();
```

### Step 3: Create Company Branch

Set up the main branch for the company:

```sql
INSERT INTO branches (
    company_code,
    code,
    name,
    document_prefix,
    location,
    iata_number,
    pcc,
    city_id,
    province_id
)
VALUES (
    'COMPANY_CODE',                       -- Same as Step 1
    'MAIN',                               -- Branch code (e.g., 'MAIN', 'HQ', 'KHI')
    'Head Office',                        -- Branch name
    'HO',                                 -- Document prefix (2-3 chars)
    'Karachi, Pakistan',                  -- Branch location
    '999',                                -- IATA number (if applicable)
    '',                                   -- PCC code (if applicable)
    19754,                                -- City ID from city_codes table
    2                                     -- Province ID from pk_province table
);

-- Store the branch ID and update the user
SET @new_branch_id = LAST_INSERT_ID();
UPDATE users SET branch_id = @new_branch_id WHERE id = @new_user_id;
```

### Step 4: Copy Chart of Accounts

Copy the chart of accounts from an existing company (default: 9876):

```sql
-- This will copy approximately 283 accounts from company 9876
INSERT INTO chart_of_accounts (
    user_id,
    key_account,
    description,
    account_type,
    journal_entry_type_id,
    level,
    type,
    account_number,
    status,
    sub_account_id,
    ar_ap_control,
    disable_sub_account,
    fx_adjustment,
    petty_cash_account
)
SELECT
    @new_user_id,                         -- New user's ID
    key_account,
    description,
    account_type,
    journal_entry_type_id,
    level,
    type,
    account_number,
    status,
    sub_account_id,
    ar_ap_control,
    disable_sub_account,
    fx_adjustment,
    petty_cash_account
FROM chart_of_accounts
WHERE user_id IN (
    SELECT id FROM users WHERE company_code = '9876'  -- Source company
);
```

### Step 5: Copy Posting Rules

Copy posting rules from the existing company:

```sql
-- This will copy approximately 127 posting rules from company 9876
INSERT INTO posting_rules (
    company_code,
    journal_entry_type_id,
    posting_number_type_id,
    posting_number_prefix_id,
    customer_type_id,
    debit_credit,
    service_type_id,
    sub_account_id,
    chart_of_account_id,
    branch_id,
    prefix_code,
    record_type
)
SELECT
    'COMPANY_CODE',                       -- New company code
    journal_entry_type_id,
    posting_number_type_id,
    posting_number_prefix_id,
    customer_type_id,
    debit_credit,
    service_type_id,
    sub_account_id,
    chart_of_account_id,
    @new_branch_id,                       -- New branch ID
    prefix_code,
    record_type
FROM posting_rules
WHERE company_code = '9876';             -- Source company
```

## Complete Transaction Script

Use this complete script with your values. Run it as a single transaction for safety:

```sql
-- ================================================
-- NEW COMPANY CREATION SCRIPT
-- ================================================
-- Replace all placeholder values before executing
-- ================================================

START TRANSACTION;

-- Variables to replace:
SET @new_company_code = 'YOUR_CODE';
SET @new_company_name = 'Your Company Name';
SET @new_company_address = 'Your Address';
SET @new_company_phone = 'Your Phone';
SET @new_company_email = 'company@email.com';
SET @new_company_licence = 'Licence Number';
SET @new_company_ntn = 'NTN Number';

SET @new_username = 'admin_username';
SET @new_password = 'HASHED_PASSWORD_HERE';  -- Must be properly hashed!
SET @new_user_first = 'First';
SET @new_user_last = 'Last';
SET @new_user_email = 'user@email.com';

SET @new_branch_code = 'MAIN';
SET @new_branch_name = 'Main Branch';
SET @new_branch_prefix = 'MB';
SET @new_branch_location = 'Location';
SET @new_branch_iata = '999';
SET @new_branch_pcc = '';
SET @new_branch_city = 19754;  -- City ID
SET @new_branch_province = 2;  -- Province ID

SET @source_company = '9876';  -- Company to copy from

-- Step 1: Create Company
INSERT INTO companies (code, name, address, phone, email, licence_no, ntn)
VALUES (@new_company_code, @new_company_name, @new_company_address,
        @new_company_phone, @new_company_email, @new_company_licence, @new_company_ntn);

-- Step 2: Create User
INSERT INTO users (
    company_code, username, password, first_name, last_name, email,
    sales_person, inactive, admin, res, acct, accounts
)
VALUES (
    @new_company_code, @new_username, @new_password,
    @new_user_first, @new_user_last, @new_user_email,
    1, 0, 1, 1, 1, 1
);
SET @new_user_id = LAST_INSERT_ID();

-- Step 3: Create Branch
INSERT INTO branches (
    company_code, code, name, document_prefix, location,
    iata_number, pcc, city_id, province_id
)
VALUES (
    @new_company_code, @new_branch_code, @new_branch_name, @new_branch_prefix,
    @new_branch_location, @new_branch_iata, @new_branch_pcc,
    @new_branch_city, @new_branch_province
);
SET @new_branch_id = LAST_INSERT_ID();

-- Update user with branch
UPDATE users SET branch_id = @new_branch_id WHERE id = @new_user_id;

-- Step 4: Copy Chart of Accounts
INSERT INTO chart_of_accounts (
    user_id, key_account, description, account_type,
    journal_entry_type_id, level, type, account_number,
    status, sub_account_id, ar_ap_control,
    disable_sub_account, fx_adjustment, petty_cash_account
)
SELECT
    @new_user_id, key_account, description, account_type,
    journal_entry_type_id, level, type, account_number,
    status, sub_account_id, ar_ap_control,
    disable_sub_account, fx_adjustment, petty_cash_account
FROM chart_of_accounts
WHERE user_id IN (SELECT id FROM users WHERE company_code = @source_company);

-- Step 5: Copy Posting Rules
INSERT INTO posting_rules (
    company_code, journal_entry_type_id, posting_number_type_id,
    posting_number_prefix_id, customer_type_id, debit_credit,
    service_type_id, sub_account_id, chart_of_account_id,
    branch_id, prefix_code, record_type
)
SELECT
    @new_company_code, journal_entry_type_id, posting_number_type_id,
    posting_number_prefix_id, customer_type_id, debit_credit,
    service_type_id, sub_account_id, chart_of_account_id,
    @new_branch_id, prefix_code, record_type
FROM posting_rules WHERE company_code = @source_company;

-- Verification
SELECT 'Company' as Entity, COUNT(*) as Count FROM companies WHERE code = @new_company_code
UNION ALL
SELECT 'User', COUNT(*) FROM users WHERE company_code = @new_company_code
UNION ALL
SELECT 'Branch', COUNT(*) FROM branches WHERE company_code = @new_company_code
UNION ALL
SELECT 'Chart of Accounts', COUNT(*) FROM chart_of_accounts WHERE user_id = @new_user_id
UNION ALL
SELECT 'Posting Rules', COUNT(*) FROM posting_rules WHERE company_code = @new_company_code;

COMMIT;
-- Use ROLLBACK; instead if any errors occur
```

## Verification Queries

After creating the company, verify the setup:

```sql
-- Check company creation
SELECT * FROM companies WHERE code = 'COMPANY_CODE';

-- Check user creation
SELECT id, username, company_code, branch_id, admin
FROM users WHERE company_code = 'COMPANY_CODE';

-- Check branch creation
SELECT * FROM branches WHERE company_code = 'COMPANY_CODE';

-- Count copied chart of accounts
SELECT COUNT(*) as total_accounts
FROM chart_of_accounts
WHERE user_id IN (SELECT id FROM users WHERE company_code = 'COMPANY_CODE');

-- Count copied posting rules
SELECT COUNT(*) as total_rules
FROM posting_rules
WHERE company_code = 'COMPANY_CODE';

-- Check specific posting rules by type
SELECT
    jet.name as entry_type,
    COUNT(*) as rule_count
FROM posting_rules pr
LEFT JOIN journal_entry_types jet ON pr.journal_entry_type_id = jet.id
WHERE pr.company_code = 'COMPANY_CODE'
GROUP BY jet.name;
```

## Additional Setup (Optional)

### Add More Users
```sql
INSERT INTO users (
    company_code, username, password, first_name, last_name,
    email, branch_id, sales_person, inactive, admin, res, acct, accounts
)
VALUES (
    'COMPANY_CODE', 'user2', 'HASHED_PWD', 'John', 'Doe',
    'john@company.com', @new_branch_id, 1, 0, 0, 1, 0, 0
);
```

### Add More Branches
```sql
INSERT INTO branches (
    company_code, code, name, document_prefix, location
)
VALUES (
    'COMPANY_CODE', 'BR2', 'Second Branch', 'B2', 'Another Location'
);
```

## Important Notes

1. **Password Security**: Always hash passwords using bcrypt or your application's hashing method. Never store plain text passwords.

2. **Transaction Safety**: Always use transactions when creating companies to ensure data consistency.

3. **Foreign Keys**: Ensure referenced IDs exist:
   - `city_id`: Check `city_codes` table
   - `province_id`: Check `pk_province` table
   - `user_group_id`: Check `user_groups` table

4. **Unique Constraints**: Ensure uniqueness for:
   - Company code
   - Username
   - Branch code within company

5. **Source Company**: The default source company (9876) contains:
   - ~283 chart of accounts entries
   - ~127 posting rules
   - Modify the source company code if copying from a different company

6. **Testing**: Always test in a development environment first before running on production.

## Troubleshooting

### Common Issues

1. **Duplicate Key Error**: Company code or username already exists
   - Solution: Use a unique code/username

2. **Foreign Key Constraint**: Referenced ID doesn't exist
   - Solution: Verify city_id, province_id exist or set to NULL

3. **Password Login Issues**: Password hashing mismatch
   - Solution: Use the same hashing method as your application

4. **Missing Accounts/Rules**: Source company has no data
   - Solution: Verify source company code has data to copy

## Contact Information

For assistance or questions about this process, contact your database administrator or system administrator.

---
*Document Version: 1.0*
*Last Updated: [Current Date]*
*System: PowerSuite Travel Management System*