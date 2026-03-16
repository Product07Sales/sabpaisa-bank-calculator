# Split Settlement — Complete Product Guide

## Powering Automated Payment Distribution for Merchants & End Users

---

## Table of Contents

1. [What is Split Settlement?](#what-is-split-settlement)
2. [How It Works — Merchant View](#how-it-works--merchant-view)
3. [How It Works — User/Customer View](#how-it-works--usercustomer-view)
4. [Key Features](#key-features)
5. [Split Settlement Types](#split-settlement-types)
6. [Segment-Specific Use Cases & Examples](#segment-specific-use-cases--examples)
7. [Settlement Flow](#settlement-flow)
8. [Reporting & Dashboard](#reporting--dashboard)
9. [Setup & Onboarding](#setup--onboarding)
10. [FAQs](#faqs)

---

## What is Split Settlement?

Split Settlement enables a **single payment to be automatically distributed to multiple parties** at the time of settlement. Instead of the entire amount landing in one merchant account, the system splits the money and routes each party's share directly to their bank account.

**In simple terms:** One payment in. Multiple payouts out. Fully automated. Zero manual reconciliation.

---

## How It Works — Merchant View

### Before Split Settlement
```
Customer pays Rs 10,000
        ↓
Full amount lands in Merchant's account
        ↓
Merchant manually calculates shares
        ↓
Merchant initiates separate bank transfers to each party
        ↓
Merchant reconciles everything in spreadsheets
```
**Pain points:** Manual effort, reconciliation errors, delayed payments to partners, disputes.

### After Split Settlement
```
Customer pays Rs 10,000
        ↓
System automatically calculates each party's share
        ↓
Each party receives their share directly via NEFT
        ↓
Full audit trail available on dashboard
```
**Benefits:** Zero manual work, instant reconciliation, real-time visibility, no disputes.

### Merchant Benefits at a Glance

| Benefit | Description |
|---------|-------------|
| **Zero Reconciliation** | No spreadsheets. Every split is tracked automatically with a full audit trail |
| **Instant Partner Payments** | Partners and vendors receive their share in the same settlement cycle |
| **Configurable Rules** | Fixed, percentage, dynamic, or remainder-based splits — your business, your rules |
| **Fee Transparency** | Clear visibility into how PG fees are distributed across parties |
| **No Integration Changes** | Works with existing payment setup — just pass the right UDF fields |
| **Dashboard Access** | Real-time view of every split, every transfer, every beneficiary |

---

## How It Works — User/Customer View

From the customer's perspective, **nothing changes**:

1. Customer sees a single payment amount (e.g., Rs 10,000)
2. Customer pays using any supported method — UPI, Net Banking, Cards, Wallets
3. Customer receives a single payment confirmation
4. The split happens entirely behind the scenes during settlement

**The customer never sees the split. Their experience remains seamless.**

---

## Key Features

### 1. Automatic Fee Proration

When PG fees apply, the system **distributes fees proportionally** across all parties — no one bears an unfair share.

**Example — Without Proration (Problem):**
```
Customer pays:      Rs 1,000
PG fees deducted:   Rs    20
Net available:      Rs   980

Party A should get: Rs   600  ← configured split
Party B should get: Rs   400  ← configured split
Total needed:       Rs 1,000  ← but only Rs 980 is available!

Result: SHORTFALL of Rs 20
```

**Example — With Proration (Solution):**
```
Customer pays:      Rs 1,000
PG fees deducted:   Rs    20
Net available:      Rs   980

Party A gets:       Rs   588  (60% of Rs 980)
Party B gets:       Rs   392  (40% of Rs 980)
Total distributed:  Rs   980  ← perfectly balanced

Result: Fees shared proportionally. No shortfall.
```

### 2. Flexible Account Configuration

| Mode | How It Works | Best For |
|------|-------------|----------|
| **Static Accounts** | Beneficiary bank details configured once during setup | Fixed partners, known recipients |
| **Dynamic Accounts** | Bank details passed per transaction via UDF fields | Marketplaces, multi-vendor platforms |
| **Hybrid** | Mix of static and dynamic beneficiaries | Platforms with fixed + variable recipients |

### 3. Remainder Handling

One beneficiary is marked as the **primary recipient** who receives whatever is left after all other splits. This ensures:
- Every rupee is accounted for
- No rounding gaps
- No leftover amounts stuck in limbo

### 4. Full Transfer Lifecycle Tracking

Every split payment is tracked end-to-end:

```
Payment Split → Queued for Transfer → NEFT File Sent to Bank → Confirmed by Bank
```

Failed or returned transfers are automatically flagged for resolution.

### 5. Multi-Beneficiary Support

No hard limit on the number of beneficiaries. Configure as many split recipients as your business requires, each with:
- Priority ordering
- Individual bank account details
- Custom amount rules (fixed, percentage, remainder)

---

## Split Settlement Types

### Type 1: Fixed Split

Each payment is split into **predefined categories** with amounts specified per transaction.

```
┌──────────────────────────────────────────────────┐
│  FIXED SPLIT                                     │
│                                                  │
│  Transaction: Rs 10,000 (College Fee Payment)    │
│                                                  │
│  ├── Rs 8,000 → College Account      [STATIC]   │
│  ├── Rs 1,500 → University Account   [STATIC]   │
│  └── Rs   500 → Platform Fee         [STATIC]   │
│                                                  │
│  Split rules: Configured once, applied to all    │
└──────────────────────────────────────────────────┘
```

**Best for:** Education platforms, fee collection portals with known fee components.

---

### Type 2: Dynamic Account

The full payment amount is routed to a **different bank account per transaction**, specified at payment time.

```
┌──────────────────────────────────────────────────┐
│  DYNAMIC ACCOUNT                                 │
│                                                  │
│  Transaction 1: Rs 5,000 → Seller A's Account   │
│  Transaction 2: Rs 3,000 → Seller B's Account   │
│  Transaction 3: Rs 8,000 → Seller C's Account   │
│                                                  │
│  Account details: Passed in UDF per transaction  │
└──────────────────────────────────────────────────┘
```

**Best for:** Marketplaces, aggregator platforms, multi-vendor portals.

---

### Type 3: Institute-Based Split

Both the **recipient details (account, bank) AND the amounts** are specified per transaction. Maximum flexibility.

```
┌──────────────────────────────────────────────────┐
│  INSTITUTE-BASED SPLIT                           │
│                                                  │
│  Transaction: Rs 5,000 (Government Fee)          │
│                                                  │
│  ├── Rs 3,000 → Dept A (A/c: XXXX1234, IFSC: SBIN...)  │
│  └── Rs 2,000 → Dept B (A/c: XXXX5678, IFSC: HDFC...)  │
│                                                  │
│  Both amounts AND accounts: Passed per txn       │
└──────────────────────────────────────────────────┘
```

**Best for:** Government collections, institutional payments with varying recipients.

---

### Type 4: Multi-Split

Multiple recipients and amounts are encoded in a **compact format** within the transaction, allowing many splits from a single payment.

```
┌──────────────────────────────────────────────────┐
│  MULTI-SPLIT                                     │
│                                                  │
│  Transaction: Rs 15,000 (Consortium Payment)     │
│                                                  │
│  ├── Rs 6,000 → Partner A                       │
│  ├── Rs 4,000 → Partner B                       │
│  ├── Rs 3,000 → Partner C                       │
│  └── Rs 2,000 → Platform                        │
│                                                  │
│  All encoded in compact UDF format               │
└──────────────────────────────────────────────────┘
```

**Best for:** Complex multi-party splits, consortium payments, syndicate models.

---

### Type 5: No Split (Default)

Standard single-beneficiary settlement. The full net amount goes to the merchant's registered bank account.

---

## Segment-Specific Use Cases & Examples

### Education Segment

**Scenario:** A student pays Rs 50,000 as admission fees through a college payment portal.

The fee structure includes tuition, university charges, exam fees, and hostel deposit — each going to a different entity.

```
┌─────────────────────────────────────────────────────────────┐
│  EDUCATION — Admission Fee Split                            │
│                                                             │
│  Student pays: Rs 50,000                                    │
│  PG Fee (1.5%): Rs 750                                      │
│  Net Available: Rs 49,250                                   │
│                                                             │
│  Split (with proration):                                    │
│  ┌─────────────────────┬───────────┬──────────┬───────────┐ │
│  │ Beneficiary         │ Intended  │ Prorated │ Account   │ │
│  ├─────────────────────┼───────────┼──────────┼───────────┤ │
│  │ College (Tuition)   │ Rs 30,000 │ Rs 29,550│ Static    │ │
│  │ University          │ Rs 10,000 │ Rs  9,850│ Static    │ │
│  │ Exam Board          │ Rs  5,000 │ Rs  4,925│ Static    │ │
│  │ Hostel              │ Rs  5,000 │ Rs  4,925│ Static    │ │
│  └─────────────────────┴───────────┴──────────┴───────────┘ │
│                                                             │
│  Split Type: FIXED SPLIT                                    │
│  Primary Beneficiary: College (receives remainder)          │
│  Fee Handling: Prorated across all parties                   │
└─────────────────────────────────────────────────────────────┘
```

**Why it matters:**
- Colleges no longer need to manually transfer university and exam board shares
- University gets its share automatically in the same settlement cycle
- Full audit trail for compliance and reporting

---

### Marketplace / E-Commerce Segment

**Scenario:** A customer buys a product worth Rs 2,500 on a multi-vendor marketplace. The platform charges a 15% commission.

```
┌─────────────────────────────────────────────────────────────┐
│  MARKETPLACE — Order Payment Split                          │
│                                                             │
│  Customer pays: Rs 2,500                                    │
│  PG Fee (2%): Rs 50                                         │
│  Net Available: Rs 2,450                                    │
│                                                             │
│  Split:                                                     │
│  ┌─────────────────────┬───────────┬──────────┬───────────┐ │
│  │ Beneficiary         │ Intended  │ Prorated │ Account   │ │
│  ├─────────────────────┼───────────┼──────────┼───────────┤ │
│  │ Seller (Vendor X)   │ Rs 2,125  │ Rs 2,082 │ Dynamic   │ │
│  │ Platform Commission │ Rs   375  │ Rs   368 │ Static    │ │
│  └─────────────────────┴───────────┴──────────┴───────────┘ │
│                                                             │
│  Split Type: DYNAMIC ACCOUNT                                │
│  Seller account changes per order (passed via UDF)          │
│  Primary Beneficiary: Seller (receives remainder)           │
└─────────────────────────────────────────────────────────────┘
```

**Why it matters:**
- Sellers receive payment directly — no waiting for marketplace to disburse
- Platform commission is automatically deducted and routed
- Each order can route to a different seller's bank account dynamically

---

### Government / Challan Segment

**Scenario:** A citizen pays Rs 1,200 as a challan fee through a government portal. The fee is split across the transport department, state treasury, and a digital convenience fee.

```
┌─────────────────────────────────────────────────────────────┐
│  GOVERNMENT — Challan Payment Split                         │
│                                                             │
│  Citizen pays: Rs 1,200                                     │
│  PG Fee (1%): Rs 12                                         │
│  Net Available: Rs 1,188                                    │
│                                                             │
│  Split:                                                     │
│  ┌──────────────────────┬───────────┬──────────┬──────────┐ │
│  │ Beneficiary          │ Intended  │ Prorated │ Account  │ │
│  ├──────────────────────┼───────────┼──────────┼──────────┤ │
│  │ Transport Dept       │ Rs   800  │ Rs   792 │ Dynamic  │ │
│  │ State Treasury       │ Rs   350  │ Rs   347 │ Dynamic  │ │
│  │ Portal Convenience   │ Rs    50  │ Rs    49 │ Static   │ │
│  └──────────────────────┴───────────┴──────────┴──────────┘ │
│                                                             │
│  Split Type: INSTITUTE-BASED SPLIT                          │
│  Both accounts and amounts vary per challan type            │
│  Primary Beneficiary: Transport Dept (receives remainder)   │
└─────────────────────────────────────────────────────────────┘
```

**Why it matters:**
- Multiple government departments receive their share without manual intervention
- Different challan types can route to different departments dynamically
- Transparent audit trail for government compliance requirements

---

### Aggregator / Service Platform Segment

**Scenario:** A customer books a home service (plumbing) for Rs 800 through a service aggregator app. The aggregator takes a 20% commission and also collects a small insurance fee.

```
┌─────────────────────────────────────────────────────────────┐
│  AGGREGATOR — Service Booking Split                         │
│                                                             │
│  Customer pays: Rs 800                                      │
│  PG Fee (2%): Rs 16                                         │
│  Net Available: Rs 784                                      │
│                                                             │
│  Split:                                                     │
│  ┌──────────────────────┬───────────┬──────────┬──────────┐ │
│  │ Beneficiary          │ Intended  │ Prorated │ Account  │ │
│  ├──────────────────────┼───────────┼──────────┼──────────┤ │
│  │ Service Provider     │ Rs   610  │ Rs   598 │ Dynamic  │ │
│  │ Platform Commission  │ Rs   160  │ Rs   157 │ Static   │ │
│  │ Insurance Pool       │ Rs    30  │ Rs    29 │ Static   │ │
│  └──────────────────────┴───────────┴──────────┴──────────┘ │
│                                                             │
│  Split Type: DYNAMIC ACCOUNT                                │
│  Service provider account changes per booking               │
│  Primary Beneficiary: Service Provider (receives remainder) │
└─────────────────────────────────────────────────────────────┘
```

**Why it matters:**
- Service providers get paid faster — same settlement cycle
- Platform commission and insurance deductions are automated
- Each booking can route to a different provider's bank account

---

### Franchise Model Segment

**Scenario:** A customer pays Rs 3,000 at a franchise outlet. Revenue is shared between the franchisor (30%) and franchisee (70%).

```
┌─────────────────────────────────────────────────────────────┐
│  FRANCHISE — Revenue Share Split                            │
│                                                             │
│  Customer pays: Rs 3,000                                    │
│  PG Fee (1.8%): Rs 54                                       │
│  Net Available: Rs 2,946                                    │
│                                                             │
│  Split:                                                     │
│  ┌──────────────────────┬───────────┬──────────┬──────────┐ │
│  │ Beneficiary          │ Intended  │ Prorated │ Account  │ │
│  ├──────────────────────┼───────────┼──────────┼──────────┤ │
│  │ Franchisee (Outlet)  │ Rs 2,100  │ Rs 2,062 │ Static   │ │
│  │ Franchisor (HQ)      │ Rs   900  │ Rs   884 │ Static   │ │
│  └──────────────────────┴───────────┴──────────┴──────────┘ │
│                                                             │
│  Split Type: FIXED SPLIT                                    │
│  Percentage-based: 70/30 revenue share                      │
│  Primary Beneficiary: Franchisee (receives remainder)       │
└─────────────────────────────────────────────────────────────┘
```

**Why it matters:**
- Automatic revenue share — no monthly reconciliation between franchisor and franchisee
- Transparent split visible to both parties
- Eliminates trust issues and payment delays

---

### Housing Society / Maintenance Segment

**Scenario:** A resident pays Rs 5,500 as monthly maintenance. The amount covers society maintenance, sinking fund, parking, and water charges — each managed by different heads.

```
┌─────────────────────────────────────────────────────────────┐
│  HOUSING SOCIETY — Maintenance Split                        │
│                                                             │
│  Resident pays: Rs 5,500                                    │
│  PG Fee (1.5%): Rs 82.50                                    │
│  Net Available: Rs 5,417.50                                 │
│                                                             │
│  Split:                                                     │
│  ┌──────────────────────┬───────────┬──────────┬──────────┐ │
│  │ Beneficiary          │ Intended  │ Prorated │ Account  │ │
│  ├──────────────────────┼───────────┼──────────┼──────────┤ │
│  │ Society Maintenance  │ Rs 3,500  │ Rs 3,447 │ Static   │ │
│  │ Sinking Fund         │ Rs 1,000  │ Rs   985 │ Static   │ │
│  │ Parking Charges      │ Rs   500  │ Rs   493 │ Static   │ │
│  │ Water Charges        │ Rs   500  │ Rs   493 │ Static   │ │
│  └──────────────────────┴───────────┴──────────┴──────────┘ │
│                                                             │
│  Split Type: MULTI-SPLIT                                    │
│  All heads configured as fixed amounts                      │
│  Primary Beneficiary: Society Maintenance (remainder)       │
└─────────────────────────────────────────────────────────────┘
```

**Why it matters:**
- Each fund head receives its designated share automatically
- Society treasurer doesn't need to manually allocate funds
- Clean audit trail for AGM reporting and compliance

---

## Settlement Flow

```
                    ┌─────────────────────┐
                    │  Customer Pays      │
                    │  Rs 10,000          │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  Payment Confirmed  │
                    │  by Payment Gateway │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  Settlement Engine  │
                    │  Calculates Net     │
                    │  (Deducts PG fees,  │
                    │   commission, GST,  │
                    │   reserves)         │
                    └─────────┬───────────┘
                              │
                    Net Amount: Rs 9,750
                              │
                              ▼
                    ┌─────────────────────┐
                    │  Split Rules Engine │
                    │  Applies configured │
                    │  split logic        │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  College A/c │ │  Univ A/c    │ │  Platform    │
     │  Rs 7,800    │ │  Rs 1,500    │ │  Rs 450      │
     └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
            │                │                │
            ▼                ▼                ▼
     ┌─────────────────────────────────────────────┐
     │  NEFT Files Generated & Sent to Bank        │
     └─────────────────────┬───────────────────────┘
                           │
                           ▼
     ┌─────────────────────────────────────────────┐
     │  Each Party Receives Their Share             │
     │  Full tracking: Queued → Sent → Confirmed   │
     └─────────────────────────────────────────────┘
```

---

## Reporting & Dashboard

### For Merchants

| Report | What You See |
|--------|-------------|
| **Settlement Summary** | Total collected, total split, per-beneficiary breakdown |
| **Transaction Detail** | Each payment with its split allocation |
| **Beneficiary Report** | Per-beneficiary totals, transaction count, bank details |
| **Transfer Status** | NEFT tracking — Pending, Sent, Confirmed, Failed |

### For Operations Team

| Capability | Description |
|-----------|-------------|
| **Split Configuration Manager** | View and manage all merchant split rules |
| **Settlement Monitor** | Real-time view of all splits being processed |
| **Failed Transfer Queue** | Flagged transfers requiring investigation |
| **Audit Trail** | Complete history of every split decision and transfer |

---

## Setup & Onboarding

### What's Needed from the Merchant

| Requirement | Details |
|-------------|---------|
| **Split Pattern** | Which of the 5 split types applies to their business |
| **Beneficiary Details** | Name, type, bank account, IFSC (for static accounts) |
| **UDF Mapping** | Which transaction fields carry dynamic split data (if applicable) |
| **Amount Rules** | Fixed amount, percentage, remainder, or from transaction data |
| **Fee Handling Preference** | Prorate fees across all parties OR deduct from one party |
| **Primary Beneficiary** | Who receives the remainder after all splits |

### Onboarding Steps

```
Step 1: Merchant submits split requirements
            ↓
Step 2: Ops team configures split rules on dashboard
            ↓
Step 3: Test transaction to verify split logic
            ↓
Step 4: Go live — splits apply to all future settlements
```

**No integration changes needed** if the merchant is already passing required data in UDF fields during payment.

---

## FAQs

**Q: Does the customer see the split?**
No. The customer pays a single amount. Splitting happens entirely on the backend during settlement.

**Q: Can split rules be changed after setup?**
Yes. Split configurations can be updated anytime by the ops team. Changes apply to future settlements.

**Q: What happens if a split transfer fails?**
The failed transfer is flagged in the system. The ops team can investigate and retry or resolve the issue.

**Q: Can a merchant have different splits for different transactions?**
Yes. With Dynamic Account or Institute-Based patterns, each transaction can specify different recipients and amounts.

**Q: Are there limits on the number of beneficiaries?**
No hard limit. Multiple beneficiaries can be configured with priority ordering.

**Q: Does the merchant need to change their payment integration?**
No, as long as the merchant is already sending the required data in their UDF (User Defined Fields) during payment.

**Q: How are PG fees handled in a split?**
Two options: (1) Prorate fees proportionally across all beneficiaries, or (2) Deduct entirely from the primary beneficiary. This is configurable per merchant.

**Q: Can I see which beneficiary received how much for a specific transaction?**
Yes. The dashboard provides per-transaction split breakdowns with full beneficiary details and transfer status.

**Q: Is there a minimum amount for a split?**
Individual split amounts must be valid for NEFT transfer (typically Rs 1 or above). There is no maximum limit.

**Q: What payment methods are supported?**
All methods supported by the payment gateway — UPI, Net Banking, Credit/Debit Cards, Wallets. The split logic is payment-method agnostic.

---

*Split Settlement — Automating payment distribution so merchants can focus on their business, not reconciliation.*
