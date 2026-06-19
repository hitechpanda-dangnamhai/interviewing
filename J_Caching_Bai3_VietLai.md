# 🎯 BÀI 3 — Invalidation · TTL · Key Design · Negative Caching
**Phủ:** J-INVAL-001 → J-INVAL-010

---

## ① Mục tiêu & vị trí trong mạch

> *"Có hai việc khó nhất trong khoa học máy tính: đặt tên biến, **cache invalidation**, và lỗi off-by-one."*

Câu đùa kinh điển này (chính nó đã cố tình sai off-by-one) nói lên một sự thật: **cache invalidation** — tức **biết khi nào và xoá cái gì khỏi cache khi nguồn thay đổi** — là một trong những bài toán **dễ làm sai nhất**.

Bài 2 cho bạn các **strategy** (luồng đọc/ghi). Bài 3 đi vào câu hỏi gai góc hơn: **"khi DB đổi, làm sao để cache không trả đồ cũ (stale)?"** Bạn sẽ học **bộ công cụ kiểm soát stale**:

- **TTL** (hết hạn tự động)
- **Explicit invalidation** (xoá tay khi ghi)
- **Versioned key** (đổi version để vô hiệu cả nhóm)
- **Negative caching** (cache cả kết quả "không tồn tại")
- **Stale-while-revalidate** (trả đồ cũ tạm thời trong lúc làm mới ngầm)

Và quan trọng không kém — **thiết kế cache key** cho đúng. Đây là **nền trực tiếp cho Bài 4 (consistency)**.

---

## ② Giảng cơ bản → nâng cao

### (a) Vì sao invalidation KHÓ? *(J-INVAL-001)*

Cái khó nằm ở **2 câu hỏi luôn đi cùng nhau**: **KHI NÀO** cần xoá, và **CÁI GÌ** cần xoá khi nguồn đổi.

**Vì sao "cái gì" lại khó?** Vì một thay đổi nhỏ ở nguồn có thể làm **sai nhiều thứ dẫn xuất (derived data / aggregate)** mà bạn không ngờ tới:

**Ví dụ trực giác — sửa 1 sản phẩm, hỏng 3 chỗ cache:**
> Bạn sửa giá của **một sản phẩm** (1 row). Nhưng row đó đang được "trộn" vào nhiều bản cache khác:
> - **Một list** "sản phẩm theo danh mục" (giờ hiển thị giá cũ).
> - **Một count** "số sản phẩm dưới 100k" (có thể đã sai vì sản phẩm này vừa vượt mốc).
> - **Một trang** kết quả tìm kiếm đã render sẵn (chứa giá cũ).
>
> Bạn sửa **1 chỗ ở nguồn**, nhưng có **≥ 3 key cache** cần được dọn. Quên một cái → stale.

**Hai kiểu sai đối xứng nhau:**
- **Xoá THIẾU** → còn sót bản cũ → **stale**.
- **Xoá THỪA** (xoá quá tay cho chắc) → **mất hiệu quả cache** (phải nạp lại nhiều thứ vô ích).

Thêm nữa: phải **phối hợp giữa nhiều writer / nhiều instance** — nếu node A ghi mà không báo được cho node B, B vẫn giữ đồ cũ.

> 🎯 **Chốt:** invalidation khó không phải vì lệnh xoá khó viết, mà vì **lập bản đồ "nguồn nào ảnh hưởng key cache nào" rất rối**, đặc biệt với **aggregate phụ thuộc nhiều nguồn**.

---

### (b) Hai cơ chế GỐC: TTL vs Explicit *(J-INVAL-002, 003)*

Mọi kỹ thuật invalidation đều xây trên **2 cơ chế nền** này:

#### TTL-based expiry (hết hạn theo thời gian)

**Đặt một thời hạn sống (`Time To Live`) cho key; hết hạn key tự biến mất.**

- ✅ **Đơn giản, tự dọn** — không cần ai nhớ đi xoá.
- 📏 **Bảo đảm:** **stale tối đa = TTL**. Đặt TTL = 60s nghĩa là "bản cache không bao giờ cũ quá 60 giây".

**Ẩn dụ:** như **hộp sữa có hạn sử dụng**. Đến hạn là tự bỏ, không cần ai kiểm tra từng hộp.

#### Explicit invalidation (xoá tường minh khi ghi)

**Chủ động `DEL` key ngay khi có write.**

- ✅ **Tươi hơn** — đổi là xoá liền, không đợi hết hạn.
- ⚠️ **Điều kiện sống còn:** phải **bắt được MỌI đường ghi** + **xoá ĐÚNG key**. Lỡ một đường ghi không xoá → bản đó stale.

> 💡 **Thực tế: KẾT HỢP cả hai.**
> **Explicit để tươi** (đổi là xoá ngay) **+ TTL làm lưới an toàn (backstop)**. Lý do: nếu bạn *lỡ một đường ghi* nào đó không kịp invalidate, **TTL vẫn đảm bảo bản sai tự chết sau cùng lắm là TTL giây** — chứ không stale vĩnh viễn.

#### TTL ngắn vs dài — đánh đổi:

| | **TTL ngắn** | **TTL dài** |
|---|---|---|
| Độ tươi | ✅ Tươi hơn | ❌ Stale lâu hơn |
| Hit ratio | ❌ Thấp (hay miss) | ✅ Cao |
| Tải DB | ❌ Cao (miss nhiều → đập DB) | ✅ Thấp |

> **Chọn theo:** **mức chấp nhận stale của data** + **chi phí mỗi lần miss**. Data nhạy cảm với cũ → TTL ngắn; data ổn định + miss đắt → TTL dài.

---

### (c) Thiết kế CACHE KEY *(J-INVAL-004)* — ⭐ cực kỳ quan trọng

**Nguyên tắc vàng:** **cache key phải xác định DUY NHẤT kết quả**, và do đó phải **chứa MỌI tham số ảnh hưởng tới output**.

Hãy liệt kê các "chiều" thường bị quên:
- **user / tenant** (ai đang hỏi? data của tenant A khác tenant B)
- **locale / lang** (tiếng Việt vs tiếng Anh ra kết quả khác)
- **filter** (lọc `active` vs `all` ra danh sách khác)
- **version schema** (format dữ liệu phiên bản mấy)

**🔥 Key TỒI kinh điển — và hậu quả thảm hoạ:**
```
products          ← THIẾU tenant, filter, locale
```
> Hậu quả: hai tenant khác nhau **dùng chung một key** → tenant B **đọc trúng cache của tenant A** → **RÒ DỮ LIỆU giữa các tenant / sai phân quyền**. Đây là một **lỗ hổng bảo mật thật sự**, không chỉ là bug hiệu năng.

**✅ Key TỐT:**
```
tenant:42:products:lang=vi:filter=active:v3
```
- Dùng **namespace rõ ràng** với **prefix + dấu `:`** (quy ước phổ biến của Redis).
- Mỗi chiều ảnh hưởng output đều có mặt trong key → **không thể đụng nhầm** data của ai khác.

> 🎯 **Cách tự kiểm:** *"Hai request cho ra kết quả KHÁC nhau thì key của chúng có KHÁC nhau không?"* Nếu hai request khác kết quả mà **trùng key** → bạn sẽ phục vụ nhầm. (Lỗi này lặp lại ở Bài 7, J-REDIS-009.)

---

### (d) Versioned key / Cache busting *(J-INVAL-005)*

**Ý tưởng:** **nhúng một số version (hoặc đổi prefix) vào key.** Khi muốn vô hiệu toàn bộ bản cũ, chỉ cần **đổi (bump) version** — *không cần `DEL` từng key một*.

**Cơ chế hay ở chỗ:** đổi version = các key mới có tên khác hoàn toàn → **bản cũ tự động không bao giờ bị hỏi tới nữa**, rồi **tự rơi rụng theo TTL / eviction**.

**Ẩn dụ:** thay vì đi xé từng tờ thông báo cũ dán khắp toà nhà, bạn **đổi màu giấy** cho thông báo mới — mọi người chỉ đọc giấy màu mới, đống cũ cứ để đó tự mục.

**Lợi ích then chốt — tránh race "xoá-rồi-đọc-lại":**
> Với `DEL` thủ công, có thể xảy ra: bạn vừa xoá key xong, *ngay lập tức* một request khác đọc miss → nạp lại **bản CŨ** từ một replica chưa kịp cập nhật → cache lại stale. **Versioned key tránh được** vì bản mới nằm ở *tên key mới*, không đụng bản cũ.

**Ví dụ:** đổi format dữ liệu → **bump `schema_v` từ 3 → 4**. Toàn bộ cache `...:v3` lập tức "vô hình", code chỉ đọc/ghi `...:v4`.

---

### (e) Invalidate list / aggregate *(J-INVAL-006)*

Đây là lời giải cho vấn đề "1 phần tử đổi → nhiều key dẫn xuất sai" ở mục (a). **Đừng đi truy từng key bằng tay.** Ba cách gọn:

1. **Tag / group key** — gắn nhãn nhóm cho các key liên quan, xoá theo nhóm.
2. **Versioned namespace** — **bump version của cả nhóm** (cùng cơ chế mục d, nhưng cho một cụm).
3. **TTL ngắn cho aggregate** — chấp nhận list/count hơi cũ một chút, để nó tự làm mới nhanh.

---

### (f) Event-based invalidation *(J-INVAL-007)*

**Vấn đề với "xoá cache ngay trong code ghi":** nó chỉ bắt được những đường ghi **đi qua app**. Còn nếu **một batch job, một service khác, hay ai đó sửa thẳng DB** thì sao? → cache không hề biết → stale.

**Giải pháp:** thay vì xoá cache inline, **phát một event khi DB thay đổi → một consumer nghe event đó và đi xoá cache.**

- ✅ **Tách concern** (logic ghi không phải lo chuyện cache).
- ✅ **Bắt được CẢ những ghi không qua app** (batch, service khác).

> 🔗 Để tránh tình huống **"ghi DB OK nhưng publish event FAIL"** (rồi cache không bao giờ được xoá), người ta gắn với **Outbox pattern** hoặc **CDC** (*Change Data Capture* — đọc thẳng log thay đổi của DB). Chi tiết ở **Bài 4** và **mục K**.

---

### (g) Negative caching *(J-INVAL-008)* — ➕ chống **cache penetration**

**Negative caching = cache CẢ kết quả "không tồn tại"** (lưu một marker `null` / empty).

**Vấn đề nó giải — cache penetration (xuyên thủng cache):**
> Kẻ tấn công (hoặc bug) liên tục hỏi những **id rác không tồn tại**: `user/999999999`, `user/abc`... Mỗi lần như vậy cache **miss** (vì không có gì để cache cả) → **request xuyên thẳng xuống DB**. Hỏi dồn dập → **DB gục**. Cache không bảo vệ được vì nó chỉ cache thứ *tồn tại*.

**Cách chặn:** khi DB trả "không có", **vẫn cache lại cái "không có" đó** (ví dụ lưu `"__NULL__"`). Lần sau hỏi cùng id rác → **negative hit** → trả `null` ngay, **không đụng DB**.

**⚠️ Rủi ro phải biết — "null cache che mất data mới":**
> Nếu sau đó dữ liệu *được tạo thật* (id đó giờ tồn tại), nhưng cache vẫn đang giữ marker `null` cũ → bạn trả "không có" cho thứ đã có. **Hai biện pháp bắt buộc:**
> 1. **TTL NGẮN cho null** (ví dụ 30s) — giới hạn thời gian che.
> 2. **Invalidate khi create** — vừa tạo data là xoá ngay marker null.

*(Cross-ref Bài 5: penetration, Bloom filter.)*

---

### (h) ĐỪNG bỏ TTL hoàn toàn *(J-INVAL-009)*

Một cám dỗ nguy hiểm: *"Tôi invalidate thủ công chuẩn lắm rồi, khỏi cần TTL cho đỡ miss."*

> 🚨 **Sai lầm chết người.** Chỉ cần **lỡ MỘT đường ghi** không gọi invalidate (một service mới, một batch job, một nhánh code quên) → key đó **stale VĨNH VIỄN**, vì **không có lưới an toàn** nào để nó tự chết.

**Quy tắc:** **LUÔN có TTL backstop**, kể cả khi đã làm explicit invalidation cẩn thận. TTL là "bảo hiểm" cho những lần con người làm sót.

---

### (i) Stale-while-revalidate *(J-INVAL-010)*

**Ý tưởng:** khi một key **vừa hết hạn**, thay vì bắt client **chờ** trong lúc đi DB nạp lại, ta **trả ngay bản stale (hơi cũ)** cho client, **đồng thời làm mới ở background**. Lần sau sẽ có bản mới.

- ✅ **Không ai phải chờ DB lúc expiry** → **bỏ được latency spike** đúng lúc key hết hạn.
- ⚖️ **Đánh đổi:** chấp nhận một **độ stale ngắn** (client nhận bản cũ vài giây) để **đổi lấy độ trễ ổn định**.

**Ẩn dụ:** bồi bàn đưa bạn **ly nước cũ còn ngon** ngay lập tức, trong khi *bếp đang chuẩn bị ly mới* — bạn không phải đứng đợi khát.

Rất phổ biến ở **HTTP / CDN** (header `stale-while-revalidate` — Bài 11) và cả app cache. *(verify cú pháp header.)*

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| `DEL` từng key khi đổi format | **Versioned key / namespace** (bump version) | Vô hiệu **cả nhóm tức thì**, tránh **race xoá-đọc** |
| *"Chỉ invalidate thủ công, khỏi TTL"* | **Explicit + TTL backstop** | Lỡ 1 đường ghi → **stale vĩnh viễn** |
| Xoá cache ngay trong code ghi | **Event / CDC-based invalidation** (Bài 4) | Bắt cả **ghi ngoài app**, tách concern |
| Key thiếu chiều (user/tenant/locale) | Key gồm **mọi tham số ảnh hưởng** | Tránh **rò dữ liệu** giữa user |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case 1 — Danh mục đa tenant, đa ngôn ngữ:**
- Key: `tenant:{id}:catalog:{lang}:v{n}`.
- Đổi schema → **bump `v`** (cache busting toàn bộ bản cũ).

**Use case 2 — Trang "không tìm thấy" bị spam id rác:**
- **Negative cache** `null` trong **30s** + **Bloom filter** (Bài 5) để chặn từ sớm.

**Pattern ở bigtech:**
- **Versioned / content-hash key** là **chuẩn cho static asset**: CDN cache-bust bằng cách **đổi tên file** → `app.a1b2c3.js`. Đổi nội dung = đổi hash = đổi tên = trình duyệt buộc tải bản mới.
- Hệ thống lớn thường dùng **CDC (Debezium)** → invalidate cache, **thay vì rải lệnh `DEL` khắp code** (bắt được mọi thay đổi ngay từ DB log).

---

## ⑤ Code thực hành + cấu hình

**Versioned key + Negative cache:**

```javascript
// catalogCache.js
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

const SCHEMA_V = 3;   // ⭐ bump số này = vô hiệu TOÀN BỘ bản cũ (cache busting)
const NEG_TTL  = 30;  // null cache phải NGẮN
const TTL      = 600;

// key gồm MỌI chiều ảnh hưởng output: tenant + lang + version
const key = (tenant, lang) => `tenant:${tenant}:catalog:${lang}:v${SCHEMA_V}`;

export async function getCatalog(tenant, lang, db) {
  const k = key(tenant, lang);
  const cached = await redis.get(k);
  if (cached === "__NULL__") return null;          // ✅ negative HIT → chặn DB
  if (cached !== null) return JSON.parse(cached);  // ✅ HIT

  const data = await db.loadCatalog(tenant, lang); // MISS → đọc DB
  if (data == null) {
    await redis.set(k, "__NULL__", "EX", NEG_TTL); // negative cache, TTL NGẮN
    return null;
  }
  await redis.set(k, JSON.stringify(data), "EX", TTL);
  return data;
}

// ⚠️ Khi tạo data mới PHẢI xoá negative cache để tránh "null che mất data mới":
export async function onCatalogCreated(tenant, lang) {
  await redis.del(key(tenant, lang));
}
```

**⚠️ Hai cảnh báo:**
- **`SCHEMA_V` đặt ở config** để bump **không phải sửa code rải rác**.
- **Negative TTL phải NGẮN** + **invalidate khi create** (nếu không sẽ che mất data vừa tạo).

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **Vì sao invalidation khó** (*khi nào* + *cái gì*, đặc biệt **aggregate**).
- **TTL vs explicit** trade-off.
- **Vì sao versioned key tốt** (vô hiệu cả nhóm, tránh race).
- **Negative caching** & rủi ro **che data mới**.
- **Stale-while-revalidate**.

📌 **Cần THUỘC:**
- **Key phải gồm MỌI tham số ảnh hưởng** kết quả.
- **Explicit + TTL backstop** (luôn có lưới an toàn).
- **Negative TTL ngắn + invalidate-on-create**.

🛠️ **Cần LÀM ĐƯỢC:** thiết kế **cache key đúng namespace**; implement **versioned busting** + **negative cache**.

---

## ⑦ Mental model

> **Key gồm MỌI thứ ảnh hưởng kết quả.**
> **Explicit invalidate + TTL backstop** (đừng bao giờ bỏ TTL).
> **Đổi format → bump version**, đừng `DEL` từng key.
> **Cache cả "không tồn tại"** để chặn id rác (nhưng TTL ngắn + xoá khi create).

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Vì sao cache invalidation nổi tiếng khó? → biết **khi nào + cái gì** xoá, nhất là **aggregate** phụ thuộc nhiều nguồn.
- **(TB)** TTL vs explicit invalidation, khi nào dùng gì? → **TTL** đơn giản (stale ≤ TTL); **explicit** tươi nhưng phải bắt mọi đường ghi; **kết hợp**.
- **(TB)** Một cache key tồi và hậu quả? → thiếu **tenant/user** → **rò data** giữa người dùng.
- **(khó)** Versioned key tốt hơn `DEL` thủ công chỗ nào? → **vô hiệu cả nhóm tức thì**, **tránh race**.
- **(khó)** Negative caching để làm gì, rủi ro? → chặn **penetration**; rủi ro **che data mới** → **TTL ngắn + invalidate-on-create**.

> **🧩 Thách đố:** *"Bạn đặt TTL 1h cho cache + invalidate trong code mỗi lần update. Một **batch job đêm** sửa thẳng DB không qua app. Chuyện gì xảy ra, fix sao?"*
>
> **Trả lời:** Batch job **không kích hoạt invalidate** (nó không đi qua app) → cache **stale tới tận 1 giờ**. **Fix:** dùng **CDC / event-based invalidation** (bắt thay đổi từ **DB log**, nên *mọi* đường ghi đều bị tóm), hoặc **TTL ngắn hơn** cho data đó. Bài học cốt lõi: **đừng bao giờ dựa vào giả định "mọi ghi đều qua app"**.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thiết kế cache key cho API `GET /reports?tenant=&from=&to=&format=`.
> ✅ **Đạt khi:** key gồm **tenant + from + to + format + version**; có **namespace rõ ràng**.

**BT2.** Thêm **negative caching** cho `getUserByEmail`.
> ✅ **Đạt khi:** cache `null` **TTL ngắn** + **xoá null khi user được tạo**.

---

## ⑩ Đọc thêm

- **redis.io/docs** — *Key expiration*, *Client-side caching invalidation*.
- **MDN / RFC 9111** — *stale-while-revalidate*.
- **Debezium docs** — *CDC để invalidate cache* (mục K).

---

> 🔎 **AI hay sai ở đây:** AI sinh cache key kiểu `cache:${queryName}` — **quên user / tenant / locale** → **rò data**.
> **Luôn tự kiểm:** *key có gồm ĐỦ chiều phân biệt không?* (Lỗi này lặp lại ở Bài 7, J-REDIS-009.)
