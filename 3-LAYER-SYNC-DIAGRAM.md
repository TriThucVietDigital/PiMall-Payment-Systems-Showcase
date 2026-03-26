# SƠ ĐỒ ĐỒNG BỘ DỮ LIỆU 3-LAYER LEDGER

## Luồng dữ liệu chi tiết theo thời gian thực

\`\`\`
TIMELINE: 0s ────────────── 10s ────────────── 60s ────────────── 61s

┌──────────────────────────────────────────────────────────────────────────┐
│                         USER INTERFACE LAYER                              │
│                                                                           │
│  T=0s: "Mua hàng"  →  T=0.5s: "Đang xử lý"  →  T=61s: "Hoàn tất ✓"     │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ T=0s - T=10s: APPLICATION LEDGER (Sổ Ứng dụng)                          ┃
┃                                                                           ┃
┃  📝 Ghi nhận tức thì:                                                     ┃
┃  ┌────────────────────────────────────────────────────────────┐         ┃
┃  │ Transaction: APP-20250203-001                              │         ┃
┃  │ Status: PENDING                                            │         ┃
┃  │ User: PIONEER_ABC                                          │         ┃
┃  │ Amount: 500 Gem                                            │         ┃
┃  │ Created: 2025-02-03 10:00:00.000                          │         ┃
┃  └────────────────────────────────────────────────────────────┘         ┃
┃                                                                           ┃
┃  💾 Storage: Redis (In-Memory)                                           ┃
┃  ⚡ Response Time: 50-200ms                                              ┃
┃  🔄 Auto-push to Queue: RabbitMQ                                         ┃
┃                                                                           ┃
┃  Status Updates:                                                          ┃
┃  • T=0.0s: PENDING (Created)                                             ┃
┃  • T=0.5s: PROCESSING (In Queue)                                         ┃
┃  • T=2.0s: QUEUED (Waiting for batch)                                    ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                    │
                                    │ Queue Consumer (Batch every 10s)
                                    ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ T=10s - T=60s: CLEARING LEDGER (Sổ Thanh toán)                          ┃
┃                                                                           ┃
┃  🔍 Batch Processing:                                                     ┃
┃  ┌────────────────────────────────────────────────────────────┐         ┃
┃  │ Batch: CLR-20250203-001                                    │         ┃
┃  │ Transactions: 48 items                                     │         ┃
┃  │ Status: PROCESSING → VALIDATED                             │         ┃
┃  │                                                            │         ┃
┃  │ Validation Pipeline:                                       │         ┃
┃  │  ✓ Balance Check     (48/48 passed)                       │         ┃
┃  │  ✓ Collateral Verify (48/48 passed)                       │         ┃
┃  │  ✓ Risk Scoring      (Avg: 15/100)                        │         ┃
┃  │  ✓ Fraud Detection   (0 flagged)                          │         ┃
┃  │  ✓ Gas Fee Calc      (Total: 24 Gem)                      │         ┃
┃  │                                                            │         ┃
┃  │ Net Settlement: 2,376 Gem                                  │         ┃
┃  │ Processed: 2025-02-03 10:00:12.000                        │         ┃
┃  └────────────────────────────────────────────────────────────┘         ┃
┃                                                                           ┃
┃  💾 Storage: PostgreSQL (transactions table)                             ┃
┃  ⚙️ Processing Time: 8-12 seconds                                        ┃
┃  📊 Validation Rate: 96.5%                                               ┃
┃                                                                           ┃
┃  Status Updates:                                                          ┃
┃  • T=10s: PROCESSING (Validation started)                                ┃
┃  • T=12s: VALIDATED (All checks passed)                                  ┃
┃  • T=15s: QUEUED_FOR_SETTLEMENT                                          ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                    │
                                    │ Queue Consumer (Finalize every 60s)
                                    ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ T=60s+: SETTLEMENT LEDGER (Sổ Quyết toán - IMMUTABLE)                   ┃
┃                                                                           ┃
┃  🔒 Block Finalization:                                                   ┃
┃  ┌────────────────────────────────────────────────────────────┐         ┃
┃  │ Block #12345                                               │         ┃
┃  │ Timestamp: 2025-02-03 10:01:00.000                        │         ┃
┃  │                                                            │         ┃
┃  │ Previous Hash:                                             │         ┃
┃  │  0x9e1a4b7c3d8f2e6a0c5b9d3f1e7a4c8b...                   │         ┃
┃  │                                                            │         ┃
┃  │ Current Hash:                                              │         ┃
┃  │  0x7f8a3c2b1e9d4a5f6c8e0b3d7a1c5e9f...                   │         ┃
┃  │                                                            │         ┃
┃  │ Merkle Root:                                               │         ┃
┃  │  0x4c2a8f1b9e3d7a5c0f6b8e2d9a3f1c7e...                   │         ┃
┃  │                                                            │         ┃
┃  │ Transactions: 48                                           │         ┃
┃  │ Total Amount: 24,000 Gem                                   │         ┃
┃  │ Total Gas: 240 Gem                                         │         ┃
┃  │                                                            │         ┃
┃  │ ISO 20022 Message: pacs.008.001.08                        │         ┃
┃  │ Signature: 0x3d5f7a9c1e8b4d6a0f2c7e5b...                  │         ┃
┃  │ Validator: ...7U24B (Master Wallet)                       │         ┃
┃  └────────────────────────────────────────────────────────────┘         ┃
┃                                                                           ┃
┃  💾 Storage: PostgreSQL + S3 Backup                                      ┃
┃  🔐 Hash Algorithm: SHA-256                                              ┃
┃  📜 Compliance: ISO 20022 XML                                            ┃
┃  ♾️ Retention: PERMANENT (Immutable)                                     ┃
┃                                                                           ┃
┃  Final Status:                                                            ┃
┃  • T=60s: FINALIZING (Block creation)                                    ┃
┃  • T=61s: EXECUTED (Immutable record)                                    ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                    │
                                    │ Notify User & Update Balance
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         WALLET BALANCE UPDATE                             │
│                                                                           │
│  Pioneer ABC: 1000 Gem → 500 Gem (Final)                                │
│  Merchant 001: 2000 Gem → 2495 Gem (Final)                              │
│  Ops Wallet (W2): +5 Gem (Gas Fee)                                       │
└──────────────────────────────────────────────────────────────────────────┘
\`\`\`

---

## Chi tiết kỹ thuật từng giai đoạn

### Stage 1: Application Layer (0-10s)

\`\`\`javascript
// API Call
POST /api/transactions/create
{
  "userId": "PIONEER_ABC",
  "amount": 500,
  "type": "PURCHASE",
  "productId": "PRODUCT_XYZ"
}

// Immediate Response (50-200ms)
{
  "transactionId": "APP-20250203-001",
  "status": "PENDING",
  "estimatedTime": "60 seconds",
  "tracking": {
    "currentLayer": "APPLICATION",
    "nextLayer": "CLEARING",
    "eta": "10 seconds"
  }
}

// Internal Process
1. Save to Redis: KEY=app:tx:APP-20250203-001
2. Publish to Queue: rabbitmq://clearing-queue
3. WebSocket notify: ws://user-notifications
4. Update UI: "Processing transaction..."
\`\`\`

### Stage 2: Clearing Layer (10-60s)

\`\`\`javascript
// Batch Processor (runs every 10s)
setInterval(async () => {
  // 1. Collect transactions
  const transactions = await queue.getJobs('waiting', 0, 100)
  
  // 2. Create batch
  const batch = {
    id: `CLR-${Date.now()}`,
    transactions: transactions,
    startedAt: new Date()
  }
  
  // 3. Validate each transaction
  for (const tx of transactions) {
    const result = await validate(tx)
    if (result.valid) {
      await db.validTransactions.insert(tx)
    } else {
      await notify(tx.userId, 'Transaction failed', result.errors)
    }
  }
  
  // 4. Calculate net settlement
  const totalAmount = sumOf(transactions, 'amount')
  const gasFee = totalAmount * 0.01
  const netSettlement = totalAmount - gasFee
  
  // 5. Save batch
  await db.clearingBatches.insert({
    ...batch,
    totalAmount,
    gasFee,
    netSettlement,
    processedAt: new Date()
  })
  
  // 6. Push to settlement queue
  await settlementQueue.add('finalize', batch)
  
}, 10000) // Every 10 seconds
\`\`\`

### Stage 3: Settlement Layer (60s+)

\`\`\`javascript
// Block Finalizer (runs every 60s)
setInterval(async () => {
  // 1. Get validated batches
  const batches = await settlementQueue.getJobs('waiting')
  
  // 2. Create settlement block
  const previousBlock = await db.blocks.findLast()
  const blockNumber = previousBlock.number + 1
  
  const block = {
    number: blockNumber,
    timestamp: new Date(),
    previousHash: previousBlock.hash,
    transactions: batches.flatMap(b => b.transactions),
    merkleRoot: calculateMerkleRoot(transactions),
    iso20022Message: generatePacs008(batches),
    validator: MASTER_WALLET
  }
  
  // 3. Generate hash
  block.hash = SHA256(
    block.number + 
    block.previousHash + 
    block.merkleRoot + 
    block.timestamp
  )
  
  // 4. Sign block
  block.signature = signWithPrivateKey(block.hash, MASTER_KEY)
  
  // 5. Save permanently
  await db.blocks.insert(block)
  await s3.upload(`blocks/${blockNumber}.json`, block)
  
  // 6. Update all transaction statuses
  for (const tx of block.transactions) {
    await redis.set(`app:tx:${tx.id}:status`, 'EXECUTED')
    await websocket.emit(`tx:${tx.id}`, { status: 'COMPLETED' })
  }
  
  // 7. Update user balances (FINAL)
  await updateBalances(block.transactions)
  
}, 60000) // Every 60 seconds
\`\`\`

---

## Monitoring Dashboard

\`\`\`
┌─────────────────────────────────────────────────────────────┐
│                    SYSTEM HEALTH MONITOR                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  APPLICATION LAYER                                           │
│  • Pending Transactions: 234                                 │
│  • Response Time: 120ms (avg)                                │
│  • Throughput: 500 tx/s                                      │
│  • Queue Size: 1,024                                         │
│  • Status: ✓ Healthy                                         │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CLEARING LAYER                                              │
│  • Current Batch: CLR-20250203-042                           │
│  • Batch Size: 48 transactions                               │
│  • Validation Rate: 96.5%                                    │
│  • Processing Time: 8.2s (avg)                               │
│  • Failed Transactions: 2 (fraud detected)                   │
│  • Status: ✓ Healthy                                         │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SETTLEMENT LAYER                                            │
│  • Current Block: #12,345                                    │
│  • Blocks/Hour: 60                                           │
│  • Avg Block Size: 152 transactions                          │
│  • Last Block Hash: 0x7f8a3c2b...                            │
│  • Chain Integrity: ✓ VERIFIED                               │
│  • ISO 20022 Export: ✓ Ready                                 │
│  • Status: ✓ Healthy                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
\`\`\`

---

## Error Handling & Recovery

\`\`\`
Scenario 1: Clearing Validation Failed
┌──────────────────────────────────────┐
│ Transaction: APP-001                  │
│ Issue: Insufficient balance          │
│                                      │
│ Recovery Flow:                       │
│ 1. Clearing marks as INVALID         │
│ 2. Notify Application Layer          │
│ 3. Update status: PENDING → FAILED   │
│ 4. Refund temp deduction             │
│ 5. WebSocket notify user             │
│ 6. UI shows: "Transaction failed"    │
└──────────────────────────────────────┘

Scenario 2: Settlement Block Generation Failed
┌──────────────────────────────────────┐
│ Block: #12345                         │
│ Issue: Hash collision detected       │
│                                      │
│ Recovery Flow:                       │
│ 1. Abort block creation              │
│ 2. Increment nonce                   │
│ 3. Regenerate hash                   │
│ 4. Retry finalization                │
│ 5. Alert system admin                │
│ 6. Log incident for audit            │
└──────────────────────────────────────┘

Scenario 3: Network Partition (Layer disconnected)
┌──────────────────────────────────────┐
│ Issue: Clearing Layer offline        │
│                                      │
│ Recovery Flow:                       │
│ 1. Application continues queuing     │
│ 2. Queue size grows (monitored)      │
│ 3. Alert: "Clearing delayed"         │
│ 4. When reconnects:                  │
│    - Process all queued txs          │
│    - Generate catch-up batches       │
│    - Resume normal operation         │
└──────────────────────────────────────┘
\`\`\`

---

**Tài liệu kỹ thuật này cung cấp cái nhìn chi tiết về cách các lớp sổ cái đồng bộ và kết nối với nhau trong hệ thống 314Mall.**
