# Payment System Design

## Table of Contents

### Section 1: Fundamentals & Requirements
- [Q1: What is a payment system and what are its core components?](#q1)
- [Q2: How do you gather requirements for a payment system?](#q2)
- [Q3: What is the difference between pay-in and pay-out flows?](#q3)
- [Q4: What back-of-the-envelope estimations would you make?](#q4)
- [Q5: What are card schemes and how does the payment ecosystem work?](#q5)

### Section 2: High-Level Architecture
- [Q6: How would you design the high-level architecture for the pay-in flow?](#q6)
- [Q7: How does the pay-out flow differ from pay-in?](#q7)
- [Q8: How do you design the payment service API?](#q8)
- [Q9: What data model would you use for the payment service?](#q9)
- [Q10: What is the double-entry ledger system?](#q10)

### Section 3: PSP Integration & Hosted Payments
- [Q11: What is a PSP and how do you integrate with one?](#q11)
- [Q12: How does the hosted payment page flow work?](#q12)
- [Q13: How do you handle payment processing delays?](#q13)

### Section 4: Reliability & Exactly-Once Delivery
- [Q14: How do you handle failed payments?](#q14)
- [Q15: What retry strategies are appropriate for payment systems?](#q15)
- [Q16: What are retry queues and dead letter queues in a payment context?](#q16)
- [Q17: How do you achieve exactly-once delivery in payments?](#q17)
- [Q18: What is idempotency and how do you implement it?](#q18)

### Section 5: Consistency & Reconciliation
- [Q19: How do you maintain data consistency across payment services?](#q19)
- [Q20: What is reconciliation and how does it work?](#q20)
- [Q21: Synchronous vs asynchronous communication in payment systems?](#q21)

### Section 6: Security & Advanced Topics
- [Q22: What security measures should a payment system implement?](#q22)
- [Q23: How do you design monitoring and alerting for payments?](#q23)
- [Q24: How do you handle currency exchange in a global payment system?](#q24)
- [Q25: How would you handle alternative payment methods?](#q25)

---

## Section 1: Fundamentals & Requirements

<a id="q1"></a>
### Q1: What is a payment system and what are its core components?

**Answer:**

A payment system is any system used to settle financial transactions through the transfer of monetary value. For an e-commerce platform like Amazon, the payment system handles everything related to money movement when a customer places an order.

```
┌───────────────────────────────────────────────────────────────┐
│                       PAYMENT SYSTEM                          │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────┐   ┌─────────┐   ┌─────────────────────────────┐ │
│  │  User   │──▶│ Payment │──▶│     Payment Executor        │ │
│  │ (Buyer) │   │ Service │   │  (Executes single orders)   │ │
│  └─────────┘   └────┬────┘   └──────────────┬──────────────┘ │
│                     │                        │                │
│                     │                        ▼                │
│                     │        ┌───────────────────────────┐    │
│                     │        │   PSP (Stripe, Braintree) │    │
│                     │        │   Moves money between     │    │
│                     │        │   accounts via card scheme │    │
│                     │        └───────────────────────────┘    │
│                     │                                         │
│                     ▼                                         │
│  ┌──────────────────────────────────────┐                     │
│  │         Post-Payment Services        │                     │
│  │  ┌──────────┐       ┌─────────────┐  │                     │
│  │  │  Ledger  │       │   Wallet    │  │                     │
│  │  │ (Records │       │ (Account    │  │                     │
│  │  │  debit/  │       │  balances)  │  │                     │
│  │  │  credit) │       │             │  │                     │
│  │  └──────────┘       └─────────────┘  │                     │
│  └──────────────────────────────────────┘                     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Core Components:**

| Component | Responsibility |
|-----------|----------------|
| **Payment Service** | Accepts payment events, coordinates the payment process, performs risk checks (AML/CFT) |
| **Payment Executor** | Executes a single payment order via a PSP; a payment event may contain multiple orders |
| **PSP** | Third-party provider (Stripe, Braintree, Square) that moves money between accounts |
| **Card Schemes** | Organizations (Visa, MasterCard) that process credit card operations |
| **Ledger** | Financial record of all transactions using double-entry bookkeeping |
| **Wallet** | Tracks account balances for merchants/sellers |

---

<a id="q2"></a>
### Q2: How do you gather requirements for a payment system?

**Answer:**

In a system design interview, you should clarify scope through targeted questions before designing anything.

**Key questions to ask the interviewer:**

1. What kind of payment system? (backend for e-commerce, digital wallet, P2P)
2. What payment options? (credit cards, PayPal, bank transfer)
3. Do we process card payments ourselves or use a third-party PSP?
4. Do we store credit card data? (PCI DSS implications)
5. Is the application global? (multiple currencies?)
6. How many transactions per day?
7. Do we need pay-out support? (paying sellers/merchants)

**Functional Requirements:**

| Requirement | Description |
|------------|-------------|
| **Pay-in flow** | Receive money from customers on behalf of sellers |
| **Pay-out flow** | Send money to sellers around the world |

**Non-functional Requirements:**

| Requirement | Why It Matters |
|------------|----------------|
| **Reliability** | Failed payments must be carefully handled; money cannot be lost |
| **Fault tolerance** | System must gracefully handle component failures |
| **Reconciliation** | Async verification that payment info is consistent across internal services (payment system, accounting) and external services (PSPs) |
| **Consistency** | Financial data must be accurate across all services |
| **Security** | PCI DSS compliance, fraud detection, encryption |

The key insight: for a payment system, **correctness matters far more than throughput**. Even at 1 million transactions per day, the TPS is modest (~10). The design challenge is handling failures, ensuring consistency, and preventing double charges.

---

<a id="q3"></a>
### Q3: What is the difference between pay-in and pay-out flows?

**Answer:**

These two flows reflect how money moves through the platform:

```
PAY-IN FLOW (Customer → Platform)
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────────┐
│  Buyer  │───▶│ Payment  │───▶│   PSP   │───▶│  Platform's  │
│         │    │ Service  │    │(Stripe) │    │ Bank Account │
└─────────┘    └──────────┘    └─────────┘    └──────────────┘
  Places order   Coordinates     Charges         Holds money
                 payment         credit card      in custody

PAY-OUT FLOW (Platform → Seller)
┌──────────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐
│  Platform's  │───▶│  Payout  │───▶│ Payout  │───▶│ Seller's │
│ Bank Account │    │ Service  │    │Provider │    │  Bank    │
└──────────────┘    └──────────┘    │(Tipalti)│    │ Account  │
  Releases funds     Coordinates    └─────────┘    └──────────┘
  after conditions   payout          Transfers
  are met                            money
```

**Pay-in:** When a buyer places an order, money flows from the buyer's credit card to the platform's bank account. The platform acts as a custodian -- it holds money on behalf of the seller.

**Pay-out:** When conditions are met (e.g., product delivered), money flows from the platform's bank account to the seller's bank account. This is typically handled by third-party payout providers like Tipalti due to complex bookkeeping and regulatory requirements.

**Key differences:**

| Aspect | Pay-in | Pay-out |
|--------|--------|---------|
| **Direction** | Customer → Platform | Platform → Seller |
| **Trigger** | Customer places order | Delivery confirmed / schedule |
| **Timing** | Immediate | Delayed (days to weeks) |
| **Provider** | PSP (Stripe, Braintree) | Payout provider (Tipalti) |
| **Data needed** | Buyer's card info | Seller's bank account info |
| **Frequency** | Per transaction | Batched (daily/weekly/monthly) |

---

<a id="q4"></a>
### Q4: What back-of-the-envelope estimations would you make?

**Answer:**

Given a requirement of 1 million transactions per day:

```
Transactions:
  1,000,000 txn/day
  ÷ 100,000 seconds/day (approximate)
  = 10 TPS (transactions per second)

Peak traffic (assume 5x average):
  10 × 5 = 50 TPS

Storage (per transaction ~1 KB):
  1,000,000 × 1 KB = 1 GB/day
  365 GB/year
  With 7-year regulatory retention: ~2.5 TB
```

**Key takeaway:** 10 TPS is not a large number for a typical database. This means the system design should **not** focus on achieving high throughput. Instead, the focus should be on:

| Priority | Reason |
|----------|--------|
| **Correctness** | Money must not be lost, duplicated, or miscounted |
| **Fault tolerance** | Every failure scenario must be handled gracefully |
| **Consistency** | All services must agree on the state of every transaction |
| **Security** | PCI DSS compliance, fraud prevention |

This is fundamentally different from systems like a chat application or news feed where throughput and latency are the primary concerns.

---

<a id="q5"></a>
### Q5: What are card schemes and how does the payment ecosystem work?

**Answer:**

The payment ecosystem involves multiple parties working together to process a credit card transaction:

```
┌──────────┐  1.Pay  ┌──────────┐ 2.Auth  ┌──────────┐
│  Card    │────────▶│ Merchant │────────▶│ Acquirer │
│  Holder  │         │ (Amazon) │         │  (Bank)  │
│  (Buyer) │         └──────────┘         └────┬─────┘
└──────────┘                                   │
                                          3. Route
                                               │
                                               ▼
                                         ┌──────────┐
                                         │   Card   │
                                         │  Scheme  │
                                         │(Visa/MC) │
                                         └────┬─────┘
                                               │
                                          4. Verify
                                               │
                                               ▼
                                         ┌──────────┐
                                         │  Issuer  │
                                         │  (Bank)  │
                                         │ Buyer's  │
                                         │  bank    │
                                         └──────────┘
```

**Participants:**

| Party | Role |
|-------|------|
| **Cardholder** | The buyer who owns the credit card |
| **Merchant** | The business accepting the payment (e.g., Amazon) |
| **Acquirer** | The merchant's bank that processes card payments on their behalf |
| **Card Scheme** | Network that routes transactions (Visa, MasterCard, Amex, Discover) |
| **Issuer** | The cardholder's bank that issued the credit card |
| **PSP** | Simplifies integration -- merchants connect to a PSP instead of directly to acquirers/card schemes |

**Transaction flow:**
1. Cardholder initiates payment at the merchant
2. Merchant (via PSP) sends authorization request to the acquirer
3. Acquirer routes through the card scheme network
4. Card scheme forwards to the issuer for verification
5. Issuer checks funds/credit limit, returns approval/decline
6. Response flows back through the same chain

**Why most companies use a PSP:** Direct connections to card schemes are highly specialized, expensive to maintain, and require extensive compliance. PSPs like Stripe abstract this complexity away, letting merchants integrate via simple APIs.

---

## Section 2: High-Level Architecture

<a id="q6"></a>
### Q6: How would you design the high-level architecture for the pay-in flow?

**Answer:**

```
┌─────────────────────────────────────────────────────────────┐
│                       PAY-IN FLOW                           │
│                                                             │
│  ┌──────┐  1   ┌─────────┐  3   ┌──────────┐  5  ┌─────┐  │
│  │ User │─────▶│ Payment │─────▶│ Payment  │────▶│ PSP │  │
│  │      │      │ Service │      │ Executor │     │     │  │
│  └──────┘      └────┬────┘      └────┬─────┘     └─────┘  │
│                  2│ save          4│ save                   │
│                   ▼                ▼                        │
│              ┌─────────┐    ┌──────────┐                   │
│              │  Event  │    │  Order   │                   │
│              │   DB    │    │   DB     │                   │
│              └─────────┘    └──────────┘                   │
│                                                             │
│           6   ┌──────────┐  8   ┌──────────┐               │
│          ────▶│  Wallet  │─────▶│  Ledger  │               │
│               │ Service  │      │ Service  │               │
│               └────┬─────┘      └────┬─────┘               │
│                 7│ save           9│ append                 │
│                  ▼                  ▼                       │
│              ┌─────────┐    ┌──────────┐                   │
│              │ Wallet  │    │  Ledger  │                   │
│              │   DB    │    │   DB     │                   │
│              └─────────┘    └──────────┘                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Step-by-step flow:**

| Step | Action | Description |
|------|--------|-------------|
| 1 | User clicks "Place Order" | Payment event sent to payment service |
| 2 | Save payment event | Payment service persists the event to database |
| 3 | Send to executor | Payment service calls executor for each payment order (one event can contain multiple orders from different sellers) |
| 4 | Save payment order | Executor persists order with status NOT_STARTED |
| 5 | Call PSP | Executor sends payment request to external PSP to charge credit card |
| 6 | Update wallet | On success, payment service updates seller's wallet balance |
| 7 | Save wallet | Wallet service persists the updated balance |
| 8 | Update ledger | Payment service calls ledger to record the transaction |
| 9 | Append ledger entry | Ledger service appends an immutable record to the database |

**Why separate payment events and payment orders?** A single checkout may include items from multiple sellers. The payment event represents the entire checkout, while each payment order corresponds to one seller's portion. This allows the system to handle partial failures -- if one seller's payment fails, others can still succeed.

---

<a id="q7"></a>
### Q7: How does the pay-out flow differ from pay-in?

**Answer:**

The pay-out flow shares the same core components but reverses the money direction and uses different external providers:

```
PAY-IN:
  Buyer's Card ──PSP──▶ Platform Bank Account
  (Stripe charges the buyer)

PAY-OUT:
  Platform Bank Account ──Payout Provider──▶ Seller's Bank Account
  (Tipalti transfers to seller)
```

**Key architectural differences:**

| Aspect | Pay-in | Pay-out |
|--------|--------|---------|
| **External provider** | PSP (Stripe, Braintree) | Payout provider (Tipalti) |
| **Trigger** | User action (place order) | Condition met (delivery confirmed) or scheduled |
| **Processing** | Real-time, per transaction | Batch processing (daily/weekly/monthly) |
| **Information needed** | Buyer's card info (via PSP) | Seller's bank account, tax info |
| **Regulatory complexity** | PCI DSS for card handling | KYC, tax withholding, international banking regulations |
| **Volume** | Many small transactions | Fewer, aggregated payouts |

**Why use a third-party payout provider?** Payouts involve complex regulatory requirements: tax withholding (1099 forms in the US), KYC (Know Your Customer) verification, international wire transfers, currency conversion, and compliance with banking regulations across different countries. Providers like Tipalti specialize in handling this complexity.

---

<a id="q8"></a>
### Q8: How do you design the payment service API?

**Answer:**

Using RESTful API conventions:

**Execute a payment:**

```
POST /v1/payments
```

Request body:

```json
{
  "buyer_info": {
    "buyer_id": "uid-12345",
    "country": "US"
  },
  "checkout_id": "chk-abc-123",
  "credit_card_info": {
    "token": "tok_visa_4242"
  },
  "payment_orders": [
    {
      "payment_order_id": "po-001",
      "seller_account": "seller-789",
      "amount": "199.99",
      "currency": "USD"
    },
    {
      "payment_order_id": "po-002",
      "seller_account": "seller-456",
      "amount": "49.95",
      "currency": "USD"
    }
  ]
}
```

**Check payment status:**

```
GET /v1/payments/{payment_order_id}
```

**Critical design decisions:**

| Decision | Choice | Reason |
|----------|--------|--------|
| **Amount as string** | `"199.99"` not `199.99` | Avoids floating-point rounding errors during serialization. Different protocols and hardware handle numeric precision differently. Also supports extreme values (Japan's GDP in yen, Bitcoin satoshis). |
| **Globally unique order IDs** | UUID for `payment_order_id` | Used as the idempotency key (deduplication ID) when sending payment requests to the PSP |
| **Separate credit card token** | PSP-issued token, not raw card number | System never handles raw card data, reducing PCI DSS scope |

**Why amount must be a string:**
1. Serialization/deserialization across protocols can introduce rounding
2. Numbers can be extremely large (5×10^14 yen) or extremely small (10^-8 Bitcoin satoshi)
3. Parsed to numeric types only for display or calculation, never for storage or transmission

---

<a id="q9"></a>
### Q9: What data model would you use for the payment service?

**Answer:**

Two core tables are needed: **payment_event** and **payment_order**.

**Database choice -- Problem/Alternatives/Recommendation:**

| Option | Pros | Cons |
|--------|------|------|
| **Relational DB (PostgreSQL)** | ACID transactions, proven in finance, rich tooling, mature DBA market | Horizontal scaling is harder |
| **NoSQL (DynamoDB, MongoDB)** | Easy horizontal scaling | Eventual consistency is unacceptable for financial data |
| **NewSQL (CockroachDB, YugabyteDB)** | Distributed + ACID | Less mature, smaller DBA talent pool |

**Recommendation:** Traditional relational database (PostgreSQL/MySQL). Performance is not the bottleneck at 10 TPS. What matters is: proven stability in financial systems, rich monitoring/investigation tools, and availability of experienced DBAs.

**Payment Event table:**

```sql
CREATE TABLE payment_event (
    checkout_id     VARCHAR(255) PRIMARY KEY,
    buyer_id        VARCHAR(255) NOT NULL,
    seller_id       VARCHAR(255) NOT NULL,
    credit_card_id  VARCHAR(255) NOT NULL,
    amount          VARCHAR(50)  NOT NULL,
    currency        CHAR(3)      NOT NULL,
    is_payment_done BOOLEAN      DEFAULT FALSE,
    created_at      TIMESTAMP    DEFAULT NOW(),
    updated_at      TIMESTAMP    DEFAULT NOW()
);
```

**Payment Order table:**

```sql
CREATE TABLE payment_order (
    payment_order_id     VARCHAR(255) PRIMARY KEY,
    checkout_id          VARCHAR(255) REFERENCES payment_event(checkout_id),
    buyer_account        VARCHAR(255) NOT NULL,
    seller_account       VARCHAR(255) NOT NULL,
    amount               VARCHAR(50)  NOT NULL,
    currency             CHAR(3)      NOT NULL,
    payment_order_status VARCHAR(20)  NOT NULL DEFAULT 'NOT_STARTED',
    wallet_updated       BOOLEAN      DEFAULT FALSE,
    ledger_updated       BOOLEAN      DEFAULT FALSE,
    created_at           TIMESTAMP    DEFAULT NOW(),
    updated_at           TIMESTAMP    DEFAULT NOW()
);
```

**Status transitions for `payment_order_status`:**

```
NOT_STARTED ──▶ EXECUTING ──▶ SUCCESS
                    │
                    └──▶ FAILED
```

1. **NOT_STARTED** → initial status when order is created
2. **EXECUTING** → payment service sends order to payment executor
3. **SUCCESS** → PSP confirms successful payment
4. **FAILED** → PSP returns a failure

**Post-success updates:**
- On SUCCESS → update wallet (set `wallet_updated = TRUE`)
- After wallet → update ledger (set `ledger_updated = TRUE`)
- When all orders under a `checkout_id` succeed → set `is_payment_done = TRUE`

A scheduled job monitors in-flight payment orders and alerts engineers when an order doesn't complete within a configurable threshold.

---

<a id="q10"></a>
### Q10: What is the double-entry ledger system?

**Answer:**

The double-entry principle is fundamental to any payment system. Every transaction is recorded as two separate ledger entries with the same amount: one account is **debited** and the other is **credited**.

**Core rule: the sum of all transaction entries must be zero.**

```
Example: Buyer pays $1 to Seller

┌─────────────┬────────────┬────────────┐
│   Account   │   Debit    │   Credit   │
├─────────────┼────────────┼────────────┤
│ Buyer       │    $1      │            │
│ Seller      │            │     $1     │
├─────────────┼────────────┼────────────┤
│ Total       │    $1      │     $1     │
│ Sum         │     $1 - $1 = $0        │
└─────────────┴────────────┴────────────┘
```

**Why double-entry?**

| Benefit | Explanation |
|---------|-------------|
| **Traceability** | Every cent can be traced end-to-end through the payment cycle |
| **Error detection** | If entries don't sum to zero, something is wrong |
| **Auditability** | Complete financial record for regulatory compliance |
| **Consistency** | One cent lost somewhere means someone else gained a cent |

**Implementation characteristics:**
- Ledger entries are **immutable** -- once written, they are never modified
- Corrections are made by adding new compensating entries, not editing old ones
- This append-only design provides a complete audit trail

**Real-world reference:** Square's engineering team built an immutable double-entry accounting database service called "Books" following these principles.

---

## Section 3: PSP Integration & Hosted Payments

<a id="q11"></a>
### Q11: What is a PSP and how do you integrate with one?

**Answer:**

A Payment Service Provider (PSP) is a third-party service that moves money between accounts. It abstracts the complexity of connecting directly to banks and card schemes.

**Two integration approaches -- Problem/Alternatives/Recommendation:**

| Approach | How It Works | PCI DSS Scope | When to Use |
|----------|-------------|---------------|-------------|
| **API Integration** | Company builds payment pages, collects card data, sends to PSP via API | High (SAQ D) -- company handles sensitive data | Large companies that can justify compliance investment |
| **Hosted Payment Page** | PSP provides a widget/iframe that collects card data directly | Low (SAQ A) -- company never touches card data | Most companies (recommended) |

**Recommendation:** Hosted payment page. The vast majority of companies should not store or handle credit card data directly. The compliance burden of PCI DSS is enormous: encryption requirements, regular security audits, network segmentation, access controls, and more.

**How the hosted approach works:**

```
┌──────────┐       ┌──────────┐       ┌──────────┐
│  Client  │       │ Payment  │       │   PSP    │
│ (Browser)│       │ Service  │       │ (Stripe) │
└────┬─────┘       └────┬─────┘       └────┬─────┘
     │  1. Checkout      │                  │
     │──────────────────▶│                  │
     │                   │  2. Register     │
     │                   │─────────────────▶│
     │                   │  3. Token        │
     │                   │◀─────────────────│
     │  4. Show hosted   │                  │
     │     payment page  │                  │
     │◀──────────────────│                  │
     │                                      │
     │  5. Card info entered directly       │
     │─────────────────────────────────────▶│
     │  6. Payment result                   │
     │◀─────────────────────────────────────│
     │                                      │
     │                   │  7. Webhook      │
     │                   │◀─────────────────│
     │                   │  (async status)  │
```

**Key point:** The buyer's card data flows directly from the browser to the PSP. It never passes through or is stored by the payment system. This dramatically reduces the security scope and compliance requirements.

---

<a id="q12"></a>
### Q12: How does the hosted payment page flow work?

**Answer:**

The complete 9-step flow for a hosted payment page integration:

| Step | Actor | Action |
|------|-------|--------|
| 1 | Client → Payment Service | User clicks "checkout," sends payment order info |
| 2 | Payment Service → PSP | Sends registration request with amount, currency, expiration, redirect URL, and a UUID (nonce) for exactly-once registration |
| 3 | PSP → Payment Service | Returns a **token** (PSP-side UUID that uniquely identifies this payment registration) |
| 4 | Payment Service → DB | Persists the token before proceeding |
| 5 | Client | Displays PSP's hosted payment page (iframe/widget for web, SDK for mobile). The page uses the token to fetch payment details from PSP |
| 6 | Client → PSP | User fills in card details and clicks pay. PSP processes the payment |
| 7 | PSP → Client | Returns payment status |
| 8 | Client | Browser redirects to the **redirect URL** (e.g., `https://shop.com/?tokenID=abc&payResult=success`) showing checkout status |
| 9 | PSP → Payment Service | **Asynchronously** calls the webhook URL with the payment status. Payment service updates `payment_order_status` in the database |

**Two critical URLs:**

| URL | Purpose | When Used |
|-----|---------|-----------|
| **Redirect URL** | Web page shown to the user after payment completes | Step 8 -- immediate user feedback |
| **Webhook URL** | Backend endpoint PSP calls with authoritative payment status | Step 9 -- async server-to-server notification |

**Why the token matters:**
- The token is stored before the hosted page is shown (step 4)
- It uniquely maps to the payment order via the nonce
- If the user pays again with the same token, the PSP recognizes it as a duplicate
- This provides idempotency on the PSP side

**Why the webhook matters:**
- The redirect URL can be manipulated by the client
- The webhook is a server-to-server call that provides the authoritative payment status
- The payment system should only trust the webhook for updating internal state

---

<a id="q13"></a>
### Q13: How do you handle payment processing delays?

**Answer:**

Not all payments complete in seconds. Some take hours or days:

| Scenario | Cause | Duration |
|----------|-------|----------|
| **Risk review** | PSP flags the payment as high-risk, requires human review | Hours |
| **3D Secure** | Card requires extra authentication (e.g., bank OTP) | Minutes |
| **Bank processing** | Issuer bank takes time to approve | Hours |
| **Manual verification** | Large or unusual transactions need additional checks | Days |

**How the system handles delays:**

```
User clicks "Pay"
       │
       ▼
┌─────────────┐     ┌──────────┐
│ PSP returns  │────▶│  Client  │
│   PENDING    │     │  shows   │
│   status     │     │ "Payment │
└──────┬──────┘     │ pending" │
       │            └──────────┘
       │
       │  (Hours/days later)
       ▼
┌─────────────┐     ┌──────────┐
│ PSP calls   │────▶│ Payment  │
│  webhook    │     │ Service  │
│ with final  │     │ updates  │
│  status     │     │   DB     │
└─────────────┘     └──────────┘
```

**Two notification patterns from PSPs:**

| Pattern | How It Works | Pros | Cons |
|---------|-------------|------|------|
| **Webhook (push)** | PSP calls your endpoint when status changes | Real-time updates, no wasted calls | Must handle webhook reliability |
| **Polling (pull)** | Your system periodically queries PSP for status | Simpler to implement | Wasteful, delayed detection |

Most modern PSPs support webhooks. The payment service should:
1. Return a **pending** status to the client immediately
2. Provide a status page where the customer can check payment progress
3. Listen for webhook notifications from the PSP
4. Update internal state and trigger downstream processes (shipping, email) when the final status arrives

---

## Section 4: Reliability & Exactly-Once Delivery

<a id="q14"></a>
### Q14: How do you handle failed payments?

**Answer:**

Failed payments are inevitable. The system must track state precisely so that at any failure point, it can decide whether to **retry** or **refund**.

**Payment state tracking:**

An append-only state history table ensures you always know the current state and how you got there:

```
payment_order_id | status      | timestamp           | details
─────────────────┼─────────────┼─────────────────────┼────────────────
po-001           | NOT_STARTED | 2025-01-15 10:00:00 | Order created
po-001           | EXECUTING   | 2025-01-15 10:00:01 | Sent to PSP
po-001           | FAILED      | 2025-01-15 10:00:03 | PSP timeout
po-001           | EXECUTING   | 2025-01-15 10:00:08 | Retry attempt 1
po-001           | SUCCESS     | 2025-01-15 10:00:09 | PSP confirmed
```

**Failure classification:**

| Type | Examples | Action |
|------|----------|--------|
| **Retryable** | Network timeout, PSP temporary error (5xx), rate limiting (429) | Route to retry queue |
| **Non-retryable** | Invalid card, insufficient funds, fraud detected | Store error, notify user |
| **Unknown** | Ambiguous PSP response, connection dropped mid-transaction | Check state via PSP API, then decide |

**Decision framework:**

```
Payment fails
     │
     ├── Is it retryable?
     │      ├── Yes → Send to retry queue
     │      └── No  → Log error, notify user
     │
     ├── Was money already charged?
     │      ├── Yes, but our DB doesn't reflect it
     │      │     → Reconciliation will catch this
     │      └── No  → Safe to retry
     │
     └── Has retry count exceeded threshold?
            ├── No  → Retry with backoff
            └── Yes → Send to dead letter queue
                      for manual investigation
```

---

<a id="q15"></a>
### Q15: What retry strategies are appropriate for payment systems?

**Answer:**

| Strategy | How It Works | When to Use |
|----------|-------------|-------------|
| **Immediate retry** | Resend request instantly | Very transient errors (rare in payments) |
| **Fixed intervals** | Wait a fixed time (e.g., 5s) between retries | Simple, predictable failures |
| **Incremental intervals** | Wait increases linearly (5s, 10s, 15s) | Moderate congestion |
| **Exponential backoff** | Wait doubles each time (1s, 2s, 4s, 8s, 16s) | Network issues unlikely to resolve quickly |
| **Cancel** | Stop retrying | Permanent failures |

**Recommendation: Exponential backoff** is the general best practice for payment systems because:
1. Network issues and PSP overload are often not resolved instantly
2. Aggressive retries waste resources and can worsen PSP overload
3. Provides natural back-pressure

**Best practice:** Include a `Retry-After` header in error responses so clients know when to retry.

**The double-payment problem with retries:**

Retrying creates a risk of charging the customer twice:

```
Scenario: Network error hides a successful payment

Client ──── POST /pay ────▶ PSP ──── charges card ✓
       ◀── (network drops) ──── response lost

Client ──── POST /pay ────▶ PSP ──── charges card again?
       ◀── 200 OK ─────────

Result: Customer charged twice!
```

This is why retries alone are not sufficient -- you need **idempotency** (covered in Q18) to guarantee at-most-once execution.

---

<a id="q16"></a>
### Q16: What are retry queues and dead letter queues in a payment context?

**Answer:**

These queues provide structured failure handling:

```
┌──────────────┐
│   Payment    │
│   Failed     │
└──────┬───────┘
       │
       ▼
┌──────────────┐     Yes    ┌──────────────┐
│  Retryable?  │───────────▶│ Retry Queue  │
└──────┬───────┘            └──────┬───────┘
       │ No                        │
       ▼                           ▼
┌──────────────┐            ┌──────────────┐
│  Store Error │            │   Process    │
│  in DB       │            │   Retry      │
└──────────────┘            └──────┬───────┘
                                   │
                              ┌────┴────┐
                              │ Success? │
                              └────┬────┘
                             No    │    Yes
                      ┌────────────┤
                      ▼            ▼
               ┌─────────────┐  ┌─────────┐
               │ Under retry │  │  Done   │
               │ threshold?  │  └─────────┘
               └──────┬──────┘
                Yes   │   No
                ┌─────┤
                ▼     ▼
          ┌─────────┐ ┌──────────────┐
          │  Back   │ │ Dead Letter  │
          │  to     │ │    Queue     │
          │ Retry Q │ │ (Manual      │
          └─────────┘ │ investigation│
                      └──────────────┘
```

**Retry Queue:**
- Contains messages for retryable failures (transient errors, timeouts, 5xx responses)
- Workers consume from this queue and re-attempt the payment
- Each message tracks its retry count
- Typically uses exponential backoff between retries

**Dead Letter Queue (DLQ):**
- Messages that fail repeatedly beyond the retry threshold land here
- Serves as an isolation zone for problematic transactions
- Engineers investigate these manually to determine root cause
- Prevents poison messages from blocking the retry queue

**Real-world example:** Uber's payment system uses Kafka for both retry and dead letter queues to achieve reliability and fault tolerance at scale. Kafka's persistence and ordering guarantees make it well-suited for financial message processing.

---

<a id="q17"></a>
### Q17: How do you achieve exactly-once delivery in payments?

**Answer:**

Double-charging a customer is one of the most serious problems a payment system can have. Exactly-once delivery is the solution.

**The key insight:** Break exactly-once into two simpler guarantees:

```
Exactly-once = At-least-once + At-most-once
```

| Guarantee | Mechanism | What It Solves |
|-----------|-----------|---------------|
| **At-least-once** | Retry on failure | Ensures the payment eventually goes through despite transient failures |
| **At-most-once** | Idempotency check | Ensures a payment is never processed more than once, even if the request is sent multiple times |

**How they work together:**

```
Request 1: POST /pay (order-123)
  → PSP processes payment ✓
  → Response lost due to network error ✗

Request 2: POST /pay (order-123)  ← RETRY (at-least-once)
  → PSP sees order-123 already processed
  → Returns cached result ← IDEMPOTENCY (at-most-once)

Result: Payment processed exactly once ✓
```

Without at-least-once (no retry): Payment might silently fail, and the customer thinks they paid but the seller never receives money.

Without at-most-once (no idempotency): Retrying could charge the customer multiple times.

Both mechanisms must be implemented together to achieve exactly-once delivery.

---

<a id="q18"></a>
### Q18: What is idempotency and how do you implement it?

**Answer:**

An operation is **idempotent** if performing it multiple times produces the same result as performing it once. For payment APIs, this means: sending the same payment request N times results in exactly one charge.

**Implementation: idempotency key in HTTP header**

```
POST /v1/payments
Headers:
  Idempotency-Key: order-abc-123
```

The idempotency key is typically:
- A UUID generated by the client
- For e-commerce: the shopping cart ID at checkout time
- Expires after a configured period

**Database-backed idempotency:**

```sql
CREATE TABLE payment_order (
    payment_order_id VARCHAR(255) PRIMARY KEY,
    ...
);
```

The `payment_order_id` serves as the idempotency key via the primary key unique constraint:

1. Payment request arrives → try to INSERT into payment_order
2. **Insert succeeds** → first time seeing this request, process the payment
3. **Insert fails (duplicate key)** → already processed, return the existing result

**Scenario 1: User double-clicks the "Pay" button**

```
Click 1: POST /pay (idempotency_key = cart-789)
  → Insert succeeds → process payment → return SUCCESS

Click 2: POST /pay (idempotency_key = cart-789)  (milliseconds later)
  → Insert fails (duplicate) → return existing SUCCESS result

Concurrent clicks: → return 429 Too Many Requests
```

**Scenario 2: Payment succeeds at PSP but response is lost**

```
Attempt 1: POST /pay → PSP charges card ✓ → response lost ✗
  Payment service registered with PSP using nonce (= payment_order_id)
  PSP returned token (maps uniquely to nonce)

Attempt 2: POST /pay → same payment_order_id → same token sent to PSP
  PSP sees same token → recognizes duplicate → returns previous result ✓

Result: Customer charged only once
```

**The nonce-token relationship is critical:**
- The **nonce** (payment_order_id) uniquely identifies the payment order on our side
- The **token** uniquely maps to the nonce on the PSP side
- Since the token is used as the PSP's idempotency key, duplicate requests are detected at the PSP level too

---

## Section 5: Consistency & Reconciliation

<a id="q19"></a>
### Q19: How do you maintain data consistency across payment services?

**Answer:**

A payment system has multiple stateful services that must stay in sync:

| Service | What It Stores |
|---------|---------------|
| **Payment service** | Nonce, token, payment orders, execution status |
| **Ledger** | All accounting data (double-entry records) |
| **Wallet** | Merchant account balances |
| **PSP** | Payment execution status (external) |
| **DB replicas** | Copies of data for reliability |

**Consistency strategies by scope:**

| Scope | Strategy | Details |
|-------|----------|---------|
| **Between internal services** | Exactly-once processing | Retry + idempotency (see Q17) |
| **Between internal and PSP** | Idempotency + reconciliation | Use same idempotency key for retries; verify with reconciliation even if PSP supports idempotent API |
| **Between DB replicas** | Consensus or single-writer | Two approaches below |

**Handling database replication lag -- Problem/Alternatives/Recommendation:**

| Option | How It Works | Pros | Cons |
|--------|-------------|------|------|
| **Primary-only reads/writes** | All traffic goes to the primary DB; replicas are for backup only | Simple, always consistent | Wastes replica resources, limits read scalability |
| **Consensus-based replication** | Use Paxos/Raft algorithms, or distributed databases like CockroachDB/YugabyteDB | Replicas serve traffic, strongly consistent | More complex, higher write latency |

**Recommendation:** For a payment system, correctness trumps performance. Start with primary-only reads/writes. Move to consensus-based replication if read scalability becomes a bottleneck. Never rely on eventual consistency for financial data.

---

<a id="q20"></a>
### Q20: What is reconciliation and how does it work?

**Answer:**

Reconciliation is the last line of defense in a payment system. It periodically compares states across services to ensure they agree.

**When it runs and what it compares:**

```
Every night:

┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│   PSP / Bank  │     │ Reconciliation│     │    Ledger     │
│               │     │    System     │     │    System     │
│  Settlement   │────▶│               │◀────│               │
│    File       │     │  Compare &    │     │  Internal     │
│ (daily txns   │     │  find         │     │  records      │
│  + balance)   │     │  mismatches   │     │               │
└───────────────┘     └───────┬───────┘     └───────────────┘
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              ┌──────────┐ ┌────────┐ ┌──────────────┐
              │Auto-fix  │ │Manual  │ │ Investigate  │
              │(Known    │ │fix     │ │ (Unknown     │
              │ cause)   │ │(Known  │ │  cause)      │
              └──────────┘ │ cause) │ └──────────────┘
                           └────────┘
```

**Settlement file:** Every night, the PSP or bank sends a file containing:
- The bank account balance
- All transactions that occurred during the day

**Three categories of mismatches:**

| Category | Description | Action |
|----------|-------------|--------|
| **Classifiable + automatable** | Known cause, cost-effective to code a fix | Automated adjustment program |
| **Classifiable + manual** | Known cause, but too complex or rare to automate | Finance team fixes via manual job queue |
| **Unclassifiable** | Unknown cause, unexpected discrepancy | Special queue for finance team investigation |

**Internal reconciliation** also verifies consistency between internal services. For example, the ledger and wallet might diverge due to a failed update. Reconciliation detects this discrepancy and triggers correction.

**Why reconciliation is necessary even with idempotent APIs:** You should never assume an external system is always correct. Reconciliation is the safety net that catches edge cases, bugs in PSP systems, timezone mismatches, and other issues that no amount of idempotency can prevent.

---

<a id="q21"></a>
### Q21: Synchronous vs asynchronous communication in payment systems?

**Answer:**

**Synchronous communication (e.g., HTTP):**

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────┐
│  Client  │───▶│ Payment  │───▶│ Executor │───▶│ PSP │
│          │◀───│ Service  │◀───│          │◀───│     │
└──────────┘    └──────────┘    └──────────┘    └─────┘
     Entire chain blocks until PSP responds
```

| Pros | Cons |
|------|------|
| Simple to implement and reason about | One slow service blocks the entire chain |
| Immediate response to client | Poor failure isolation |
| Easy debugging (linear flow) | Tight coupling between services |
| | Hard to scale for traffic spikes |

**Asynchronous communication (message queues):**

*Single receiver (each message processed by one consumer):*

```
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ Payment  │───▶│ Message Queue │───▶│ Executor │
│ Service  │    │ (RabbitMQ)   │    │          │
└──────────┘    └──────────────┘    └──────────┘
  Messages consumed and removed by one receiver
```

*Multiple receivers (each message processed by many consumers):*

```
                                    ┌──────────┐
                               ┌───▶│ Payment  │
                               │    │ Service  │
┌──────────┐    ┌─────────┐    │    └──────────┘
│ Payment  │───▶│  Kafka  │────┤    ┌──────────┐
│  Event   │    │         │    ├───▶│Analytics │
└──────────┘    └─────────┘    │    └──────────┘
                               │    ┌──────────┐
                               └───▶│ Billing  │
                                    └──────────┘
  Messages NOT removed; each service consumes independently
```

**Recommendation for payment systems:**

| Use Case | Communication | Why |
|----------|--------------|-----|
| **User-facing authorization** | Synchronous | User needs immediate feedback ("payment approved") |
| **Post-payment processing** | Asynchronous | Wallet updates, ledger entries, analytics, notifications can happen independently |
| **Settlement/batch** | Asynchronous | Daily batch processing, no user waiting |

For large-scale payment systems with complex business logic and many third-party dependencies, asynchronous communication is the better default. It trades design simplicity for scalability and failure resilience. Use Kafka for multiple-receiver scenarios (one payment event triggers notifications, analytics, billing, etc.).

---

## Section 6: Security & Advanced Topics

<a id="q22"></a>
### Q22: What security measures should a payment system implement?

**Answer:**

| Threat | Measure | Implementation |
|--------|---------|----------------|
| **DDoS attacks** | Rate limiting, traffic filtering | WAF (Web Application Firewall), CDN-level protection, API gateway rate limits |
| **Card theft / fraud** | Fraud detection | Rule-based and ML-based detection systems; Address Verification Service (AVS); CVV verification |
| **Data breaches** | Encryption | TLS for data in transit; AES-256 for data at rest; tokenize sensitive data |
| **Unauthorized access** | Authentication & authorization | OAuth 2.0, API keys, role-based access control (RBAC) |
| **Man-in-the-middle** | Certificate pinning | Enforce HTTPS, HSTS headers, certificate pinning in mobile apps |
| **Internal threats** | Audit logging | Immutable audit logs, principle of least privilege, segregation of duties |

**PCI DSS compliance -- Problem/Alternatives/Recommendation:**

| Approach | PCI Scope | Effort |
|----------|-----------|--------|
| **Handle card data yourself** | Full PCI DSS (SAQ D): 300+ security controls | Enormous -- annual audits, dedicated security team |
| **Use PSP hosted pages** | Minimal PCI DSS (SAQ A): ~20 controls | Manageable -- PSP handles sensitive data |
| **Hybrid (tokenization)** | Reduced scope (SAQ A-EP) | Moderate -- card data passes through but isn't stored |

**Recommendation:** Use PSP hosted payment pages. The compliance cost of handling raw card data directly almost never justifies the engineering effort for most companies.

**Additional security practices:**
- **3D Secure:** Extra cardholder verification (bank sends OTP to the card owner)
- **Velocity checks:** Flag accounts making an unusual number of transactions
- **Geolocation matching:** Compare transaction origin with cardholder's known location
- **Amount anomaly detection:** Flag unusually large transactions for review

---

<a id="q23"></a>
### Q23: How do you design monitoring and alerting for payments?

**Answer:**

Payment monitoring must cover both business metrics and system health:

**Business metrics:**

| Metric | What It Measures | Alert Threshold |
|--------|-----------------|-----------------|
| **Authorization rate** | % of payments approved by PSP | Drop below baseline (e.g., < 90%) |
| **Decline rate by reason** | Why payments fail (insufficient funds, fraud, expired card) | Spike in any category |
| **Revenue throughput** | Total payment volume per hour | Significant drop vs same period last week |
| **Refund rate** | % of transactions refunded | Spike above normal |
| **Chargeback rate** | Disputed transactions | Above 1% (card scheme penalty threshold) |

**System health metrics:**

| Metric | What It Measures | Alert Threshold |
|--------|-----------------|-----------------|
| **PSP latency (p50/p95/p99)** | How fast the PSP responds | p99 > 500ms |
| **Error rate** | % of failed API calls | > 1% |
| **Queue depth** | Messages waiting in retry/DLQ | Growing beyond baseline |
| **Reconciliation mismatches** | Discrepancies found nightly | Any unclassified mismatch |
| **In-flight payment age** | How long payments stay in EXECUTING state | > configurable threshold |

**Debugging tools:**
- Transaction lookup by ID showing full state history and every service involved
- PSP communication logs (request/response pairs)
- End-to-end trace of a payment through all services (distributed tracing)
- Customer support portal to review payment status without engineering involvement

---

<a id="q24"></a>
### Q24: How do you handle currency exchange in a global payment system?

**Answer:**

Supporting multiple currencies adds significant complexity:

**Key considerations:**

| Concern | Details |
|---------|---------|
| **Currency codes** | Use ISO 4217 standard (USD, EUR, JPY, etc.) |
| **Decimal precision** | Different currencies have different decimals: USD has 2 (cents), JPY has 0, BHD has 3 |
| **Exchange rates** | Rates fluctuate constantly; must decide when to lock the rate |
| **Rounding** | Different rounding rules by currency and jurisdiction |
| **Amount representation** | Store as smallest unit (cents for USD, yen for JPY) to avoid decimals |

**Exchange rate strategies -- Problem/Alternatives/Recommendation:**

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Lock at checkout** | Fix the rate when user clicks "pay" | User sees exact amount | Platform bears exchange rate risk |
| **Lock at settlement** | Use rate at daily settlement time | Platform has less risk | User might pay different from displayed amount |
| **Hedging** | Lock rate via financial instruments | Minimizes risk | Complex, costly |

**Amount storage best practice:**

```json
{
  "amount": "19999",
  "currency": "USD",
  "exponent": 2
}
```

This means $199.99. Storing the amount as the smallest integer unit avoids all floating-point issues. The `exponent` field (or a lookup table by currency) determines how to display the amount.

**Architecture impact:**
- Each payment order must store the original currency and the converted currency
- Exchange rate service (or third-party API) provides rates
- Ledger must record both the original and converted amounts
- Reconciliation must account for exchange rate differences

---

<a id="q25"></a>
### Q25: How would you handle alternative payment methods?

**Answer:**

Different regions have vastly different payment preferences:

| Region | Popular Methods |
|--------|----------------|
| **US/Europe** | Credit/debit cards, Apple Pay, Google Pay |
| **China** | Alipay, WeChat Pay |
| **India** | UPI, cash on delivery, Paytm |
| **Brazil** | PIX (instant bank transfer), Boleto (cash voucher) |
| **Southeast Asia** | GrabPay, GoPay, bank transfer |
| **Africa** | M-Pesa (mobile money) |

**Cash payment challenges:**

Cash-on-delivery (COD) is common in India and Brazil. This introduces unique challenges:

| Challenge | Solution |
|-----------|----------|
| **No upfront payment confirmation** | Reserve order, confirm only after cash collected |
| **Delivery driver collects cash** | Driver app records collection; reconcile with warehouse |
| **Higher refund/cancellation rates** | Implement stricter order validation, deposits |
| **Reconciliation complexity** | Daily reconciliation between driver collections and bank deposits |

**Architecture for multiple payment methods:**

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ▼
┌──────────────────────────┐
│    Payment Service        │
│  (Routing + Orchestration)│
└────┬─────────────────────┘
     │
     ├── Credit Card ──▶ Stripe / Braintree
     │
     ├── Apple Pay ────▶ Apple Pay SDK → same PSP
     │
     ├── Alipay ───────▶ Alipay API
     │
     ├── UPI ──────────▶ UPI payment gateway
     │
     ├── Bank Transfer ▶ Direct bank API
     │
     └── Cash (COD) ───▶ Driver collection service
```

**Design principles for supporting multiple methods:**
1. **Payment method abstraction:** Define a common interface that all payment methods implement (authorize, capture, refund)
2. **Strategy pattern:** Route to the correct payment provider based on method type and region
3. **Unified status model:** All methods map to the same internal status states (NOT_STARTED, EXECUTING, SUCCESS, FAILED) regardless of provider
4. **Provider-specific adapters:** Encapsulate each provider's API quirks behind a standard adapter

---

[← Back to System Design Index](README.md) | [← Back to Main Index](../README.md)
