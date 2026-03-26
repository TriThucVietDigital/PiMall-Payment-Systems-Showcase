# CƠ CHẾ VÀ LOGIC GEM TOKEN - 314MALL

## TỔNG QUAN HỆ THỐNG

**Tổng cung Gem:** 100 tỷ GEM (100,000,000,000 GEM)
**Tỷ lệ:** 1 GEM = 1,000 VNĐ (giá trị đồng thuận nội bộ)
**Bảo chứng:** 1:1 backing với VNĐ/XRP

---

## CẤU TRÚC GEM BALANCE CỦA USER

### 1. **GEM BALANCE TOTAL** (Tổng số dư Gem)
\`\`\`
Gem Balance Total = Gem Available + Gem Locked
\`\`\`

**Hiển thị:** Tab selector phía trên wallet page
- Gem khả dụng: 10,780.50 GEM
- Gem đang khóa: 5,000 GEM
- **Tổng: 15,780.50 GEM**

---

### 2. **GEM AVAILABLE** (Gem khả dụng)
\`\`\`
Gem Available = Total - (Store Deposit + Trade Collateral + Rental Locked + P2P Lending)
\`\`\`

**Công thức chi tiết:**
- User mint được: 15,780.50 GEM từ VNĐ/XRP
- Trừ Store Registration Deposit: -5,000 GEM
- Trừ Trade Collateral (mua bán): -0 GEM (chưa dùng)
- Trừ Rental Locked (cho thuê): -0 GEM (tính năng tương lai)
- Trừ P2P Lending: -0 GEM (tính năng tương lai)
- **= 10,780.50 GEM khả dụng**

**Hiển thị:** Main balance card (số to, màu trắng)
**Mục đích:** 
- Có thể dùng ngay để đặt cọc thêm
- Có thể bán đổi ra VNĐ/XRP
- Có thể chuyển cho user khác
- Có thể dùng để lock cho các mục đích khác

---

### 3. **GEM LOCKED** (Gem đang khóa)

Chia thành 4 loại lock:

#### 3.1. Store Registration Deposit (Cọc mở cửa hàng)
\`\`\`sql
deposit_contracts:
  - deposit_purpose: 'store_registration'
  - deposit_amount_gem: 5,000 GEM
  - status: 'active'
\`\`\`

**Logic:**
- Merchant muốn mở cửa hàng → Phải lock 5,000 GEM
- Gem này bị khóa cho đến khi:
  - Merchant đóng cửa hàng (được hoàn lại)
  - Vi phạm điều khoản (bị tịch thu)
- **Hiển thị:** Card riêng trên lịch sử giao dịch với badge "Đang khóa"

**Mục đích:**
- Đảm bảo merchant có "da trong cuộc"
- Chống spam, scam
- Tạo niềm tin cho buyer

---

#### 3.2. Trade Collateral (Cọc mua bán - Tính năng hiện tại)
\`\`\`sql
deposit_contracts:
  - deposit_purpose: 'trade_collateral'
  - deposit_amount_gem: X GEM
  - status: 'active'
  - linked_order_id: order_id
\`\`\`

**Logic:**
- Buyer/Seller có thể yêu cầu cọc cho giao dịch lớn
- VD: Mua điện thoại 50M VNĐ → Yêu cầu cọc 10,000 GEM
- Gem lock cho đến khi:
  - Giao dịch hoàn tất → Hoàn lại cọc
  - Tranh chấp → Admin xử lý
- **Hiển thị:** Trong transaction history với tag "Cọc giao dịch"

**Mục đích:**
- Bảo vệ cả buyer và seller
- Giảm rủi ro giao dịch giá trị cao
- Tạo hệ thống ký quỹ tự động

---

#### 3.3. Rental Locked (Cho thuê mặt bằng - Tương lai)
\`\`\`sql
rental_payments:
  - rental_period: 'month' / 'year'
  - amount_gem: X GEM
  - payment_status: 'active'
\`\`\`

**Logic (Dự kiến):**
- Platform sẽ có "Khu thương mại ảo" với vị trí prime
- Merchant thuê vị trí tốt → Trả bằng Gem hàng tháng
- VD: Vị trí trang chủ = 2,000 GEM/tháng
- Gem lock trước 30 ngày, tự động gia hạn hoặc release

**Hiển thị:** Tab riêng "Cho thuê mặt bằng" trong wallet
**Mục đích:**
- Tạo thu nhập cho platform
- Merchant cạnh tranh vị trí tốt
- Gem có utility thực tế

---

#### 3.4. P2P Lending (Vay/Cho vay P2P - Tương lai)
\`\`\`sql
lending_contracts:
  - lender_user_id
  - borrower_user_id  
  - collateral_gem_amount: X GEM
  - loan_amount_vnd: Y VNĐ
  - interest_rate: Z%
  - locked_until: timestamp
\`\`\`

**Logic (Dự kiến):**
- User có nhiều Gem → Cho vay P2P để kiếm lãi
- Borrower cần VNĐ → Cầm Gem để vay
- Platform tự động:
  - Lock Gem collateral
  - Transfer VNĐ cho borrower
  - Tính lãi tự động
  - Release Gem khi trả nợ đủ
  - Liquidate Gem nếu không trả nợ

**Đối tác dự kiến:**
- Ngân hàng: Cung cấp VNĐ flow
- Công ty tài chính: Đánh giá tín dụng
- Sàn: Liquidate Gem khi cần

**Hiển thị:** App/trang web riêng "314 Finance"

---

## DATABASE TRACKING

### Bảng `wallets`
\`\`\`sql
wallets:
  - wallet_id
  - user_id
  - wallet_type: 'application' / 'clearing' / 'settlement'
  - currency: 'GEM'
  - balance: 15780.50 (total balance)
\`\`\`

### Bảng `deposit_contracts`
\`\`\`sql
deposit_contracts:
  - contract_id
  - merchant_user_id
  - deposit_amount_gem: 5000
  - deposit_purpose: 'store_registration' / 'trade_collateral'
  - status: 'active' / 'released' / 'forfeited'
  - locked_until: timestamp
  - created_at
\`\`\`

### Function tính Gem Available
\`\`\`typescript
async function getAvailableGemBalance(userId: string): Promise<number> {
  // Get total Gem balance
  const totalBalance = await getWalletBalance(userId, 'GEM')
  
  // Get all locked Gem
  const { data: lockedContracts } = await supabase
    .from('deposit_contracts')
    .select('deposit_amount_gem')
    .eq('merchant_user_id', userId)
    .eq('status', 'active')
  
  const totalLocked = lockedContracts?.reduce(
    (sum, contract) => sum + contract.deposit_amount_gem, 
    0
  ) || 0
  
  return totalBalance - totalLocked
}
\`\`\`

---

## HIỂN THỊ TRONG UI

### Wallet Page - Tab Gem

**1. Gem Balance (Tab selector)**
\`\`\`
Gem Balance
15,780.50 GEM
≈ 15,780,500 VNĐ
\`\`\`

**2. Main Card - Available**
\`\`\`
Số dư Gem (Khả dụng)
10,780.50 GEM
≈ 10,780,500 VNĐ
\`\`\`

**3. Store Registration Deposit Card**
\`\`\`
[Gem Icon] Store Registration Deposit
           Cọc mở cửa hàng
                             -5,000 GEM  [Đang khóa]
\`\`\`

**4. Transaction History**
- Nạp Gem: +10,000 GEM (từ VNĐ)
- Cọc cửa hàng: -5,000 GEM (khóa)
- Mua Gem: +5,780.50 GEM (từ XRP)
- etc.

---

## FLOW HOẠT ĐỘNG

### Flow 1: Mint Gem
\`\`\`
User deposit 15,780,500 VNĐ
→ Verify payment
→ Mint 15,780.50 GEM (1:1 ratio)
→ Create collateral_reserves record
→ Update wallet balance: +15,780.50 GEM
→ Gem Available = 15,780.50 (chưa lock gì)
\`\`\`

### Flow 2: Store Registration
\`\`\`
Merchant click "Đăng ký cửa hàng"
→ Check Gem Available >= 5,000
→ Create deposit_contract (purpose: store_registration, amount: 5000)
→ Gem Available = 10,780.50 (15,780.50 - 5,000)
→ Gem Locked = 5,000
→ Status: 'active'
\`\`\`

### Flow 3: Sell Gem
\`\`\`
User click "Bán Gem"
→ Input: 10,000 GEM
→ Check Available >= 10,000
→ Burn 10,000 GEM
→ Transfer 10,000,000 VNĐ to user bank
→ Update collateral_reserves (reduce backing)
→ Gem Available = 780.50
\`\`\`

---

## BẢO MẬT & KIỂM SOÁT

### 1. Tổng cung 100 tỷ Gem
\`\`\`typescript
const MAX_GEM_SUPPLY = 100_000_000_000 // 100 billion

async function validateMint(amountGem: number): Promise<boolean> {
  const currentSupply = await getTotalGemSupply()
  
  if (currentSupply + amountGem > MAX_GEM_SUPPLY) {
    throw new Error('Exceeded max Gem supply')
  }
  
  return true
}
\`\`\`

### 2. 1:1 Backing Validation
\`\`\`typescript
async function validate1to1Backing(): Promise<boolean> {
  const totalGemMinted = await getTotalGemSupply()
  const totalCollateral = await getTotalCollateralValue() // VNĐ + XRP value
  
  // Must be 1:1 or over-collateralized
  const ratio = totalCollateral / (totalGemMinted * 1000) // 1 GEM = 1000 VNĐ
  
  if (ratio < 1) {
    throw new Error('Under-collateralized! Cannot mint more Gem')
  }
  
  return true
}
\`\`\`

### 3. Lock/Unlock Audit Trail
\`\`\`typescript
// Every lock/unlock is logged immutably
audit_logs:
  - action: 'GEM_LOCKED'
  - user_id
  - amount: 5000
  - purpose: 'store_registration'
  - previous_hash
  - current_hash (SHA-256 chain)
\`\`\`

---

## TÓM TẮT

| Khái niệm | Giá trị hiện tại | Mục đích |
|-----------|------------------|----------|
| **Gem Balance Total** | 15,780.50 GEM | Tổng số Gem user sở hữu |
| **Gem Available** | 10,780.50 GEM | Có thể dùng ngay (bán, cọc thêm) |
| **Store Deposit** | 5,000 GEM | Cọc mở cửa hàng (đang khóa) |
| **Trade Collateral** | 0 GEM | Cọc giao dịch (tính năng hiện tại) |
| **Rental Locked** | 0 GEM | Thuê mặt bằng (tương lai) |
| **P2P Lending** | 0 GEM | Vay/cho vay (tương lai) |

**Logic chính:**
- Store Registration Deposit CHỈ áp dụng cho merchant mở cửa hàng
- Trade Collateral là tính năng riêng cho giao dịch mua bán có yêu cầu cọc
- Rental và P2P Lending là tính năng tương lai, trang riêng
- Tất cả đều tracking trong deposit_contracts và rental_payments tables
- UI hiển thị rõ ràng Available vs Locked cho user dễ hiểu

**Gem không dùng để:**
- Mua sắm hàng ngày (dùng Pi)
- Thanh toán sản phẩm thông thường (dùng Pi)

**Gem CHỈ dùng để:**
- Cọc mở cửa hàng
- Cọc giao dịch lớn (optional)
- Thuê mặt bằng prime (tương lai)
- Vay/cho vay P2P (tương lai)
- Store of value (giữ giá trị)
