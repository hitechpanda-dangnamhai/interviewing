# 🎯 BÀI 10 — Case study: Rate limiter

**Phủ:** A-RL-001 → A-RL-006

> 💡 **Tư tưởng cốt lõi của cả bài:** Một **rate limiter** trông đơn giản ("đếm số request"), nhưng khi có **50 server** thì nó biến thành bài toán **distributed counter + atomicity** — nơi bộc lộ rõ ai **thật sự hiểu hệ phân tán** (**race condition**, **state trung tâm**).

---

## ① Mục tiêu & vị trí trong mạch

Case study về **distributed counter + atomicity** — ráp **Bài 4** (consistent hashing), **Bài 5** (hot key), **Bài 6** (**fail-open/closed** là trade-off), giao mạnh với **mục I** (concurrency) và **mục M** (security/abuse).

> 🎯 **Vì sao bài này "lọc người"?** Vì nó ép bạn nhìn ra: *"đếm" trên một máy thì dễ, nhưng đếm đúng trên 50 máy cùng lúc* là bài toán **consistency + race**. Người chỉ biết code một máy sẽ trả lời sai ngay câu đầu.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** **Rate limiter** = **người gác cửa** đếm *"anh đã vào mấy lần trong phút này"*. Vấn đề khó: khi có **50 cửa (server)**, mỗi cửa đếm riêng thì **tổng vượt giới hạn**. Phải có **một cuốn sổ chung** (**Redis**) và **ghi sổ không bị tranh** (**atomic**).

---

### (b) Cơ chế — thuật toán 📘 **A-RL-001**

| Thuật toán | Cách hoạt động | Đặc tính |
|---|---|---|
| **Token bucket** | bình chứa **token**, đổ đều theo thời gian; mỗi request lấy 1 token | cho phép **burst** (đến dung tích bình) + **smoothing** average rate; **phổ biến nhất** |
| **Fixed window** | đếm trong cửa sổ cố định (vd mỗi phút reset) | đơn giản nhưng có **boundary burst** |
| **Sliding window log** | lưu **timestamp** mọi request | **chính xác tuyệt đối** nhưng **tốn memory** |
| **Sliding window counter** | nội suy giữa 2 cửa sổ | **dung hòa** — gần chính xác, rẻ memory |

#### 📘 **A-RL-002 — Boundary burst của Fixed window**

> **Ví dụ trực giác:** Giới hạn 100 request/phút. Khách dồn 100 request vào *giây cuối* phút 1, rồi 100 request vào *giây đầu* phút 2 → trong khoảng ~2 giây có **200 request**, gấp đôi ý định.

```
limit = 100/phút (fixed window)

phút 1 ........................[59s: 100 req]│[00s: 100 req].................... phút 2
                                            ▲ reset cửa sổ
                              └──── ~2 giây có 200 req = ❌ ~2× limit ────┘
```

> 🎯 **Khắc phục:** **sliding window (counter)** — vì nó không "reset cứng" ở mốc phút mà trượt liên tục.

---

### (c) Phân tán & race (cốt lõi)

#### 📘 **A-RL-003 — State cục bộ mỗi server là SAI**

> **Ví dụ trực giác:** 50 cửa, mỗi cửa cho phép 100 lượt. Mỗi cửa tưởng mình đúng giới hạn, nhưng tổng cộng cho qua **5000 lượt** — gấp 50 lần ý định.

Mỗi server chỉ thấy **phần traffic của nó** ⇒ **tổng vượt limit**. Cần **state trung tâm (Redis)** HOẶC **local counter + sync định kỳ**.

> 🔍 **Chốt nhận thức:** đây thực chất là một **bài toán consistency** — nhận ra điều đó là bước đầu để giải đúng.

#### 📘 **A-RL-004 — Race condition** *(giao với I)*

> **Ví dụ trực giác:** Hai nhân viên cùng nhìn sổ thấy "đã 9 lượt", cùng cộng thành 10, cùng ghi 10 — nhưng thực ra phải là 11. Một lượt "bốc hơi".

2 server cùng **read-modify-write** 1 counter ⇒ **lost update**:

```
counter hiện tại = 9 (limit = 10)
T1  server A: READ count = 9
T2  server B: READ count = 9      ← cả hai đọc cùng giá trị
T3  server A: tính 9+1 = 10, WRITE 10
T4  server B: tính 9+1 = 10, WRITE 10   ❌ phải là 11 → 1 request "lọt" mất
   → LOST UPDATE: limiter đếm thiếu, cho qua nhiều hơn giới hạn
```

> 💡 **Fix:** **atomic op** — **Redis INCR**, **Lua script** (đọc-kiểm-ghi **nguyên tử**), hoặc **transaction**. **KHÔNG** read-modify-write rời rạc.
>
> 🎯 **Chốt phỏng vấn:** **Thừa nhận race = điểm cộng**; lờ đi = **red flag**. Đây là chỗ interviewer chờ xem bạn có nhìn ra không.

#### 📘 **A-RL-005 — Redis chết ⇒ fail-open hay fail-closed?**

Đây là **trade-off có chủ đích** *(Bài 6)*:

| | **fail-open** | **fail-closed** |
|---|---|---|
| Khi Redis chết | **cho qua** | **chặn** |
| Ưu tiên | **availability** | **bảo vệ** |
| Hợp với | đa số **API public** | endpoint **tài chính/security** |

> ⚠️ **Cách trả lời đúng:** *"tùy"* + **nêu tiêu chí**, **KHÔNG** cứng nhắc. Hành vi khi lỗi phải là một **quyết định**, không phải tai nạn.

---

### (d) Mép giới hạn 📘 **A-RL-006**

> 🚨 Ở **throughput cực lớn**, chính **Redis cũng thành bottleneck / hot key** *(đúng như Bài 5)*.

Giảm tải:

- **local in-memory counter** sync Redis mỗi **~100ms** (chấp nhận **sai số nhỏ**).
- **shard key qua slot**.
- **per-user dedicated key** cho **super-user**.
- **consistent hashing** để phân tán.

> 💡 (Pattern phổ biến — **verify** chi tiết.)

---

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| **Counter cục bộ** mỗi app server | **State trung tâm (Redis)** hoặc **local+sync** | Mỗi server thấy 1 phần ⇒ tổng vượt limit |
| **GET rồi SET count+1** rời rạc | **INCR / Lua script** (atomic) | **Read-modify-write** bị race ⇒ **lost update** |
| **Fixed window** cho giới hạn chặt | **Sliding window (counter)** | Fixed window cho **boundary burst ~2×** |
| *"Redis chết thì hệ tự xử"* (không định nghĩa) | Quyết **fail-open/closed** có chủ đích | Hành vi khi lỗi phải là **quyết định**, không phải tai nạn |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Rate limiting** là lớp phòng vệ chuẩn ở **API gateway** (chống **abuse**, bảo vệ **backend**, **fair-use**, chống **DDoS lớp ứng dụng**).

> ➕ **Pattern phổ biến:** **limit nhiều tầng** (**per-IP, per-user, per-endpoint, global**); **adaptive rate limiting** (tự siết khi backend quá tải); thuật toán **token bucket** dùng rộng (**nginx limit_req**, cloud API gateway, **Stripe**…). (**verify** chi tiết hãng.)

---

## ⑤ Code thực hành + cấu hình

**Token bucket** atomic bằng **Redis Lua** (an toàn race, A-RL-004):

```python
# rate_limiter.py — pip install redis ; REDIS_URL qua env
import redis, time

# Lua chạy NGUYÊN TỬ trên Redis: đọc token + thời gian, refill, trừ, ghi lại — KHÔNG bị race.
TOKEN_BUCKET = """
local key      = KEYS[1]
local rate     = tonumber(ARGV[1])   -- token/giây đổ vào
local capacity = tonumber(ARGV[2])   -- dung tích bình (cho phép burst)
local now      = tonumber(ARGV[3])
local tokens   = tonumber(redis.call('hget', key, 'tokens') or capacity)
local last     = tonumber(redis.call('hget', key, 'ts') or now)
local refill   = math.min(capacity, tokens + (now - last) * rate)  -- đổ token theo thời gian trôi
local allowed  = refill >= 1
if allowed then refill = refill - 1 end
redis.call('hset', key, 'tokens', refill, 'ts', now)
redis.call('expire', key, math.ceil(capacity / rate) * 2)
return allowed and 1 or 0
"""

class RateLimiter:
    def __init__(self, url, rate=10, capacity=20):  # 10 req/s, burst tới 20
        self.r = redis.from_url(url)
        self.sha = self.r.script_load(TOKEN_BUCKET)
        self.rate, self.capacity = rate, capacity

    def allow(self, user_id) -> bool:
        try:
            return bool(self.r.evalsha(self.sha, 1, f"rl:{user_id}",
                                       self.rate, self.capacity, time.time()))
        except redis.RedisError:
            return True   # FAIL-OPEN (A-RL-005): ưu availability. Đổi thành False nếu endpoint nhạy cảm.
```

```text
# Cấu hình & scale:
- Atomicity: luôn dùng Lua/INCR; KHÔNG read-modify-write ở app.
- Fail mode: chọn fail-open (public API) hay fail-closed (payment/auth) — ghi rõ trong design.
- Hot key/throughput cao (A-RL-006): thêm local counter sync mỗi ~100ms + shard key + dedicated key cho super-user.
- Đặt limiter ở API gateway/edge để chặn sớm, giảm tải backend.
```

> 🔎 **Vì sao phải gói cả logic vào Lua chứ không làm ở app?** Vì **Lua script chạy nguyên tử trên Redis** — toàn bộ "đọc token → refill → trừ → ghi" diễn ra như **một thao tác không thể chen ngang**. Nếu tách các bước này ra app (đọc về app, tính, ghi lại), bạn lại rơi đúng vào **lost update** của A-RL-004.

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- vì sao **local counter** sai.
- vì sao cần **atomic**.
- **boundary burst**.
- **fail-open vs fail-closed** là trade-off.
- **Redis cũng có thể là hot key**.

📌 **Cần THUỘC:**
- 4 thuật toán (**token bucket / fixed / sliding log / sliding counter**) + đặc tính.
- **INCR/Lua** atomic.

🛠️ **Cần LÀM ĐƯỢC:**
- cài **token bucket** atomic.
- quyết **fail mode**.
- giảm tải **Redis** ở throughput cực lớn.

---

## ⑦ Mental model

> 🎯 **"Một cuốn sổ chung, ghi không tranh."**
> **State trung tâm + atomic op; chọn fail-open/closed có chủ đích; Redis nóng thì local+sync.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** **Token bucket vs fixed vs sliding window** — đặc tính?
→ **bucket** cho burst+smoothing; **fixed** đơn giản nhưng boundary burst; **sliding log** chính xác/tốn RAM; **sliding counter** dung hòa.

**(TB)** **Boundary burst** là gì?
→ dồn cuối+đầu 2 cửa sổ ⇒ tới **~2× limit**; **sliding window** khắc phục.

**(khó)** Rate limiter trên **50 server**: vì sao local sai, fix sao?
→ mỗi server 1 phần traffic ⇒ tổng vượt; cần **Redis trung tâm** hoặc **local+sync**.

**(khó)** **Race** khi 2 server +1 cùng counter — xử lý?
→ **atomic INCR/Lua/transaction**; không **read-modify-write** rời.

**(khó)** **Redis chết** thì làm gì?
→ **fail-open** (public, ưu availability) vs **fail-closed** (nhạy cảm); nêu **trade-off**.

> 🧩 **Thách đố (trick):** *"Em để mỗi server giữ counter trong RAM cho nhanh, đỡ gọi Redis."* Sai ở đâu?
>
> ```
> ❌ Tối ưu latency phá đúng đắn:
> 50 server × (limit mỗi server)  →  tổng cho qua ~50× ý định
>
> ✅ Muốn local cho nhanh thì PHẢI sync về Redis (chấp nhận sai số nhỏ),
>    KHÔNG để mỗi server đếm độc lập
> ```
> 🔎 **Vì sao là bẫy?** Nghe "nhanh hơn" rất hấp dẫn, nhưng nó **phá tính đúng đắn**: 50 server × limit-mỗi-server ⇒ tổng cho qua **gấp ~50×**. Nếu muốn local cho nhanh, phải **sync về Redis** (chấp nhận sai số nhỏ), không để mỗi server đếm độc lập.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Cài `rate_limiter.py`, test **30 request liên tiếp** với **rate=10/s, capacity=20**: quan sát **burst** rồi bị **chặn**.

> ✅ **Đạt khi:** giải thích vì sao **20 request đầu qua (burst)** rồi **nhỏ giọt theo rate**.

**BT2.** Thiết kế rate limiter cho **API gateway** phục vụ **1M QPS toàn cục**, có **super-user**. Nêu cách tránh **Redis thành hot key**.

> ✅ **Đạt khi:** dùng **local+sync, shard key, dedicated key cho super-user, đặt ở edge**.

---

## ⑩ Đọc thêm

- **System Design Interview** (Alex Xu) — *"Design a Rate Limiter"*.
- **Redis docs** — *INCR, Lua scripting*.
- **Stripe blog** — *Scaling your API with rate limiters*.

---

> 🎯 **Khẩu quyết khép bài:** *Một cuốn sổ chung (Redis), ghi nguyên tử (Lua/INCR). Local counter là sai trừ khi có sync. Khi Redis chết, fail-open hay fail-closed là quyết định có chủ đích.*
