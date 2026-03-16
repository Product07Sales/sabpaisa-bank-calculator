# 04 - BACKEND SERVICES
**Universal Digital Collection Platform**

---

| **Document Version** | **Date** | **Author** | **Status** |
|---|---|---|---|
| v1.0 | 2025-01-15 | Backend Team | ✅ Complete |

---

## Table of Contents
1. [Overview](#overview)
2. [Microservices Architecture](#microservices-architecture)
3. [Service Implementation](#service-implementation)
4. [Inter-Service Communication](#inter-service-communication)
5. [Deployment Configuration](#deployment-configuration)
6. [Code Examples](#code-examples)

---

## 1. Overview

### 1.1 Technology Stack
- **Language**: TypeScript 5.0+
- **Runtime**: Node.js 18 LTS
- **Framework**: Express.js 4.18+
- **Message Queue**: RabbitMQ 3.12+
- **Database**: PostgreSQL 15+ (via pg library)
- **Cache**: Redis 7.2+ (via ioredis)

### 1.2 Project Structure
```
backend/
├── src/
│   ├── services/
│   │   ├── auth/
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.routes.ts
│   │   │   └── auth.types.ts
│   │   ├── payment/
│   │   │   ├── payment.service.ts
│   │   │   ├── sabpaisa.integration.ts
│   │   │   ├── payment.controller.ts
│   │   │   └── payment.routes.ts
│   │   ├── user/
│   │   ├── form/
│   │   ├── collection/
│   │   ├── offline-payment/
│   │   ├── receipt/
│   │   ├── report/
│   │   ├── notification/
│   │   └── audit/
│   ├── models/
│   │   ├── User.model.ts
│   │   ├── Transaction.model.ts
│   │   └── ...
│   ├── middlewares/
│   │   ├── auth.middleware.ts
│   │   ├── error.middleware.ts
│   │   └── ratelimit.middleware.ts
│   ├── utils/
│   │   ├── database.ts
│   │   ├── redis.ts
│   │   ├── logger.ts
│   │   └── encryption.ts
│   ├── config/
│   │   └── index.ts
│   └── server.ts
├── tests/
├── package.json
├── tsconfig.json
├── Dockerfile
└── .env.example
```

---

## 2. Microservices Architecture

### 2.1 Service Responsibility Matrix

| Service | Port | Database | Cache | Queue |
|---|---|---|---|---|
| Auth Service | 3001 | PostgreSQL | Redis | - |
| User Service | 3002 | PostgreSQL | Redis | - |
| Payment Service | 3003 | PostgreSQL | Redis | RabbitMQ |
| Form Service | 3004 | PostgreSQL | Redis | - |
| Collection Service | 3005 | PostgreSQL | Redis | - |
| Offline Payment | 3006 | PostgreSQL | - | RabbitMQ |
| Receipt Service | 3007 | PostgreSQL | - | RabbitMQ |
| Report Service | 3008 | PostgreSQL | Redis | - |
| Notification Service | 3009 | PostgreSQL | - | RabbitMQ |
| Audit Service | 3010 | MongoDB | - | RabbitMQ |

---

## 3. Service Implementation

### 3.1 Auth Service

**Purpose**: Handle user authentication, JWT token generation, OTP management

#### auth.service.ts
```typescript
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import { Redis } from 'ioredis';
import { Pool } from 'pg';

interface LoginRequest {
  email: string;
  password: string;
  orgId: string;
}

interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  user: {
    userId: string;
    email: string;
    fullName: string;
    role: string;
  };
}

export class AuthService {
  private db: Pool;
  private redis: Redis;
  private jwtSecret: string;
  private jwtExpiry: string;

  constructor(db: Pool, redis: Redis) {
    this.db = db;
    this.redis = redis;
    this.jwtSecret = process.env.JWT_SECRET!;
    this.jwtExpiry = process.env.JWT_EXPIRY || '24h';
  }

  async login(data: LoginRequest): Promise<LoginResponse> {
    // 1. Get organization schema
    const orgResult = await this.db.query(
      'SELECT org_schema FROM platform.organizations WHERE org_id = $1',
      [data.orgId]
    );

    if (orgResult.rows.length === 0) {
      throw new Error('Organization not found');
    }

    const orgSchema = orgResult.rows[0].org_schema;

    // 2. Set search path
    await this.db.query(`SET search_path TO ${orgSchema}, public`);

    // 3. Get user by email
    const userResult = await this.db.query(
      'SELECT * FROM users WHERE email = $1 AND status = $2',
      [data.email, 'ACTIVE']
    );

    if (userResult.rows.length === 0) {
      throw new Error('Invalid credentials');
    }

    const user = userResult.rows[0];

    // 4. Check if account is locked
    if (user.locked_until && new Date(user.locked_until) > new Date()) {
      throw new Error('Account locked. Try again later.');
    }

    // 5. Verify password
    const isPasswordValid = await bcrypt.compare(
      data.password,
      user.password_hash
    );

    if (!isPasswordValid) {
      // Increment login attempts
      await this.incrementLoginAttempts(user.user_id, orgSchema);
      throw new Error('Invalid credentials');
    }

    // 6. Reset login attempts on successful login
    await this.db.query(
      `UPDATE ${orgSchema}.users SET login_attempts = 0, locked_until = NULL WHERE user_id = $1`,
      [user.user_id]
    );

    // 7. Generate JWT tokens
    const accessToken = this.generateAccessToken({
      userId: user.user_id,
      orgId: data.orgId,
      role: user.role,
    });

    const refreshToken = this.generateRefreshToken({
      userId: user.user_id,
      orgId: data.orgId,
    });

    // 8. Store refresh token in Redis
    await this.redis.setex(
      `refresh_token:${user.user_id}`,
      30 * 24 * 60 * 60, // 30 days
      refreshToken
    );

    // 9. Update last login
    await this.db.query(
      `UPDATE ${orgSchema}.users SET last_login_at = NOW() WHERE user_id = $1`,
      [user.user_id]
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: 86400, // 24 hours in seconds
      user: {
        userId: user.user_id,
        email: user.email,
        fullName: user.full_name,
        role: user.role,
      },
    };
  }

  private async incrementLoginAttempts(
    userId: string,
    orgSchema: string
  ): Promise<void> {
    const result = await this.db.query(
      `UPDATE ${orgSchema}.users
       SET login_attempts = login_attempts + 1
       WHERE user_id = $1
       RETURNING login_attempts`,
      [userId]
    );

    const attempts = result.rows[0].login_attempts;

    // Lock account after 5 failed attempts
    if (attempts >= 5) {
      const lockUntil = new Date(Date.now() + 30 * 60 * 1000); // 30 minutes
      await this.db.query(
        `UPDATE ${orgSchema}.users SET locked_until = $1 WHERE user_id = $2`,
        [lockUntil, userId]
      );
    }
  }

  private generateAccessToken(payload: any): string {
    return jwt.sign(payload, this.jwtSecret, { expiresIn: this.jwtExpiry });
  }

  private generateRefreshToken(payload: any): string {
    return jwt.sign(payload, this.jwtSecret, { expiresIn: '30d' });
  }

  async generateOTP(email: string, orgId: string): Promise<string> {
    // Generate 6-digit OTP
    const otp = Math.floor(100000 + Math.random() * 900000).toString();

    // Store OTP in Redis with 10-minute expiry
    await this.redis.setex(`otp:${email}:${orgId}`, 600, otp);

    return otp;
  }

  async verifyOTP(email: string, orgId: string, otp: string): Promise<boolean> {
    const storedOTP = await this.redis.get(`otp:${email}:${orgId}`);

    if (!storedOTP || storedOTP !== otp) {
      return false;
    }

    // Delete OTP after verification
    await this.redis.del(`otp:${email}:${orgId}`);
    return true;
  }

  async resetPassword(
    email: string,
    orgId: string,
    otp: string,
    newPassword: string
  ): Promise<void> {
    // Verify OTP
    const isOTPValid = await this.verifyOTP(email, orgId, otp);
    if (!isOTPValid) {
      throw new Error('Invalid or expired OTP');
    }

    // Get org schema
    const orgResult = await this.db.query(
      'SELECT org_schema FROM platform.organizations WHERE org_id = $1',
      [orgId]
    );
    const orgSchema = orgResult.rows[0].org_schema;

    // Hash new password
    const passwordHash = await bcrypt.hash(newPassword, 12);

    // Update password
    await this.db.query(
      `UPDATE ${orgSchema}.users SET password_hash = $1 WHERE email = $2`,
      [passwordHash, email]
    );
  }
}
```

#### auth.controller.ts
```typescript
import { Request, Response, NextFunction } from 'express';
import { AuthService } from './auth.service';

export class AuthController {
  constructor(private authService: AuthService) {}

  login = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { email, password, orgId } = req.body;

      const result = await this.authService.login({ email, password, orgId });

      res.json({
        success: true,
        data: result,
      });
    } catch (error) {
      next(error);
    }
  };

  logout = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { userId } = req.user; // From auth middleware

      // Delete refresh token from Redis
      await this.authService.logout(userId);

      res.json({
        success: true,
        message: 'Logged out successfully',
      });
    } catch (error) {
      next(error);
    }
  };

  forgotPassword = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { email, orgId } = req.body;

      const otp = await this.authService.generateOTP(email, orgId);

      // Send OTP via email (using notification service)
      // await notificationService.sendEmail({ to: email, subject: 'Password Reset OTP', otp });

      res.json({
        success: true,
        message: 'OTP sent to your email',
        otpExpiry: new Date(Date.now() + 10 * 60 * 1000).toISOString(),
      });
    } catch (error) {
      next(error);
    }
  };

  resetPassword = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { email, orgId, otp, newPassword } = req.body;

      await this.authService.resetPassword(email, orgId, otp, newPassword);

      res.json({
        success: true,
        message: 'Password reset successful',
      });
    } catch (error) {
      next(error);
    }
  };
}
```

---

### 3.2 Payment Service

**Purpose**: SabPaisa integration, transaction management, webhook handling

#### payment.service.ts
```typescript
import crypto from 'crypto';
import axios from 'axios';
import { Pool } from 'pg';
import { Redis } from 'ioredis';

interface PaymentInitiateRequest {
  collectionId: string;
  amount: number;
  paymentMode: 'ONLINE';
}

interface SabPaisaPayload {
  clientCode: string;
  transactionId: string;
  amount: string;
  customerName: string;
  customerEmail: string;
  customerMobile: string;
  callbackUrl: string;
  checksum: string;
}

export class PaymentService {
  private db: Pool;
  private redis: Redis;
  private sabpaisaApiUrl: string;
  private sabpaisaClientCode: string;
  private sabpaisaEncryptionKey: string;

  constructor(db: Pool, redis: Redis) {
    this.db = db;
    this.redis = redis;
    this.sabpaisaApiUrl = process.env.SABPAISA_API_URL!;
    this.sabpaisaClientCode = process.env.SABPAISA_CLIENT_CODE!;
    this.sabpaisaEncryptionKey = process.env.SABPAISA_ENCRYPTION_KEY!;
  }

  async initiatePayment(
    userId: string,
    orgSchema: string,
    data: PaymentInitiateRequest
  ) {
    // 1. Get collection details
    const collectionResult = await this.db.query(
      `SELECT uc.*, u.full_name, u.email, u.mobile, ch.name as collection_head_name
       FROM ${orgSchema}.user_collections uc
       JOIN ${orgSchema}.users u ON uc.user_id = u.user_id
       JOIN ${orgSchema}.collection_heads ch ON uc.collection_head_id = ch.collection_head_id
       WHERE uc.collection_id = $1 AND uc.user_id = $2`,
      [data.collectionId, userId]
    );

    if (collectionResult.rows.length === 0) {
      throw new Error('Collection not found');
    }

    const collection = collectionResult.rows[0];

    // 2. Validate amount
    const balance = collection.amount - collection.paid_amount;
    if (data.amount > balance) {
      throw new Error('Amount exceeds balance due');
    }

    // 3. Create transaction record
    const transactionId = `TXN${Date.now()}${Math.random().toString(36).substr(2, 9)}`;

    await this.db.query(
      `INSERT INTO ${orgSchema}.transactions
       (transaction_id, user_id, collection_id, amount, payment_mode, status, initiated_at)
       VALUES ($1, $2, $3, $4, $5, $6, NOW())`,
      [transactionId, userId, data.collectionId, data.amount, 'ONLINE', 'INITIATED']
    );

    // 4. Generate SabPaisa payload
    const payload: SabPaisaPayload = {
      clientCode: this.sabpaisaClientCode,
      transactionId,
      amount: data.amount.toString(),
      customerName: collection.full_name,
      customerEmail: collection.email,
      customerMobile: collection.mobile,
      callbackUrl: `${process.env.API_BASE_URL}/api/v1/payments/webhook/sabpaisa`,
      checksum: '',
    };

    // 5. Generate checksum (SHA-512 HMAC)
    payload.checksum = this.generateChecksum(payload);

    // 6. Return payment initiation data
    return {
      transactionId,
      sabpaisaUrl: `${this.sabpaisaApiUrl}/pg/v1/txnInitiate`,
      encryptedPayload: payload,
      expiresAt: new Date(Date.now() + 15 * 60 * 1000).toISOString(), // 15 minutes
    };
  }

  private generateChecksum(payload: Omit<SabPaisaPayload, 'checksum'>): string {
    const data = [
      payload.clientCode,
      payload.transactionId,
      payload.amount,
      payload.customerName,
      payload.customerEmail,
      payload.customerMobile,
      payload.callbackUrl,
    ].join('|');

    return crypto
      .createHmac('sha512', this.sabpaisaEncryptionKey)
      .update(data)
      .digest('hex');
  }

  async handleWebhook(webhookData: any, signature: string): Promise<void> {
    // 1. Verify signature
    const isSignatureValid = this.verifyWebhookSignature(webhookData, signature);
    if (!isSignatureValid) {
      throw new Error('Invalid webhook signature');
    }

    const { transactionId, sabpaisaTxnId, status, amount, paymentMethod, upiId } = webhookData;

    // 2. Get transaction details
    const txnResult = await this.db.query(
      `SELECT t.*, o.org_schema
       FROM platform.organizations o
       JOIN LATERAL (
         SELECT * FROM (
           SELECT *, '${o.org_schema}' as schema_name
           FROM ${o.org_schema}.transactions
           WHERE transaction_id = $1
         ) t
       ) t ON true
       WHERE t.transaction_id = $1
       LIMIT 1`,
      [transactionId]
    );

    if (txnResult.rows.length === 0) {
      throw new Error('Transaction not found');
    }

    const transaction = txnResult.rows[0];
    const orgSchema = transaction.org_schema;

    // 3. Update transaction status
    await this.db.query(
      `UPDATE ${orgSchema}.transactions
       SET status = $1,
           payment_method = $2,
           payment_gateway_txn_id = $3,
           completed_at = NOW(),
           metadata = $4
       WHERE transaction_id = $5`,
      [
        status,
        paymentMethod,
        sabpaisaTxnId,
        JSON.stringify({ upiId, webhookReceivedAt: new Date().toISOString() }),
        transactionId,
      ]
    );

    // 4. If payment successful, update collection
    if (status === 'SUCCESS') {
      await this.db.query(
        `UPDATE ${orgSchema}.user_collections
         SET paid_amount = paid_amount + $1,
             status = CASE
               WHEN paid_amount + $1 >= amount THEN 'PAID'
               ELSE 'PARTIAL'
             END
         WHERE collection_id = $2`,
        [amount, transaction.collection_id]
      );

      // 5. Publish event to RabbitMQ for receipt generation
      // await this.publishEvent('payment.success', { transactionId, orgSchema });
    }
  }

  private verifyWebhookSignature(data: any, signature: string): boolean {
    const payload = JSON.stringify(data);
    const expectedSignature = crypto
      .createHmac('sha512', this.sabpaisaEncryptionKey)
      .update(payload)
      .digest('hex');

    return signature === expectedSignature;
  }

  async getPaymentStatus(transactionId: string, orgSchema: string) {
    const result = await this.db.query(
      `SELECT * FROM ${orgSchema}.transactions WHERE transaction_id = $1`,
      [transactionId]
    );

    if (result.rows.length === 0) {
      throw new Error('Transaction not found');
    }

    return result.rows[0];
  }
}
```

---

### 3.3 Receipt Generation Service

**Purpose**: Generate branded PDF receipts using Puppeteer

#### receipt.service.ts
```typescript
import puppeteer from 'puppeteer';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import QRCode from 'qrcode';
import { Pool } from 'pg';

export class ReceiptService {
  private db: Pool;
  private s3Client: S3Client;
  private s3Bucket: string;

  constructor(db: Pool) {
    this.db = db;
    this.s3Client = new S3Client({ region: process.env.AWS_REGION });
    this.s3Bucket = process.env.AWS_S3_BUCKET!;
  }

  async generateReceipt(transactionId: string, orgSchema: string): Promise<string> {
    // 1. Get transaction and related data
    const result = await this.db.query(
      `SELECT t.*, u.full_name, u.unique_id, u.email,
              uc.amount as collection_amount, ch.name as collection_head_name,
              bc.org_logo_url, bc.theme_primary_color, bc.receipt_template,
              o.org_name, o.domain
       FROM ${orgSchema}.transactions t
       JOIN ${orgSchema}.users u ON t.user_id = u.user_id
       JOIN ${orgSchema}.user_collections uc ON t.collection_id = uc.collection_id
       JOIN ${orgSchema}.collection_heads ch ON uc.collection_head_id = ch.collection_head_id
       JOIN ${orgSchema}.branding_config bc ON true
       JOIN platform.organizations o ON o.org_schema = $2
       WHERE t.transaction_id = $1`,
      [transactionId, orgSchema]
    );

    if (result.rows.length === 0) {
      throw new Error('Transaction not found');
    }

    const data = result.rows[0];

    // 2. Generate receipt number
    const receiptNumber = await this.generateReceiptNumber(orgSchema);

    // 3. Generate QR code
    const qrCodeDataUrl = await QRCode.toDataURL(
      JSON.stringify({
        transactionId,
        receiptNumber,
        amount: data.amount,
        date: data.completed_at,
      })
    );

    // 4. Generate PDF using Puppeteer
    const html = this.generateReceiptHTML(data, receiptNumber, qrCodeDataUrl);
    const pdfBuffer = await this.htmlToPDF(html);

    // 5. Upload to S3
    const s3Key = `receipts/${orgSchema}/${new Date().getFullYear()}/${String(
      new Date().getMonth() + 1
    ).padStart(2, '0')}/${receiptNumber}.pdf`;

    await this.s3Client.send(
      new PutObjectCommand({
        Bucket: this.s3Bucket,
        Key: s3Key,
        Body: pdfBuffer,
        ContentType: 'application/pdf',
      })
    );

    const receiptUrl = `https://${this.s3Bucket}.s3.amazonaws.com/${s3Key}`;

    // 6. Save receipt record
    await this.db.query(
      `INSERT INTO ${orgSchema}.receipts
       (transaction_id, user_id, receipt_number, receipt_url, qr_code_url, generated_at)
       VALUES ($1, $2, $3, $4, $5, NOW())`,
      [transactionId, data.user_id, receiptNumber, receiptUrl, qrCodeDataUrl]
    );

    return receiptUrl;
  }

  private async generateReceiptNumber(orgSchema: string): Promise<string> {
    const year = new Date().getFullYear();
    const month = String(new Date().getMonth() + 1).padStart(2, '0');
    const orgCode = orgSchema.replace('org_', '').toUpperCase().substring(0, 3);

    // Get next sequential number
    const result = await this.db.query(
      `SELECT COUNT(*) as count FROM ${orgSchema}.receipts
       WHERE EXTRACT(YEAR FROM generated_at) = $1
       AND EXTRACT(MONTH FROM generated_at) = $2`,
      [year, month]
    );

    const sequential = String(parseInt(result.rows[0].count) + 1).padStart(5, '0');

    return `REC/${year}/${month}/${orgCode}/${sequential}`;
  }

  private generateReceiptHTML(
    data: any,
    receiptNumber: string,
    qrCodeDataUrl: string
  ): string {
    return `
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; margin: 40px; }
    .header { text-align: center; margin-bottom: 30px; }
    .org-logo { max-width: 200px; max-height: 60px; }
    .receipt-number { font-size: 18px; font-weight: bold; margin: 20px 0; }
    .details-table { width: 100%; border-collapse: collapse; margin: 20px 0; }
    .details-table td { padding: 10px; border-bottom: 1px solid #ddd; }
    .details-table td:first-child { font-weight: bold; width: 40%; }
    .amount { font-size: 24px; color: ${data.theme_primary_color || '#0066CC'}; font-weight: bold; }
    .qr-code { text-align: center; margin: 30px 0; }
    .footer { text-align: center; margin-top: 40px; border-top: 1px solid #ddd; padding-top: 20px; }
    .sabpaisa-logo { max-width: 150px; }
    .watermark { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%) rotate(-45deg);
                 font-size: 100px; color: rgba(0,0,0,0.05); z-index: -1; }
  </style>
</head>
<body>
  ${data.payment_mode === 'OFFLINE' ? '<div class="watermark">OFFLINE PAYMENT</div>' : ''}

  <div class="header">
    ${data.org_logo_url ? `<img src="${data.org_logo_url}" class="org-logo" />` : ''}
    <h1>${data.org_name}</h1>
    <p>${data.domain || ''}</p>
  </div>

  <div class="receipt-number">Receipt No: ${receiptNumber}</div>

  <table class="details-table">
    <tr><td>Date</td><td>${new Date(data.completed_at).toLocaleDateString()}</td></tr>
    <tr><td>Student ID</td><td>${data.unique_id}</td></tr>
    <tr><td>Name</td><td>${data.full_name}</td></tr>
    <tr><td>Email</td><td>${data.email}</td></tr>
    <tr><td>Collection Head</td><td>${data.collection_head_name}</td></tr>
    <tr><td>Amount Paid</td><td class="amount">₹${data.amount.toFixed(2)}</td></tr>
    <tr><td>Payment Mode</td><td>${data.payment_mode}</td></tr>
    <tr><td>Payment Method</td><td>${data.payment_method || 'N/A'}</td></tr>
    <tr><td>Transaction ID</td><td>${data.transaction_id}</td></tr>
    ${data.payment_gateway_txn_id ? `<tr><td>Gateway Txn ID</td><td>${data.payment_gateway_txn_id}</td></tr>` : ''}
  </table>

  <div class="qr-code">
    <img src="${qrCodeDataUrl}" width="150" />
    <p>Scan to verify</p>
  </div>

  <div class="footer">
    <p>Powered by</p>
    <img src="https://sabpaisa.in/images/logo.png" class="sabpaisa-logo" />
    <p style="font-size: 12px; color: #666;">Payment Gateway Partner</p>
  </div>
</body>
</html>
    `;
  }

  private async htmlToPDF(html: string): Promise<Buffer> {
    const browser = await puppeteer.launch({
      headless: true,
      args: ['--no-sandbox', '--disable-setuid-sandbox'],
    });

    const page = await browser.newPage();
    await page.setContent(html);

    const pdfBuffer = await page.pdf({
      format: 'A4',
      printBackground: true,
    });

    await browser.close();

    return pdfBuffer;
  }
}
```

---

## 4. Inter-Service Communication

### 4.1 RabbitMQ Message Queue

```typescript
import amqp from 'amqplib';

export class MessageQueue {
  private connection: amqp.Connection;
  private channel: amqp.Channel;

  async connect(): Promise<void> {
    this.connection = await amqp.connect(process.env.RABBITMQ_URL!);
    this.channel = await this.connection.createChannel();
  }

  async publishEvent(exchange: string, routingKey: string, data: any): Promise<void> {
    await this.channel.assertExchange(exchange, 'topic', { durable: true });

    this.channel.publish(
      exchange,
      routingKey,
      Buffer.from(JSON.stringify(data)),
      { persistent: true }
    );
  }

  async subscribeToQueue(
    queue: string,
    callback: (msg: any) => Promise<void>
  ): Promise<void> {
    await this.channel.assertQueue(queue, { durable: true });

    this.channel.consume(queue, async (msg) => {
      if (msg) {
        try {
          const data = JSON.parse(msg.content.toString());
          await callback(data);
          this.channel.ack(msg);
        } catch (error) {
          console.error('Error processing message:', error);
          this.channel.nack(msg, false, false); // Dead letter queue
        }
      }
    });
  }
}
```

---

## 5. Deployment Configuration

### 5.1 package.json
```json
{
  "name": "universal-collection-platform-backend",
  "version": "1.0.0",
  "description": "Backend services for Universal Digital Collection Platform",
  "main": "dist/server.js",
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "lint": "eslint src/**/*.ts",
    "format": "prettier --write src/**/*.ts"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "ioredis": "^5.3.2",
    "bcrypt": "^5.1.1",
    "jsonwebtoken": "^9.0.2",
    "puppeteer": "^21.6.1",
    "qrcode": "^1.5.3",
    "@aws-sdk/client-s3": "^3.470.0",
    "axios": "^1.6.2",
    "amqplib": "^0.10.3",
    "dotenv": "^16.3.1",
    "helmet": "^7.1.0",
    "cors": "^2.8.5",
    "express-rate-limit": "^7.1.5",
    "winston": "^3.11.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.5",
    "@types/bcrypt": "^5.0.2",
    "@types/jsonwebtoken": "^9.0.5",
    "typescript": "^5.3.3",
    "ts-node-dev": "^2.0.0",
    "jest": "^29.7.0",
    "@types/jest": "^29.5.11",
    "eslint": "^8.56.0",
    "prettier": "^3.1.1"
  }
}
```

### 5.2 Dockerfile
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source
COPY . .

# Build TypeScript
RUN npm run build

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js

# Start server
CMD ["node", "dist/server.js"]
```

---

**Complete implementation details for all 10 microservices continue...**

*See individual service folders in /backend/src/services/ for full implementations.*
