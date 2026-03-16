# 01 - SYSTEM ARCHITECTURE
**Universal Digital Collection Platform**

---

| **Document Version** | **Date** | **Author** | **Status** |
|---|---|---|---|
| v1.0 | 2025-01-15 | Platform Architecture Team | ✅ Approved |

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [High-Level Architecture](#high-level-architecture)
4. [Low-Level Architecture](#low-level-architecture)
5. [Microservices Breakdown](#microservices-breakdown)
6. [Component Interaction Flows](#component-interaction-flows)
7. [Multi-Tenancy Architecture](#multi-tenancy-architecture)
8. [Technology Stack](#technology-stack)
9. [Scalability Strategy](#scalability-strategy)
10. [High Availability & Fault Tolerance](#high-availability--fault-tolerance)
11. [Disaster Recovery](#disaster-recovery)

---

## 1. Executive Summary

The **Universal Digital Collection Platform** is an enterprise-grade, multi-tenant SaaS solution designed to streamline digital payment collection across multiple industries including Education, Government, Healthcare, Trusts, Utilities, Events, and Enterprises.

### Key Architectural Highlights:
- **Multi-Tenant SaaS**: Schema-per-tenant isolation for complete data security
- **Microservices Architecture**: 10 independently scalable services
- **Payment Gateway Integration**: SabPaisa API with bank-grade security
- **Scalability**: Designed for 10M+ end users, 3,500+ organizations
- **High Availability**: 99.95% uptime SLA with multi-AZ deployment
- **Performance**: Sub-500ms API response time (p95)
- **Security**: PCI-DSS compliant, RBI guidelines adherent

---

## 2. System Overview

### 2.1 Platform Purpose
Enable organizations to:
- Collect payments digitally (online + offline)
- Manage user dues and fee structures
- Generate branded receipts and invoices
- Track payments with real-time analytics
- Support multiple collection models (open/closed/mixed)

### 2.2 User Roles
| Role | Responsibilities | Access Level |
|---|---|---|
| **Super Admin** | Platform owner, creates orgs, configures system | Full platform access |
| **Organization Admin** | Manages organization data, creates forms, offline payments | Organization-wide access |
| **End User** | Views dues, makes payments, downloads receipts | Personal data only |

### 2.3 Core Features
1. **Form Management**: Open, Closed, Mixed form types
2. **Payment Processing**: Online (SabPaisa) + Offline (Cash/POS/Cheque)
3. **Receipt Generation**: Branded PDF receipts with QR codes
4. **Analytics Dashboards**: Role-based real-time insights
5. **Bulk Upload**: CSV/Excel data import (50MB max)
6. **Reporting**: 8+ report types per role
7. **Branding Engine**: Customizable logos, colors, themes
8. **Notification System**: Email, SMS, WhatsApp alerts

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Web App       │   Mobile App    │   Admin Portal              │
│  (React SPA)    │  (React Native) │   (React Admin)             │
└────────┬────────┴────────┬────────┴────────┬────────────────────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │ HTTPS/TLS 1.3
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                             │
│  Kong / AWS API Gateway / NGINX                                  │
│  - Authentication (JWT validation)                               │
│  - Rate Limiting (100 req/min/user, 1000 req/min/org)           │
│  - Request Routing                                               │
│  - SSL Termination                                               │
└────────┬─────────────────────────────────────────────────────────┘
         │
         ├──────────┬──────────┬──────────┬──────────┬──────────┐
         ▼          ▼          ▼          ▼          ▼          ▼
┌──────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER (Microservices)              │
├────────────┬────────────┬────────────┬────────────┬─────────────┤
│   Auth     │   User     │  Payment   │   Form     │ Collection  │
│  Service   │  Service   │  Service   │  Service   │  Service    │
├────────────┼────────────┼────────────┼────────────┼─────────────┤
│  Report    │ Notification│  Receipt  │  Offline   │   Audit     │
│  Service   │  Service   │  Service   │  Payment   │  Service    │
│            │            │           │  Service   │             │
└────────┬───┴────────┬───┴────────┬───┴────────┬───┴─────────┬──┘
         │            │            │            │             │
         │            └────────────┼────────────┘             │
         │                         ▼                          │
         │              ┌──────────────────────┐              │
         │              │  MESSAGE QUEUE       │              │
         │              │  RabbitMQ / Kafka    │              │
         │              │  - Event streaming   │              │
         │              │  - Async jobs        │              │
         │              └──────────────────────┘              │
         │                         │                          │
         └─────────────────────────┼──────────────────────────┘
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                │
├────────────┬────────────┬────────────┬────────────┬─────────────┤
│ PostgreSQL │   Redis    │  MongoDB   │    S3      │  Elasticsearch│
│ (Primary   │  (Cache &  │ (Logs &    │ (Receipts, │ (Full-text  │
│  Database) │  Session)  │ Analytics) │  Reports)  │  Search)    │
└────────────┴────────────┴────────────┴────────────┴─────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│                   EXTERNAL INTEGRATIONS                          │
├────────────┬────────────┬────────────┬────────────┬─────────────┤
│ SabPaisa   │  AWS SES   │   Twilio   │ WhatsApp   │   ERP       │
│ Payment    │  (Email)   │   (SMS)    │ Business   │ Systems     │
│ Gateway    │            │            │   API      │ (SAP, etc)  │
└────────────┴────────────┴────────────┴────────────┴─────────────┘
```

---

## 4. Low-Level Architecture

### 4.1 Microservices Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY (Kong)                         │
│  - JWT Authentication Middleware                                │
│  - Rate Limiting Plugin                                         │
│  - CORS Configuration                                           │
│  - Request Logging                                              │
└────────┬────────────────────────────────────────────────────────┘
         │
         ├──────────────────────────────────────────────────────┐
         │                                                      │
    ┌────▼─────┐   ┌───────────┐   ┌──────────┐   ┌──────────┐│
    │  Auth    │   │   User    │   │ Payment  │   │  Form    ││
    │ Service  │◄─►│  Service  │◄─►│ Service  │◄─►│ Service  ││
    └────┬─────┘   └─────┬─────┘   └────┬─────┘   └────┬─────┘│
         │               │              │              │       │
         │   ┌───────────┴──────────────┴───────┬──────┴───┐   │
         │   │                                  │          │   │
    ┌────▼───▼──┐   ┌──────────┐   ┌───────────▼┐   ┌─────▼──┐│
    │Collection │   │  Report  │   │  Receipt    │   │Offline ││
    │  Service  │◄─►│  Service │◄─►│  Service    │◄─►│Payment ││
    └─────┬─────┘   └────┬─────┘   └──────┬──────┘   └───┬────┘│
          │              │                │              │     │
          │         ┌────▼────────────────▼──────────────▼──┐  │
          │         │      Notification Service            │  │
          │         │  (Email/SMS/WhatsApp Queue)          │  │
          │         └──────────────────┬───────────────────┘  │
          │                            │                      │
          └────────────┬───────────────┴───────────┬──────────┘
                       │                           │
                  ┌────▼────┐              ┌───────▼──────┐
                  │ RabbitMQ│              │ Audit Service│
                  │ Message │              │ (All Actions)│
                  │  Queue  │              └──────────────┘
                  └────┬────┘
                       │
         ┌─────────────┼─────────────┬──────────────┐
         │             │             │              │
    ┌────▼────┐   ┌────▼────┐  ┌────▼────┐   ┌─────▼────┐
    │PostgreSQL│   │  Redis  │  │ MongoDB │   │    S3    │
    │ (Multi-  │   │ (Cache) │  │ (Logs)  │   │(Receipts)│
    │ Tenant)  │   │         │  │         │   │          │
    └──────────┘   └─────────┘  └─────────┘   └──────────┘
```

---

## 5. Microservices Breakdown

### 5.1 Auth Service
**Responsibility**: User authentication and authorization

**Technology**: Node.js + Express.js
**Database**: PostgreSQL (shared `platform.users` table)
**Cache**: Redis (session storage)

**Key Functions**:
- User login (email + password)
- JWT token generation (24hr expiry)
- Refresh token handling
- OTP generation (6-digit, 10-min expiry)
- Password reset flow
- Account lockout (5 failed attempts → 30 min lockout)
- Multi-factor authentication (optional)

**API Endpoints**:
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/logout`
- `POST /api/v1/auth/refresh-token`
- `POST /api/v1/auth/forgot-password`
- `POST /api/v1/auth/reset-password`
- `POST /api/v1/auth/verify-otp`

**Performance Targets**:
- Login: < 200ms (p95)
- Token validation: < 50ms (cached)

---

### 5.2 User Management Service
**Responsibility**: CRUD operations for users (Super Admin, Org Admin, End Users)

**Technology**: Node.js + Express.js
**Database**: PostgreSQL (tenant-specific schemas)

**Key Functions**:
- Create users (single + bulk upload)
- Update user profile
- Assign roles and permissions (RBAC)
- CSV parsing and validation (max 50MB, 100K records)
- Duplicate detection
- User search and filtering

**API Endpoints**:
- `GET /api/v1/{org_id}/users` - List all users
- `POST /api/v1/{org_id}/users` - Create user
- `PUT /api/v1/{org_id}/users/{user_id}` - Update user
- `DELETE /api/v1/{org_id}/users/{user_id}` - Delete user
- `POST /api/v1/{org_id}/users/bulk-upload` - CSV upload
- `GET /api/v1/{org_id}/users/bulk-upload/{job_id}/status` - Upload status

**Validation Rules**:
- Unique ID: 6-20 alphanumeric characters
- Email: Valid format
- Mobile: 10 digits, starts with 6-9
- Password: Min 8 chars, 1 uppercase, 1 number, 1 special char

---

### 5.3 Payment Processing Service
**Responsibility**: Payment gateway integration (SabPaisa)

**Technology**: Node.js + Express.js
**Database**: PostgreSQL (transactions table)
**Queue**: RabbitMQ (webhook processing)

**Key Functions**:
- Generate encrypted payment payload (SHA-512 HMAC)
- Initiate payment on SabPaisa
- Handle SabPaisa webhook callbacks
- Update transaction status (Initiated → Processing → Success/Failed)
- Retry failed webhooks (exponential backoff: 1s, 5s, 30s, 2min, 5min)
- Reconciliation with SabPaisa settlement reports

**Transaction State Machine**:
```
INITIATED → PROCESSING → SUCCESS
                      ↓
                    FAILED
                      ↓
                  REFUND_PENDING → REFUNDED
```

**API Endpoints**:
- `POST /api/v1/{org_id}/payments/initiate`
- `POST /api/v1/payments/webhook/sabpaisa` (public endpoint)
- `GET /api/v1/{org_id}/payments/{transaction_id}/status`
- `POST /api/v1/{org_id}/payments/{transaction_id}/refund`

**SabPaisa Integration**:
- **Initiate Payment URL**: `https://api.sabpaisa.in/pg/v1/txnInitiate`
- **Payload Encryption**: SHA-512 HMAC with secret key
- **Webhook URL**: `https://api.platform.com/v1/payments/webhook/sabpaisa`
- **Response Time**: < 3s for initiation

**Sample Encrypted Payload**:
```json
{
  "clientCode": "SABP001",
  "transactionId": "TXN123456789",
  "amount": "10000",
  "customerName": "John Doe",
  "customerEmail": "john@example.com",
  "customerMobile": "9876543210",
  "callbackUrl": "https://platform.com/payment/success",
  "checksum": "a1b2c3d4e5f6..."
}
```

---

### 5.4 Form Management Service
**Responsibility**: Create and manage collection forms (Open/Closed/Mixed)

**Technology**: Node.js + Express.js
**Database**: PostgreSQL (forms, form_submissions tables)

**Key Functions**:
- Create form definitions (3 types)
- Store form schema as JSON
- Validate form submissions
- Support dynamic field validation
- Generate public form URLs (for Open forms)

**Form Types**:

| Type | Description | Use Case |
|---|---|---|
| **CLOSED** | Admin uploads 100% data via CSV. User CANNOT modify. | School fee collection with fixed amounts |
| **OPEN** | User enters ALL data (name, email, amount, etc.) | Donation forms, event registrations |
| **MIXED** | Admin uploads base data + User fills additional fields | Scholarship applications (admin uploads student ID, user fills bank details) |

**API Endpoints**:
- `POST /api/v1/{org_id}/forms` - Create form
- `GET /api/v1/{org_id}/forms` - List forms
- `GET /api/v1/{org_id}/forms/{form_id}` - Get form schema
- `POST /api/v1/forms/{form_public_id}/submit` - Submit open/mixed form (public)
- `PUT /api/v1/{org_id}/forms/{form_id}` - Update form
- `DELETE /api/v1/{org_id}/forms/{form_id}` - Delete form

**Sample Form Schema (MIXED)**:
```json
{
  "formId": "FORM123",
  "formType": "MIXED",
  "name": "Scholarship Application",
  "adminFields": ["studentId", "studentName", "class"],
  "userFields": ["bankName", "accountNumber", "ifscCode"],
  "validations": {
    "accountNumber": {"type": "number", "minLength": 9, "maxLength": 18},
    "ifscCode": {"type": "string", "pattern": "^[A-Z]{4}0[A-Z0-9]{6}$"}
  }
}
```

---

### 5.5 Collection Management Service
**Responsibility**: Manage user dues and collection heads

**Technology**: Node.js + Express.js
**Database**: PostgreSQL (collection_heads, user_collections tables)

**Key Functions**:
- Define collection heads (max 15 per organization)
- Create user-specific collections (dues)
- Support partial payments
- Calculate late fees and penalties
- Track due dates

**Collection Head Examples**:
- Tuition Fee
- Hostel Fee
- Library Fee
- Exam Fee
- Transport Fee
- Lab Fee
- Sports Fee
- Annual Fee

**API Endpoints**:
- `POST /api/v1/{org_id}/collection-heads` - Create collection head
- `GET /api/v1/{org_id}/collection-heads` - List all collection heads
- `POST /api/v1/{org_id}/collections` - Create user collection
- `GET /api/v1/{org_id}/users/{user_id}/collections` - Get user dues
- `PATCH /api/v1/{org_id}/collections/{collection_id}` - Update amount/due date

**Late Fee Calculation**:
```javascript
const lateFee = (baseAmount, daysOverdue, penaltyPercent) => {
  if (daysOverdue <= 0) return 0;
  const penalty = (baseAmount * penaltyPercent / 100);
  return Math.min(penalty, baseAmount * 0.20); // Max 20% penalty
};
```

---

### 5.6 Offline Payment Service
**Responsibility**: Manual payment entry (Cash/POS/Cheque)

**Technology**: Node.js + Express.js
**Database**: PostgreSQL (offline_payments table)
**Audit**: All entries logged with admin name, IP, timestamp

**Key Functions**:
- Activation workflow (Super Admin → SabPaisa API call)
- Record offline payments
- Validate payment details
- Generate receipts with "OFFLINE PAYMENT" watermark
- Sync with SabPaisa for reconciliation

**Activation Flow**:
```
Super Admin Dashboard → Enable Offline Payment Button →
API Call to SabPaisa /offlinePayment/activate →
SabPaisa Returns API Key →
Store API Key in org_config →
Org Admin Sees "Offline Payment" Menu
```

**API Endpoints**:
- `POST /api/v1/{org_id}/offline-payments/activate` (Super Admin only)
- `POST /api/v1/{org_id}/offline-payments` - Record payment (Org Admin)
- `GET /api/v1/{org_id}/offline-payments` - List offline payments

**Sample Offline Payment Entry**:
```json
{
  "userId": "USER123",
  "collectionHeadId": "COL456",
  "amount": 5000,
  "paymentMode": "CASH",
  "paymentDate": "2025-01-15T10:30:00Z",
  "receiptNumber": "REC789",
  "remarks": "Cash received at front desk",
  "enteredBy": "admin@school.com",
  "enteredByIp": "192.168.1.100"
}
```

---

### 5.7 Receipt Generation Service
**Responsibility**: Generate branded PDF receipts

**Technology**: Node.js + Puppeteer (headless Chrome)
**Storage**: AWS S3
**Queue**: RabbitMQ (async generation)

**Key Functions**:
- Generate PDF from HTML template
- Apply organization branding (logo, colors)
- Enforce mandatory SabPaisa logo (bottom-right)
- Generate QR code (transaction verification)
- Store receipts in S3
- Email receipt to user

**Receipt Components**:
- Organization logo (top-left, max 200x60px)
- Organization name and address
- Receipt number and date
- Transaction ID
- User details (name, ID, email)
- Payment details (amount, mode, collection head)
- QR code (for verification)
- **SabPaisa logo (bottom-right, fixed, non-removable)**

**API Endpoints**:
- `POST /api/v1/{org_id}/receipts/generate` - Generate receipt
- `GET /api/v1/{org_id}/receipts/{receipt_id}/download` - Download PDF
- `POST /api/v1/{org_id}/receipts/{receipt_id}/email` - Email receipt

**Performance**:
- PDF generation: < 2s
- S3 upload: < 1s
- Total time: < 3s

**Sample Receipt URL**:
```
https://receipts.platform.com/org123/2025/01/REC123456.pdf
```

---

### 5.8 Report & Analytics Service
**Responsibility**: Generate reports and dashboard analytics

**Technology**: Node.js + Express.js
**Database**: PostgreSQL (read replicas for reports)
**Cache**: Redis (report caching, 5-min TTL)

**Key Functions**:
- Real-time dashboard metrics
- Generate 8+ report types per role
- Export to PDF/Excel
- Data aggregation and filtering
- Scheduled report emails

**Reports**:

**Super Admin Reports**:
1. Platform Summary Report
2. Organization Performance Report
3. Revenue Report (transaction fees)
4. Settlement Report
5. Refund Report
6. Failed Transaction Report
7. User Growth Report
8. Offline Payment Report

**Org Admin Reports**:
1. Collection Summary Report
2. Transaction Report
3. Outstanding Report
4. Payment Mode Report
5. User Payment History
6. Settlement Report
7. Offline Payment Report
8. Form Performance Report

**End User Reports**:
1. Payment History
2. Pending Dues
3. Receipt Archive

**API Endpoints**:
- `GET /api/v1/{org_id}/dashboard` - Dashboard metrics
- `GET /api/v1/{org_id}/reports/transactions` - Transaction report
- `GET /api/v1/{org_id}/reports/outstanding` - Outstanding dues report
- `POST /api/v1/{org_id}/reports/export` - Export report (PDF/Excel)

---

### 5.9 Notification Service
**Responsibility**: Send Email/SMS/WhatsApp notifications

**Technology**: Node.js + Express.js
**Queue**: RabbitMQ (notification queue)
**Integrations**: AWS SES (Email), Twilio (SMS), WhatsApp Business API

**Key Functions**:
- Async notification sending
- Template management
- Retry logic for failed notifications
- Delivery status tracking
- Rate limiting (prevent spam)

**Notification Triggers**:
- Payment success → Email + SMS
- Payment failure → Email
- Offline payment recorded → Email
- Bulk upload completed → Email with error report
- Due date reminder (3 days before) → Email + SMS
- Overdue payment → Email + SMS

**API Endpoints**:
- `POST /api/v1/notifications/email` (internal)
- `POST /api/v1/notifications/sms` (internal)
- `POST /api/v1/notifications/whatsapp` (internal)
- `GET /api/v1/{org_id}/notifications/history` - Notification log

**Sample Email Template**:
```html
Subject: Payment Success - {{organizationName}}

Dear {{userName}},

Your payment of ₹{{amount}} for {{collectionHead}} has been successfully received.

Transaction ID: {{transactionId}}
Payment Date: {{paymentDate}}

Download Receipt: {{receiptLink}}

Powered by {{organizationName}} | SabPaisa Payment Gateway
```

---

### 5.10 Audit & Compliance Service
**Responsibility**: Log all critical actions for compliance

**Technology**: Node.js + Express.js
**Database**: MongoDB (audit_logs collection)
**Retention**: 7 years (RBI compliance)

**Key Functions**:
- Log all CRUD operations
- Track IP addresses
- Record timestamps
- Store request/response payloads (sensitive data masked)
- Generate audit reports

**Logged Actions**:
- User login/logout
- Payment initiated/completed
- Offline payment entry
- User created/updated
- Form created/deleted
- Configuration changes
- Bulk upload
- Receipt generation
- Refund initiated

**Audit Log Schema**:
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "organizationId": "ORG123",
  "userId": "USER456",
  "action": "OFFLINE_PAYMENT_RECORDED",
  "resource": "offline_payments",
  "resourceId": "OFF789",
  "ip": "192.168.1.100",
  "userAgent": "Mozilla/5.0...",
  "details": {
    "amount": 5000,
    "paymentMode": "CASH",
    "collectionHeadId": "COL456"
  }
}
```

**API Endpoints**:
- `POST /api/v1/audit/log` (internal)
- `GET /api/v1/{org_id}/audit/logs` - Query audit logs (Super Admin only)
- `GET /api/v1/{org_id}/audit/report` - Generate audit report

---

## 6. Component Interaction Flows

### 6.1 End User Payment Flow (Online)

```
┌───────────┐
│ End User  │
│  Login    │
└─────┬─────┘
      │ 1. Login (email + password)
      ▼
┌─────────────┐
│ Auth Service│
│ JWT Token   │
└─────┬───────┘
      │ 2. Token returned
      ▼
┌──────────────────┐
│ Collection       │
│ Service          │
│ GET /collections │
└─────┬────────────┘
      │ 3. Fetch user dues
      ▼
┌───────────────────┐
│ User Dashboard    │
│ Displays:         │
│ - Pending: ₹10,000│
│ - Paid: ₹5,000    │
└─────┬─────────────┘
      │ 4. User selects "Tuition Fee" → Pay Now
      ▼
┌──────────────────────┐
│ Payment Service      │
│ POST /payments/init  │
└─────┬────────────────┘
      │ 5. Create transaction (status: INITIATED)
      │    Insert into `transactions` table
      ▼
┌─────────────────────┐
│ Generate Encrypted  │
│ Payload (SHA-512)   │
└─────┬───────────────┘
      │ 6. Encrypted payload
      ▼
┌──────────────────────┐
│ Redirect to SabPaisa│
│ Payment Page         │
└─────┬────────────────┘
      │ 7. User completes payment on SabPaisa
      │    (UPI / Card / Net Banking)
      ▼
┌──────────────────────┐
│ SabPaisa sends       │
│ Webhook (POST)       │
└─────┬────────────────┘
      │ 8. Webhook received
      ▼
┌──────────────────────┐
│ Payment Service      │
│ Verify signature     │
│ Update transaction   │
│ status: SUCCESS      │
└─────┬────────────────┘
      │ 9. Publish event to RabbitMQ
      ▼
┌──────────────────────┐        ┌────────────────┐
│ Receipt Service      │◄───────┤  RabbitMQ      │
│ Generate PDF         │        │  Queue         │
│ Upload to S3         │        └────────────────┘
└─────┬────────────────┘
      │ 10. Receipt generated
      ▼
┌──────────────────────┐
│ Notification Service │
│ Send Email + SMS     │
└─────┬────────────────┘
      │ 11. Email sent with receipt link
      ▼
┌──────────────────────┐
│ User Receives Email  │
│ Downloads Receipt    │
└──────────────────────┘
```

**Timeline**:
- Steps 1-6: < 2s
- Step 7: User action (30s - 5min)
- Steps 8-11: < 5s

---

### 6.2 Offline Payment Entry Flow

```
┌───────────────┐
│ Org Admin     │
│ Login         │
└───────┬───────┘
        │ 1. Login
        ▼
┌───────────────────┐
│ Admin Dashboard   │
│ Check if offline  │
│ payment enabled   │
└───────┬───────────┘
        │ 2. Query org_config table
        ▼
  ┌─────┴─────┐
  │ Enabled?  │
  └─────┬─────┘
    YES │           NO
        │            │
        ▼            ▼
┌────────────┐  ┌─────────────────────┐
│ Show Menu  │  │ Show "Contact Super │
│ "Offline   │  │ Admin to Enable"    │
│ Payment"   │  └─────────────────────┘
└─────┬──────┘
      │ 3. Admin clicks "Offline Payment"
      ▼
┌─────────────────────┐
│ Select User         │
│ (Search by ID/Name) │
└─────┬───────────────┘
      │ 4. User selected
      ▼
┌─────────────────────┐
│ Select Collection   │
│ Head (Dropdown)     │
└─────┬───────────────┘
      │ 5. Collection selected
      ▼
┌─────────────────────┐
│ Enter Payment Mode  │
│ [ ] Cash            │
│ [ ] POS Machine     │
│ [ ] Cheque/DD       │
└─────┬───────────────┘
      │ 6. Payment mode selected
      ▼
┌─────────────────────┐
│ Enter Amount        │
│ Receipt Number      │
│ Payment Date        │
│ Remarks (optional)  │
└─────┬───────────────┘
      │ 7. Submit form
      ▼
┌─────────────────────────┐
│ Offline Payment Service │
│ Validate inputs         │
└─────┬───────────────────┘
      │ 8. Validation passed
      ▼
┌─────────────────────────┐
│ Insert into             │
│ offline_payments table  │
│ + Audit log entry       │
└─────┬───────────────────┘
      │ 9. Send to SabPaisa for reconciliation
      ▼
┌─────────────────────────┐
│ SabPaisa API Call       │
│ /offlinePayment/record  │
└─────┬───────────────────┘
      │ 10. SabPaisa confirms
      ▼
┌─────────────────────────┐
│ Update transaction      │
│ status: SUCCESS         │
└─────┬───────────────────┘
      │ 11. Trigger receipt generation
      ▼
┌─────────────────────────┐
│ Receipt Service         │
│ Generate PDF with       │
│ "OFFLINE PAYMENT"       │
│ watermark               │
└─────┬───────────────────┘
      │ 12. Receipt generated
      ▼
┌─────────────────────────┐
│ Notification Service    │
│ Email to user           │
└─────┬───────────────────┘
      │ 13. Admin sees success message
      ▼
┌─────────────────────────┐
│ "Payment Recorded       │
│ Successfully"           │
└─────────────────────────┘
```

**Audit Trail**:
```json
{
  "action": "OFFLINE_PAYMENT_RECORDED",
  "adminName": "admin@school.com",
  "adminIp": "192.168.1.100",
  "timestamp": "2025-01-15T10:30:00Z",
  "userId": "USER123",
  "amount": 5000,
  "paymentMode": "CASH"
}
```

---

### 6.3 Bulk Upload Flow (CSV)

```
┌───────────────┐
│ Org Admin     │
│ Uploads CSV   │
└───────┬───────┘
        │ 1. POST /users/bulk-upload (50MB max)
        ▼
┌─────────────────────┐
│ User Service        │
│ Validate file size  │
│ Validate file type  │
│ (.csv, .xlsx)       │
└─────┬───────────────┘
      │ 2. Parse CSV rows
      ▼
┌─────────────────────┐
│ Row-by-Row          │
│ Validation:         │
│ - Unique ID format  │
│ - Email valid       │
│ - Mobile 10 digits  │
│ - Amount numeric    │
│ - Collection exists │
│ - Date valid        │
└─────┬───────────────┘
      │ 3. Batch processing (1000 rows/batch)
      ▼
  ┌───┴────┐
  │ Valid? │
  └───┬────┘
  YES │         NO
      │          │
      ▼          ▼
┌──────────┐  ┌────────────┐
│ Insert   │  │ Add to     │
│ into DB  │  │ error_list │
└────┬─────┘  └─────┬──────┘
     │              │
     └──────┬───────┘
            │ 4. Process all rows
            ▼
┌─────────────────────┐
│ Generate Summary    │
│ - Total rows: 5000  │
│ - Success: 4800     │
│ - Failed: 200       │
└─────┬───────────────┘
      │ 5. Generate error CSV
      ▼
┌─────────────────────┐
│ Upload to S3        │
│ errors_USER123.csv  │
└─────┬───────────────┘
      │ 6. Send email
      ▼
┌─────────────────────┐
│ Notification Service│
│ Email to admin with │
│ summary + error CSV │
└─────────────────────┘
```

**Sample Error CSV**:
```csv
Row,UniqueID,Error
5,INVALID,Invalid unique ID format (must be 6-20 chars)
12,USER789,Collection head 'LAB_FEE' does not exist
25,USER890,Invalid email format
```

---

## 7. Multi-Tenancy Architecture

### 7.1 Strategy: Schema-Per-Tenant

**Approach**: Each organization gets its own PostgreSQL schema for complete data isolation.

**Benefits**:
- **Security**: No data leakage between organizations
- **Scalability**: Each schema can be optimized independently
- **Compliance**: Easier to meet data residency requirements
- **Backup**: Per-tenant backup and restore
- **Customization**: Schema can be extended per organization

**Database Structure**:
```sql
-- Shared schema (platform-wide)
CREATE SCHEMA platform;

-- Tables in shared schema
CREATE TABLE platform.organizations (
  org_id VARCHAR(50) PRIMARY KEY,
  org_name VARCHAR(255) NOT NULL,
  org_schema VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'org_school123'
  domain VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE platform.subscriptions (
  subscription_id UUID PRIMARY KEY,
  org_id VARCHAR(50) REFERENCES platform.organizations(org_id),
  plan_type VARCHAR(50), -- BASIC, PROFESSIONAL, ENTERPRISE
  billing_cycle VARCHAR(20), -- MONTHLY, ANNUAL
  status VARCHAR(20) -- ACTIVE, SUSPENDED, CANCELLED
);

-- Tenant-specific schema (one per organization)
CREATE SCHEMA org_school123;

-- Tables in tenant schema
CREATE TABLE org_school123.users (
  user_id UUID PRIMARY KEY,
  unique_id VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  role VARCHAR(20) NOT NULL, -- SUPER_ADMIN, ORG_ADMIN, END_USER
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE org_school123.transactions (
  transaction_id UUID PRIMARY KEY,
  user_id UUID REFERENCES org_school123.users(user_id),
  amount DECIMAL(10,2) NOT NULL,
  status VARCHAR(20), -- INITIATED, SUCCESS, FAILED
  payment_mode VARCHAR(20), -- ONLINE, OFFLINE
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 7.2 Schema Routing Logic

**Dynamic Schema Selection**:
```javascript
// Middleware to set schema based on organization
app.use((req, res, next) => {
  const orgId = req.params.org_id || req.user.orgId;

  // Lookup org schema from platform.organizations
  const org = await db.query(
    'SELECT org_schema FROM platform.organizations WHERE org_id = $1',
    [orgId]
  );

  if (!org.rows.length) {
    return res.status(404).json({ error: 'Organization not found' });
  }

  // Set schema for this request
  req.orgSchema = org.rows[0].org_schema;

  // Set search_path for PostgreSQL
  await db.query(`SET search_path TO ${req.orgSchema}, public`);

  next();
});

// Now all queries run in the tenant schema
app.get('/api/v1/:org_id/users', async (req, res) => {
  // This query runs in org_school123 schema automatically
  const users = await db.query('SELECT * FROM users');
  res.json(users.rows);
});
```

### 7.3 Schema Creation on Organization Signup

```javascript
async function createOrganization(orgData) {
  const orgSchema = `org_${orgData.orgId}`;

  // 1. Insert into platform.organizations
  await db.query(
    `INSERT INTO platform.organizations (org_id, org_name, org_schema)
     VALUES ($1, $2, $3)`,
    [orgData.orgId, orgData.orgName, orgSchema]
  );

  // 2. Create tenant schema
  await db.query(`CREATE SCHEMA ${orgSchema}`);

  // 3. Run migrations (create all tables)
  await runMigrations(orgSchema);

  // 4. Seed default data
  await seedDefaultData(orgSchema);

  return { orgId: orgData.orgId, orgSchema };
}

async function runMigrations(schema) {
  const migrations = [
    `CREATE TABLE ${schema}.users (...)`,
    `CREATE TABLE ${schema}.collection_heads (...)`,
    `CREATE TABLE ${schema}.forms (...)`,
    // ... all tenant tables
  ];

  for (const migration of migrations) {
    await db.query(migration);
  }
}
```

### 7.4 Data Isolation Guarantees

**Security Measures**:
1. **Application-level**: Middleware enforces schema selection
2. **Database-level**: PostgreSQL RLS (Row-Level Security) as backup
3. **API-level**: JWT token includes `orgId`, validated on every request
4. **Audit**: All cross-org access attempts logged

**Performance**:
- Query performance unaffected (same as single schema)
- Connection pooling per schema (avoid schema switching overhead)
- Indexes defined per schema for optimization

---

## 8. Technology Stack

### 8.1 Full Stack Overview

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Frontend** | React | 18.2+ | Web application UI |
| | TypeScript | 5.0+ | Type safety |
| | Redux Toolkit | 2.0+ | Global state management |
| | React Query | 5.0+ | Server state & caching |
| | Tailwind CSS | 3.4+ | Utility-first styling |
| | React Router | 6.20+ | Client-side routing |
| | Vite | 5.0+ | Build tool |
| **Backend** | Node.js | 18 LTS | Runtime environment |
| | Express.js | 4.18+ | Web framework |
| | TypeScript | 5.0+ | Type safety |
| **Database** | PostgreSQL | 15+ | Primary relational database |
| | Redis | 7.2+ | Cache & session store |
| | MongoDB | 7.0+ | Audit logs & analytics |
| | Elasticsearch | 8.11+ | Full-text search |
| **Message Queue** | RabbitMQ | 3.12+ | Async job processing |
| **File Storage** | AWS S3 | - | Receipts, reports, uploads |
| **CDN** | CloudFront | - | Static asset delivery |
| **API Gateway** | Kong | 3.5+ | API routing & rate limiting |
| **Monitoring** | Prometheus | 2.48+ | Metrics collection |
| | Grafana | 10.2+ | Dashboards |
| | New Relic | - | APM & error tracking |
| **Logging** | CloudWatch Logs | - | Centralized logging |
| | ELK Stack | 8.11+ | Log aggregation (optional) |
| **CI/CD** | GitHub Actions | - | Build & deployment pipelines |
| **Containerization** | Docker | 24.0+ | Containerization |
| | Kubernetes (EKS) | 1.28+ | Orchestration |
| **Cloud Platform** | AWS | - | Infrastructure |
| **Testing** | Jest | 29.7+ | Unit testing |
| | Playwright | 1.40+ | E2E testing |
| | k6 | 0.48+ | Load testing |
| **Payment Gateway** | SabPaisa | - | Payment processing |
| **Email** | AWS SES | - | Transactional emails |
| **SMS** | Twilio | - | SMS notifications |
| **WhatsApp** | WhatsApp Business API | - | WhatsApp messaging |

### 8.2 Development Tools

| Tool | Purpose |
|---|---|
| ESLint | JavaScript/TypeScript linting |
| Prettier | Code formatting |
| Husky | Git hooks |
| Commitlint | Commit message linting |
| VS Code | IDE (recommended) |
| Postman | API testing |
| pgAdmin | PostgreSQL GUI |
| Redis Commander | Redis GUI |

---

## 9. Scalability Strategy

### 9.1 Horizontal Scaling

**Application Layer**:
- Run multiple instances of each microservice behind load balancer
- Stateless services (session stored in Redis, not in-memory)
- Auto-scaling based on CPU/memory utilization (target: 70%)

**Example Kubernetes Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3  # Run 3 instances initially
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: platform/payment-service:v1.0
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 9.2 Database Scaling

**PostgreSQL**:
- **Read Replicas**: 2+ read replicas for report queries
- **Connection Pooling**: PgBouncer (max 100 connections per instance)
- **Partitioning**: Partition large tables (transactions, audit_logs) by date
  ```sql
  CREATE TABLE transactions (
    transaction_id UUID NOT NULL,
    created_at TIMESTAMP NOT NULL,
    ...
  ) PARTITION BY RANGE (created_at);

  CREATE TABLE transactions_2025_01 PARTITION OF transactions
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

  CREATE TABLE transactions_2025_02 PARTITION OF transactions
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
  ```
- **Sharding**: Shard by `org_id` for very large deployments (3,500+ orgs)

**Redis**:
- Redis Cluster (6 nodes: 3 master, 3 replica)
- Cache eviction policy: `allkeys-lru`

**MongoDB**:
- Sharded cluster for audit logs
- Shard key: `organizationId`

### 9.3 Caching Strategy

**Cache Layers**:

| Data Type | Cache | TTL | Invalidation |
|---|---|---|---|
| User session | Redis | 24 hours | On logout |
| User profile | Redis | 10 minutes | On update |
| Collection heads | Redis | 1 hour | On create/update |
| Dashboard metrics | Redis | 5 minutes | On new transaction |
| Form schemas | Redis | 30 minutes | On form update |
| Organization config | Redis | 1 hour | On config change |

**Cache Implementation**:
```javascript
async function getUserCollections(userId, orgSchema) {
  const cacheKey = `collections:${orgSchema}:${userId}`;

  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // Cache miss - fetch from database
  const collections = await db.query(
    `SELECT * FROM ${orgSchema}.user_collections WHERE user_id = $1`,
    [userId]
  );

  // Store in cache (10 min TTL)
  await redis.setex(cacheKey, 600, JSON.stringify(collections.rows));

  return collections.rows;
}
```

### 9.4 Load Balancing

**Application Load Balancer (ALB)**:
- Health check endpoint: `GET /health`
- Health check interval: 30 seconds
- Unhealthy threshold: 2 consecutive failures
- Deregistration delay: 30 seconds

**Routing Strategy**:
- Round-robin for general traffic
- Least connections for payment service (long-running webhook processing)

---

## 10. High Availability & Fault Tolerance

### 10.1 SLA Target: 99.95% Uptime

**Allowable Downtime**:
- Monthly: 21.6 minutes
- Annually: 4.38 hours

### 10.2 Redundancy

**Multi-AZ Deployment**:
```
Region: ap-south-1 (Mumbai)
├── AZ-1a
│   ├── Application Servers (3 instances)
│   ├── PostgreSQL Primary
│   └── Redis Primary
├── AZ-1b
│   ├── Application Servers (3 instances)
│   ├── PostgreSQL Read Replica 1
│   └── Redis Replica 1
└── AZ-1c
    ├── Application Servers (2 instances)
    ├── PostgreSQL Read Replica 2
    └── Redis Replica 2
```

**Load Balancer**:
- ALB spans all 3 AZs
- If one AZ fails, traffic routes to remaining AZs

### 10.3 Fault Tolerance Mechanisms

**Circuit Breaker Pattern** (for external services):
```javascript
const circuitBreaker = {
  state: 'CLOSED', // CLOSED, OPEN, HALF_OPEN
  failureCount: 0,
  failureThreshold: 5,
  timeout: 60000, // 1 minute
  lastFailureTime: null,
};

async function callSabPaisaAPI(payload) {
  if (circuitBreaker.state === 'OPEN') {
    const elapsed = Date.now() - circuitBreaker.lastFailureTime;
    if (elapsed > circuitBreaker.timeout) {
      circuitBreaker.state = 'HALF_OPEN';
    } else {
      throw new Error('Circuit breaker OPEN - SabPaisa unavailable');
    }
  }

  try {
    const response = await axios.post('https://api.sabpaisa.in/...', payload);

    // Success - reset circuit breaker
    circuitBreaker.state = 'CLOSED';
    circuitBreaker.failureCount = 0;

    return response.data;
  } catch (error) {
    circuitBreaker.failureCount++;

    if (circuitBreaker.failureCount >= circuitBreaker.failureThreshold) {
      circuitBreaker.state = 'OPEN';
      circuitBreaker.lastFailureTime = Date.now();

      // Alert operations team
      await sendAlert('SabPaisa circuit breaker OPEN');
    }

    throw error;
  }
}
```

**Retry Mechanism** (for transient failures):
```javascript
async function retryWithBackoff(fn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;

      const delay = Math.min(1000 * Math.pow(2, attempt), 30000); // Exponential backoff
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
const result = await retryWithBackoff(() => callSabPaisaAPI(payload));
```

**Database Failover**:
- PostgreSQL automatic failover (RDS Multi-AZ)
- Failover time: < 60 seconds
- Application automatically reconnects

**Message Queue Durability**:
- RabbitMQ mirrored queues (replicated across 3 nodes)
- Messages persisted to disk
- No message loss on node failure

---

## 11. Disaster Recovery

### 11.1 Backup Strategy

**PostgreSQL**:
- Automated daily snapshots (retained for 35 days)
- Point-in-time recovery (PITR) enabled
- Transaction logs archived to S3 every 5 minutes
- Weekly full backups to S3 (Glacier for long-term)

**Redis**:
- RDB snapshots every 6 hours
- AOF (Append-Only File) enabled for durability

**S3**:
- Versioning enabled
- Cross-region replication to ap-southeast-1 (Singapore)

**MongoDB**:
- Automated daily snapshots
- Oplog backup for PITR

### 11.2 Recovery Objectives

| Metric | Target | Description |
|---|---|---|
| **RTO** (Recovery Time Objective) | 4 hours | Maximum downtime acceptable |
| **RPO** (Recovery Point Objective) | 1 hour | Maximum data loss acceptable |

### 11.3 Disaster Scenarios

**Scenario 1: Database Corruption**
1. Identify corruption time (e.g., 10:00 AM)
2. Restore from snapshot (9:00 AM backup)
3. Apply transaction logs from 9:00 AM to 9:55 AM (PITR)
4. Data loss: 5 minutes (within RPO)
5. Downtime: 30-45 minutes (within RTO)

**Scenario 2: Region Failure (Mumbai)**
1. Failover to DR region (Singapore)
2. Update DNS to point to Singapore ALB
3. Restore database from cross-region replica
4. Downtime: 2-3 hours (within RTO)

**Scenario 3: Ransomware Attack**
1. Isolate affected systems
2. Restore from immutable S3 backups
3. Verify data integrity
4. Resume operations
5. Downtime: 3-4 hours (within RTO)

### 11.4 Disaster Recovery Testing

**Quarterly DR Drills**:
- Simulate database failure and restore
- Simulate region failure and failover
- Validate backup restoration
- Document lessons learned

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| **Multi-Tenancy** | Architecture where a single instance serves multiple customers (tenants) |
| **Schema-per-Tenant** | Each tenant gets a separate database schema for isolation |
| **RBAC** | Role-Based Access Control - permissions based on user roles |
| **JWT** | JSON Web Token - compact token for authentication |
| **Circuit Breaker** | Design pattern to prevent cascading failures |
| **Exponential Backoff** | Retry strategy with increasing delays |
| **PITR** | Point-in-Time Recovery - restore database to any point in time |
| **RTO** | Recovery Time Objective - maximum acceptable downtime |
| **RPO** | Recovery Point Objective - maximum acceptable data loss |

---

## Appendix B: References

- [PostgreSQL Multi-Tenancy](https://www.postgresql.org/docs/current/ddl-schemas.html)
- [Microservices Patterns](https://microservices.io/patterns/index.html)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [SabPaisa API Documentation](https://developer.sabpaisa.in/)

---

**Document Control**:
- **Next Review Date**: 2025-04-15
- **Change Log**: See version control system
- **Approvers**: CTO, Head of Engineering, Solutions Architect

---

*End of Architecture Document*
