# 🎯 BÀI 9 — Case study: URL shortener / pastebin

**Phủ:** A-URL-001 → A-URL-005

> 💡 **Tư tưởng cốt lõi của cả bài:** Một **URL shortener** nhìn thì như trò "tạo mã ngắn", nhưng bài toán thật **không nằm ở việc tạo key** — nó nằm ở **tra ngược cực nhanh, cực nhiều lần** (**read ≫ write**). Đây là case study nhập môn để luyện **drive the interview** end-to-end mà không bị ngợp.

---

## ① Mục tiêu & vị trí trong mạch

**Case study nhập môn của khối IV** — ráp lại **Bài 1** (quy trình), **Bài 2** (read-heavy), **Bài 4** (cache/object storage).

> 🎯 **Vì sao bắt đầu bằng đề này?** Vì đề **đủ đơn giản** để bạn chạy trọn khung 6 bước mà không ngợp — nơi tốt nhất để luyện **drive the interview** end-to-end. Học xong, bạn xử được mọi biến thể *"tạo mã ngắn → tra ngược nhanh"* (mã giảm giá, mã mời, share link…).

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** **URL shortener** = một **cuốn từ điển khổng lồ** ánh xạ `short_key → long_url`. Tra từ điển (đọc) thì cả triệu lần; thêm từ mới (ghi) thì thi thoảng.

> 🎯 **Chốt:** Bài toán thật **không phải "tạo key"** mà là **tra ngược cực nhanh, cực nhiều lần** — **read ≫ write**.

---

### (b) Cơ chế

#### Clarify (theo Bài 1)

- **functional** = tạo short URL, **redirect**, *(optional: **custom alias**, **expiration**, **analytics**)*.
- **non-functional** = **read ≫ write** (📘 **A-URL-002**), **latency redirect thấp**, **availability cao**, **không trùng key**.

#### 📘 **A-URL-002 — Read-heavy ⇒ tối ưu đường đọc**

Tạo **1 lần**, redirect **hàng triệu lần** ⇒ dồn công sức vào **đường đọc**: **cache redirect nóng (Redis)** + **read replica** + *(tùy)* **CDN**.

```
ĐƯỜNG ĐỌC (redirect) — tối ưu hết mức:
request /abc123
   ↓
Redis cache  ──HIT──►  trả long_url ngay   ✅ nhanh nhất
   │ MISS
   ↓
read replica / DB  →  lấy long_url  →  nạp lại vào cache  →  trả
```

#### 📘 **A-URL-001 — Sinh short key: hai hướng đánh đổi**

| Tiêu chí | **base62 của ID tăng dần** | **hash + xử lý collision** |
|---|---|---|
| Đặc tính | gọn, **tuần tự** | **ngẫu nhiên** (khó đoán) |
| Nhược điểm | **đoán được** (lộ thứ tự, **enumerable**) | phải **check trùng (collision)** |

> 🔍 **Vì sao "enumerable" là vấn đề?** Nếu key chạy tuần tự `abc1, abc2, abc3…`, người ngoài có thể **dò tuần tự** để xem toàn bộ link người khác tạo — rò rỉ dữ liệu. Hash ngẫu nhiên né được điều này nhưng đổi lại phải **check collision** (hai URL ra cùng key).

> 💡 **Độ dài key ↔ không gian địa chỉ:** **base62 với 7 ký tự** ≈ **62⁷ ≈ 3.5 nghìn tỷ key**. Key càng dài, không gian càng lớn, va chạm càng hiếm — nhưng URL càng kém "ngắn".

#### 📘 **A-URL-003 — Unique ID ở quy mô phân tán**

> **Ví dụ trực giác:** Một quầy duy nhất phát số thứ tự cho cả thành phố → quầy đó tắc nghẽn. Giải pháp: mỗi phường được cấp sẵn một **dải số** rồi tự phát.

Tránh **auto-increment 1 DB** thành **bottleneck** (và **SPOF**):

| Cơ chế | Cách làm |
|---|---|
| **counter trung tâm / range allocation** | mỗi node **xin một dải ID** (vd 1000 số) rồi cấp local |
| **Snowflake-like** | **timestamp + machine_id + sequence** ⇒ ID **64-bit**, tuần tự theo thời gian, **không cần phối hợp toàn cục** |
| **UUID / KGS (key generation service)** | **pre-generate** key sẵn |

> 🎯 **Đánh đổi:** **tính tuần tự** (tốt cho **index** B-tree) vs **phân tán/khó đoán**.

#### 📘 **A-URL-004 — 301 vs 302 redirect**

> **Ví dụ trực giác:** **301** như dán giấy "nhà đã chuyển vĩnh viễn sang địa chỉ X" — bưu tá *nhớ luôn*, lần sau khỏi hỏi bạn. **302** như nói "tạm thời chuyển sang X" — lần sau bưu tá *vẫn ghé hỏi* bạn.

| Tiêu chí | **301 (permanent)** | **302 (temporary)** |
|---|---|---|
| Browser | **cache redirect** | mỗi lần **qua server** |
| Analytics | ❌ **mất count** các lần sau | ✅ **đếm được** |
| Lợi | nhanh hơn cho user, tốt cho **SEO** | tracking đầy đủ |

> 🎯 **Chọn theo nhu cầu tracking.**

---

### (c) Mép giới hạn 📘 **A-URL-005**

Mỗi tính năng thêm vào sẽ làm phức tạp ở một chỗ:

- **custom alias** ⇒ phải **check unique** (đụng với key sinh tự động).
- **expiration** ⇒ **TTL** + **cleanup job** (xóa key hết hạn).
- **analytics** ⇒ đẩy **event async qua queue** để **KHÔNG làm chậm redirect**.

> 🎯 **Nguyên tắc vàng:** **Redirect phải nhanh; đếm là việc phụ.** Mọi thứ "phụ" (analytics) phải dạt ra khỏi **đường nóng**.

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| **AUTO_INCREMENT 1 DB** sinh ID | **Range allocation / Snowflake / KGS** | 1 DB là **bottleneck + SPOF** ở scale lớn |
| Đếm **analytics đồng bộ** trong redirect | Đẩy **event async qua queue** | Redirect phải nhanh; đếm chậm làm hỏng **UX** |
| Dùng **301** rồi than *"không đếm được click"* | **302** nếu cần tracking | 301 bị **browser cache** ⇒ mất count |
| **Hash** rồi không check **collision** | Check trùng (hoặc dùng **counter→base62**) | Collision ⇒ **ghi đè URL người khác** |

---

## ④ Áp dụng thực tế + So sánh bigtech

Pattern này dùng ở **mọi link shortener** (**bit.ly**, **t.co**…) và mọi hệ cần **short id** (mã giảm giá, mã mời, share link).

> ➕ **Mở rộng thực tế:** short URL thường phục vụ **redirect ở edge/CDN** để **latency cực thấp toàn cầu**; **analytics pipeline** (**Kafka → warehouse**) tách hẳn khỏi **đường redirect** (pattern phổ biến).

---

## ⑤ Code thực hành + cấu hình

Sinh key bằng **counter → base62** + **cache redirect** (Python, minh hoạ):

```python
# url_shortener.py — pip install redis ; cấu hình REDIS_URL, DB qua env
import string
ALPHABET = string.digits + string.ascii_lowercase + string.ascii_uppercase  # base62

def to_base62(n: int) -> str:
    if n == 0: return ALPHABET[0]
    s = []
    while n:
        n, r = divmod(n, 62)
        s.append(ALPHABET[r])
    return "".join(reversed(s))            # ID tăng dần -> key gọn, TUẦN TỰ

# --- Giả lập tầng lưu trữ ---
class Store:
    def __init__(self, redis, db):
        self.redis = redis                 # cache redirect nóng (ĐƯỜNG ĐỌC)
        self.db = db                       # nguồn sự thật (short_key -> long_url, meta)

    def create(self, long_url, alias=None, ttl=None):
        if alias:                          # custom alias: PHẢI check unique (A-URL-005)
            if self.db.exists(alias): raise ValueError("alias đã dùng")
            key = alias
        else:
            new_id = self.db.next_id()     # range allocation / Snowflake để tránh 1-DB bottleneck
            key = to_base62(new_id)
        self.db.put(key, long_url, ttl)    # ttl -> expiration + cleanup job dọn sau
        return key

    def resolve(self, key):                # ĐƯỜNG ĐỌC: cache TRƯỚC, DB SAU
        url = self.redis.get(f"u:{key}")
        if url: return url                                 # ✅ cache HIT
        url = self.db.get(key)                             # MISS -> xuống DB
        if url: self.redis.setex(f"u:{key}", 3600, url)    # nạp lại cache nóng
        return url
```

```text
# Redirect handler: trả 302 nếu cần đếm click; đẩy event click vào QUEUE (async), KHÔNG đếm inline.
# Cấu hình:
- ID: dùng range allocation (mỗi instance xin dải 10k id) hoặc Snowflake; KHÔNG auto-increment đơn DB.
- Redirect: 302 nếu cần analytics; 301 nếu ưu tiên tốc độ/SEO và không cần đếm.
- Analytics: click -> Kafka/queue -> aggregate offline; redirect không chờ analytics.
- Cleanup: cron quét key hết hạn theo TTL (hoặc TTL native của Redis cho cache).
```

> 🔎 **Đọc kỹ `resolve`:** thứ tự **cache trước, DB sau** chính là pattern **cache-aside** — đường đọc luôn hỏi Redis trước, chỉ chạm DB khi **MISS**, rồi **nạp lại cache** cho lần sau. Đây là lý do hệ chịu được "triệu lần đọc" mà DB không gục.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- vì sao **read-heavy** định hình kiến trúc.
- **base62-counter vs hash** đánh đổi.
- **301 vs 302** ảnh hưởng count.
- vì sao **analytics phải async**.

📌 **Cần THUỘC:**
- **base62** (**62⁷ ≈ 3.5T** với 7 ký tự).
- cách sinh **distributed unique ID** (**range / Snowflake / KGS**).
- **301 = cache, 302 = đếm được**.

🛠️ **Cần LÀM ĐƯỢC:**
- thiết kế **end-to-end** URL shortener.
- chọn **cơ chế ID**.
- tách **analytics** khỏi **đường redirect**.

---

## ⑦ Mental model

> 🎯 **"Tạo 1 lần, đọc triệu lần."**
> Tối ưu **đường đọc** (cache+replica); **ID phân tán không-collision**; **đếm click async**; **302** nếu cần tracking.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(dễ)** Hệ này **read-** hay **write-heavy**, suy ra gì?
→ **read ≫ write** ⇒ **cache redirect + replica + (CDN)**.

**(TB)** Sinh short key: **base62-counter vs hash** — đánh đổi?
→ counter **gọn/tuần tự/đoán được**; hash **ngẫu nhiên/cần check collision**.

**(khó)** Sinh **unique ID phân tán** không đụng nhau?
→ **range allocation / Snowflake (ts+machine+seq) / KGS**; tránh **1-DB auto-increment**.

**(TB)** **301 vs 302** ảnh hưởng analytics/cache thế nào?
→ **301** browser-cache mất count; **302** mỗi lần qua server, đếm được.

**(khó)** Thêm **custom alias + expiration + analytics** phức tạp ở đâu?
→ alias **check unique**; expiration **TTL+cleanup**; analytics **async qua queue**.

> 🧩 **Thách đố (trick):** *"Em dùng `UPDATE clicks = clicks+1` trong handler redirect cho chính xác."* Sao không nên?
>
> ```
> ❌ Ghi DB ĐỒNG BỘ trong đường nóng:
> mỗi redirect → UPDATE clicks=clicks+1 trên CÙNG 1 row
>        ↓
> triệu redirect → triệu lần ghi vào 1 row → ❌ HOT ROW + chậm + bottleneck
>
> ✅ Đẩy event async → aggregate sau (chấp nhận đếm trễ/xấp xỉ)
> ```
> 🔎 **Vì sao là bẫy?** Ghi DB **đồng bộ trong đường nóng**: mỗi redirect +1 lần ghi vào **1 row** ⇒ **hot row** + chậm + **bottleneck** ở scale lớn. Đẩy **event async**, **aggregate** sau (chấp nhận **đếm trễ/xấp xỉ** — đúng tinh thần "đếm là việc phụ").

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thiết kế full URL shortener cho **100M URL, 10:1 read:write**. Vẽ luồng tạo + redirect, chọn **ID scheme**, đặt **cache** ở đâu.

> ✅ **Đạt khi:** tối ưu **đường đọc** rõ ràng, **ID không-collision phân tán**, **analytics async**.

**BT2.** Ước lượng **storage** cho 100M URL (mỗi record ~500B) và **QPS redirect** nếu mỗi URL được click 50 lần/đời trong 1 năm.

> ✅ **Đạt khi:** dùng **Bài 2** ra **bậc độ lớn** hợp lý.

---

## ⑩ Đọc thêm

- **System Design Interview** (Alex Xu) — *"Design A URL Shortener"*.
- **Twitter Snowflake** (ID generation).
- **system-design-primer** — *pastebin/URL shortener*.

---

> 🎯 **Khẩu quyết khép bài:** *Tạo 1 lần, đọc triệu lần → tối ưu đường đọc. ID phân tán đừng đụng nhau. Redirect phải nhanh; đếm click đẩy async. 302 nếu cần đếm, 301 nếu cần tốc độ.*
