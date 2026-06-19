# 🎯 BÀI 7 — Redis as Cache (bản chất bên trong)
**Phủ:** J-REDIS-001 → J-REDIS-009

---

## ① Mục tiêu & vị trí trong mạch

Đến giờ ta mới **"mở hộp đen" Redis** — nhìn vào *bên trong* để hiểu nó vận hành thế nào, thay vì chỉ gọi lệnh. Bài này trả lời:

- **Vì sao Redis nhanh DÙ single-thread** (đơn luồng)?
- **Hệ quả vận hành** của single-thread mà dev hay quên là gì?
- **Data structures** nào dùng cho việc gì?
- **Persistence RDB vs AOF** khác nhau ra sao?
- **Pipelining vs Transaction** vs **Lua** — ba thứ dễ lẫn.
- Và **lỗi cache kinh điển AI hay tạo**: dùng *cùng một key cho mọi user*.

Đây là bài **tổng hợp vận hành Redis**, làm nền cho **Bài 8–10**.

> 💡 **Ý chính:** Hầu hết "lỗi Redis" thực ra bắt nguồn từ **một sự thật kiến trúc duy nhất — Redis thực thi lệnh TUẦN TỰ trên một luồng**. Hiểu điều này, bạn tự suy ra được vì sao `KEYS *` nguy hiểm, vì sao big key giết hệ thống, và vì sao Lua phải ngắn.

---

## ② Giảng cơ bản → nâng cao

### (a) Vì sao Redis nhanh DÙ single-thread *(J-REDIS-001)*

Nghe có vẻ nghịch lý: chỉ một luồng mà phục vụ hàng trăm nghìn lệnh/giây? Lý do:

- **In-memory** — dữ liệu nằm trong **RAM**, **không có disk I/O trên đường nóng** (hot path). Đây là nguồn tốc độ lớn nhất.
- **Single-thread → không cần lock, không context-switch.** Vì chỉ một luồng đụng dữ liệu, **mọi thao tác tự nhiên atomic** mà **không phải khoá** gì cả. (Đa luồng phải tốn lock + chuyển ngữ cảnh — Redis bỏ qua được hết.)
- **I/O multiplexing (epoll)** — một luồng vẫn **xử lý nhiều kết nối** hiệu quả bằng cách "ghép kênh" I/O.
- **Data structure tối ưu** (viết bằng **C**, **encoding gọn**).

> 🎯 **Chốt:** phần *có thể* chậm của Redis là **mạng** & **persistence**, **KHÔNG phải execution**. Thực thi lệnh trong RAM cực nhanh.

**Ẩn dụ single-thread atomic:** một **đầu bếp duy nhất** làm từng món **lần lượt**. Vì chỉ có một người, **không bao giờ có chuyện hai người tranh nhau cùng cái chảo** (không cần "khoá chảo"). Mỗi món xong trọn vẹn rồi mới sang món sau → **không có trạng thái nửa vời**.

> 🔍 **Cập nhật (verify):** **Redis 6+** có **I/O threads** (đọc/ghi socket *song song*), **Redis 8.x / Valkey 8.x** còn mạnh hơn → câu *"Redis chỉ 1 core"* **không còn tuyệt đối**. Nhưng **việc thực thi LỆNH vẫn tuần tự** (để giữ tính atomic) — phần song song chỉ là I/O mạng.

---

### (b) Hệ quả vận hành của single-thread *(J-REDIS-002)* — ⭐ dev hay quên

Vì lệnh chạy **tuần tự trên một luồng**, **một command chậm O(n) sẽ BLOCK MỌI client** đang chờ. Cả hàng phải đứng yên vì một người làm lâu.

**Ẩn dụ:** một **quầy thanh toán duy nhất**. Nếu một khách lôi ra **xe đẩy 500 món** (lệnh O(n) lớn), **toàn bộ hàng phía sau đóng băng** cho tới khi xong.

Các lệnh nguy hiểm & cách thay:
- **`KEYS *`** → **dùng `SCAN`** (duyệt theo **cursor**, **non-blocking**, trả từng mẻ).
- **Thao tác big-key** → tránh (Bài 5).
- **Lua nặng** → giữ **ngắn**.

> 🎯 **Hệ quả scale:** throughput thực thi giới hạn **~1 core** → muốn scale phải **shard** (chia khoá ra nhiều node — **Bài 9**), không thể "thêm core" cho một instance.

---

### (c) Redis vs Memcached *(J-REDIS-003)*

| | **Memcached** | **Redis** |
|---|---|---|
| **Mô hình** | Đơn giản, **multi-thread**, **chỉ key-value string** | Giàu **data structure**, **persistence**, **replication**, **pub/sub**, **scripting** |
| **Hợp khi** | Cache thuần đơn giản, cần **multi-core** thuần cho string | Đa năng, cần **structure / HA / persistence** |
| **Lựa chọn mới** | Ít dùng hơn | Phần lớn case mới chọn **Redis** (hoặc **Valkey / DragonflyDB**) |

*(verify)*

> 🎯 **Tóm:** Memcached = **dao gọt** (một việc, sắc, đa lõi). Redis = **dao đa năng** (nhiều structure, HA, persistence). Case mới thường chọn Redis trừ khi *chỉ* cần cache string thuần tốc độ tối đa.

---

### (d) Data structures + use case *(J-REDIS-004)*

Chọn **đúng structure** giúp **cắt được nhiều logic** ở app:

| Structure | Use case cache / đời thật |
|---|---|
| **String** | giá trị serialize (**JSON**), counter (**`INCR`**) |
| **Hash** | object theo field (`HSET user:1 name ... age ...`) |
| **Sorted Set (zset)** | **leaderboard**, rate-limit theo time, **top-N** |
| **Set** | unique / tag, kiểm tra **membership** |
| **List** | queue đơn giản (`LPUSH` / `BRPOP`) |
| **Stream** | event / job log (Bài 10) |
| **Bitmap / HyperLogLog** | đếm / flag, **đếm xấp xỉ unique** (Bài 8) |

> 🎯 **Nguyên tắc:** **đừng tự sort/đếm trong app** khi **zset** đã làm sẵn việc đó *atomic và nhanh*. Chọn đúng structure = ít code hơn, ít race hơn. *(verify)*

**Ví dụ trực giác — leaderboard:** muốn bảng xếp hạng top-10 realtime? Nếu tự làm: kéo hết điểm về app, sort, cắt 10 → chậm + dễ race. Với **zset**: `ZINCRBY` cập nhật điểm, `ZREVRANGE 0 9` lấy top-10 — **Redis lo hết, atomic sẵn**.

---

### (e) Persistence: RDB vs AOF *(J-REDIS-005)*

| | **RDB** (snapshot) | **AOF** (append-only file) |
|---|---|---|
| Cơ chế | **Chụp ảnh định kỳ** toàn bộ data | **Ghi log MỖI write** |
| Ưu | **Gọn**, **restore nhanh** | **Bền hơn** (mất ít data) |
| Nhược | **Mất data** từ snapshot cuối → hiện tại | **File lớn / chậm hơn** |
| Tinh chỉnh | tần suất snapshot | `appendfsync` (`everysec` / `always`) |

> 🎯 **Theo vai trò:**
> - **Cache thuần** → **có thể TẮT persistence** (data **tái tạo được từ DB**) để nhanh nhất.
> - **Store bền** (session/queue) → **cần AOF/RDB**. *(verify)*

**Ẩn dụ:** **RDB** = chụp ảnh gia đình mỗi tối (mất các khoảnh khắc giữa hai lần chụp). **AOF** = quay video liên tục (đầy đủ hơn, nhưng tốn dung lượng).

---

### (f) Pipelining vs Transaction *(J-REDIS-006)* — ⚠️ ĐỪNG LẪN

Hai thứ **mục đích khác nhau hoàn toàn**:

| | **Pipelining** | **MULTI/EXEC (Transaction)** |
|---|---|---|
| Mục đích | **Cắt round-trip mạng (RTT)** | **Đảm bảo atomic** (không xen giữa) |
| Cách | Gộp nhiều command gửi **1 lượt** | Nhóm lệnh chạy **liền mạch** |
| Atomic? | ❌ **KHÔNG** (lệnh khác có thể xen vào) | ✅ Có (không ai xen) |
| Rollback? | — | ❌ **KHÔNG như SQL** (lệnh lỗi runtime → các lệnh khác **vẫn chạy**) |

> 🚨 **Hiểu lầm chí mạng:** **`MULTI/EXEC` KHÔNG rollback kiểu SQL.** Nếu một lệnh trong nhóm lỗi lúc chạy, **các lệnh còn lại VẪN thực thi** — không có chuyện "hoàn tác toàn bộ". Đừng coi nó như transaction của database.

**Ẩn dụ:**
- **Pipelining** = thay vì gửi **10 lá thư riêng** (10 lần đi bưu điện = 10 RTT), bỏ chung **1 phong bì** gửi một lần. Nhanh, nhưng người khác vẫn có thể chen thư vào giữa.
- **MULTI/EXEC** = "khoá phòng làm liền 10 việc không ai chen", nhưng **không có nút Undo** nếu việc thứ 5 hỏng.

---

### (g) Lua script *(J-REDIS-007)*

> Chạy **nhiều thao tác atomic ngay phía server** trong một script — ví dụ *"so sánh token rồi `DEL`"* khi **release lock**, hay **rate-limit atomic**. → **Tránh được race check-then-act** (kiểm tra rồi hành động, kẻ khác chen vào giữa).

**Vì sao Lua atomic được?** Vì Redis chạy nguyên script **trên luồng đơn, không ai xen** — đúng đặc tính single-thread ở mục (a).

> ⚠️ **Rủi ro:** script chạy **lâu** sẽ **block single-thread** (đúng bài (b)) → **giữ Lua NGẮN**. *(Cross-ref I-DLOCK-003.)*

**Ví dụ trực giác — release lock an toàn:** nếu làm 2 bước rời (đọc token → nếu đúng thì `DEL`), giữa hai bước có thể bị chen → xoá nhầm lock của người khác. Gói cả hai vào **một Lua** → **không thể bị chen**.

---

### (h) Đặt TTL & lỗi "quên TTL" *(J-REDIS-008)*

Cú pháp: **`SET key val EX <giây>` / `PX <ms>`** hoặc **`EXPIRE key <giây>`**.

> 🚨 **Quên TTL** → key **tồn tại mãi** → **memory phình**, dễ chạm **`maxmemory`** (Bài 6) + **giữ data stale**.

> 🎯 **Khẩu quyết:** **Mọi cache value NÊN có TTL**, trừ khi có lý do rõ ràng để không. *(verify cú pháp)*

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **`KEYS *`** | **`SCAN`** | `KEYS` **block single-thread** |
| **`DEL` big key** | **`UNLINK`** | `DEL` **block** |
| *"Redis chỉ single-thread, max 1 core"* | **I/O threads** (6+ / 8.x / Valkey) — execution vẫn tuần tự | *"1 core"* không còn tuyệt đối *(verify)* |
| Tự sort/đếm trong app | Dùng **zset / HLL / bitmap** | Structure đúng **cắt logic** |
| Cache value **không TTL** | **Luôn TTL** (trừ lý do rõ) | Tránh **phình memory + stale** |
| Coi **`MULTI/EXEC` như SQL transaction** (rollback) | Hiểu **không rollback**, dùng **Lua** cho atomic logic | **Ngữ nghĩa khác** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case (chọn đúng structure):**
- **leaderboard game** = **zset**.
- **đếm unique visitor** = **HLL**.
- **rate limit** = **zset / `INCR`**.
- **release distributed lock** = **Lua** (compare-token → `DEL`).
- **Cache thuần product** = **string JSON + TTL**, **tắt persistence** instance đó.

**Pattern ở bigtech:**
- **Facebook** quy mô khổng lồ dùng **Memcached** (multi-thread, string thuần) cho **lookaside cache**.
- Nhiều hệ thống mới chọn **Redis / Valkey** vì **data structure + HA**.
- **DragonflyDB** nổi lên như **drop-in đa-core hiệu năng cao**. *(verify landscape)*

---

## ⑤ Code thực hành + cấu hình

**Pipelining + đúng cú pháp TTL + dùng structure đúng:**

```javascript
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

// Pipelining: gộp nhiều lệnh, cắt RTT (⚠️ KHÔNG atomic — lệnh khác có thể xen)
const pipe = redis.pipeline();
pipe.set("product:1", JSON.stringify(p1), "EX", 300);
pipe.set("product:2", JSON.stringify(p2), "EX", 300);
pipe.incr("metrics:writes");
await pipe.exec();

// Transaction atomic (MULTI/EXEC) — ⚠️ KHÔNG rollback kiểu SQL
await redis.multi().incr("a").incr("b").exec();

// Leaderboard bằng zset — Redis lo sort, atomic sẵn
await redis.zincrby("leaderboard:game1", 10, "userA");
const top10 = await redis.zrevrange("leaderboard:game1", 0, 9, "WITHSCORES");

// SCAN thay KEYS để quét AN TOÀN (non-blocking, theo cursor)
let cursor = "0";
do {
  const [next, keys] = await redis.scan(cursor, "MATCH", "product:*", "COUNT", 100);
  cursor = next; /* xử lý keys từng mẻ */
} while (cursor !== "0");
```

> ⚠️ **Không `KEYS *`** trên production.
> ⚠️ **Luôn kèm `EX`/`PX`** khi set cache. *(verify cú pháp lệnh tại redis.io)*

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **Vì sao nhanh** (in-memory + single-thread atomic + **epoll**).
- **Hệ quả O(n) block** mọi client.
- **pipeline ≠ transaction**.
- **Lua atomic**.
- **RDB vs AOF**.
- **structure đúng cắt logic**.

📌 **Cần THUỘC:**
- **`SCAN` thay `KEYS`**, **`UNLINK` thay `DEL`** big key.
- **`SET..EX/PX` / `EXPIRE`**.
- **zset** cho leaderboard.
- **`MULTI/EXEC` KHÔNG rollback**.

🛠️ **Cần LÀM ĐƯỢC:** dùng **pipeline** cắt RTT; **chọn structure đúng**; viết **Lua atomic ngắn**; **cấu hình persistence theo vai trò**.

---

## ⑦ Mental model

> **Redis nhanh vì in-memory + thực thi TUẦN TỰ (atomic, không lock).**
> **Hệ quả:** 1 lệnh chậm block tất cả → **`SCAN` / `UNLINK`, giữ Lua ngắn**.
> **Chọn đúng data structure.**
> **Cache thuần TẮT persistence; store bền BẬT AOF.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(TB)** Vì sao Redis nhanh dù single-thread? → **in-memory + atomic không lock + epoll + structure tối ưu**.
- **(TB)** Hệ quả vận hành của single-thread? → lệnh **O(n) block mọi client** → **`SCAN` / tránh big key / Lua ngắn**; **~1 core**.
- **(TB)** Redis vs Memcached chọn khi nào? → Memcached **đơn giản multi-thread string**; Redis **đa năng structure/HA**.
- **(khó)** Pipelining khác MULTI/EXEC? → pipeline **cắt RTT không atomic**; MULTI **atomic không rollback**.
- **(khó)** RDB vs AOF, cache thuần nên thế nào? → **snapshot vs log**; cache thuần **có thể tắt persistence**.

> **🧩 Thách đố *(J-REDIS-009, ⭐⭐⭐⭐⭐)*:** *"AI viết code cache trả về object đã serialize nhưng dùng CÙNG key cho MỌI user. Lỗi gì, kiểm tra ở đâu?"*
>
> **Trả lời:** **Thiếu chiều phân biệt** (**user / tenant / locale / permission**) trong key → **trả data người này cho người khác** (**rò dữ liệu / sai quyền**). **Kiểm tra:** *key có gồm ĐỦ context ảnh hưởng kết quả không?* Đây là lỗi **correctness/security** AI hay tạo — **lặp lại từ Bài 3 (J-INVAL-004)**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Refactor code dùng `KEYS user:*` + `DEL` big hash thành an toàn.
> ✅ **Đạt khi:** đổi **`KEYS` → `SCAN`**, **`DEL` → `UNLINK` / `HSCAN`**.

**BT2.** Cài **leaderboard top-10** + **đếm unique visitor/ngày** bằng structure Redis đúng.
> ✅ **Đạt khi:** **zset** cho leaderboard, **HLL** (`PFADD` / `PFCOUNT`) cho unique — **không lưu mọi phần tử**.

---

## ⑩ Đọc thêm

- **redis.io/docs** — *Data types, Pipelining, Transactions, Programmability (Lua/Functions), Persistence*.
- **redis.io/blog** — *Redis 8* (AGPLv3, vector sets, I/O).
- **valkey.io** / **dragonflydb.io** (lựa chọn thay thế).

---

> 🔎 **AI hay sai ở đây:** (1) **cùng key cho mọi user**; (2) **`KEYS *` trên prod**; (3) **quên TTL**; (4) **coi `MULTI/EXEC` như rollback SQL**.
> **Mỗi cái là một mục review riêng** khi đọc code cache.
