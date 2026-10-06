# Salone One

## V1 Engineering Specification

**Version:** 1.0\
**Platform:** Android-first + Web Admin/Merchant\
**Backend:** NestJS / Node.js\
**Database:** PostgreSQL\
**Cache/Queues:** Redis\
**Storage:** S3-compatible object storage\
**Architecture:** Modular monolith initially, service-ready for future extraction

---

## 1 Engineering Objective

Salone One V1 shall provide one unified platform for:

1. Customers
2. Merchants
3. Delivery operators
4. Administrators
5. Payment providers

The first production release shall concentrate on:

- Marketplace
- Merchant management
- Ordering
- Delivery
- Payments
- Protected transaction workflow
- Commission calculation
- Settlement
- Refunds
- Disputes
- Notifications
- Administration

The architecture must allow Food, Transport, Home Services, Property, Tourism, Agriculture, Jobs and other modules to be added without redesigning the financial core.

---

# 2. Core Architectural Principle

The most important rule is:

> **The application owns the transaction state; payment providers own the actual payment rails.**

Salone One must never assume that a client-side payment-success screen means money was received.

Every payment must be independently verified against the provider.

### Supported customer payment methods

```text
ORANGE
AFRIMONEY
QMONEY
BANK
CARD
INTERNATIONAL
```

### Supported merchant settlement methods

```text
ORANGE
AFRIMONEY
QMONEY
BANK
```

A merchant may have multiple settlement accounts.

Example:

```text
Merchant ABC

Primary:
    QMoney

Backup:
    Orange Money

Secondary:
    Afrimoney

Bank:
    Sierra Leone commercial bank
```

The customer may pay using a different provider from the merchant's settlement provider.

Example:

```text
Customer
   |
   | Orange Money
   v
Salone One Payment Orchestrator
   |
   | approved settlement route
   v
Merchant QMoney
```

Whether a particular cross-provider route is available must be determined by the approved provider arrangement rather than assumed by the application. Sierra Leone's National Payments Switch architecture is also relevant to future interoperability; the Bank of Sierra Leone reported an Instant Payments Platform intended to connect banks and mobile-money operators.

---

# 3. System Architecture

```text
                     SALONE ONE
                         |
        +----------------+----------------+
        |                |                |
     Customer         Merchant          Admin
        |                |                |
        +----------------+----------------+
                         |
                    API Gateway
                         |
              +----------+----------+
              |                     |
        Authentication        Application Core
                                    |
       +------------+---------------+-------------+
       |            |               |             |
   Marketplace    Orders       Delivery       Accounts
       |            |               |             |
       +------------+---------------+-------------+
                         |
                 Payment Orchestrator
                         |
       +---------+--------+--------+---------+
       |         |        |        |         |
     Orange   Afrimoney QMoney    Bank    International
       |
       +-------------------------------+
                       |
                 Ledger Engine
                       |
              Settlement Engine
                       |
                Reconciliation
```

---

# 4. Backend Modules

The backend shall initially be a modular monolith.

Recommended modules:

```text
auth
users
businesses
merchants
catalog
orders
delivery
payments
payment-providers
escrow
commissions
ledger
settlements
payouts
refunds
disputes
notifications
fraud
kyc
exchange-rates
audit
admin
```

Each module owns its database entities and business rules.

---

# 5. Database Core

## 5.1 users

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    phone VARCHAR(30) UNIQUE,
    email VARCHAR(255) UNIQUE,
    password_hash TEXT,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    country_code VARCHAR(10),
    preferred_currency VARCHAR(10) DEFAULT 'SLE',
    preferred_language VARCHAR(20) DEFAULT 'en',
    kyc_status VARCHAR(30) DEFAULT 'NOT_REQUIRED',
    account_status VARCHAR(30) DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 6. Businesses

```sql
CREATE TABLE businesses (
    id UUID PRIMARY KEY,
    owner_user_id UUID NOT NULL REFERENCES users(id),
    legal_name VARCHAR(255) NOT NULL,
    trading_name VARCHAR(255),
    business_type VARCHAR(50),
    registration_number VARCHAR(100),
    phone VARCHAR(30),
    email VARCHAR(255),
    address TEXT,
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    verification_status VARCHAR(30) DEFAULT 'PENDING',
    status VARCHAR(30) DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 7. Merchant Payment Accounts

A merchant can have multiple payment/settlement accounts.

```sql
CREATE TABLE merchant_settlement_accounts (
    id UUID PRIMARY KEY,
    merchant_id UUID NOT NULL REFERENCES businesses(id),

    provider VARCHAR(30) NOT NULL,
    account_type VARCHAR(30) NOT NULL,

    account_name VARCHAR(255),
    account_number VARCHAR(100),
    phone_number VARCHAR(30),

    verification_status VARCHAR(30) DEFAULT 'PENDING',

    is_primary BOOLEAN DEFAULT FALSE,
    is_backup BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Provider values:

```text
ORANGE
AFRIMONEY
QMONEY
BANK
```

Sensitive provider credentials must **never** be stored directly in plaintext.

---

# 8. Customer Payment Methods

```sql
CREATE TABLE customer_payment_methods (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),

    provider VARCHAR(30) NOT NULL,
    account_type VARCHAR(30) NOT NULL,

    masked_identifier VARCHAR(100),
    provider_customer_reference VARCHAR(255),

    verification_status VARCHAR(30) DEFAULT 'PENDING',
    is_default BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The database should store provider tokens/references rather than unnecessary sensitive credentials.

---

# 9. Products

```sql
CREATE TABLE products (
    id UUID PRIMARY KEY,
    merchant_id UUID NOT NULL REFERENCES businesses(id),

    name VARCHAR(255) NOT NULL,
    description TEXT,

    category_id UUID,
    sku VARCHAR(100),

    price_minor BIGINT NOT NULL,
    currency VARCHAR(10) NOT NULL DEFAULT 'SLE',

    stock_quantity INTEGER DEFAULT 0,

    status VARCHAR(30) DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Important

Do not use floating-point numbers for financial amounts.

Use integer minor units.

Example:

```text
SLE 1,250
```

stored according to the application's defined currency precision.

---

# 10. Orders

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,

    customer_id UUID NOT NULL REFERENCES users(id),
    merchant_id UUID NOT NULL REFERENCES businesses(id),

    order_number VARCHAR(50) UNIQUE NOT NULL,

    currency VARCHAR(10) NOT NULL,

    subtotal_minor BIGINT NOT NULL,
    delivery_fee_minor BIGINT NOT NULL DEFAULT 0,
    discount_minor BIGINT NOT NULL DEFAULT 0,
    tax_minor BIGINT NOT NULL DEFAULT 0,

    total_minor BIGINT NOT NULL,

    status VARCHAR(40) NOT NULL DEFAULT 'CREATED',

    delivery_address TEXT,
    delivery_latitude DECIMAL(10,7),
    delivery_longitude DECIMAL(10,7),

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 11. Order Items

```sql
CREATE TABLE order_items (
    id UUID PRIMARY KEY,

    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL REFERENCES products(id),

    product_name_snapshot VARCHAR(255) NOT NULL,
    unit_price_minor BIGINT NOT NULL,
    quantity INTEGER NOT NULL,

    total_minor BIGINT NOT NULL
);
```

The snapshot fields are essential.

If a merchant changes the product name or price tomorrow, the historical order must remain unchanged.

---

# 12. Payment Transactions

```sql
CREATE TABLE payment_transactions (
    id UUID PRIMARY KEY,

    order_id UUID NOT NULL REFERENCES orders(id),
    customer_id UUID NOT NULL REFERENCES users(id),
    merchant_id UUID NOT NULL REFERENCES businesses(id),

    provider VARCHAR(30) NOT NULL,

    provider_reference VARCHAR(255),
    internal_reference VARCHAR(255) UNIQUE NOT NULL,

    currency VARCHAR(10) NOT NULL,
    amount_minor BIGINT NOT NULL,

    status VARCHAR(40) NOT NULL,

    failure_code VARCHAR(100),
    failure_reason TEXT,

    initiated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    authorized_at TIMESTAMPTZ,
    confirmed_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 13. Payment Attempts

A single order may have multiple payment attempts.

Example:

```text
Attempt 1 → Orange → FAILED
Attempt 2 → QMoney → FAILED
Attempt 3 → Afrimoney → SUCCESS
```

Therefore:

```sql
CREATE TABLE payment_attempts (
    id UUID PRIMARY KEY,

    payment_id UUID NOT NULL REFERENCES payment_transactions(id),

    provider VARCHAR(30) NOT NULL,

    attempt_number INTEGER NOT NULL,

    provider_reference VARCHAR(255),

    amount_minor BIGINT NOT NULL,
    currency VARCHAR(10) NOT NULL,

    status VARCHAR(40) NOT NULL,

    request_id VARCHAR(255),
    idempotency_key VARCHAR(255),

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
```

---

# 14. Protected Transaction

The application should use the term:

> **Protected Transaction**

rather than automatically calling itself a bank or escrow institution.

The legal implementation must be finalized with the relevant regulated payment/settlement partners.

```sql
CREATE TABLE protected_transactions (
    id UUID PRIMARY KEY,

    payment_id UUID NOT NULL REFERENCES payment_transactions(id),
    order_id UUID NOT NULL REFERENCES orders(id),

    gross_amount_minor BIGINT NOT NULL,
    merchant_amount_minor BIGINT NOT NULL,
    delivery_amount_minor BIGINT NOT NULL,
    commission_amount_minor BIGINT NOT NULL,

    currency VARCHAR(10) NOT NULL,

    status VARCHAR(40) NOT NULL,

    held_at TIMESTAMPTZ,
    release_requested_at TIMESTAMPTZ,
    released_at TIMESTAMPTZ,

    dispute_id UUID,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 15. Commission Rules

```sql
CREATE TABLE commission_rules (
    id UUID PRIMARY KEY,

    service_type VARCHAR(50) NOT NULL,

    percentage_bps INTEGER DEFAULT 0,
    fixed_amount_minor BIGINT DEFAULT 0,

    minimum_fee_minor BIGINT,
    maximum_fee_minor BIGINT,

    effective_from TIMESTAMPTZ NOT NULL,
    effective_until TIMESTAMPTZ,

    active BOOLEAN DEFAULT TRUE
);
```

Use basis points rather than floating percentages.

Example:

```text
8% = 800 basis points
10% = 1000 basis points
12% = 1200 basis points
```

This prevents financial rounding problems.

---

# 16. Commission Entries

```sql
CREATE TABLE commission_entries (
    id UUID PRIMARY KEY,

    order_id UUID NOT NULL REFERENCES orders(id),
    commission_rule_id UUID REFERENCES commission_rules(id),

    base_amount_minor BIGINT NOT NULL,
    commission_amount_minor BIGINT NOT NULL,

    currency VARCHAR(10) NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 17. Double-Entry Ledger

This is one of the most important components of Salone One.

Do not calculate balances simply with:

```text
balance = previous balance + transaction
```

Use a double-entry ledger.

```sql
CREATE TABLE ledger_accounts (
    id UUID PRIMARY KEY,

    account_code VARCHAR(100) UNIQUE NOT NULL,

    owner_type VARCHAR(30),
    owner_id UUID,

    currency VARCHAR(10) NOT NULL,

    account_type VARCHAR(30) NOT NULL,

    status VARCHAR(30) DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Ledger entries:

```sql
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY,

    transaction_reference VARCHAR(255) NOT NULL,

    debit_account_id UUID REFERENCES ledger_accounts(id),
    credit_account_id UUID REFERENCES ledger_accounts(id),

    amount_minor BIGINT NOT NULL,
    currency VARCHAR(10) NOT NULL,

    description TEXT,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Every financial transaction must balance.

---

# 18. Example Financial Flow

Customer purchases:

```text
Product             SLE 1,000
Delivery            SLE    50
-----------------------------
Customer pays       SLE 1,050
```

Commission:

```text
8% × 1,000 = SLE 80
```

Merchant:

```text
SLE 920
```

Driver:

```text
SLE 50
```

Salone One:

```text
SLE 80
```

Ledger concept:

```text
CUSTOMER_PAYMENT
        |
        | SLE 1,050
        v
PROTECTED_FUNDS
        |
        +---- SLE 920 ---> MERCHANT_PAYABLE
        |
        +---- SLE 50 ----> DELIVERY_PAYABLE
        |
        +---- SLE 80 ----> PLATFORM_REVENUE
```

---

# 19. Settlement

```sql
CREATE TABLE settlements (
    id UUID PRIMARY KEY,

    merchant_id UUID NOT NULL REFERENCES businesses(id),

    provider VARCHAR(30) NOT NULL,

    settlement_account_id UUID NOT NULL
        REFERENCES merchant_settlement_accounts(id),

    currency VARCHAR(10) NOT NULL,

    gross_amount_minor BIGINT NOT NULL,
    fees_minor BIGINT NOT NULL DEFAULT 0,
    net_amount_minor BIGINT NOT NULL,

    status VARCHAR(40) NOT NULL,

    requested_at TIMESTAMPTZ,
    processed_at TIMESTAMPTZ,

    provider_reference VARCHAR(255),

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 20. Refunds

```sql
CREATE TABLE refunds (
    id UUID PRIMARY KEY,

    payment_id UUID NOT NULL REFERENCES payment_transactions(id),
    order_id UUID NOT NULL REFERENCES orders(id),

    amount_minor BIGINT NOT NULL,
    currency VARCHAR(10) NOT NULL,

    reason TEXT,

    provider VARCHAR(30),

    status VARCHAR(40) NOT NULL,

    provider_reference VARCHAR(255),

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
```

---

# 21. Disputes

```sql
CREATE TABLE disputes (
    id UUID PRIMARY KEY,

    order_id UUID NOT NULL REFERENCES orders(id),
    customer_id UUID NOT NULL REFERENCES users(id),

    type VARCHAR(50) NOT NULL,

    description TEXT,

    status VARCHAR(40) NOT NULL DEFAULT 'OPEN',

    resolution TEXT,

    resolved_by UUID REFERENCES users(id),

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at TIMESTAMPTZ
);
```

Supported initial dispute types:

```text
PRODUCT_NOT_RECEIVED
WRONG_PRODUCT
DAMAGED_PRODUCT
MISSING_ITEM
SERVICE_NOT_COMPLETED
UNAUTHORIZED_TRANSACTION
DUPLICATE_PAYMENT
OTHER
```

---

# 22. Provider Webhook Events

```sql
CREATE TABLE payment_provider_events (
    id UUID PRIMARY KEY,

    provider VARCHAR(30) NOT NULL,

    event_id VARCHAR(255) NOT NULL,

    provider_reference VARCHAR(255),

    event_type VARCHAR(100),

    payload JSONB NOT NULL,

    signature_verified BOOLEAN DEFAULT FALSE,

    processing_status VARCHAR(40) DEFAULT 'RECEIVED',

    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMPTZ,

    UNIQUE(provider, event_id)
);
```

The unique constraint prevents duplicate webhook processing.

---

# 23. Audit Log

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY,

    actor_user_id UUID REFERENCES users(id),

    action VARCHAR(100) NOT NULL,

    entity_type VARCHAR(100),
    entity_id UUID,

    old_value JSONB,
    new_value JSONB,

    ip_address INET,
    user_agent TEXT,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Administrators must never be able to silently alter financial records.

---

# 24. Order State Machine

```text
CREATED
   |
   v
PAYMENT_PENDING
   |
   +---- PAYMENT_FAILED
   |
   v
PAYMENT_CONFIRMED
   |
   v
PROTECTED
   |
   v
MERCHANT_ACCEPTED
   |
   v
PREPARING
   |
   v
READY_FOR_PICKUP
   |
   v
IN_TRANSIT
   |
   v
DELIVERED
   |
   v
CUSTOMER_CONFIRMED
   |
   v
RELEASE_PENDING
   |
   v
SETTLED
```

Cancellation:

```text
CREATED
PAYMENT_PENDING
PAYMENT_CONFIRMED
PROTECTED
MERCHANT_ACCEPTED
PREPARING
```

may enter:

```text
CANCELLATION_REQUESTED
        |
        v
REFUND_PENDING
        |
        v
REFUNDED
```

Dispute:

```text
PROTECTED
   |
   v
DISPUTED
   |
   +---- REFUND_CUSTOMER
   |
   +---- RELEASE_MERCHANT
   |
   +---- PARTIAL_SETTLEMENT
```

---

# 25. Payment State Machine

```text
INITIATED
    |
    v
PAYMENT_REQUESTED
    |
    +---- EXPIRED
    |
    +---- FAILED
    |
    v
PROVIDER_PROCESSING
    |
    v
PROVIDER_CONFIRMED
    |
    v
VERIFIED
    |
    v
PROTECTED
```

Critical rule:

```text
CLIENT_SUCCESS != PAYMENT_SUCCESS
```

Only this sequence confirms payment:

```text
Provider
   |
   v
Server verification
   |
   v
Amount verified
   |
   v
Currency verified
   |
   v
Provider reference verified
   |
   v
Transaction marked VERIFIED
```

---

# 26. Delivery State Machine

```text
CREATED
   |
   v
SEARCHING_DRIVER
   |
   v
DRIVER_ASSIGNED
   |
   v
DRIVER_ACCEPTED
   |
   v
AT_PICKUP
   |
   v
PICKED_UP
   |
   v
IN_TRANSIT
   |
   v
AT_DESTINATION
   |
   v
OTP_VERIFIED
   |
   v
DELIVERED
```

---

# 27. Delivery Confirmation

Each delivery receives:

```text
delivery_id
pickup_code
delivery_otp
```

The customer receives a one-time delivery code.

Example:

```text
Your Salone One delivery code is:

4827
```

Driver enters:

```text
4827
```

Server verifies:

```text
delivery.status == AT_DESTINATION
AND
OTP matches
AND
OTP not expired
AND
OTP not previously used
```

Then:

```text
DELIVERED
```

The driver must not be able to mark an order delivered merely by pressing a button.

---

# 28. Protected Release Rules

Funds should not be released simply because a merchant presses "Complete".

Default rule:

```text
DELIVERED
       +
Customer confirmation
       =
RELEASE
```

Alternative automatic release:

```text
DELIVERED
+
No dispute
+
Defined protection period expires
=
RELEASE
```

For example, the business may eventually define:

```text
Low-value goods:
automatic release after delivery

High-value goods:
customer confirmation required

Disputed goods:
manual review
```

These time periods should be configurable.

---

# 29. Payment Provider Interface

The application must not contain code such as:

```text
if provider == Orange
...
else if provider == QMoney
...
```

throughout the business logic.

Instead:

```typescript
interface PaymentProvider {
    createPayment(request: CreatePaymentRequest):
        Promise<CreatePaymentResult>;

    verifyPayment(
        reference: string
    ): Promise<VerifyPaymentResult>;

    getPaymentStatus(
        reference: string
    ): Promise<PaymentStatusResult>;

    refundPayment(
        request: RefundRequest
    ): Promise<RefundResult>;

    createPayout(
        request: PayoutRequest
    ): Promise<PayoutResult>;

    verifyPayout(
        reference: string
    ): Promise<PayoutStatusResult>;
}
```

Implementations:

```text
OrangePaymentProvider
AfrimoneyPaymentProvider
QMoneyPaymentProvider
BankPaymentProvider
InternationalPaymentProvider
```

---

# 30. Payment Orchestrator

```typescript
class PaymentOrchestrator {

    async pay(request) {

        const provider =
            this.providerRegistry.get(request.provider);

        return provider.createPayment(request);
    }

    async verify(providerName, reference) {

        const provider =
            this.providerRegistry.get(providerName);

        return provider.verifyPayment(reference);
    }
}
```

The business-order module should only know:

```text
PAY
VERIFY
REFUND
```

It should not need to know how Orange, QMoney or Afrimoney works internally.

---

# 31. API Contract

Base URL:

```text
/api/v1
```

Authentication:

```text
Authorization: Bearer <access_token>
```

Idempotency:

```text
Idempotency-Key: <unique-client-generated-key>
```

Every money-moving POST request should support idempotency.

---

# 32. Authentication API

### Register

```http
POST /api/v1/auth/register
```

Request:

```json
{
  "phone": "+232XXXXXXXX",
  "firstName": "John",
  "lastName": "Doe",
  "countryCode": "SL"
}
```

Response:

```json
{
  "userId": "uuid",
  "verificationRequired": true
}
```

---

### Verify OTP

```http
POST /api/v1/auth/verify-otp
```

```json
{
  "phone": "+232XXXXXXXX",
  "otp": "123456"
}
```

---

# 33. Product API

```http
GET /api/v1/products
```

Query:

```text
category
merchant
search
minPrice
maxPrice
location
page
limit
```

Create:

```http
POST /api/v1/products
```

```json
{
  "name": "Product Name",
  "description": "Description",
  "priceMinor": 100000,
  "currency": "SLE",
  "stockQuantity": 20
}
```

---

# 34. Order API

### Create order

```http
POST /api/v1/orders
```

```json
{
  "merchantId": "uuid",
  "items": [
    {
      "productId": "uuid",
      "quantity": 2
    }
  ],
  "deliveryAddress": "Freetown",
  "deliveryLatitude": 8.48,
  "deliveryLongitude": -13.23
}
```

Response:

```json
{
  "orderId": "uuid",
  "orderNumber": "SO-20261006-000123",
  "subtotalMinor": 200000,
  "deliveryFeeMinor": 10000,
  "totalMinor": 210000,
  "currency": "SLE",
  "status": "CREATED"
}
```

---

# 35. Payment API

### Create payment

```http
POST /api/v1/payments
```

```json
{
  "orderId": "uuid",
  "provider": "ORANGE",
  "currency": "SLE"
}
```

Response:

```json
{
  "paymentId": "uuid",
  "status": "PAYMENT_PENDING",
  "provider": "ORANGE",
  "amountMinor": 210000,
  "currency": "SLE",
  "paymentReference": "SO-PAY-XXXXXXXX"
}
```

---

# 36. Payment Verification

```http
GET /api/v1/payments/{paymentId}
```

Response:

```json
{
  "paymentId": "uuid",
  "orderId": "uuid",
  "provider": "ORANGE",
  "amountMinor": 210000,
  "currency": "SLE",
  "status": "VERIFIED",
  "protectedStatus": "PROTECTED"
}
```

---

# 37. Webhook Endpoints

```text
POST /api/v1/webhooks/orange
POST /api/v1/webhooks/afrimoney
POST /api/v1/webhooks/qmoney
POST /api/v1/webhooks/bank
POST /api/v1/webhooks/international
```

Every webhook must:

1. Authenticate the provider
2. Verify signature where supported
3. Validate provider reference
4. Validate amount
5. Validate currency
6. Check duplicate event
7. Store raw event
8. Process transaction
9. Update ledger
10. Return provider-required response

---

# 38. Commission API

The client must never calculate the final commission.

Bad:

```text
mobile app calculates 8%
```

Correct:

```text
Mobile App
     |
     v
Backend
     |
     v
Commission Engine
     |
     v
Final financial breakdown
```

Response:

```json
{
  "subtotalMinor": 100000,
  "deliveryMinor": 5000,
  "commissionMinor": 8000,
  "merchantNetMinor": 92000,
  "customerTotalMinor": 105000,
  "currency": "SLE"
}
```

---

# 39. Refund API

```http
POST /api/v1/refunds
```

```json
{
  "paymentId": "uuid",
  "amountMinor": 105000,
  "reason": "PRODUCT_NOT_RECEIVED"
}
```

The backend determines:

```text
Can refund?
How much?
Which provider?
Which original payment?
Is there an open dispute?
Has money already settled?
```

---

# 40. Dispute API

```http
POST /api/v1/disputes
```

```json
{
  "orderId": "uuid",
  "type": "PRODUCT_NOT_RECEIVED",
  "description": "The order has not arrived."
}
```

The order becomes:

```text
DISPUTED
```

and the protected transaction becomes:

```text
RELEASE_BLOCKED
```

until resolved.

---

# 41. Settlement API

Merchant:

```http
GET /api/v1/settlements
```

Admin:

```http
POST /api/v1/settlements/{id}/approve
```

Provider:

```text
Settlement request
       |
       v
Provider
       |
       v
Provider reference
       |
       v
Verification
       |
       v
SETTLED
```

---

# 42. Merchant Settlement Preference

```http
PUT /api/v1/merchants/{merchantId}/settlement-preference
```

```json
{
  "primaryAccountId": "uuid",
  "backupAccountId": "uuid"
}
```

Example:

```text
Primary: QMoney
Backup: Orange
```

If QMoney settlement is unavailable, the system can flag the merchant for approved fallback processing rather than silently redirecting funds.

---

# 43. Cross-Network Payment Architecture

This is important for the Salone One concept.

The application should represent:

```text
CUSTOMER_PAYMENT_PROVIDER
```

separately from:

```text
MERCHANT_SETTLEMENT_PROVIDER
```

Example:

```text
Customer:
Orange

Merchant:
QMoney
```

Database:

```text
payment.provider = ORANGE

merchant_settlement_account.provider = QMONEY
```

This allows the architecture to support:

```text
Orange → Orange
Orange → Afrimoney
Orange → QMoney

Afrimoney → Orange
Afrimoney → Afrimoney
Afrimoney → QMoney

QMoney → Orange
QMoney → Afrimoney
QMoney → QMoney
```

But the backend must only enable a route when the actual provider/regulated settlement arrangement supports it.

Afrimoney's current public information, for example, explicitly describes merchant payments and says users can send money to people on other Sierra Leone networks using its unregistered option.

---

# 44. QMoney Integration Boundary

QMoney currently documents QuickPay as:

```text
QuickPay
   |
QR Code Number
   |
Amount
   |
PIN
   |
Confirmation
```

Therefore V1 should support a provider response such as:

```json
{
  "type": "QR_PAYMENT",
  "provider": "QMONEY",
  "reference": "..."
}
```

rather than assuming QMoney has the same API mechanics as Orange.

The official QMoney help material currently documents QuickPay/QR and transaction confirmation through QMoney.

---

# 45. Foreign Customer Architecture

Foreign customers should be represented separately from local mobile-money customers.

Supported methods may include:

```text
VISA
MASTERCARD
INTERNATIONAL_BANK
APPROVED_INTERNATIONAL_PSP
```

The internal transaction currency model should support:

```text
SLE
GBP
USD
EUR
```

but the settlement currency must be explicitly defined.

Example:

```text
Customer sees:

£10.00

        ↓

International payment provider

        ↓

SLE settlement

        ↓

Merchant
```

The exchange rate must be stored with the transaction.

```text
exchange_rate
source
timestamp
source_currency
target_currency
```

Do not use a live approximate exchange rate during settlement.

Foreign-currency checkout and settlement must also be structured around Sierra Leone's applicable payment/FX rules and approved providers rather than simply routing customer funds through a personal bank account.

---

# 46. HSBC Account

The architecture should **not** contain:

```text
customer
   ↓
personal HSBC account
   ↓
Salone One escrow
```

Instead:

```text
Customer
   ↓
Approved payment provider
   ↓
Salone One corporate/regulated settlement structure
   ↓
Merchant
```

A corporate banking relationship may later be used for legitimate business treasury and international settlement functions, subject to the bank's terms and applicable regulation.

---

# 47. Idempotency

Every money-moving request must have an idempotency key.

Example:

```http
Idempotency-Key: 6e1c7c3d-...
```

If the same request arrives twice:

```text
Request 1 → PAYMENT_CREATED
Request 2 → return original result
```

Never create:

```text
Payment A = SLE 100,000
Payment B = SLE 100,000
```

because the user's network connection retried the request.

---

# 48. Financial Invariants

The backend must enforce these rules.

### Invariant 1

```text
payment.amount == order.total
```

### Invariant 2

```text
merchant_amount
+
delivery_amount
+
commission_amount
=
gross_amount
```

### Invariant 3

```text
ledger_debits == ledger_credits
```

### Invariant 4

A settled transaction cannot be settled again.

### Invariant 5

A refunded transaction cannot be refunded beyond the original captured amount.

### Invariant 6

A disputed transaction cannot be released automatically.

### Invariant 7

Provider webhooks are idempotent.

---

# 49. Reconciliation Engine

Daily reconciliation:

```text
Salone One Ledger
        |
        | compare
        v
Orange Report
        |
        +---- MATCH
        |
        +---- MISMATCH

Afrimoney Report
        |
        +---- MATCH
        |
        +---- MISMATCH

QMoney Report
        |
        +---- MATCH
        |
        +---- MISMATCH
```

Mismatch examples:

```text
Payment exists in provider but not Salone One
Payment exists in Salone One but not provider
Wrong amount
Wrong currency
Duplicate transaction
Missing settlement
Unexpected refund
```

These should generate an admin reconciliation case.

---

# 50. Security Requirements

Minimum requirements:

```text
TLS everywhere
JWT access tokens
Refresh tokens
Password hashing
OTP rate limiting
API rate limiting
Webhook signature validation
Encrypted secrets
Database encryption where appropriate
Audit logs
Role-based access control
Admin MFA
Device/session management
Fraud monitoring
Idempotency
Input validation
SQL injection protection
CSRF protection for browser applications
Secure file upload validation
```

---

# 51. Role Model

```text
CUSTOMER
MERCHANT_OWNER
MERCHANT_STAFF
DRIVER
SERVICE_PROVIDER
ADMIN
FINANCE_ADMIN
SUPPORT_AGENT
SUPER_ADMIN
```

Financial permissions should be separated from normal administration.

For example:

```text
SUPPORT_AGENT
    cannot approve settlement

ADMIN
    cannot alter ledger entries

FINANCE_ADMIN
    can approve financial operations

SUPER_ADMIN
    requires MFA + audit
```

---

# 52. V1 Admin Command Center

Dashboard:

```text
Users
Businesses
Orders
Payments
Protected Transactions
Settlements
Refunds
Disputes
Commission
Providers
Reconciliation
Fraud
Audit Logs
System Health
```

Financial dashboard:

```text
Today's GMV
Today's Commission
Protected Funds
Pending Settlement
Completed Settlement
Refunds
Disputes
Provider Failures
Reconciliation Exceptions
```

---

# 53. Notification Events

The notification system should support:

```text
ORDER_CREATED
PAYMENT_PENDING
PAYMENT_CONFIRMED
PAYMENT_FAILED
ORDER_ACCEPTED
ORDER_PREPARING
ORDER_READY
DRIVER_ASSIGNED
ORDER_PICKED_UP
ORDER_NEAR_DESTINATION
DELIVERY_COMPLETED
PAYMENT_RELEASED
REFUND_STARTED
REFUND_COMPLETED
DISPUTE_OPENED
DISPUTE_RESOLVED
SETTLEMENT_COMPLETED
```

Channels:

```text
Push
SMS
Email
In-app
```

---

# 54. Recommended V1 Repository

```text
salone-one/
│
├── apps/
│   ├── mobile/
│   ├── admin/
│   └── merchant/
│
├── backend/
│   ├── src/
│   │   ├── auth/
│   │   ├── users/
│   │   ├── merchants/
│   │   ├── catalog/
│   │   ├── orders/
│   │   ├── delivery/
│   │   ├── payments/
│   │   ├── providers/
│   │   │   ├── orange/
│   │   │   ├── afrimoney/
│   │   │   ├── qmoney/
│   │   │   ├── bank/
│   │   │   └── international/
│   │   ├── protected-transactions/
│   │   ├── commissions/
│   │   ├── ledger/
│   │   ├── settlements/
│   │   ├── refunds/
│   │   ├── disputes/
│   │   ├── reconciliation/
│   │   ├── notifications/
│   │   ├── fraud/
│   │   └── admin/
│   │
│   └── migrations/
│
├── infrastructure/
│   ├── docker/
│   ├── nginx/
│   └── deployment/
│
└── documentation/
    ├── api/
    ├── payments/
    ├── database/
    └── architecture/
```

---

# 55. V1 Development Sequence

The development team should build in this order:

### Phase 1 — Foundation

```text
Authentication
Users
Roles
Businesses
Database
API framework
Logging
Audit
```

### Phase 2 — Marketplace

```text
Products
Categories
Search
Cart
Orders
Merchant dashboard
```

### Phase 3 — Payment Core

```text
Payment abstraction
Payment transactions
Idempotency
Webhooks
Ledger
Commission engine
Protected transactions
```

### Phase 4 — First Payment Provider

Integrate one provider completely.

Recommended engineering strategy:

```text
Provider adapter
    ↓
Sandbox/test
    ↓
Payment creation
    ↓
Verification
    ↓
Webhook
    ↓
Refund
    ↓
Reconciliation
```

### Phase 5 — Additional Providers

```text
Orange
Afrimoney
QMoney
```

Each gets its own adapter.

### Phase 6 — Delivery

```text
Drivers
Assignment
Pickup
GPS
OTP
Delivery completion
```

### Phase 7 — Settlement

```text
Merchant settlement accounts
Payouts
Settlement batching
Reconciliation
```

### Phase 8 — Disputes

```text
Customer dispute
Merchant response
Evidence
Admin resolution
Refund
Partial settlement
```

---

# 56. Definition of Done for V1 Payments

The payment system is not considered production-ready until it can demonstrate:

```text
[✓] Customer selects Orange
[✓] Customer selects Afrimoney
[✓] Customer selects QMoney
[✓] Payment request created
[✓] Provider transaction verified server-side
[✓] Duplicate payment prevented
[✓] Provider webhook processed
[✓] Amount verified
[✓] Currency verified
[✓] Commission calculated server-side
[✓] Protected transaction created
[✓] Delivery completed
[✓] Customer confirmation recorded
[✓] Merchant entitlement calculated
[✓] Delivery entitlement calculated
[✓] Platform commission recorded
[✓] Settlement generated
[✓] Ledger balanced
[✓] Refund supported
[✓] Dispute blocks release
[✓] Reconciliation detects mismatches
[✓] Complete audit trail exists
```

---

# 57. Most Important Engineering Decision

The **ledger + payment orchestrator + protected-transaction engine** should be built before adding ten different business modules.

That gives Salone One this structure:

```text
                    SALONE ONE
                        |
             ┌──────────┴──────────┐
             |                     |
        BUSINESS MODULES       FINANCIAL CORE
             |                     |
    ┌────────┼────────┐      ┌─────┼─────┐
    |        |        |      |     |     |
Marketplace Food   Services  Payment Ledger Settlement
    |        |        |      |     |     |
    └────────┴────────┘      └─────┴─────┘
```

Then every future service uses the same financial infrastructure.

For example:

```text
Marketplace
    → Payment
    → Protected Transaction
    → Delivery
    → Settlement

Food
    → Payment
    → Protected Transaction
    → Delivery
    → Settlement

Home Service
    → Payment
    → Milestone Protection
    → Service Completion
    → Settlement

Tourism
    → Payment
    → Booking Protection
    → Check-in
    → Settlement

Agriculture
    → Payment
    → Delivery
    → Settlement
```

This is what prevents Salone One from becoming ten separate systems glued together later.

---

# 58. Engineering Principle

The final architecture should be:

```text
ONE IDENTITY
      +
ONE ORDER ENGINE
      +
ONE PAYMENT ORCHESTRATOR
      +
ONE PROTECTED-TRANSACTION ENGINE
      +
ONE LEDGER
      +
ONE SETTLEMENT ENGINE
      +
ONE ADMIN COMMAND CENTER
      +
MANY BUSINESS MODULES
```

That is the foundation for the complete Salone One platform.
