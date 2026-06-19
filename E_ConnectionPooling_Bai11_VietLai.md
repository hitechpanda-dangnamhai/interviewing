# 🎯 BÀI 11 — Connection pooling

**Phủ:** E-POOL-001 → 006

> 💡 Bài này là **hệ quả trực tiếp của "connection = process"** (Bài 9). **Sai pooling là nguyên nhân kinh điển của sập DB dưới tải** — DB không chết vì query nặng, mà vì *quá nhiều connection*. Hiểu pool là hiểu cách bảo vệ DB khỏi chính traffic của mình.

---

## ① Mục tiêu & vị trí

Bài này về **connection pooling** — và nó là **hệ quả trực tiếp** của bài học **"connection = process"** ở **Bài 9** (mỗi connection Postgres là một backend process tốn RAM/CPU).

> 🚨 **Sai pooling = sập DB dưới tải.** Đây là một trong những sự cố production phổ biến nhất.

Bài này liên hệ **mục G** (Node) nhưng **nhìn từ phía database** — tức là quan tâm "DB chịu được bao nhiêu connection", không phải "app muốn mở bao nhiêu".

> 🎯 **Chốt định vị:** Pool không phải để app *nhanh hơn* — nó để **bảo vệ DB khỏi quá tải connection**. Đây là tư duy phòng thủ, không phải tăng tốc.

---

## ② Giảng cơ bản → nâng cao

### (a) Vì sao cần pool  `E-POOL-001`

> **Ẩn dụ:** Mở connection mới mỗi request giống *mỗi lần cần nước lại đi khoan một cái giếng mới*. **Pool** là *giữ sẵn vài vòi nước đã nối* — ai cần thì dùng, dùng xong trả lại, không phải khoan lại.

Mở **connection mới mỗi request** tốn:
- **handshake TCP** (bắt tay mạng),
- **auth** (xác thực),
- **tạo backend process** (tốn nhất — Bài 9).

**Pool tái dùng connection sẵn có** → **giảm latency** + **bảo vệ DB khỏi quá tải connection**.

```text
KHÔNG pool — mỗi request khoan giếng mới:
  Request 1: handshake TCP → auth → tạo process → query → ĐÓNG    ❌ tốn 3 bước đầu
  Request 2: handshake TCP → auth → tạo process → query → ĐÓNG    ❌ lặp lại từ đầu

CÓ pool — tái dùng vòi sẵn:
  Request 1: MƯỢN connection từ pool → query → TRẢ lại pool   ✅ khỏi 3 bước đầu
  Request 2: MƯỢN connection từ pool → query → TRẢ lại pool   ✅ tái dùng
```

---

### (b) Pool size  `E-POOL-002`

> 🚨 **"Càng to càng tốt" là SAI.** Đây là hiểu lầm nguy hiểm nhất về pool.

**Vì sao pool quá lớn lại chậm hơn?** Vì pool lớn → DB nhận quá nhiều connection cùng lúc → **context switch** (CPU phải chuyển qua lại giữa hàng trăm process) + **lock contention** (tranh chấp khóa) → **chậm hơn**, không nhanh hơn.

> 🔎 **Quy mô theo capacity của DB**, không theo số connection app muốn:
> - **Heuristic** (verify, đo thực tế): khoảng **cores × 2 + effective spindles**.
> - Quan trọng nhất: **đo thực tế**, không áp con số lý thuyết.

> 💡 **Ví dụ trực giác:** Một quầy bếp 4 đầu bếp phục vụ tốt nhất với ~8 đơn xử lý song song. Nhồi 200 đơn cùng lúc không làm bếp nhanh hơn — chỉ làm hỗn loạn. DB cũng vậy.

---

### (c) PgBouncer & 3 mode  `E-POOL-003`

> **Ví dụ trực giác:** 3 mode là 3 mức "chia sẻ vòi nước". Giữ riêng cả phiên (an toàn, ít chia sẻ) → trả sau mỗi tx (chia sẻ tốt) → trả sau mỗi câu lệnh (chia sẻ tối đa, khắt khe nhất).

| Mode | Cách hoạt động | Ưu | Nhược |
|------|----------------|----|-------|
| **session mode** | giữ connection **cả phiên client** | ✅ **an toàn** (session state ok) | ❌ **tái dùng kém** |
| **transaction mode** | trả connection **sau mỗi tx** | ✅ **tái dùng tốt nhất** | ❌ **cấm session-state** (một số **prepared statement** / **SET** / **advisory lock**) |
| **statement mode** | trả sau **mỗi câu lệnh** | chặt nhất | **hiếm dùng** |

> ⚠️ **transaction mode** là lựa chọn phổ biến nhất cho web app, nhưng phải nhớ: nó **cấm session-state** — code dùng `SET`, một số **prepared statement**, hay **advisory lock** giữ qua nhiều tx sẽ hỏng.

---

### (d) Mép giới hạn

#### 🔍 Serverless cháy connection  `E-POOL-004`

> **Ẩn dụ:** **Serverless** (như **Lambda**) scale bằng cách *đẻ thêm bản sao*. Mỗi bản sao tự mở connection riêng → 1000 bản sao = 1000 connection đập vào DB cùng lúc — như 1000 người cùng khoan giếng.

**Mỗi instance Lambda mở connection riêng**, **scale ngang** → **bùng nổ số connection** (**connection storm**).

> 🎯 **Giải:** dùng **external pooler** (**PgBouncer**/**RDS Proxy**) hoặc **Data API** — một lớp ở giữa giữ ít connection thật tới DB.

#### 🔍 Quan hệ pool ↔ max_connections  `E-POOL-005`

> ⭐ **Công thức sống còn:**
> **tổng connection = pool_size × số instance**, phải **≤ max_connections − dự phòng** (**superuser**, **replication**).

```text
Vì sao nhiều instance dễ vượt trần DB:
  pool_size = 30, autoscale lên 40 instance
        ↓
  tổng = 30 × 40 = 1200 connection
        ↓
  DB max_connections = 300
        ↓
  ❌ 1200 ≫ 300 → DB từ chối connection → app lỗi hàng loạt
```

> 🚨 **Điều tệ nhất:** vượt trần → DB từ chối connection mới → **không chỉ app quá tải mà cả các kết nối quản trị (superuser) cũng không vào được để cứu**.

#### 🔍 Triệu chứng "pool exhausted/timeout acquire"  `E-POOL-006`

> ⚠️ Lỗi **"pool exhausted"** / **"timeout acquiring connection"** = không mượn được connection từ pool. Nguyên nhân gốc:

| Nguyên nhân | Giải thích |
|-------------|------------|
| **query giữ connection lâu** | **slow query** / **long tx** / **N+1** (Bài 6) ôm connection mãi không trả |
| **pool quá nhỏ** | không đủ vòi cho tải |
| **connection leak** | **quên release** connection sau dùng |
| **DB nghẽn** | DB chậm → connection trả về chậm |

> 🎯 **Chốt:** **Sửa gốc (query/tx) TRƯỚC khi tăng pool.** Tăng pool để "chữa" timeout thường chỉ che triệu chứng — gốc rễ là query giữ connection lâu.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| **Pool/max_connections càng lớn càng tốt** | Pool **nhỏ + đúng**, đo theo **DB capacity** | Connection nhiều làm **chậm** |
| **Serverless** nối thẳng Postgres | **RDS Proxy** / **PgBouncer** ở giữa | Tránh **bùng nổ connection** |
| "Timeout pool? **Tăng pool**" | Sửa **slow query**/**long tx**/**leak** trước | Tăng pool **che triệu chứng**, không chữa gốc |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify):**
> - **Web app điển hình:** **PgBouncer transaction mode** trước Postgres; **pool_size × instance ≤ max_connections − dự phòng**.
> - **Serverless/edge** dùng **pooler quản lý** (**RDS Proxy**, **Supabase pooler**, **Neon**).
> - **TL luôn tính:** *"nếu autoscale lên 50 instance thì tổng connection bao nhiêu?"* — đây là câu hỏi phải hỏi *trước* khi sự cố xảy ra.

---

## ⑤ Code + cấu hình

```ini
# PgBouncer (verify version & options tại pgbouncer.org)
[databases]
app = host=127.0.0.1 dbname=app

[pgbouncer]
pool_mode = transaction      ; tái dùng tốt nhất (⚠️ cẩn thận prepared statement/SET)
default_pool_size = 20        ; theo DB capacity, KHÔNG theo số app muốn
max_client_conn = 1000        ; client nối vào pooler (nhiều), pooler giữ ÍT tới DB

# Quy tắc trần: pool_size_per_instance × instances ≤ max_connections − reserved
# Ví dụ: 20 × 10 = 200 ≤ max_connections(300) − reserved(20) = 280  ✓
```

> 💡 Tinh thần cấu hình: **client nối vào pooler thì nhiều** (`max_client_conn = 1000`), nhưng **pooler giữ rất ít connection thật tới DB** (`default_pool_size = 20`). Pooler đóng vai "van giảm áp" giữa app và DB.

> 🚨 **AI hay sai ở đây:** AI hay khuyên **tăng pool** để fix "timeout acquiring connection" (che triệu chứng), hoặc để **serverless nối thẳng Postgres** (gây connection storm), hoặc đặt **pool_size theo số app muốn** thay vì theo **DB capacity**.

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> Vì sao **pool nhỏ tốt hơn** · **session vs transaction mode** trade-off · **connection leak** · **serverless connection storm**.

> 📌 **THUỘC** (nhớ chính xác):
> **transaction mode cấm session-state** · **tổng connection = pool × instance ≤ max_connections − reserved**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Tính **pool size an toàn** cho N instance · chẩn đoán **"pool exhausted"** theo gốc **query/tx**.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Pool tái dùng connection; nhỏ-mà-đúng > to**. **Tổng (pool × instance) phải lọt dưới max_connections**. **Pool cạn? Sửa query trước, đừng tăng pool**."*

Cách dùng: trước khi deploy, làm phép nhân **pool × instance tối đa (autoscale)** và so với **max_connections**. Khi gặp "pool exhausted", điều tra **slow query/long tx/leak** trước, đừng phản xạ tăng pool.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(TB)** | Vì sao cần **pool**? | tránh **handshake**/**auth**/**process** mỗi request |
| **(khá)** | **Pool size** bao nhiêu, vì sao to là sai? | theo **DB capacity**; to gây **contention** |
| **(khá)** | 3 mode **PgBouncer** trade-off? | **transaction** tái dùng tốt nhưng **cấm session-state** |
| **(khó)** | **Serverless** cháy connection — vì sao + giải? | mỗi instance 1 pool; **external pooler** |
| **(TB)** | Quan hệ **pool × instance** với **max_connections**? | tổng **≤ trần − dự phòng** |

> 🧩 **Thách đố tình huống:**
> *"Pool exhausted" dưới tải — điều tra hướng nào?*
>
> Trả lời chuẩn: **slow query** / **long tx** / **N+1** / **connection leak** — sửa gốc **trước khi tăng pool**.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — App autoscale 2→40 instance, pool 30/instance, max_connections=300

Vấn đề? Sửa?

> ✅ **Tiêu chí Đạt:**
> - Tính **40 × 30 = 1200 ≫ 300** → vượt trần DB.
> - Sửa: đặt **PgBouncer transaction mode** / **giảm pool/instance**.
> - ❌ Trượt nếu không nhận ra phép nhân pool × instance vượt max_connections.

### BT2 — Log "timeout acquiring connection" lúc cao điểm

Liệt kê **3 nguyên nhân gốc** trước khi nghĩ tới tăng pool.

> ✅ **Tiêu chí Đạt:**
> - **slow query/long tx**, **N+1**, **connection leak**.
> - Nhấn: sửa gốc trước, tăng pool sau cùng.
> - ❌ Trượt nếu đề xuất đầu tiên là "tăng pool".

---

## ⑩ Đọc thêm

- **PgBouncer docs** — `pgbouncer.org`
- **AWS RDS Proxy**; **"About connection pool sizing"** (**HikariCP** wiki — nguyên lý chung)

> 🎯 **Một câu mang về:** DB không sập vì query nặng — nó sập vì *quá nhiều connection*. Pool nhỏ-mà-đúng, đặt một pooler ở giữa, và luôn tự hỏi "autoscale tối đa thì tổng connection bao nhiêu?" — đó là cách giữ DB sống dưới tải.
