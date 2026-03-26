# HỆ THỐNG SỔ CÁI 3 LỚP (3-LAYER LEDGER) - 314MALL
## Hướng dẫn Áp dụng Thực tế

---

## I. TỔNG QUAN KIẾN TRÚC

Hệ thống sổ cái 3 lớp của 314Mall được thiết kế dựa trên mô hình hệ thống thanh toán ngân hàng quốc tế, đảm bảo tính minh bạch, an toàn và khả năng kiểm toán cao.

\`\`\`
┌─────────────────────────────────────────────────────────────┐
│                    USER INTERFACE LAYER                      │
│              (Giao diện người dùng - Web/Mobile)             │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│           LAYER 1: APPLICATION LEDGER (Sổ Ứng dụng)         │
│  - Ghi nhận giao dịch ngay lập tức                          │
│  - Tạo "Pending" transactions                                │
│  - User-facing balance updates                              │
│  - High throughput, low latency                             │
└─────────────────────────┬───────────────────────────────────┘
                          │ Batch Processing (mỗi 10s)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│          LAYER 2: CLEARING LEDGER (Sổ Thanh toán)          │
│  - Xác thực và nhóm giao dịch                               │
│  - Tính toán gas fee và net settlement                       │
│  - Kiểm tra tính hợp lệ của collateral                      │
│  - Risk assessment & fraud detection                         │
└─────────────────────────┬───────────────────────────────────┘
                          │ Final Settlement (mỗi 60s)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│         LAYER 3: SETTLEMENT LEDGER (Sổ Quyết toán)         │
│  - Ghi nhận giao dịch cuối cùng (Immutable)                 │
│  - Generate blockchain-style hash chain                      │
│  - ISO 20022 compliance record                              │
│  - Audit trail & regulatory reporting                       │
└─────────────────────────────────────────────────────────────┘
\`\`\`

---

## II. CHI TIẾT TỪNG LỚP SỔ CÁI

### 📱 LAYER 1: APPLICATION LEDGER (Sổ Ứng dụng)

**Nhiệm vụ chính:**
1. **Instant User Feedback** - Phản hồi tức thì cho người dùng
2. **Transaction Initiation** - Khởi tạo giao dịch
3. **Balance Preview** - Hiển thị số dư tạm thời

**Cấu trúc dữ liệu:**
\`\`\`typescript
interface ApplicationTransaction {
  id: string                    // APP-{timestamp}-{random}
  userId: string                // Pioneer wallet address
  type: 'PURCHASE' | 'DEPOSIT' | 'TRANSFER'
  amount: number                // Amount in Gem
  status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED'
  timestamp: Date
  metadata: {
    productId?: string
    merchantId?: string
    paymentMethod: string
  }
}
\`\`\`

**Luồng xử lý:**
\`\`\`
1. User nhấn "Mua hàng 100 Gem"
   └─> Application Ledger tạo transaction APP-001 (status: PENDING)
   └─> UI hiển thị "Đang xử lý..."
   └─> Số dư hiển thị: 1000 - 100 = 900 Gem (tạm thời)

2. Transaction được đẩy vào Queue
   └─> Chờ Clearing Ledger xử lý

3. Sau 0.5-2 giây, status update
   └─> PENDING → PROCESSING (đã vào Clearing)
   └─> UI cập nhật: "Đang xác thực..."
\`\`\`

**Ví dụ code thực tế:**
\`\`\`typescript
// File: lib/services/application-ledger.ts

export async function createApplicationTransaction(params: {
  userId: string
  type: TransactionType
  amount: number
  metadata: Record<string, any>
}): Promise<ApplicationTransaction> {
  const transaction = {
    id: `APP-${Date.now()}-${generateRandomId()}`,
    userId: params.userId,
    type: params.type,
    amount: params.amount,
    status: 'PENDING',
    timestamp: new Date(),
    metadata: params.metadata
  }
  
  // Lưu vào temporary storage (Redis/Memory)
  await redis.set(`app:tx:${transaction.id}`, JSON.stringify(transaction))
  
  // Push to Clearing Queue
  await clearingQueue.add('process-transaction', transaction)
  
  console.log(`[v0] Application Ledger: Created ${transaction.id}`)
  
  return transaction
}
\`\`\`

---

### ⚙️ LAYER 2: CLEARING LEDGER (Sổ Thanh toán)

**Nhiệm vụ chính:**
1. **Batch Processing** - Xử lý giao dịch theo lô (batch)
2. **Validation** - Kiểm tra tính hợp lệ
3. **Risk Assessment** - Đánh giá rủi ro
4. **Fee Calculation** - Tính toán phí gas
5. **Net Settlement** - Tính toán thanh toán ròng

**Cấu trúc dữ liệu:**
\`\`\`typescript
interface ClearingBatch {
  batchId: string               // CLR-{date}-{sequence}
  transactions: ApplicationTransaction[]
  totalAmount: number
  totalGasFee: number
  netSettlement: number
  validationResults: ValidationResult[]
  createdAt: Date
  processedAt?: Date
  status: 'QUEUED' | 'PROCESSING' | 'VALIDATED' | 'SETTLED'
}

interface ValidationResult {
  transactionId: string
  isValid: boolean
  checks: {
    sufficientBalance: boolean
    collateralVerified: boolean
    riskScore: number           // 0-100 (0 = safe, 100 = high risk)
    fraudDetection: boolean
  }
  errors?: string[]
}
\`\`\`

**Luồng xử lý:**
\`\`\`
1. Mỗi 10 giây, Clearing Ledger thu thập transactions từ Queue
   └─> Tạo batch CLR-20250203-001 với 50 transactions

2. Validation Pipeline:
   ┌─> Check 1: Sufficient Balance
   ├─> Check 2: Collateral Verification (XRP, VND, Pi)
   ├─> Check 3: Risk Scoring (fraud detection)
   ├─> Check 4: Gas Fee Calculation (1% auto-deduct)
   └─> Check 5: Net Settlement Calculation

3. Kết quả:
   ├─> 48 transactions: VALID → Chuyển sang Settlement
   ├─> 2 transactions: INVALID → Return to Application (status: FAILED)
   └─> Batch status: VALIDATED

4. Gửi batch sang Settlement Ledger để ghi vĩnh viễn
\`\`\`

**Ví dụ code thực tế:**
\`\`\`typescript
// File: lib/services/clearing-ledger.ts

export async function processClearingBatch(): Promise<ClearingBatch> {
  console.log('[v0] Clearing Ledger: Starting batch processing...')
  
  // 1. Collect pending transactions
  const transactions = await clearingQueue.getJobs('active', 0, 100)
  
  const batch: ClearingBatch = {
    batchId: `CLR-${new Date().toISOString().split('T')[0]}-${sequenceNumber}`,
    transactions: transactions.map(job => job.data),
    totalAmount: 0,
    totalGasFee: 0,
    netSettlement: 0,
    validationResults: [],
    createdAt: new Date(),
    status: 'PROCESSING'
  }
  
  // 2. Validate each transaction
  for (const tx of batch.transactions) {
    const validation = await validateTransaction(tx)
    batch.validationResults.push(validation)
    
    if (validation.isValid) {
      const gasFee = tx.amount * 0.01 // 1% gas fee
      batch.totalAmount += tx.amount
      batch.totalGasFee += gasFee
      batch.netSettlement += (tx.amount - gasFee)
    }
  }
  
  batch.processedAt = new Date()
  batch.status = 'VALIDATED'
  
  // 3. Save to database
  await db.clearingBatches.insert(batch)
  
  // 4. Push to Settlement Queue
  await settlementQueue.add('finalize-batch', batch)
  
  console.log(`[v0] Clearing Ledger: Batch ${batch.batchId} validated with ${batch.validationResults.filter(v => v.isValid).length}/${batch.transactions.length} valid transactions`)
  
  return batch
}

async function validateTransaction(tx: ApplicationTransaction): Promise<ValidationResult> {
  const checks = {
    sufficientBalance: await checkBalance(tx.userId, tx.amount),
    collateralVerified: await verifyCollateral(tx.userId),
    riskScore: await calculateRiskScore(tx),
    fraudDetection: await detectFraud(tx)
  }
  
  return {
    transactionId: tx.id,
    isValid: checks.sufficientBalance && checks.collateralVerified && checks.riskScore < 70 && !checks.fraudDetection,
    checks
  }
}
\`\`\`

---

### 🔒 LAYER 3: SETTLEMENT LEDGER (Sổ Quyết toán)

**Nhiệm vụ chính:**
1. **Immutable Record** - Ghi nhận bất biến
2. **Hash Chain Generation** - Tạo chuỗi hash blockchain-style
3. **ISO 20022 Compliance** - Tuân thủ chuẩn quốc tế
4. **Audit Trail** - Dấu vết kiểm toán
5. **Regulatory Reporting** - Báo cáo cho cơ quan quản lý

**Cấu trúc dữ liệu:**
\`\`\`typescript
interface SettlementBlock {
  blockNumber: number           // Sequential: 1, 2, 3...
  batchId: string              // Reference to Clearing batch
  transactions: FinalTransaction[]
  timestamp: Date
  previousHash: string         // Hash of previous block
  currentHash: string          // SHA-256 hash of this block
  merkleRoot: string           // Merkle root of all transactions
  totalGemMinted: number
  totalGemBurned: number
  totalGasFee: number
  iso20022Message?: string     // pacs.008 XML message
  signature: string            // Digital signature
  validator: string            // Master wallet address
}

interface FinalTransaction {
  id: string                   // STL-{blockNumber}-{txIndex}
  originalId: string           // Link to Application tx
  from: string
  to: string
  amount: number
  gasFee: number
  collateralType: 'VND' | 'XRP' | 'PI' | 'IP_TECH'
  status: 'EXECUTED'           // Always EXECUTED in Settlement
  executedAt: Date
  hash: string                 // Individual tx hash
}
\`\`\`

**Luồng xử lý:**
\`\`\`
1. Mỗi 60 giây, Settlement Ledger nhận batch từ Clearing
   └─> Batch CLR-20250203-001 (48 valid transactions)

2. Generate Settlement Block:
   ├─> Block Number: 12345 (sequential)
   ├─> Previous Hash: 0xabc123... (from block 12344)
   ├─> Merkle Root: Calculate from 48 transactions
   └─> Current Hash: SHA-256(blockNumber + previousHash + merkleRoot + timestamp)

3. Create ISO 20022 Message:
   └─> Generate pacs.008 XML for regulatory compliance
   └─> Include: Instruction ID, Settlement Amount, Charge Info

4. Sign and Lock:
   ├─> Digital signature using Master Wallet private key
   ├─> Write to permanent storage (PostgreSQL + Backup)
   └─> Status: IMMUTABLE (cannot be changed)

5. Update user balances (final):
   └─> Application Ledger: PENDING → COMPLETED
   └─> User UI: "Giao dịch hoàn tất"
\`\`\`

**Ví dụ code thực tế:**
\`\`\`typescript
// File: lib/services/settlement-ledger.ts

export async function finalizeSettlementBlock(batch: ClearingBatch): Promise<SettlementBlock> {
  console.log(`[v0] Settlement Ledger: Finalizing batch ${batch.batchId}`)
  
  // 1. Get previous block
  const previousBlock = await db.settlementBlocks.findLast()
  const blockNumber = (previousBlock?.blockNumber || 0) + 1
  
  // 2. Convert to final transactions
  const finalTransactions: FinalTransaction[] = batch.transactions
    .filter((_, idx) => batch.validationResults[idx].isValid)
    .map((tx, idx) => ({
      id: `STL-${blockNumber}-${idx}`,
      originalId: tx.id,
      from: tx.userId,
      to: tx.metadata.merchantId || 'SYSTEM',
      amount: tx.amount,
      gasFee: tx.amount * 0.01,
      collateralType: tx.metadata.collateralType || 'VND',
      status: 'EXECUTED',
      executedAt: new Date(),
      hash: generateTxHash(tx)
    }))
  
  // 3. Calculate Merkle Root
  const merkleRoot = calculateMerkleRoot(finalTransactions.map(tx => tx.hash))
  
  // 4. Create block
  const block: SettlementBlock = {
    blockNumber,
    batchId: batch.batchId,
    transactions: finalTransactions,
    timestamp: new Date(),
    previousHash: previousBlock?.currentHash || '0'.repeat(64),
    currentHash: '',
    merkleRoot,
    totalGemMinted: 0,
    totalGemBurned: batch.netSettlement,
    totalGasFee: batch.totalGasFee,
    iso20022Message: '',
    signature: '',
    validator: MASTER_WALLET
  }
  
  // 5. Generate hash
  block.currentHash = generateBlockHash(block)
  
  // 6. Generate ISO 20022 message
  block.iso20022Message = generatePacs008Message({
    instructionId: block.batchId,
    endToEndId: `314M-${blockNumber}`,
    amount: block.totalGemBurned,
    currency: 'GEM',
    debtorName: '314Mall Users',
    debtorAccount: 'COLLECTIVE',
    creditorName: '314Mall Settlement',
    creditorAccount: 'SETTLEMENT_ACCOUNT',
    remittanceInfo: `Batch settlement for ${finalTransactions.length} transactions`,
    chargeAmount: block.totalGasFee,
    chargeCurrency: 'GEM'
  })
  
  // 7. Digital signature
  block.signature = signBlock(block, MASTER_WALLET_PRIVATE_KEY)
  
  // 8. Save to permanent storage
  await db.settlementBlocks.insert(block)
  await backupToS3(block) // Backup to cloud storage
  
  // 9. Update Application Ledger statuses
  for (const tx of finalTransactions) {
    await redis.set(`app:tx:${tx.originalId}:status`, 'COMPLETED')
  }
  
  console.log(`[v0] Settlement Ledger: Block ${blockNumber} finalized with hash ${block.currentHash.slice(0, 10)}...`)
  
  return block
}

function generateBlockHash(block: SettlementBlock): string {
  const data = `${block.blockNumber}${block.previousHash}${block.merkleRoot}${block.timestamp.toISOString()}`
  return crypto.createHash('sha256').update(data).digest('hex')
}

function calculateMerkleRoot(hashes: string[]): string {
  if (hashes.length === 0) return '0'.repeat(64)
  if (hashes.length === 1) return hashes[0]
  
  const newHashes: string[] = []
  for (let i = 0; i < hashes.length; i += 2) {
    const left = hashes[i]
    const right = hashes[i + 1] || left
    const combined = crypto.createHash('sha256').update(left + right).digest('hex')
    newHashes.push(combined)
  }
  
  return calculateMerkleRoot(newHashes)
}
\`\`\`

---

## III. ĐỒNG BỘ DỮ LIỆU GIỮA CÁC LỚP

### 🔄 Cơ chế Đồng bộ

\`\`\`
Application Ledger  ──▶  Clearing Ledger  ──▶  Settlement Ledger
     (0-2s)                   (10s batch)           (60s final)
        │                          │                      │
        │                          │                      │
        ▼                          ▼                      ▼
   Redis/Memory              PostgreSQL              PostgreSQL
   (Temporary)               (Validated)             (Immutable)
        │                          │                      │
        └──────────────────────────┴──────────────────────┘
                                   │
                                   ▼
                          User Balance Update
                        (Final state in wallet)
\`\`\`

**Timeline Example:**

\`\`\`
T=0s     User clicks "Buy Product 100 Gem"
         └─> App Ledger: Create APP-001 (PENDING)
         └─> UI shows: "Processing..."

T=0.5s   App Ledger pushes to Clearing Queue
         └─> Status: PENDING → PROCESSING

T=10s    Clearing Ledger processes batch
         └─> Validation checks run
         └─> Batch CLR-001 created (VALIDATED)

T=12s    App Ledger receives validation result
         └─> Status: PROCESSING → VALIDATED

T=60s    Settlement Ledger finalizes block
         └─> Block 12345 created with hash chain
         └─> ISO 20022 message generated

T=61s    Settlement confirms to App Ledger
         └─> Status: VALIDATED → COMPLETED
         └─> User balance updated (final)
         └─> UI shows: "Transaction completed ✓"
\`\`\`

### 📡 API Endpoints cho Đồng bộ

\`\`\`typescript
// 1. Application Layer API
POST /api/transactions/create
→ Creates transaction in App Ledger
→ Returns: { transactionId, status: 'PENDING' }

GET /api/transactions/:id
→ Queries current status across all layers
→ Returns: { status: 'PROCESSING', layer: 'CLEARING', eta: '48s' }

// 2. Clearing Layer API (Internal)
POST /api/clearing/process-batch
→ Triggered by cron job every 10s
→ Processes pending transactions
→ Returns: { batchId, validCount, invalidCount }

// 3. Settlement Layer API (Internal)
POST /api/settlement/finalize-block
→ Triggered by cron job every 60s
→ Creates immutable block
→ Returns: { blockNumber, hash, txCount }

// 4. Sync Status API (User-facing)
GET /api/sync/status/:userId
→ Shows user's transaction status across all layers
→ Returns: {
  pending: 2,      // In Application
  processing: 5,   // In Clearing
  completed: 143   // In Settlement
}
\`\`\`

---

## IV. KẾT NỐI GIỮA CÁC LỚP

### 🔗 Sơ đồ Kết nối

\`\`\`
┌─────────────────────────────────────────────────────────┐
│                    USER REQUEST                          │
│               (POST /api/payment)                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │  Application Service   │
        │  createTransaction()   │
        └────────┬───────────────┘
                 │
                 ├─> Redis: Store temp data
                 ├─> RabbitMQ: Push to queue
                 └─> WebSocket: Notify user
                 
                 ▼ (Queue Consumer)
        ┌────────────────────────┐
        │   Clearing Service     │
        │   processBatch()       │
        └────────┬───────────────┘
                 │
                 ├─> PostgreSQL: Save batch
                 ├─> Risk Engine: Check fraud
                 └─> RabbitMQ: Push to settlement queue
                 
                 ▼ (Queue Consumer)
        ┌────────────────────────┐
        │  Settlement Service    │
        │  finalizeBlock()       │
        └────────┬───────────────┘
                 │
                 ├─> PostgreSQL: Save block (immutable)
                 ├─> S3: Backup block data
                 ├─> ISO 20022: Generate XML
                 └─> WebSocket: Notify completion
                 
                 ▼
        ┌────────────────────────┐
        │   User Wallet Update   │
        │   (Final Balance)      │
        └────────────────────────┘
\`\`\`

### 🛠️ Technologies Stack

\`\`\`yaml
Application Ledger:
  - Storage: Redis (in-memory, fast)
  - Queue: RabbitMQ / Bull
  - Real-time: WebSocket
  - Cache TTL: 5 minutes

Clearing Ledger:
  - Storage: PostgreSQL (transactions table)
  - Processing: Node.js workers
  - Batch Interval: 10 seconds
  - Retention: 30 days

Settlement Ledger:
  - Storage: PostgreSQL (blocks table) + S3 Backup
  - Indexing: Elasticsearch (for fast queries)
  - Hash Algorithm: SHA-256
  - Retention: Permanent (immutable)
  - Compliance: ISO 20022 XML export
\`\`\`

---

## V. VÍ DỤ THỰC TẾ: LUỒNG MUA HÀNG

### Scenario: Pioneer mua sản phẩm 500 Gem

**Step 1: Application Ledger (T=0s)**
\`\`\`typescript
// User clicks "Buy Now"
const tx = await createApplicationTransaction({
  userId: 'PIONEER_ABC123',
  type: 'PURCHASE',
  amount: 500,
  metadata: {
    productId: 'PRODUCT_XYZ',
    merchantId: 'MERCHANT_001'
  }
})

// Response to user
console.log(`Transaction ${tx.id} created - Status: PENDING`)
// UI shows: "Đang xử lý giao dịch..."
\`\`\`

**Step 2: Clearing Ledger (T=10s)**
\`\`\`typescript
// Batch processor runs
const batch = await processClearingBatch()

// Validation for this transaction:
const validation = {
  transactionId: 'APP-001',
  isValid: true,
  checks: {
    sufficientBalance: true,    // User has 1000 Gem
    collateralVerified: true,   // Backed by VND deposit
    riskScore: 15,              // Low risk (0-100 scale)
    fraudDetection: false       // No fraud detected
  }
}

// Calculate fees
const gasFee = 500 * 0.01 // = 5 Gem (1% fee)
const netAmount = 500 - 5  // = 495 Gem to merchant

// Batch result
console.log(`Batch ${batch.batchId}: Transaction validated`)
// UI shows: "Đang xác thực..."
\`\`\`

**Step 3: Settlement Ledger (T=60s)**
\`\`\`typescript
// Block creation
const block = await finalizeSettlementBlock(batch)

// Block data:
{
  blockNumber: 12345,
  transactions: [
    {
      id: 'STL-12345-0',
      from: 'PIONEER_ABC123',
      to: 'MERCHANT_001',
      amount: 495,      // Net amount after gas
      gasFee: 5,
      status: 'EXECUTED'
    }
  ],
  currentHash: '0x7f8a3c2b1e9d4a5f6c8e0b3d7a1c5e9f...',
  previousHash: '0x9e1a4b7c3d8f2e6a0c5b9d3f1e7a4c8b...',
  iso20022Message: '<Document xmlns="urn:iso:std:iso:20022:tech:xsd:pacs.008.001.08">...',
  signature: '0x3d5f7a9c1e8b4d6a0f2c7e5b9d3a1f8c...'
}

// Final balance update
await updateWalletBalance('PIONEER_ABC123', -500)
await updateWalletBalance('MERCHANT_001', +495)
await updateWalletBalance('W2_OPS_WALLET', +5) // Gas fee

console.log(`Block ${block.blockNumber} finalized - Transaction COMPLETED`)
// UI shows: "Giao dịch hoàn tất ✓"
\`\`\`

---

## VI. LỢI ÍCH CỦA HỆ THỐNG 3 LỚP

### ✅ Về Hiệu suất
- **Instant Feedback**: User thấy phản hồi ngay lập tức (Layer 1)
- **Batch Processing**: Xử lý hàng nghìn giao dịch cùng lúc (Layer 2)
- **Optimized Storage**: Chỉ lưu dữ liệu cuối cùng vĩnh viễn (Layer 3)

### ✅ Về Bảo mật
- **Triple Validation**: Giao dịch được kiểm tra 3 lần
- **Fraud Detection**: Layer 2 phát hiện giao dịch bất thường
- **Immutable Record**: Layer 3 không thể thay đổi sau khi ghi

### ✅ Về Compliance
- **ISO 20022**: Tuân thủ chuẩn quốc tế
- **Audit Trail**: Dễ dàng kiểm toán
- **Regulatory Reporting**: Xuất báo cáo cho ngân hàng/cơ quan quản lý

### ✅ Về Scalability
- **Horizontal Scaling**: Có thể thêm workers cho mỗi layer
- **Queue-based**: Xử lý bất đồng bộ, không bị nghẽn
- **Microservices Ready**: Mỗi layer là một service độc lập

---

## VII. MONITORING & DEBUGGING

### 📊 Dashboard Metrics

\`\`\`typescript
// Real-time metrics
{
  applicationLayer: {
    pendingTransactions: 234,
    averageResponseTime: '120ms',
    throughput: '500 tx/s'
  },
  clearingLayer: {
    currentBatchSize: 48,
    validationRate: 96.5,
    averageProcessingTime: '8.2s'
  },
  settlementLayer: {
    currentBlockNumber: 12345,
    blocksPerHour: 60,
    averageBlockSize: 152,
    chainIntegrity: 'VERIFIED'
  }
}
\`\`\`

### 🔍 Debugging Commands

\`\`\`bash
# Check transaction status across all layers
curl -X GET /api/debug/transaction/APP-001

# Response:
{
  transactionId: 'APP-001',
  currentLayer: 'SETTLEMENT',
  journey: [
    { layer: 'APPLICATION', timestamp: '2025-02-03T10:00:00Z', status: 'PENDING' },
    { layer: 'CLEARING', timestamp: '2025-02-03T10:00:10Z', status: 'VALIDATED' },
    { layer: 'SETTLEMENT', timestamp: '2025-02-03T10:01:00Z', status: 'EXECUTED' }
  ],
  finalBlock: 12345,
  finalHash: '0x7f8a3c2b...'
}
\`\`\`

---

## VIII. KẾT LUẬN

Hệ thống sổ cái 3 lớp của 314Mall là nền tảng vững chắc cho một hệ sinh thái tài chính phi tập trung, kết hợp:

✓ **Tốc độ** của ứng dụng hiện đại (Layer 1)  
✓ **Độ tin cậy** của hệ thống ngân hàng (Layer 2)  
✓ **Tính bất biến** của blockchain (Layer 3)  
✓ **Tuân thủ** chuẩn quốc tế ISO 20022  

Mỗi layer đóng vai trò riêng nhưng kết nối chặt chẽ, tạo nên một hệ thống hoàn chỉnh, sẵn sàng scale và đáp ứng yêu cầu của hàng triệu người dùng.

---

**Tài liệu này được tạo bởi v0 cho dự án 314Mall**  
*Cập nhật lần cuối: 2025-02-03*
