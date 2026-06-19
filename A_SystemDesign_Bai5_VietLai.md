# 🎯 BÀI 5 — Bottleneck & failure modes ở production

**Phủ:** A-BOT-001 → A-BOT-006

> 💡 **Tư tưởng cốt lõi của cả bài:** Có những lỗi **không bao giờ xuất hiện trên giấy** mà chỉ lộ ra ở **production** dưới tải thật: **hot key**, **cache stampede**, **pool cạn**. Bài này dạy bạn **soi điểm nghẽn (bottleneck)** và **nhận diện các failure mode** đó trước khi chúng nhận diện bạn.

---

## ① Mục tiêu & vị trí trong mạch

Bài này **khép khối "Scaling primitives"**.

- **Bài 3–4** cho bạn các **thành phần** (LB, cache, shard, lego…).
- **Bài 5** dạy cách **soi điểm nghẽn** và nhận diện **failure mode chỉ lộ ở production** (**hot key**, **stampede**, **pool cạn**).

> 🎯 **Mạch nối:** đây là cầu nối tới **Bài 16** (*"đúng trên giấy" vs "đúng ở production"*), và giao nhiều với **mục O** (observability), **mục I** (concurrency), **mục J** (caching).

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác

> **Ẩn dụ:** Một con đường tắc không tắc đều khắp — nó tắc ở **nút cổ chai**: cây cầu hẹp, ngã tư duy nhất mọi xe phải đi qua. Tìm chỗ tắc = tìm **chỗ mọi luồng dồn vào một điểm**.

Khi interviewer hỏi *"bottleneck của thiết kế anh vừa vẽ là gì?"*, họ kiểm tra xem bạn có **nhìn được nơi tải dồn** không.

> 🎯 **Chốt:** **Bottleneck luôn ở tầng có tải/đồng thời cao nhất, hoặc nơi state tập trung** — chỗ **mọi request phải đi qua một điểm**.

---

### (b) Cơ chế — soi theo thứ tự

#### 📘 **A-BOT-001 — Soi từ đâu trước**

Soi tầng có **concurrency cao nhất** hoặc **state tập trung**: **DB ghi**, **single LB**, **cache**, **hot key/partition**.

> ⚠️ **Nguyên tắc:** chỉ rõ **điểm cụ thể + cách giảm**, **không** nói chung chung *"cần scale"*. Câu "cần scale" là câu của người chưa nhìn ra nghẽn ở đâu.

#### 📘 **A-BOT-002 — DB thường là bottleneck đầu tiên**

DB là nơi **ghi/đọc tập trung** nên hay nghẽn trước nhất. Có một **thang nấc giảm tải từ rẻ → đắt**:

```
index  →  cache  →  read replica  →  sharding  →  CQRS / queue
(rẻ nhất)                                          (đắt/phức tạp nhất)
```

> 🚨 **Cấm nhảy thẳng vào shard.** **Sharding** đắt về **vận hành** và **mất join** (dữ liệu nằm rải nhiều máy, không JOIN trực tiếp được nữa).
>
> 🔍 **Vì sao đi theo nấc?** Vì đa số ca tải cao **giải quyết được ở nấc rẻ** — thêm một **index** đúng có khi đủ. Nhảy thẳng sang shard giống như đập nhà xây lại chỉ vì cái cửa kẹt.

#### 📘 **A-BOT-003 — Hot partition / hot key**

> **Ví dụ trực giác:** Một siêu thị có 10 quầy thu ngân, nhưng tất cả khách đều xếp vào *một quầy* vì chỉ quầy đó bán vé số. Chín quầy kia rảnh, một quầy quá tải.

**Hot key/partition** = 1 shard/key nhận **tải lệch** — **celebrity** (1 user triệu follower), **popular item**, hoặc **key thiết kế xấu** kiểu `country=VN` (cả nước dồn một key).

- **Phát hiện:** phân phối lệch — **p99 một shard cao** bất thường.
- **Fix:** **shard key tốt hơn**, **salting** (thêm hậu tố ngẫu nhiên để rải key nóng ra nhiều shard), **dedicated cache** cho key nóng, **replicate read**.

#### 📘 **A-BOT-005 — Connection / thread pool cạn**

*(giao với **E/I**)*

> **Ẩn dụ:** Nhà hàng có 20 bàn (pool). Khách tới đông, hết bàn → khách mới **xếp hàng ngoài cửa**. Hàng dài dần, người chờ bực bỏ đi (timeout), và sự bực lan ra cả phố (**cascading**).

**Request xếp hàng chờ connection** ⇒ **latency tăng** ⇒ **timeout lan cả hệ (cascading)**.

```
T1  pool đầy (20/20 connection đang dùng)
       ↓
T2  request mới phải CHỜ connection            → latency bắt đầu tăng
       ↓
T3  chờ quá lâu → TIMEOUT → request retry        → càng nhiều request đổ vào
       ↓
T4  service gọi nó cũng timeout theo             → ❌ CASCADING: chết dây chuyền
```

> 💡 **Fix:** chỉnh **pool size** theo **DB limit**; **tách pool** theo loại tải; dùng **circuit breaker** để **cắt sớm** thay vì kéo chết dây chuyền.

#### 📘 **A-BOT-006 — Thundering herd / cache stampede**

*(giao với **J**)*

> **Ẩn dụ:** Cửa hàng mở cửa lúc 8h. Nếu *tất cả* khách dồn vào đúng giây mở cửa → nhân viên (DB) bị giẫm đạp. Nếu khách đến **rải rác** → ổn.

**Nhiều key cùng hết hạn** ⇒ request **đồng loạt miss** dồn vào DB cùng lúc:

```
❌ stampede (TTL đồng loạt):
T0  cache nóng, mọi key cùng TTL=8h00
       ↓
T1 (8h00)  TẤT CẢ key hết hạn cùng lúc
       ↓
T2  100.000 request cùng MISS → cùng lao xuống DB → ❌ DB sập

✅ có chống stampede:
T1  key hết hạn → CHỈ 1 request đi tính (single-flight), số còn lại CHỜ kết quả
       ↓
T2  + TTL có jitter → key hết hạn rải rác, không dồn một giây
```

> 💡 **Fix:** **lock / single-flight** (chỉ 1 request đi tính, số còn lại chờ), **jittered TTL** (rải hạn ngẫu nhiên), **stale-while-revalidate** (trả bản cũ trong lúc làm mới), **pre-warm cache** (làm nóng trước giờ cao điểm).

---

### (c) Mép giới hạn — debug đúng cách 📘 **A-BOT-004** *(giao với O)*

> 🎯 **Nguyên tắc vàng:** *"Hệ chậm ở production"* — **đo trước, đừng đoán.**

```
metrics / tracing / p99  →  tìm tầng latency cao  →  soi queue depth / CPU / connection pool / DB slow query
```

> 🔎 **Vì sao data-driven mới đúng?** Đây là khác biệt giữa *"đoán mò vá lung tung"* và một **TL (tech lead) điều tra có phương pháp**. Sửa theo cảm tính thường vá nhầm tầng, làm hệ phức tạp thêm mà bệnh vẫn còn.

> 💡 **Phân biệt 📘 vs ➕:** các **failure mode** đều trong bộ đề (📘). Ý *"circuit breaker cắt cascading"* và *"single-flight"* là ➕ làm sâu.

---

## ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (❌) | ✅ Nên làm | Vì sao |
|---|---|---|
| *"Hệ chậm thì thêm server"* | **Đo trước** (p99, tracing) rồi sửa đúng tầng | Thêm server không cứu **hot key / pool cạn / DB lock** |
| Gặp tải cao là **shard DB** ngay | index → cache → replica → mới shard | **Shard** đắt; đa số ca giải quyết ở nấc rẻ hơn |
| Đặt **TTL** bằng nhau cho mọi key | **Jittered TTL + single-flight** | TTL đồng loạt ⇒ **stampede** khi hết hạn cùng lúc |
| **Pool** to vô tội vạ "cho chắc" | Pool theo **DB limit** + **circuit breaker** | Pool quá lớn làm **sập DB**; quá nhỏ gây nghẽn |

> 🔍 **Đào sâu dòng cuối:** Vì sao pool *quá lớn* lại sập DB? Vì mỗi connection là một "khe" tài nguyên ở phía DB. App mở 1000 connection trong khi DB chỉ chịu được 200 → DB bị bão hoà connection và gục. Pool không phải "càng to càng an toàn" — nó phải **khớp với DB limit**.

---

## ④ Áp dụng thực tế + So sánh bigtech

- **Celebrity hot key** là failure mode kinh điển ở mạng xã hội (1 user/triệu follower, hoặc 1 post viral) — giải bằng **dedicated cache** + (ở feed) **hybrid fan-out** *(Bài 11)*.
- **Cache stampede** xảy ra mỗi khi một cache phổ biến **hết hạn đồng loạt sau deploy/restart**; mọi hệ **read-heavy** đều phải có **chống stampede**.
- **Cascading failure** do **pool cạn** là nguyên nhân nhiều outage lớn; **circuit breaker + timeout + bulkhead** là pattern phòng vệ phổ biến (**Netflix Hystrix** là ví dụ lịch sử; nay thường dùng **resilience4j / service mesh** — **verify** đời công cụ).

---

## ⑤ Code thực hành + cấu hình

**Single-flight** chống **cache stampede** (Python, dùng asyncio lock — minh hoạ ý tưởng):

```python
# single_flight.py — gộp các request cùng miss vào 1 lần tính DB
import asyncio, random, time

cache = {}                       # demo in-memory; thực tế là Redis
locks = {}                       # 1 lock / key
inflight_db_calls = 0

async def load_from_db(key):     # giả lập query DB tốn 200ms
    global inflight_db_calls
    inflight_db_calls += 1
    await asyncio.sleep(0.2)
    return f"value-of-{key}"

async def get(key, ttl=5):
    now = time.time()
    item = cache.get(key)
    if item and item["exp"] > now:
        return item["val"]                       # ✅ cache HIT → trả luôn
    lock = locks.setdefault(key, asyncio.Lock())
    async with lock:                              # SINGLE-FLIGHT: chỉ 1 coroutine được vào DB
        item = cache.get(key)                     # double-check sau khi giành lock
        if item and item["exp"] > now:            # (có thể coroutine trước đã điền cache rồi)
            return item["val"]
        val = await load_from_db(key)             # chỉ 1 lần MISS thật sự xuống DB
        jitter = random.uniform(0, ttl * 0.2)     # JITTERED TTL: rải hạn, chống hết hạn đồng loạt
        cache[key] = {"val": val, "exp": now + ttl + jitter}
        return val

async def main():
    await asyncio.gather(*[get("hot-key") for _ in range(100)])  # 100 request CÙNG miss
    print(f"100 request đồng thời nhưng chỉ {inflight_db_calls} lần gọi DB")  # kỳ vọng: 1

asyncio.run(main())
```

> 🔎 **Điểm cốt lõi của double-check:** sau khi một coroutine **giành được lock**, nó phải **kiểm tra cache lại lần nữa** — vì rất có thể trong lúc nó *chờ* lock, một coroutine khác đã tính xong và điền cache. Không double-check = vẫn gọi DB thừa.

```text
# Cấu hình chống các failure mode (checklist):
- TTL: thêm jitter (±10–20%); bật stale-while-revalidate nếu hệ chịu được data hơi cũ.
- Pool: maxConnections theo (DB max_connections × % an toàn); tách pool đọc/ghi; đặt acquire-timeout ngắn.
- Circuit breaker: mở khi error-rate/latency vượt ngưỡng; half-open thử lại; tránh cascading.
- Observability: bật tracing + p99 dashboard + alert trên queue depth & pool saturation (giao với O).
```

---

## ⑥ Keywords cần nhớ

🧠 **Cần HIỂU:**
- *"đo trước khi sửa"*.
- **cascading failure** từ **pool cạn**.
- vì sao **hot key** phá **sharding**.
- vì sao **TTL đồng loạt** gây **stampede**.

📌 **Cần THUỘC:**
- nấc giảm tải DB: **index → cache → replica → shard → CQRS**.
- fix **hot key**: **salting / dedicated cache**.
- fix **stampede**: **single-flight / jitter / SWR / pre-warm**.
- **circuit breaker**.

🛠️ **Cần LÀM ĐƯỢC:**
- chỉ **đúng bottleneck** của một sơ đồ.
- **debug data-driven**.
- viết **single-flight + jittered TTL**.

---

## ⑦ Mental model

> 🎯 **"Bottleneck nằm ở chỗ state tập trung / tải dồn."**
> **Đo trước, sửa nấc rẻ trước, chặn cascading bằng circuit breaker.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** *"Bottleneck của thiết kế này là gì?"* — soi từ đâu?
→ tầng **concurrency cao nhất / state tập trung**; chỉ điểm cụ thể + cách giảm.

**(TB)** **DB** là bottleneck — các nấc giảm tải theo thứ tự?
→ **index → cache → read replica → sharding → CQRS/queue**.

**(khó)** **Hot key** là gì, fix sao?
→ 1 key/shard tải lệch; **salting**, **dedicated cache**, **shard key tốt hơn**, **replicate read**.

**(khó)** **Cache stampede** phòng sao?
→ **single-flight/lock**, **jittered TTL**, **stale-while-revalidate**, **pre-warm**.

**(rất khó)** *"Hệ chậm ở production"* — debug từ đâu?
→ đo **p99/tracing** trước; tìm tầng latency cao; soi **queue/CPU/pool/slow query**; **data-driven**.

> 🧩 **Thách đố (trick):** Thêm cache xong hệ vẫn **sập mỗi sáng 8h khi cache nguội**. Vì sao, fix?
>
> ```
> ❌ Niềm tin: "thêm cache là xong"
>
> Thực tế:
> đêm restart/deploy  →  cache COLD (rỗng)
>          ↓
> 8h sáng cao điểm    →  nhiều key cùng MISS + nhiều key cùng hết hạn
>          ↓
>                     →  request dồn xuống DB cùng lúc → ❌ STAMPEDE → sập
>
> ✅ Fix: pre-warm trước giờ cao điểm + jittered TTL + single-flight
> ```
> 🔎 **Vì sao là bẫy?** Nghĩ "cache là xong" mà quên **cache cold** (vừa restart, rỗng) + **nhiều key hết hạn cùng lúc** ⇒ **stampede** + miss dồn DB.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Chạy `single_flight.py`; **tắt lock** (cho mỗi request tự load) và so số lần gọi DB.

> ✅ **Đạt khi:** giải thích vì sao **có lock thì 100 request ⇒ 1 lần DB** (và không lock thì ~100 lần).

**BT2.** Cho sơ đồ *"client → LB → app → Postgres"* read-heavy bị chậm lúc cao điểm. Liệt kê **4 nghi phạm bottleneck** và cách **đo** từng cái.

> ✅ **Đạt khi:** nêu được **DB slow query**, **pool saturation**, **cache miss/stampede**, **hot key** — kèm **metric** để xác nhận từng cái.

---

## ⑩ Đọc thêm

- **Google SRE Book** — chương *Addressing Cascading Failures*.
- **AWS Builders' Library** — *Avoiding fallback in distributed systems* & *timeouts/retries/backoff with jitter*.
- **resilience4j docs** (**circuit breaker**, **bulkhead**).

> ⚠️ **Lưu ý (verify):** **verify** đời công cụ.

---

> 🎯 **Khẩu quyết khép bài:** *Nghẽn nằm ở chỗ tải dồn. Đo trước khi sửa. Giảm tải từ nấc rẻ. Rải TTL, gộp single-flight, chặn cascading bằng circuit breaker.*
