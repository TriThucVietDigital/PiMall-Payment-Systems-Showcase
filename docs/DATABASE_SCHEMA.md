# DATABASE SCHEMA DOCUMENTATION
## PiMall: Digital Central Bank Infrastructure for E-commerce, Internal Credit & RWA

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Database:** PostgreSQL 15+  
**Status:** Production-Ready

---

## TABLE OF CONTENTS

1. [Overview & Philosophy](#overview)
2. [Core Tables](#core-tables)
3. [Commerce System](#commerce-system)
4. [Financial & Ledger System](#financial--ledger-system)
5. [Row-Level Security (RLS)](#row-level-security)
6. [Indexes & Performance](#indexes--performance)
7. [Data Integrity & Constraints](#data-integrity--constraints)
8. [Backup & Disaster Recovery](#backup--disaster-recovery)

---

## OVERVIEW

### Purpose
PiMall database schema implements a **digital central bank infrastructure** combining:
- **E-commerce Platform:** Merchants, products, orders, payments
- **Internal Credit System:** GEM currency with 70-100% collateral backing
- **Real-World Assets (RWA):** Escrow contracts, collateral tracking
- **Audit & Compliance:** Immutable ledger, transaction trails, KYC/AML

### Architecture Principles
- **Row-Level Security (RLS):** Merchant data isolation at database level
- **Immutability:** Append-only ledgers with SHA-256 hash chains
- **Auditability:** Every transaction tracked with cryptographic signatures
- **Performance:** Sub-30ms latency, 30-50 TPS throughput target
- **Compliance:** GDPR/CCPA privacy-by-design, KYC/AML integration

### Technology Stack
- **DBMS:** PostgreSQL 15+ with pgcrypto, uuid-ossp extensions
- **Encryption:** AES-256 (pgcrypto), TLS 1.3 (in-transit)
- **Hashing:** SHA-256 for audit chains, immutability verification
- **Archival:** WAL-based point-in-time recovery, monthly snapshots

---

## CORE TABLES

### 1. USER PROFILES (auth.users extension)

```sql
user_profiles
├── id (UUID PK) → Supabase Auth reference
├── pi_user_id (TEXT UNIQUE) → Pi Network identity
├── pi_username (TEXT) → Merchant/seller display name
├── display_name (TEXT NOT NULL)
├── avatar_url (TEXT)
├── internal_credit_score (INTEGER 300-850) → CreditScore tier system
├── credit_tier (TEXT) → {Poor, Fair, Standard, Good, Excellent}
├── contribution_points (INTEGER) → Community reputation
├── cp_level (TEXT) → {Bronze, Silver, Gold, Platinum, Diamond}
├── kyc_status (TEXT) → {pending, verified, rejected}
├── is_verified (BOOLEAN)
├── created_at, updated_at, last_login_at (TIMESTAMPTZ)
└── CONSTRAINTS:
    ├── PK: id
    ├── RLS: Users see own profile only
    └── Audit: Tracked in audit_logs
```

**Purpose:** User identity, credit scoring, KYC verification

**RLS Policy:**
```sql
-- Users view own profile
CREATE POLICY "Users can view own profile" 
  ON user_profiles FOR SELECT 
  USING (auth.uid() = id);

-- Users update own profile
CREATE POLICY "Users can update own profile" 
  ON user_profiles FOR UPDATE 
  USING (auth.uid() = id);

-- Admins can view all (requires admin role)
CREATE POLICY "Admins can view all profiles" 
  ON user_profiles FOR SELECT 
  USING (current_setting('app.current_role') = 'admin');
```

---

### 2. PI WALLET BINDINGS

```sql
pi_wallet_bindings
├── id (UUID PK)
├── user_id (UUID FK → user_profiles.id)
├── pi_wallet_hash (TEXT UNIQUE) → Hashed for security
├── verified_at (TIMESTAMPTZ)
├── signature_proof (TEXT) → Cryptographic proof
├── is_primary (BOOLEAN)
├── status (TEXT) → {active, suspended, revoked}
└── CONSTRAINTS:
    ├── UNIQUE INDEX on (user_id) WHERE is_primary=true
    └── RLS: Users manage own wallets
```

**Purpose:** Link user accounts to Pi Network wallet addresses

---

## COMMERCE SYSTEM

### 3. MERCHANT STORES

```sql
merchant_stores
├── store_id (UUID PK)
├── owner_user_id (UUID FK → auth.users.id)
├── store_name (TEXT NOT NULL)
├── store_slug (TEXT UNIQUE) → URL-friendly identifier
├── store_description (TEXT)
├── store_avatar_url, store_cover_url (TEXT)
├── contact_info (JSONB) → {phone, email, address, bank_account}
├── operating_hours (JSONB) → {monday: "9-17", ...}
├── business_license_verified (BOOLEAN)
├── verification_notes (TEXT)
├── store_status (TEXT) → {pending, active, suspended, closed}
├── contribution_points (INTEGER)
├── gem_deposit_required, gem_deposit_paid (DECIMAL)
├── gem_deposit_locked (BOOLEAN)
└── CONSTRAINTS:
    ├── PK: store_id
    ├── FK: owner_user_id → auth.users
    ├── UNIQUE: store_slug
    └── RLS: Owner sees own store only
```

**Purpose:** Merchant account, store metadata, business verification

**RLS Policy:**
```sql
CREATE POLICY "Merchants view own store" 
  ON merchant_stores FOR SELECT 
  USING (owner_user_id = auth.uid());

CREATE POLICY "Merchants update own store" 
  ON merchant_stores FOR UPDATE 
  USING (owner_user_id = auth.uid());

CREATE POLICY "Public can view active stores" 
  ON merchant_stores FOR SELECT 
  USING (store_status = 'active');
```

---

### 4. PRODUCTS

```sql
products
├── product_id (UUID PK)
├── store_id (UUID FK → merchant_stores.store_id)
├── product_name (TEXT NOT NULL)
├── description (TEXT)
├── price_pi (DECIMAL NOT NULL, >0) → Priced in Pi currency
├── price_vnd (DECIMAL) → Optional VND reference
├── stock_quantity (INTEGER >=0)
├── sku (TEXT)
├── category (TEXT)
├── tags (TEXT[]) → Search tags
├── specifications (JSONB) → {color, size, material, ...}
├── product_status (TEXT) → {draft, active, out_of_stock, discontinued}
├── view_count, sale_count (INTEGER)
└── CONSTRAINTS:
    ├── FK: store_id
    ├── CHECK: price_pi > 0
    └── RLS: Store owner can edit own products
```

**RLS Policy:**
```sql
CREATE POLICY "Store owners can manage products" 
  ON products FOR ALL 
  USING (store_id IN (
    SELECT store_id FROM merchant_stores 
    WHERE owner_user_id = auth.uid()
  ));

CREATE POLICY "Public view active products" 
  ON products FOR SELECT 
  USING (product_status = 'active');
```

---

### 5. ORDERS

```sql
orders
├── order_id (UUID PK)
├── buyer_user_id (UUID FK → user_profiles.id)
├── seller_user_id (UUID FK → user_profiles.id)
├── store_id (UUID FK → merchant_stores.store_id)
├── product_id (UUID FK → products.product_id)
├── quantity (INTEGER >0)
├── total_amount_pi (DECIMAL NOT NULL)
├── platform_fee_gem (DECIMAL) → 1% fee in GEM
├── platform_fee_paid (BOOLEAN)
├── fee_due_date (TIMESTAMPTZ) → 30-day deadline for fee
├── fee_transaction_id (UUID FK → gem_fees.id)
├── order_status (TEXT) → {pending, confirmed, shipped, delivered, cancelled}
├── payment_method (TEXT) → {GEM, PI_EXCHANGE, PI_GCV}
├── created_at, updated_at (TIMESTAMPTZ)
└── CONSTRAINTS:
    ├── FKs: buyer, seller, store, product
    ├── CHECK: quantity > 0
    └── RLS: Buyer/seller see own orders
```

**RLS Policy:**
```sql
CREATE POLICY "Users view own orders" 
  ON orders FOR SELECT 
  USING (
    buyer_user_id = auth.uid() 
    OR seller_user_id = auth.uid()
  );

CREATE POLICY "Buyers can update own orders" 
  ON orders FOR UPDATE 
  USING (buyer_user_id = auth.uid());
```

---

## FINANCIAL & LEDGER SYSTEM

### 6. WALLETS (3-Layer Architecture)

```sql
wallets
├── id (UUID PK)
├── user_id (UUID FK → user_profiles.id)
├── slot_number (INTEGER) → {1: Application, 2: Clearing, 3: Settlement}
├── slot_name (TEXT) → {Application, Clearing, Settlement}
├── wallet_address (TEXT UNIQUE)
├── wallet_name (TEXT)
├── balance (BIGINT >=0) → GEM balance in base units
├── gem_fee_owed (DECIMAL >=0) → Platform fees owed (30-day credit)
├── gem_fee_max (DECIMAL) → Max allowed credit: 100 GEM
├── gem_fee_deadline (TIMESTAMPTZ) → Fee payment deadline
├── is_withdrawal_blocked (BOOLEAN) → Block if balance<0
├── status (TEXT) → {ACTIVE, FROZEN, CLOSED}
└── CONSTRAINTS:
    ├── UNIQUE: (user_id, slot_number)
    ├── CHECK: balance >= 0
    ├── CHECK: gem_fee_owed >= 0
    ├── CHECK: gem_fee_owed <= gem_fee_max
    └── RLS: Users see own wallets
```

**Purpose:** 3-layer wallet system for high-concurrency handling

**RLS Policy:**
```sql
CREATE POLICY "Users view own wallets" 
  ON wallets FOR SELECT 
  USING (user_id = auth.uid());

CREATE POLICY "Users cannot directly modify" 
  ON wallets FOR UPDATE 
  USING (false); -- Only system can update via triggers
```

---

### 7. COLLATERAL RESERVES (70-100% Backing)

```sql
collateral_reserves
├── id (UUID PK)
├── wallet_id (UUID FK → wallets.id)
├── collateral_type (TEXT) → {VND, XRP, Pi}
├── source_amount (NUMERIC) → Amount of collateral asset
├── gem_amount (BIGINT) → GEM amount backed
├── conversion_rate (NUMERIC) → Rate at time of backing
├── rate_timestamp (TIMESTAMPTZ)
├── oracle_source (TEXT) → {Gate.io, OKX, MEXC, ...}
├── oracle_price_usd (NUMERIC)
├── status (TEXT) → {active, released, liquidated}
├── backed_at, released_at (TIMESTAMPTZ)
└── CONSTRAINTS:
    ├── FK: wallet_id
    ├── CHECK: collateral_type IN (...)
    └── RLS: Wallet owner sees own collateral
```

**Purpose:** Track real-world asset backing for GEM currency

**Example:**
```
User deposits: 1,000,000 VND
Backing rate: 85% (standard)
GEM issued: 850 GEM
Collateral stored in: collateral_reserves
Status: Active
```

---

### 8. LEDGER ENTRIES (3-Layer Immutable)

```sql
ledger_entries
├── id (UUID PK)
├── entry_sequence (BIGSERIAL UNIQUE) → Global ordering
├── ledger_hash (TEXT UNIQUE) → SHA-256 of (seq, type, from, to, amount, prev_hash)
├── previous_hash (TEXT) → Hash chain for immutability
├── transaction_type (TEXT) → {
│   ├── DEPOSIT (VND→GEM)
│   ├── WITHDRAWAL (GEM→VND)
│   ├── TRANSFER_P2P (GEM P2P)
│   ├── ORDER_PAYMENT (Order settlement)
│   ├── GEM_PLATFORM_FEE (1% fee deduction)
│   ├── FEE_CREDIT_APPLIED (30-day extension)
│   ├── BLOCK_SETTLEMENT (Layer 3 finality)
│   └── ARCHIVE (Move to cold storage)
├── from_wallet_id (UUID FK)
├── to_wallet_id (UUID FK)
├── amount (BIGINT) → Amount in base units
├── balance_before, balance_after (BIGINT)
├── status (TEXT) → {pending, committed, finalized, archived}
├── settlement_time (TIMESTAMPTZ)
├── block_number (BIGINT FK → blocks.block_number)
└── CONSTRAINTS:
    ├── IMMUTABLE: Triggers prevent UPDATE/DELETE
    ├── APPEND-ONLY: INSERT only
    └── RLS: Users see own entries
```

**Immutability Trigger:**
```sql
CREATE OR REPLACE FUNCTION prevent_ledger_modification()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Ledger entries are immutable';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER protect_ledger_update
  BEFORE UPDATE ON ledger_entries
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_modification();

CREATE TRIGGER protect_ledger_delete
  BEFORE DELETE ON ledger_entries
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_modification();
```

---

### 9. GEM FEES (Receivable Tracking)

```sql
gem_fees
├── id (UUID PK)
├── user_id (UUID FK → user_profiles.id)
├── order_id (UUID FK → orders.order_id)
├── amount (DECIMAL NOT NULL) → 1% of order total
├── due_date (TIMESTAMPTZ) → 30 days from order date
├── paid_date (TIMESTAMPTZ)
├── status (TEXT) → {pending, partial, paid, waived, defaulted}
├── payment_method (TEXT)
├── ledger_entry_id (UUID FK → ledger_entries.id)
└── CONSTRAINTS:
    ├── CHECK: amount > 0
    ├── PK: id
    └── RLS: Users see own fees
```

**Purpose:** Track 1% platform fees and 30-day credit lines

**Example Fee Calculation:**
```
Order: 10 Pi
GCV Rate: 1π = $314,159 USD
VND Rate: 1 USD = 25,800 VND
Total VND: 10 × 314,159 × 25,800 = 81,073,620 VND

Platform Fee (1%): 810,736 VND = 0.81 GEM
Due Date: NOW() + 30 days
Status: Pending
```

---

### 10. TRANSACTIONS (General)

```sql
transactions
├── id (UUID PK)
├── from_user_id (UUID FK → user_profiles.id)
├── to_user_id (UUID FK → user_profiles.id)
├── transaction_type (TEXT) → {transfer, payment, deposit, withdrawal}
├── pi_type (TEXT) → {GCV, EXCHANGE, PINK}
├── amount (DECIMAL NOT NULL)
├── status (TEXT) → {pending, success, failed, cancelled}
├── fee (DECIMAL)
├── reference_id (TEXT) → Link to order/payment
├── created_at, completed_at (TIMESTAMPTZ)
└── CONSTRAINTS:
    ├── CHECK: amount > 0
    └── RLS: Users see own transactions
```

---

## ROW-LEVEL SECURITY (RLS)

### RLS Architecture

| Table | Read | Insert | Update | Delete |
|-------|------|--------|--------|--------|
| **user_profiles** | Self | Admin | Self | Admin |
| **pi_wallet_bindings** | Self | Self | Self | Admin |
| **merchant_stores** | Owner + Public | Owner | Owner | Admin |
| **products** | Owner + Public | Owner | Owner | Admin |
| **orders** | Buyer/Seller | Buyer | Buyer/Seller | Admin |
| **wallets** | Self | System | System | Admin |
| **ledger_entries** | Self (RLS) | System | None | None |
| **gem_fees** | Self | System | System | Admin |
| **audit_logs** | Self (RLS) | System | None | None |

### Example RLS Implementation

**Table: orders (Multi-stakeholder access)**
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy 1: Buyers see own orders
CREATE POLICY "Buyer access" 
  ON orders FOR SELECT 
  USING (buyer_user_id = auth.uid());

-- Policy 2: Sellers see orders for their products
CREATE POLICY "Seller access" 
  ON orders FOR SELECT 
  USING (
    product_id IN (
      SELECT product_id FROM products 
      WHERE store_id IN (
        SELECT store_id FROM merchant_stores 
        WHERE owner_user_id = auth.uid()
      )
    )
  );

-- Policy 3: Admins see all
CREATE POLICY "Admin access" 
  ON orders FOR ALL 
  USING (current_setting('app.current_role') = 'admin');

-- Policy 4: Buyers can update own orders
CREATE POLICY "Buyer update" 
  ON orders FOR UPDATE 
  USING (buyer_user_id = auth.uid())
  WITH CHECK (buyer_user_id = auth.uid());
```

**Table: ledger_entries (Financial privacy)**
```sql
ALTER TABLE ledger_entries ENABLE ROW LEVEL SECURITY;

-- Policy: Users see only ledger entries involving their wallets
CREATE POLICY "User ledger visibility" 
  ON ledger_entries FOR SELECT 
  USING (
    from_wallet_id IN (SELECT id FROM wallets WHERE user_id = auth.uid())
    OR
    to_wallet_id IN (SELECT id FROM wallets WHERE user_id = auth.uid())
  );

-- Policy: System inserts only (immutable)
CREATE POLICY "System insert only" 
  ON ledger_entries FOR INSERT 
  WITH CHECK (current_setting('app.current_role') = 'system');

-- Policy: Prevent all updates/deletes
CREATE POLICY "Prevent modifications" 
  ON ledger_entries FOR UPDATE 
  USING (false);

CREATE POLICY "Prevent deletes" 
  ON ledger_entries FOR DELETE 
  USING (false);
```

---

## INDEXES & PERFORMANCE

### Primary Indexes

```sql
-- Users & Auth
CREATE INDEX idx_user_profiles_kyc_status ON user_profiles(kyc_status);
CREATE INDEX idx_pi_wallet_bindings_primary ON pi_wallet_bindings(user_id) 
  WHERE is_primary = true;

-- Commerce
CREATE INDEX idx_merchant_stores_owner ON merchant_stores(owner_user_id);
CREATE INDEX idx_merchant_stores_status ON merchant_stores(store_status);
CREATE INDEX idx_merchant_stores_slug ON merchant_stores(store_slug);
CREATE INDEX idx_products_store ON products(store_id);
CREATE INDEX idx_products_status ON products(product_status);
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_orders_buyer ON orders(buyer_user_id);
CREATE INDEX idx_orders_seller ON orders(seller_user_id);
CREATE INDEX idx_orders_status ON orders(order_status);
CREATE INDEX idx_orders_store ON orders(store_id);

-- Financial
CREATE INDEX idx_wallets_user ON wallets(user_id);
CREATE INDEX idx_wallets_slot ON wallets(user_id, slot_number);
CREATE UNIQUE INDEX idx_wallets_unique_slot ON wallets(user_id, slot_number);
CREATE INDEX idx_collateral_reserves_wallet ON collateral_reserves(wallet_id);
CREATE INDEX idx_collateral_reserves_status ON collateral_reserves(status);
CREATE INDEX idx_ledger_entries_from_wallet ON ledger_entries(from_wallet_id);
CREATE INDEX idx_ledger_entries_to_wallet ON ledger_entries(to_wallet_id);
CREATE INDEX idx_ledger_entries_type ON ledger_entries(transaction_type);
CREATE INDEX idx_ledger_entries_timestamp ON ledger_entries(created_at);
CREATE INDEX idx_gem_fees_user ON gem_fees(user_id);
CREATE INDEX idx_gem_fees_due_date ON gem_fees(due_date);
CREATE INDEX idx_gem_fees_status ON gem_fees(status);

-- Audit
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_event_type ON audit_logs(event_type);
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(created_at);
```

### Performance Targets
- **Query latency:** <20ms (p95)
- **Index scan:** <5ms for non-sequential reads
- **Full table scan:** Avoid (use indexes)
- **Concurrent connections:** 200 (pool size)

---

## DATA INTEGRITY & CONSTRAINTS

### Invariants (Never Broken)

```sql
-- 1. Collateral Ratio >= 70%
ALTER TABLE collateral_reserves 
  ADD CONSTRAINT check_collateral_ratio
  CHECK (
    CASE 
      WHEN collateral_type = 'VND' THEN gem_amount * 1000 <= source_amount
      WHEN collateral_type = 'XRP' THEN gem_amount * 4300 <= source_amount * 25800
      WHEN collateral_type = 'Pi' THEN gem_amount * 314159 <= source_amount * 25800
    END
  );

-- 2. GEM Fee <= Max Allowed (100 GEM)
ALTER TABLE wallets
  ADD CONSTRAINT check_gem_fee_max
  CHECK (gem_fee_owed <= gem_fee_max AND gem_fee_max = 100);

-- 3. Order Amount Positive
ALTER TABLE orders
  ADD CONSTRAINT check_order_amount
  CHECK (total_amount_pi > 0);

-- 4. Ledger Entry Hash Chain Integrity
-- Validated by application logic (not DB constraint)
-- See: lib/ledger/core.ts verify_hash_chain()
```

---

## BACKUP & DISASTER RECOVERY

### Strategy
- **RPO (Recovery Point Objective):** 1 minute (transaction log WAL)
- **RTO (Recovery Time Objective):** 5 minutes (failover)
- **Retention:** 30 days rolling, monthly snapshots for 1 year

### Backup Procedures
```sql
-- Full backup (monthly)
pg_dump --format=custom --jobs=4 pimall_db > backup_2026_03.dump

-- Point-in-time recovery
pg_restore --dbname=pimall_db_restored backup_2026_03.dump

-- WAL archival (continuous)
wal_level = replica
max_wal_senders = 10
archive_mode = on
archive_command = 'aws s3 cp %p s3://pimall-backups/wal/%f'
```

---

## AUDIT & COMPLIANCE

### Regulatory Compliance
- **GDPR:** Right to export (SELECT * FROM user_profiles WHERE id=?), soft-delete
- **CCPA:** Data privacy, opt-out mechanism
- **PCI DSS:** No card data stored (use payment provider)
- **SOX:** Audit trail, immutable records

### Monthly Audit Checklist
```
□ Verify ledger_entries hash chain integrity
□ Reconcile collateral_reserves backing ratio
□ Validate gem_fees aged over 30 days
□ Check for unauthorized RLS policy changes
□ Audit audit_logs for anomalies
□ Validate user KYC status updates
□ Test disaster recovery procedures
```

---

## SCHEMA VERSION HISTORY

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2026-03-26 | Added gem_fees, 1% platform fees, 30-day credit system |
| 1.9 | 2026-03-20 | Added Pi Pink token support to wallets |
| 1.8 | 2026-03-10 | Fixed merchant_stores schema conflict (migration 012) |
| 1.7 | 2026-03-01 | Implemented 3-layer ledger architecture |
| 1.0 | 2025-10-01 | Initial schema launch |

---

**End of Database Schema Documentation**  
*For questions, contact: architects@pimall.io*
