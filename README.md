# N8N Zalo Workflows

Collection of n8n workflows for Zalo automation for the OTIMO/Cẩm Nang Gối Lò project.

## Overview

These workflows handle:
- Automated message sending to Zalo groups
- Zalo Official Account (OA) management
- AI chatbot for automated responses
- Data sync with Google Sheets & Larkbase
- Automatic friend adding and invitation acceptance

## Directory Structure

```
N8N_Zalo/
├── Zalo - OA - Tự động lấy Access Token.json    # Token refresh every 24h
├── Zalo - OA - Webhook.json                    # Follow/unfollow event handling
├── Zalo - OA - Lấy thông tin khách hàng.json   # Customer list sync
├── Zalo - OA - Update thông tin người dùng.json # Periodic user info update
├── Zalo - OA -  Gửi tin nhắn.json               # OA message sending
├── Zalo - Gửi Tin Nhắn Tự Động Cho Group CNGL Tài Liệu.json  # Auto post to groups
├── Zalo - Gửi Tin Nhắn Video cho Group CNGL.json # Auto post videos to groups
├── Zalo - Bot Notifications.json                # AI chatbot
├── Zalo - Tự Động Add & Accept Bạn Bè.json     # Auto friend request
└── Zalo - Larkbase - Lấy Thông Tin Khách Hàng.json # Larkbase sync
```

---

## Workflow Details

### 1. Zalo - OA - Auto Token Refresh
**File:** `Zalo - OA - Tự động lấy Access Token.json`

**Purpose:** Automatically refresh Zalo OA access token every 24 hours.

**Flow:**
```
Schedule Trigger (24h)
    ↓
Get row(s) from DataTable "Zalo - Access"
    ↓
Limit (get latest item)
    ↓
get_access_token_zns (refresh token via API)
    ↓
Upsert row(s) (save new token)
```

**Status:** ✅ Active

---

### 2. Zalo - OA - Webhook
**File:** `Zalo - OA - Webhook.json`

**Purpose:** Handle events from Zalo OA (follow/unfollow/messages).

**Flow:**
```
Webhook receives event
    ↓
Switch (classify by event_name)
    ├── unfollow → Update row (follow=FALSE)
    ├── follow → Get access_token → Get User Detail → Check if exists
    │                                     ↓
    │                          If (new) → Append row
    │                          If (exists) → Update row
    └── Message → (route elsewhere)
```

**Webhook ID:** `67db90d6-d72c-462a-87b4-5e8df6596430`

**Storage:** Google Sheets "Khách hàng Zalo OA"

---

### 3. Zalo - OA - Get Customer Info
**File:** `Zalo - OA - Lấy thông tin khách hàng.json`

**Purpose:** Fetch all customers from Zalo OA (with pagination).

**Flow:**
```
Manual Trigger
    ↓
Get access_token
    ↓
Get First Page (get total count)
    ↓
Generate Offsets (calculate: 0, 50, 100, ...)
    ↓
Loop Over Offsets
    ↓
Get Users Page (50 users per page)
    ↓
Split Users
    ↓
Get User Detail (get each user details)
    ↓
Code (filter null/error)
    ↓
Append row in sheet
```

**Limit:** 50 users/page

**Storage:** Google Sheets "Khách hàng Zalo OA"

---

### 4. Zalo - OA - Update User Info
**File:** `Zalo - OA - Update thông tin người dùng.json`

**Purpose:** Update customer info from form submissions to Zalo OA.

**Flow:**
```
Schedule Trigger (30 minutes)
    ↓
Get row(s) from sheet "Christmas Form Submit 2025"
    ↓
If2 (filter: not updated + form 2 sent + tag "đk phần 2")
    ↓
If3 (filter: form 2 sent + no "Full Received Date")
    ↓
Loop Over Items
    ↓
Get access_token
    ↓
Get User Detail2
    ↓
Wait
    ↓
HTTP Request (update user info on Zalo)
    ↓
Append or update row in sheet
    ↓
Update row (mark "Updated")
```

**Status:** ✅ Active

---

### 5. Zalo - OA - Send Message
**File:** `Zalo - OA -  Gửi tin nhắn.json`

**Purpose:** Send various message types to Zalo OA customers.

**Called from:** `Zalo - OA - Webhook.json`

**Supported message types:**
- Text message
- Template (generic, media)
- Promotion template
- Transaction template

**API endpoints:**
- `https://openapi.zalo.me/v2.0/oa/message`
- `https://openapi.zalo.me/v3.0/oa/message/cs`
- `https://openapi.zalo.me/v3.0/oa/message/promotion`
- `https://openapi.zalo.me/v3.0/oa/message/transaction`

**Status:** ✅ Active

---

### 6. Zalo - Auto Send Messages to CNGL Documents Group
**File:** `Zalo - Gửi Tin Nhắn Tự Động Cho Group CNGL Tài Liệu.json`

**Purpose:** Automatically post articles from Google Sheets to 2 Zalo groups.

**Target groups:**
- Documents Group: `3805678902304196474`
- Mid-Autumn Group: `4704866323941501409`

**Flow:**
```
Schedule Trigger (12:30 daily)
    ↓
Fetch articles from Google Sheets
    ↓
Limit (get latest article)
    ↓
If (check "Zalo Post Status")
    ↓ (FALSE)
HTTP Request (download image)
    ↓
Read/Write Files from Disk
    ↓
Code (split text @ 480 words)
    ↓
├── tài liệu (send text + image to Documents Group)
├── tài liệu1 (send remaining text)
├── trung thu (send text + image to Mid-Autumn Group)
├── trung thu1 (send remaining text)
    ↓
Update row in sheet (mark "Posted")
```

**Status:** ✅ Active

---

### 7. Zalo - Send Video Messages to CNGL Group
**File:** `Zalo - Gửi Tin Nhắn Video cho Group CNGL.json`

**Purpose:** Similar to workflow #6 but for video content.

**Flow:**
```
Schedule Trigger (12:30 daily)
    ↓
Fetch articles from Google Sheets
    ↓
Limit
    ↓
If (check if already posted)
    ↓ (FALSE)
HTTP Request (download image)
    ↓
Image Format Converter (convert to PNG)
    ↓
Edit Fields
    ↓
Upload a file (to Cloudflare R2)
    ↓
Code (split text @ 480 words)
    ↓
├── tài liệu (send text + imageUrl to group)
├── tài liệu1 (send remaining text)
├── trung thu, trung thu1 (similar)
    ↓
Update row in sheet
```

---

### 8. Zalo - Bot Notifications
**File:** `Zalo - Bot Notifications.json`

**Purpose:** AI chatbot that automatically replies to messages (Telegram-like bot).

**Flow:**
```
Webhook receives message
    ↓
AI Agent (Gemini)
    ↓
HTTP Request3 (send typing indicator)
    ↓
HTTP Request2 (send reply)
```

**Bot API:** `https://bot-api.zapps.me/bot1373614001553898373:rYZxMjpkiUmAlakyeOiQeBHFFnXcfNvTanWDzrlpWCROciXiQSIzvnAYtaTPYXrT`

**Webhook path:** `zalo-bot-notification`

**System prompt:** "Bạn là OTIMO bot, là cô chăm sóc khách hàng thân thiện"

---

### 9. Zalo - Auto Add & Accept Friends
**File:** `Zalo - Tự Động Add & Accept Bạn Bè.json`

**Purpose:** Handle automatic friend requests and AI chatbot guiding users to join groups.

**Flow:**
```
Zalo Event Trigger (message, friend_event, group_event)
    ↓
Get user info
    ↓
Switch1 (classify)
    ├── User adds friend → Check status
    │                       ├── Not friends → Send friend request
    │                       └── Already friends → AI Agent
    └── User messages → Accept friend request → AI Agent
                                              ↓
                                      AI Agent (Gemini)
                                              ↓
                                    Zalo Send message
```

**AI Agent prompts:** Guide users to join 2 groups:
- Documents Group: https://zalo.me/g/ujerqq444
- Mid-Autumn Knowledge Sharing: https://zalo.me/g/szigxh684

---

### 10. Zalo - Larkbase - Get Customer Info
**File:** `Zalo - Larkbase - Lấy Thông Tin Khách Hàng Từ Group CNGLTài Liệu.json`

**Purpose:** Fetch member info from Zalo group and save to Larkbase.

**Flow:**
```
Manual Trigger
    ↓
Input (app_id, app_secret, table_id, sheet_id)
    ↓
Get Lark Token
    ↓
Get group info (groupId: 3805678902304196474)
    ↓
Split Out (split memVerList)
    ↓
Code in JavaScript (slice: items 0-100)
    ↓
Get user info (by userId)
    ↓
Split Out1
    ↓
HTTP Request (create record in Larkbase)
```

**Note:**
- Limit ~180 people/day
- Limit ~50 records/batch

---

## External Integrations

### Google Sheets
- **TỔNG HỢP BÀI VIẾT TRÊN FANPAGE** - Source for Zalo posts
- **Khách hàng Zalo OA** - Customer info storage

### Larkbase (Larksuite)
- **app_id:** `cli_a8746eb09c395ed4`
- **table_id:** `VVfFbUMNlalM81shqKcldt4IgBb`
- **sheet_id:** `tblKVJyN8G47Z8qc`

### Cloudflare R2
- Bucket: `otimo`
- Used for workflow #7 (image upload)

### Zalo API Credentials
- **Zalo API** - `ZNGzItbJ2er3g4Pv` (Lam Tuyền)
- **n8n API Zalo** - `FuDMiFvG3yGpaalJ` (n8n Account)

---

## Deployment

1. Import `.json` files into n8n instance
2. Configure credentials for:
   - Google Sheets OAuth2 (`Lamtuyen account`)
   - Zalo API (`Lam Tuyền 05-11-2025 14:55`, etc.)
   - Google Gemini (`Google Gemini - Tân`)
   - Cloudflare R2 (`Cloudflare R2 Storage account`)
3. Update hardcoded values:
   - Group IDs
   - DataTable IDs
   - Document IDs
   - API endpoints/credentials if changed
4. Enable workflows as needed

---

## Important Notes

- **Credentials:** Credential IDs in JSON are references; you need to create new credentials suitable for your instance
- **Hardcoded IDs:** Many workflows contain hardcoded IDs (group IDs, sheet IDs, user IDs) - adjust as needed
- **Rate Limits:** Workflow #10 has limits of ~180 people/day and ~50 records/batch
- **Schedule:** Workflows #1, #4, #6, #7 run automatically on schedule
