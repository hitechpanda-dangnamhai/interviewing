# 🎯 BÀI 4 — Cache ↔ DB Consistency (Nhất quán giữa Cache và DB)
**Phủ:** J-CONSIST-001 → J-CONSIST-009

---

## ① Mục tiêu & vị trí trong mạch

Đây là **bài NẶNG nhất của phần lý thuyết** — và cũng là chỗ **phân biệt một senior thật với người chỉ thuộc lý thuyết**. Bài này lấy **strategy (Bài 2)** + **invalidation (Bài 3)** làm nền để trả lời những câu hỏi khó:

- **Vì sao cache + DB KHÔNG THỂ nhất quán tuyệt đối "miễn phí"?**
- Khi ghi, nên **delete cache** hay **update cache**? (và vì sao câu trả lời là *delete*)
- Thứ tự **"update DB rồi delete cache"** có **race condition** gì? Thứ tự ngược lại thì sao?
- **CDC / Outbox** giữ đồng bộ tin cậy ra sao?

> 💡 **Tinh thần cả bài:** Đừng đi tìm "giải pháp nhất quán hoàn hảo" — **nó không tồn tại miễn phí**. Việc của kỹ sư là **thu nhỏ và kiểm soát cửa sổ stale**, rồi **chọn đúng path nào được phép stale, path nào tuyệt đối không**.

---

## ② Giảng cơ bản → nâng cao

### (a) Bản chất vấn đề: Dual-write *(J-CONSIST-001)*

**Cache và DB là HAI store riêng biệt.** Khi bạn cập nhật dữ liệu, bạn phải đụng tới *cả hai*. Nhưng **không có cách nào cập nhật hai store một cách atomic** (đồng thời, "được ăn cả ngã về không") — đây gọi là **bài toán dual-write (ghi đôi)**.

**Hệ quả không thể tránh:** **luôn tồn tại một cửa sổ thời gian (window)** mà **một store đã đổi, store kia chưa**. Trong cửa sổ đó, ai đọc store chưa đổi sẽ thấy **đồ cũ (stale)**.

**Ẩn dụ — hai cuốn sổ ở hai phòng:**
> Bạn có **sổ chính** (DB) và **sổ phụ** (cache) ở hai phòng khác nhau. Bạn không thể có mặt ở cả hai phòng cùng lúc để sửa đồng thời. Sửa xong sổ chính, đi sang phòng kia sửa sổ phụ — **giữa hai lần sửa đó, hai cuốn sổ ghi khác nhau**. Ai liếc nhằm cuốn chưa sửa → đọc sai.

> 🎯 **Kết luận nền:** Mục tiêu thực tế là **thu nhỏ + kiểm soát window stale**, **không phải phủ nhận nó**. Đây chính là **eventual consistency** (nhất quán *cuối cùng*) — liên quan tới **CAP theorem**. **Không có "nhất quán tuyệt đối, miễn phí".**

---

### (b) Delete vs Update cache *(J-CONSIST-002)* — khi ghi, nên làm gì?

Khi có write, bạn có hai lựa chọn cho cache: **ghi đè giá trị mới vào cache (update)**, hay **xoá key khỏi cache (delete)**. **Câu trả lời chuẩn: DELETE.** Vì sao?

**Vì sao KHÔNG update cache:**
1. **Update dễ race:** hai writer ghi cache theo **thứ tự lệch** so với thứ tự ghi DB → **giá trị cũ có thể đè lên giá trị mới** → **stale "bền"** (nằm lì trong cache, không tự sửa). *(timeline ở mục c)*
2. **Phí ghi:** bạn ghi vào cache một giá trị **có thể không bao giờ được đọc lại** — tốn công và RAM vô ích.

**Vì sao DELETE tốt hơn:**
- Sau khi xoá, **lần đọc kế tiếp sẽ miss → tự nạp lại bản MỚI NHẤT từ DB**. Cache luôn lấy "sự thật" từ nguồn.
- **Đơn giản hơn, ít sai hơn** — bạn không phải lo thứ tự ghi cache.

> 🎯 **Khẩu quyết:** **"Khi ghi: DELETE, đừng UPDATE cache."**

---

### (c) Thứ tự thao tác & Race *(J-CONSIST-003)* — ⭐⭐⭐⭐⭐ phần đinh của bài

Giờ là phần khó nhất. Bạn có 2 thao tác: **update DB** và **delete cache**. Thứ tự nào trước?

#### ❌ Thứ tự XẤU: "Delete cache TRƯỚC, update DB SAU"

Đọc timeline này thật chậm (T1 = writer, T2 = reader):

```
T1: DEL cache                          (cache giờ trống)
T2: đọc cache → MISS → đọc DB
        → lấy giá trị CŨ (vì T1 CHƯA commit DB!)
        → set cache = CŨ
T1: commit DB = MỚI
─────────────────────────────────────
KẾT QUẢ: cache = CŨ, DB = MỚI  ❌
         → stale "BỀN" cho tới khi hết TTL
```

> **Vì sao tệ:** reader chen vào *giữa lúc cache đã xoá nhưng DB chưa commit*, nạp đúng giá trị cũ vào lại cache. Sau đó DB mới commit. Cache giữ đồ cũ **rất lâu**.

#### ✅ Thứ tự AN TOÀN HƠN: "Update DB TRƯỚC, delete cache SAU"

```
T1: commit DB = MỚI
T1: DEL cache
─────────────────────────────────────
KẾT QUẢ: lần đọc sau MISS → nạp MỚI từ DB  ✅
```

**Nhưng — vẫn còn một race HIẾM** (cần đủ điều kiện mới xảy ra):

```
T2: đọc cache → MISS → đọc DB = CŨ   (trước khi T1 commit)
T1: commit DB = MỚI → DEL cache
T2: set cache = CŨ   (set SAU khi T1 đã DEL!)
─────────────────────────────────────
KẾT QUẢ: cache = CŨ  ❌  → stale
```

> **Vì sao race này HIẾM hơn nhiều:** nó cần reader vừa **đọc-DB-trước-commit** *vừa* **set-cache-sau-DEL** — hai điều kiện hẹp phải trùng nhau. Còn race của thứ tự xấu thì xảy ra dễ hơn nhiều.

> 🎯 **Kết luận chuẩn công nghiệp:** **"Update DB trước, delete cache sau"** là **lựa chọn tiêu chuẩn** (chính là **Cache-Aside của AWS**). Nhưng **KHÔNG kín tuyệt đối** — vẫn còn race hiếm, nên cần thêm lưới (TTL, double-delete, versioned set).

---

### (d) Delayed double delete *(J-CONSIST-004)*

**Ý tưởng:** xoá cache **HAI lần** — lần 1 ngay sau update, **lần 2 sau một delay ngắn**. Mục đích: **dọn giá trị cũ mà một reader có thể đã lỡ nạp vào cache trong cửa sổ race** ở mục (c).

```
T1: update DB → DEL cache (lần 1)
    ... (reader xấu số có thể set lại giá trị cũ ở đây) ...
T1: [sau delay] DEL cache (lần 2)  → dọn sạch giá trị cũ đó
```

**⚠️ Hạn chế:**
- **Chọn delay rất khó:** quá ngắn thì lần 2 chạy *trước* khi reader kịp set giá trị cũ (không dọn được gì); quá dài thì *kéo dài window stale*.
- **Vẫn còn xác suất** sai + **thêm phức tạp**.

> 🎯 **Nhớ:** double-delete chỉ **GIẢM** xác suất, **không TRIỆT TIÊU**. Muốn chắc chắn → dùng **versioned / CAS** (mục e) hoặc **bỏ cache** cho path đó.

---

### (e) Race read-miss đồng thời với write *(J-CONSIST-005)*

Đây là **gốc rễ** của race ở (c), phát biểu tổng quát:

```
Reader: MISS → đọc DB = CŨ
        ... (chưa kịp ghi cache) ...
Writer: update DB = MỚI + DEL cache
Reader: ghi giá trị CŨ ĐÈ lên  → stale  ❌
```

**Đây là một dạng của lỗi check-then-act** (kiểm tra rồi hành động, nhưng giữa hai bước có kẻ chen ngang). *(Cross-ref I-RACE.)*

**Cách giảm:**
1. **Versioned / CAS khi set cache** — chỉ ghi cache nếu **version mới ≥ version đang lưu** (*Compare-And-Set* nguyên tử). Reader mang version cũ sẽ **bị từ chối ghi đè**.
2. **Double-delete** (mục d).
3. **TTL ngắn** (giới hạn thời gian stale).

> **Ẩn dụ versioned set:** mỗi giá trị dán một **số thứ tự**. Khi ai đó định cập nhật cache, người gác cổng hỏi *"số của anh có mới hơn số đang treo không?"* — **cũ hơn thì không cho dán đè**. Nhờ vậy bản cũ không bao giờ đè bản mới.

---

### (f) Write-through VẪN stale ở môi trường phân tán *(J-CONSIST-006)*

Một hiểu lầm phổ biến: *"Dùng write-through thì cache↔DB nhất quán tuyệt đối."* **Không.**

Write-through chỉ đảm bảo nhất quán **qua đúng cái cache layer đó**. Nhưng thực tế có nhiều thứ phá vỡ:
- **Nhiều cache node** — các node khác chưa chắc đã đồng bộ.
- **Read replica của DB** — replica có **độ trễ replication**, đọc replica vẫn ra đồ cũ.
- **Đường ghi không qua cache layer** — batch job, service khác ghi thẳng DB.

> 🎯 **Chốt:** **Consistency còn phụ thuộc replication + MỌI đường ghi**, không chỉ phụ thuộc strategy. Đừng tin "write-through = hết lo".

---

### (g) Khi nào CHẤP NHẬN stale *(J-CONSIST-007)*

Không phải path nào cũng cần nhất quán mạnh. **Chọn theo HẬU QUẢ của việc sai**, đừng cache đồng loạt:

| Loại path | Stale có sao không? | Cách làm |
|---|---|---|
| **Hiển thị phụ:** đếm view, gợi ý, số like | 😌 **OK** — sai vài đơn vị không chết | Cache thoải mái |
| **Quyết định: tiền / tồn kho / quyền** | 🚨 **KHÔNG** — sai = quyết định sai | **Đọc nguồn (DB)** hoặc **bỏ cache** cho path đó |

> 🎯 **Nguyên tắc:** Câu hỏi luôn là *"nếu giá trị này cũ, điều tệ nhất xảy ra là gì?"* Tệ → đừng cache. Không sao → cache được.

---

### (h) Read-your-own-writes *(J-CONSIST-008)*

**Vấn đề UX kinh điển:** user **vừa cập nhật** một thứ (đổi avatar, sửa tên), bấm xem lại thì... **vẫn thấy cái cũ** vì đọc trúng cache cũ → *"sao thay đổi của tôi không có?"* → trải nghiệm rất tệ.

**Nguyên tắc read-your-own-writes:** một user phải **luôn nhìn thấy thay đổi của CHÍNH MÌNH** ngay lập tức.

**Cách fix:**
1. **Invalidate ngay sau write của họ** (xoá cache liền sau khi user ghi).
2. **Bypass cache** cho read **ngay sau write** của chính user đó — ví dụ: trong **N giây** sau khi user ghi, cho user đó **đọc thẳng DB** (người khác vẫn đọc cache bình thường).

---

### (i) CDC / Outbox *(J-CONSIST-009)* — ⭐⭐⭐⭐⭐ chuẩn production

Đây là **cách đồng bộ TIN CẬY NHẤT** và là kiến thức "ăn điểm" khi phỏng vấn senior.

**Vấn đề của "xoá cache trong code ghi":** chỉ bắt được ghi **qua app**, và còn rủi ro **"commit DB OK nhưng quên/lỗi xoá cache"**.

**CDC (Change Data Capture):**
> **Đọc thẳng commit log của DB** (công cụ phổ biến: **Debezium**) → **mỗi thay đổi commit phát ra một event** → consumer nghe event và **invalidate cache**.
>
> ✅ **Đảm bảo MỌI commit đều kích hoạt invalidate** — kể cả **ghi ngoài app** (batch, service khác), vì nó nghe ở *tầng DB log*, nơi mọi đường ghi đều đi qua.

**Ẩn dụ CDC:** thay vì bắt từng nhân viên tự nhớ "ghi xong nhớ báo bộ phận cache", bạn đặt **một camera ngay cửa kho** — *bất cứ ai* ra vào (sửa DB) đều bị ghi hình → tự động báo cache. Không ai trốn được.

**Outbox pattern:** để tránh **"ghi DB ok nhưng publish event fail"**:
> **Ghi event vào CÙNG transaction DB** với thay đổi dữ liệu (một bảng `outbox`). Nếu transaction thành công thì *cả data lẫn event* cùng có; nếu fail thì *cả hai* cùng không. Sau đó một **relay** đọc bảng outbox và publish → **không mất event**.

*(verify; cross-ref mục K Outbox/CDC.)*

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **Update cache** khi write | **Delete cache** | Update race → **stale bền** |
| **Delete-cache-TRƯỚC-update-DB** | **Update-DB-TRƯỚC-delete-cache** (+ TTL / double-delete) | Thứ tự kia có race **nạp giá trị cũ** |
| *"Write-through = nhất quán tuyệt đối"* | Vẫn **stale ở multi-node / replica** | **Dual-write + replication** |
| Xoá cache **rải rác trong code** | **CDC / Outbox-based invalidation** | Bắt **mọi commit**, không mất |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case:**
- **Đổi giá sản phẩm** → **update DB → DEL cache** → (tuỳ chọn) **double-delete sau 500ms**.
- **Hồ sơ user vừa sửa** → **bypass cache 5s** cho chính user đó (**read-your-writes**).
- **Nhiều service ghi chung bảng** → **CDC invalidate**, thay vì tin mỗi service tự xoá.

**Pattern ở bigtech:**
- **Facebook** mô tả bài toán cache↔DB ở quy mô lớn trong paper **Memcached**, dùng **"leases"** để chống **stale set** & **thundering herd**.
- Hệ thống hiện đại nghiêng về **CDC / Debezium → invalidate** vì nó **bắt mọi đường ghi**.

---

## ⑤ Code thực hành + cấu hình

**Update-DB-trước-delete-cache + double-delete + versioned set chống race:**

```javascript
// consistency.js
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);
const TTL = 300;

export async function updateProduct(id, patch, db) {
  await db.updateProduct(id, patch);          // 1) DB TRƯỚC
  await redis.del(`product:${id}`);           // 2) DEL cache
  // 3) double-delete: dọn giá trị reader có thể vừa nạp trong race window
  setTimeout(() => redis.del(`product:${id}`).catch(() => {}), 500);
}

// Set cache CÓ VERSION để chống ghi-đè-cũ (J-CONSIST-005)
// Lua: CHỈ set nếu version mới >= version đang lưu (atomic → tránh check-then-act race)
const SET_IF_NEWER = `
local cur = redis.call('HGET', KEYS[1], 'v')
if (cur == false) or (tonumber(ARGV[2]) >= tonumber(cur)) then
  redis.call('HSET', KEYS[1], 'v', ARGV[2], 'data', ARGV[1])
  redis.call('EXPIRE', KEYS[1], ARGV[3])
  return 1
end
return 0`;  // verify Lua/HSET tại redis.io

export async function setVersioned(id, data, version) {
  await redis.eval(SET_IF_NEWER, 1, `product:${id}`,
                   JSON.stringify(data), version, TTL);
}
```

**⚠️ Hai cảnh báo:**
- **`setTimeout` chỉ là minh hoạ** double-delete; **production nên dùng job/queue** để không mất khi process chết giữa chừng.
- **Với tiền / tồn kho:** **đừng cache giá trị quyết định** — **đọc thẳng DB**.

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **Dual-write không atomic** (gốc của mọi stale).
- **Vì sao delete > update**.
- **Race của 2 thứ tự** (delete-trước vs update-trước).
- **Write-through vẫn stale** ở phân tán.
- **Read-your-own-writes**.
- **CDC / Outbox**.

📌 **Cần THUỘC:**
- **Update-DB-TRƯỚC-delete-cache**.
- **Double-delete** = xoá 2 lần (lần 2 có delay).
- **Versioned / CAS set** chống race.

🛠️ **Cần LÀM ĐƯỢC:** code **update-then-delete** đúng thứ tự; **nhận diện path cần bypass cache**; mô tả **luồng CDC → invalidate**.

---

## ⑦ Mental model

> **Cache + DB = dual-write → LUÔN có window stale, chỉ thu nhỏ được.**
> **DELETE (không update), DB trước cache sau, TTL backstop.**
> **Tiền / quyền → đọc DB.**
> **Đồng bộ tin cậy = CDC / Outbox.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(TB)** Vì sao cache + DB không nhất quán tuyệt đối miễn phí? → **dual-write không atomic** → **window stale**.
- **(khó)** Delete vs update cache, vì sao delete an toàn hơn? → **update race** lệch thứ tự → **stale bền**.
- **(rất khó)** "Update DB rồi delete" vs "delete rồi update" — race mỗi cái? → *(trình bày 2 timeline ở mục ②c)*.
- **(khó)** Read-your-own-writes vỡ vì cache thế nào, giữ sao? → đọc trúng **cache cũ**; **invalidate / bypass** ngay sau write của user.
- **(rất khó)** CDC / Outbox giữ cache đồng bộ thế nào, vì sao tin cậy hơn xoá trong code? → bắt **mọi commit từ DB log** → invalidate; **không miss ghi ngoài app**.

> **🧩 Thách đố:** *"Delayed double delete đặt delay = 1ms. Vẫn stale. Vì sao?"*
>
> **Trả lời:** Delay **phải dài hơn** thời gian một reader **đọc-DB-rồi-set-cache** (thường vài chục–vài trăm ms tuỳ tải). **1ms quá ngắn** → lần xoá thứ 2 chạy **trước khi** reader kịp set giá trị cũ → **không dọn được gì**. Nhưng đặt **dài** thì lại **tăng window stale**. → Chính vì thế **double-delete chỉ GIẢM, không TRIỆT TIÊU**; muốn chắc → **versioned / CAS** hoặc **bỏ cache** cho path đó.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Sửa hàm update sai (**delete-trước-update**) thành đúng thứ tự + giải thích race cũ.
> ✅ **Đạt khi:** đổi thành **update-DB → DEL-cache** + nêu đúng **race nạp-giá-trị-cũ**.

**BT2.** Cho path **"trừ tồn kho khi đặt hàng"**: quyết định cache hay không + lý do.
> ✅ **Đạt khi:** **KHÔNG cache** giá trị tồn kho quyết định; **đọc / giảm trên DB** (hoặc atomic store) — **không tin cache**.

---

## ⑩ Đọc thêm

- **AWS Whitepaper** — *Database Caching Strategies Using Redis*, phần **Cache-Aside consistency**.
- **Facebook** — *Scaling Memcache at Facebook* (**leases**, **thundering herd**).
- **Debezium** / **Transactional Outbox pattern** (microservices.io).

---

> 🔎 **AI hay sai ở đây:** AI viết **"update cache = giá trị mới"** hoặc **"delete cache rồi update DB"** — **cả hai đều sinh stale**.
> **Mỗi khi review code cache write, luôn kiểm:** *(1) delete chứ không update? (2) DB trước, cache sau?*
