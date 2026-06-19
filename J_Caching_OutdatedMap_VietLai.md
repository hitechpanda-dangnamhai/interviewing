# 🗺️ OUTDATED MAP — Caching (Bản đồ "Cũ → Mới")
**Phạm vi:** tổng kết xuyên suốt mục **J (Caching)** · *đã verify 6/2026*

---

## 📌 Tài liệu này là gì & dùng thế nào

Đây **KHÔNG phải một bài học mới** (không có cấu trúc ①–⑩, không có mã phủ J-XXX riêng). Đây là **bảng tra cứu tổng** gom mọi cặp **"hiểu lầm phổ biến → cách làm đúng hiện nay"** rải khắp Bài 1–11.

Dùng nó như một **checklist review**: mỗi khi đọc code cache (của mình hoặc do AI sinh), lướt qua các mục dưới — nếu thấy "cái cũ" xuất hiện, đó là **red flag** cần sửa.

> 💡 **Lưu ý vận hành:** các dòng có gắn **(verify)** là thứ **thay đổi theo thời gian** (license, version, default) — **luôn đối chiếu lại** tại nguồn chính thức khi học/áp dụng, đừng tin trí nhớ.

---

## ⚡ Bảng tra nhanh (gọn để liếc)

| # | Cái CŨ / hiểu lầm | Hiện nay nên dùng | Liên quan |
|---|---|---|---|
| 1 | *"Redis là BSD, free vô tư"* | **Redis 8.x tri-license**; **Valkey** là fork BSD | Bài 7, 9 |
| 2 | *"Redis hoàn toàn single-thread"* | Execution 1 luồng + **I/O threads** | Bài 7 |
| 3 | **`KEYS *`** quét key | **`SCAN`** (cursor, non-blocking) | Bài 7 |
| 4 | **`DEL`** một big key | **`UNLINK`** (async) | Bài 5, 7 |
| 5 | Cứ đập DB khi "không tồn tại" | **Negative caching + Bloom filter** | Bài 3, 5 |
| 6 | TTL cố định cho hot key/top-k | **TTL jitter + single-flight + L1/L2** | Bài 5 |
| 7 | *"Update cache khi write"* | **Delete (invalidate)** khi write | Bài 4 |
| 8 | **Bull** (cũ) | **BullMQ 5.x** (Redis Streams) | Bài 10 |
| 9 | Chung 1 Redis cache + session/queue | **Tách instance** (policy khác nhau) | Bài 6, 8, 10 |
| 10 | `Cache-Control` mơ hồ / lẫn `no-cache`↔`no-store` | Hiểu rõ theo **RFC 9111** | Bài 11 |

---

## 🔍 Giải thích chi tiết từng mục

### 1. Redis license: *"BSD free vô tư"* → **tri-license / Valkey** *(verify)*

- **Cũ:** ai cũng tưởng **Redis** mãi mãi là **BSD** (giấy phép cực thoáng, dùng sao cũng được).
- **Mới:** **Redis 8.x** dùng **tri-license** — **RSALv2 / SSPLv1 / AGPLv3**. Còn **Valkey** là một **fork BSD** (do **Linux Foundation** đỡ đầu, **AWS / GCP** hậu thuẫn).

> 🎯 **Vì sao quan trọng:** **license đổi năm 2024** (→ **SSPL**) rồi **2025** (Redis 8 thêm **AGPLv3**). **Cloud provider** muốn bán **managed service** phải để ý license → nhiều team **chọn Valkey hoặc DragonflyDB** thay vì Redis "chính chủ". *(verify)*

**Ví dụ trực giác:** bạn xây sản phẩm thương mại dựa trên một thư viện tưởng "free vô tư", rồi một ngày license siết lại → buộc phải mở mã nguồn hoặc trả phí. Đó là lý do **đọc kỹ license trước khi cắm sâu** vào một hạ tầng.

---

### 2. *"Redis hoàn toàn single-thread"* → **execution 1 luồng + I/O threads** *(verify)*

- **Cũ:** *"Redis chỉ 1 luồng, max 1 core"* — coi như tuyệt đối.
- **Mới:** **command execution VẪN chạy 1 luồng** (để giữ **atomic** — Bài 7), **nhưng** có **I/O threads** (**Redis 6+**, mạnh hơn ở **8.x / Valkey 8.x**) lo phần đọc/ghi socket *song song*.

> 🎯 **Vì sao tinh tế:** **lý do Redis nhanh vẫn đúng** (in-memory + thực thi tuần tự không lock), nhưng câu **"1 core"** **không còn tuyệt đối** — phần I/O mạng đã song song hoá. *(verify)*

---

### 3. **`KEYS *`** → **`SCAN`**

- **Cũ:** dùng **`KEYS *`** để liệt kê/quét key.
- **Mới:** dùng **`SCAN`** (**cursor**, **non-blocking**, trả từng mẻ).

> 🚨 **Vì sao cấm `KEYS *` trên production:** nó là **O(n)** và **block single-thread** → trong lúc nó chạy, **MỌI client khác đứng yên** → **giết latency toàn hệ thống** (đúng hệ quả single-thread ở Bài 7).

**Ẩn dụ:** `KEYS *` như bắt **một quầy thanh toán duy nhất** dừng lại để **đếm toàn bộ kho** — cả hàng khách phía sau đóng băng. `SCAN` chia việc đếm thành **nhiều mẻ nhỏ xen kẽ**, không chặn ai.

---

### 4. **`DEL` big key** → **`UNLINK`**

- **Cũ:** **`DEL`** một **big key** (value/collection khổng lồ).
- **Mới:** **`UNLINK`** — **giải phóng bộ nhớ ASYNC**, trả về ngay.

> 🚨 **Vì sao:** **`DEL` một big key block single-thread** (phải giải phóng O(n) phần tử ngay) → chặn mọi client. **`UNLINK` trả ngay**, dọn dẹp ở nền (Bài 5).

---

### 5. Cứ đập DB khi "không tồn tại" → **Negative caching + Bloom filter**

- **Cũ:** id không tồn tại → **lần nào cũng xuống DB** (cache không điền được vì chẳng có gì).
- **Mới:** **negative caching** (cache marker **null** với **TTL ngắn**) + **Bloom filter** chống **penetration**.

> 🎯 **Vì sao:** **bảo vệ DB khỏi id rác / tấn công** — kẻ gửi dồn dập id không tồn tại sẽ bị **chặn ngay ở cache/Bloom filter** thay vì **xuyên thủng** xuống DB (Bài 3 + Bài 5).

> ⚠️ Nhắc lại từ Bài 5: **null-cache đơn thuần KHÔNG đủ** trước tấn công đa dạng key (sinh vô số null key → phình RAM) → cần **Bloom filter + validate + rate limit**.

---

### 6. TTL cố định cho hot key/top-k → **TTL jitter + single-flight + L1/L2**

- **Cũ:** "naive RAG/cache top-k + **TTL cố định**" — đặt cùng một TTL cho cả loạt key.
- **Mới:** **TTL jitter** (base + random) + **single-flight / lock** cho **hot key** + nhiều tầng **L1/L2**.

> 🎯 **Vì sao:** tránh **avalanche** (nhiều key hết hạn cùng lúc → DB sốc) **và** **stampede** (1 hot key vỡ → N request cùng rebuild). **Đúng bệnh đúng thuốc** — jitter cho avalanche, single-flight cho stampede (Bài 5).

---

### 7. *"Update cache khi write"* → **Delete (invalidate) cache khi write**

- **Cũ:** khi ghi, **ghi đè giá trị mới vào cache**.
- **Mới:** khi ghi, **delete (xoá) key khỏi cache**; lần đọc sau tự nạp lại từ DB.

> 🚨 **Vì sao:** **update-cache dễ race** — hai writer ghi cache lệch thứ tự so với DB → **giá trị cũ đè giá trị mới → stale "bền"** nằm lì trong cache. **Delete an toàn hơn nhiều** (Bài 4).

> 📌 Kèm theo (Bài 4): **update DB TRƯỚC, delete cache SAU** + **TTL backstop**.

---

### 8. **Bull** → **BullMQ 5.x** *(verify 5.78.x — 6/2026)*

- **Cũ:** thư viện queue **Bull**.
- **Mới:** **BullMQ 5.x** (**TypeScript-native**, dựa **Redis Streams**).

> 🎯 **Vì sao:** **Bull đã được kế nhiệm** bởi BullMQ — TS-native, tận dụng **Streams + consumer group** cho **at-least-once** (Bài 10). Nhớ: **consumer phải idempotent**.

---

### 9. Chung 1 Redis cho cache + session/queue → **Tách instance**

- **Cũ:** nhét **cache + session + queue** vào **một Redis** với cùng **`allkeys-lru`** "cho tiện".
- **Mới:** **tách instance** — **cache** (**`allkeys-lru`**, evict thoải mái) **vs store bền** (**`noeviction` + persistence**).

> 🚨 **Vì sao:** khi đầy memory, **eviction xoá nhầm session/job** ("không-được-mất") → **user bị logout / mất việc**; hoặc job ngốn RAM **đẩy cache ra**. Tải & failure mode khác nhau → **phải tách** (Bài 6, 8, 10).

**Ẩn dụ:** đừng để **giấy nháp vứt đi** (cache) nằm chung ngăn với **hợp đồng gốc** (session/job) — lúc dọn bàn dễ vứt nhầm.

---

### 10. `Cache-Control` mơ hồ / lẫn `no-cache` ↔ `no-store` → hiểu rõ theo **RFC 9111**

- **Cũ:** đặt header `Cache-Control` mơ hồ, hay **lẫn `no-cache` với `no-store`**.
- **Mới:** hiểu rõ **`max-age` / `s-maxage` / `no-store` / `no-cache` / `private`** theo **RFC 9111**.

> 🚨 **Vì sao:** đặt sai → **rò cache giữa user** (data riêng đánh `public`) **hoặc miss tràn** (Vary quá rộng). Nhắc lại: **`no-cache` = lưu + revalidate**, **`no-store` = cấm lưu** — **hai nghĩa khác hẳn** (Bài 11).

---

## ⑦ Mental model (cho cả bản đồ)

> Mỗi mục trên là một **bẫy AI/dev hay mắc**. Quy về **4 câu hỏi review**:
> 1. **Lệnh nào block single-thread?** → `KEYS`→`SCAN`, `DEL` big key→`UNLINK`, Lua ngắn.
> 2. **Có thứ "không-được-mất" nằm chung instance có eviction không?** → tách.
> 3. **Khi ghi: DELETE chứ không UPDATE; DB trước, cache sau.**
> 4. **Response có data riêng user không?** → `private`/`no-store`, key gồm user.

---

> 🔎 **AI hay sai ở đây (tổng):** AI hay bê **mặc định cũ** ("Redis BSD free", "update cache", "1 Redis cho mọi việc", "public max-age cho mọi response", "exactly-once"). **Bản đồ này chính là danh sách điểm cần soi** khi review code do AI sinh.
