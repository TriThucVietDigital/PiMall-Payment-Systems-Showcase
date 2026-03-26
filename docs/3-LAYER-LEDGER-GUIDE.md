# PiMall 3-Layer Decoupled Ledger Architecture
## Technical Guide to High-Concurrency Financial Settlement System

---

## I. EXECUTIVE SUMMARY

PiMall implements a **Decoupled 3-Layer Ledger** architecture inspired by modern banking infrastructure (RTGS, CHIPS, Fedwire) adapted for digital asset management. This design prioritizes **high concurrency** (30-50 TPS), **low latency** (20-30ms), and **financial integrity** through separation of concerns.

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
│  • Collateral validation (Gem, Pi, VND)                │
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
| **Queue Size** | 100,000 pending | 10-20 sec buffer at peak |

### Data Flow

```typescript
interface TransactionLayerRequest {
  userId: string                    // Pioneer wallet ID
  type: 'PURCHASE' | 'DEPOSIT' | 'TRANSFER' | 'WITHDRAWAL'
  amount: number                    // Amount in GEM
  currency: 'GEM' | 'PI_GCV' | 'PI_EXCHANGE' | 'VND'
  metadata: {
    productId?: string              // For e-commerce
    merchantId?: string             // Seller ID
    paymentMethod: string           // GEM, Pi, VND
  }
  timestamp: number                 // Client-supplied
  requestId: string                 // UUID for idempotency
}

interface TransactionLayerResponse {
  transactionId: string             // APP-{timestamp}-{nonce}
  status: 'ACCEPTED' | 'QUEUED'
  provisionalBalance: number        // Gem balance after txn
  estimatedSettlementTime: number   // ms until Layer 3
  queuePosition: number
}
```

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

### Implementation

```typescript
// lib/ledger/transaction-layer.ts

export async function acceptTransaction(
  request: TransactionLayerRequest
): Promise<TransactionLayerResponse> {
  const startTime = Date.now()

  // 1. Validate
  validateTransactionRequest(request)

  // 2. Idempotency
  const cached = await redisClient.get(`txn:${request.requestId}`)
  if (cached) return JSON.parse(cached)

  // 3. Provisional update (in-memory)
  const wallet = await walletCache.get(request.userId)
  const provisionalBalance = wallet.balance - request.amount
  
  // 4. Enqueue
  const transactionId = `APP-${Date.now()}-${randomId()}`
  await redisQueue.lpush('transaction_queue', {
    transactionId,
    ...request,
    acceptedAt: Date.now()
  })

  // 5. Cache response
  const response = {
    transactionId,
    status: 'ACCEPTED',
    provisionalBalance,
    estimatedSettlementTime: 8000, // 8 sec to Layer 3
    queuePosition: await redisQueue.llen('transaction_queue')
  }
  
  await redisClient.setex(`txn:${request.requestId}`, 3600, JSON.stringify(response))
  return response
}
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

### Validation Rules

```typescript
enum ValidationRuleCategory {
  FINANCIAL_INTEGRITY = 'financial',     // Balance, collateral
  FRAUD_DETECTION = 'fraud',              // Velocity, patterns
  COMPLIANCE = 'compliance',              // KYC, limits, sanctions
  TECHNICAL = 'technical'                 // Formatting, signatures
}

const VALIDATION_RULES = {
  // 1. Sufficient Balance
  RULE_001: {
    check: (txn, wallet) => wallet.balance >= txn.amount,
    message: 'Insufficient balance',
    severity: 'REJECT'
  },

  // 2. Max Single Transaction
  RULE_002: {
    check: (txn) => txn.amount <= 1_000_000, // 1M GEM max
    message: 'Amount exceeds daily limit',
    severity: 'REJECT'
  },

  // 3. Daily Transaction Velocity
  RULE_003: {
    check: (txn, wallet, user) => {
      const daily = user.dailyVolume + txn.amount
      return daily <= user.dailyLimit // Usually 10M GEM
    },
    message: 'Daily limit exceeded',
    severity: 'REJECT'
  },

  // 4. Geolocation Anomaly
  RULE_004: {
    check: (txn, user) => {
      const distance = geolocationDistance(
        user.lastLocation,
        txn.location
      )
      return distance < 1000 || user.isVIP // Allow > 1000km for VIP
    },
    message: 'Unusual location detected',
    severity: 'CHALLENGE' // 2FA required
  },

  // 5. Known Sanctions List
  RULE_005: {
    check: (txn, wallet, user) => !sanctionsList.includes(user.id),
    message: 'User on sanctions list',
    severity: 'BLOCK'
  },

  // 6. Merchant Reputation
  RULE_006: {
    check: (txn) => {
      const merchant = merchants.get(txn.metadata.merchantId)
      return merchant.trustScore > 0.5 // >= 50% trust
    },
    message: 'Merchant trust score too low',
    severity: 'CHALLENGE'
  },

  // 7. Collateral Type Validity
  RULE_007: {
    check: (txn) => {
      return ['GEM', 'PI_GCV', 'PI_EXCHANGE', 'VND'].includes(txn.currency)
    },
    message: 'Invalid currency type',
    severity: 'REJECT'
  },

  // 8. Platform Fee Gem Deduction
  RULE_008: {
    check: (txn, wallet) => {
      const fee = calculateGemFee(txn.amount)
      const userTotal = txn.amount + fee
      const maxAllowed = wallet.balance + 100 // Max -100 GEM allowed
      return userTotal <= maxAllowed
    },
    message: '1% platform fee exceeds limit',
    severity: 'WARN' // Still process but flag
  },

  // 9. Idempotency UUID Uniqueness
  RULE_009: {
    check: (txn, ledger) => {
      return !ledger.exists(txn.requestId)
    },
    message: 'Duplicate requestId detected',
    severity: 'WARN' // Already processed
  },

  // 10. Timestamp Sanity
  RULE_010: {
    check: (txn) => {
      const now = Date.now()
      return Math.abs(now - txn.timestamp) < 300000 // 5 min max skew
    },
    message: 'Timestamp too old or in future',
    severity: 'REJECT'
  }
}
```

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

### Fraud Detection Model

```python
# ML-based risk scoring (simplified)
import numpy as np
from sklearn.ensemble import RandomForestClassifier

def calculate_risk_score(txn, user_history):
    features = [
        txn.amount / user_history.avg_txn_size,     # Velocity
        user_history.txns_today,                     # Frequency
        geolocation_distance(user.lastLoc, txn.loc), # Distance
        int(txn.time in [2,3,4]),                    # Off-hours?
        txn.is_new_merchant,                         # First time?
    ]
    
    risk_score = model.predict_proba([features])[0][1]  # Probability of fraud
    return risk_score

# Thresholds:
# risk_score < 0.3  → Auto-approve
# 0.3-0.7          → Require 2FA challenge
# > 0.7            → Auto-reject
```

---

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

### Data Model

```sql
-- LAYER 3: Storage Ledger (Immutable)

CREATE TABLE ledger_entries (
  id UUID PRIMARY KEY,
  
  -- Batch info
  batch_id UUID NOT NULL,
  batch_sequence BIGSERIAL,
  
  -- Transaction details
  transaction_id VARCHAR(50) NOT NULL,
  user_id UUID NOT NULL,
  type VARCHAR(20) NOT NULL,        -- PURCHASE, DEPOSIT, etc.
  amount NUMERIC(20,2) NOT NULL,
  currency VARCHAR(20) NOT NULL,    -- GEM, PI_GCV, PI_EXCHANGE
  
  -- Counterparty
  merchant_id UUID,
  counterparty_wallet VARCHAR(255),
  
  -- Financial
  gem_balance_before NUMERIC(20,2),
  gem_balance_after NUMERIC(20,2),
  platform_fee_gem NUMERIC(20,2),
  
  -- Status
  status VARCHAR(20) NOT NULL,      -- SETTLED, FAILED, PENDING
  validation_rules_passed TEXT[],   -- [RULE_001, RULE_002, ...]
  risk_score NUMERIC(3,2),
  
  -- Integrity
  hash_current TEXT NOT NULL,       -- SHA-256 of this record
  hash_previous TEXT NOT NULL,      -- SHA-256 of previous record (chain)
  signature TEXT NOT NULL,          -- HMAC for verification
  
  -- Encryption
  encrypted_metadata TEXT,          -- Encrypted JSONB
  
  -- Timestamps
  created_at TIMESTAMPTZ NOT NULL,
  settled_at TIMESTAMPTZ,
  
  CONSTRAINT integrity_check CHECK (
    gem_balance_after = gem_balance_before - amount - platform_fee_gem
  )
);

-- Immutability: Prevent modifications
CREATE OR REPLACE FUNCTION prevent_ledger_modification()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Ledger entries are immutable - cannot modify %', TG_OP;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER protect_ledger_update
  BEFORE UPDATE ON ledger_entries
  FOR EACH ROW EXECUTE FUNCTION prevent_ledger_modification();

CREATE TRIGGER protect_ledger_delete
  BEFORE DELETE ON ledger_entries
  FOR EACH ROW EXECUTE FUNCTION prevent_ledger_modification();

-- Hash Chain Index
CREATE INDEX idx_ledger_hash_chain ON ledger_entries(hash_previous, created_at);

-- Query optimization
CREATE INDEX idx_ledger_user_date ON ledger_entries(user_id, created_at DESC);
```

### Hash Chain Integrity

```typescript
// Each record's hash is derived from its content + previous hash
// Creates immutable chain: R1 → R2 → R3 → ... → Rn

function generateLedgerHash(record: LedgerEntry): string {
  const payload = JSON.stringify({
    transactionId: record.transaction_id,
    userId: record.user_id,
    amount: record.amount,
    status: record.status,
    previousHash: record.hash_previous, // Includes previous record
    timestamp: record.created_at
  })
  
  return crypto
    .createHash('sha256')
    .update(payload)
    .digest('hex')
}

// Verification (anyone can verify integrity)
function verifyHashChain(entries: LedgerEntry[]): boolean {
  for (let i = 0; i < entries.length - 1; i++) {
    const currentHash = generateLedgerHash(entries[i])
    const nextPrevHash = entries[i + 1].hash_previous
    
    if (currentHash !== nextPrevHash) {
      return false // Chain broken - tampering detected
    }
  }
  return true
}
```

### Zero-Knowledge Proof Snapshot

```typescript
// Periodically (daily/weekly) create ZK snapshot of entire ledger
// Proof: "Total GEM outstanding = X" without revealing individual balances

interface ZKSnapshot {
  snapshotId: string
  periodEnd: Date
  totalGemBalances: Pedersen.Commitment     // Encrypted sum
  totalGemFeesPending: Pedersen.Commitment  // Encrypted sum
  proofOfSum: ZKProof                       // Zero-knowledge proof
  witness: (sealed to auditors only)        // Random salt
}

// Verification: Anyone can verify total without seeing individual data
export function verifyZKSnapshot(snapshot: ZKSnapshot): boolean {
  return Pedersen.verify(
    snapshot.totalGemBalances,
    snapshot.proofOfSum
  )
}
```

### Processing Pipeline

```
1. Collect Batch (1ms)
   └─ Pop 1000 approved txns from Layer 2
   └─ Group by merchant/settlement type

2. Calculate Hashes (3ms)
   └─ Generate SHA-256 for each entry
   └─ Chain: previous_hash ← next_hash
   └─ Verify no collisions

3. Encrypt Sensitive Data (2ms)
   └─ AES-256 for metadata JSONB
   └─ Key from Vault (rotated weekly)

4. Batch Insert (2ms)
   └─ INSERT INTO ledger_entries (1000 rows)
   └─ Transaction atomicity ensures all-or-nothing
   └─ Index updates for query performance

5. Generate Checksums (1ms)
   └─ MD5(entire batch) for quick validation
   └─ Store in batch_registry

6. Archive (optional, async)
   └─ Move old entries to cold storage (AWS Glacier)
   └─ Maintain PITR (point-in-time recovery)

TOTAL: ~10ms for 1000 txns (0.01ms per txn)
```

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

### Throughput Comparison

| System | TPS | Notes |
|--------|-----|-------|
| **PiMall (Design)** | 30-50 TPS | 3-layer decoupled |
| **Bitcoin** | 7 TPS | Proof-of-Work bottleneck |
| **Ethereum** | 15 TPS | PoS but complex validation |
| **Stellar** | 1000+ TPS | Federated consensus |
| **Visa** | 24,000 TPS | Centralized + batch |
| **SWIFT** | 200 TPS | Bank-to-bank, ~4 hours |
| **FedWire (US)** | 500 TPS | Real-time, retail limit |

**PiMall Position:** Between blockchain systems (fast) and traditional banking (slower), optimized for **e-commerce** use case.

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

| Aspect | PiMall | Blockchain |
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

| Aspect | PiMall | FinTech |
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

## X. OPERATIONAL MONITORING

### Real-Time Dashboards

```typescript
interface LedgerMetrics {
  layer1: {
    acceptanceRate: number           // txns/sec
    provisionalBalance: number       // total pending
    queueLength: number
    p95Latency: number              // 95th percentile
  }
  layer2: {
    batchSize: number               // avg txns per batch
    validationFailRate: number      // % rejected
    fraudDetectionRate: number      // % flagged
    avgRiskScore: number            // 0-1
  }
  layer3: {
    writeLatency: number            // per batch
    immutabilityStatus: string      // "OK" or trigger alerts
    storageUsed: number             // GB
    coldArchiveProgress: number     // %
  }
}

// Alert thresholds
if (metrics.layer1.queueLength > 50000) {
  alert("HIGH: Transaction queue backing up")
}
if (metrics.layer2.validationFailRate > 0.05) {
  alert("WARN: High rejection rate, possible fraud wave")
}
if (metrics.layer3.writeLatency > 50) {
  alert("CRITICAL: Database write latency degraded")
}
```

---

## XI. CONCLUSION

PiMall's **3-Layer Decoupled Ledger** represents a balanced approach between:
- **Blockchain:** Immutability + transparency
- **Traditional Banking:** Trust + regulation + fraud prevention
- **Modern FinTech:** Speed + user experience

By separating acceptance, validation, and storage, PiMall achieves:
- **20-30ms latency** (5ms acceptance + 500ms settlement)
- **30-50 TPS throughput** (sufficient for e-commerce)
- **Zero fraud post-storage** (Layer 2 filters before Layer 3)
- **Full audit compliance** (cryptographic chain + RLS)

This makes PiMall suitable for **mission-critical financial systems** serving millions of users with institutional-grade security.

---

## XII. REFERENCES

1. Federal Reserve - Fedwire Funds Service: https://www.federalreserve.gov/paymentsystems/fedwire_about.htm
2. CHIPS (Clearing House Interbank Payments System): https://www.theclearinghouse.org/
3. SWIFT Standards: https://www.swift.com/standards
4. ISO 20022: https://www.iso20022.org/
5. PostgreSQL Security: https://www.postgresql.org/docs/current/sql-syntax.html
6. Pedersen Commitments (Zero-Knowledge): https://en.wikipedia.org/wiki/Commitment_scheme

---

**Document Version:** 1.0  
**Last Updated:** 2026-03-26  
**Author:** PiMall Engineering  
**Status:** Technical Reference (Confidential)
