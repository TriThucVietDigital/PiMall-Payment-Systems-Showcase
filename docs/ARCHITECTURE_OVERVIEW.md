# PiMall Financial System Architecture Overview

## Executive Summary

PiMall is architected as a **Digital Central Bank ecosystem** designed to manage multi-currency financial transactions at scale with institutional-grade security, stability, and performance. The system processes transactions across three asset classes (Pi GCV, Pi Sàn, GEM) while maintaining real-time settlement, comprehensive audit trails, and zero data leakage through multi-layer isolation strategies.

---

## Core Philosophy: "Digital Central Bank"

### Principles

1. **Long-Term Stability Over Short-Term Features**
   - Immutable transaction ledgers (write-once ledger entries)
   - Cryptographically-signed audit trails
   - 30-day gem fee grace periods with recovery mechanisms
   - No data deletion; only logical soft-deletes for compliance

2. **Large-Scale Scalability Without Performance Degradation**
   - Designed for 10+ million concurrent users
   - Horizontal scaling via read replicas and connection pooling
   - Sub-30ms response time target globally
   - Stateless API layer enabling load balancing

3. **Trust & Transparency**
   - Public ledger summaries (GCV rate locked at 1π = $314,159 USD)
   - Admin-auditable transaction history
   - Automated settlement confirmations
   - Gem receivable reports with 30-day aging

---

## System Architecture Layers

### Layer 1: Edge Security (Cloudflare)

**Purpose:** Protect origin infrastructure from DDoS, bot attacks, and compliance filtering.

**Components:**
- **Cloudflare WAF (Web Application Firewall)**
  - Pattern-based threat detection
  - Rate limiting: 100 req/s per IP for checkout endpoints
  - Geo-blocking: Restrict high-risk regions per jurisdiction
  
- **Cloudflare Workers** (optional serverless compute)
  - Cache warming for public ledger summaries
  - Request preprocessing (remove sensitive headers)
  - Request logging without storing PII

**Response Time Contribution:** <5ms

---

### Layer 2: API Gateway (Node.js + Express)

**Purpose:** Stateless request routing, authentication, and business logic orchestration.

**Tech Stack:**
- **Node.js 18+** — Non-blocking I/O, 50K+ concurrent connections per process
- **Express.js** — Minimal overhead (<2ms per request)
- **Authentication:** JWT (access token) + HTTP-only cookies (refresh token)
- **Rate Limiting:** Redis-backed sliding window (50 req/min per user for sensitive endpoints)

**Key Endpoints:**
- `POST /api/checkout` — Order creation + 1% Gem fee deduction
- `POST /api/orders/:id/pay` — Pi Network payment trigger
- `POST /api/wallet/withdraw` — GEM withdrawal (blocked if balance < 0)
- `GET /api/ledger/entries` — Transaction history (RLS-filtered)

**Response Time Contribution:** 5-10ms

---

### Layer 3: Business Logic (NestJS Microservices)

**Purpose:** Encapsulate domain logic with type safety and dependency injection.

**Services:**

#### Commerce Service
```typescript
- calculateGemFee(totalPiAmount) → returns 1% Gem fee
- createOrder(product, buyer, merchant) → validates inventory + payment method
- blockWithdrawal(userId) → enforces negative Gem balance freeze
```

#### Wallet Service
```typescript
- deductGemFee(userId, feeAmount) → allows negative balance
- recordGemFeeLedger(userId, feeAmount) → creates audit entry
- checkWithdrawalEligibility(userId) → rejects if Gem < 0
- settleGemReceivable(userId, paymentAmount) → applies credit on next fee
```

#### Ledger Service
```typescript
- recordTransaction(type, user, amount) → immutable write
- generateAuditReport(dateRange) → compliance reporting
- calculateUserBalance(userId) → sum all ledger entries
```

**Response Time Contribution:** 8-15ms

---

### Layer 4: Data Layer (PostgreSQL 15+)

**Purpose:** Durable, transactional storage with row-level security.

**Architecture:**
- **Primary:** Single-master, write-all transactions
- **Read Replicas:** 2-3 replicas for SELECT queries (eventual consistency OK for reporting)
- **Connection Pool:** PgBouncer (100-200 concurrent connections)
- **Backup:** WAL-archived (Point-in-Time Recovery to any second)

**Key Tables:**

#### `wallets`
```sql
id UUID PRIMARY KEY
user_id UUID FOREIGN KEY (users.id)
balance NUMERIC(20,2)           -- GEM balance (can be negative)
locked_balance NUMERIC(20,2)    -- GEM locked for deposits
gem_fee_owed NUMERIC(20,2)      -- Amount owed to platform (1% fee)
gem_fee_max NUMERIC(20,2)       -- Max allowed: 100 GEM
gem_fee_deadline TIMESTAMPTZ    -- Repay by (NOW() + 30 days)
is_withdrawal_blocked BOOLEAN   -- Lock if balance < 0
created_at TIMESTAMPTZ
updated_at TIMESTAMPTZ

CHECK (balance >= -100)         -- Max negative: -100 GEM
CHECK (gem_fee_owed >= 0)
CHECK (gem_fee_max = 100)
```

#### `orders`
```sql
id UUID PRIMARY KEY
buyer_id UUID FOREIGN KEY (users.id)
seller_id UUID FOREIGN KEY (users.id)
product_id UUID FOREIGN KEY (products.id)
total_amount_pi NUMERIC(20,8)       -- Order value in Pi
platform_fee_gem NUMERIC(20,2)      -- 1% of order in GEM
platform_fee_paid BOOLEAN DEFAULT FALSE
fee_due_date TIMESTAMPTZ            -- When fee must be settled
payment_method VARCHAR (GEM|PI_EXCHANGE|PI_GCV)
status VARCHAR (pending|processing|completed|failed)
transaction_id UUID                 -- Link to Pi Network tx
created_at TIMESTAMPTZ
updated_at TIMESTAMPTZ
```

#### `ledger_entries`
```sql
id UUID PRIMARY KEY
user_id UUID FOREIGN KEY (users.id)
transaction_type VARCHAR (GEM_PLATFORM_FEE|DEPOSIT|WITHDRAWAL|TRANSFER)
amount NUMERIC(20,8)
balance_after NUMERIC(20,8)         -- Running balance (audit trail)
reference_id UUID                    -- Link to orders or wallets
created_at TIMESTAMPTZ              -- Immutable timestamp
-- No update_at: write-once
```

#### `gem_fees` (Receivable Ledger)
```sql
id UUID PRIMARY KEY
user_id UUID FOREIGN KEY (users.id)
order_id UUID FOREIGN KEY (orders.id)
amount_owed NUMERIC(20,2)
due_date TIMESTAMPTZ
paid_date TIMESTAMPTZ               -- NULL until settled
status VARCHAR (pending|overdue|settled)
settlement_payment_id UUID          -- Reference to settlement fee payment
created_at TIMESTAMPTZ
```

**Response Time Contribution:** 15-20ms (avg query latency 3-5ms + network RTT)

---

### Layer 5: Row-Level Security (PostgreSQL RLS)

**Purpose:** Enforce data isolation at the database layer; users cannot access other users' records even with leaked credentials.

**Policies:**

#### `ledger_entries`
```sql
-- User can only view their own ledger entries
CREATE POLICY user_ledger_isolation ON ledger_entries
  FOR SELECT
  USING (auth.uid() = user_id)
```

#### `wallets`
```sql
-- User can view own wallet; admins can view all
CREATE POLICY wallet_isolation ON wallets
  FOR SELECT
  USING (
    auth.uid() = user_id
    OR auth.role() = 'admin'
  )

-- User cannot manually update balance (only backend procedures can)
CREATE POLICY wallet_no_direct_update ON wallets
  FOR UPDATE
  USING (FALSE)  -- All direct updates blocked
```

#### `orders`
```sql
-- Buyers see their purchases; sellers see their sales; admins see all
CREATE POLICY order_visibility ON orders
  FOR SELECT
  USING (
    auth.uid() IN (buyer_id, seller_id)
    OR auth.role() = 'admin'
  )
```

**Security Guarantee:** Even if API token is compromised, attacker cannot query other users' financial records.

---

## Security Stack

### 1. Authentication & Authorization

| Layer | Mechanism | TTL |
|-------|-----------|-----|
| **Client** | HTTP-only cookie (refresh token) | 30 days |
| **API Gateway** | JWT (access token) | 15 min |
| **Database** | RLS policy via `auth.uid()` | Per connection |

### 2. Data Encryption

| Data | Encryption | Key Management |
|------|-----------|-----------------|
| **In Transit** | TLS 1.3 (Cloudflare → Origin) | Automatic cert renewal |
| **At Rest** | PostgreSQL pgcrypto for PII | Separated in AWS KMS |
| **Wallet Seeds** | Never stored (calculated per session) | Hardware security module (optional) |

### 3. Audit & Compliance

| Audit Point | Mechanism |
|------------|-----------|
| **Transaction Log** | Immutable `ledger_entries` (no DELETE) |
| **Fee Deductions** | `gem_fees` table with timestamps |
| **Admin Actions** | Separate `admin_audit_log` |
| **Data Access** | PostgreSQL statement logging |
| **Reconciliation** | Daily user_gem_receivables report |

---

## Performance Optimization

### Global Response Time Target: 20-30ms

| Component | Latency | Optimization |
|-----------|---------|--------------|
| **Cloudflare Edge** | <5ms | PoP nearest to user |
| **TLS Handshake** | <10ms | Session resumption |
| **API Gateway** | 5-10ms | Stateless, horizontal scaling |
| **Business Logic** | 8-15ms | Cached configurations |
| **Database Query** | 3-5ms | Indexed wallets, ledger_entries |
| **Network RTT** | 2-3ms | Global CDN + local read replicas |

**Total:** 26-48ms (p95: 35ms, p99: 50ms)

### Optimization Strategies

1. **Connection Pooling**
   - PgBouncer: 200 concurrent connections
   - Reuse connections across requests

2. **Query Optimization**
   - Index on `(user_id, created_at)` for ledger retrieval
   - Materialized view `user_gem_receivables` refreshed hourly
   - No N+1 queries via batch loading

3. **Caching Layer** (Optional: Redis)
   - User balance cache (invalidate on transaction)
   - Product inventory cache (TTL 5min)
   - Ledger summary cache (TTL 1min)

4. **Read Replica Strategy**
   - All SELECT queries → read replicas
   - All INSERT/UPDATE/DELETE → primary
   - Replication lag: <100ms

---

## Scalability Model

### Horizontal Scaling

| Component | Method | Max Scale |
|-----------|--------|-----------|
| **API Servers** | Load balancer (AWS ALB) | 100+ instances |
| **Read Replicas** | PostgreSQL streaming replication | 10+ replicas |
| **Cache** | Redis cluster | 16+ shards |
| **Blob Storage** | S3/Vercel Blob | Unlimited |

### Vertical Scaling

| Database | Current | Growth Path |
|----------|---------|-------------|
| **Storage** | 100GB | 10TB (5-year projection) |
| **Connections** | 200 | 2,000 (partition if needed) |
| **QPS** | 1,000 | 100,000+ (shard by user_id) |

---

## Deployment Architecture

### Production Environment

```
┌─────────────────────────────────────────────┐
│          Cloudflare Global Network           │  <- DDoS protection, WAF
└────────────┬────────────────────────────────┘
             │ TLS 1.3
┌────────────▼────────────────────────────────┐
│       AWS ALB (Load Balancer)                │  <- Health checks, SSL termination
└────────────┬────────────────────────────────┘
             │
     ┌───────┼───────┐
     ▼       ▼       ▼
  [API-1] [API-2] [API-3]                       <- Node.js + NestJS instances
     │       │       │
     └───────┼───────┘
             │
     ┌───────▼────────┐
     │ PgBouncer      │                         <- Connection pooling
     │ (100-200 conn) │
     └───────┬────────┘
             │
    ┌────────┼──────────┐
    ▼        ▼          ▼
 [Primary] [Replica-1] [Replica-2]             <- PostgreSQL instances
    │        │          │
    │        └──────────┴──────────────┐
    │                                   │
    ▼ (WAL streaming replication)       ▼
[S3 WAL Archive]              [Point-in-Time Recovery]
```

---

## Data Flow: Order with 1% Gem Fee

```
1. User clicks "Buy Now"
   └─> Checkout calculates 1% Gem fee from total_amount_pi
       • Fee = (total_pi × 314,159 USD × 25,800 VND/USD × 0.01)
       • Example: 1π order → Fee = 81,025.25 GEM

2. Backend validates:
   └─> Check gem_fee_owed + current fee ≤ 100 GEM (max allowed)
   └─> Check gem_fee_deadline (≤ 30 days since last fee)
   └─> IF gem_balance ≥ fee → deduct immediately
   └─> ELSE allow negative balance, set is_withdrawal_blocked = true

3. Ledger records:
   ├─> INSERT ledger_entries (GEM_PLATFORM_FEE)
   ├─> INSERT gem_fees (status='pending', due_date=NOW()+30d)
   └─> UPDATE wallets SET balance = balance - fee

4. User sees:
   ├─> "Order placed" ✓
   ├─> Gem balance: -12,500 (shown in red)
   └─> Notification: "Settle fee by [DATE]"

5. Next checkout:
   ├─> Recalculate new fee
   ├─> IF balance < 0 → apply settlement: new_fee - previous_owed
   │   • Example: -12,500 GEM owed, next fee 500 GEM → net 12,000 deduction
   └─> UPDATE gem_fees SET paid_date, status='settled'
```

---

## Monitoring & Observability

### Metrics

| Metric | Alert Threshold | Action |
|--------|-----------------|--------|
| **API Response Time p95** | >100ms | Scale API horizontally |
| **Database Connection Pool** | >80% | Investigate slow queries |
| **Gem Receivable Overdue** | >5% of users | Send payment reminders |
| **Transaction Failure Rate** | >1% | Investigate payment processor |
| **Ledger Reconciliation** | Mismatch | Page on-call engineer |

### Observability Stack

- **Logging:** Structured JSON logs to CloudWatch
- **Tracing:** Distributed traces (OpenTelemetry) for request flows
- **Metrics:** Prometheus + Grafana dashboards
- **Alerting:** PagerDuty for critical incidents

---

## Compliance & Governance

### Financial Regulations

1. **KYC/AML (if applicable by jurisdiction)**
   - User identity verification before withdrawal
   - Transaction monitoring for suspicious patterns

2. **Data Protection (GDPR/CCPA)**
   - User right to data export (ledger + personal data)
   - Right to deletion (soft-delete, retain audit trail)

3. **Accounting Standards**
   - Double-entry accounting for Gem fees
   - Monthly reconciliation reports
   - Annual audit trail export

### Disaster Recovery

| Scenario | RTO | RPO | Procedure |
|----------|-----|-----|-----------|
| **Single Server Failure** | <5min | 0 | Automatic failover to replica |
| **Region Failure** | <1hr | <5min | Restore from WAL archive to standby region |
| **Data Corruption** | <24hr | <1hr | Point-in-Time Recovery to before corruption |
| **Ransomware** | <48hr | <6hr | Restore from immutable WAL backup |

---

## Roadmap

### Phase 1 (Current): Core Financial System
- ✅ Multi-currency wallet system
- ✅ 1% Gem transaction fees with 30-day grace period
- ✅ Negative Gem balance with withdrawal blocking
- ✅ Comprehensive ledger & audit trails

### Phase 2 (Q3 2026): Advanced Features
- Settlement automation (auto-repay fees)
- Recurring billing (subscription support)
- API webhooks for merchant integrations
- Mobile app (iOS/Android native wallets)

### Phase 3 (Q4 2026): Institutional Features
- Role-based access control (RBAC) for merchants
- Batch payment processing
- Real-time data analytics dashboard
- Regulatory reporting (FinCEN, FATF)

---

## Conclusion

PiMall's architecture embodies the principles of a **Digital Central Bank**: immutable transaction records, multi-layer security, transparent governance, and global-scale performance. By leveraging PostgreSQL's row-level security, Cloudflare's edge protection, and Node.js's scalability, the system provides institutional-grade financial infrastructure for the Pi Network ecosystem.

**Key Guarantees:**
- No unauthorized data access (RLS isolation)
- No transaction loss (immutable ledger)
- Sub-30ms response time (edge + pooling optimization)
- 30-day audit trail retention (compliance)
- Horizontal scalability to 10M+ users

---

**Document Version:** 1.0  
**Last Updated:** March 2026  
**Author:** PiMall Architecture Team  
**Classification:** Public
