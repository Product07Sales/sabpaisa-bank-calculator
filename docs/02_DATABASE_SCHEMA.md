# 02 - DATABASE SCHEMA
**Universal Digital Collection Platform**

---

| **Document Version** | **Date** | **Author** | **Status** |
|---|---|---|---|
| v1.0 | 2025-01-15 | Database Architecture Team | ✅ Approved |

---

## Table of Contents
1. [Overview](#overview)
2. [Multi-Tenancy Strategy](#multi-tenancy-strategy)
3. [ER Diagram](#er-diagram)
4. [Shared Schema Tables](#shared-schema-tables)
5. [Tenant Schema Tables](#tenant-schema-tables)
6. [Complete SQL Schema](#complete-sql-schema)
7. [Indexing Strategy](#indexing-strategy)
8. [Data Retention & Archival](#data-retention--archival)
9. [Partitioning Strategy](#partitioning-strategy)
10. [Database Performance Optimization](#database-performance-optimization)

---

## 1. Overview

### 1.1 Database Technology
- **RDBMS**: PostgreSQL 15+
- **Why PostgreSQL?**
  - Native support for JSONB (flexible schema)
  - Excellent multi-tenancy support (schemas)
  - ACID compliance (critical for financial transactions)
  - Robust partitioning and replication
  - Strong community and ecosystem

### 1.2 Database Architecture
- **Primary Database**: PostgreSQL RDS Multi-AZ (Mumbai)
- **Read Replicas**: 2 replicas for report queries
- **Connection Pooling**: PgBouncer (max 100 connections)
- **Backup**: Automated daily snapshots + PITR (35-day retention)

---

## 2. Multi-Tenancy Strategy

### 2.1 Schema-Per-Tenant Approach

**Structure**:
```
Database: universal_collection_platform
├── Schema: platform (shared across all organizations)
│   ├── organizations
│   ├── subscriptions
│   ├── platform_admins
│   └── audit_logs (platform-wide)
├── Schema: org_school001
│   ├── users
│   ├── collection_heads
│   ├── forms
│   ├── user_collections
│   ├── transactions
│   ├── offline_payments
│   ├── receipts
│   ├── form_submissions
│   ├── branding_config
│   └── audit_logs (org-specific)
├── Schema: org_hospital002
│   ├── users
│   ├── ... (same structure)
└── Schema: org_govt003
    ├── users
    ├── ... (same structure)
```

**Benefits**:
- ✅ Complete data isolation
- ✅ Per-tenant schema customization
- ✅ Independent scaling and optimization
- ✅ Easier compliance (data residency)
- ✅ Simplified backup/restore per org

---

## 3. ER Diagram

### 3.1 Shared Schema (Platform)

```
┌─────────────────────────┐
│   organizations         │
├─────────────────────────┤
│ PK org_id              │◄────┐
│    org_name             │     │
│    org_schema           │     │
│    domain               │     │
│    status               │     │
│    created_at           │     │
└─────────────────────────┘     │
                                │
┌─────────────────────────┐     │
│   subscriptions         │     │
├─────────────────────────┤     │
│ PK subscription_id      │     │
│ FK org_id               │─────┘
│    plan_type            │
│    billing_cycle        │
│    amount               │
│    status               │
│    valid_from           │
│    valid_to             │
└─────────────────────────┘

┌─────────────────────────┐
│   platform_admins       │
├─────────────────────────┤
│ PK admin_id             │
│    email                │
│    password_hash        │
│    role                 │
│    created_at           │
└─────────────────────────┘
```

### 3.2 Tenant Schema (Per Organization)

```
┌─────────────────────────┐
│   users                 │
├─────────────────────────┤
│ PK user_id              │◄────┐
│    unique_id            │     │
│    email                │     │
│    password_hash        │     │
│    mobile               │     │
│    role                 │     │
│    status               │     │
│    created_at           │     │
└─────────────────────────┘     │
                                │
┌─────────────────────────┐     │
│   collection_heads      │     │
├─────────────────────────┤     │
│ PK collection_head_id   │◄──┐ │
│    name                 │   │ │
│    description          │   │ │
│    is_active            │   │ │
│    created_at           │   │ │
└─────────────────────────┘   │ │
                              │ │
┌─────────────────────────┐   │ │
│   forms                 │   │ │
├─────────────────────────┤   │ │
│ PK form_id              │   │ │
│    form_type            │   │ │
│    name                 │   │ │
│    form_schema (JSONB)  │   │ │
│    public_url           │   │ │
│    is_active            │   │ │
│    created_at           │   │ │
└─────────────────────────┘   │ │
                              │ │
┌─────────────────────────┐   │ │
│   user_collections      │   │ │
├─────────────────────────┤   │ │
│ PK collection_id        │   │ │
│ FK user_id              │───┘ │
│ FK collection_head_id   │─────┘
│ FK form_id (nullable)   │
│    amount               │
│    paid_amount          │
│    due_date             │
│    status               │
│    created_at           │
└────────┬────────────────┘
         │
         │
┌────────▼────────────────┐
│   transactions          │
├─────────────────────────┤
│ PK transaction_id       │
│ FK user_id              │
│ FK collection_id        │
│    amount               │
│    payment_mode         │
│    payment_gateway_txn_id│
│    status               │
│    initiated_at         │
│    completed_at         │
└────────┬────────────────┘
         │
         │
┌────────▼────────────────┐
│   receipts              │
├─────────────────────────┤
│ PK receipt_id           │
│ FK transaction_id       │
│ FK user_id              │
│    receipt_number       │
│    receipt_url (S3)     │
│    generated_at         │
└─────────────────────────┘

┌─────────────────────────┐
│   offline_payments      │
├─────────────────────────┤
│ PK offline_payment_id   │
│ FK user_id              │
│ FK collection_id        │
│    amount               │
│    payment_mode         │
│    payment_date         │
│    receipt_number       │
│    entered_by_user_id   │
│    entered_by_ip        │
│    remarks              │
│    created_at           │
└─────────────────────────┘

┌─────────────────────────┐
│   form_submissions      │
├─────────────────────────┤
│ PK submission_id        │
│ FK form_id              │
│    submission_data (JSONB)│
│    submitted_by_email   │
│    submitted_at         │
└─────────────────────────┘

┌─────────────────────────┐
│   branding_config       │
├─────────────────────────┤
│ PK config_id            │
│    org_logo_url         │
│    theme_primary_color  │
│    theme_secondary_color│
│    receipt_template     │
│    updated_at           │
└─────────────────────────┘
```

---

## 4. Shared Schema Tables

### 4.1 organizations

**Purpose**: Master table for all organizations (tenants)

```sql
CREATE TABLE platform.organizations (
  org_id VARCHAR(50) PRIMARY KEY,
  org_name VARCHAR(255) NOT NULL,
  org_schema VARCHAR(50) UNIQUE NOT NULL,
  domain VARCHAR(255),
  industry VARCHAR(50), -- EDUCATION, GOVERNMENT, HEALTHCARE, etc.
  status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, SUSPENDED, DELETED
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_organizations_status ON platform.organizations(status);
CREATE INDEX idx_organizations_industry ON platform.organizations(industry);
```

**Sample Data**:
```sql
INSERT INTO platform.organizations VALUES
  ('ORG001', 'ABC School', 'org_abc_school', 'abcschool.edu', 'EDUCATION', 'ACTIVE', NOW(), NOW()),
  ('ORG002', 'City Hospital', 'org_city_hospital', 'cityhospital.org', 'HEALTHCARE', 'ACTIVE', NOW(), NOW());
```

---

### 4.2 subscriptions

**Purpose**: Track organization subscription plans and billing

```sql
CREATE TABLE platform.subscriptions (
  subscription_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id VARCHAR(50) NOT NULL REFERENCES platform.organizations(org_id) ON DELETE CASCADE,
  plan_type VARCHAR(50) NOT NULL, -- BASIC, PROFESSIONAL, ENTERPRISE
  billing_cycle VARCHAR(20) NOT NULL, -- MONTHLY, ANNUAL
  amount DECIMAL(10,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'INR',
  status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, SUSPENDED, CANCELLED
  valid_from DATE NOT NULL,
  valid_to DATE NOT NULL,
  auto_renew BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_org_id ON platform.subscriptions(org_id);
CREATE INDEX idx_subscriptions_status ON platform.subscriptions(status);
CREATE INDEX idx_subscriptions_valid_to ON platform.subscriptions(valid_to);
```

**Plan Pricing** (reference):
| Plan | Monthly | Annual | Max Users | Features |
|---|---|---|---|---|
| BASIC | ₹5,000 | ₹50,000 | 1,000 | Core features only |
| PROFESSIONAL | ₹15,000 | ₹1,50,000 | 10,000 | + Offline payments, Analytics |
| ENTERPRISE | Custom | Custom | Unlimited | + API access, Custom branding, Dedicated support |

---

### 4.3 platform_admins

**Purpose**: Super admins who manage the platform

```sql
CREATE TABLE platform.platform_admins (
  admin_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  role VARCHAR(20) DEFAULT 'SUPER_ADMIN', -- SUPER_ADMIN, SUPPORT_ADMIN
  status VARCHAR(20) DEFAULT 'ACTIVE',
  last_login_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_platform_admins_email ON platform.platform_admins(email);
```

---

### 4.4 platform.audit_logs

**Purpose**: Platform-wide audit trail

```sql
CREATE TABLE platform.audit_logs (
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id VARCHAR(50),
  user_id UUID,
  action VARCHAR(100) NOT NULL,
  resource VARCHAR(50),
  resource_id VARCHAR(100),
  ip_address INET,
  user_agent TEXT,
  request_payload JSONB,
  response_payload JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_org_id ON platform.audit_logs(org_id);
CREATE INDEX idx_audit_logs_user_id ON platform.audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON platform.audit_logs(action);
CREATE INDEX idx_audit_logs_created_at ON platform.audit_logs(created_at);
```

---

## 5. Tenant Schema Tables

**Note**: All tables below exist in each tenant schema (e.g., `org_abc_school.users`)

### 5.1 users

**Purpose**: All users within an organization (Super Admin, Org Admin, End Users)

```sql
CREATE TABLE {org_schema}.users (
  user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  unique_id VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  mobile VARCHAR(15),
  role VARCHAR(20) NOT NULL, -- SUPER_ADMIN, ORG_ADMIN, END_USER
  status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, INACTIVE, LOCKED
  login_attempts INT DEFAULT 0,
  locked_until TIMESTAMP,
  last_login_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_users_unique_id ON {org_schema}.users(unique_id);
CREATE UNIQUE INDEX idx_users_email ON {org_schema}.users(email);
CREATE INDEX idx_users_role ON {org_schema}.users(role);
CREATE INDEX idx_users_status ON {org_schema}.users(status);
```

**Constraints**:
- `unique_id`: 6-20 alphanumeric characters (e.g., STUD001, EMP123)
- `email`: Valid email format
- `mobile`: 10 digits, starts with 6-9 (Indian format)
- `password_hash`: bcrypt hash (cost factor 12)

**Sample Data**:
```sql
INSERT INTO org_abc_school.users VALUES
  (gen_random_uuid(), 'ADMIN001', 'admin@abcschool.edu', '$2b$12$...', 'John Admin', '9876543210', 'ORG_ADMIN', 'ACTIVE', 0, NULL, NULL, NOW(), NOW()),
  (gen_random_uuid(), 'STUD001', 'student1@abcschool.edu', '$2b$12$...', 'Jane Student', '9876543211', 'END_USER', 'ACTIVE', 0, NULL, NULL, NOW(), NOW());
```

---

### 5.2 collection_heads

**Purpose**: Define fee/collection types (max 15 per organization)

```sql
CREATE TABLE {org_schema}.collection_heads (
  collection_head_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  is_active BOOLEAN DEFAULT true,
  display_order INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  CONSTRAINT max_collection_heads CHECK (
    (SELECT COUNT(*) FROM {org_schema}.collection_heads WHERE is_active = true) <= 15
  )
);

CREATE UNIQUE INDEX idx_collection_heads_code ON {org_schema}.collection_heads(code);
CREATE INDEX idx_collection_heads_is_active ON {org_schema}.collection_heads(is_active);
```

**Sample Data** (Education):
```sql
INSERT INTO org_abc_school.collection_heads (code, name, description, display_order) VALUES
  ('TUITION_FEE', 'Tuition Fee', 'Quarterly tuition fee', 1),
  ('HOSTEL_FEE', 'Hostel Fee', 'Hostel accommodation fee', 2),
  ('LIBRARY_FEE', 'Library Fee', 'Library usage fee', 3),
  ('EXAM_FEE', 'Exam Fee', 'Examination fee', 4),
  ('TRANSPORT_FEE', 'Transport Fee', 'Bus transport fee', 5);
```

---

### 5.3 forms

**Purpose**: Form definitions (Open/Closed/Mixed)

```sql
CREATE TABLE {org_schema}.forms (
  form_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  form_type VARCHAR(20) NOT NULL, -- OPEN, CLOSED, MIXED
  name VARCHAR(255) NOT NULL,
  description TEXT,
  form_schema JSONB NOT NULL, -- Stores field definitions
  public_url VARCHAR(255) UNIQUE, -- For OPEN forms
  is_active BOOLEAN DEFAULT true,
  created_by UUID REFERENCES {org_schema}.users(user_id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_forms_form_type ON {org_schema}.forms(form_type);
CREATE INDEX idx_forms_is_active ON {org_schema}.forms(is_active);
CREATE INDEX idx_forms_public_url ON {org_schema}.forms(public_url) WHERE public_url IS NOT NULL;
```

**Sample form_schema (MIXED type)**:
```json
{
  "formType": "MIXED",
  "adminFields": [
    {
      "fieldName": "studentId",
      "fieldLabel": "Student ID",
      "fieldType": "text",
      "required": true
    },
    {
      "fieldName": "studentName",
      "fieldLabel": "Student Name",
      "fieldType": "text",
      "required": true
    }
  ],
  "userFields": [
    {
      "fieldName": "bankName",
      "fieldLabel": "Bank Name",
      "fieldType": "text",
      "required": true
    },
    {
      "fieldName": "accountNumber",
      "fieldLabel": "Account Number",
      "fieldType": "number",
      "required": true,
      "validation": {
        "minLength": 9,
        "maxLength": 18
      }
    }
  ]
}
```

---

### 5.4 user_collections

**Purpose**: Individual user dues/collections

```sql
CREATE TABLE {org_schema}.user_collections (
  collection_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES {org_schema}.users(user_id) ON DELETE CASCADE,
  collection_head_id UUID NOT NULL REFERENCES {org_schema}.collection_heads(collection_head_id),
  form_id UUID REFERENCES {org_schema}.forms(form_id), -- NULL for closed, populated for open/mixed
  amount DECIMAL(10,2) NOT NULL,
  paid_amount DECIMAL(10,2) DEFAULT 0.00,
  due_date DATE NOT NULL,
  late_fee_percent DECIMAL(5,2) DEFAULT 0.00,
  status VARCHAR(20) DEFAULT 'PENDING', -- PENDING, PARTIAL, PAID, OVERDUE
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  CONSTRAINT valid_paid_amount CHECK (paid_amount <= amount),
  CONSTRAINT valid_late_fee CHECK (late_fee_percent >= 0 AND late_fee_percent <= 100)
);

CREATE INDEX idx_user_collections_user_id ON {org_schema}.user_collections(user_id);
CREATE INDEX idx_user_collections_collection_head_id ON {org_schema}.user_collections(collection_head_id);
CREATE INDEX idx_user_collections_status ON {org_schema}.user_collections(status);
CREATE INDEX idx_user_collections_due_date ON {org_schema}.user_collections(due_date);
```

**Calculated Fields** (via application logic or trigger):
- `status`:
  - `PENDING` if `paid_amount = 0`
  - `PARTIAL` if `0 < paid_amount < amount`
  - `PAID` if `paid_amount = amount`
  - `OVERDUE` if `due_date < CURRENT_DATE AND status != 'PAID'`

**Sample Data**:
```sql
INSERT INTO org_abc_school.user_collections (user_id, collection_head_id, amount, due_date) VALUES
  ((SELECT user_id FROM org_abc_school.users WHERE unique_id = 'STUD001'),
   (SELECT collection_head_id FROM org_abc_school.collection_heads WHERE code = 'TUITION_FEE'),
   15000.00, '2025-02-01');
```

---

### 5.5 transactions

**Purpose**: Payment transactions (online + offline)

```sql
CREATE TABLE {org_schema}.transactions (
  transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES {org_schema}.users(user_id),
  collection_id UUID NOT NULL REFERENCES {org_schema}.user_collections(collection_id),
  amount DECIMAL(10,2) NOT NULL,
  payment_mode VARCHAR(20) NOT NULL, -- ONLINE, OFFLINE
  payment_method VARCHAR(20), -- For ONLINE: UPI, CARD, NETBANKING | For OFFLINE: CASH, POS, CHEQUE
  payment_gateway_txn_id VARCHAR(255), -- SabPaisa transaction ID (for ONLINE)
  status VARCHAR(20) DEFAULT 'INITIATED', -- INITIATED, PROCESSING, SUCCESS, FAILED, REFUNDED
  initiated_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP,
  refund_initiated_at TIMESTAMP,
  refund_completed_at TIMESTAMP,
  metadata JSONB, -- Additional payment details
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_transactions_user_id ON {org_schema}.transactions(user_id);
CREATE INDEX idx_transactions_collection_id ON {org_schema}.transactions(collection_id);
CREATE INDEX idx_transactions_status ON {org_schema}.transactions(status);
CREATE INDEX idx_transactions_payment_mode ON {org_schema}.transactions(payment_mode);
CREATE INDEX idx_transactions_payment_gateway_txn_id ON {org_schema}.transactions(payment_gateway_txn_id) WHERE payment_gateway_txn_id IS NOT NULL;
CREATE INDEX idx_transactions_created_at ON {org_schema}.transactions(created_at);

-- Partitioning (see section 9 for details)
-- PARTITION BY RANGE (created_at);
```

**Sample Transaction** (Online Success):
```sql
INSERT INTO org_abc_school.transactions (user_id, collection_id, amount, payment_mode, payment_method, payment_gateway_txn_id, status, completed_at) VALUES
  ((SELECT user_id FROM org_abc_school.users WHERE unique_id = 'STUD001'),
   '...', 15000.00, 'ONLINE', 'UPI', 'SABP123456789', 'SUCCESS', NOW());
```

---

### 5.6 offline_payments

**Purpose**: Manual payment entries (Cash/POS/Cheque)

```sql
CREATE TABLE {org_schema}.offline_payments (
  offline_payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL REFERENCES {org_schema}.transactions(transaction_id),
  user_id UUID NOT NULL REFERENCES {org_schema}.users(user_id),
  collection_id UUID NOT NULL REFERENCES {org_schema}.user_collections(collection_id),
  amount DECIMAL(10,2) NOT NULL,
  payment_mode VARCHAR(20) NOT NULL, -- CASH, POS, CHEQUE
  payment_date DATE NOT NULL,
  receipt_number VARCHAR(100),
  cheque_number VARCHAR(50), -- For CHEQUE mode
  cheque_date DATE, -- For CHEQUE mode
  bank_name VARCHAR(255), -- For CHEQUE mode
  entered_by_user_id UUID NOT NULL REFERENCES {org_schema}.users(user_id),
  entered_by_ip INET NOT NULL,
  remarks TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_offline_payments_user_id ON {org_schema}.offline_payments(user_id);
CREATE INDEX idx_offline_payments_entered_by ON {org_schema}.offline_payments(entered_by_user_id);
CREATE INDEX idx_offline_payments_payment_date ON {org_schema}.offline_payments(payment_date);
CREATE INDEX idx_offline_payments_payment_mode ON {org_schema}.offline_payments(payment_mode);
```

---

### 5.7 receipts

**Purpose**: Generated receipt records

```sql
CREATE TABLE {org_schema}.receipts (
  receipt_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL REFERENCES {org_schema}.transactions(transaction_id),
  user_id UUID NOT NULL REFERENCES {org_schema}.users(user_id),
  receipt_number VARCHAR(100) UNIQUE NOT NULL,
  receipt_url VARCHAR(500) NOT NULL, -- S3 URL
  qr_code_url VARCHAR(500), -- QR code for verification
  is_offline BOOLEAN DEFAULT false,
  generated_at TIMESTAMP DEFAULT NOW(),
  emailed_at TIMESTAMP,
  downloaded_count INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_receipts_receipt_number ON {org_schema}.receipts(receipt_number);
CREATE INDEX idx_receipts_transaction_id ON {org_schema}.receipts(transaction_id);
CREATE INDEX idx_receipts_user_id ON {org_schema}.receipts(user_id);
CREATE INDEX idx_receipts_generated_at ON {org_schema}.receipts(generated_at);
```

**Receipt Number Format**: `REC/{YYYY}/{MM}/{ORG_CODE}/{SEQUENTIAL}`
- Example: `REC/2025/01/ABC/00001`

---

### 5.8 form_submissions

**Purpose**: Store submissions for OPEN and MIXED forms

```sql
CREATE TABLE {org_schema}.form_submissions (
  submission_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  form_id UUID NOT NULL REFERENCES {org_schema}.forms(form_id) ON DELETE CASCADE,
  collection_id UUID REFERENCES {org_schema}.user_collections(collection_id),
  submission_data JSONB NOT NULL, -- User-entered data
  submitted_by_email VARCHAR(255),
  submitted_by_ip INET,
  submitted_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_form_submissions_form_id ON {org_schema}.form_submissions(form_id);
CREATE INDEX idx_form_submissions_submitted_at ON {org_schema}.form_submissions(submitted_at);
CREATE INDEX idx_form_submissions_data_gin ON {org_schema}.form_submissions USING GIN (submission_data);
```

**Sample submission_data**:
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "mobile": "9876543210",
  "amount": 5000,
  "purpose": "Donation for library books"
}
```

---

### 5.9 branding_config

**Purpose**: Organization branding settings

```sql
CREATE TABLE {org_schema}.branding_config (
  config_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_logo_url VARCHAR(500), -- S3 URL
  theme_primary_color VARCHAR(7) DEFAULT '#0066CC', -- Hex color
  theme_secondary_color VARCHAR(7) DEFAULT '#FFFFFF',
  receipt_template VARCHAR(20) DEFAULT 'STANDARD', -- STANDARD, FORMAL, MODERN
  email_footer TEXT,
  sms_sender_id VARCHAR(20),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Only one config per organization
CREATE UNIQUE INDEX idx_branding_config_singleton ON {org_schema}.branding_config ((1));
```

**SabPaisa Logo Enforcement**:
- Application enforces SabPaisa logo on all receipts (bottom-right)
- Not configurable via database

---

### 5.10 audit_logs (per organization)

**Purpose**: Organization-specific audit trail

```sql
CREATE TABLE {org_schema}.audit_logs (
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES {org_schema}.users(user_id),
  action VARCHAR(100) NOT NULL,
  resource VARCHAR(50),
  resource_id VARCHAR(100),
  ip_address INET,
  user_agent TEXT,
  changes JSONB, -- Before/after values for updates
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_user_id ON {org_schema}.audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON {org_schema}.audit_logs(action);
CREATE INDEX idx_audit_logs_created_at ON {org_schema}.audit_logs(created_at);
```

**Retention**: 7 years (RBI compliance)

---

## 6. Complete SQL Schema

### 6.1 Automated Schema Creation Script

```sql
-- Function to create tenant schema
CREATE OR REPLACE FUNCTION create_tenant_schema(org_schema_name VARCHAR)
RETURNS VOID AS $$
BEGIN
  -- Create schema
  EXECUTE format('CREATE SCHEMA %I', org_schema_name);

  -- Create tables
  EXECUTE format('
    CREATE TABLE %I.users (
      user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      unique_id VARCHAR(50) UNIQUE NOT NULL,
      email VARCHAR(255) UNIQUE NOT NULL,
      password_hash VARCHAR(255) NOT NULL,
      full_name VARCHAR(255) NOT NULL,
      mobile VARCHAR(15),
      role VARCHAR(20) NOT NULL,
      status VARCHAR(20) DEFAULT ''ACTIVE'',
      login_attempts INT DEFAULT 0,
      locked_until TIMESTAMP,
      last_login_at TIMESTAMP,
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW()
    );

    CREATE UNIQUE INDEX idx_users_unique_id ON %I.users(unique_id);
    CREATE UNIQUE INDEX idx_users_email ON %I.users(email);
    CREATE INDEX idx_users_role ON %I.users(role);
    CREATE INDEX idx_users_status ON %I.users(status);
  ', org_schema_name, org_schema_name, org_schema_name, org_schema_name, org_schema_name);

  -- Collection heads table
  EXECUTE format('
    CREATE TABLE %I.collection_heads (
      collection_head_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      code VARCHAR(50) UNIQUE NOT NULL,
      name VARCHAR(255) NOT NULL,
      description TEXT,
      is_active BOOLEAN DEFAULT true,
      display_order INT DEFAULT 0,
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW()
    );

    CREATE UNIQUE INDEX idx_collection_heads_code ON %I.collection_heads(code);
    CREATE INDEX idx_collection_heads_is_active ON %I.collection_heads(is_active);
  ', org_schema_name, org_schema_name, org_schema_name);

  -- Forms table
  EXECUTE format('
    CREATE TABLE %I.forms (
      form_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      form_type VARCHAR(20) NOT NULL,
      name VARCHAR(255) NOT NULL,
      description TEXT,
      form_schema JSONB NOT NULL,
      public_url VARCHAR(255) UNIQUE,
      is_active BOOLEAN DEFAULT true,
      created_by UUID REFERENCES %I.users(user_id),
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW()
    );

    CREATE INDEX idx_forms_form_type ON %I.forms(form_type);
    CREATE INDEX idx_forms_is_active ON %I.forms(is_active);
  ', org_schema_name, org_schema_name, org_schema_name, org_schema_name);

  -- User collections table
  EXECUTE format('
    CREATE TABLE %I.user_collections (
      collection_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      user_id UUID NOT NULL REFERENCES %I.users(user_id) ON DELETE CASCADE,
      collection_head_id UUID NOT NULL REFERENCES %I.collection_heads(collection_head_id),
      form_id UUID REFERENCES %I.forms(form_id),
      amount DECIMAL(10,2) NOT NULL,
      paid_amount DECIMAL(10,2) DEFAULT 0.00,
      due_date DATE NOT NULL,
      late_fee_percent DECIMAL(5,2) DEFAULT 0.00,
      status VARCHAR(20) DEFAULT ''PENDING'',
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW(),
      CONSTRAINT valid_paid_amount CHECK (paid_amount <= amount)
    );

    CREATE INDEX idx_user_collections_user_id ON %I.user_collections(user_id);
    CREATE INDEX idx_user_collections_status ON %I.user_collections(status);
  ', org_schema_name, org_schema_name, org_schema_name, org_schema_name, org_schema_name);

  -- Transactions table
  EXECUTE format('
    CREATE TABLE %I.transactions (
      transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      user_id UUID NOT NULL REFERENCES %I.users(user_id),
      collection_id UUID NOT NULL REFERENCES %I.user_collections(collection_id),
      amount DECIMAL(10,2) NOT NULL,
      payment_mode VARCHAR(20) NOT NULL,
      payment_method VARCHAR(20),
      payment_gateway_txn_id VARCHAR(255),
      status VARCHAR(20) DEFAULT ''INITIATED'',
      initiated_at TIMESTAMP DEFAULT NOW(),
      completed_at TIMESTAMP,
      metadata JSONB,
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW()
    );

    CREATE INDEX idx_transactions_user_id ON %I.transactions(user_id);
    CREATE INDEX idx_transactions_status ON %I.transactions(status);
    CREATE INDEX idx_transactions_created_at ON %I.transactions(created_at);
  ', org_schema_name, org_schema_name, org_schema_name, org_schema_name, org_schema_name, org_schema_name);

  -- Continue for all other tables...
  -- (offline_payments, receipts, form_submissions, branding_config, audit_logs)

  RAISE NOTICE 'Tenant schema % created successfully', org_schema_name;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT create_tenant_schema('org_abc_school');
```

---

## 7. Indexing Strategy

### 7.1 Index Types Used

| Index Type | Use Case | Example |
|---|---|---|
| **B-Tree** (default) | Equality and range queries | `CREATE INDEX idx_users_email ON users(email);` |
| **Unique** | Enforce uniqueness | `CREATE UNIQUE INDEX idx_users_unique_id ON users(unique_id);` |
| **Partial** | Index subset of rows | `CREATE INDEX idx_active_users ON users(user_id) WHERE status = 'ACTIVE';` |
| **GIN (Generalized Inverted)** | JSONB queries | `CREATE INDEX idx_form_schema_gin ON forms USING GIN (form_schema);` |
| **BRIN (Block Range)** | Large time-series tables | `CREATE INDEX idx_audit_logs_brin ON audit_logs USING BRIN (created_at);` |

### 7.2 Critical Indexes

**Users Table**:
```sql
CREATE UNIQUE INDEX idx_users_unique_id ON users(unique_id);
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_mobile ON users(mobile);
```

**Transactions Table** (most queried):
```sql
CREATE INDEX idx_transactions_user_id ON transactions(user_id);
CREATE INDEX idx_transactions_collection_id ON transactions(collection_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_payment_gateway_txn_id ON transactions(payment_gateway_txn_id) WHERE payment_gateway_txn_id IS NOT NULL;

-- Composite index for dashboard queries
CREATE INDEX idx_transactions_user_status_date ON transactions(user_id, status, created_at DESC);
```

**Audit Logs Table**:
```sql
-- BRIN index for large time-series data
CREATE INDEX idx_audit_logs_created_at_brin ON audit_logs USING BRIN (created_at);

-- Standard indexes for filtering
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
```

### 7.3 JSONB Indexes

```sql
-- Index for form_schema queries
CREATE INDEX idx_forms_schema_gin ON forms USING GIN (form_schema);

-- Query examples
SELECT * FROM forms WHERE form_schema @> '{"formType": "OPEN"}';
SELECT * FROM forms WHERE form_schema ? 'adminFields';
```

---

## 8. Data Retention & Archival

### 8.1 Retention Policies

| Table | Retention (Hot) | Archive (Warm) | Compliance (Cold) |
|---|---|---|---|
| `transactions` | 1 year | 3 years | 7 years (RBI) |
| `audit_logs` | 1 year | 3 years | 7 years (RBI) |
| `receipts` | 1 year | 5 years | 10 years |
| `form_submissions` | 2 years | 5 years | N/A |
| `users` | Active only | Deleted after 5 years | N/A |

### 8.2 Archival Strategy

**Hot Storage** (PostgreSQL):
- Tables: `transactions`, `receipts`, `audit_logs`
- Period: Last 1 year
- Access: Real-time queries

**Warm Storage** (S3 + Glacier):
- Tables: Partitioned tables > 1 year old
- Period: 1-3 years
- Access: On-demand (restore within 24 hours)

**Cold Storage** (S3 Glacier Deep Archive):
- Tables: All > 3 years
- Period: 3-7 years
- Access: Compliance only (restore within 48 hours)

**Archival Script** (monthly cron):
```sql
-- Move transactions older than 1 year to archive table
INSERT INTO archived_transactions
SELECT * FROM transactions WHERE created_at < NOW() - INTERVAL '1 year';

-- Delete from hot table
DELETE FROM transactions WHERE created_at < NOW() - INTERVAL '1 year';

-- Export archive table to S3
COPY archived_transactions TO '/tmp/transactions_archive_2024.csv' CSV HEADER;
-- Upload to S3: aws s3 cp transactions_archive_2024.csv s3://bucket/archives/
```

---

## 9. Partitioning Strategy

### 9.1 Range Partitioning (by Date)

**Transactions Table** (partitioned by month):
```sql
CREATE TABLE transactions (
  transaction_id UUID NOT NULL,
  created_at TIMESTAMP NOT NULL,
  -- other columns
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE transactions_2025_01 PARTITION OF transactions
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE transactions_2025_02 PARTITION OF transactions
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Automated partition creation (monthly cron job)
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS VOID AS $$
DECLARE
  start_date DATE := DATE_TRUNC('month', CURRENT_DATE + INTERVAL '1 month');
  end_date DATE := start_date + INTERVAL '1 month';
  partition_name TEXT := 'transactions_' || TO_CHAR(start_date, 'YYYY_MM');
BEGIN
  EXECUTE format('
    CREATE TABLE IF NOT EXISTS %I PARTITION OF transactions
    FOR VALUES FROM (%L) TO (%L)',
    partition_name, start_date, end_date
  );
END;
$$ LANGUAGE plpgsql;

-- Schedule via cron
-- 0 0 1 * * psql -c "SELECT create_monthly_partition();"
```

**Benefits**:
- Faster queries (partition pruning)
- Easier archival (drop old partitions)
- Better maintenance (VACUUM only recent partitions)

### 9.2 List Partitioning (by Organization - Future)

For very large deployments (10,000+ organizations):
```sql
CREATE TABLE transactions_partitioned (
  org_id VARCHAR(50) NOT NULL,
  -- other columns
) PARTITION BY LIST (org_id);

CREATE TABLE transactions_org001 PARTITION OF transactions_partitioned
  FOR VALUES IN ('ORG001');

CREATE TABLE transactions_org002 PARTITION OF transactions_partitioned
  FOR VALUES IN ('ORG002');
```

---

## 10. Database Performance Optimization

### 10.1 Query Optimization

**Use EXPLAIN ANALYZE**:
```sql
EXPLAIN ANALYZE
SELECT t.*, u.full_name, c.name AS collection_name
FROM transactions t
JOIN users u ON t.user_id = u.user_id
JOIN user_collections uc ON t.collection_id = uc.collection_id
JOIN collection_heads c ON uc.collection_head_id = c.collection_head_id
WHERE t.user_id = 'abc-123-def'
  AND t.status = 'SUCCESS'
  AND t.created_at >= '2025-01-01'
ORDER BY t.created_at DESC
LIMIT 10;
```

**Optimization Techniques**:
1. Ensure indexes on `user_id`, `status`, `created_at`
2. Use composite index: `(user_id, status, created_at DESC)`
3. Partition table by `created_at`

### 10.2 Connection Pooling

**PgBouncer Configuration**:
```ini
[databases]
universal_collection_platform = host=postgres-rds.amazonaws.com port=5432 dbname=universal_collection_platform

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
```

### 10.3 Vacuuming Strategy

**Auto-Vacuum Settings**:
```sql
ALTER TABLE transactions SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_analyze_scale_factor = 0.02
);
```

**Manual VACUUM** (monthly):
```sql
VACUUM ANALYZE transactions;
VACUUM ANALYZE users;
VACUUM ANALYZE audit_logs;
```

### 10.4 Monitoring Queries

**Slow Query Log**:
```sql
-- Enable slow query logging (queries > 1s)
ALTER DATABASE universal_collection_platform SET log_min_duration_statement = 1000;

-- View slow queries
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

**Table Bloat**:
```sql
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS size,
  n_dead_tup
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 10;
```

---

## Appendix: Sample Queries

### A.1 Get User Dues (with Late Fee)
```sql
SELECT
  uc.collection_id,
  ch.name AS fee_type,
  uc.amount,
  uc.paid_amount,
  uc.amount - uc.paid_amount AS balance,
  uc.due_date,
  CASE
    WHEN uc.due_date < CURRENT_DATE AND uc.status != 'PAID'
    THEN (uc.amount - uc.paid_amount) * uc.late_fee_percent / 100
    ELSE 0
  END AS late_fee,
  uc.status
FROM org_abc_school.user_collections uc
JOIN org_abc_school.collection_heads ch ON uc.collection_head_id = ch.collection_head_id
WHERE uc.user_id = 'abc-123-def'
  AND uc.status != 'PAID'
ORDER BY uc.due_date ASC;
```

### A.2 Dashboard Metrics (Org Admin)
```sql
-- Today's collections
SELECT
  COUNT(*) AS total_transactions,
  COUNT(*) FILTER (WHERE status = 'SUCCESS') AS successful,
  SUM(amount) FILTER (WHERE status = 'SUCCESS') AS total_amount,
  AVG(amount) FILTER (WHERE status = 'SUCCESS') AS avg_transaction_value
FROM org_abc_school.transactions
WHERE DATE(created_at) = CURRENT_DATE;

-- Outstanding dues
SELECT
  SUM(amount - paid_amount) AS total_outstanding,
  COUNT(DISTINCT user_id) AS users_with_dues
FROM org_abc_school.user_collections
WHERE status IN ('PENDING', 'PARTIAL', 'OVERDUE');
```

### A.3 Super Admin - Platform Summary
```sql
-- Platform-wide metrics
SELECT
  COUNT(DISTINCT org_id) AS total_organizations,
  SUM((SELECT COUNT(*) FROM pg_tables WHERE schemaname LIKE 'org_%')) AS total_tenants,
  (SELECT SUM(amount) FROM platform.get_all_transactions() WHERE status = 'SUCCESS') AS total_gmv
FROM platform.organizations
WHERE status = 'ACTIVE';
```

---

**Document Control**:
- **Next Review Date**: 2025-04-15
- **Change Log**: See version control system
- **Approvers**: Database Architect, CTO

---

*End of Database Schema Document*
