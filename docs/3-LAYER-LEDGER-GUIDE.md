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

---

## Layer 1: Transaction Layer (High Concurrency)

**Purpose:** Accept transactions with minimal latency and provide immediate user feedback.

**Key Capabilities:**
- High concurrency handling
- Idempotency checks to prevent duplicates
- In-memory provisional balance updates
- Redis-backed queuing for downstream processing

**Processing Flow (End-to-end ~5ms):**
1. Request validation
2. Idempotency check
3. Provisional balance update (in-memory)
4. Enqueue to processing queue
5. Immediate response to user

---

## Layer 2: Validation Layer (Integrity & Fraud Prevention)

**Purpose:** Perform comprehensive checks on batched transactions before permanent storage.

**Key Features:**
- 10+ financial and compliance rules
- ML-based fraud detection & risk scoring
- Batch processing for efficiency
- Categorization: Approved / Challenged / Rejected

**Processing Flow (~15ms per batch of 100 txns):**
1. Batch collection from queue
2. Multi-rule validation
3. Risk scoring
4. Outcome categorization
5. Write to clearing ledger

---

## Layer 3: Storage Layer (Immutable & Encrypted)

**Purpose:** Provide permanent, tamper-proof record-keeping.

**Key Features:**
- Write-once (append-only) design
- SHA-256 hash chain for integrity
- AES-256 encryption at rest
- WAL archiving + Point-in-Time Recovery

---

## Performance Highlights

**End-to-End Latency:**
- Transaction acceptance: **<5ms** (user sees instant feedback)
- Full settlement: **~500ms** (average)
- Global target: **sub-30ms** for most operations

**Optimization Techniques:**
- Edge caching (CDN)
- Connection pooling
- Selective in-memory & Redis caching
- Batch processing for validation & storage

---

## Comparison with Common Architectures

| Architecture       | Acceptance Latency | Validation | Finality     | Strength                  |
|--------------------|--------------------|------------|--------------|---------------------------|
| **Traditional (Single-Layer)** | High              | Combined   | Slow         | Simplicity                |
| **Blockchain**     | Fast               | Limited    | 10-60 min    | Trustlessness             |
| **This 3-Layer**   | **<5ms**           | Thorough   | **~500ms**   | Speed + Security + Control|

**Advantages of 3-Layer Decoupled Design:**
- Instant user experience (Layer 1)
- Strong fraud prevention before storage (Layer 2)
- Full immutability & auditability (Layer 3)
- Easy scaling and debugging

---

## System Strengths

1. **High Concurrency + Financial Integrity** – Accepts transactions instantly while validating thoroughly.
2. **Immutable Audit Trail** – Cryptographic hash chain enables complete traceability.
3. **Layered Fraud Prevention** – Fraud caught in Layer 2 before it reaches permanent storage.
4. **Enterprise Compliance Ready** – Supports audit requirements and data protection standards.
5. **Scalable & Resilient** – Horizontal scaling, low RTO/RPO.

---

## Conclusion

This **3-Layer Decoupled Ledger** architecture strikes an optimal balance between speed, security, and reliability for modern high-volume payment systems. It delivers sub-30ms user experience, strong fraud protection, and institutional-grade immutability — making it suitable for mission-critical fintech platforms serving millions of users.

**Key Achievements:**
- Ultra-fast acceptance with immediate feedback
- Robust validation before immutable storage
- Cryptographic integrity and privacy controls
- Horizontal scalability and high availability

---


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


**Document Version:** 1.0  
**Authored by:** Khanh – Senior Backend Engineer  
**Purpose:** Portfolio & Technical Showcase
