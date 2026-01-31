# 6. Security & Governance

## Security Overview

Snowflake implements defense-in-depth security: network isolation, authentication (MFA/SSO), role-based access control (RBAC), encryption (at-rest and in-transit), and data governance (masking, row policies). All security features are enforced at the cloud services layer, preventing unauthorized access regardless of client tool.

```
Security Layers:
┌────────────────────────────────────────────────┐
│ Network Security: VPC, PrivateLink, IP         │
├────────────────────────────────────────────────┤
│ Authentication: MFA, SSO (SAML/OAuth)          │
├────────────────────────────────────────────────┤
│ Authorization: RBAC (Roles & Privileges)       │
├────────────────────────────────────────────────┤
│ Data Protection: Masking, Row Policies         │
├────────────────────────────────────────────────┤
│ Encryption: AES-256 (rest), TLS 1.2 (transit)  │
├────────────────────────────────────────────────┤
│ Audit & Compliance: Query history, access logs │
└────────────────────────────────────────────────┘
```

---

## Role-Based Access Control (RBAC)

Snowflake uses roles (collections of privileges) assigned to users. Roles can inherit from other roles creating a hierarchy. Best practice: grant least privilege, separate duties (ETL_ROLE, BI_ROLE, ADMIN_ROLE).

### System Roles

| Role | Purpose | Default Privileges |
|------|---------|-------------------|
| `ACCOUNTADMIN` | Account management | All privileges, create users/roles, billing |
| `SECURITYADMIN` | User/role management | CREATE USER, CREATE ROLE, GRANT ROLE |
| `SYSADMIN` | Object management | CREATE WAREHOUSE, CREATE DATABASE, all objects |
| `USERADMIN` | User/role creation | CREATE USER, CREATE ROLE (cannot grant roles) |
| `PUBLIC` | Default role | Minimal privileges, granted to all users |

### Role Hierarchy Diagram

```
        ACCOUNTADMIN (Top Level - All Privileges)
               ↓
        SECURITYADMIN (User/Role Management)
           ↙       ↘
    SYSADMIN       USERADMIN
  (Objects)        (Users Only)
       ↓
   Custom Roles (ETL_ROLE, BI_ROLE, ANALYST_ROLE)
       ↓
     PUBLIC (Minimal Access)
```

---

## Syntax: CREATE ROLE

### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] ROLE [ IF NOT EXISTS ] <role_name>
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ];
```

### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `role_name` | String | Required | Role identifier | `DATA_ENGINEER` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (team='engineering')` |
| `COMMENT` | String | NULL | Role description | `'Data engineering team role'` |

### Basic Example

```sql
-- Create custom role
CREATE ROLE analyst_role
  COMMENT = 'Analysts with read-only access to production data';

-- Grant role to user
GRANT ROLE analyst_role TO USER john_doe;

-- Use role
USE ROLE analyst_role;

-- Check current role
SELECT CURRENT_ROLE();
```

### Advanced Example (SDLC: Multi-Environment RBAC)

```sql
-- Admin roles
CREATE ROLE data_admin COMMENT = 'Data platform administrators';
GRANT ROLE data_admin TO ROLE SYSADMIN;

-- Environment-specific roles
CREATE ROLE prod_read_only COMMENT = 'Production read-only access';
CREATE ROLE dev_full_access COMMENT = 'Development full access';
CREATE ROLE etl_role COMMENT = 'ETL pipeline service account';

-- Team roles
CREATE ROLE data_engineering_team 
  WITH TAG (department = 'engineering', cost_center = 'data_platform')
  COMMENT = 'Data engineering team members';
  
CREATE ROLE analytics_team
  WITH TAG (department = 'analytics')
  COMMENT = 'Business analytics team';

-- Role hierarchy
GRANT ROLE prod_read_only TO ROLE analytics_team;
GRANT ROLE dev_full_access TO ROLE data_engineering_team;
GRANT ROLE etl_role TO ROLE data_engineering_team;

-- Grant teams to admin
GRANT ROLE data_engineering_team TO ROLE data_admin;
GRANT ROLE analytics_team TO ROLE data_admin;
```

---

## Syntax: CREATE USER

### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] USER [ IF NOT EXISTS ] <user_name>
  PASSWORD = '<password>'
  [ LOGIN_NAME = '<login_name>' ]
  [ DISPLAY_NAME = '<display_name>' ]
  [ FIRST_NAME = '<first_name>' ]
  [ LAST_NAME = '<last_name>' ]
  [ EMAIL = '<email>' ]
  [ MUST_CHANGE_PASSWORD = TRUE | FALSE ]
  [ DISABLED = TRUE | FALSE ]
  [ DEFAULT_WAREHOUSE = <warehouse_name> ]
  [ DEFAULT_NAMESPACE = <database_name>.<schema_name> ]
  [ DEFAULT_ROLE = <role_name> ]
  [ DEFAULT_SECONDARY_ROLES = ( 'ALL' | 'NONE' ) ]
  [ MINS_TO_UNLOCK = <num> ]
  [ DAYS_TO_EXPIRY = <num> ]
  [ MINS_TO_BYPASS_MFA = <num> ]
  [ RSA_PUBLIC_KEY = '<key>' ]
  [ RSA_PUBLIC_KEY_2 = '<key>' ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ];
```

### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `PASSWORD` | String | Required | User password (min 8 chars) | `'SecurePass123!'` |
| `LOGIN_NAME` | String | user_name | Login identifier (case-insensitive) | `'john.doe'` |
| `EMAIL` | String | NULL | User email (for notifications) | `'john@company.com'` |
| `MUST_CHANGE_PASSWORD` | Boolean | FALSE | Force password change on first login | `TRUE` |
| `DISABLED` | Boolean | FALSE | Disable user account | `TRUE` |
| `DEFAULT_WAREHOUSE` | String | NULL | Default warehouse for user | `'analyst_wh'` |
| `DEFAULT_NAMESPACE` | String | NULL | Default database.schema | `'prod.analytics'` |
| `DEFAULT_ROLE` | String | PUBLIC | Default role on login | `'analyst_role'` |
| `DAYS_TO_EXPIRY` | Integer | NULL | Password expiration days | `90` |
| `RSA_PUBLIC_KEY` | String | NULL | Public key for key-pair auth | `'MIIBIjAN...'` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (contractor='true')` |
| `COMMENT` | String | NULL | User description | `'Senior Data Analyst'` |

### Basic Example

```sql
-- Create user
CREATE USER john_doe
  PASSWORD = 'TempPass123!'
  MUST_CHANGE_PASSWORD = TRUE
  EMAIL = 'john.doe@company.com'
  DEFAULT_WAREHOUSE = 'analyst_wh'
  DEFAULT_ROLE = 'analyst_role'
  COMMENT = 'Data analyst - hired 2025-01-15';

-- Grant role to user
GRANT ROLE analyst_role TO USER john_doe;

-- Reset password
ALTER USER john_doe SET PASSWORD = 'NewSecurePass456!';
```

### Advanced Example (Service Account for CI/CD)

```sql
-- Create service account with key-pair authentication
CREATE USER cicd_service_account
  LOGIN_NAME = 'svc_cicd'
  RSA_PUBLIC_KEY = 'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...'
  DEFAULT_WAREHOUSE = 'deployment_wh'
  DEFAULT_ROLE = 'etl_role'
  DEFAULT_NAMESPACE = 'prod.etl'
  DISABLED = FALSE
  WITH TAG (
    account_type = 'service',
    managed_by = 'devops_team'
  )
  COMMENT = 'CI/CD service account for automated deployments';

-- Grant necessary roles
GRANT ROLE etl_role TO USER cicd_service_account;
GRANT ROLE sysadmin TO USER cicd_service_account;  -- For object creation

-- Disable password authentication (key-pair only)
ALTER USER cicd_service_account UNSET PASSWORD;
```

---

## Syntax: GRANT (Privileges)

### Complete Syntax Template

```sql
-- Grant privileges on objects
GRANT { 
    ALL [ PRIVILEGES ] 
  | <privilege> [ , <privilege> ... ] 
}
  ON { 
      ACCOUNT
    | DATABASE <db_name> 
    | SCHEMA <schema_name>
    | TABLE <table_name>
    | VIEW <view_name>
    | WAREHOUSE <wh_name>
    | STAGE <stage_name>
    | FUNCTION <func_name>
    | PROCEDURE <proc_name>
    | STREAM <stream_name>
    | TASK <task_name>
    | PIPE <pipe_name>
    | FILE FORMAT <format_name>
  }
  TO ROLE <role_name> [ WITH GRANT OPTION ];

-- Grant role to user
GRANT ROLE <role_name> TO USER <user_name>;

-- Grant role to role (hierarchy)
GRANT ROLE <child_role> TO ROLE <parent_role>;

-- Grant future privileges (on objects not yet created)
GRANT <privilege> ON FUTURE <object_type> IN { DATABASE <db> | SCHEMA <schema> }
  TO ROLE <role_name>;
```

### Object Privileges

| Object Type | Key Privileges | Description |
|-------------|---------------|-------------|
| **DATABASE** | USAGE, CREATE SCHEMA, MODIFY, MONITOR | Access, create schemas, modify settings |
| **SCHEMA** | USAGE, CREATE TABLE, CREATE VIEW, MODIFY | Access, create objects, modify |
| **TABLE** | SELECT, INSERT, UPDATE, DELETE, TRUNCATE, OWNERSHIP | DML operations, ownership |
| **VIEW** | SELECT, OWNERSHIP | Query view, ownership |
| **WAREHOUSE** | USAGE, OPERATE, MODIFY, MONITOR, OWNERSHIP | Execute queries, resize, monitor |
| **STAGE** | USAGE, READ, WRITE | List files, download, upload |
| **STREAM** | SELECT, OWNERSHIP | Query stream changes |
| **TASK** | MONITOR, OPERATE, OWNERSHIP | View status, resume/suspend |
| **PIPE** | MONITOR, OPERATE, OWNERSHIP | View status, refresh |

### Basic Example

```sql
-- Grant database usage
GRANT USAGE ON DATABASE analytics_db TO ROLE analyst_role;

-- Grant schema usage
GRANT USAGE ON SCHEMA analytics_db.sales TO ROLE analyst_role;

-- Grant table SELECT
GRANT SELECT ON TABLE analytics_db.sales.fact_orders TO ROLE analyst_role;

-- Grant warehouse usage
GRANT USAGE ON WAREHOUSE analyst_wh TO ROLE analyst_role;

-- Verify grants
SHOW GRANTS TO ROLE analyst_role;
```

### Advanced Example (Production Access Control)

```sql
-- Production read-only access
GRANT USAGE ON DATABASE prod_db TO ROLE prod_read_only;
GRANT USAGE ON ALL SCHEMAS IN DATABASE prod_db TO ROLE prod_read_only;
GRANT SELECT ON ALL TABLES IN SCHEMA prod_db.sales TO ROLE prod_read_only;
GRANT SELECT ON ALL VIEWS IN SCHEMA prod_db.sales TO ROLE prod_read_only;
GRANT USAGE ON WAREHOUSE analyst_wh TO ROLE prod_read_only;

-- Future objects (automatically grant on creation)
GRANT SELECT ON FUTURE TABLES IN SCHEMA prod_db.sales TO ROLE prod_read_only;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA prod_db.sales TO ROLE prod_read_only;

-- ETL role (full access to staging, write to production)
GRANT ALL ON DATABASE staging_db TO ROLE etl_role;
GRANT ALL ON ALL SCHEMAS IN DATABASE staging_db TO ROLE etl_role;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN DATABASE prod_db TO ROLE etl_role;
GRANT USAGE, OPERATE ON WAREHOUSE etl_wh TO ROLE etl_role;
GRANT MONITOR, OPERATE ON ALL TASKS IN SCHEMA prod_db.etl TO ROLE etl_role;
GRANT MONITOR, OPERATE ON ALL PIPES IN SCHEMA prod_db.ingestion TO ROLE etl_role;

-- Admin role (object creation)
GRANT CREATE SCHEMA ON DATABASE prod_db TO ROLE data_admin;
GRANT CREATE TABLE, CREATE VIEW ON ALL SCHEMAS IN DATABASE prod_db TO ROLE data_admin;
GRANT OWNERSHIP ON ALL TABLES IN SCHEMA prod_db.sales TO ROLE data_admin;
```

---

## Data Masking Policies

Masking policies dynamically mask column values based on user role at query time. No data duplication; masking applied during query execution.

### Syntax: CREATE MASKING POLICY

#### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] MASKING POLICY <policy_name> AS (
  <val> <data_type>
) RETURNS <data_type> ->
  CASE
    WHEN <condition> THEN <masking_expression>
    ELSE <original_value>
  END
[ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
[ COMMENT = '<string>' ];

-- Apply masking policy to column
ALTER TABLE <table_name> MODIFY COLUMN <col_name> 
  SET MASKING POLICY <policy_name> [ USING ( <col_name>, ... ) ];

-- Remove masking policy
ALTER TABLE <table_name> MODIFY COLUMN <col_name> UNSET MASKING POLICY;
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `val` | Variable | Required | Input value (column value) | `email STRING` |
| `data_type` | Type | Required | Input and return data type | `VARCHAR` |
| `condition` | Boolean | Required | Role/context check | `CURRENT_ROLE() IN ('ANALYST')` |
| `masking_expression` | Expression | Required | Masked value | `'***MASKED***'` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (pii='true')` |
| `COMMENT` | String | NULL | Policy description | `'Email masking for analysts'` |

#### Basic Example

```sql
-- Create masking policy for emails
CREATE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('ADMIN', 'SECURITY_ADMIN') THEN val
    ELSE REGEXP_REPLACE(val, '.+@', '***@')  -- Mask local part
  END
  COMMENT = 'Mask email addresses for non-admin users';

-- Apply policy to column
ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY email_mask;

-- Test masking
USE ROLE analyst_role;
SELECT email FROM customers LIMIT 5;
-- Returns: ***@example.com, ***@company.com

USE ROLE admin;
SELECT email FROM customers LIMIT 5;
-- Returns: john@example.com, jane@company.com (unmasked)
```

#### Advanced Example (Multi-Policy PII Protection)

```sql
-- SSN masking (show last 4 digits)
CREATE OR REPLACE MASKING POLICY ssn_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('COMPLIANCE_ADMIN', 'ACCOUNTADMIN') THEN val
    WHEN CURRENT_ROLE() IN ('HR_MANAGER') THEN '***-**-' || RIGHT(val, 4)
    ELSE '***-**-****'
  END
  WITH TAG (pii = 'high', compliance = 'HIPAA')
  COMMENT = 'SSN masking - compliance approved';

-- Credit card masking
CREATE OR REPLACE MASKING POLICY credit_card_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('PAYMENT_ADMIN', 'ACCOUNTADMIN') THEN val
    WHEN CURRENT_ROLE() IN ('CUSTOMER_SERVICE') THEN '****-****-****-' || RIGHT(val, 4)
    ELSE '****-****-****-****'
  END
  COMMENT = 'Credit card masking - PCI-DSS compliant';

-- Salary masking (bucket ranges)
CREATE OR REPLACE MASKING POLICY salary_mask AS (val NUMBER) RETURNS VARCHAR ->
  CASE
    WHEN CURRENT_ROLE() IN ('HR_ADMIN', 'ACCOUNTADMIN') THEN TO_VARCHAR(val)
    WHEN CURRENT_ROLE() IN ('MANAGER') THEN
      CASE
        WHEN val < 50000 THEN '$0-$50K'
        WHEN val < 100000 THEN '$50K-$100K'
        WHEN val < 150000 THEN '$100K-$150K'
        ELSE '$150K+'
      END
    ELSE '<CONFIDENTIAL>'
  END
  COMMENT = 'Salary range masking';

-- Apply policies
ALTER TABLE employees MODIFY COLUMN ssn SET MASKING POLICY ssn_mask;
ALTER TABLE customers MODIFY COLUMN credit_card SET MASKING POLICY credit_card_mask;
ALTER TABLE employees MODIFY COLUMN salary SET MASKING POLICY salary_mask;

-- Show masking policies
SHOW MASKING POLICIES;

-- Describe policy
DESC MASKING POLICY ssn_mask;
```

---

## Row Access Policies

Row access policies filter rows based on user role at query time. Users see only authorized rows without additional WHERE clauses.

### Syntax: CREATE ROW ACCESS POLICY

#### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] ROW ACCESS POLICY <policy_name> AS (
  <col_name> <data_type> [ , ... ]
) RETURNS BOOLEAN ->
  <boolean_expression>
[ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
[ COMMENT = '<string>' ];

-- Apply row access policy to table
ALTER TABLE <table_name> 
  ADD ROW ACCESS POLICY <policy_name> ON ( <col_name> [ , ... ] );

-- Remove row access policy
ALTER TABLE <table_name> DROP ROW ACCESS POLICY <policy_name>;
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `col_name` | String | Required | Mapping columns for policy | `region VARCHAR` |
| `data_type` | Type | Required | Column data type | `VARCHAR` |
| `boolean_expression` | Boolean | Required | Row filter condition | `region IN ('US', 'CA')` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (security='rls')` |
| `COMMENT` | String | NULL | Policy description | `'Region-based access control'` |

#### Basic Example

```sql
-- Create region-based access policy
CREATE ROW ACCESS POLICY region_policy AS (region VARCHAR) RETURNS BOOLEAN ->
  CASE
    WHEN CURRENT_ROLE() = 'GLOBAL_ADMIN' THEN TRUE
    WHEN CURRENT_ROLE() = 'US_ANALYST' AND region = 'US' THEN TRUE
    WHEN CURRENT_ROLE() = 'EU_ANALYST' AND region = 'EU' THEN TRUE
    ELSE FALSE
  END
  COMMENT = 'Filter rows by user region access';

-- Apply policy to table
ALTER TABLE sales.orders 
  ADD ROW ACCESS POLICY region_policy ON (region);

-- Test access
USE ROLE us_analyst;
SELECT COUNT(*) FROM sales.orders;
-- Returns: Only US region rows

USE ROLE global_admin;
SELECT COUNT(*) FROM sales.orders;
-- Returns: All rows
```

#### Advanced Example (Multi-Tenant SaaS)

```sql
-- Tenant isolation policy
CREATE OR REPLACE ROW ACCESS POLICY tenant_isolation AS (
  tenant_id INT
) RETURNS BOOLEAN ->
  CASE
    WHEN CURRENT_ROLE() = 'SUPER_ADMIN' THEN TRUE
    WHEN CURRENT_ROLE() LIKE 'TENANT_%' THEN
      tenant_id = TO_NUMBER(REGEXP_SUBSTR(CURRENT_ROLE(), '[0-9]+'))
    ELSE FALSE
  END
  WITH TAG (security = 'multi_tenant', compliance = 'SOC2')
  COMMENT = 'Tenant data isolation - SaaS security';

-- Apply to multi-tenant tables
ALTER TABLE customers ADD ROW ACCESS POLICY tenant_isolation ON (tenant_id);
ALTER TABLE orders ADD ROW ACCESS POLICY tenant_isolation ON (tenant_id);
ALTER TABLE invoices ADD ROW ACCESS POLICY tenant_isolation ON (tenant_id);

-- Create tenant-specific roles
CREATE ROLE tenant_1001 COMMENT = 'Tenant 1001 access';
CREATE ROLE tenant_1002 COMMENT = 'Tenant 1002 access';

-- Grant roles to users
GRANT ROLE tenant_1001 TO USER user_from_tenant_1001;
GRANT ROLE tenant_1002 TO USER user_from_tenant_1002;

-- Test isolation
USE ROLE tenant_1001;
SELECT * FROM customers;
-- Returns: Only tenant_id = 1001 rows

USE ROLE tenant_1002;
SELECT * FROM customers;
-- Returns: Only tenant_id = 1002 rows
```

---

## Secure Data Sharing

Share live data across Snowflake accounts without data movement or ETL. Data provider grants access to specific objects; consumers query shared data in real-time. Zero-copy, no storage costs for consumers.

### Syntax: CREATE SHARE

```sql
-- Create outbound share (provider)
CREATE [ OR REPLACE ] SHARE <share_name>
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ];

-- Grant objects to share
GRANT USAGE ON DATABASE <db_name> TO SHARE <share_name>;
GRANT USAGE ON SCHEMA <schema_name> TO SHARE <share_name>;
GRANT SELECT ON TABLE <table_name> TO SHARE <share_name>;
GRANT SELECT ON VIEW <view_name> TO SHARE <share_name>;

-- Add consumer accounts
ALTER SHARE <share_name> ADD ACCOUNTS = <account_id> [ , ... ];

-- Consumer: Create database from share
CREATE DATABASE <db_name> FROM SHARE <provider_account>.<share_name>;
```

#### Example: Cross-Account Data Sharing

```sql
-- PROVIDER SIDE:
-- Create share
CREATE SHARE sales_data_share
  COMMENT = 'Share sales data with partner accounts';

-- Grant database and schema
GRANT USAGE ON DATABASE prod_db TO SHARE sales_data_share;
GRANT USAGE ON SCHEMA prod_db.sales TO SHARE sales_data_share;

-- Grant tables (use secure views for filtering)
CREATE SECURE VIEW prod_db.sales.v_orders_shared AS
SELECT 
  order_id,
  order_date,
  total_amount,
  region
FROM prod_db.sales.fact_orders
WHERE region IN ('US', 'CA');  -- Filter sensitive regions

GRANT SELECT ON VIEW prod_db.sales.v_orders_shared TO SHARE sales_data_share;

-- Add consumer accounts
ALTER SHARE sales_data_share ADD ACCOUNTS = xy12345, ab67890;

-- View share details
SHOW GRANTS TO SHARE sales_data_share;

-- CONSUMER SIDE:
-- Create database from share
CREATE DATABASE partner_sales_data 
  FROM SHARE provider_account.sales_data_share
  COMMENT = 'Shared sales data from provider';

-- Query shared data
SELECT * FROM partner_sales_data.sales.v_orders_shared LIMIT 10;

-- Grant access to internal roles
GRANT USAGE ON DATABASE partner_sales_data TO ROLE analyst_role;
GRANT USAGE ON SCHEMA partner_sales_data.sales TO ROLE analyst_role;
GRANT SELECT ON ALL VIEWS IN SCHEMA partner_sales_data.sales TO ROLE analyst_role;
```

---

## SDLC Use Case: DevSecOps Access Testing

### Scenario: Automated Security Testing in CI/CD

```python
# security_tests.py (pytest framework)
import snowflake.connector
import pytest

class SecurityAccessTests:
    def __init__(self, account, warehouse):
        self.account = account
        self.warehouse = warehouse
    
    def get_connection(self, user, password, role):
        """Create Snowflake connection with specific role"""
        return snowflake.connector.connect(
            account=self.account,
            user=user,
            password=password,
            warehouse=self.warehouse,
            role=role
        )
    
    def test_analyst_read_only_access(self):
        """Test analysts can only read production data"""
        conn = self.get_connection('analyst_user', '***', 'ANALYST_ROLE')
        cursor = conn.cursor()
        
        # Should succeed: SELECT
        cursor.execute("SELECT COUNT(*) FROM prod.sales.fact_orders")
        assert cursor.fetchone()[0] > 0, "Should access production data"
        
        # Should fail: INSERT
        with pytest.raises(Exception) as exc:
            cursor.execute("""
                INSERT INTO prod.sales.fact_orders 
                VALUES (999, 999, CURRENT_DATE(), 100.00, 'PENDING')
            """)
        assert "Insufficient privileges" in str(exc.value)
        
        # Should fail: DELETE
        with pytest.raises(Exception) as exc:
            cursor.execute("DELETE FROM prod.sales.fact_orders WHERE order_id = 1")
        assert "Insufficient privileges" in str(exc.value)
        
        conn.close()
        print("✅ Analyst read-only access verified")
    
    def test_masking_policy_enforcement(self):
        """Test PII masking for analysts"""
        conn = self.get_connection('analyst_user', '***', 'ANALYST_ROLE')
        cursor = conn.cursor()
        
        # Query masked email
        cursor.execute("SELECT email FROM prod.sales.customers LIMIT 1")
        masked_email = cursor.fetchone()[0]
        
        assert '***@' in masked_email, "Email should be masked"
        assert '@' in masked_email, "Email format should be preserved"
        
        conn.close()
        print("✅ Masking policy enforced for analysts")
    
    def test_row_access_policy_enforcement(self):
        """Test row-level security"""
        # US analyst should only see US region
        conn_us = self.get_connection('us_analyst', '***', 'US_ANALYST')
        cursor_us = conn_us.cursor()
        
        cursor_us.execute("""
            SELECT DISTINCT region FROM prod.sales.fact_orders
        """)
        regions = [row[0] for row in cursor_us.fetchall()]
        
        assert regions == ['US'], f"US analyst should only see US region, got {regions}"
        
        conn_us.close()
        print("✅ Row access policy enforced for US region")
    
    def test_service_account_etl_access(self):
        """Test ETL service account can write to staging"""
        conn = self.get_connection('cicd_service_account', '***', 'ETL_ROLE')
        cursor = conn.cursor()
        
        # Should succeed: Write to staging
        cursor.execute("""
            CREATE OR REPLACE TRANSIENT TABLE staging.tmp.test_table (
              id INT,
              value VARCHAR
            )
        """)
        
        cursor.execute("INSERT INTO staging.tmp.test_table VALUES (1, 'test')")
        
        cursor.execute("SELECT COUNT(*) FROM staging.tmp.test_table")
        assert cursor.fetchone()[0] == 1
        
        # Cleanup
        cursor.execute("DROP TABLE staging.tmp.test_table")
        
        conn.close()
        print("✅ ETL service account write access verified")
    
    def test_future_grants_propagation(self):
        """Test future grants apply to new objects"""
        conn = self.get_connection('admin_user', '***', 'SYSADMIN')
        cursor = conn.cursor()
        
        # Create new table
        cursor.execute("""
            CREATE OR REPLACE TABLE prod.sales.test_future_grants (
              id INT
            )
        """)
        
        # Verify analyst role can access (future grants)
        conn_analyst = self.get_connection('analyst_user', '***', 'ANALYST_ROLE')
        cursor_analyst = conn_analyst.cursor()
        
        cursor_analyst.execute("SELECT * FROM prod.sales.test_future_grants")
        cursor_analyst.fetchall()  # Should succeed
        
        # Cleanup
        cursor.execute("DROP TABLE prod.sales.test_future_grants")
        
        conn.close()
        conn_analyst.close()
        print("✅ Future grants working correctly")

# Run security tests
if __name__ == '__main__':
    tests = SecurityAccessTests(
        account='myorg.us-east-1',
        warehouse='test_wh'
    )
    
    tests.test_analyst_read_only_access()
    tests.test_masking_policy_enforcement()
    tests.test_row_access_policy_enforcement()
    tests.test_service_account_etl_access()
    tests.test_future_grants_propagation()
```

---

## Key Concepts

- **RBAC**: Role-Based Access Control using hierarchical roles and privilege grants
- **Masking Policy**: Dynamic column-level data masking based on user role (no data duplication)
- **Row Access Policy**: Row-level security filtering data by user role at query time
- **Secure View**: Encrypted view definition preventing SQL inspection (required for data sharing)
- **Data Sharing**: Zero-copy live data sharing across Snowflake accounts without ETL

---
