## Payment Gateway Pseudo Design

This document provides a high-level pseudo design for integrating the payment gateway from Paymob in a backend system.
It includes API endpoints, database relations, payment flow, webhook handling, and security checks a backend developer should consider.

---

#### 1. Required Backend Endpoints

##### Description:
These endpoints represent the minimum API routes needed to support the payment process.

---

##### 1.1 Create Payment Order

POST /api/payments/create-order

**Purpose:**

Creates a payment order in the local database and requests a payment session from Paymob.

**Pseudo Logic:**

```
function createPaymentOrder(user_id, order_id):

    order = DB.getOrder(order_id)

    if order.status != "pending":
        return error("Order already processed")

    transaction = DB.createTransaction(
        user_id,
        order_id,
        amount = order.total,
        status = "initiated"
    )

    paymob_token = Paymob.authenticate()

    paymob_order = Paymob.createOrder(
        token = paymob_token,
        amount = order.total,
        merchant_order_id = transaction.id
    )

    payment_key = Paymob.generatePaymentKey(
        order_id = paymob_order.id,
        amount = order.total,
        billing_data = user.billing_info
    )

    return payment_url(payment_key)
```

---

##### 1.2 Webhook Endpoint (Payment Notification)

POST /api/payments/webhook/paymob

**Purpose:**

Receives payment status updates from Paymob.

**Pseudo Logic:**

```
function handlePaymobWebhook(request):

    if not verifySignature(request):
        return error("Invalid webhook signature")

    transaction_id = request.merchant_order_id
    payment_status = request.success

    transaction = DB.getTransaction(transaction_id)

    if transaction is null:
        return error("Transaction not found")

    if payment_status == true:
        transaction.status = "paid"
        updateOrderStatus(transaction.order_id, "paid")
    else:
        transaction.status = "failed"

    DB.save(transaction)

    return success()
```

---

##### 1.3 Payment Status Endpoint (Optional)

GET /api/payments/status/{transaction_id}

**Purpose:**

Allows the mobile app to check the payment result.

**Pseudo Logic:**

```
function getPaymentStatus(transaction_id):

    transaction = DB.getTransaction(transaction_id)

    if not transaction:
        return error("Not found")

    return transaction.status
```

---

#### 2. Required Database Tables

##### 2.1 Users

```
Users
-----
id
name
email
phone
created_at
```

Relationship:

User 1 ---- N Orders

---

##### 2.2 Orders

```
Orders
------
id
user_id
total_amount
currency
status (pending, paid, cancelled)
created_at
```

---

##### 2.3 Transactions

```
Transactions
------------
id
order_id
user_id
paymob_order_id
paymob_transaction_id
amount
currency
status (initiated, pending, paid, failed)
payment_method
created_at
updated_at
```

Relationship:

Order 1 ---- 1 Transaction

---

##### 2.4 WebhookLogs (optional but recommended)

```
WebhookLogs
-----------
id
provider
payload
signature
received_at
processed
```

Purpose: auditing and debugging webhook issues.

---

#### 3. Payment Flow

##### Step 1: User Initiates Payment

1. User selects **Pay with Card / Wallet** in mobile app.
2. Mobile app calls:

```
POST /api/payments/create-order
```

---

##### Step 2: Backend Creates Paymob Order

Backend performs:

1. Create local transaction.
2. Authenticate with Paymob.
3. Create order in Paymob.
4. Generate payment key.
5. Return payment URL or iframe token to mobile app.

---

##### Step 3: User Completes Payment

- User completes payment through Paymob's hosted payment page.
- Paymob processes the payment and contacts the bank.

---

##### Step 4: Paymob Sends Webhook

Paymob calls:

```
POST /api/payments/webhook/paymob
```

Payload includes:

- transaction_id
- success
- amount
- merchant_order_id
- signature

---

##### Step 5: Backend Validates Webhook

Backend must:

- verify signature
- validate transaction exists
- confirm amount matches
- confirm order not already paid

Then update:

- Transaction.status
- Order.status

---

##### Step 6: Mobile App Checks Status

Mobile app can call:

```
GET /api/payments/status/{transaction_id}
```

to confirm payment.

---

#### 4. Necessary Security Checks

##### 4.1 Webhook Signature Verification

Always verify Paymob's webhook signature.

```
if signature != expected_hash:
    reject request
```

---

##### 4.2 Amount Verification

Confirm the amount matches your local order.

```
if webhook.amount != order.total:
    mark as suspicious
```

---

##### 4.3 Idempotency Protection

Prevent duplicate processing.

```
if transaction.status == "paid":
    ignore webhook
```

---

##### 4.4 Order Ownership Validation

Ensure user owns the order.

```
if order.user_id != authenticated_user.id:
    reject request
```

---

##### 4.5 Payment Expiration

Prevent paying expired orders.

```
if order.created_at > 30 minutes:
    reject payment
```

---

##### 4.6 Logging and Monitoring

Store webhook events:

```
save payload in WebhookLogs
```

This helps debugging payment failures.

---

#### 5. Simplified Payment Flow Diagram

```
Mobile App
    |
    v
Backend API
    |
    v
Create Paymob Order
    |
    v
Paymob Payment Page
    |
User completes payment
    |
    v
Paymob Webhook
    |
    v
Backend verifies signature
    |
    v
Update Transaction + Order
    |
    v
Mobile App checks status
```
