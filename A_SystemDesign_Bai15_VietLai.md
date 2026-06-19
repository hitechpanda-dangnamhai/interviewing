# 🎯 BÀI 15 — E-commerce Checkout: Chống Oversell & Thanh Toán Idempotent

**Phủ:** A-CHK-001 → A-CHK-005

> 💡 **Tư tưởng cốt lõi của cả bài:** Checkout là nơi **không được sai về tiền và tồn kho**. Đây là case study **strong consistency** — đối lập hẳn với feed/chat (thiên **eventual**). Sai một ly là **oversell** (bán quá số hàng), **trừ tiền hai lần**, hoặc **giữ hàng vĩnh viễn**.

> 🧭 **Vị trí & liên kết:** Khối IV — Case study. Giao với **Bài 4** (transaction/lock), **Bài 6** (consistency), **Bài 11** (fan-out đọc).

---

## ① Mục tiêu & vị trí trong mạch

Checkout là nơi **không được sai về tiền và tồn kho**. Bài này dạy ba thứ cốt lõi:

1. trừ tồn kho **atomic** chống **race**,
2. **idempotency** cho thanh toán chống **trùng**,
3. **Saga + Outbox** thay vì **2PC** để giữ nhất quán giữa nhiều service.

> 🎯 **Vì sao gọi là bài "kết tinh" của khối Đánh đổi?** Vì bạn sẽ thấy rõ **vì sao ở đây ta chọn strong consistency** — ngược với feed (Bài 11) nơi eventual là đủ. Biết khi nào nghiêng bên nào mới là năng lực thật *(nối Bài 6)*.

---

## ② Giảng: cơ bản → nâng cao

### 📘 Vì sao "đọc rồi trừ" (read-modify-write) là SAI — race condition

> **Ví dụ trực giác:** Hai người cùng nhìn vào kệ thấy *"còn 1 món"*, cùng nghĩ "còn hàng, mua được", cùng lấy → bán **2 món** dù chỉ có **1**.

Naive code:

```sql
qty = SELECT stock FROM product WHERE id=1   -- đọc được 1
if qty > 0:
    SELECT ... process ...
    UPDATE product SET stock = qty - 1 WHERE id=1  -- ghi đè bằng 0
```

```
counter stock = 1
T1  request A: READ stock = 1
T2  request B: READ stock = 1     ← cả hai cùng thấy >0
T3  request A: UPDATE stock = 0   → bán 1 món
T4  request B: UPDATE stock = 0   → ❌ bán THÊM 1 món (oversell!) dù chỉ có 1
```

> 🚨 **Đây là lost update / race condition kinh điển.** **Khoảng trống giữa đọc và ghi** chính là nơi tai nạn xảy ra — hai request chen vào nhau ở đúng khe hở đó.

---

### 📘 Ba cách trừ tồn kho atomic (đóng khoảng trống đọc–ghi)

#### Cách 1 — **Atomic conditional UPDATE** (đơn giản & mạnh nhất cho 1 dòng)

```sql
-- DB tự khóa dòng trong câu UPDATE; điều kiện stock>0 nằm NGAY trong câu ghi
UPDATE product
SET stock = stock - 1
WHERE id = 1 AND stock > 0;
-- Kiểm tra affected_rows: =1 là giữ được hàng, =0 là hết hàng (KHÔNG cho mua)
```

> 💡 **Vì sao đây là mặc định?** Vì **không có khoảng trống đọc–ghi**: điều kiện `stock > 0` và phép trừ là **một câu lệnh atomic**. DB tự lo phần khóa. Rẻ và đúng.

#### Cách 2 — **SELECT ... FOR UPDATE** (pessimistic lock, khi cần đọc nhiều dòng rồi quyết định)

```sql
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;  -- khóa dòng tới hết transaction
-- ... logic phức tạp: kiểm nhiều sản phẩm, tính combo ...
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;  -- nhả khóa
```

> ⚠️ **Đánh đổi:** giữ **khóa lâu** = **giảm throughput**, dễ thành **bottleneck** *(Bài 5)* khi gặp **hot product**. Chỉ dùng khi logic thật sự cần "khóa rồi suy nghĩ".

#### Cách 3 — **Redis atomic** (cho flash sale tải cực cao, giảm tải DB)

```text
DECR stock:product:1     -- atomic, trả về giá trị mới
-- nếu kết quả < 0 => hết hàng => INCR trả lại + từ chối
```

> 🔍 **Vì sao DECR atomic tự nhiên?** Vì **Redis single-thread** — các lệnh chạy tuần tự, không chen ngang. Cực nhanh, nhưng phải **đồng bộ ngược về DB** (**source of truth**) và **xử lý khi Redis chết**.

| Cách | Khi dùng | Đổi lại |
|---|---|---|
| **Conditional UPDATE** | 1 dòng, mặc định | (gần như không) |
| **SELECT FOR UPDATE** | cần đọc nhiều dòng rồi quyết | khóa lâu → giảm throughput |
| **Redis DECR** | flash sale tải cực cao | phải reconcile DB + lo Redis chết |

---

### 📘 Reservation có TTL — vì sao cần "giữ chỗ tạm"

> **Ví dụ trực giác:** Mua vé xem phim — lúc bạn chọn ghế, hệ **giữ ghế tạm 15 phút** cho bạn trả tiền. Trả xong → ghế thành của bạn; quá hạn → ghế nhả lại cho người khác.

Người dùng bấm "mua" nhưng chưa trả tiền:
- Nếu **trừ hẳn** → bỏ giỏ thì hàng **kẹt**.
- Nếu **không trừ** → **oversell** lúc thanh toán.

Giải pháp: **reservation** (giữ chỗ) có thời hạn (**TTL**).

```
1. Reserve: trừ available, ghi reservation{order, qty, expire_at = now+15p}
2. Trả tiền trong 15p  -> commit: chuyển reserved thành sold
3. Quá hạn / hủy       -> release: cộng trả available (compensating action)
```

> 🎯 **Chốt:** **TTL** biến *"giữ vĩnh viễn"* thành *"giữ tạm"*. Job quét hết hạn (hoặc **Redis key TTL**) tự nhả hàng.

---

### ⬆️ Idempotency cho thanh toán — chống trừ tiền 2 lần

> **Ví dụ trực giác:** Cà thẻ ở quầy, máy báo lỗi mạng nhưng tiền *đã trừ*. Bạn cà lại → trừ hai lần. **Idempotency key** như tờ biên nhận: cùng một biên nhận thì quầy chỉ tính tiền **một lần**.

Mạng **timeout**, user bấm "Pay" lại, client **retry** → nguy cơ **charge 2 lần**. Cách chặn: **idempotency key**.

```
- Client sinh 1 key duy nhất cho mỗi ý định trả tiền (vd uuid), gửi kèm request.
- Server: trước khi charge, SET key vào store (Redis/DB) với điều kiện "chưa tồn tại".
  + Nếu key MỚI    -> thực hiện charge, lưu kết quả gắn với key.
  + Nếu key ĐÃ CÓ  -> trả lại KẾT QUẢ CŨ, KHÔNG charge lần nữa.
```

> 🎯 **Quy tắc TL:** Mọi payment gateway nghiêm túc (**Stripe, PayPal**…) đều nhận **Idempotency-Key**. **Request làm thay đổi tiền/tồn kho bắt buộc idempotent.**

---

### ⬆️ Saga + Compensating action + Outbox — thay cho 2PC

Checkout chạm nhiều service: **Inventory, Payment, Order, Shipping**. Muốn *"tất cả cùng thành công hoặc cùng hủy"*.

> 🚨 **Vì sao không 2PC?** **2PC (two-phase commit)** làm được trên giấy nhưng **khóa tài nguyên xuyên service**, **chậm**, và **kẹt khi coordinator chết** → gần như không ai dùng ở quy mô lớn.

Thay bằng **Saga**: chuỗi **bước cục bộ**, mỗi bước có **compensating action** (hành động bù trừ) nếu bước sau hỏng:

```
reserve_inventory  --hỏng-->  (không cần bù, chưa làm gì)
charge_payment     --hỏng-->  release_inventory                       (bù bước trước)
create_order       --hỏng-->  refund_payment + release_inventory
confirm_shipping   --hỏng-->  cancel_order + refund + release
```

> 🔍 **Outbox pattern — vì sao cần?** Để **không mất event** giữa "ghi DB" và "bắn message": ghi **event vào bảng outbox** trong **cùng transaction** với thay đổi nghiệp vụ; một **relay** đọc outbox và publish lên **message bus**.
>
> ```
> ❌ Hai bước rời (ghi DB xong MỚI bắn message):
> T1  INSERT order vào DB  ✅
> T2  CRASH trước khi bắn message  → ❌ order có nhưng event MẤT → hệ lệch
>
> ✅ Outbox (cùng transaction):
> T1  INSERT order + INSERT outbox  (cùng 1 transaction → cùng sống hoặc cùng chết)
> T2  relay đọc outbox 'NEW' → publish → đánh dấu 'SENT'  ✅ atomic state↔event
> ```
> Kết quả: **atomic giữa state và event**. Kèm **at-least-once + consumer idempotent** *(Bài 14)*.

---

### ⬆️ Flash sale — khi 100k người giành 1000 món trong 1 giây

> **Vấn đề:** **hot key** trên 1 product *(Bài 16)*, DB sập vì khóa.

Pattern phổ biến:

```
1. Waiting room / queue: chặn dòng người ngay cửa, thả vào từ từ (token).
2. Preload tồn kho vào Redis: DECR atomic ở Redis, KHÔNG đụng DB mỗi request.
3. Ai DECR ra >=0 mới được "giữ chỗ"; số còn lại trả "hết hàng" ngay (fail fast).
4. Ghi nhận người thắng -> đẩy async xuống DB qua queue (làm phẳng burst).
5. Reconcile: định kỳ đối soát Redis <-> DB; Redis chết thì có cơ chế khôi phục.
```

> 🎯 **Tinh thần:** **đẩy điểm tranh chấp ra Redis atomic** + **làm phẳng ghi xuống DB bằng queue** + **fail fast** cho người thua. Người thua biết kết quả ngay còn hơn chờ rồi vẫn trượt.

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Sai lầm hay gặp (❌) | Vì sao sai | ✅ Cách đúng |
|---|---|---|
| `SELECT qty; if>0; UPDATE qty-1` | **Race**: 2 request cùng đọc, cùng trừ → **oversell** | `UPDATE ... WHERE stock>0` atomic / **FOR UPDATE** |
| Trừ hẳn tồn kho lúc **add-to-cart** | Bỏ giỏ → hàng **kẹt vĩnh viễn** | **Reservation** có **TTL**, hết hạn tự nhả |
| Không có **idempotency key** cho Pay | Retry/double-click → **charge 2 lần** | **Idempotency key**, trả kết quả cũ nếu trùng |
| Dùng **2PC** cho cross-service | Khóa lâu, kẹt khi coordinator chết, không scale | **Saga + compensating action + Outbox** |
| Ghi DB xong mới bắn message (2 bước rời) | Crash giữa chừng → **mất/trùng event** | **Outbox** trong cùng transaction |
| Flash sale đập thẳng DB mỗi request | **Hot key** + khóa → DB sập | **Redis atomic DECR** + queue làm phẳng + **fail fast** |
| Coi checkout như feed (eventual OK) | Tiền/tồn kho sai là không chấp nhận được | Checkout = **strong consistency** có chủ đích |

---

## ④ Áp dụng thực tế + so sánh bigtech (pattern phổ biến)

- Sàn TMĐT lớn thường tách **available / reserved / sold**, **reservation TTL**, và **Saga** điều phối **Inventory–Payment–Order**. (Mô tả ở mức pattern, không khẳng định chi tiết nội bộ.)
- **Payment gateway** (**Stripe/PayPal**) công khai hỗ trợ **Idempotency-Key** — đây là **chuẩn ngành**, không phải nội bộ.
- **Flash sale** (kiểu Xiaomi/loại "giật" hàng) phổ biến dùng **waiting room + Redis preload + atomic decrement** — pattern được chia sẻ rộng rãi.

> 🎯 **Quy tắc TL chốt:** ở **checkout** chọn **strong consistency**; ở **feed/notification** chấp nhận **eventual**. Biết khi nào nghiêng bên nào mới là năng lực thật *(nối Bài 6)*.

---

## ⑤ Code thực hành + cấu hình

### (a) Trừ tồn kho atomic + reservation TTL (Python, psycopg) — KHÔNG hardcode secret

```python
import os, uuid, psycopg2
# Cấu hình từ ENV, KHÔNG bao giờ hardcode chuỗi kết nối/mật khẩu
DB_DSN = os.environ["DB_DSN"]  # ví dụ: postg:// user:pass @host/db (đặt ở ENV/secret manager)

def reserve_stock(product_id: int, qty: int, order_id: str, ttl_minutes: int = 15) -> bool:
    """Giữ chỗ tồn kho atomic. Trả True nếu giữ được, False nếu hết hàng."""
    with psycopg2.connect(DB_DSN) as conn, conn.cursor() as cur:
        # Điều kiện available>=qty NẰM TRONG câu UPDATE => atomic, KHÔNG có khoảng trống đọc-ghi
        cur.execute(
            """
            UPDATE inventory
               SET available = available - %s,
                   reserved  = reserved  + %s
             WHERE product_id = %s AND available >= %s
            """,
            (qty, qty, product_id, qty),
        )
        if cur.rowcount == 0:
            return False  # hết hàng -> từ chối, KHÔNG cho mua

        # Ghi reservation kèm hạn dùng; job/cron sẽ quét cái quá hạn để release
        cur.execute(
            """
            INSERT INTO reservation (order_id, product_id, qty, expire_at)
            VALUES (%s, %s, %s, now() + (%s || ' minutes')::interval)
            """,
            (order_id, product_id, qty, ttl_minutes),
        )
        return True  # commit tự động khi thoát "with conn"
```

### (b) Thanh toán idempotent (Redis SET NX) — chống charge 2 lần

```python
import os, json, redis
r = redis.Redis.from_url(os.environ["REDIS_URL"])  # URL lấy từ ENV

def charge_once(idem_key: str, order_id: str, amount_cents: int) -> dict:
    """Idempotent charge: cùng idem_key chỉ charge 1 lần, lần sau trả kết quả cũ."""
    cache_key = f"charge:idem:{idem_key}"
    # SET NX: chỉ set nếu CHƯA tồn tại; giữ 24h để hứng mọi retry hợp lý
    acquired = r.set(cache_key, "PENDING", nx=True, ex=24 * 3600)
    if not acquired:
        prev = r.get(cache_key)
        if prev and prev != b"PENDING":
            return json.loads(prev)          # đã có kết quả -> trả lại, KHÔNG charge
        raise RuntimeError("charge đang xử lý, hãy thử lại sau")  # tránh double in-flight

    result = call_payment_gateway(order_id, amount_cents)  # gọi gateway (cũng nên gửi Idempotency-Key)
    r.set(cache_key, json.dumps(result), ex=24 * 3600)      # lưu kết quả gắn với key
    return result
```

### (c) Outbox pattern (event không lạc khỏi transaction)

```python
def place_order_with_outbox(cur, order: dict, event: dict):
    """Ghi đơn + ghi outbox trong CÙNG transaction => state và event atomic với nhau."""
    cur.execute(
        "INSERT INTO orders (id, user_id, total_cents, status) VALUES (%s,%s,%s,'CREATED')",
        (order["id"], order["user_id"], order["total_cents"]),
    )
    # Cùng transaction: nếu rollback thì cả đơn lẫn event đều biến mất -> KHÔNG lệch
    cur.execute(
        "INSERT INTO outbox (id, topic, payload, status) VALUES (%s,%s,%s,'NEW')",
        (str(uuid.uuid4()), "order.created", json.dumps(event)),
    )
    # Một relay riêng đọc outbox status='NEW' -> publish message bus -> đánh dấu 'SENT'
    # Consumer phía sau idempotent (Bài 14) vì giao hàng at-least-once.
```

### (d) Cấu hình Postgres giảm kẹt khóa khi tải cao (gợi ý, tinh chỉnh theo tải thật)

```ini
# postgresql.conf — đặt thời gian chờ để FAIL FAST thay vì treo vô hạn khi tranh khóa
lock_timeout = '2s'                          # chờ khóa quá 2s thì bỏ, tránh kẹt dây chuyền
idle_in_transaction_session_timeout = '10s'  # giết transaction "ngậm khóa" rồi ngồi im
statement_timeout = '5s'                     # chặn truy vấn chạy quá lâu giữ khóa
```

> 🔎 **Vì sao `SET NX` ở (b) là chìa khóa?** Lần đầu gặp `idem_key` → set thành công → charge. Mọi **retry** sau (cùng key) → set thất bại → **trả kết quả cũ**, không charge lại. Trạng thái trung gian `PENDING` còn chặn cả **double in-flight** (hai request cùng key ập vào *trước khi* cái đầu xong).

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- vì sao **read-modify-write** race.
- vì sao **reservation** cần **TTL**.
- vì sao **Saga** thay **2PC**.
- vì sao **Outbox** cần **cùng transaction**.
- vì sao **checkout** nghiêng **strong consistency**.

📌 **Cần THUỘC:**
- `UPDATE...WHERE stock>0`.
- `SELECT FOR UPDATE`.
- **Redis DECR atomic**.
- **idempotency key**.
- **compensating action**.
- **available/reserved/sold**.

🛠️ **Cần LÀM ĐƯỢC:**
- viết **trừ tồn kho atomic**.
- cài **idempotent charge**.
- phác **Saga có bù trừ**.
- thiết kế **flash sale** Redis preload + queue.

---

## ⑦ Mental model

> 🎯 **"Đóng khoảng trống đọc–ghi (atomic), giữ chỗ có hạn (TTL), trả tiền một lần (idempotent), nhiều service thì bù trừ (Saga) chứ đừng khóa chéo (2PC)."**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Vì sao *"đọc tồn kho, nếu >0 thì trừ"* gây **oversell**? Sửa thế nào?
→ **race** giữa đọc và ghi; sửa bằng `UPDATE...WHERE stock>0` atomic hoặc **FOR UPDATE**.

**(TB)** **Reservation TTL** giải quyết vấn đề gì?
→ giữ hàng **tạm** cho người đang thanh toán mà không **kẹt hàng** khi bỏ giỏ; hết hạn tự nhả (**compensating**).

**(khó)** Chống **charge 2 lần** khi client retry?
→ **idempotency key**: lần đầu charge & lưu kết quả, lần sau trả kết quả cũ; gateway cũng nhận **Idempotency-Key**.

**(khó)** Vì sao không **2PC** mà dùng **Saga**? **Outbox** để làm gì?
→ 2PC khóa chéo/kẹt coordinator/không scale; **Saga** = bước cục bộ + bù trừ; **Outbox** giữ event **atomic với state**.

**(TB)** **Flash sale** 100k giành 1000 món chịu tải sao?
→ **waiting room + Redis preload + DECR atomic + fail fast + queue** làm phẳng ghi xuống DB.

> 🧩 **Thách đố (trick):** Team cho rằng *"dùng transaction DB là đủ chống oversell, khỏi cần Redis hay reservation"*. Phản biện?
>
> ```
> ❌ "transaction DB là đủ" — thiếu ở 3 chỗ:
> (a) hot product 1 dòng bị khóa NỐI ĐUÔI → throughput sụp ở flash sale
> (b) tiền nằm ở service KHÁC (payment) → 1 transaction DB KHÔNG bao trùm → cần Saga/idempotency
> (c) "giữ chỗ trong lúc user trả tiền" là vấn đề THỜI GIAN/TTL → transaction không giải được
>
> ✅ Transaction là điều kiện CẦN, không phải ĐỦ
> ```
> 🔎 **Vì sao là bẫy?** Transaction đúng **cho một DB**, nhưng không che được hot key, cross-service, và bài toán thời gian (reservation TTL). Nó là **điều kiện cần, không phải đủ**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thiết kế luồng checkout chống **oversell** cho sản phẩm thường (không flash sale).

> ✅ **Đạt khi:** trừ tồn kho **atomic**, **reservation TTL**, **idempotent payment**, **Saga** có **compensating** cho payment-fail.

**BT2.** Nâng cấp cho **flash sale** 1000 món / 100k người / 1 giây.

> ✅ **Đạt khi:** **Redis preload + atomic decrement**, **fail fast**, **queue** làm phẳng ghi DB, có **reconcile Redis↔DB**, nêu xử lý **Redis chết**.

**BT3.** Vẽ **Saga 4 bước** (reserve→charge→order→ship) kèm **compensating action** từng bước và chỉ rõ chỗ đặt **Outbox**.

> ✅ **Đạt khi:** mỗi bước có hành động bù **đúng thứ tự ngược**, **Outbox** nằm **cùng transaction** với state.

---

## ⑩ Đọc thêm

- **Microservices Patterns** (Chris Richardson) — *Saga, Outbox, transactional messaging*.
- **Stripe Docs** — *Idempotent requests*. **AWS** — *Saga orchestration với Step Functions*.
- **Designing Data-Intensive Applications** (Kleppmann) — ch. *weak isolation & race conditions* (lost update, write skew).

---

> 🎯 **Khẩu quyết khép bài:** *Tiền và tồn kho không được sai → strong consistency có chủ đích. Trừ kho atomic (đóng khe đọc–ghi). Giữ chỗ có TTL. Trả tiền idempotent. Nhiều service thì Saga + Outbox, đừng 2PC. Flash sale thì đẩy tranh chấp ra Redis + queue làm phẳng.*
