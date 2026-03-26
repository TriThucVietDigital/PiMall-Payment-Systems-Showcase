# DATABASE SCHEMA - ShopPIGCVMarket

## Tổng quan
Cơ sở dữ liệu ShopPIGCVMarket được thiết kế với PostgreSQL/MySQL, đảm bảo mã hóa và bảo mật thông tin.

---

## 1. Bảng USERS (Người dùng)

\`\`\`sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pi_username VARCHAR(100) UNIQUE NOT NULL,
  pi_uid VARCHAR(255) UNIQUE,
  full_name VARCHAR(255),
  phone_number_encrypted TEXT, -- Mã hóa AES-256
  email VARCHAR(255),
  role ENUM('buyer', 'seller', 'admin') DEFAULT 'buyer',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  is_active BOOLEAN DEFAULT true,
  
  -- Index
  INDEX idx_pi_username (pi_username),
  INDEX idx_role (role)
);
\`\`\`

**Dữ liệu nhạy cảm cần mã hóa:**
- `phone_number_encrypted`: Số điện thoại (AES-256)

---

## 2. Bảng SELLERS (Người bán)

\`\`\`sql
CREATE TABLE sellers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  shop_name VARCHAR(255) NOT NULL,
  pickup_address_encrypted TEXT, -- Mã hóa địa chỉ lấy hàng
  pickup_latitude DECIMAL(10, 8),
  pickup_longitude DECIMAL(11, 8),
  bank_account_encrypted TEXT, -- Mã hóa số tài khoản
  bank_name VARCHAR(255),
  
  -- GCV Payment Settings
  pi_percentage INT DEFAULT 60, -- 60-100%
  fiat_percentage INT DEFAULT 40, -- 0-40%
  
  verified BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Index
  INDEX idx_user_id (user_id),
  INDEX idx_verified (verified)
);
\`\`\`

**Dữ liệu nhạy cảm cần mã hóa:**
- `pickup_address_encrypted`: Địa chỉ lấy hàng (AES-256)
- `bank_account_encrypted`: Số tài khoản ngân hàng (AES-256)

---

## 3. Bảng PRODUCTS (Sản phẩm)

\`\`\`sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id UUID REFERENCES sellers(id) ON DELETE CASCADE,
  name VARCHAR(500) NOT NULL,
  description TEXT,
  category VARCHAR(100),
  
  -- Pricing (GCV Standard: 1 Pi = 314,159 USD)
  price_vnd DECIMAL(15, 2), -- Giá gốc VNĐ
  price_pi DECIMAL(20, 8), -- Giá quy đổi Pi (8 chữ số thập phân)
  price_usd DECIMAL(15, 2), -- Giá quy đổi USD
  
  image_url TEXT,
  stock_quantity INT DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Index
  INDEX idx_seller_id (seller_id),
  INDEX idx_category (category),
  INDEX idx_is_active (is_active)
);
\`\`\`

---

## 4. Bảng ORDERS (Đơn hàng)

\`\`\`sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number VARCHAR(50) UNIQUE NOT NULL, -- Format: PI12345
  
  -- User info
  buyer_id UUID REFERENCES users(id),
  seller_id UUID REFERENCES sellers(id),
  
  -- Shipping info (encrypted)
  shipping_name VARCHAR(255),
  shipping_phone_encrypted TEXT,
  shipping_address_encrypted TEXT,
  
  -- Payment info
  total_amount_pi DECIMAL(20, 8),
  total_amount_vnd DECIMAL(15, 2),
  total_amount_usd DECIMAL(15, 2),
  
  -- Payment split (GCV)
  payment_pi_amount DECIMAL(20, 8), -- Số Pi thực tế thanh toán
  payment_fiat_amount DECIMAL(15, 2), -- Số tiền mặt (VNĐ)
  payment_pi_percentage INT, -- % thanh toán bằng Pi
  
  -- Pi Network Transaction
  pi_payment_id VARCHAR(255),
  pi_transaction_id VARCHAR(255),
  
  status ENUM('pending', 'paid', 'shipping', 'completed', 'cancelled') DEFAULT 'pending',
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Index
  INDEX idx_order_number (order_number),
  INDEX idx_buyer_id (buyer_id),
  INDEX idx_seller_id (seller_id),
  INDEX idx_status (status),
  INDEX idx_pi_payment_id (pi_payment_id)
);
\`\`\`

**Dữ liệu nhạy cảm cần mã hóa:**
- `shipping_phone_encrypted`: SĐT người nhận (AES-256)
- `shipping_address_encrypted`: Địa chỉ giao hàng (AES-256)

---

## 5. Bảng ORDER_ITEMS (Chi tiết đơn hàng)

\`\`\`sql
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID REFERENCES orders(id) ON DELETE CASCADE,
  product_id UUID REFERENCES products(id),
  
  product_name VARCHAR(500),
  quantity INT NOT NULL,
  unit_price_pi DECIMAL(20, 8),
  unit_price_vnd DECIMAL(15, 2),
  total_price_pi DECIMAL(20, 8),
  total_price_vnd DECIMAL(15, 2),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Index
  INDEX idx_order_id (order_id),
  INDEX idx_product_id (product_id)
);
\`\`\`

---

## 6. Bảng PI_PAYMENTS (Lịch sử thanh toán Pi)

\`\`\`sql
CREATE TABLE pi_payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID REFERENCES orders(id),
  
  pi_payment_id VARCHAR(255) UNIQUE,
  pi_transaction_id VARCHAR(255),
  
  amount_pi DECIMAL(20, 8),
  memo TEXT,
  
  status ENUM('initiated', 'pending', 'approved', 'completed', 'cancelled') DEFAULT 'initiated',
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  approved_at TIMESTAMP,
  completed_at TIMESTAMP,
  
  -- Index
  INDEX idx_pi_payment_id (pi_payment_id),
  INDEX idx_order_id (order_id),
  INDEX idx_status (status)
);
\`\`\`

---

## MÃ HÓA DỮ LIỆU

### Phương pháp: AES-256-GCM

\`\`\`javascript
// Encryption Key (Bạn PHẢI tự tạo và lưu trữ an toàn)
const ENCRYPTION_KEY = process.env.DB_ENCRYPTION_KEY // 32 bytes

// Hàm mã hóa
function encrypt(text) {
  const iv = crypto.randomBytes(16)
  const cipher = crypto.createCipheriv('aes-256-gcm', ENCRYPTION_KEY, iv)
  const encrypted = Buffer.concat([cipher.update(text, 'utf8'), cipher.final()])
  const authTag = cipher.getAuthTag()
  return iv.toString('hex') + ':' + authTag.toString('hex') + ':' + encrypted.toString('hex')
}

// Hàm giải mã
function decrypt(encryptedData) {
  const parts = encryptedData.split(':')
  const iv = Buffer.from(parts[0], 'hex')
  const authTag = Buffer.from(parts[1], 'hex')
  const encrypted = Buffer.from(parts[2], 'hex')
  const decipher = crypto.createDecipheriv('aes-256-gcm', ENCRYPTION_KEY, iv)
  decipher.setAuthTag(authTag)
  return decipher.update(encrypted) + decipher.final('utf8')
}
\`\`\`

---

## QUYỀN QUẢN TRỊ

### Super Admin Account (Chủ sở hữu)

\`\`\`sql
-- Tạo tài khoản admin cho chủ sở hữu
INSERT INTO users (pi_username, full_name, role) 
VALUES ('owner_admin', 'Chủ sở hữu ShopPIGCVMarket', 'admin');
\`\`\`

### Database User Privileges

\`\`\`sql
-- Tạo database user cho chủ sở hữu với full privileges
CREATE USER 'shoppi_owner'@'%' IDENTIFIED BY 'STRONG_PASSWORD_HERE';
GRANT ALL PRIVILEGES ON shoppigcvmarket.* TO 'shoppi_owner'@'%';
FLUSH PRIVILEGES;
\`\`\`

---

## GHI CHÚ BẢO MẬT

1. **Encryption Key**: Lưu trong biến môi trường `DB_ENCRYPTION_KEY`, KHÔNG commit vào Git
2. **Database Password**: Phải là mật khẩu mạnh (>16 ký tự, chữ + số + ký tự đặc biệt)
3. **Backup Encryption**: Tất cả backup file phải được mã hóa trước khi lưu trữ
4. **Access Control**: Chỉ IP của Raspberry Pi và server được phép truy cập database

---

**Lưu ý:** Schema này tuân thủ chuẩn GCV (1π = 314,159 USD) và đảm bảo mã hóa toàn bộ thông tin nhạy cảm.
