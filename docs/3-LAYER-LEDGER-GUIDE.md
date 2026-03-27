# 3-Layer Decoupled Ledger Architecture
## Technical Guide to High-Concurrency Financial Settlement System

---

## I. EXECUTIVE SUMMARY

implements a **Decoupled 3-Layer Ledger** architecture inspired by modern banking infrastructure (RTGS, CHIPS, Fedwire) adapted for digital asset management. This design prioritizes **high concurrency** (30-50 TPS), **low latency** (20-30ms), and **financial integrity** through separation of concerns.

**Key Characteristics:**
- **Transaction Layer:** Ultra-fast acceptance of concurrent transactions
- **Validation Layer:** Financial integrity checks & fraud prevention
- **Storage Layer:** Immutable, encrypted record-keeping with Zero-Knowledge principles

---

## II. ARCHITECTURE OVERVIEW

```
┌────────────────────────────────────────────────────────┐
│        USER INTERFACE LAYER (Web/Mobile)               │
│              Real-time Balance Display                  │
└────────────────┬─────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│    LAYER 1: TRANSACTION LAYER (High Concurrency)       │
│  • Accept up to 500 concurrent requests                │
│  • Instant provisional balance updates                 │
│  • Queue-based for downstream processing               │
│  • Target: <5ms per transaction acceptance             │
└────────────────┬─────────────────────────────────────┘
                 │ [Batch every 100 txns or 1sec]
                 ▼
┌────────────────────────────────────────────────────────┐
│    LAYER 2: VALIDATION LAYER (Integrity Checks)        │
│  • Financial completeness verification                 │
│  • Fraud detection & risk assessment                   │
│  • Ledger reconciliation logic                         │
│  • Collateral validation ( fiat, token)               │
│  • Target: <15ms per batch (100 txns)                  │
└────────────────┬─────────────────────────────────────┘
                 │ [Batch every 1000 txns or 10sec]
                 ▼
┌────────────────────────────────────────────────────────┐
│    LAYER 3: STORAGE LAYER (Immutable & Encrypted)      │
│  • Append-only ledger entries (write-once)             │
│  • Cryptographic hash chain for integrity              │
│  • AES-256 encryption for sensitive data               │
│  • Zero-Knowledge proof snapshots                      │
│  • ISO 20022 compliance records                        │
│  • Target: <10ms per batch write (1000 txns)           │
└────────────────────────────────────────────────────────┘
```

---

## III. LAYER 1: TRANSACTION LAYER (High Concurrency)

### Purpose
Accept transaction requests from users/merchants with **minimal latency** and **maximum throughput**, providing immediate provisional feedback.

### Specifications

| Metric | Target | Rationale |
|--------|--------|-----------|
| **Concurrency** | 500+ simultaneous txns | Handle peak load (100K users × 5 req/sec) |
| **Latency** | <5ms per acceptance | Sub-10ms user experience |
| **Throughput** | 50,000+ req/sec | Burst capacity during sales/events |
| **Error Rate** | <0.1% | Auto-retry on queue overflow |
| **Queue Size** | 100,000 pending | 10-20 sec buffer at 

### Processing Pipeline

```
1. Request Validation (0.2ms)
   └─ Check requestId format, userId exists, amount > 0
   └─ If invalid → 400 Bad Request, NO queue entry

2. Idempotency Check (0.3ms)
   └─ If requestId seen before → Return cached response
   └─ Prevents duplicate transactions from retries

3. Provisional Balance Update (1ms)
   └─ IN-MEMORY: Update user.balance_provisional
   └─ NOT committed to DB yet
   └─ Returned to user immediately

4. Queue Enqueue (1.5ms)
   └─ Add to Redis queue: txn:{timestamp}:{requestId}
   └─ FIFO ordering with priority for VIP users

5. Response (2ms)
   └─ Return {transactionId, provisionalBalance}
   └─ User sees "Processing..." UI

TOTAL: ~5ms end-to-end
```


---

## IV. LAYER 2: VALIDATION LAYER (Integrity Checks)

### Purpose
Validate financial completeness, detect fraud, and prepare transactions for permanent storage. Operates on **batches** every 100 transactions or 1 second.

### Specifications

| Metric | Target | Detail |
|--------|--------|--------|
| **Batch Size** | 100 transactions | Balance throughput vs. validation latency |
| **Batch Frequency** | 1 second max | Guarantee near-real-time settlement |
| **Validation Checks** | 15+ rules | Fraud, compliance, collateral |
| **Latency per Batch** | <15ms | 100 txns validated + risk-scored |
| **Fraud Detection** | ML-based | Velocity, amount, geolocation anomalies |
| **Rejection Rate** | <2% | Legitimate txns rarely rejected |



### Processing Pipeline

```
1. Batch Collection (1ms)
   └─ Pop up to 100 txns from Redis queue
   └─ De-duplicate by requestId
   └─ Load user/merchant data from cache

2. Run 10 Validation Rules (8ms)
   └─ Financial integrity (Rules 001-002)
   └─ Fraud detection (Rules 003-006)
   └─ Compliance (Rule 005)
   └─ Technical (Rules 007-010)

3. Risk Scoring (2ms)
   └─ ML model predicts fraud likelihood (0-1)
   └─ If > 0.7 → Challenge or Reject
   └─ Store risk_score in txn record

4. Categorize Outcomes (2ms)
   ├─ APPROVED: 97% of txns
   ├─ CHALLENGED: Require 2FA (2% of txns)
   └─ REJECTED: Insufficient funds, sanctions (1% of txns)

5. Write to Clearing Ledger (2ms)
   └─ Insert batch record: {
       batch_id, timestamp, txn_count,
       approved_count, rejected_count
     }

TOTAL: ~15ms for 100 txns (0.15ms per txn)
```


## V. LAYER 3: STORAGE LAYER (Immutable & Encrypted)

### Purpose
Permanently record validated transactions with **cryptographic integrity** and **encryption at rest**. Write-once, read-many (WORM) design.

### Specifications

| Metric | Target | Implementation |
|--------|--------|-----------------|
| **Immutability** | 100% | Triggers prevent UPDATE/DELETE |
| **Encryption** | AES-256 | Postgres pgcrypto extension |
| **Hash Chain** | SHA-256 | Each record → hash of previous |
| **Batch Frequency** | 10 sec | 1000 txns per batch |
| **Retention** | 99 years | Cold storage after 7 years |
| **RPO/RTO** | <1 hour / <5 min | WAL archiving + PITR |


### Hash Chain Integrity
### Zero-Knowledge Proof Snapshot

``


## VI. PERFORMANCE BENCHMARKS

### End-to-End Latency Breakdown

| Component | Latency | Cumulative |
|-----------|---------|------------|
| **Layer 1 Acceptance** | 5ms | 5ms |
| **Network RTT** | 2ms | 7ms |
| **Layer 2 Batch Wait** | 0-1000ms | ~500ms (avg) |
| **Layer 2 Validation** | 3ms | 503ms |
| **Network RTT** | 2ms | 505ms |
| **Layer 3 Write** | 5ms | 510ms |
| **Total to Settlement** | - | **~500ms** |

**User Experience:**
- Immediate: "Processing..." UI at 5ms
- Fast: "Validating..." at 510ms
- Final: "Settled" email at 30 sec (batch confirmation)

**Mall Position:** Between blockchain systems (fast) and traditional banking (slower), optimized for **e-commerce** use case.

---

## VII. COMPARISON WITH OTHER ARCHITECTURES

### Single-Layer (Traditional Approach)

```
User → Single Ledger (validate + write simultaneously)
```

**Pros:** Simple, deterministic
**Cons:** 
- Can't accept transactions faster than write speed
- Block time = user latency (500ms+)
- No separation of concerns

---

### 2-Layer (Blockchain Model)

```
Layer 1 (Mempool): Accept txns
  ↓
Layer 2 (Chain): Permanent record
```

**Pros:** Fast acceptance, permanent immutability
**Cons:**
- No intermediate validation
- Fraud/double-spend risk until finality
- Batch confirmation latency (10-60 sec)

---

### 3-Layer Decoupled (PiMall)

```
Layer 1 (Transaction): 5ms acceptance
  ↓ (batch every 100 txns or 1 sec)
Layer 2 (Validation): 15ms integrity checks
  ↓ (batch every 1000 txns or 10 sec)
Layer 3 (Storage): 10ms immutable write
```

**Pros:**
- Fast user experience (5ms)
- Fraud prevention before storage (Layer 2)
- Immutable record (Layer 3)
- Clear separation of concerns
- Easy to audit/debug

**Cons:**
- More infrastructure (3 systems to operate)
- Need strong consistency between layers

---

## VIII. SYSTEM STRENGTHS vs. COMPETITORS

### vs. Traditional Banking (SWIFT, FedWire)

| Aspect | PiMall | Traditional |
|--------|--------|-------------|
| **Settlement Speed** | <1 sec | 4+ hours |
| **24/7 Availability** | Yes | Business hours only |
| **Finality** | After Layer 3 (~500ms) | Next business day |
| **Transparency** | Full ledger visible (RLS) | Black box |
| **Scalability** | Horizontal (add servers) | Vertical (bigger mainframes) |
| **Cost** | <$0.01 per txn | $5-25 per txn |

**Winner:** PiMall for speed/cost/transparency

---

### vs. Blockchain (Bitcoin, Ethereum)

| Aspect | Mall | Blockchain |
|--------|--------|-----------|
| **Finality** | ~500ms (Layer 3 commit) | 10-60 min (confirmation blocks) |
| **Throughput** | 30-50 TPS | 7-15 TPS |
| **User Fee** | 1% GEM | $5-100+ in gas |
| **Privacy** | RLS-based (encrypted) | Pseudo-anonymous (transparent) |
| **Centralization** | Trusted operator | Decentralized consensus |
| **Fraud Reversal** | Yes (admin override) | No (immutable chain) |

**Winner:** PiMall for speed/cost; Blockchain for trustlessness

---

### vs. Modern FinTech (Square, Stripe)

| Aspect | Mall | FinTech |
|--------|--------|--------|
| **Settlement** | <1 sec | 1-3 days |
| **International** | Native multi-currency | Requires intermediaries |
| **Infrastructure** | Own database | Third-party API dependency |
| **User Control** | Self-custodial | Centralized accounts |
| **Audit Trail** | Full cryptographic chain | Limited visibility |
| **Regulatory** | Self-regulated | Heavily regulated |

**Winner:** PiMall for speed/control; FinTech for regulation/trust

---

## IX. SYSTEM STRENGTHS (Core Differentiators)

### 1. High Concurrency with Integrity
Unlike traditional banking (batch processing), PiMall accepts transactions **instantly** while maintaining fraud detection.

### 2. Immutable Audit Trail
SHA-256 hash chain + cryptographic signatures enable **post-hoc verification** without blockchain consensus overhead.

### 3. Layered Fraud Prevention
- Layer 1: Accept fast
- Layer 2: Validate thoroughly
- Layer 3: Record immutably

Fraud detected in Layer 2 → rejected before permanent storage.

### 4. Sub-30ms Global Latency
Edge caching (Cloudflare) + database in-region + connection pooling → true 20-30ms target.

### 5. Zero-Knowledge Privacy
Users can verify system integrity (total balances) without exposing individual transactions.

### 6. Audit-Ready Compliance
Every transaction has:
- Validation rules passed
- Risk score
- Timestamp
- User/merchant/method tracked
- ISO 20022 compatible formatting

---

## X .  CONCLUSION

Mall's **3-Layer Decoupled Ledger** represents a balanced approach between:
- **Blockchain:** Immutability + transparency
- **Traditional Banking:** Trust + regulation + fraud prevention
- **Modern FinTech:** Speed + user experience

By separating acceptance, validation, and storage, PiMall achieves:
- **20-30ms latency** (5ms acceptance + 500ms settlement)
- **30-50 TPS throughput** (sufficient for e-commerce)
- **Zero fraud post-storage** (Layer 2 filters before Layer 3)
- **Full audit compliance** (cryptographic chain + RLS)

This makes PiMall suitable for **mission-critical financial systems** serving millions of users with institutional-grade security.


**Document Version:** 1.0  
**Last Updated:** 2026
**Author:**  LÊ TUẤN KHANH
**Status:** Technical Reference (Confidential)
