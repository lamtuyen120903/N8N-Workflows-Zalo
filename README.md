# N8N Zalo Workflows

Bộ sưu tập các workflow n8n tự động hóa Zalo cho dự án OTIMO/Cẩm Nang Gối Lò.

## Tổng Quan

Các workflow này xử lý:
- Gửi tin nhắn tự động đến nhóm Zalo
- Quản lý OA (Official Account) Zalo
- Chatbot AI tự động trả lời
- Đồng bộ dữ liệu với Google Sheets & Larkbase
- Tự động kết bạn và chấp nhận lời mời

## Cấu Trúc Thư Mục

```
N8N_Zalo/
├── Zalo - OA - Tự động lấy Access Token.json    # Làm mới token mỗi 24h
├── Zalo - OA - Webhook.json                    # Xử lý sự kiện follow/unfollow
├── Zalo - OA - Lấy thông tin khách hàng.json   # Sync danh sách khách hàng
├── Zalo - OA - Update thông tin người dùng.json # Cập nhật thông tin định kỳ
├── Zalo - OA -  Gửi tin nhắn.json               # Gửi tin nhắn OA
├── Zalo - Gửi Tin Nhắn Tự Động Cho Group CNGL Tài Liệu.json  # Auto post nhóm
├── Zalo - Gửi Tin Nhắn Video cho Group CNGL.json # Auto post video nhóm
├── Zalo - Bot Notifications.json                # Chatbot AI
├── Zalo - Tự Động Add & Accept Bạn Bè.json     # Auto friend request
└── Zalo - Larkbase - Lấy Thông Tin Khách Hàng.json # Sync Larkbase
```

---

## Chi Tiết Workflow

### 1. Zalo - OA - Tự động lấy Access Token
**File:** `Zalo - OA - Tự động lấy Access Token.json`

**Mục đích:** Tự động làm mới access token Zalo OA mỗi 24 giờ.

**Luồng hoạt động:**
```
Schedule Trigger (24h)
    ↓
Get row(s) từ DataTable "Zalo - Access"
    ↓
Limit (lấy item mới nhất)
    ↓
get_access_token_zns (refresh token qua API)
    ↓
Upsert row(s) (lưu token mới)
```

**Điều kiện active:** ✅ Đang bật

---

### 2. Zalo - OA - Webhook
**File:** `Zalo - OA - Webhook.json`

**Mục đích:** Xử lý sự kiện từ Zalo OA (follow/unfollow/tin nhắn).

**Luồng hoạt động:**
```
Webhook nhận sự kiện
    ↓
Switch (phân loại theo event_name)
    ├── unfollow → Update row (follow=FALSE)
    ├── follow → Get access_token → Get User Detail → Kiểm tra tồn tại
    │                                     ↓
    │                          If (chưa có) → Append row
    │                          If (đã có) → Update row
    └── Nhắn tin → (route khác)
```

**Webhook ID:** `67db90d6-d72c-462a-87b4-5e8df6596430`

**Lưu trữ:** Google Sheets "Khách hàng Zalo OA"

---

### 3. Zalo - OA - Lấy thông tin khách hàng
**File:** `Zalo - OA - Lấy thông tin khách hàng.json`

**Mục đích:** Lấy toàn bộ danh sách khách hàng từ Zalo OA (có pagination).

**Luồng hoạt động:**
```
Manual Trigger
    ↓
Get access_token
    ↓
Get First Page (lấy total count)
    ↓
Generate Offsets (tính offset: 0, 50, 100, ...)
    ↓
Loop Over Offsets
    ↓
Get Users Page (mỗi page 50 users)
    ↓
Split Users
    ↓
Get User Detail (lấy chi tiết từng user)
    ↓
Code (filter null/error)
    ↓
Append row in sheet
```

**Giới hạn:** 50 users/page

**Lưu trữ:** Google Sheets "Khách hàng Zalo OA"

---

### 4. Zalo - OA - Update thông tin người dùng
**File:** `Zalo - OA - Update thông tin người dùng.json`

**Mục đích:** Cập nhật thông tin khách hàng từ form submissions vào Zalo OA.

**Luồng hoạt động:**
```
Schedule Trigger (30 phút)
    ↓
Get row(s) từ sheet "Christmas Form Submit 2025"
    ↓
If2 (lọc: chưa update + đã gửi form 2 + có tag "đk phần 2")
    ↓
If3 (lọc: gửi form 2 + chưa có Ngày nhận full)
    ↓
Loop Over Items
    ↓
Get access_token
    ↓
Get User Detail2
    ↓
Wait (chờ)
    ↓
HTTP Request (update user info lên Zalo)
    ↓
Append or update row in sheet
    ↓
Update row (đánh dấu "Đã update")
```

**Điều kiện active:** ✅ Đang bật

---

### 5. Zalo - OA - Gửi tin nhắn
**File:** `Zalo - OA -  Gửi tin nhắn.json`

**Mục đích:** Gửi các loại tin nhắn đến khách hàng Zalo OA.

**Được gọi từ:** `Zalo - OA - Webhook.json`

**Loại tin nhắn hỗ trợ:**
- Text message
- Template (generic, media)
- Promotion template
- Transaction template

**Các endpoint API:**
- `https://openapi.zalo.me/v2.0/oa/message`
- `https://openapi.zalo.me/v3.0/oa/message/cs`
- `https://openapi.zalo.me/v3.0/oa/message/promotion`
- `https://openapi.zalo.me/v3.0/oa/message/transaction`

**Điều kiện active:** ✅ Đang bật

---

### 6. Zalo - Gửi Tin Nhắn Tự Động Cho Group CNGL Tài Liệu
**File:** `Zalo - Gửi Tin Nhắn Tự Động Cho Group CNGL Tài Liệu.json`

**Mục đích:** Tự động đăng bài viết từ Google Sheets lên 2 nhóm Zalo.

**Nhóm đích:**
- Group Tài Liệu: `3805678902304196474`
- Group Trung Thu: `4704866323941501409`

**Luồng hoạt động:**
```
Schedule Trigger (12:30 hàng ngày)
    ↓
Lấy bài viết từ Google Sheets
    ↓
Limit (lấy bài mới nhất)
    ↓
If (kiểm tra "Trạng thái đăng Zalo")
    ↓ (FALSE)
HTTP Request (download image)
    ↓
Read/Write Files from Disk
    ↓
Code (split text @ 480 words)
    ↓
├── tài liệu (gửi text + ảnh vào group Tài Liệu)
├── tài liệu1 (gửi phần còn lại text)
├── trung thu (gửi text + ảnh vào group Trung Thu)
├── trung thu1 (gửi phần còn lại text)
    ↓
Update row in sheet (đánh dấu "Đã đăng")
```

**Điều kiện active:** ✅ Đang bật

---

### 7. Zalo - Gửi Tin Nhắn Video cho Group CNGL
**File:** `Zalo - Gửi Tin Nhắn Video cho Group CNGL.json`

**Mục đích:** Tương tự workflow #6 nhưng dành cho video content.

**Luồng hoạt động:**
```
Schedule Trigger (12:30 hàng ngày)
    ↓
Lấy bài viết từ Google Sheets
    ↓
Limit
    ↓
If (kiểm tra đã đăng chưa)
    ↓ (FALSE)
HTTP Request (download image)
    ↓
Image Format Converter (chuyển sang PNG)
    ↓
Edit Fields
    ↓
Upload a file (lên Cloudflare R2)
    ↓
Code (split text @ 480 words)
    ↓
├── tài liệu (gửi text + imageUrl vào group)
├── tài liệu1 (gửi phần còn lại text)
├── trung thu, trung thu1 (tương tự)
    ↓
Update row in sheet
```

---

### 8. Zalo - Bot Notifications
**File:** `Zalo - Bot Notifications.json`

**Mục đích:** Chatbot AI tự động trả lời tin nhắn Telegram-like bot.

**Luồng hoạt động:**
```
Webhook nhận tin nhắn
    ↓
AI Agent (Gemini)
    ↓
HTTP Request3 (gửi typing indicator)
    ↓
HTTP Request2 (gửi reply)
```

**Bot API:** `https://bot-api.zapps.me/bot1373614001553898373:rYZxMjpkiUmAlakyeOiQeBHFFnXcfNvTanWDzrlpWCROciXiQSIzvnAYtaTPYXrT`

**Webhook path:** `zalo-bot-notification`

**System prompt:** "Bạn là OTIMO bot, là cô chăm sóc khách hàng thân thiện"

---

### 9. Zalo - Tự Động Add & Accept Bạn Bè
**File:** `Zalo - Tự Động Add & Accept Bạn Bè.json`

**Mục đích:** Xử lý tự động friend request và chatbot AI hướng dẫn tham gia group.

**Luồng hoạt động:**
```
Zalo Event Trigger (message, friend_event, group_event)
    ↓
Lấy thông tin người dùng
    ↓
Switch1 (phân loại)
    ├── Người dùng kết bạn → Kiểm tra trạng thái
    │                       ├── Chưa kết bạn → Gửi lời mời kết bạn
    │                       └── Đã là bạn bè → AI Agent
    └── Người dùng nhắn tin → Chấp nhận lời mời kết bạn → AI Agent
                                              ↓
                                      AI Agent (Gemini)
                                              ↓
                                    Zalo Gửi tin nhắn
```

**AI Agent prompts:** Hướng dẫn người dùng tham gia 2 group:
- Tài Liệu Ngành Bánh: https://zalo.me/g/ujerqq444
- Chia Sẻ Kiến Thức Làm Bánh Trung Thu: https://zalo.me/g/szigxh684

---

### 10. Zalo - Larkbase - Lấy Thông Tin Khách Hàng
**File:** `Zalo - Larkbase - Lấy Thông Tin Khách Hàng Từ Group CNGLTài Liệu.json`

**Mục đích:** Lấy thông tin thành viên từ nhóm Zalo và lưu vào Larkbase.

**Luồng hoạt động:**
```
Manual Trigger
    ↓
Input (app_id, app_secret, table_id, sheet_id)
    ↓
Get Lark Token
    ↓
Lấy thông tin nhóm (groupId: 3805678902304196474)
    ↓
Split Out (tách memVerList)
    ↓
Code in JavaScript (slice: items 0-100)
    ↓
Lấy thông tin người dùng (theo userId)
    ↓
Split Out1
    ↓
HTTP Request (tạo record trong Larkbase)
```

**Lưu ý:**
- Giới hạn ~180 người/ngày
- Giới hạn ~50 records/lần

---

## External Integrations

### Google Sheets
- **TỔNG HỢP BÀI VIẾT TRÊN FANPAGE** - Nguồn bài viết đăng Zalo
- **Khách hàng Zalo OA** - Lưu thông tin khách hàng

### Larkbase (Larksuite)
- **app_id:** `cli_a8746eb09c395ed4`
- **table_id:** `VVfFbUMNlalM81shqKcldt4IgBb`
- **sheet_id:** `tblKVJyN8G47Z8qc`

### Cloudflare R2
- Bucket: `otimo`
- Dùng cho workflow #7 (upload hình ảnh)

### Zalo API Credentials
- **Zalo API** - `ZNGzItbJ2er3g4Pv` (Lam Tuyền)
- **n8n API Zalo** - `FuDMiFvG3yGpaalJ` (n8n Account)

---

## Triển Khai

1. Import các file `.json` vào n8n instance
2. Cấu hình credentials cho:
   - Google Sheets OAuth2 (`Lamtuyen account`)
   - Zalo API (`Lam Tuyền 05-11-2025 14:55`, etc.)
   - Google Gemini (`Google Gemini - Tân`)
   - Cloudflare R2 (`Cloudflare R2 Storage account`)
3. Cập nhật các hardcoded values:
   - Group IDs
   - DataTable IDs
   - Document IDs
   - API endpoints/credentials nếu thay đổi
4. Kích hoạt các workflow theo nhu cầu

---

## Lưu Ý Quan Trọng

- **Credentials:** Các credential IDs trong JSON là reference, cần tạo credentials mới phù hợp với instance của bạn
- **Hardcoded IDs:** Nhiều workflow chứa IDs cứng (group IDs, sheet IDs, user IDs) - cần điều chỉnh
- **Rate Limits:** Workflow #10 có giới hạn ~180 người/ngày và ~50 records/lần
- **Schedule:** Workflow #1, #4, #6, #7 chạy tự động theo lịch
