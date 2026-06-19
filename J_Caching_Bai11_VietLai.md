# 🎯 BÀI 11 — HTTP / CDN Caching
**Phủ:** J-HTTP-001 → J-HTTP-006

---

## ① Mục tiêu & vị trí trong mạch

Đây là **tầng cache XA NHẤT khỏi DB nhưng GẦN NHẤT với user** — và nó **đóng lại bức tranh "các tầng cache"** đã mở ở **Bài 1** (`browser → CDN → proxy → L1 → L2 → DB buffer`).

Học xong bài này bạn nắm: **Cache-Control**, **ETag**, **CDN**, **vì sao GraphQL/POST khó cache**, **purge & Vary**, và **bẫy cache response cá nhân hoá** (rò data giữa user). Chuẩn tham chiếu: **RFC 9111**.

> 💡 **Ý chính:** Ở các bài trước, cache nằm trong tay bạn (Redis, app). Ở tầng HTTP, **cache nằm trong tay người khác** — trình duyệt của user và các CDN edge khắp thế giới — và bạn **chỉ điều khiển chúng qua HEADER**. Đặt header sai một chữ (`public` thay vì `private`) là **rò dữ liệu giữa user**.

---

## ② Giảng cơ bản → nâng cao

### (a) Cache-Control *(J-HTTP-001)*

**`Cache-Control`** là header chính để **điều khiển cache HTTP**:

| Directive | Ý nghĩa |
|---|---|
| **`max-age=N`** | TTL (giây) cho **client cache** (trình duyệt) |
| **`s-maxage=N`** | TTL cho **shared cache / CDN** (ghi đè `max-age` *tại CDN*) |
| **`no-store`** | **KHÔNG lưu gì cả** (data nhạy cảm) |
| **`no-cache`** | **Lưu** nhưng **phải revalidate** trước khi dùng (**≠ `no-store`!**) |
| **`private`** | **Chỉ browser** cache (CDN không được) |
| **`public`** | **CDN được phép** cache |

> 🚨 **Bẫy kinh điển — `no-cache` ≠ `no-store`:**
> - **`no-store`** = **CẤM lưu** (như tài liệu mật, không được photocopy).
> - **`no-cache`** = **ĐƯỢC lưu**, nhưng **mỗi lần dùng phải hỏi lại server** "bản này còn đúng không?" (revalidate).
>
> Tên `no-cache` gây hiểu nhầm là "không cache" — thực ra nó *vẫn cache*, chỉ là kiểm lại trước khi dùng. *(verify)*

**Ẩn dụ:** `max-age` = "tin tờ giấy ghi chú này trong N giây, khỏi hỏi lại". `no-cache` = "giữ tờ giấy, nhưng mỗi lần dùng gọi điện xác nhận còn đúng không". `no-store` = "đọc xong xé ngay, không được giữ".

---

### (b) ETag + If-None-Match *(J-HTTP-002)*

> Server gắn **`ETag`** = **fingerprint (dấu vân tay) của nội dung**. Lần sau client gửi lại **`If-None-Match: <etag>`** → nếu **khớp** (nội dung không đổi), server trả **`304 Not Modified`** — **KHÔNG gửi body** → **tiết kiệm băng thông**.

Đây là **revalidation** — giữ **tươi** mà vẫn **cache được**. (Tương tự cặp **`Last-Modified` + `If-Modified-Since`**.)

```
Lần 1:  client → GET /api/products
        server → 200 + body + ETag: "abc123"
Lần 2:  client → GET /api/products  (kèm If-None-Match: "abc123")
        ├── nội dung KHÔNG đổi → server trả 304 (không body) → dùng lại bản cũ ✅ rẻ
        └── nội dung ĐÃ đổi    → server trả 200 + body mới + ETag mới
```

> 🎯 **Lợi hơn chỉ dùng `max-age`:** **tươi hơn** (kiểm lại nội dung thật) mà vẫn **rẻ** (304 không có body). *(verify)*

**Ẩn dụ ETag:** như **niêm phong dán trên hộp**. Bạn không mở hộp kiểm; chỉ liếc niêm phong — còn nguyên (ETag khớp) thì khỏi gửi lại cả hộp, dán nguyên mà dùng.

---

### (c) CDN vs app cache *(J-HTTP-003)*

| | **CDN** (edge) | **Redis / app cache** (data center) |
|---|---|---|
| Vị trí | **Edge gần user** (khắp địa lý) | Trong data center |
| Giải bài toán | **Cắt latency địa lý** + **offload origin** | **dynamic / shared state** |
| Hợp cache | **static / asset / response GET công khai** | data động, state chia sẻ |

> 🎯 **Đặt lên CDN:** **static / ảnh / JS / CSS** + **response GET công khai**.

**Ví dụ trực giác:** user ở Việt Nam tải `logo.png`. Không có CDN → request bay sang origin ở Mỹ (latency cao). Có CDN → **node ở Việt Nam** phục vụ luôn → nhanh hơn nhiều, **origin cũng nhẹ tải** vì không phải gửi lại file đó cho mọi người.

---

### (d) GraphQL / POST khó cache *(J-HTTP-004)*

> HTTP/CDN cache theo **URL + method**, và chỉ **`GET`** mới **idempotent/cacheable** tự nhiên.

**Vấn đề của GraphQL:** thường là **một endpoint `POST`** duy nhất với **body khác nhau** mỗi lần → **URL không phân biệt được query** → **mất HTTP cache**.

**Giải:**
- **Cache normalized ở client** (**Apollo / urql**).
- Hoặc **persisted query + `GET`** (biến query thành GET có URL ổn định → cache lại được). *(Cross-ref F GraphQL cache.)*

**Vì sao POST không cache?** Vì `POST` ngầm hiểu là *"thao tác có thể thay đổi state"* → cache nó là sai về ngữ nghĩa. Cache hạ tầng vì thế **bỏ qua POST**.

---

### (e) Purge & Vary *(J-HTTP-005)*

**Purge (xoá cache CDN):** xoá trên **nhiều PoP (Points of Presence)** toàn cầu có **độ trễ + tốn**.
> 🎯 **Nên dùng versioned URL** (**content hash** `app.a1b2.js`) **thay vì purge** — đổi nội dung = đổi tên file = client buộc tải bản mới, **không cần purge gì cả**.

**Vary:** header **chia nhỏ cache key theo các header khác** (như `Accept-Encoding`, `Authorization`).
> ⚠️ **Đặt Vary sai = hai kiểu hỏng đối xứng:**
> - **Vary quá RỘNG** → cache key vỡ vụn → **miss nhiều**.
> - **Thiếu Vary cho data riêng** → **rò cache giữa user** (bản của người này phục vụ người kia). *(verify)*

---

### (f) Cache response cá nhân hoá *(J-HTTP-006)* — ⚠️⚠️ NGUY HIỂM

> 🚨 **Bẫy chí mạng:** cache **`public`** một response **có data riêng** → **rò cho user khác**. Nguyên nhân: **`Cache-Control: private` bị đặt sai/bỏ qua**, hoặc **CDN cache nhầm**.

**Đúng cách — 3 lựa chọn:**
1. Cá nhân hoá → **`private` / `no-store`**.
2. Hoặc **cache theo key gồm `user`**.
3. Hoặc **tách phần public** (cache được) **khỏi phần riêng** (không cache) — dùng **ESI (edge-side includes)**.

**Ẩn dụ:** CDN như **bảng tin chung của toà nhà**. Dán thông báo chung (public) thì ai đọc cũng được. Nhưng **dán nhầm bảng lương cá nhân lên bảng tin chung** (đánh `public` cho data riêng) → cả toà nhà đọc được lương của bạn.

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Lẫn **`no-cache` = không cache** | `no-cache` = **lưu + revalidate**; `no-store` = **cấm lưu** | **Nghĩa khác nhau hoàn toàn** |
| **Purge CDN** mỗi lần đổi asset | **Versioned / content-hash URL** | Purge toàn cầu **chậm/tốn** |
| Cache **GraphQL như REST GET** ở CDN | **Normalized client cache** / **persisted query + GET** | 1 endpoint POST **không cache HTTP được** |
| Cache response **user-specific là `public`** | **`private`/`no-store`** hoặc **key gồm user** | Tránh **rò data giữa user** |
| Quên **Vary** cho data theo header | Đặt **Vary đúng (vừa đủ)** | Sai → **miss tràn** hoặc **rò cache** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case:**
- **static asset** → `Cache-Control: public, max-age=31536000, immutable` + **versioned filename**.
- **API GET công khai** → **`s-maxage` ở CDN** + **`ETag`**.
- **Trang dashboard cá nhân** → **`private, no-store`**.
- **SPA bundle** → **content-hash** để cache bust.

**Pattern ở bigtech:**
- **Cloudflare / Akamai / CloudFront / Fastly** ở **edge**.
- **`immutable` + versioned URL** là **chuẩn cho asset**.
- **`stale-while-revalidate`** (Bài 3) phổ biến để giữ latency thấp.
- **GraphQL APIs** dùng **persisted queries** để lấy lại HTTP/CDN cache. *(verify)*

---

## ⑤ Code thực hành + cấu hình

**Express set header đúng:**

```javascript
import express from "express";
const app = express();

// Static asset versioned → cache CỰC DÀI, immutable (1 năm)
app.use("/assets", express.static("public", {
  setHeaders: (res) => res.setHeader("Cache-Control", "public, max-age=31536000, immutable"),
}));

// API GET công khai: CDN cache 60s + revalidate bằng ETag (Express tự sinh ETag)
app.get("/api/products", (req, res) => {
  res.setHeader("Cache-Control", "public, s-maxage=60, stale-while-revalidate=30"); // verify SWR
  res.json(products);
});

// Dashboard cá nhân hoá: KHÔNG cache công khai → tránh rò data giữa user
app.get("/api/me/dashboard", (req, res) => {
  res.setHeader("Cache-Control", "private, no-store"); // ⚠️ data riêng KHÔNG được public
  res.json(buildDashboard(req.user));
});
```

> ⚠️ **Đừng để response có data user là `public`.**
> ⚠️ **`Vary: Authorization`** nếu response đổi theo auth (hoặc **tốt hơn: `private`**). *(verify header tại MDN/RFC 9111)*

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **`no-cache` ≠ `no-store`**.
- **`ETag` / `304` revalidation**.
- **CDN giải latency địa lý**.
- **GraphQL/POST mất HTTP cache**.
- **Vary chia cache key**.
- **Rủi ro cache cá nhân hoá**.

📌 **Cần THUỘC:**
- **`max-age` (client) vs `s-maxage` (CDN)**.
- **`private` vs `public`**.
- **versioned URL thay purge**.
- **`private`/`no-store` cho data riêng**.

🛠️ **Cần LÀM ĐƯỢC:** set **`Cache-Control` đúng** cho static / public-API / private; dùng **`ETag`**; dùng **content-hash bust**; **tránh rò cache**.

---

## ⑦ Mental model

> **CDN/HTTP cache ở edge gần user, theo URL + method.**
> **Static → `public` + versioned + `immutable`.**
> **API GET công khai → `s-maxage` + `ETag`.**
> **Data riêng → `private`/`no-store` (ĐỪNG public!).**
> **`no-cache` = lưu + revalidate, KHÁC `no-store`.**

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(TB)** Phân biệt `max-age` / `s-maxage` / `no-store` / `no-cache` / `private`/`public`. → *(bảng ②a)*.
- **(khó)** ETag + If-None-Match hoạt động thế nào, lợi gì? → **fingerprint → 304 không body → tiết kiệm băng thông + revalidate**.
- **(TB)** CDN giải gì mà Redis không? → **latency địa lý** + **offload origin** cho static/public.
- **(khó)** Vì sao GraphQL/POST khó cache HTTP hơn REST GET? → **1 endpoint POST body khác** → mất HTTP cache.
- **(khó)** Cache response cá nhân hoá nguy hiểm sao, làm đúng? → **rò data**; **`private`/`no-store`/key gồm user/tách public-riêng**.

> **🧩 Thách đố:** *"Bạn đặt `Cache-Control: public, max-age=300` cho `/api/me/profile`. Vài user báo thấy thông tin của người khác. Vì sao?"*
>
> **Trả lời:** Response chứa **data riêng** nhưng đánh **`public`** → **CDN / shared cache lưu bản của user A và trả cho user B** (cùng URL, **không phân biệt auth**).
> ```
> user A → GET /api/me/profile → CDN lưu bản A (vì public)
> user B → GET /api/me/profile → CDN trả LẠI bản A ❌ (rò data)
> ```
> **Sửa:** **`private, no-store`** (hoặc tối thiểu `private` + `Vary: Authorization`). Đây đúng là **bẫy rò cache J-HTTP-006** — lỗi **correctness/security** phải bắt.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Set header đúng cho 3 endpoint: `/assets/app.<hash>.js`, `/api/news` (public), `/api/me/orders` (riêng).
> ✅ **Đạt khi:** asset **`immutable` + versioned**; news **`public` + `s-maxage` + `ETag`**; orders **`private`/`no-store`**.

**BT2.** Giải thích cách cache một trang có **phần public** (tin tức) + **phần riêng** (tên user) mà **không rò**.
> ✅ **Đạt khi:** **tách phần public cache** + **phần riêng không cache** (**ESI** / client render), **không đánh `public` cho cả trang**.

---

## ⑩ Đọc thêm

- **MDN** — *HTTP caching, Cache-Control, ETag*.
- **RFC 9111** (HTTP Caching) — nguồn chuẩn nhất.
- **web.dev** — *Love your cache* (HTTP caching best practices).

---

> 🔎 **AI hay sai ở đây:** AI đặt **`public, max-age`** cho **mọi response "cho nhanh"**, kể cả endpoint có **data user** → **rò cache**.
> **Luôn tự kiểm:** *response này có data riêng của user không?* Có → **`private`/`no-store`**.
