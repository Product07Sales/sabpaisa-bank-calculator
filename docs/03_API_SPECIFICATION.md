# 03 - API SPECIFICATION
**Universal Digital Collection Platform**

---

| **Document Version** | **Date** | **Author** | **Status** |
|---|---|---|---|
| v1.0 | 2025-01-15 | API Team | ✅ Approved |

---

## Table of Contents
1. [API Overview](#api-overview)
2. [Authentication](#authentication)
3. [Rate Limiting](#rate-limiting)
4. [Auth APIs](#auth-apis)
5. [User Management APIs](#user-management-apis)
6. [Form Management APIs](#form-management-apis)
7. [Collection APIs](#collection-apis)
8. [Payment APIs](#payment-apis)
9. [Offline Payment APIs](#offline-payment-apis)
10. [Receipt APIs](#receipt-apis)
11. [Report APIs](#report-apis)
12. [Notification APIs](#notification-apis)
13. [Admin Configuration APIs](#admin-configuration-apis)
14. [Error Codes](#error-codes)
15. [Webhooks](#webhooks)

---

## 1. API Overview

### 1.1 Base URL Structure
```
Production:  https://api.platform.com/v1
Staging:     https://api-staging.platform.com/v1
Development: http://localhost:3000/api/v1
```

### 1.2 API Principles
- **RESTful Design**: Standard HTTP methods (GET, POST, PUT, DELETE)
- **JSON Payload**: All requests/responses use JSON
- **Versioning**: URL-based versioning (/v1, /v2)
- **HTTPS Only**: Production enforces TLS 1.3
- **Idempotency**: POST/PUT requests support idempotency keys

---

## 2. Authentication

### 2.1 JWT Token Authentication

**Login Flow**:
```
POST /api/v1/auth/login
├─> Returns JWT access token (24hr expiry)
├─> Returns refresh token (30-day expiry)
└─> Client stores token in secure storage
```

**Authorization Header**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Token Payload**:
```json
{
  "userId": "abc-123-def",
  "orgId": "ORG001",
  "role": "END_USER",
  "iat": 1705325000,
  "exp": 1705411400
}
```

---

## 3. Rate Limiting

| User Type | Limit | Window |
|---|---|---|
| End User | 100 requests | per minute |
| Org Admin | 500 requests | per minute |
| Super Admin | 1000 requests | per minute |
| Public Endpoints | 20 requests | per minute per IP |

**Rate Limit Headers**:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705325060
```

**429 Too Many Requests Response**:
```json
{
  "error": "RATE_LIMIT_EXCEEDED",
  "message": "Too many requests. Please try again in 60 seconds.",
  "retryAfter": 60
}
```

---

## 4. Auth APIs

### 4.1 Login
```http
POST /api/v1/auth/login
```

**Request**:
```json
{
  "email": "student@school.com",
  "password": "SecurePass123!",
  "orgId": "ORG001"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGc...",
    "refreshToken": "eyJhbGc...",
    "expiresIn": 86400,
    "user": {
      "userId": "abc-123-def",
      "email": "student@school.com",
      "fullName": "John Doe",
      "role": "END_USER"
    }
  }
}
```

**Errors**:
- `401 UNAUTHORIZED`: Invalid credentials
- `423 LOCKED`: Account locked after 5 failed attempts

---

### 4.2 Refresh Token
```http
POST /api/v1/auth/refresh-token
```

**Request**:
```json
{
  "refreshToken": "eyJhbGc..."
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGc...",
    "expiresIn": 86400
  }
}
```

---

### 4.3 Logout
```http
POST /api/v1/auth/logout
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

---

### 4.4 Forgot Password
```http
POST /api/v1/auth/forgot-password
```

**Request**:
```json
{
  "email": "student@school.com",
  "orgId": "ORG001"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "message": "OTP sent to your email",
  "otpExpiry": "2025-01-15T10:40:00Z"
}
```

---

### 4.5 Reset Password
```http
POST /api/v1/auth/reset-password
```

**Request**:
```json
{
  "email": "student@school.com",
  "otp": "123456",
  "newPassword": "NewPass123!"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "message": "Password reset successful"
}
```

---

## 5. User Management APIs

### 5.1 List Users
```http
GET /api/v1/{orgId}/users?page=1&limit=50&role=END_USER&status=ACTIVE
Authorization: Bearer {token}
```

**Query Parameters**:
| Parameter | Type | Description |
|---|---|---|
| page | number | Page number (default: 1) |
| limit | number | Items per page (max: 100) |
| role | string | Filter by role |
| status | string | Filter by status |
| search | string | Search by name/email/unique_id |

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "userId": "abc-123-def",
        "uniqueId": "STUD001",
        "email": "student1@school.com",
        "fullName": "John Doe",
        "mobile": "9876543210",
        "role": "END_USER",
        "status": "ACTIVE",
        "createdAt": "2025-01-10T10:00:00Z"
      }
    ],
    "pagination": {
      "total": 500,
      "page": 1,
      "limit": 50,
      "totalPages": 10
    }
  }
}
```

---

### 5.2 Create User
```http
POST /api/v1/{orgId}/users
Authorization: Bearer {token}
```

**Request**:
```json
{
  "uniqueId": "STUD002",
  "email": "student2@school.com",
  "password": "Pass123!",
  "fullName": "Jane Smith",
  "mobile": "9876543211",
  "role": "END_USER"
}
```

**Response (201 CREATED)**:
```json
{
  "success": true,
  "data": {
    "userId": "def-456-ghi",
    "uniqueId": "STUD002",
    "email": "student2@school.com"
  }
}
```

**Errors**:
- `409 CONFLICT`: Email or unique_id already exists

---

### 5.3 Bulk Upload Users
```http
POST /api/v1/{orgId}/users/bulk-upload
Authorization: Bearer {token}
Content-Type: multipart/form-data
```

**Request**:
```
------FormBoundary
Content-Disposition: form-data; name="file"; filename="users.csv"
Content-Type: text/csv

uniqueId,email,fullName,mobile,role
STUD003,student3@school.com,Alice Johnson,9876543212,END_USER
STUD004,student4@school.com,Bob Williams,9876543213,END_USER
------FormBoundary--
```

**Response (202 ACCEPTED)**:
```json
{
  "success": true,
  "data": {
    "jobId": "job-789-xyz",
    "status": "PROCESSING",
    "statusUrl": "/api/v1/{orgId}/users/bulk-upload/job-789-xyz/status"
  }
}
```

---

### 5.4 Bulk Upload Status
```http
GET /api/v1/{orgId}/users/bulk-upload/{jobId}/status
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "jobId": "job-789-xyz",
    "status": "COMPLETED",
    "summary": {
      "totalRows": 2,
      "successCount": 2,
      "errorCount": 0
    },
    "errorFileUrl": null,
    "completedAt": "2025-01-15T10:35:00Z"
  }
}
```

---

## 6. Form Management APIs

### 6.1 Create Form
```http
POST /api/v1/{orgId}/forms
Authorization: Bearer {token}
```

**Request (CLOSED Form)**:
```json
{
  "formType": "CLOSED",
  "name": "Tuition Fee Q1 2025",
  "description": "Quarterly tuition fee collection",
  "formSchema": {
    "collectionHeadId": "abc-123-def"
  }
}
```

**Request (OPEN Form)**:
```json
{
  "formType": "OPEN",
  "name": "Donation Form",
  "description": "Accept donations for library",
  "formSchema": {
    "fields": [
      {"name": "fullName", "label": "Full Name", "type": "text", "required": true},
      {"name": "email", "label": "Email", "type": "email", "required": true},
      {"name": "amount", "label": "Donation Amount", "type": "number", "required": true}
    ]
  }
}
```

**Response (201 CREATED)**:
```json
{
  "success": true,
  "data": {
    "formId": "form-456-xyz",
    "publicUrl": "https://platform.com/forms/abc123xyz" // Only for OPEN forms
  }
}
```

---

### 6.2 Get Form Schema
```http
GET /api/v1/{orgId}/forms/{formId}
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "formId": "form-456-xyz",
    "formType": "OPEN",
    "name": "Donation Form",
    "formSchema": {...},
    "publicUrl": "https://platform.com/forms/abc123xyz",
    "isActive": true
  }
}
```

---

### 6.3 Submit Form (Public Endpoint)
```http
POST /api/v1/forms/{formPublicId}/submit
```

**Request**:
```json
{
  "fullName": "John Doe",
  "email": "john@example.com",
  "amount": 5000,
  "mobile": "9876543210"
}
```

**Response (201 CREATED)**:
```json
{
  "success": true,
  "message": "Form submitted successfully",
  "data": {
    "submissionId": "sub-789-abc",
    "collectionId": "col-123-def"
  }
}
```

---

## 7. Collection APIs

### 7.1 Get User Collections
```http
GET /api/v1/{orgId}/users/{userId}/collections?status=PENDING
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "collections": [
      {
        "collectionId": "col-123-def",
        "collectionHead": {
          "code": "TUITION_FEE",
          "name": "Tuition Fee"
        },
        "amount": 15000.00,
        "paidAmount": 0.00,
        "balance": 15000.00,
        "dueDate": "2025-02-01",
        "lateFee": 0.00,
        "status": "PENDING"
      }
    ],
    "summary": {
      "totalDue": 15000.00,
      "totalPaid": 0.00,
      "totalBalance": 15000.00
    }
  }
}
```

---

### 7.2 Create Collection
```http
POST /api/v1/{orgId}/collections
Authorization: Bearer {token}
```

**Request**:
```json
{
  "userId": "abc-123-def",
  "collectionHeadId": "col-head-456",
  "amount": 15000.00,
  "dueDate": "2025-02-01",
  "lateFeePercent": 5.0
}
```

**Response (201 CREATED)**:
```json
{
  "success": true,
  "data": {
    "collectionId": "col-789-xyz"
  }
}
```

---

## 8. Payment APIs

### 8.1 Initiate Payment
```http
POST /api/v1/{orgId}/payments/initiate
Authorization: Bearer {token}
```

**Request**:
```json
{
  "collectionId": "col-123-def",
  "amount": 15000.00,
  "paymentMode": "ONLINE"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "transactionId": "txn-789-abc",
    "sabpaisaUrl": "https://api.sabpaisa.in/pg/v1/payment",
    "encryptedPayload": "a1b2c3d4e5f6...",
    "expiresAt": "2025-01-15T11:00:00Z"
  }
}
```

**Client Action**:
1. Redirect user to `sabpaisaUrl` with `encryptedPayload`
2. SabPaisa handles payment
3. SabPaisa sends webhook to platform
4. Platform updates transaction status
5. User redirected to success/failure page

---

### 8.2 Payment Callback (from SabPaisa)
```http
POST /api/v1/payments/webhook/sabpaisa
Content-Type: application/json
X-SabPaisa-Signature: sha512_hmac_signature
```

**Webhook Payload**:
```json
{
  "clientCode": "SABP001",
  "transactionId": "txn-789-abc",
  "sabpaisaTxnId": "SABP123456789",
  "status": "SUCCESS",
  "amount": 15000.00,
  "paymentMethod": "UPI",
  "upiId": "john@paytm",
  "timestamp": "2025-01-15T10:35:00Z",
  "signature": "a1b2c3..."
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "message": "Webhook processed"
}
```

**Platform Actions**:
1. Verify signature
2. Update transaction status → SUCCESS
3. Update user_collection.paid_amount
4. Trigger receipt generation (async)
5. Send email/SMS notification (async)

---

### 8.3 Get Payment Status
```http
GET /api/v1/{orgId}/payments/{transactionId}/status
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "transactionId": "txn-789-abc",
    "amount": 15000.00,
    "status": "SUCCESS",
    "paymentMode": "ONLINE",
    "paymentMethod": "UPI",
    "initiatedAt": "2025-01-15T10:30:00Z",
    "completedAt": "2025-01-15T10:35:00Z"
  }
}
```

---

### 8.4 Initiate Refund
```http
POST /api/v1/{orgId}/payments/{transactionId}/refund
Authorization: Bearer {token}
```

**Request**:
```json
{
  "amount": 15000.00,
  "reason": "Duplicate payment"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "refundId": "ref-456-xyz",
    "status": "PROCESSING",
    "estimatedCompletionTime": "3-5 business days"
  }
}
```

---

## 9. Offline Payment APIs

### 9.1 Activate Offline Payment
```http
POST /api/v1/{orgId}/offline-payments/activate
Authorization: Bearer {token}
Role: SUPER_ADMIN only
```

**Request**:
```json
{
  "orgId": "ORG001"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "message": "Offline payment activated successfully",
  "data": {
    "sabpaisaOfflineApiKey": "SABP_OFFLINE_KEY_..."
  }
}
```

**Backend Action**:
1. Call SabPaisa `/offlinePayment/activate` API
2. Store returned API key in `org_config` table
3. Set `offline_payment_enabled = true`

---

### 9.2 Record Offline Payment
```http
POST /api/v1/{orgId}/offline-payments
Authorization: Bearer {token}
Role: ORG_ADMIN only
```

**Request**:
```json
{
  "userId": "abc-123-def",
  "collectionId": "col-123-def",
  "amount": 15000.00,
  "paymentMode": "CASH",
  "paymentDate": "2025-01-15",
  "receiptNumber": "REC001",
  "remarks": "Cash received at front desk"
}
```

**Response (201 CREATED)**:
```json
{
  "success": true,
  "data": {
    "offlinePaymentId": "off-789-abc",
    "transactionId": "txn-456-xyz",
    "receiptId": "rec-123-def"
  }
}
```

**Backend Actions**:
1. Create transaction record (status: SUCCESS, payment_mode: OFFLINE)
2. Insert into `offline_payments` table
3. Update `user_collections.paid_amount`
4. Generate receipt with "OFFLINE PAYMENT" watermark
5. Send to SabPaisa for reconciliation
6. Log audit trail (admin name, IP, timestamp)

---

### 9.3 List Offline Payments
```http
GET /api/v1/{orgId}/offline-payments?fromDate=2025-01-01&toDate=2025-01-31
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "offlinePayments": [
      {
        "offlinePaymentId": "off-789-abc",
        "user": {
          "uniqueId": "STUD001",
          "fullName": "John Doe"
        },
        "amount": 15000.00,
        "paymentMode": "CASH",
        "paymentDate": "2025-01-15",
        "receiptNumber": "REC001",
        "enteredBy": {
          "email": "admin@school.com",
          "fullName": "Admin User"
        },
        "createdAt": "2025-01-15T10:30:00Z"
      }
    ]
  }
}
```

---

## 10. Receipt APIs

### 10.1 Generate Receipt
```http
POST /api/v1/{orgId}/receipts/generate
Authorization: Bearer {token}
```

**Request**:
```json
{
  "transactionId": "txn-789-abc"
}
```

**Response (202 ACCEPTED)**:
```json
{
  "success": true,
  "message": "Receipt generation queued",
  "data": {
    "receiptId": "rec-123-def",
    "statusUrl": "/api/v1/{orgId}/receipts/rec-123-def"
  }
}
```

---

### 10.2 Get Receipt
```http
GET /api/v1/{orgId}/receipts/{receiptId}
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "receiptId": "rec-123-def",
    "receiptNumber": "REC/2025/01/ABC/00001",
    "receiptUrl": "https://receipts.platform.com/org001/2025/01/REC00001.pdf",
    "qrCodeUrl": "https://receipts.platform.com/org001/2025/01/REC00001_qr.png",
    "generatedAt": "2025-01-15T10:35:00Z"
  }
}
```

---

### 10.3 Download Receipt PDF
```http
GET /api/v1/{orgId}/receipts/{receiptId}/download
Authorization: Bearer {token}
```

**Response (200 OK)**:
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="REC_2025_01_00001.pdf"

[PDF binary data]
```

---

### 10.4 Email Receipt
```http
POST /api/v1/{orgId}/receipts/{receiptId}/email
Authorization: Bearer {token}
```

**Request**:
```json
{
  "recipientEmail": "student@school.com"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "message": "Receipt emailed successfully"
}
```

---

## 11. Report APIs

### 11.1 Dashboard Analytics
```http
GET /api/v1/{orgId}/dashboard
Authorization: Bearer {token}
```

**Response (200 OK - Org Admin)**:
```json
{
  "success": true,
  "data": {
    "todaySummary": {
      "totalCollections": 150000.00,
      "successRate": 98.5,
      "failedTransactions": 2
    },
    "outstandingDues": {
      "totalDue": 5000000.00,
      "overdueAmount": 250000.00,
      "usersWithDues": 1500
    },
    "collectionPerformance": [
      {"collectionHead": "Tuition Fee", "collected": 100000.00, "pending": 50000.00},
      {"collectionHead": "Hostel Fee", "collected": 50000.00, "pending": 30000.00}
    ],
    "paymentModeBreakdown": {
      "ONLINE": 120000.00,
      "OFFLINE": 30000.00
    }
  }
}
```

---

### 11.2 Transaction Report
```http
GET /api/v1/{orgId}/reports/transactions?fromDate=2025-01-01&toDate=2025-01-31&status=SUCCESS
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "transactions": [
      {
        "transactionId": "txn-789-abc",
        "user": {"uniqueId": "STUD001", "fullName": "John Doe"},
        "amount": 15000.00,
        "paymentMode": "ONLINE",
        "paymentMethod": "UPI",
        "status": "SUCCESS",
        "createdAt": "2025-01-15T10:30:00Z"
      }
    ],
    "summary": {
      "totalTransactions": 150,
      "totalAmount": 2250000.00,
      "averageTransactionValue": 15000.00
    }
  }
}
```

---

### 11.3 Export Report
```http
POST /api/v1/{orgId}/reports/export
Authorization: Bearer {token}
```

**Request**:
```json
{
  "reportType": "TRANSACTION_REPORT",
  "format": "PDF",
  "filters": {
    "fromDate": "2025-01-01",
    "toDate": "2025-01-31",
    "status": "SUCCESS"
  }
}
```

**Response (202 ACCEPTED)**:
```json
{
  "success": true,
  "data": {
    "exportId": "export-123-abc",
    "statusUrl": "/api/v1/{orgId}/reports/export/export-123-abc/status"
  }
}
```

---

## 12. Notification APIs

### 12.1 Send Email (Internal)
```http
POST /api/v1/notifications/email
Authorization: Bearer {internal_service_token}
```

**Request**:
```json
{
  "to": "student@school.com",
  "subject": "Payment Successful",
  "template": "payment_success",
  "data": {
    "userName": "John Doe",
    "amount": 15000.00,
    "transactionId": "txn-789-abc",
    "receiptUrl": "https://receipts.platform.com/..."
  }
}
```

---

## 13. Admin Configuration APIs

### 13.1 Configure Branding
```http
PUT /api/v1/{orgId}/config/branding
Authorization: Bearer {token}
Role: SUPER_ADMIN only
```

**Request**:
```json
{
  "orgLogoUrl": "https://s3.amazonaws.com/logos/school_logo.png",
  "themePrimaryColor": "#0066CC",
  "themeSecondaryColor": "#FFFFFF",
  "receiptTemplate": "FORMAL"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "message": "Branding updated successfully"
}
```

---

### 13.2 Manage Collection Heads
```http
POST /api/v1/{orgId}/collection-heads
Authorization: Bearer {token}
Role: SUPER_ADMIN only
```

**Request**:
```json
{
  "code": "TUITION_FEE",
  "name": "Tuition Fee",
  "description": "Quarterly tuition fee",
  "displayOrder": 1
}
```

**Response (201 CREATED)**:
```json
{
  "success": true,
  "data": {
    "collectionHeadId": "col-head-789"
  }
}
```

---

## 14. Error Codes

| Code | HTTP Status | Description |
|---|---|---|
| `ERR_AUTH_001` | 401 | Invalid credentials |
| `ERR_AUTH_002` | 401 | Token expired |
| `ERR_AUTH_003` | 423 | Account locked |
| `ERR_PAY_001` | 400 | Invalid payment amount |
| `ERR_PAY_002` | 500 | Payment gateway timeout |
| `ERR_PAY_003` | 400 | Transaction already processed |
| `ERR_FORM_001` | 400 | Invalid form schema |
| `ERR_FORM_002` | 404 | Form not found |
| `ERR_UPLOAD_001` | 400 | File size exceeds limit |
| `ERR_UPLOAD_002` | 400 | Invalid CSV format |
| `ERR_PERM_001` | 403 | Insufficient permissions |

**Error Response Format**:
```json
{
  "success": false,
  "error": {
    "code": "ERR_AUTH_001",
    "message": "Invalid email or password",
    "details": null
  }
}
```

---

## 15. Webhooks

### 15.1 SabPaisa Webhook Signature Verification

**Signature Generation** (SabPaisa side):
```javascript
const crypto = require('crypto');

const payload = JSON.stringify(webhookData);
const signature = crypto
  .createHmac('sha512', SABPAISA_SECRET_KEY)
  .update(payload)
  .digest('hex');
```

**Signature Verification** (Platform side):
```javascript
const receivedSignature = req.headers['x-sabpaisa-signature'];
const payload = JSON.stringify(req.body);
const expectedSignature = crypto
  .createHmac('sha512', process.env.SABPAISA_SECRET_KEY)
  .update(payload)
  .digest('hex');

if (receivedSignature !== expectedSignature) {
  return res.status(401).json({ error: 'Invalid signature' });
}
```

---

**Document Control**:
- **Next Review Date**: 2025-04-15
- **Approvers**: API Lead, Backend Team Lead

---

*End of API Specification Document*
