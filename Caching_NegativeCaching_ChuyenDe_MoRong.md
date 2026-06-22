# 🚫 NEGATIVE CACHING — Chuyên đề mở rộng (Caching · nối tiếp Bài 1)
> **Cache cả kết quả "KHÔNG TỒN TẠI" để khỏi đập nguồn lặp lại tìm thứ không có** · Keyword · Concept · Giải pháp · Case study · Staff-level
> (Đọc sau khi đã nắm: *miss/hit · TTL · invalidate · cache penetration 穿透 · Bloom filter · stampede*)

---

## ① ĐỊNH VỊ — Negative caching bịt "lỗ thủng đi thẳng xuống DB"

Bài 1 dạy: cache trả lời thay DB cho những thứ **có**. Nhưng có một loại request **cache thường KHÔNG che được**: hỏi thứ **không tồn tại**.

> Với key **không có giá trị**, cache *không có gì để lưu theo kiểu thông thường* → **mọi** request loại này đều **miss → đập thẳng DB** để rồi nhận lại "không có". Lặp lại mãi. Đây chính là **cache penetration (穿透)** ở chuyên đề Cache stampede.

**Định nghĩa:** **Negative caching = lưu lại sự VẮNG MẶT** — "đã tra rồi, không có gì cả" — để lần sau trả ngay từ cache thay vì lại đi hỏi nguồn.

> 🔑 **Cú lật tư duy:** Bình thường **miss = "đi hỏi nguồn"**. Nhưng nếu nguồn **cũng** không có → mỗi request là một vòng đi-về **lãng phí** chỉ để học lại "vẫn không có". Negative caching biến cái đó thành một **HIT trả lời "không có"** → DB được tha.

🎬 *Câu thần chú:* **"Tra rồi, rỗng — thì NHỚ là rỗng, đừng đi tra lại."**

### Ba trạng thái (thay vì hai)

Khi có negative caching, một lần tra cache cho ra **3** khả năng, không phải 2:

| Trạng thái | Nghĩa | Hành động |
|---|---|---|
| **HIT dương** | Có giá trị trong cache | Trả luôn |
| **HIT âm (sentinel)** | **Biết chắc là KHÔNG có** | Trả "không tồn tại" — **không đụng DB** |
| **MISS thật** | **Chưa biết** | Đi hỏi nguồn |

> ⚠️ Phải **phân biệt rạch ròi** "HIT âm" (đã biết là rỗng) với "MISS thật" (chưa biết). Nhầm hai cái này là bug kinh điển (mục ⑧).

---

## ② BÀI TOÁN GỐC — Vì sao "không tồn tại" lại nguy hiểm

Không có negative caching, **path "not found" là một lỗ thủng không được bảo vệ** đi thẳng xuống DB:

```
Hỏi key KHÔNG tồn tại
   → cache không có gì để trả (không lưu positive được)
   → MISS → xuống DB → DB trả "không có"
   → KHÔNG cache lại "không có" → lần sau LẶP LẠI y hệt
   → mọi request loại này đều đập DB (penetration)
```

Hai nguồn gốc của loại request này:

- **Vô tình:** link gãy, item đã xoá, gõ nhầm ID, client poll một resource **chưa được tạo** ("đợi đơn hàng xuất hiện"), crawler quét ID tuần tự.
- **Cố ý (tấn công):** kẻ xấu **liệt kê hàng loạt ID ngẫu nhiên không tồn tại** → cố tình **né cache** để **DDoS thẳng DB**. Đây là kịch bản đáng sợ nhất, và là lý do negative caching gắn liền **an ninh**.

> Nối Bài 1: cache sinh ra để **giữ DB QPS xuống**; thiếu negative caching thì đúng cái path "not found" trở thành **đường tắt phá vỡ sự bảo vệ đó**.

---

## ③ CƠ CHẾ — Sentinel / tombstone & TTL ngắn

Cách làm: lưu một **dấu hiệu đặc biệt (sentinel / tombstone)** đánh dấu "đã biết là rỗng", kèm **TTL riêng, thường NGẮN HƠN** positive.

```javascript
// Negative caching với sentinel rõ ràng + TTL ngắn cho "vắng mặt"
const NEG = "__MISS__"; // sentinel: KHÁC null/undefined để khỏi nhầm với "miss thật"
const NEG_TTL = 30;     // ⭐ negative TTL NGẮN (30s) — vì "không có" có thể thành "có" bất cứ lúc nào
const POS_TTL = 300;    // positive TTL dài hơn

async function getUser(id, loadFromDb) {
  const cached = await redis.get(`user:${id}`);
  if (cached === NEG) return null;             // HIT ÂM → biết chắc không có, KHÔNG đụng DB
  if (cached !== null) return JSON.parse(cached); // HIT DƯƠNG

  const row = await loadFromDb(id);            // MISS THẬT → hỏi DB
  if (row) {
    await redis.set(`user:${id}`, JSON.stringify(row), "EX", POS_TTL);
    return row;
  }
  await redis.set(`user:${id}`, NEG, "EX", NEG_TTL); // ⭐ cache cả SỰ VẮNG MẶT (TTL ngắn)
  return null;
}
```

> 🔑 **Hai quy tắc vàng nằm ngay trong đoạn code:**
> 1. **Negative TTL NGẮN** — vì *vắng mặt là tạm thời*: resource có thể được tạo ngay sau đó; bạn không muốn nói "không có" cho một sản phẩm vừa lên kệ 1 giây trước.
> 2. **Sentinel KHÁC `null`/miss** — để không nhầm "đã biết rỗng" với "chưa tra".

---

## ④ TWO PILLARS — Negative entry vs BLOOM FILTER

Có **hai công cụ** chống penetration, chọn theo tình huống:

| | **Negative entry (sentinel/TTL)** | **Bloom filter** |
|---|---|---|
| Bản chất | Lưu **từng key rỗng** một cách chính xác | **Tập xác suất**: "chắc chắn KHÔNG có" / "có thể có" |
| Bộ nhớ | 1 entry mỗi key rỗng → **tốn nếu key đa dạng** | **Vài bit mỗi key** → cực gọn |
| Chống liệt-kê-ngẫu-nhiên (tấn công) | ❌ Kém — mỗi ID lạ tạo 1 entry → **nổ bộ nhớ** | ✅ Tốt — bao nhiêu ID lạ cũng vẫn vài bit |
| Chính xác | Chính xác | **False positive** (vài key lạ lọt qua), **không false negative** |
| Cập nhật/xoá | Dễ (TTL/del) | Khó xoá (cần **Counting Bloom / Cuckoo filter**) |

### Bloom filter — chống tấn công penetration (nâng cao)

Đặt **Bloom filter của TẤT CẢ key hợp lệ** ngay phía trước cache+DB:

```
Request key X
   → Bloom hỏi: "X có khả năng tồn tại?"
       ├─ "CHẮC CHẮN KHÔNG"  → trả not-found NGAY, KHÔNG đụng cache/DB  ✅ chặn tấn công
       └─ "CÓ THỂ CÓ"        → đi tiếp xuống cache → DB (như thường)
```

- ✅ Kẻ tấn công liệt kê **triệu ID ngẫu nhiên** → Bloom gạt gần hết tại cửa, **DB không hề hấn**, mà **không tốn 1 entry/ID** như negative entry.
- ⚠️ Giới hạn: **false positive** (một số ID lạ vẫn lọt — nhưng vô hại, chỉ rơi xuống DB như bình thường); **không false negative** (key thật **không bao giờ** bị gạt nhầm — an toàn); **khó xoá** key khi tập hợp lệ đổi → dùng **Cuckoo filter** (hỗ trợ xoá + gọn hơn).

```javascript
// RedisBloom (module): chặn key rác trước khi xuống cache/DB
await redis.call("BF.ADD", "valid_users", "user:123");     // nạp key hợp lệ
const maybe = await redis.call("BF.EXISTS", "valid_users", `user:${id}`);
if (maybe === 0) return null; // "chắc chắn không" → chặn ngay, DB an toàn
// maybe === 1 → "có thể có" → đi tiếp như thường
```

> 🎯 **Quy tắc chọn:** keyspace **nhỏ/lành** → **negative entry** là đủ. Keyspace **lớn / có nguy cơ bị liệt kê tấn công** → **Bloom/Cuckoo filter** phía trước. Thực tế hệ lớn **dùng cả hai**: Bloom chặn rác diện rộng + negative entry cho phần lọt qua.

---

## ⑤ CASE STUDY THỰC TẾ

**1. ⭐ DNS NXDOMAIN (ví dụ kinh điển nhất — RFC 2308).**
Khi bạn tra một domain **không tồn tại**, resolver nhận **NXDOMAIN** và **cache lại chính sự vắng mặt đó** trong một khoảng (theo trường **SOA minimum/TTL**). Nhờ vậy, gõ nhầm `gooogle.com` hàng triệu lần **không** làm các authoritative server ngập. Đây là negative caching **có tuổi đời hàng thập kỷ**, và dạy đúng hai bài: **(1) chỉ cache vắng-mặt-có-thẩm-quyền** (NXDOMAIN thật, không phải timeout), **(2) TTL có giới hạn** để domain mới đăng ký sớm xuất hiện.

**2. ⭐ Bloom filter trong storage engine (RocksDB / Cassandra / HBase / Bigtable).**
Mỗi **SSTable** gắn một **Bloom filter** để trả lời "key này có khả năng nằm trong file này không?". Nếu Bloom nói "chắc chắn không" → **bỏ qua, KHỎI đọc đĩa**. Đây là negative caching ở **tầng lưu trữ** — tiết kiệm khổng lồ I/O. Một trong những ứng dụng Bloom filter có ảnh hưởng nhất thực tế.

**3. CDN 404 caching (Cloudflare / Akamai / Fastly).**
Cache phản hồi **not-found** ngay tại edge (TTL thường ngắn, cấu hình được) → một URL gãy bị truy cập dồn dập **không** đập về origin mỗi lần. Đúng tinh thần tầng reverse-proxy/CDN của Bài 1.

**4. Bảo vệ DB tier ở hệ lớn (kết hợp với leases — Stampede doc).**
Negative caching cho "no such key" + **lease/singleflight** để ngay cả **key-rỗng-nóng** cũng chỉ một kẻ đi DB → bịt cả penetration lẫn stampede trên cùng path.

**5. Auth / rate-limit.**
Cache ngắn "token không hợp lệ" / "user không có quyền" để khỏi tra store xác thực lặp lại — nhưng **TTL rất ngắn** vì quyền có thể đổi.

**6. Playbook chống penetration ở big-tech.**
**Bloom filter + negative entry TTL-ngắn** là bộ đôi tiêu chuẩn (mô tả nhiều trong tài liệu SRE, gắn với taxonomy 穿透/击穿/雪崩 ở chuyên đề Stampede).

---

## ⑥ STAFF-LEVEL: 3 QUYẾT ĐỊNH THIẾT KẾ CỐT TỬ

### 6.1 — ⭐ CHỈ cache "vắng mặt có thẩm quyền", KHÔNG cache lỗi tạm thời

> Đây là **cái bẫy staff số một**. **Timeout / 500 / mất kết nối ≠ "không tồn tại".** Nếu bạn cache một **lỗi tạm thời** như là negative → bạn biến **một cú chớp** thành **sai kéo dài**: thứ đó **thật ra CÓ**, nhưng bạn trả "không có" suốt cả TTL.

```javascript
const row = await loadFromDb(id).catch((e) => ({ __error: true, e }));
if (row?.__error) throw row.e;        // ❌ LỖI → KHÔNG cache negative (hoặc TTL cực ngắn 1-2s)
if (!row) await redis.set(key, NEG, "EX", NEG_TTL); // ✅ chỉ cache khi DB KHẲNG ĐỊNH "không có"
```

> Phân biệt: **"DB nói rõ: 0 hàng"** (authoritative absence — cache được) vs **"không hỏi được DB"** (error — đừng cache như absence).

### 6.2 — Invalidate negative khi resource được TẠO (write path)

Khi cái-đang-vắng-mặt **được tạo ra**, bạn **phải xoá/ghi đè** negative entry, nếu không sẽ phục vụ "không tồn tại" sai sau khi nó đã tồn tại:

```javascript
async function createUser(user) {
  await db.insert(user);
  await redis.del(`user:${user.id}`);          // ⭐ XOÁ negative entry (nếu có) ngay khi tạo
  // hoặc với Bloom: await redis.call("BF.ADD", "valid_users", `user:${user.id}`);
}
```
> Cộng với **negative TTL ngắn** (6.1), đây là hai lớp bảo hiểm để "vắng mặt" không kẹt lại sau khi đã "có mặt".

### 6.3 — Chặn negative-cache pollution / DoS amplification

Kẻ tấn công liệt kê ID lạ → mỗi ID một negative entry → **nổ bộ nhớ + đuổi mất positive thật → THRASHING** (nối chuyên đề Thrashing!). Phòng:
- **Bloom/Cuckoo filter phía trước** (mục ④) → rác bị gạt **trước khi** tạo entry.
- **Cap số negative entry** + **TTL ngắn**.
- **Rate-limit theo client trên MISS** → một client gây quá nhiều miss-không-tồn-tại bị bóp.

---

## ⑦ CÁC TRƯỜNG HỢP MỞ RỘNG / KIẾN THỨC NÂNG CAO

- **Stampede trên chính negative (击穿 cho key rỗng-nóng).** Một key-không-tồn-tại **rất nóng** mà negative entry vừa hết hạn → cả đàn cùng miss xuống DB. ➜ **singleflight/lock cũng áp cho negative**; hoặc **refresh-ahead / XFetch** (Stampede doc) cho negative entry nóng.
- **Counting Bloom & Cuckoo filter — khi tập hợp lệ THAY ĐỔI.** Bloom thường **không xoá được**. Khi key hợp lệ bị xoá (cần "rút" khỏi filter) → dùng **Counting Bloom** (đếm thay vì bit) hoặc **Cuckoo filter** (gọn hơn + hỗ trợ xoá + tra nhanh). Phải **rebuild/refresh filter** khi tập hợp lệ trôi.
- **Tiered negative caching.** Bloom **in-process (L1)** chặn rác cực nhanh không tốn network + negative entry ở **Redis (L2)** cho phần lọt qua (Bài 1 multi-tier).
- **Liên hệ "tombstone" trong DB phân tán (Cassandra).** Xoá = ghi **tombstone** (đánh dấu vắng mặt) thay vì xoá vật lý — cùng họ tư tưởng "lưu sự vắng mặt". Cảnh báo: **tombstone tích tụ** gây chậm đọc (GC grace) — bài học rằng "đánh dấu vắng mặt" cũng có cái giá vận hành.
- **GraphQL/REST trả `null`.** Cache `null`/empty cẩn thận: phân biệt "field thật sự null" vs "không tra được" vs "chưa tra".
- **An ninh là góc nhìn chính.** Negative caching + Bloom + rate-limit-trên-miss là **bộ chống cache-penetration-DDoS** tiêu chuẩn — đừng coi đây chỉ là tối ưu hiệu năng.

---

## ⑧ KHUNG QUYẾT ĐỊNH STAFF ENGINEER

```
1. Path "not found" có bị hỏi LẶP LẠI / có nguy cơ bị LIỆT KÊ tấn công không?
   ├─ KHÔNG (hiếm khi hỏi thứ không có) → có thể bỏ qua, không cần negative caching
   └─ CÓ → đi tiếp

2. Keyspace lớn / có nguy cơ tấn công enumeration?
   ├─ CÓ  → BLOOM / CUCKOO FILTER phía trước (chặn rác diện rộng, không nổ bộ nhớ)
   └─ KHÔNG → NEGATIVE ENTRY (sentinel) là đủ

3. "Not found" là THẨM QUYỀN (DB nói 0 hàng) hay LỖI (timeout/500)?
   ├─ Thẩm quyền → cache negative (TTL NGẮN)
   └─ Lỗi → KHÔNG cache như absence (hoặc TTL 1-2s + phân loại lỗi)

4. "Vắng mặt" có thể thành "có mặt" (resource được tạo)?
   └─ CÓ (gần như luôn) → NEGATIVE TTL NGẮN + INVALIDATE-ON-CREATE (xoá negative/BF.ADD khi tạo)

5. Có key-rỗng-NÓNG (stampede trên negative)?
   └─ SINGLEFLIGHT/LOCK cho negative + (nếu cần) refresh-ahead

6. Chống pollution/DoS: cap entry + TTL ngắn + rate-limit theo client trên MISS
```

> 🥇 **Mặc định thực dụng:** **negative entry + sentinel rõ ràng + TTL ngắn + invalidate-on-create**; thêm **Bloom/Cuckoo filter** nếu keyspace lớn hoặc lo tấn công; **tuyệt đối không** cache lỗi tạm thời như "không tồn tại".

---

## ⑨ BẪY THƯỜNG GẶP / 🔎 AI HAY SAI

- 🔎 **Cache lỗi tạm thời như "không tồn tại".** Timeout/500 bị cache thành negative → thứ-có-thật bị báo "không có" suốt TTL. **Chỉ cache vắng-mặt-thẩm-quyền.**
- 🔎 **Negative TTL quá dài.** User vừa tạo tài khoản → "user không tồn tại" mấy phút. **TTL ngắn + invalidate-on-create.**
- 🔎 **Quên invalidate negative khi tạo** → "not found" kẹt vĩnh viễn dù đã tồn tại.
- 🔎 **Dùng `null` làm sentinel** → nhầm với "miss thật" trong nhiều client. **Dùng marker riêng biệt.**
- 🔎 **Negative entry không giới hạn** → kẻ tấn công liệt kê ID lạ làm **nổ bộ nhớ + thrashing**. **Bloom + cap + rate-limit.**
- 🔎 **Tưởng Bloom xoá được** → không (Bloom thường). Cần **Counting Bloom/Cuckoo** khi tập hợp lệ đổi.
- 🔎 **Quên stampede trên negative** → key-rỗng-nóng vẫn đập DB khi negative hết hạn. **Singleflight cho negative.**

---

## ⑩ MENTAL MODEL + CÂU THẦN CHÚ

> **Negative caching = nhớ rằng "đã tra, không có gì" để khỏi đi tra lại** — bịt lỗ thủng penetration đi thẳng xuống DB.
> **Hai quy tắc vàng:** (1) **vắng mặt expire NHANH** (vì "không có" dễ thành "có") + **invalidate khi tạo**; (2) **chỉ cache vắng-mặt-THẨM-QUYỀN, KHÔNG cache lỗi tạm thời.**
> **Keyspace lớn / bị tấn công liệt kê → dùng BLOOM/CUCKOO FILTER** thay cho (hoặc trước) negative entry — vài bit/key, không nổ bộ nhớ.

---

## ⑪ CÂU HỎI PHỎNG VẤN

- **(dễ)** Negative caching là gì, giải bài toán nào? → cache "không tồn tại" để chống **penetration** (hỏi thứ không có đập DB lặp lại).
- **(TB)** Vì sao negative TTL nên NGẮN hơn positive? → "vắng mặt" có thể thành "có mặt" bất cứ lúc nào; TTL dài = phục vụ "không có" sai cho thứ vừa được tạo.
- **(TB)** Negative entry vs Bloom filter — khi nào dùng cái nào? → entry chính xác nhưng tốn bộ nhớ & yếu trước enumeration; Bloom gọn + chống tấn công nhưng có false-positive & khó xoá. Keyspace lớn/bị tấn công → Bloom.
- **(TB)** Vì sao **không** được cache timeout/500 như negative? → đó là **lỗi**, không phải vắng-mặt-thẩm-quyền; cache nó biến chớp nhoáng thành sai kéo dài.
- **(khó)** DB dùng Bloom filter để negative-cache ở đâu? → mỗi **SSTable** gắn Bloom; "chắc chắn không có" → **bỏ qua đọc đĩa** (RocksDB/Cassandra/HBase).
- **(khó)** Tập key hợp lệ thay đổi, Bloom không xoá được — giải sao? → **Counting Bloom / Cuckoo filter** (hỗ trợ xoá) hoặc **rebuild filter** định kỳ.
- **(thách đố)** "Kẻ tấn công bắn triệu ID ngẫu nhiên không tồn tại, DB sắp sập" — phòng tuyến? → **Bloom/Cuckoo filter** chặn cửa + **negative TTL ngắn** cho phần lọt + **rate-limit theo client trên miss** + (nếu nóng) **singleflight**.

---

## ⑫ GLOSSARY MỞ RỘNG (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh |
|---|---|---|
| **Negative caching** | Cache cả kết quả "không tồn tại" | *"Cache 'user không có' TTL ngắn để khỏi đập DB lặp."* |
| **Cache penetration** (穿透) | Hỏi key không tồn tại → luôn miss → đập DB | *"Bot bắn ID rác → **penetration** → cần negative cache."* |
| **Sentinel / tombstone** | Marker đánh dấu "đã biết là rỗng" | *"Lưu `__MISS__`, KHÁC null, để phân biệt miss thật."* |
| **Authoritative absence** | Nguồn KHẲNG ĐỊNH "không có" (vs lỗi) | *"DB trả 0 hàng = cache được; timeout = đừng."* |
| **Negative TTL** | TTL (ngắn) cho entry vắng-mặt | *"Negative 30s, positive 300s."* |
| **Invalidate-on-create** | Xoá negative khi resource được tạo | *"`createUser` → `redis.del` / `BF.ADD`."* |
| **Bloom filter** | Tập xác suất: "chắc chắn không / có thể có" | *"Bloom gạt ID rác trước khi xuống DB."* |
| **False positive / false negative** | Bloom: có FP (lọt vài key), KHÔNG có FN | *"Key thật không bao giờ bị gạt nhầm → an toàn."* |
| **Cuckoo / Counting Bloom** | Bloom **hỗ trợ xoá** | *"Tập hợp lệ đổi → dùng **Cuckoo filter**."* |
| **NXDOMAIN (DNS)** | "Domain không tồn tại" được negative-cache | *"RFC 2308: cache NXDOMAIN theo SOA TTL."* |
| **SSTable Bloom filter** | Bloom mỗi file để bỏ qua đọc đĩa | *"RocksDB: 'không có ở đây' → khỏi đọc SSTable."* |
| **Negative-cache pollution** | Entry rỗng tràn cache → thrashing | *"ID lạ nổ bộ nhớ → cap + Bloom + rate-limit."* |
| **Rate-limit on miss** | Bóp client gây quá nhiều miss-không-tồn-tại | *"Một IP bắn 10k miss/s → chặn."* |

---

> 🧭 **Một dòng để mang theo:** *Negative caching là nhớ "đã tra, rỗng" để bịt lỗ penetration xuống DB — nhưng vắng-mặt phải expire NHANH và được xoá khi resource ra đời, và bạn chỉ cache vắng-mặt-có-thẩm-quyền, TUYỆT ĐỐI không cache lỗi tạm thời. Keyspace lớn hay bị bắn ID rác? Dựng Bloom/Cuckoo filter ở cửa — vài bit mỗi key, chặn cả triệu ID lạ mà không nổ bộ nhớ.*
