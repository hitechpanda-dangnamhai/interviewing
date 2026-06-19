# 🎯 BÀI 8 — Redis ngoài Cache (Redis beyond cache)
**Phủ:** J-BEYOND-001 → J-BEYOND-006

---

## ① Mục tiêu & vị trí trong mạch

**Redis không chỉ là cache.** Nhờ **data structures (Bài 7)** + **tính atomic** + **in-memory**, *một* hạ tầng có thể đảm nhận nhiều việc: **rate limiter**, **distributed lock**, **session store**, **leaderboard**, **pub/sub**, **đếm xấp xỉ**.

Bài này giúp bạn **nhận diện các use case** đó + **lưu ý riêng của mỗi cái** (đặc biệt **eviction / persistence**), và **trỏ sang mục I / F / M** nơi đào sâu từng phần.

> 💡 **Ý chính:** Sức mạnh "đa năng" của Redis cũng là cái bẫy. Mỗi use case có một yêu cầu **"được mất hay không được mất"** khác nhau. Nhồi tất cả vào một instance với policy cache là cách nhanh nhất để **đá nhầm session/lock/job** ra ngoài.

---

## ② Giảng cơ bản → nâng cao

### (a) Vì sao MỘT hạ tầng làm được NHIỀU việc *(J-BEYOND-001)*

Ba tính chất của Redis khiến nó "lấn sân":
- **In-memory nhanh** — đủ nhanh cho đường nóng.
- **Atomic ops** — các thao tác như `INCR`, `ZADD` *tự nguyên tử* (Bài 7), rất hợp cho đếm/khoá/xếp hạng nhiều client cùng lúc.
- **Đã có sẵn** — đa số hệ thống *đã chạy Redis* cho cache → tận dụng luôn, đỡ thêm một hạ tầng mới.

Từ đó Redis làm được: **rate limiter**, **distributed lock**, **session store**, **leaderboard (zset)**, **pub/sub**, **queue/streams**, **geospatial**, **đếm xấp xỉ (HLL)**.

> 🚨 **Nhưng — mỗi use case có lưu ý riêng**, đặc biệt về **eviction & persistence**: **đừng để evict nhầm thứ "KHÔNG-ĐƯỢC-MẤT"** (session/lock/job). Đây là sợi chỉ đỏ xuyên cả bài.

**Ẩn dụ:** Redis như **con dao Thụy Sĩ** — mở bia, cắt dây, vặn ốc đều được. Nhưng dùng *lưỡi cắt* để *vặn ốc* thì hỏng việc. Mỗi lưỡi (use case) có cách dùng và rủi ro riêng.

---

### (b) Rate limiter ở mức store *(J-BEYOND-002)*

> **Rate limiter** = giới hạn số request mỗi client trong một khoảng thời gian (vd 100 req/phút). Redis làm phần **lưu trạng thái đếm** này tốt vì nó **atomic** và **share state giữa các instance**.

- **Fixed window:** **`INCR` + `EXPIRE`** — đếm trong một "ô thời gian" cố định.
- **Sliding window:** **sorted set / Lua** — chính xác hơn, mượt hơn.

> 🔍 **Phân vai:** *thuật toán* (**token bucket / sliding window**) và *đặt ở đâu* (**gateway vs app**) được **đào sâu ở F-RL, M**. Ở Bài J này **chỉ chạm góc store** (Redis giữ bộ đếm). *(verify cú pháp)*

**Vì sao phải atomic + share state?** Vì nếu mỗi instance app tự đếm riêng trong RAM của nó, một user gọi qua 5 instance = được đếm 5 lần riêng → **lọt giới hạn gấp 5**. Redis là **một nơi đếm chung** → đúng tổng.

---

### (c) Leaderboard / Top-N *(J-BEYOND-003)*

> **zset (sorted set)** giữ phần tử kèm **score** và **luôn có thứ tự sẵn**:
> - **`ZADD` / `ZINCRBY`** — thêm/cộng điểm, độ phức tạp **O(log n)**.
> - **`ZREVRANGE`** — lấy **top-N** nhanh.
> - **`ZRANK` / `ZREVRANK`** — lấy **hạng** của một phần tử.

**Vì sao hợp hơn DB?** Vì DB phải **sort lại** (hoặc quét index) mỗi lần lấy bảng xếp hạng, còn **zset giữ sẵn thứ tự** → lấy top-N gần như tức thì, **không re-sort toàn bộ** → **realtime leaderboard cực hợp**. *(verify)*

**Ví dụ trực giác:** 1 triệu người chơi, cứ ghi điểm là cập nhật. Với DB: mỗi lần xem top-10 phải `ORDER BY score LIMIT 10` trên cả triệu hàng. Với zset: điểm đã được "treo đúng chỗ" ngay khi ghi → `ZREVRANGE 0 9` là xong.

---

### (d) Session store *(J-BEYOND-004)*

> Lưu **session** trong Redis cho phép **share giữa các instance** (không cần **sticky session** — không phải ghim user vào đúng một node) + **nhanh**.

> 🚨 **Cạm bẫy lớn nhất:** session **"KHÔNG ĐƯỢC MẤT"**. Nếu instance đó đặt **`allkeys-*`**, khi đầy bộ nhớ Redis sẽ **đuổi nhầm session key** → **user bị đăng xuất ngẫu nhiên** (Bài 6).

> 🎯 **Đúng cách:** **TTL = session timeout** + **policy phù hợp / instance riêng** (`noeviction` cho instance session). *(Cross-ref M session góc security, J-EVICT-005.)*

---

### (e) Distributed lock *(J-BEYOND-005)* — ⚠️ chỉ NHẬN DIỆN, đừng tự tin

> **Distributed lock** = khoá phân tán để chỉ một tiến trình được làm một việc tại một thời điểm. Sơ bộ: **`SET NX PX` + token**, **release bằng compare-token (Lua)**.

> 🚨 **Cảnh báo:** Redis-as-lock **trông đơn giản nhưng KHÔNG đơn giản**. Có hàng loạt cạm bẫy: **TTL** (lock hết hạn giữa chừng), **failover** (master chết, lock "nhân bản" sai), **fencing token**. Toàn bộ phân tích **Redlock / fencing** nằm ở **mục I (I-DLOCK)**.
>
> Ở đây chỉ cần khắc cốt: **đừng để AI (hay chính mình) viết distributed lock "cho nhanh"** — nó là một use case **đầy bẫy**. *(Cross-ref I-DLOCK-002/005/006.)*

**Vì sao compare-token khi release?** Nếu chỉ `DEL` lock mà không kiểm token, bạn có thể **xoá nhầm lock của người khác** (lock của bạn đã hết hạn, người kế tiếp đã chiếm lock, bạn `DEL` đúng lock của họ). Gói "so token rồi DEL" vào **Lua** để **atomic** (Bài 7).

---

### (f) HyperLogLog / Bitmap *(J-BEYOND-006)*

> **Đếm unique** (vd số khách *khác nhau*) bằng cách lưu mọi id vào một **Set** sẽ tốn **O(n) RAM** — 100 triệu khách = 100 triệu id trong RAM.

> **HyperLogLog (HLL)** **ước lượng cardinality** (số phần tử khác nhau) với **bộ nhớ CỐ ĐỊNH cực nhỏ** (vài KB), **sai số ~0.81%** — bất kể 1 nghìn hay 1 tỷ phần tử.

| | **Set thường** | **HyperLogLog** |
|---|---|---|
| Kết quả | **Chính xác tuyệt đối** | **Xấp xỉ** (~0.81% sai số) |
| RAM | **O(n)** — tăng theo số phần tử | **Cố định**, cực nhỏ |
| Khi nào dùng | Cần con số chính xác / cần truy xuất từng phần tử | Chỉ cần *đếm xấp xỉ* số unique |

> 🎯 **Bản chất:** **chấp nhận xấp xỉ để đổi lấy memory.** Với "đếm unique visitor" thì sai 0.81% không ai quan tâm, nhưng tiết kiệm RAM khổng lồ.

**Bitmap** dùng cho **đếm/flag theo user-id liên tục** (vd **daily active** — bit thứ i bật nghĩa là user i hoạt động hôm nay). *(verify)*

**Ẩn dụ HLL:** như **ước lượng số người trong sân vận động** bằng cách lấy mẫu một khu rồi suy ra — không đếm từng đầu người, nhưng ra con số "đủ đúng" với chi phí rẻ.

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **Sticky session** ở load balancer | **Session store Redis** share | Không khoá user vào 1 node, **scale ngang** |
| Đếm unique bằng **Set** (lưu mọi id) | **HyperLogLog** (`PFADD`/`PFCOUNT`) | Set **O(n) RAM**; HLL **cố định nhỏ** |
| **Sort leaderboard trong DB** mỗi lần | **zset** (`ZINCRBY`/`ZREVRANGE`) | **O(log n)**, không re-sort toàn bộ |
| Lock tự chế **`SETNX` rồi `DEL`** | **`SET NX PX` + compare-token Lua** (chi tiết I-DLOCK) | Tránh **xoá nhầm lock của người khác** |
| Chung instance cho **session + cache `allkeys-lru`** | **Tách** / `volatile-*` + protect | Tránh **evict nhầm session** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case:**
- App **scale ngang** → **session ở Redis riêng** (`noeviction`/TTL).
- **Game realtime** → **leaderboard zset**.
- **API public** → **rate limit Redis** (`INCR`/zset) ở **gateway**.
- **Analytics** → **HLL** đếm **DAU/unique**.

**Pattern ở bigtech:**
- **Redis/Valkey** là **"dao đa năng"** phổ biến cho **rate-limit, session, leaderboard** ở nhiều công ty.
- **HLL** được dùng rộng cho **đếm cardinality lớn** (vd unique view). *(verify khi cần số liệu cụ thể)*

---

## ⑤ Code thực hành + cấu hình

```javascript
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

// Rate limit FIXED WINDOW (góc store; thuật toán đầy đủ xem F-RL)
async function allow(userId, limit = 100, windowSec = 60) {
  // key gắn "ô thời gian" → mỗi window là một key riêng
  const k = `rl:${userId}:${Math.floor(Date.now() / 1000 / windowSec)}`;
  const n = await redis.incr(k);                 // atomic: đếm chung mọi instance
  if (n === 1) await redis.expire(k, windowSec); // chỉ set TTL ở lần đầu của window
  return n <= limit;
}

// Leaderboard — zset giữ thứ tự sẵn
await redis.zincrby("lb:season1", 50, "userA");
const top  = await redis.zrevrange("lb:season1", 0, 9, "WITHSCORES"); // top-10
const rank = await redis.zrevrank("lb:season1", "userA");             // hạng (0-based)

// Unique visitors bằng HLL — RAM cố định, kết quả xấp xỉ
await redis.pfadd("uv:2026-06-18", "userA", "userB");
const approxUnique = await redis.pfcount("uv:2026-06-18");
```

> ⚠️ **Session / lock / queue:** dùng **instance/policy phù hợp** (**đừng `allkeys-*`**).
> ⚠️ **Distributed lock đúng → I-DLOCK**, **không tự chế**.

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **Vì sao 1 hạ tầng đa năng** (nhanh + atomic + sẵn có).
- **Mỗi use case có lưu ý eviction/persistence**.
- **HLL** đánh đổi **xấp xỉ ↔ RAM**.
- **Vì sao zset hợp leaderboard**.
- **Session không được evict**.

📌 **Cần THUỘC:**
- **rate-limit store** = **`INCR`+`EXPIRE`** / **zset**.
- **leaderboard** = **zset**.
- **unique** = **HLL** (`PFADD`/`PFCOUNT`).
- **lock** = **`SET NX PX` + Lua** (chi tiết I).

🛠️ **Cần LÀM ĐƯỢC:** cài **rate-limit store**, **leaderboard zset**, **HLL counter**; biết **khi nào trỏ sang I / F / M**.

---

## ⑦ Mental model

> **Redis = dao đa năng** (nhanh + atomic + sẵn có): **rate-limit, lock, session, leaderboard, pub/sub, HLL**.
> **Nhưng** thứ **"KHÔNG ĐƯỢC MẤT"** (session/job/lock) cần **policy/instance RIÊNG** — **đừng để cache evict nhầm**.

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Ngoài cache, Redis hay làm gì? → **rate-limit, lock, session, leaderboard, pub/sub, queue, HLL**.
- **(TB)** Vì sao zset hợp leaderboard? → **score có thứ tự**, `ZINCRBY`/`ZREVRANGE` **O(log n)**, **không re-sort**.
- **(TB)** Session store Redis lợi gì, eviction phải thế nào? → **share không sticky**; **không `allkeys-*`** (mất session).
- **(khó)** HLL vs set thường để đếm unique? → **HLL xấp xỉ RAM cố định**; **set chính xác nhưng O(n) RAM**.
- **(khó)** Vì sao Redis-as-lock "không đơn giản"? → **TTL / failover / fencing** → xem **I-DLOCK**.

> **🧩 Thách đố:** *"Team dùng CÙNG một Redis (`allkeys-lru`) cho cache, session đăng nhập, và rate-limit. Người dùng thỉnh thoảng bị đăng xuất ngẫu nhiên. Vì sao?"*
>
> **Trả lời:** Khi **memory đầy**, **`allkeys-lru` evict CẢ session key** (vốn không được mất) → user **mất session** → **bị logout**. **Sửa:** **tách instance session** (`noeviction`/TTL) khỏi cache, hoặc tối thiểu cho session key thuộc **nhóm protected**. → **Đúng bệnh J-EVICT-005 / J-BEYOND-004.**

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Cài đếm **DAU (daily active users)** bằng **HLL** theo ngày.
> ✅ **Đạt khi:** **`PFADD`** theo `uv:YYYY-MM-DD`, **`PFCOUNT`**, giải thích **xấp xỉ + RAM cố định**.

**BT2.** Thiết kế hạ tầng Redis cho app có **cache + session + rate-limit**.
> ✅ **Đạt khi:** **tách session** ra policy/instance an toàn, **cache `allkeys-lru`**, **rate-limit store atomic**.

---

## ⑩ Đọc thêm

- **redis.io/docs** — *Sorted sets, HyperLogLog, Bitmaps, Pub/Sub*.
- Mục **I-DLOCK** (distributed lock), **F-RL / M** (rate limiting), **M** (session security) trong project.

---

> 🔎 **AI hay sai ở đây:** AI **nhét mọi thứ vào một Redis** với policy cache.
> **Luôn tự kiểm:** *key nào KHÔNG-ĐƯỢC-MẤT?* → **tách khỏi eviction**. Và **đừng để AI tự viết distributed lock "đơn giản"** — **trỏ về I-DLOCK**.
