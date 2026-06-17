# 📘 Giáo trình F — API Design & Best Practices (Tech Lead Backend)

> **Nguồn:** dựng ngược từ `QB_F_apidesign.md` (87 câu, 11 mục con) theo **WORKFLOW 2 — Bước B & C**.
> **Mục tiêu:** một tài liệu tự học đầy đủ — đọc xong tự tin trả lời 87 câu của bộ đề và đi phỏng vấn Tech Lead phần API.
> **Cách dùng:** đọc theo thứ tự bài (đã sắp cơ bản → nâng cao, theo *dependency order* chứ không theo tần suất). Mỗi bài là một đơn vị tiêu hóa được, theo **Hợp đồng đầu ra 10 mục**.
> **Câu thần chú:** *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

---

## ⚠️ Bối cảnh chuẩn & phiên bản (đã verify 6/2026 — verify lại khi đọc về sau)

| Chuẩn | Trạng thái (6/2026) | Ghi chú |
|---|---|---|
| **RFC 9110** | RFC ổn định (HTTP Semantics) | Định nghĩa method semantics: GET/HEAD/PUT/DELETE/OPTIONS **idempotent**; POST/PATCH **không**. |
| **RFC 9457 "Problem Details"** | RFC ổn định (7/2023) | **Obsoletes RFC 7807**. Media type `application/problem+json`. |
| **OpenAPI 3.2.0** | **Stable hiện hành** (19/9/2025) | Thêm mô tả native cho streaming (SSE, JSON Lines), hierarchical tags, custom HTTP methods. 3.1 vẫn dùng rộng. |
| **OpenAPI 4.0 "Moonwalk"** | **Đang thiết kế, CHƯA release** (3/2026) | Không nói "4.0 hiện hành". Tooling thực tế đều target 3.x. |
| **`Idempotency-Key` header** | **IETF Internet-Draft** (`draft-ietf-httpapi-idempotency-key-header-07`, 10/2025) | **Chưa thành RFC**. Stripe/Adyen/Dwolla dùng de-facto. |
| **`RateLimit` / `RateLimit-Policy` header** | **IETF Internet-Draft** (`-11`, 5/2026) | **Chưa thành RFC**. De-facto cũ `RateLimit-Limit/Remaining/Reset` vẫn deploy rộng. |
| **RFC 8288** | RFC ổn định | Web Linking — `Link` header (dùng cho pagination). |
| **RFC 6902 / 7386** | RFC ổn định | JSON Patch / JSON Merge Patch (cho PATCH). |

> Nguồn verify: rfc-editor.org, openapis.org, datatracker.ietf.org. Mọi chỗ chạm **tên/đời model, giá, đời spec** đều đánh dấu *(verify)* và phải search lại khi nội dung quá hạn.

---

## 🗺️ BƯỚC B — Giáo trình ngược từ bộ đề (mục lục + ánh xạ Bài → câu hỏi)

15 bài, gom 87 câu theo cụm khái niệm, sắp theo phụ thuộc kiến thức:

### Module 1 — Nền tảng REST (phải vững trước khi đi xa)
| Bài | Tên | Phủ câu hỏi |
|---|---|---|
| **1** | REST đúng nghĩa: 6 ràng buộc, resource modeling, method semantics | F-REST-001, 002, 003, 004 |
| **2** | Status code, hợp đồng tạo resource, content negotiation | F-REST-005, 006, 009 |
| **3** | Richardson Maturity Model & HATEOAS | F-REST-007, 008 |
| **4** | CORS — bảo vệ ai, preflight, vì sao "lỗi CORS" không phải lỗi server | F-REST-010 |
| **5** | Batch/bulk & thao tác chạy lâu (async operation) | F-REST-011, 012 |

### Module 2 — Tiến hóa API & hợp đồng (contract)
| Bài | Tên | Phủ câu hỏi |
|---|---|---|
| **6** | API Versioning & deprecation | F-VER-001…007 |
| **7** | Error handling & chuẩn hóa response (RFC 9457) | F-ERR-001…008 |
| **8** | Idempotency | F-IDEM-001…008 |

### Module 3 — Collection ở quy mô lớn
| Bài | Tên | Phủ câu hỏi |
|---|---|---|
| **9** | Pagination / filtering / sorting | F-PAG-001…008 |
| **10** | Rate limiting (thuật toán + phân tán) | F-RL-001…008 |

### Module 4 — Spec & các paradigm khác REST
| Bài | Tên | Phủ câu hỏi |
|---|---|---|
| **11** | OpenAPI/Swagger, contract-first, codegen | F-OAS-001…007 |
| **12** | GraphQL vs REST, DataLoader, federation | F-GQL-001…009 |
| **13** | Async API: WebSocket / SSE / realtime | F-WS-001…007 |
| **14** | gRPC + Protobuf (nội bộ) | F-GRPC-001…007 |

### Module 5 — Quyết định cấp Tech Lead
| Bài | Tên | Phủ câu hỏi |
|---|---|---|
| **15** | Gateway / BFF / Contract testing / Deprecation / Webhook / Governance | F-TL-001…006 |

> **Mạch xuyên suốt:** REST là *mặc định*; mọi thứ khác (GraphQL, gRPC, WS) là *lựa chọn có lý do*. Tech Lead không học để "gõ đúng cú pháp" mà để **quyết định đúng paradigm + bảo vệ tính nhất quán/khả tiến hóa của hợp đồng API qua thời gian và qua nhiều team**.

---

# MODULE 1 — NỀN TẢNG REST

---

## BÀI 1 — REST đúng nghĩa: 6 ràng buộc, resource modeling, method semantics
> Phủ: F-REST-001, 002, 003, 004

### ① Mục tiêu & vị trí trong mạch
Bài mở màn của cả mục F. Học xong bạn phân biệt được **REST thật** (theo Fielding) với "JSON over HTTP" bị gọi nhầm là REST, biết **mô hình hóa resource** và **đặt tên URL** đúng, và phân loại 5 HTTP method theo 2 trục **safe/idempotent**. Đây là nền cho mọi bài sau: status code (Bài 2), idempotency (Bài 8), versioning (Bài 6) đều dựa trên hiểu biết này.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** REST không phải "API trả JSON". REST là một **kiến trúc** Roy Fielding mô tả trong luận án 2000: coi mọi thứ là **resource** (danh từ) có **địa chỉ (URI)**, thao tác lên chúng bằng một **tập method thống nhất** của HTTP (động từ), và mỗi request **tự đủ ngữ cảnh** (stateless). Ẩn dụ: REST giống một **tủ hồ sơ chuẩn hóa** — bạn không gọi "hàm lấy hồ sơ", bạn chỉ tay vào *ngăn kéo nào, hồ sơ nào* rồi nói động tác chuẩn (lấy/ghi/xóa).

**(b) Cơ chế — 6 ràng buộc của Fielding:**
1. **Client–Server** — tách UI khỏi lưu trữ; hai bên tiến hóa độc lập.
2. **Stateless** — server **không giữ session state** giữa các request; mỗi request mang đủ thông tin (auth, params). Đây là ràng buộc bị vi phạm nhiều nhất.
3. **Cacheable** — response phải tự khai báo có cache được không (`Cache-Control`, `ETag`).
4. **Uniform Interface** — *linh hồn của REST*: định danh resource qua URI; thao tác qua representation; self-descriptive messages; và HATEOAS (hypermedia).
5. **Layered System** — client không biết nó nói chuyện trực tiếp với server hay qua proxy/gateway/LB.
6. **Code-on-Demand** (tùy chọn) — server có thể gửi code thực thi (hiếm dùng).

**Resource modeling & URL (F-REST-002):**
- Resource = **danh từ số nhiều**: `/users`, `/orders`.
- **Không động từ trong path**: `/getUserOrders` ❌ → `GET /users/{id}/orders` ✅.
- **Nesting** khi quan hệ sở hữu rõ và truy cập luôn qua cha: `/users/{id}/orders`. **Flatten** khi resource có đời sống độc lập / cần truy cập trực tiếp: `/orders/{orderId}` thay vì lồng sâu `/users/{u}/orders/{o}/items/{i}/...`. Quy tắc ngón tay cái: **không lồng quá 2 cấp**.
- Action không map gọn vào CRUD (vd "publish", "cancel") → mô hình hóa thành **sub-resource trạng thái** (`POST /orders/{id}/cancellation`) hoặc chấp nhận một "controller resource". Đây là điểm REST thuần lúng túng, sẽ gặp lại ở Bài 5.

**Method semantics theo 2 trục (F-REST-003) — RFC 9110:**

| Method | Safe (không đổi state)? | Idempotent (lặp = 1 lần)? | Có body request? |
|---|---|---|---|
| GET | ✅ | ✅ | thường không |
| HEAD | ✅ | ✅ | không |
| OPTIONS | ✅ | ✅ | không |
| PUT | ❌ | ✅ | có |
| DELETE | ❌ | ✅ | thường không |
| POST | ❌ | ❌ | có |
| PATCH | ❌ | ❌ (thường) | có |

- **Safe** = chỉ đọc, không gây side-effect lên server (GET/HEAD/OPTIONS).
- **Idempotent** = gọi N lần cho cùng *hiệu ứng cuối* lên server như gọi 1 lần. DELETE xóa lần 2 vẫn "đã xóa" → idempotent (dù status có thể khác). POST tạo lần 2 → tạo thêm bản ghi → **không** idempotent.

**PUT vs PATCH (F-REST-004):**
- **PUT** = **thay toàn bộ** representation tại URI (client gửi full state) → idempotent.
- **PATCH** = **sửa một phần** → thường không idempotent (vd `{"op":"increment"}`).
- PATCH gửi gì? **Hai định dạng chuẩn khác nhau:**
  - **JSON Merge Patch (RFC 7386)** — gửi object "đè": `{"email":"new@x.com"}` đặt field, `null` để xóa field. Đơn giản, phổ biến.
  - **JSON Patch (RFC 6902)** — gửi *mảng thao tác*: `[{"op":"replace","path":"/email","value":"..."}]`. Mạnh hơn (add/remove/move/test) nhưng phức tạp.

**(c) Mép giới hạn & sai lầm thường gặp:**
- Gọi mọi API JSON là "RESTful" trong khi thực chất là **RPC-over-HTTP** (một endpoint `/api`, mọi thứ là POST, động từ trong body). Không sai về kỹ thuật nhưng đừng tự nhận RESTful.
- Vi phạm **stateless** bằng server-side session bám vào 1 node → vỡ khi scale ngang (xem lại ở Bài WS-004 & rate limit).
- Dùng POST cho thao tác lẽ ra idempotent (PUT) → tự chuốc bài toán chống trùng (Bài 8).
- Lồng resource quá sâu → URL khó dùng, khó version.

### ③ ⚠️ Kiến thức cũ / dễ nhầm

| Quan niệm cũ / phổ biến | Đúng hơn (hiện nay) | Vì sao |
|---|---|---|
| "REST = trả JSON qua HTTP" | REST = 6 ràng buộc; JSON chỉ là một representation | Tránh tự nhận RESTful sai |
| PATCH gửi nguyên object như PUT nhỏ | PATCH có **2 format chuẩn** (Merge Patch / JSON Patch) | Hợp đồng PATCH phải nói rõ format |
| DELETE lần 2 phải lỗi | DELETE **idempotent**; lần 2 thường 204/404 tùy thiết kế, *hiệu ứng* không đổi | Đừng nhầm "status khác" với "không idempotent" |
| Lồng resource càng sâu càng "RESTful" | Flatten khi resource độc lập; ≤2 cấp | URL dễ dùng & version |

### ④ Áp dụng thực tế + so sánh bigtech
- **GitHub REST API** là ví dụ kinh điển: resource danh từ (`/repos/{owner}/{repo}/issues`), dùng đúng verb + status, có `Link` header phân trang.
- **Stripe** thiên RPC-ish ở vài chỗ (`POST /v1/refunds`) nhưng vẫn resource-based — thực tế production hiếm khi REST thuần 100%, *pragmatic REST* là chuẩn ngầm. *(pattern phổ biến, không phải luật)*
- Kiến trúc tối thiểu: client → (gateway/LB, nhờ ràng buộc *Layered System* client không cần biết) → service stateless → DB. Vì stateless nên đặt thêm instance phía sau LB là xong — đây là lý do REST scale ngang dễ.

### ⑤ Code thực hành + cấu hình

```ts
// Express 5.x — minh hoạ resource modeling + method semantics đúng
// package.json: "express": "^5.1.0"  (verify bản mới nhất tại expressjs.com)
import express from 'express';
const app = express();
app.use(express.json());

// GET an toàn + idempotent: liệt kê đơn hàng của 1 user (nesting 1 cấp hợp lý)
app.get('/users/:id/orders', async (req, res) => {
  const orders = await db.orders.findByUser(req.params.id);
  res.json({ data: orders }); // 200
});

// PUT: thay toàn bộ -> idempotent. Client gửi FULL representation.
app.put('/orders/:id', async (req, res) => {
  await db.orders.replace(req.params.id, req.body); // ghi đè toàn bộ
  res.status(200).json(await db.orders.get(req.params.id));
});

// PATCH bằng JSON Merge Patch (RFC 7386): chỉ sửa field gửi lên
app.patch('/orders/:id', express.json({ type: 'application/merge-patch+json' }),
  async (req, res) => {
    await db.orders.merge(req.params.id, req.body); // null = xoá field
    res.status(200).json(await db.orders.get(req.params.id));
  });

app.listen(3000);
```

Cấu hình: `.env` cho DB URL/secret, không hardcode. `requirements`/`package.json` pin version chính. Cảnh báo: dùng `express.json()` mặc định sẽ không tách được `application/merge-patch+json` khỏi `application/json` — phải khai báo `type` như trên.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** stateless, uniform interface, resource vs representation, safe vs idempotent, "RPC-over-HTTP nhầm là REST".
- 📌 **Cần THUỘC:** 6 ràng buộc Fielding; bảng safe/idempotent của 7 method; RFC 7386 (Merge) vs RFC 6902 (JSON Patch); RFC 9110.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế URL cho một domain (chọn nest/flatten); chọn đúng method; viết PATCH đúng format.

### ⑦ Mental model
**"Danh từ ở URL, động từ ở method, ngữ cảnh ở mỗi request"** — và *safe = chỉ đọc, idempotent = lặp lại an toàn*.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Kể 6 ràng buộc REST. → Gợi ý: client-server, stateless, cacheable, uniform interface, layered, code-on-demand.
2. *(TB)* `/getUserOrders` sai chỗ nào, sửa thế nào? → động từ trong path; `GET /users/{id}/orders`.
3. *(TB)* DELETE có idempotent không dù lần 2 trả 404? → Có; *hiệu ứng cuối* không đổi.
4. *(khó)* Khi nào nest, khi nào flatten resource? → quan hệ sở hữu + truy cập luôn qua cha → nest (≤2 cấp); resource độc lập → flatten.
5. *(khó)* PATCH một field nhưng người ta báo "mất các field khác" — vì sao? → client/server hiểu nhầm PATCH là PUT, hoặc dùng sai format (Merge Patch vs full replace).

**Thách đố:** "API tôi POST mọi thứ vào `/api` và trả JSON — RESTful chưa?" → Không; đó là RPC-over-HTTP. Không sai, nhưng đừng gọi là REST và đừng kỳ vọng cache/uniform interface.

### ⑨ Bài tập + tiêu chí tự chấm
- **BT1:** Thiết kế URL + method cho domain "blog: user, post, comment, like". *Đạt khi:* danh từ số nhiều, ≤2 cấp nest, like mô hình hóa hợp lý (`PUT /posts/{id}/likes/{userId}` idempotent), không động từ.
- **BT2:** Viết handler PATCH theo cả Merge Patch và JSON Patch cho `/users/{id}`. *Đạt khi:* phân biệt được 2 content-type và hành vi `null`-để-xóa.

### ⑩ Đọc thêm
- Fielding dissertation (chương 5). RFC 9110 §9 (method semantics). RFC 7386 & RFC 6902. GitHub REST API docs (ví dụ tham chiếu).

---

## BÀI 2 — Status code, hợp đồng tạo resource, content negotiation
> Phủ: F-REST-005, 006, 009

### ① Mục tiêu & vị trí
Hợp đồng REST nằm ở **status code** (transport-level) — đây là tầng giao tiếp đầu tiên client đọc. Học xong bạn ánh xạ tình huống → status đúng, biết trả gì khi POST tạo thành công, và hiểu content negotiation (`Accept`/`Content-Type`, 406 vs 415). Nối thẳng sang Bài 7 (error body) và Bài 8 (idempotency dùng 409).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Status code là "tiêu đề một dòng" của response: chỉ nhìn con số, client/proxy/monitoring biết đại khái chuyện gì xảy ra mà chưa cần đọc body. **4xx = client sai, 5xx = server sai** — phân biệt này quyết định *ai phải sửa* và *có nên alert/retry không*.

**(b) Cơ chế — bản đồ status (F-REST-005):**

| Tình huống | Status | Ghi chú |
|---|---|---|
| Tạo thành công (đồng bộ) | **201 Created** | + header `Location` |
| Nhận xử lý sau (async) | **202 Accepted** | trả job/operation resource (Bài 5) |
| Thành công, không có body | **204 No Content** | vd DELETE, PUT không trả body |
| Malformed / parse fail | **400 Bad Request** | JSON hỏng, thiếu field bắt buộc |
| Hợp lệ cú pháp, sai nghiệp vụ | **422 Unprocessable Entity** | validation ngữ nghĩa (có tranh luận, xem Bài 7) |
| Chưa xác thực (chưa đăng nhập / token sai) | **401 Unauthorized** | thực chất là *unauthenticated* |
| Đã xác thực nhưng không đủ quyền | **403 Forbidden** | *unauthorized* |
| Conflict trạng thái | **409 Conflict** | trùng key, version mismatch, optimistic lock |
| Không tồn tại | **404 Not Found** | (đôi khi dùng cho ẩn sự tồn tại thay 403) |
| Quá hạn rate | **429 Too Many Requests** | + `Retry-After` (Bài 10) |
| Bug server | **500** / **503** (quá tải/đang bảo trì) | 5xx → alert |

**Hợp đồng tạo resource (F-REST-006):** POST tạo thành công → **201 + `Location: /orders/123`** (URI resource mới), thường kèm body chứa resource vừa tạo. **Không** trả 200 chung chung — client cần biết *resource mới ở đâu*.

**Content negotiation (F-REST-009):**
- Client gửi **`Accept: application/json`** → "tôi muốn nhận định dạng này".
- Server đọc **`Content-Type`** của *body request* → "client gửi lên định dạng gì".
- **415 Unsupported Media Type** = server không hiểu **body client gửi** (sai `Content-Type`).
- **406 Not Acceptable** = server **không tạo được representation** mà `Accept` yêu cầu.
- Negotiation cũng dùng cho version (Bài 6, media-type versioning) và ngôn ngữ (`Accept-Language`).

**(c) Mép giới hạn & sai lầm:**
- Trả **200 cho mọi thứ** rồi nhét `success:false` trong body → phá hợp đồng HTTP, monitoring/CDN/retry hiểu sai (đây cũng là điểm gây tranh cãi của GraphQL — Bài 12).
- Lẫn **401 ↔ 403**: chưa đăng nhập = 401; đăng nhập rồi nhưng cấm = 403.
- Trả 500 cho lỗi validation → kéo alert vô ích; validation là 4xx.
- Dùng 200 thay 201 khi tạo → client không biết Location.

### ③ ⚠️ Cũ → mới

| Cũ / hay gặp | Nên dùng | Vì sao |
|---|---|---|
| 200 + `{success:false}` cho lỗi | Status 4xx/5xx đúng nghĩa | Hợp đồng transport rõ ràng, hỗ trợ proxy/monitor |
| 200 khi POST tạo | **201 + Location** | Client biết resource mới ở đâu |
| 401 cho "không đủ quyền" | **403** | 401 = chưa xác thực; 403 = cấm |
| 400 cho mọi lỗi input | 400 (malformed) vs 422 (sai nghiệp vụ) | Phân biệt giúp client xử lý đúng |

### ④ Áp dụng thực tế + bigtech
- **Stripe/GitHub** trả 201+Location khi tạo, 422 cho validation field-level, 409 cho conflict (vd idempotency key đang xử lý). *(pattern phổ biến)*
- 401 thường kèm `WWW-Authenticate`. 429 kèm `Retry-After`. Đây là các "hợp đồng phụ" gắn với status.
- Kiến trúc: đặt một **global response/error mapper** ở edge của service (Bài 7) để mọi handler trả status nhất quán — TL nào cũng phải ép convention này.

### ⑤ Code thực hành

```ts
// Express 5.x — tạo resource đúng chuẩn + content negotiation
app.post('/orders', async (req, res) => {
  // 415 nếu client gửi body không phải JSON
  if (!req.is('application/json')) return res.sendStatus(415);

  const errors = validate(req.body);            // validation nghiệp vụ
  if (errors.length) return res.status(422).json({ errors }); // field-level (Bài 7)

  const order = await db.orders.create(req.body);
  res.status(201)
     .location(`/orders/${order.id}`)           // 201 + Location bắt buộc
     .json(order);
});

// 406 nếu không đáp ứng được Accept
app.get('/reports/:id', (req, res) => {
  if (!req.accepts(['json', 'csv'])) return res.sendStatus(406);
  const fmt = req.accepts(['json', 'csv']);     // chọn representation
  // ...trả theo fmt
});
```

### ⑥ Keywords
- 🧠 **HIỂU:** 4xx vs 5xx (ai sai), 401 vs 403, 406 vs 415, vì sao không "200 cho tất cả".
- 📌 **THUỘC:** 201/202/204/400/401/403/404/409/422/429; `Location`, `Retry-After`, `WWW-Authenticate`.
- 🛠️ **LÀM ĐƯỢC:** map tình huống → status; trả 201+Location; cài content negotiation.

### ⑦ Mental model
**"Số trước, body sau"** — status code là hợp đồng để máy đọc; body là chi tiết để người/đoạn code đọc.

### ⑧ Phỏng vấn & thách đố
1. 201 khác 202 ở đâu? → 201 đã tạo xong; 202 nhận, xử lý sau.
2. 401 vs 403? → chưa xác thực vs đủ xác thực nhưng cấm.
3. 415 vs 406? → body gửi sai kiểu vs không đáp ứng được `Accept`.
4. POST tạo xong trả gì? → 201 + `Location` (+ body).
5. *(khó)* Vì sao "200 + success:false" là anti-pattern? → vô hiệu hóa semantics HTTP cho cache/retry/monitoring/middleware.

**Thách đố:** "DELETE một resource đã xóa nên trả gì?" → 204 (idempotent, coi như đã đạt trạng thái mong muốn) hoặc 404 — chọn một và nhất quán; đừng trả 500.

### ⑨ Bài tập + tự chấm
- **BT:** Cho 10 tình huống (tạo, validate fail, chưa login, cấm, trùng, async...), gán status. *Đạt khi:* ≥9/10 đúng và giải thích được 4xx/5xx.

### ⑩ Đọc thêm
RFC 9110 §15 (status codes). MDN HTTP status reference.

---

## BÀI 3 — Richardson Maturity Model & HATEOAS
> Phủ: F-REST-007, 008

### ① Mục tiêu & vị trí
Hiểu "REST chín tới mức nào" qua **4 level của Richardson**, và HATEOAS (level cao nhất) là gì, lợi/hại, vì sao thực tế hiếm ai làm. Đây là bài "định vị": giúp bạn nói chuyện với interviewer về *mức độ RESTful* thay vì cãi nhau "RESTful hay không".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Richardson Maturity Model (RMM) là **thang đo độ trưởng thành** của một API HTTP, do Leonard Richardson đề xuất, Martin Fowler phổ biến. Không phải "chuẩn bắt buộc" — là *thước đo*.

**(b) Cơ chế — 4 level:**

| Level | Tên | Đặc trưng |
|---|---|---|
| **0** | The Swamp of POX | Một endpoint, một verb (thường POST), RPC-over-HTTP. Vd SOAP. |
| **1** | Resources | Nhiều URI resource riêng (`/orders/1`) nhưng vẫn dùng verb tùy tiện |
| **2** | HTTP Verbs | Dùng **đúng method + status code** (GET/POST/PUT/DELETE, 200/201/404...). **Đa số API production dừng ở đây.** |
| **3** | Hypermedia (HATEOAS) | Response chứa **link** để client tự khám phá hành động kế tiếp |

**HATEOAS (F-REST-008):** "Hypermedia As The Engine Of Application State". Response không chỉ trả data mà kèm **hypermedia controls** — các link/action khả dụng tiếp theo:
```json
{
  "id": 123, "status": "pending", "total": 50,
  "_links": {
    "self":   { "href": "/orders/123" },
    "cancel": { "href": "/orders/123/cancellation", "method": "POST" },
    "pay":    { "href": "/orders/123/payment", "method": "POST" }
  }
}
```
Ý tưởng: client **không hardcode URL/luồng**, mà đi theo link server cung cấp → **giảm coupling**, server đổi URL/luồng mà client không vỡ. Định dạng chuẩn hóa: **HAL**, **JSON:API**, **Siren**.

**(c) Mép giới hạn & vì sao ít ai làm L3:**
- Client (đặc biệt mobile/SPA) thường **vẫn hardcode** vì dev thấy tiện hơn → lợi ích "tự khám phá" không được tận dụng.
- Tăng kích thước payload + độ phức tạp cả 2 phía.
- Tooling/codegen quanh L2 (OpenAPI) trưởng thành hơn nhiều so với hypermedia.
→ Với **phần lớn ca, L2 là điểm ngọt**; ép L3 thường là **over-engineering**. Ngoại lệ hợp lý: API có **workflow/state machine phức tạp** (link khả dụng thay đổi theo trạng thái) — lúc đó HATEOAS thật sự đáng.

### ③ ⚠️ Cũ → mới / ngộ nhận

| Ngộ nhận | Thực tế |
|---|---|
| "Không HATEOAS thì không phải REST" | Đúng theo Fielding, nhưng **thực tế L2 được chấp nhận rộng** là "pragmatic REST" |
| HATEOAS giúp khỏi versioning | Giảm coupling URL, **không** loại bỏ breaking change ở schema/semantics (Bài 6) |
| L3 luôn tốt hơn | L3 thường over-engineering nếu client không tận dụng |

### ④ Áp dụng + bigtech
- **GitHub API** có yếu tố hypermedia (trả `*_url` cho resource liên quan) — gần L3 một phần. **PayPal**, một số API ngân hàng dùng HATEOAS cho payment workflow (link `approve`/`capture` đổi theo trạng thái) — đúng chỗ HATEOAS tỏa sáng. *(pattern, verify khi cần con số)*
- Đa số public API (Stripe, Twilio) dừng ở L2 + OpenAPI. Đây là *chuẩn ngầm của ngành*.

### ⑤ Code thực hành

```ts
// Thêm hypermedia controls tùy trạng thái (HAL-ish) — minh hoạ HATEOAS có chọn lọc
function withLinks(order) {
  const links = { self: { href: `/orders/${order.id}` } };
  if (order.status === 'pending') {
    links.pay = { href: `/orders/${order.id}/payment`, method: 'POST' };
    links.cancel = { href: `/orders/${order.id}/cancellation`, method: 'POST' };
  }
  return { ...order, _links: links };
}
app.get('/orders/:id', async (req, res) =>
  res.json(withLinks(await db.orders.get(req.params.id))));
```

### ⑥ Keywords
- 🧠 **HIỂU:** 4 level RMM; HATEOAS là gì & trade-off; vì sao L2 phổ biến.
- 📌 **THUỘC:** L0 POX / L1 resources / L2 verbs+status / L3 hypermedia; HAL/JSON:API/Siren.
- 🛠️ **LÀM ĐƯỢC:** đánh giá một API đang ở level nào; thêm link có chọn lọc khi workflow đáng.

### ⑦ Mental model
**"L2 là điểm ngọt; HATEOAS chỉ đáng khi luồng/trạng thái phức tạp."**

### ⑧ Phỏng vấn & thách đố
1. Kể 4 level RMM. 
2. Đa số production dừng ở đâu? → L2.
3. HATEOAS lợi gì? → giảm coupling, client tự khám phá.
4. *(khó)* Vì sao ít ai làm L3 đầy đủ? → client không tận dụng + phức tạp + tooling thiên L2.
5. *(khó)* Khi nào HATEOAS thật sự đáng? → workflow/state machine (payment, order lifecycle).

**Thách đố:** "Sếp bảo phải L3 cho mới REST chuẩn." → giải thích trade-off; đa số ca L2 đủ; đề xuất HATEOAS chỉ cho phần có workflow.

### ⑨ Bài tập + tự chấm
- **BT:** Phân loại 3 API bạn biết theo RMM + lý do. *Đạt khi:* nêu bằng chứng (verb/status/link) cho mỗi level.

### ⑩ Đọc thêm
Martin Fowler "Richardson Maturity Model". Spec HAL / JSON:API.

---

## BÀI 4 — CORS: bảo vệ ai, preflight, vì sao "lỗi CORS" không phải lỗi server
> Phủ: F-REST-010

### ① Mục tiêu & vị trí
CORS là chủ đề bị hiểu sai nhiều nhất. Học xong bạn biết **CORS bảo vệ ai** (không phải server), khi nào **preflight (`OPTIONS`)** kích hoạt, các header liên quan, và phân biệt CORS với CSRF/authz. Đây là điểm "AI viết code thường sai" rất hay gặp.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** **Same-Origin Policy (SOP)** là luật của *trình duyệt*: JS ở origin A không được tự do đọc response từ origin B. **CORS** là cơ chế cho server B **nới lỏng có kiểm soát** luật đó — server B nói với trình duyệt "tôi cho phép origin A đọc tôi". Mấu chốt: **CORS là cơ chế phía trình duyệt**, bảo vệ *người dùng* khỏi việc một site độc đọc trộm dữ liệu site khác *bằng credential của họ* — **không** bảo vệ server.

**(b) Cơ chế:**
- **Simple request** (GET/POST/HEAD + content-type "đơn giản" như `text/plain`, `application/x-www-form-urlencoded`, không custom header) → trình duyệt gửi thẳng, kèm header `Origin`; server trả `Access-Control-Allow-Origin` thì JS mới đọc được response.
- **Preflighted request** → trước request thật, trình duyệt tự gửi **`OPTIONS`** (preflight) để "hỏi xin phép". Kích hoạt khi: method "không đơn giản" (PUT/DELETE/PATCH), có **custom header** (vd `Authorization`, `X-...`), hoặc content-type `application/json`.
  - Preflight gửi: `Origin`, `Access-Control-Request-Method`, `Access-Control-Request-Headers`.
  - Server đáp: `Access-Control-Allow-Origin`, `-Allow-Methods`, `-Allow-Headers`, (tùy) `-Allow-Credentials`, `-Max-Age` (cache preflight).
- **Credentials** (cookie): cần `Access-Control-Allow-Credentials: true` **và** `Allow-Origin` phải là origin cụ thể (không được `*`).

**(c) Mép giới hạn & sai lầm:**
- **"Lỗi CORS" KHÔNG phải lỗi server**: request *vẫn tới server và vẫn chạy*; trình duyệt chỉ **chặn JS đọc response** vì thiếu header cho phép. (Đây là lý do bạn thấy 200 trong network tab nhưng JS vẫn báo lỗi.)
- CORS **không thay thế authz**: cho phép origin đọc ≠ cho phép hành động; vẫn phải check quyền ở server.
- CORS **khác CSRF**: CSRF lợi dụng việc trình duyệt *tự đính cookie*; chống CSRF bằng token/SameSite cookie, không phải bằng CORS.
- `Allow-Origin: *` + credentials → trình duyệt **từ chối**; phải echo origin cụ thể.
- Quên xử lý `OPTIONS` → preflight fail → request thật không bao giờ chạy.

### ③ ⚠️ Ngộ nhận → đúng

| Ngộ nhận | Đúng |
|---|---|
| Lỗi CORS = server lỗi | Server vẫn chạy; trình duyệt chặn *đọc response* |
| CORS bảo vệ server | CORS bảo vệ *người dùng* (browser-side) |
| Bật CORS = an toàn / là authz | CORS không phải authz; vẫn cần check quyền |
| `Allow-Origin: *` dùng được với cookie | Không; cần origin cụ thể + `Allow-Credentials` |
| CORS chống CSRF | Không; CSRF dùng token/SameSite |

### ④ Áp dụng + bigtech
- API công khai (đọc public, không cookie) thường để `Allow-Origin: *`. API có credential → whitelist origin cụ thể + `Allow-Credentials: true`. Bigtech thường cấu hình CORS **ở gateway/CDN** (Bài 15) chứ không rải trong từng service. *(pattern phổ biến)*
- Tránh `Max-Age` quá ngắn (preflight tốn round-trip mỗi lần) hoặc quá dài (đổi policy chậm có hiệu lực).

### ⑤ Code thực hành

```ts
// Express + cors — whitelist origin, cho phép credentials
// "cors": "^2.8.5"
import cors from 'cors';
const allowed = ['https://app.example.com'];
app.use(cors({
  origin: (o, cb) => cb(null, !o || allowed.includes(o)), // echo origin cụ thể
  credentials: true,                 // cho cookie
  methods: ['GET','POST','PUT','PATCH','DELETE'],
  allowedHeaders: ['Authorization','Content-Type'],
  maxAge: 600,                       // cache preflight 10'
}));
// LƯU Ý: cors() tự xử lý OPTIONS preflight. Đừng tự chặn OPTIONS phía trên.
```

> ⚠️ **AI có thể viết đúng cú pháp nhưng sai bảo mật:** generate `origin: '*'` *kèm* `credentials: true` — chạy "có vẻ ổn" ở dev nhưng trình duyệt sẽ từ chối khi có cookie, hoặc tệ hơn là mở toang cho mọi site. Người hiểu CORS phải bắt được và whitelist origin.

### ⑥ Keywords
- 🧠 **HIỂU:** SOP vs CORS; CORS bảo vệ ai; "lỗi CORS không phải lỗi server"; CORS ≠ CSRF ≠ authz.
- 📌 **THUỘC:** simple vs preflighted; `Access-Control-Allow-Origin/Methods/Headers/Credentials/Max-Age`; điều kiện kích hoạt preflight.
- 🛠️ **LÀM ĐƯỢC:** cấu hình CORS whitelist + credentials đúng; debug "200 nhưng JS báo CORS".

### ⑦ Mental model
**"CORS là luật của trình duyệt nới lỏng SOP — nó cho phép *đọc*, không cho phép *làm*."**

### ⑧ Phỏng vấn & thách đố
1. CORS bảo vệ ai? → người dùng (browser-side).
2. Preflight kích hoạt khi nào? → method không-đơn-giản / custom header / content-type json.
3. Vì sao 200 nhưng JS vẫn lỗi CORS? → server chạy, browser chặn đọc response.
4. *(khó)* `*` + credentials sao không được? → spec cấm; phải echo origin cụ thể.
5. *(khó)* CORS có chống CSRF không? → không; dùng SameSite/token.

**Thách đố:** "Bật CORS `*` cho nhanh rồi auth sau" — sai chỗ nào? → CORS không phải lớp auth; mở `*` không tự gây lộ data nếu authz đúng, nhưng tạo cảm giác sai về bảo mật và vỡ khi cần cookie.

### ⑨ Bài tập + tự chấm
- **BT:** Cấu hình CORS cho SPA `https://app.x.com` gọi API `https://api.x.com` có cookie. *Đạt khi:* whitelist origin cụ thể, `credentials:true`, xử lý preflight, không dùng `*`.

### ⑩ Đọc thêm
MDN "Cross-Origin Resource Sharing (CORS)". Fetch standard (CORS protocol).

---

## BÀI 5 — Batch/bulk & thao tác chạy lâu (async operation)
> Phủ: F-REST-011, 012

### ① Mục tiêu & vị trí
REST thuần xoay quanh "một resource / một request" → lúng túng với (a) **thao tác hàng loạt** (tạo nhiều, một số fail) và (b) **thao tác chạy lâu** (export 10 phút). Học xong bạn thiết kế được cả hai mà không block connection hay fail-all. Nối sang Bài 7 (partial-success error) và Bài 13 (SSE/webhook báo xong).

### ② Giảng cơ bản → nâng cao

**(a) Batch/bulk (F-REST-011).** Trực giác: REST không có "transaction nhiều resource" sẵn. Hai cách:
- **Endpoint `/batch`** nhận mảng item, trả mảng kết quả per-item.
- **`207 Multi-Status`** (từ WebDAV, hay được mượn) — body chứa status riêng cho từng item.
- Quyết định lớn nhất: **partial success** hay **all-or-nothing**?
  - *Partial success* (mặc định cho bulk import): item nào fail báo riêng, item khác vẫn thành công. Phải trả tường minh từng item (index/id + status + error).
  - *All-or-nothing* (cần tính nguyên tử): chỉ dùng khi nghiệp vụ đòi transaction; toàn batch fail nếu một item fail. Đắt và khó scale.

```json
// 207 Multi-Status (partial success)
{ "results": [
  { "index": 0, "status": 201, "id": "a1" },
  { "index": 1, "status": 422, "error": { "code": "INVALID_EMAIL" } },
  { "index": 2, "status": 201, "id": "a3" }
]}
```

**(b) Thao tác chạy lâu (F-REST-012).** Vấn đề: export báo cáo 10 phút — **không thể giữ HTTP connection chờ đồng bộ** (timeout proxy/LB ~30–60s, tốn tài nguyên, client treo). Pattern chuẩn — **async request-reply / job resource**:
1. `POST /reports` → server nhận, tạo **job/operation resource**, trả **`202 Accepted`** + `Location: /operations/{id}` (+ thường `Retry-After`).
2. Client **poll** `GET /operations/{id}` → `{status: "running"|"succeeded"|"failed", result_url?}`.
3. Khi xong: trả link tới kết quả (vd `GET /reports/{id}/download`), hoặc đẩy chủ động qua **webhook/SSE** (Bài 13) để khỏi poll.

**(c) Mép giới hạn & sai lầm:**
- Cố giữ request đồng bộ cho việc chạy lâu → timeout, retry mù, double-execute. **Đừng.**
- Batch fail-all khi nghiệp vụ chấp nhận partial → trải nghiệm tệ (import 1000 dòng fail vì 1 dòng).
- Polling quá dày → tốn tài nguyên; trả `Retry-After` để hướng dẫn nhịp poll, hoặc dùng webhook/SSE.
- Job resource không có TTL/cleanup → phình storage.
- Thiếu **idempotency** ở bước POST tạo job → client retry tạo job trùng (gặp lại ở Bài 8).

### ③ ⚠️ Cũ → mới

| Cũ / sai | Nên dùng | Vì sao |
|---|---|---|
| Giữ connection chờ tác vụ 10' | `202` + job resource + poll/webhook | Tránh timeout, giải phóng tài nguyên |
| Bulk fail-all mặc định | `207`/mảng kết quả + partial success | Không hỏng cả batch vì 1 item |
| Trả 200 + "đang xử lý" | `202 Accepted` + `Location` operation | Status đúng nghĩa async |

### ④ Áp dụng + bigtech
- **Google/AWS** dùng "Long-Running Operations": trả operation resource có `name`, client poll/`wait`. Stripe trả async job cho một số report. **S3** multipart upload là biến thể batch/partial. *(pattern phổ biến, verify chi tiết tại docs từng hãng)*
- Kiến trúc tối thiểu: API nhận → đẩy job vào **queue** (SQS/RabbitMQ) → worker xử lý → cập nhật trạng thái job (DB/Redis) → client poll hoặc nhận webhook. Đây là chỗ giao với mục messaging/queue.

### ⑤ Code thực hành

```ts
// Async long-running: POST tạo job -> 202, GET poll trạng thái
app.post('/reports', async (req, res) => {
  const job = await db.jobs.create({ status: 'queued', spec: req.body });
  await queue.enqueue('build-report', { jobId: job.id }); // worker xử lý nền
  res.status(202)
     .location(`/operations/${job.id}`)
     .set('Retry-After', '5')      // gợi ý poll sau 5s
     .json({ id: job.id, status: 'queued' });
});

app.get('/operations/:id', async (req, res) => {
  const job = await db.jobs.get(req.params.id);
  if (!job) return res.sendStatus(404);
  const body = { id: job.id, status: job.status };
  if (job.status === 'succeeded') body.result_url = `/reports/${job.id}/download`;
  res.json(body); // 200; client tự quyết định poll tiếp hay tải kết quả
});
```

### ⑥ Keywords
- 🧠 **HIỂU:** async request-reply, partial success vs all-or-nothing, vì sao không block connection.
- 📌 **THUỘC:** `202 Accepted` + `Location` operation; `207 Multi-Status`; `Retry-After`.
- 🛠️ **LÀM ĐƯỢC:** thiết kế job resource + poll; thiết kế batch trả per-item result.

### ⑦ Mental model
**"Việc lâu → trả biên nhận (202 + job), đừng bắt client đứng chờ"; "batch → báo cáo từng dòng, đừng fail cả xe vì một dòng".**

### ⑧ Phỏng vấn & thách đố
1. Export 10 phút thiết kế thế nào? → 202 + operation resource + poll/webhook.
2. Batch một số fail xử lý ra sao? → 207/partial success, per-item result.
3. Vì sao không giữ connection đồng bộ? → timeout LB/proxy, tốn tài nguyên, retry mù.
4. *(khó)* Polling vs webhook khi nào? → poll đơn giản/ít job; webhook khi nhiều/độ trễ thấp + consumer chịu được at-least-once.
5. *(khó)* Job resource cần gì để production-ready? → TTL/cleanup, idempotency ở POST, trạng thái + lý do fail, link kết quả.

**Thách đố:** "Client retry POST /reports 3 lần do mạng → 3 job trùng." Sửa? → idempotency key ở bước tạo job (Bài 8).

### ⑨ Bài tập + tự chấm
- **BT:** Thiết kế API "import CSV 50k dòng". *Đạt khi:* 202+job, poll trạng thái có tiến độ, kết quả per-row error (partial), idempotent khi retry.

### ⑩ Đọc thêm
"Asynchronous Request-Reply pattern" (Microsoft Azure Architecture). Google AIP-151 Long-running operations. RFC 4918 §11 (207).

---

# MODULE 2 — TIẾN HÓA API & HỢP ĐỒNG

---

## BÀI 6 — API Versioning & Deprecation
> Phủ: F-VER-001…007

### ① Mục tiêu & vị trí
API là **hợp đồng** với client mà bạn không kiểm soát. Học xong bạn biết các cách version, phân biệt **breaking vs non-breaking**, khi nào bump major, vận hành nhiều version song song tốn gì, một **deprecation policy** đầy đủ gồm gì, và sự thật về "GraphQL/HATEOAS không cần version". Đây là tư duy *governance* — rất TL.

### ② Giảng cơ bản → nâng cao

**(a) Cách version (F-VER-001):**

| Cách | Ví dụ | Ưu | Nhược |
|---|---|---|---|
| **URI path** | `/v1/users` | Phổ biến nhất, dễ thấy/test/cache, route đơn giản | "Không RESTful thuần" (version không phải thuộc tính của resource) |
| **Query param** | `/users?version=1` | Dễ default | Lẫn với filter, dễ quên |
| **Custom header** | `X-API-Version: 1` | URL sạch | Khó test bằng URL, khó cache, "ẩn" |
| **Media type (accept)** | `Accept: application/vnd.x.v1+json` | "Đúng REST" (content negotiation) | Phức tạp, ít tooling, khó debug |

→ Thực tế: **URI path thắng vì pragmatic** (nhìn thấy, cache, test dễ). Media-type "thuần REST" nhất nhưng đắt.

**(b) Breaking vs non-breaking (F-VER-002) — quy tắc vàng:**
- **Non-breaking (additive):** thêm field *optional* vào response; thêm endpoint mới; thêm *optional* param; thêm giá trị enum mới (nếu client xử lý gracefully).
- **Breaking:** xóa/đổi tên field; đổi kiểu dữ liệu field; thêm *required* param; đổi status code; đổi nghĩa/định dạng giá trị; thắt chặt validation.
- Nguyên tắc **robustness / Postel**: "conservative in what you send, liberal in what you accept" — và **chỉ mở rộng additive** để tránh phải bump version.

**(c) Khi nào bump major (F-VER-003):** **chỉ khi có breaking change**. Ưu tiên tiến hóa additive để **không** phải maintain nhiều version song song. Bump major là "vũ khí cuối".

**(d) Chi phí nhiều version + deprecation policy (F-VER-004):**
- Mỗi version song song = nhân code path + test + bug surface + tài liệu. N version = N gánh nặng.
- **Deprecation policy đầy đủ:** thông báo sớm → **sunset timeline rõ** → header **`Deprecation`** (báo đã deprecated) + **`Sunset`** (RFC 8594, ngày tắt) → **migration guide** → **telemetry** để biết *ai còn dùng* (không tắt mù). Không tắt đột ngột.

**(e) HATEOAS giảm version? (F-VER-005):** Hypermedia giảm coupling về **URL/navigation** (client không hardcode link) → giảm *một loại* breaking. Nhưng **không** loại bỏ breaking ở **schema/semantics** (đổi field, đổi nghĩa). Hợp lý một phần, không phải thuốc tiên.

**(f) Stripe model (F-VER-006):** Stripe version **theo ngày** (`2025-xx-xx`), **pin per-account**; server giữ **transformation layer** map request/response cũ↔mới. Giải quyết: client cũ **không phải đổi code**, account mới nhận default mới. Đánh đổi: **layer transform phức tạp**, phải maintain mọi phép biến đổi giữa các version. Đây là đỉnh cao của versioning ở public API quy mô lớn. *(verify chi tiết tại stripe.com/docs/api/versioning)*

**(g) GraphQL "không cần version" (F-VER-007):** GraphQL evolve bằng **thêm field + `@deprecated`** thay vì version cả API; client chỉ lấy field nó cần nên thêm field không vỡ ai. Nhưng **breaking vẫn xảy ra** (xóa field, đổi type, đổi nullability). "Versionless" nói về *cách evolve* (continuous evolution), **không** phải miễn nhiễm breaking.

### ③ ⚠️ Cũ → mới

| Cũ / hay gặp | Nên dùng | Vì sao |
|---|---|---|
| Bump version cho mọi thay đổi | Chỉ bump khi **breaking**; còn lại additive | Tránh maintain N version |
| Tắt version cũ khi "thấy ít dùng" | Deprecation policy + `Deprecation`/`Sunset` + telemetry | Không phá client âm thầm |
| Tin "GraphQL miễn version" | GraphQL = continuous evolution, vẫn có breaking | Hiểu đúng giới hạn |

### ④ Áp dụng + bigtech
- **GitHub** dùng version theo ngày qua header (`X-GitHub-Api-Version`) cho REST. **Stripe** date-based + per-account pin + transform. **Google** thường URI path (`/v1/`). **GraphQL (Shopify, GitHub GraphQL)** evolve + `@deprecated`. → **Không có một chuẩn duy nhất**; chọn theo đối tượng client & quy mô. *(verify từng hãng)*

### ⑤ Code thực hành

```ts
// URI path versioning + deprecation headers (RFC 8594 Sunset)
const v1 = express.Router();
v1.use((req, res, next) => {
  res.set('Deprecation', 'true');                       // đã deprecated
  res.set('Sunset', 'Wed, 31 Dec 2026 23:59:59 GMT');   // ngày tắt (RFC 8594)
  res.set('Link', '</v2/docs>; rel="successor-version"');
  next();
});
v1.get('/users', listUsersV1);
app.use('/v1', v1);
app.use('/v2', v2Router); // additive evolution; chỉ tách v2 khi thật sự breaking
```

> ⚠️ **AI hay sai ở "additive":** AI đề xuất bump `/v2` chỉ vì thêm một field optional — thừa thãi. Người hiểu sẽ giữ `/v1` và thêm field optional (non-breaking).

### ⑥ Keywords
- 🧠 **HIỂU:** breaking vs non-breaking; chi phí N version; transformation layer (Stripe); "versionless" GraphQL.
- 📌 **THUỘC:** 4 cách version + ưu nhược; header `Deprecation`/`Sunset` (RFC 8594); `@deprecated` (GraphQL).
- 🛠️ **LÀM ĐƯỢC:** quyết định bump hay additive; soạn deprecation policy; cài deprecation header + telemetry.

### ⑦ Mental model
**"Mặc định additive; bump major là vũ khí cuối; tắt version cũ phải có timeline + telemetry."**

### ⑧ Phỏng vấn & thách đố
1. Các cách version + ưu nhược? 
2. Thêm field optional có breaking không? → không.
3. Deprecation policy gồm gì? → thông báo + Sunset + migration guide + telemetry.
4. *(khó)* Stripe giải quyết gì với date-based + transform? → client cũ không đổi code; đổi lại layer transform phức tạp.
5. *(khó)* "GraphQL không cần version" đúng đến đâu? → continuous evolution, vẫn có breaking khi remove/đổi type.

**Thách đố:** "Đổi `status: number` thành `status: string` trong v1 cho gọn" — sao? → breaking (đổi kiểu); phải v2 hoặc thêm field mới song song.

### ⑨ Bài tập + tự chấm
- **BT:** Cho 6 thay đổi (thêm field, xóa field, đổi enum nghĩa, thêm endpoint, thêm required param, đổi 200→201) → phân loại breaking/non-breaking + cách triển khai. *Đạt khi:* phân loại đúng + nêu deprecation cho cái breaking.

### ⑩ Đọc thêm
RFC 8594 (Sunset header). Stripe API versioning docs. "GraphQL Best Practices: Versioning" (graphql.org).

---

## BÀI 7 — Error handling & chuẩn hóa response (RFC 9457)
> Phủ: F-ERR-001…008

### ① Mục tiêu & vị trí
"Hợp đồng lỗi" quan trọng ngang hợp đồng thành công. Học xong bạn thiết kế **error envelope chuẩn**, biết **RFC 9457 Problem Details**, phân biệt **business code vs HTTP status**, chọn **400 vs 422**, **không leak** internal mà vẫn debug được, xử lý lỗi **tập trung**, phân loại lỗi để log/alert đúng, và lỗi cho **async/batch/webhook**. *(Ranh giới: đây là hợp đồng lỗi — không phải chi tiết wiring Exception Filter của Nest ở mục H.)*

### ② Giảng cơ bản → nâng cao

**(a) Vì sao chuẩn hóa + body tối thiểu (F-ERR-001).** Mỗi endpoint trả lỗi một kiểu → client phải viết N parser, dễ vỡ. Body lỗi tối thiểu cần:
- **code máy-đọc** ổn định (vd `INSUFFICIENT_FUNDS`),
- **message người-đọc**,
- (tùy) **details/field errors**,
- **trace/correlation id** để map sang log.

**(b) RFC 9457 Problem Details (F-ERR-002).** Chuẩn IETF cho error body REST. Media type **`application/problem+json`**. Field chuẩn:
- `type` (URI định danh loại lỗi), `title` (tóm tắt), `status` (HTTP status lặp lại), `detail` (mô tả ca cụ thể), `instance` (URI ca lỗi). Cho phép **extension members** (thêm field riêng như `errors`, `traceId`).
- **9457 obsoletes 7807** (cùng nội dung, 9457 là bản cập nhật/làm rõ). *(verify)*

```json
// application/problem+json (RFC 9457) + extension
{
  "type": "https://api.x.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Balance 10 < required 50",
  "instance": "/accounts/123/transfers/abc",
  "code": "INSUFFICIENT_FUNDS",
  "traceId": "9f1c..."
}
```

**(c) Business code vs HTTP status (F-ERR-003).** HTTP status = **transport, coarse-grained** (422 nói "không xử lý được"). Business code = **chi tiết ổn định** để client `switch` logic mà **không phụ thuộc message** (message đổi/đa ngôn ngữ không vỡ client). Cần **cả hai**: status cho máy/middleware, code cho business logic.

**(d) 400 vs 422 (F-ERR-004).** 
- **400 Bad Request**: malformed / parse-fail (JSON hỏng, kiểu sai cú pháp).
- **422 Unprocessable Entity**: cú pháp hợp lệ nhưng **sai nghiệp vụ/validation ngữ nghĩa** (email đúng format nhưng đã tồn tại). *Có tranh luận* — nhiều API chỉ dùng 400 cho tất cả; chọn một convention và nhất quán.
- Trả **field-level errors** dạng mảng `{field/path, code, message}` để client highlight đúng ô.

**(e) Không leak gì + vẫn debug (F-ERR-005).** Không lộ **stack trace / SQL / internal path / secret / tên service nội bộ**. Cân bằng: 5xx trả **message generic** cho client + gắn **correlation/trace id**; **chi tiết để ở log** (map qua trace id). Đây là điểm bảo mật + observability gặp nhau.

**(f) Xử lý tập trung (F-ERR-006).** Dùng **global error handler/middleware/filter** map exception → response chuẩn, thay vì try-catch rải rác. Một nơi quyết định format + status + có log/alert không.

**(g) Phân loại lỗi (F-ERR-007):**

| Loại | Status | Log level | Alert? | Xử lý |
|---|---|---|---|---|
| **Expected/business** (số dư thiếu) | 4xx | info/warn | Không | trả code rõ |
| **Unexpected/bug** (null pointer) | 5xx | error | **Có** | fix code |
| **Transient** (DB timeout, downstream 503) | 5xx/503 | warn | tùy ngưỡng | **retry/circuit breaker** |

→ Đừng alert vào lỗi business (gây mỏi cảnh báo); đừng nuốt lỗi bug thành 200.

**(h) Lỗi async/batch/webhook (F-ERR-008).** Batch một số item fail → **partial success** (207 / mảng kết quả per-item), **không fail-all** (nối Bài 5). Webhook → **retry + consumer dedupe** (nối Bài 8 & 15). Phân biệt lỗi **cả-batch** (toàn bộ reject) vs **một-item**.

### ③ ⚠️ Cũ → mới

| Cũ / hay gặp | Nên dùng | Vì sao |
|---|---|---|
| Mỗi endpoint một format lỗi | Envelope chuẩn / **RFC 9457** | Client parse nhất quán |
| Trả stack trace ra response | Message generic + trace id; chi tiết ở log | Bảo mật + vẫn debug |
| Chỉ dựa HTTP status | Status + **business code** ổn định | Client switch logic không vỡ vì message |
| try-catch rải rác | Global error handler | Nhất quán, ít lặp |
| RFC 7807 | **RFC 9457** | 9457 obsoletes 7807 |

### ④ Áp dụng + bigtech
- **Stripe** error có `type` + `code` + `message` + `param` (field) — gần tinh thần Problem Details. **Google API** dùng `google.rpc.Status` (code + message + details). Nhiều API mới adopt `application/problem+json`. *(pattern; verify chi tiết tại docs từng hãng)*
- TL ép một **error contract chung** toàn org (Bài 15 governance): cùng envelope, cùng cách trả field errors, cùng trace id.

### ⑤ Code thực hành

```ts
// Global error handler trả RFC 9457 + không leak + trace id
class AppError extends Error {
  constructor(public status, public code, msg, public details) { super(msg); }
}
app.use((err, req, res, next) => {
  const traceId = req.headers['x-trace-id'] ?? crypto.randomUUID();
  if (err instanceof AppError) {           // expected/business -> 4xx
    return res.status(err.status)
      .type('application/problem+json')
      .json({ type: `https://api.x.com/errors/${err.code.toLowerCase()}`,
              title: err.message, status: err.status,
              code: err.code, errors: err.details, traceId });
  }
  // unexpected/bug -> 5xx generic + log chi tiết (KHÔNG trả ra ngoài)
  logger.error({ traceId, err });          // stack/SQL ở log, không ở response
  res.status(500).type('application/problem+json')
     .json({ type:'about:blank', title:'Internal Server Error', status:500, traceId });
});
```

> ⚠️ **AI hay sai:** generate handler trả `err.stack`/`err.message` của lỗi DB ra response cho "dễ debug" → **lộ schema/SQL/secret**. Người hiểu phải tách: client thấy generic + traceId, log giữ chi tiết.

### ⑥ Keywords
- 🧠 **HIỂU:** business code vs HTTP status; 400 vs 422; không-leak vs debug-được; phân loại business/bug/transient.
- 📌 **THUỘC:** RFC 9457 field (`type/title/status/detail/instance`) + `application/problem+json`; 9457 obsoletes 7807; 207 cho batch.
- 🛠️ **LÀM ĐƯỢC:** viết global error handler chuẩn; trả field-level errors; gắn trace id.

### ⑦ Mental model
**"Một envelope lỗi cho cả app: status cho máy, code cho business, message cho người, trace id cho log — và không bao giờ leak internal."**

### ⑧ Phỏng vấn & thách đố
1. Error body tối thiểu gồm gì? → code, message, (details), trace id.
2. RFC 9457 thay chuẩn nào, media type gì? → 7807; `application/problem+json`.
3. Vì sao cần business code lẫn HTTP status? → status coarse; code ổn định để switch logic.
4. *(khó)* 400 vs 422? → malformed vs hợp lệ-cú-pháp-sai-nghiệp-vụ (có tranh luận).
5. *(khó)* Cân bằng không-leak vs debug production? → generic + trace id ra ngoài, chi tiết ở log.

**Thách đố:** "Để dễ debug, trả luôn SQL lỗi cho client." Sai? → leak schema/khả năng injection recon; dùng trace id thay thế.

### ⑨ Bài tập + tự chấm
- **BT:** Thiết kế error contract cho một app + viết global handler. *Đạt khi:* RFC 9457 hoặc envelope nhất quán, field-level errors, trace id, không leak, phân loại log/alert.

### ⑩ Đọc thêm
RFC 9457. Stripe error docs. Google `google.rpc.Status`. Zalando RESTful API Guidelines (error section).

---

## BÀI 8 — Idempotency
> Phủ: F-IDEM-001…008

### ① Mục tiêu & vị trí
Mạng không đáng tin: client retry, double-submit, timeout. Idempotency là cách **đảm bảo lặp request không gây hại** (double charge). Học xong bạn hiểu idempotency theo spec, vì sao POST payment cần nó, cơ chế `Idempotency-Key`, xử lý **race song song**, **fingerprint body**, **TTL/scope**, vì sao "đổi POST thành PUT" không phải lúc nào cũng được, và quan hệ với **exactly-once**. Tổng hợp Bài 1 (method semantics) + Bài 2 (409) + Bài 5 (job).

### ② Giảng cơ bản → nâng cao

**(a) Định nghĩa (F-IDEM-001).** Idempotent = lặp cùng request → **cùng hiệu ứng cuối** lên server như gọi 1 lần. Theo RFC 9110: **GET/HEAD/OPTIONS/PUT/DELETE idempotent**, **POST/PATCH không**. Phân biệt **safe** (không đổi state) vs **idempotent** (lặp an toàn) — Bài 1.

**(b) Vì sao POST payment cần idempotent (F-IDEM-002).** Client gửi `POST /charges`, mạng timeout — client **không biết** server đã xử lý chưa. Nếu retry mù → **double charge**. Cần cơ chế để server nhận ra "đây là retry của request cũ" dù nó *trông như* request mới.

**(c) `Idempotency-Key` header (F-IDEM-003).** Client sinh **key duy nhất / mỗi operation** (vd UUID), gửi kèm. Server lưu **key → response**. Lần sau cùng key → **replay response cũ** thay vì làm lại. → Đây là **IETF Internet-Draft** (`draft-ietf-httpapi-idempotency-key-header`, chưa thành RFC); Stripe/Adyen/Dwolla dùng **de-facto**. *(verify)*

**(d) Race song song (F-IDEM-004).** Hai request **cùng key đến cùng lúc** (double-click nhanh). Nếu "check rồi mới insert" → race, cả hai cùng thấy "chưa có" → xử lý 2 lần. Giải: **claim key nguyên tử** bằng **unique constraint** / `INSERT ... ON CONFLICT DO NOTHING` (Postgres) / `SET NX` (Redis). Request thắng claim → xử lý; request thua → trả **409 "đang xử lý"** hoặc chờ rồi replay. Tránh check-then-act.

**(e) Fingerprint body (F-IDEM-005).** Cùng key nhưng **body khác** = client dùng lại key sai (hoặc tấn công). Lưu **fingerprint/checksum của payload** cùng key; nếu key tái dùng với payload khác → trả **422 (key reuse mismatch)**, không trộn hai thao tác.

**(f) TTL & scope (F-IDEM-006).** Key có **TTL hữu hạn** (vd 24h) — không giữ vĩnh viễn (tốn storage; sau lâu key có thể mang nghĩa khác). Scope **per-endpoint / per-account / per-key** để key của user A không đụng user B.

**(g) PUT idempotent "miễn phí", POST phải tự làm (F-IDEM-007).** PUT idempotent vì client **cung cấp toàn bộ state tại id xác định** (`PUT /users/123` → đặt user 123 = body; lặp lại = cùng kết quả). POST sinh **id server-side** → lặp = resource mới. Có nên cứ đổi POST→PUT để khỏi xử lý? **Chỉ khi client biết/sinh được id** (client-generated id, vd UUID v4/v7). Nếu id do server sinh, không đổi được → phải dùng Idempotency-Key.

**(h) Exactly-once có tồn tại? (F-IDEM-008).** **Exactly-once *delivery* là ảo tưởng** trong distributed system (FLP/two generals). Thực tế: **at-least-once delivery + idempotent consumer = effectively-once**. Đây là nối với **dedupe / outbox pattern / transactional messaging** (mục messaging). Hiểu câu này phân biệt senior với mid.

### ③ ⚠️ Cũ → mới

| Cũ / sai | Đúng | Vì sao |
|---|---|---|
| Retry POST payment mù | Idempotency-Key + replay | Tránh double charge |
| Check-then-insert key | Claim nguyên tử (unique constraint / ON CONFLICT / SET NX) | Tránh race song song |
| Cùng key, body khác → cứ chạy | Fingerprint → 422 nếu mismatch | Không trộn thao tác |
| Tin "exactly-once delivery" | at-least-once + idempotent = effectively-once | Phản ánh đúng distributed reality |

### ④ Áp dụng + bigtech
- **Stripe**: `Idempotency-Key` header, lưu key 24h, replay response, fingerprint body. **PayPal/Adyen/Dwolla** tương tự. → de-facto standard ngành payment. *(verify chi tiết tại stripe.com/docs/api/idempotent_requests)*
- Kiến trúc tối thiểu: bảng `idempotency_keys(key PK, account_id, fingerprint, status, response, created_at)` + unique constraint trên `(account_id, key)`; hoặc Redis `SET key NX EX 86400`.

### ⑤ Code thực hành

```ts
// Idempotency-Key với claim nguyên tử (Postgres) + fingerprint + replay
app.post('/charges', async (req, res) => {
  const key = req.header('Idempotency-Key');
  if (!key) return res.status(400).json({ code: 'IDEMPOTENCY_KEY_REQUIRED' });
  const fp = sha256(JSON.stringify(req.body)); // fingerprint payload

  // Claim nguyên tử: chỉ 1 request chèn được hàng 'processing'
  const claim = await db.query(
    `INSERT INTO idempotency_keys(account_id, key, fingerprint, status)
     VALUES ($1,$2,$3,'processing')
     ON CONFLICT (account_id, key) DO NOTHING RETURNING id`,
    [req.accountId, key, fp]);

  if (claim.rowCount === 0) {                 // key đã tồn tại
    const row = await db.oneOrNone(
      `SELECT * FROM idempotency_keys WHERE account_id=$1 AND key=$2`,
      [req.accountId, key]);
    if (row.fingerprint !== fp)               // cùng key, body khác
      return res.status(422).json({ code: 'IDEMPOTENCY_KEY_REUSE' });
    if (row.status === 'processing')          // race: đang xử lý
      return res.status(409).json({ code: 'REQUEST_IN_PROGRESS' });
    return res.status(row.response_status).json(row.response_body); // replay
  }

  const charge = await processPayment(req.body); // xử lý thật, đúng 1 lần
  await db.none(`UPDATE idempotency_keys SET status='done',
                 response_status=201, response_body=$1 WHERE account_id=$2 AND key=$3`,
                [charge, req.accountId, key]);
  res.status(201).json(charge);
});
```

> ⚠️ **AI hay sai:** generate idempotency bằng `SELECT ... ; if not exists then INSERT` (check-then-act) — đúng cú pháp, **vỡ khi 2 request song song** (race). Người hiểu phải dùng claim nguyên tử (ON CONFLICT / unique constraint / SET NX).

### ⑥ Keywords
- 🧠 **HIỂU:** idempotent vs safe; vì sao POST payment cần key; race check-then-act; effectively-once.
- 📌 **THUỘC:** method nào idempotent (RFC 9110); `Idempotency-Key` (IETF draft); fingerprint; TTL/scope; 409/422 ở các nhánh.
- 🛠️ **LÀM ĐƯỢC:** cài Idempotency-Key có claim nguyên tử + fingerprint + replay; biết khi nào dùng PUT thay POST.

### ⑦ Mental model
**"Mạng sẽ retry — thiết kế để lặp không hại: PUT/DELETE miễn phí, POST cần Idempotency-Key + claim nguyên tử; exactly-once = at-least-once + idempotent."**

### ⑧ Phỏng vấn & thách đố
1. Method nào idempotent? → GET/HEAD/OPTIONS/PUT/DELETE; POST/PATCH không.
2. `Idempotency-Key` hoạt động sao? → key→response, replay; là IETF draft, Stripe de-facto.
3. Hai request cùng key song song? → claim nguyên tử; kẻ thua nhận 409 hoặc chờ replay.
4. *(khó)* Cùng key, body khác → ? → 422 (fingerprint mismatch).
5. *(khó)* "Exactly-once delivery" có thật? → không; at-least-once + idempotent = effectively-once.

**Thách đố:** "Tôi đổi hết POST thành PUT để khỏi lo idempotency." Đúng không? → chỉ khi client sinh được id; id server-side thì không, vẫn cần key.

### ⑨ Bài tập + tự chấm
- **BT:** Cài endpoint tạo payment idempotent. *Đạt khi:* claim nguyên tử (không check-then-act), fingerprint→422, replay response cũ, TTL, race-safe.

### ⑩ Đọc thêm
RFC 9110 §9.2.2. `draft-ietf-httpapi-idempotency-key-header` (datatracker). Stripe idempotent requests docs. "You Cannot Have Exactly-Once Delivery" (Tyler Treat).

---

# MODULE 3 — COLLECTION Ở QUY MÔ LỚN

---

## BÀI 9 — Pagination / filtering / sorting
> Phủ: F-PAG-001…008

### ① Mục tiêu & vị trí
Mọi list endpoint đều cần phân trang. Học xong bạn phân biệt **offset vs cursor/keyset**, biết nhược điểm cursor, đặt **metadata** ở đâu, thiết kế **filter/sort an toàn** (whitelist, chống injection), xử lý **total count đắt**, và đảm bảo **ổn định khi data thay đổi**. Giao thoa với mục database (index).

### ② Giảng cơ bản → nâng cao

**(a) Offset/limit (F-PAG-001).** `?page=3&limit=20` → SQL `OFFSET 40 LIMIT 20`. Đơn giản, jump trang bất kỳ được. **Hai nhược điểm chí mạng:**
- **Chậm khi offset lớn**: DB phải **scan rồi bỏ** 40 (hay 40.000) dòng đầu → O(offset).
- **Data shift**: có insert/delete giữa lúc lật trang → **trùng hoặc sót** record.

**(b) Cursor/keyset (F-PAG-002).** Thay vì "bỏ N dòng", nhớ **điểm dừng**: `WHERE (created_at, id) < (:last_created, :last_id) ORDER BY created_at DESC, id DESC LIMIT 20`. 
- Tận dụng **index** trên `(created_at, id)` → không scan-bỏ → nhanh & ổn định ngay cả khi data thay đổi. Lý tưởng cho **dataset lớn / real-time feed**.
- `cursor` thường **opaque** (base64 của `(sort_key, id)`), client không tự chế.

**(c) Nhược điểm cursor (F-PAG-003):**
- **Không jump tới "trang N" bất kỳ** (chỉ next/prev).
- Cần **sort key ổn định + tie-breaker** (id) khi sort key trùng (vd nhiều record cùng `created_at`).
- **Khó tính total count** chính xác.

**(d) Metadata ở đâu (F-PAG-004).** Hai lựa chọn:
- **Envelope** `{ data: [...], meta: { next_cursor, has_more } }`.
- **`Link` header (RFC 8288)**: `Link: </items?cursor=abc>; rel="next"` (GitHub dùng).
- Cân nhắc **bỏ `total`** vì đắt; trả `has_more` thay vì total chính xác.

**(e) Filtering an toàn (F-PAG-005).** `?status=active&created_after=...`. Rủi ro: client filter lên **column nhạy cảm/không index** → injection hoặc full scan. Giải: **whitelist** field được filter + ánh xạ sang column thật; validate operator; không nối thẳng vào SQL.

**(f) Sorting động an toàn (F-PAG-006).** `?sort=-created_at,name` (`-` = desc). Implement: **parse + whitelist** cột sort, ánh xạ sang cột có index; **không nối thẳng** input vào `ORDER BY` (injection). Sort phải **deterministic** → luôn kèm `id` làm tie-breaker (nếu không, cursor pagination vỡ).

**(g) Total count đắt (F-PAG-007).** `COUNT(*)` trên bảng lớn ≈ gần full scan → chậm. Giải pháp:
- **Estimate**: Postgres `reltuples` từ `pg_class` (xấp xỉ, rất nhanh).
- Trả **`has_more`** thay vì total chính xác.
- **Count tách/cache** (job định kỳ) nếu thật cần con số.

**(h) Ổn định khi list thay đổi liên tục (F-PAG-008).** Feed đang được insert liên tục → offset gây trùng/sót. **Keyset** theo sort key gần-immutable (id giảm dần / created_at) + cursor encode điểm dừng chính xác → không trùng/sót.

### ③ ⚠️ Cũ → mới

| Cũ / hay gặp | Nên dùng | Vì sao |
|---|---|---|
| Offset/limit cho mọi list | **Cursor/keyset** cho dataset lớn / feed | Tránh O(offset) + data shift |
| Luôn trả `total` | `has_more` / estimate (`reltuples`) | COUNT(*) đắt |
| Nối `sort`/`filter` thẳng vào SQL | Whitelist + map column + tie-breaker | Chống injection + deterministic |
| Sort không kèm id | Luôn thêm `id` tie-breaker | Cursor cần thứ tự xác định |

### ④ Áp dụng + bigtech
- **GitHub/Stripe/Slack** dùng **cursor-based** cho list lớn (Stripe: `starting_after`/`ending_before`; GitHub: `Link` header). Twitter/X timeline = keyset. → cursor là chuẩn ngầm cho feed quy mô lớn. *(verify chi tiết tại docs từng hãng)*
- Index hỗ trợ: composite index `(sort_col, id)` đúng chiều ORDER BY.

### ⑤ Code thực hành

```ts
// Cursor/keyset pagination + sort whitelist (Postgres)
const SORTABLE = { created_at: 'created_at', name: 'name' }; // whitelist
app.get('/items', async (req, res) => {
  const limit = Math.min(+req.query.limit || 20, 100);
  const sortKey = SORTABLE[(req.query.sort || 'created_at').replace('-', '')] ?? 'created_at';
  const cursor = req.query.cursor ? decodeCursor(req.query.cursor) : null; // {ts,id}

  // keyset: lấy bản ghi "sau" cursor, luôn kèm id làm tie-breaker
  const rows = await db.any(
    `SELECT * FROM items
     ${cursor ? `WHERE (${sortKey}, id) < ($1, $2)` : ''}
     ORDER BY ${sortKey} DESC, id DESC
     LIMIT $3`,
    cursor ? [cursor.ts, cursor.id, limit + 1] : [limit + 1]);

  const has_more = rows.length > limit;
  const data = rows.slice(0, limit);
  const next = has_more ? encodeCursor({ ts: data.at(-1)[sortKey], id: data.at(-1).id }) : null;
  res.json({ data, meta: { next_cursor: next, has_more } }); // không trả total
});
```

> ⚠️ **AI hay sai:** generate `ORDER BY ${req.query.sort}` nối thẳng input → **SQL injection** + sort không deterministic. Người hiểu phải whitelist + tie-breaker id.

### ⑥ Keywords
- 🧠 **HIỂU:** O(offset) & data shift; vì sao keyset ổn định; vì sao total đắt.
- 📌 **THUỘC:** offset vs cursor; `Link` header (RFC 8288); `reltuples`; tie-breaker id; whitelist sort/filter.
- 🛠️ **LÀM ĐƯỢC:** cài cursor pagination + sort/filter an toàn + index phù hợp.

### ⑦ Mental model
**"List lớn dùng keyset (nhớ điểm dừng, không bỏ N dòng); sort/filter luôn whitelist + tie-breaker id; total thì estimate hoặc has_more."**

### ⑧ Phỏng vấn & thách đố
1. Offset nhược điểm gì? → chậm offset lớn + trùng/sót khi data đổi.
2. Cursor tốt hơn vì? → dùng index, không scan-bỏ, ổn định.
3. Cursor nhược điểm? → không jump trang N, cần tie-breaker, khó total.
4. *(khó)* Total đắt xử lý sao? → estimate/has_more/cache.
5. *(khó)* `?sort=...` an toàn thế nào? → whitelist + map column + không nối SQL + id tie-breaker.

**Thách đố:** "Feed real-time, user báo thấy bài trùng khi cuộn." Nguyên nhân? → offset pagination + data shift; chuyển keyset.

### ⑨ Bài tập + tự chấm
- **BT:** Cài `/posts` cursor pagination, sort whitelist `created_at|likes`, filter `author`. *Đạt khi:* keyset + tie-breaker id, opaque cursor, không injection, không trả total.

### ⑩ Đọc thêm
RFC 8288 (Web Linking). "Pagination" — Stripe/GitHub API docs. "We need tool support for keyset pagination" (Markus Winand, use-the-index-luke).

---

## BÀI 10 — Rate limiting (thuật toán + phân tán)
> Phủ: F-RL-001…008

### ① Mục tiêu & vị trí
Bảo vệ API khỏi abuse/DoS/cost. Học xong bạn phân biệt **rate limit / quota / throttling**, 4 thuật toán (**token/leaky bucket, fixed/sliding window**), bài toán **đếm phân tán**, chọn **key** (IP vs user), header khi vượt limit, **đặt limiter ở đâu**, và thiết kế **công bằng chống noisy-neighbor**. *(Góc nhìn ở đây: thiết kế API + thuật toán + phân tán — không lặp wiring thư viện ở mục Security.)*

### ② Giảng cơ bản → nâng cao

**(a) Rate limit vs quota vs throttling (F-RL-001):**
- **Rate limiting**: req / đơn vị thời gian (100 req/phút) — chống burst/abuse.
- **Quota**: tổng theo billing period (10k req/tháng) — gắn pricing.
- **Throttling**: **làm chậm** thay vì chặn hẳn (queue/delay) — degrade mượt.

**(b) 4 thuật toán (F-RL-002, F-RL-003):**

| Thuật toán | Cách hoạt động | Đặc tính |
|---|---|---|
| **Token bucket** | Bucket chứa token, refill đều; mỗi req tốn 1 token | **Cho phép burst** tới dung lượng bucket; phổ biến nhất |
| **Leaky bucket** | Request vào queue, "rỉ" ra với rate cố định | **Làm mượt output** (smoothing), không cho burst |
| **Fixed window** | Đếm req trong cửa sổ cố định (mỗi phút reset) | Đơn giản; **lỗi burst ở ranh giới** |
| **Sliding window** (log / counter) | Tính theo cửa sổ trượt theo thời gian thật | Mượt hơn, công bằng hơn; tốn hơn |

**Burst ở ranh giới (F-RL-003):** fixed window cho 100/phút — client gửi 100 vào giây 0:59 **và** 100 vào giây 1:01 → **~200 req trong ~2 giây** quanh mốc reset (gấp đôi limit). **Sliding window** khắc phục: sliding log (lưu timestamp từng req) hoặc weighted counter (nội suy giữa 2 window) → trải đều theo thời gian thật.

**(c) Đếm phân tán (F-RL-004).** Nhiều instance, mỗi instance đếm **in-memory** riêng → tổng vượt limit thật (3 instance × 100 = 300 thực tế dù limit 100). Giải: **store tập trung** — Redis `INCR` + `EXPIRE` (atomic), hoặc **Lua script** atomic cho token bucket/sliding window; hoặc đẩy việc đếm lên **gateway/edge** (một điểm).

**(d) Limit theo gì (F-RL-005).** 
- **Per-IP**: dễ false-positive (**NAT/proxy/shared IP**, mạng công ty → nhiều user một IP bị chặn oan) và dễ bypass (đổi IP).
- **Per-user / per-API-key**: công bằng hơn, gắn danh tính.
- Thực tế: **tiered + kết hợp** (per-key cho authenticated, per-IP cho anonymous, + global).

**(e) Header khi vượt (F-RL-006).** Trả **`429 Too Many Requests`** + **`Retry-After`** (giây hoặc HTTP-date). Header báo hạn mức: **`RateLimit` / `RateLimit-Policy`** (IETF **draft**, chưa RFC) hoặc de-facto cũ `RateLimit-Limit/Remaining/Reset` vẫn deploy rộng. *(verify tên header — draft)*

**(f) Đặt limiter ở đâu (F-RL-007).** 
- **Edge/gateway** (Nginx/Kong/CDN/Cloudflare): chặn **sớm**, tiết kiệm tài nguyên app, bảo vệ trước cả khi request vào hệ.
- **App-level**: granular theo **business/tenant/endpoint** (mỗi endpoint mỗi limit).
- Thực tế **kết hợp cả hai**: thô ở edge, tinh ở app.

**(g) Công bằng chống noisy-neighbor (F-RL-008).** Một tenant "ồn" làm cạn tài nguyên dùng chung (đặc biệt **DB downstream**). Kết hợp:
- **Per-tenant limit** (cô lập hạn mức).
- **Concurrency limit / bulkhead** (giới hạn số request đồng thời, không chỉ rate).
- **Load shedding** (chủ động drop khi quá tải).
- **Backpressure** (nối Bài 13 / stream).
→ Mục tiêu: một tenant không kéo sập downstream cho mọi tenant.

### ③ ⚠️ Cũ → mới

| Cũ / hay gặp | Nên dùng | Vì sao |
|---|---|---|
| Đếm in-memory mỗi instance | Redis atomic / gateway tập trung | Đa instance không vượt limit thật |
| Fixed window | Sliding window (khi cần chính xác) | Tránh burst ×2 ở ranh giới |
| Chỉ per-IP | Per-key/user + tiered | Tránh false-positive NAT, công bằng |
| Chỉ rate limit | + concurrency/bulkhead/load shedding | Bảo vệ downstream, chống noisy-neighbor |

### ④ Áp dụng + bigtech
- **GitHub** dùng `RateLimit-*` header + 429. **Stripe** rate limit per-key. **Cloudflare/Kong/Envoy** làm rate limit ở edge với store tập trung. Token bucket là default phổ biến (cho phép burst hợp lý). *(pattern; verify header/giới hạn cụ thể tại docs)*

### ⑤ Code thực hành

```ts
// Token bucket phân tán bằng Redis Lua (atomic) — chống race đa instance
const TOKEN_BUCKET = `
local key=KEYS[1]; local rate=tonumber(ARGV[1]); local cap=tonumber(ARGV[2])
local now=tonumber(ARGV[3]); local cost=tonumber(ARGV[4])
local b=redis.call('HMGET',key,'tokens','ts')
local tokens=tonumber(b[1]) or cap; local ts=tonumber(b[2]) or now
tokens=math.min(cap, tokens+(now-ts)*rate)   -- refill theo thời gian trôi
local allowed=0
if tokens>=cost then tokens=tokens-cost; allowed=1 end
redis.call('HMSET',key,'tokens',tokens,'ts',now); redis.call('EXPIRE',key,60)
return {allowed, math.floor(tokens)}`;

app.use(async (req, res, next) => {
  const id = req.apiKey ?? req.ip;             // per-key, fallback per-IP
  const [allowed, remaining] = await redis.eval(
    TOKEN_BUCKET, 1, `rl:${id}`, 10, 100, Date.now()/1000, 1); // 10 tok/s, cap 100
  res.set('RateLimit-Remaining', String(remaining)); // de-facto (verify draft name)
  if (!allowed) return res.status(429).set('Retry-After', '1')
                          .json({ code: 'RATE_LIMITED' });
  next();
});
```

> ⚠️ **AI hay sai:** generate counter `if (count[ip]++ > limit)` trong biến process — đúng cú pháp, **vô dụng khi scale ngang** (mỗi pod đếm riêng) và race. Người hiểu phải đẩy đếm vào store atomic (Redis Lua) hoặc gateway.

### ⑥ Keywords
- 🧠 **HIỂU:** rate vs quota vs throttling; burst ranh giới; vì sao in-memory không đủ; noisy-neighbor.
- 📌 **THUỘC:** 4 thuật toán; 429 + `Retry-After`; `RateLimit`/`RateLimit-Policy` (draft); per-IP cạm bẫy.
- 🛠️ **LÀM ĐƯỢC:** cài token bucket phân tán (Redis atomic); chọn key + đặt limiter đúng tầng.

### ⑦ Mental model
**"Đếm phải tập trung & atomic; token bucket cho burst, sliding window cho chính xác; per-key công bằng hơn per-IP; chống sập downstream cần rate + concurrency + shedding."**

### ⑧ Phỏng vấn & thách đố
1. Rate vs quota vs throttling? 
2. Burst ranh giới của fixed window? → ×2 quanh mốc reset; sliding window khắc phục.
3. Vì sao in-memory không đủ đa instance? → mỗi instance đếm riêng → vượt tổng.
4. *(khó)* Per-IP cạm bẫy? → NAT/shared IP false-positive + dễ bypass.
5. *(khó)* Bảo vệ DB downstream khỏi 1 tenant ồn? → per-tenant limit + concurrency/bulkhead + load shedding.

**Thách đố:** "Rate limit 100/phút mà thỉnh thoảng thấy 190 req lọt." Nguyên nhân? → fixed window burst ranh giới hoặc đếm in-memory đa instance.

### ⑨ Bài tập + tự chấm
- **BT:** Cài rate limit phân tán cho API đa instance. *Đạt khi:* atomic (Redis Lua/INCR), không race, 429 + Retry-After, per-key + fallback IP, giải thích chọn thuật toán.

### ⑩ Đọc thêm
`draft-ietf-httpapi-ratelimit-headers` (datatracker). "How we built rate limiting" (Stripe/Cloudflare engineering blogs). Token bucket / leaky bucket (Wikipedia, để nắm thuật toán).

---

# MODULE 4 — SPEC & CÁC PARADIGM KHÁC REST

---

## BÀI 11 — OpenAPI / Swagger, contract-first, codegen
> Phủ: F-OAS-001…007

### ① Mục tiêu & vị trí
OpenAPI là "ngôn ngữ chung" mô tả REST API máy đọc được. Học xong bạn phân biệt **OpenAPI vs Swagger**, biết bản hiện hành, **design-first vs code-first**, codegen sinh gì + rủi ro, chống **drift** spec↔code, `$ref` tái dùng, OpenAPI mô tả tốt/không tốt cái gì, và dùng trong **CI** (lint + breaking-change). Nối Bài 6 (versioning), Bài 15 (governance, contract test).

### ② Giảng cơ bản → nâng cao

**(a) OpenAPI vs Swagger + bản hiện hành (F-OAS-001).**
- **OpenAPI** = **đặc tả** (specification) — trước 2016 tên là "Swagger Specification", nay do OpenAPI Initiative (Linux Foundation) quản lý.
- **Swagger** = **bộ tool** quanh spec (Swagger UI, Editor, Codegen) của SmartBear.
- **Bản stable hiện hành: OpenAPI 3.2.0** (9/2025) — thêm mô tả native cho **streaming (SSE, JSON Lines)**, hierarchical tags, custom HTTP methods. **3.1** vẫn dùng rộng. **OpenAPI 4.0 "Moonwalk"** đang thiết kế, **chưa release** — đừng nói "4.0 hiện hành". *(verify — đã xác nhận 6/2026)*

**(b) Design-first vs code-first (F-OAS-002):**

| | Design-first (contract-first) | Code-first |
|---|---|---|
| Quy trình | Viết spec trước → generate stub/client | Viết code → generate spec từ annotation |
| Ưu | Đồng thuận sớm; FE/BE **làm song song**; mock từ spec | Nhanh cho team nhỏ; spec luôn khớp code (nếu generate) |
| Nhược | Cần kỷ luật; tooling | Spec dễ **lệch** thực tế nếu viết tay |

→ Public API / nhiều team / cần parallel → **contract-first**. Team nhỏ, nội bộ → code-first chấp nhận được.

**(c) Codegen sinh gì + rủi ro (F-OAS-003).** Sinh **client SDK / server stub / type models** đồng bộ spec. Rủi ro: code sinh ra **xấu/khó tùy biến**, và **drift** nếu quên regenerate khi spec đổi.

**(d) Chống drift (F-OAS-004).** Hai hướng:
- Spec là **source-of-truth** + **runtime/contract validation** (validate request/response theo spec lúc chạy/test).
- Hoặc **generate spec từ code** rồi lint.
- Gắn **CI gate** so spec với hành vi thật (contract test — Bài 15).

**(e) `$ref` & components (F-OAS-005).** `components/schemas` + `$ref` để **tái dùng** model/error/pagination dùng chung → DRY, tránh copy-paste, dễ maintain, nhất quán (vd một `Error` schema RFC 9457 dùng khắp nơi).

**(f) OpenAPI mô tả tốt/không tốt gì (F-OAS-006).**
- **Tốt**: REST đồng bộ — path, param, request/response schema, status, security scheme.
- **Không tốt / cần công cụ khác**: **event-driven/async** → dùng **AsyncAPI**; **business invariant / stateful workflow / ordering / ràng buộc giữa nhiều field** → spec không bắt được (phải tài liệu + test).

**(g) OpenAPI trong CI (F-OAS-007).** 
- **Lint** theo style guide (vd **Spectral**) — ép convention naming/error/pagination.
- **Breaking-change detection**: so spec cũ↔mới (vd `oasdiff`) → **chặn breaking trước merge**.
- Kết hợp **contract test (Pact)** phía consumer (Bài 15).

### ③ ⚠️ Cũ → mới

| Cũ / hay nhầm | Đúng | Vì sao |
|---|---|---|
| "Swagger = spec" | Swagger = tool; **OpenAPI** = spec | Tên đã tách từ 2016 |
| "OpenAPI 4.0 hiện hành" | **3.2.0 stable**; 4.0 đang thiết kế | Tránh nói sai đời spec |
| Spec viết tay, hy vọng khớp code | Source-of-truth + validation/CI gate | Chống drift |
| Copy-paste schema | `$ref` + components | DRY, nhất quán |
| Mô tả async bằng OpenAPI | **AsyncAPI** cho event-driven | OpenAPI cho REST sync |

### ④ Áp dụng + bigtech
- **Stripe/GitHub** publish OpenAPI spec → cộng đồng generate SDK đa ngôn ngữ. Nhiều org dùng **Spectral** trong CI để ép style guide (Bài 15). **AsyncAPI** dùng cho Kafka/event API. *(pattern; verify tooling version khi dùng)*

### ⑤ Code thực hành

```yaml
# openapi.yaml (contract-first) — $ref tái dùng + Problem Details
openapi: 3.2.0            # bản stable hiện hành (verify)
info: { title: Orders API, version: 1.0.0 }
paths:
  /orders/{id}:
    get:
      parameters: [{ name: id, in: path, required: true, schema: { type: string } }]
      responses:
        '200': { description: OK,
          content: { application/json: { schema: { $ref: '#/components/schemas/Order' } } } }
        '404': { description: Not found,
          content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } } }
components:
  schemas:
    Order: { type: object, properties: { id: {type: string}, total: {type: number} } }
    Problem:   # RFC 9457, tái dùng khắp API
      type: object
      properties: { type: {type: string}, title: {type: string},
                    status: {type: integer}, detail: {type: string} }
```

```bash
# CI: lint + breaking-change gate
npx @stoplight/spectral-cli lint openapi.yaml          # style guide
npx oasdiff breaking old.yaml openapi.yaml             # chặn breaking (verify tool)
```

### ⑥ Keywords
- 🧠 **HIỂU:** OpenAPI vs Swagger; design-first vs code-first; drift; OpenAPI vs AsyncAPI.
- 📌 **THUỘC:** 3.2.0 stable / 4.0 chưa ra; `components`+`$ref`; Spectral (lint); oasdiff (breaking).
- 🛠️ **LÀM ĐƯỢC:** viết spec contract-first; cài CI lint + breaking gate; generate client/stub.

### ⑦ Mental model
**"Spec là hợp đồng máy-đọc; contract-first để FE/BE chạy song song; CI lint + breaking-gate để hợp đồng không lệch và không vỡ ngầm."**

### ⑧ Phỏng vấn & thách đố
1. OpenAPI khác Swagger? Bản hiện hành? → spec vs tool; 3.2.0 (4.0 chưa ra).
2. Design-first lợi gì? → đồng thuận sớm + parallel + mock.
3. Codegen rủi ro? → code xấu/khó sửa + drift.
4. *(khó)* Chống drift spec↔code? → source-of-truth + validation + CI gate.
5. *(khó)* OpenAPI không mô tả tốt gì? → event-driven (AsyncAPI), invariant/workflow/ordering.

**Thách đố:** "Ta dùng OpenAPI mô tả luôn Kafka event cho gọn." Ổn? → không; dùng AsyncAPI cho event-driven.

### ⑨ Bài tập + tự chấm
- **BT:** Viết OpenAPI 3.x cho 2 endpoint + Problem schema `$ref`, thêm CI lint Spectral. *Đạt khi:* dùng components/$ref, error theo 9457, lint pass, có breaking-check trong CI.

### ⑩ Đọc thêm
spec.openapis.org/oas/v3.2.0.html. Spectral docs. AsyncAPI.com. oasdiff (GitHub).

---

## BÀI 12 — GraphQL vs REST, DataLoader, federation
> Phủ: F-GQL-001…009

### ① Mục tiêu & vị trí
GraphQL là paradigm thay thế REST cho một số ca. Học xong bạn biết GraphQL **giải quyết gì**, **khi nào chọn** GraphQL vs REST, vì sao **cache khó hơn**, **N+1 resolver + DataLoader**, **DoS query** + phòng, tranh cãi **200 + errors**, **versioning**, **authorization khó** hơn, và **federation** khi gộp nhiều service. *(N+1 ở đây là tầng resolver, khác N+1 database.)*

### ② Giảng cơ bản → nâng cao

**(a) GraphQL giải quyết gì (F-GQL-001).** REST hay bị **over-fetching** (trả thừa field) và **under-fetching** (phải gọi nhiều endpoint để ghép). GraphQL: client **khai báo đúng field cần** trong một query, lấy nhiều resource liên quan **một round-trip**.

**(b) Khi nào GraphQL vs REST (F-GQL-002):**

| Hợp GraphQL | Hợp REST |
|---|---|
| Client đa dạng (web/mobile) cần shape khác nhau | Public API đơn giản, ổn định |
| UI đổi nhanh, cần linh hoạt field | Cần **HTTP/CDN cache** mạnh |
| Aggregate nhiều nguồn/service | File upload/download, streaming đơn giản |
| | Ops/debug đơn giản (curl được) |

**(c) Vì sao cache khó hơn (F-GQL-003).** REST cache theo **URL + method** (HTTP/CDN/browser cache "miễn phí"). GraphQL thường **POST một endpoint `/graphql`** → mất HTTP cache (mọi query cùng URL+method). Giải: **normalized cache phía client** (Apollo/Relay) hoặc **persisted query + GET** (hash query → GET cache được).

**(d) N+1 resolver + DataLoader (F-GQL-004).** Query `users { posts { title } }`: resolver `posts` chạy **cho từng user** → 1 query lấy N user + N query lấy posts = **N+1**. **DataLoader**: gom các key trong **một tick** → **batch** thành 1 query (`WHERE user_id IN (...)`) + **cache trong 1 request**. ⚠️ Khác N+1 phía database (mục E) — đây là tầng **resolver**.

**(e) DoS query (F-GQL-005).** Query lồng sâu/độc hại (`a{b{c{d{...}}}}` hoặc alias bomb) → tốn tài nguyên khủng. Phòng: **depth limit**, **complexity/cost analysis** (gán cost mỗi field), **persisted/allow-listed queries** (chỉ cho query đã duyệt), **timeout**, **rate limit theo cost** thay vì theo số request.

**(f) 200 + mảng errors (F-GQL-006).** GraphQL truyền thống trả **200** + field `errors` (kể cả khi lỗi). Tranh cãi: **khó tận dụng HTTP status** (200 cho cả lỗi → monitoring/CDN/middleware hiểu sai); có thể **partial data + errors** cùng response; client/tooling **phải đọc `errors` field**. (GraphQL-over-HTTP spec mới cho phép dùng status code linh hoạt hơn — *verify*.)

**(g) Versioning (F-GQL-007).** Evolve bằng **thêm field + `@deprecated`**, mở rộng additive → "versionless". Nhưng **remove field / đổi type vẫn breaking** (nối Bài 6).

**(h) Authorization khó hơn (F-GQL-008).** REST check quyền **per-endpoint** (gọn). GraphQL: client tự ghép field/quan hệ → phải authz **field-level / type-level trong resolver**; chỉ check ở root → **lộ data qua nested traversal** (vd `me { friends { salary } }`). Đây là bẫy bảo mật điển hình.

**(i) Federation (F-GQL-009).** Nhiều service muốn ghép thành **một GraphQL endpoint**: **Apollo Federation / subgraph** (hoặc schema stitching cũ) — gateway compose schema từ các subgraph. Trade-off: **latency fan-out** (gateway gọi nhiều subgraph), coupling schema, **độ phức tạp vận hành** cao.

### ③ ⚠️ Cũ → mới / bẫy

| Quan niệm | Thực tế |
|---|---|
| "GraphQL thay REST mọi nơi" | Chỉ hợp ca client đa dạng/aggregate; REST vẫn tốt cho public/cache/file |
| Resolver gọi DB thẳng | **DataLoader** batch+cache để tránh N+1 |
| Check authz ở root là đủ | Cần field/type-level (nested traversal lộ data) |
| GraphQL miễn nhiễm DoS | Cần depth/complexity limit + persisted query |
| Schema stitching | **Apollo Federation** (subgraph) hiện đại hơn |

### ④ Áp dụng + bigtech
- **GitHub** có cả REST và GraphQL API. **Shopify** dùng GraphQL mạnh (cost-based rate limit). **Netflix/Apollo** dùng Federation gộp nhiều subgraph. **Meta** tạo GraphQL ban đầu cho mobile (under/over-fetching). *(pattern; verify chi tiết tại docs)*

### ⑤ Code thực hành

```ts
// DataLoader chống N+1 ở resolver (Apollo Server)
import DataLoader from 'dataloader';
const postLoader = new DataLoader(async (userIds: string[]) => {
  const rows = await db.any(`SELECT * FROM posts WHERE user_id IN ($1:csv)`, [userIds]);
  const byUser = groupBy(rows, 'user_id');           // gom 1 query thành map
  return userIds.map(id => byUser[id] ?? []);        // trả đúng thứ tự key
});
const resolvers = {
  User: { posts: (user) => postLoader.load(user.id) } // batch trong 1 request
};
// + depth limit + cost analysis ở server config để chống DoS (verify plugin version)
```

> ⚠️ **AI hay sai (kép):** (1) generate resolver gọi `db.query` trực tiếp → **N+1**; (2) check quyền chỉ ở root resolver → **lộ data nested**. Người hiểu phải dùng DataLoader + authz field-level.

### ⑥ Keywords
- 🧠 **HIỂU:** over/under-fetching; vì sao cache khó; N+1 resolver vs N+1 DB; nested-traversal authz leak.
- 📌 **THUỘC:** DataLoader (batch+cache/request); depth/complexity limit; persisted query; `@deprecated`; Apollo Federation/subgraph.
- 🛠️ **LÀM ĐƯỢC:** cài DataLoader; thêm depth/cost limit; authz field-level; quyết định GraphQL vs REST.

### ⑦ Mental model
**"GraphQL = client tự chọn shape (đổi lại: cache khó, authz khó, dễ N+1/DoS); REST = đơn giản + cache mạnh. Chọn theo client, không theo trend."**

### ⑧ Phỏng vấn & thách đố
1. GraphQL giải quyết gì? → over/under-fetching.
2. Vì sao cache khó? → POST một endpoint, mất HTTP cache.
3. N+1 resolver & DataLoader? → resolve từng item → N+1; DataLoader batch+cache/request.
4. *(khó)* DoS query phòng sao? → depth/complexity limit + persisted query + cost-based rate limit.
5. *(khó)* Authz khó hơn REST chỗ nào? → field/type-level; nested traversal lộ data.

**Thách đố:** "GraphQL trả 200 nên alert/monitoring báo all-green dù lỗi tùm lum." Vì sao & fix? → 200+errors; phải đọc `errors` field / dùng GraphQL-over-HTTP status mới.

### ⑨ Bài tập + tự chấm
- **BT:** Cài schema `User/Post` + DataLoader + depth limit + authz field-level. *Đạt khi:* không N+1 (1 batch query), nested authz chặn được, query quá sâu bị từ chối.

### ⑩ Đọc thêm
graphql.org/learn (best practices, caching, security). DataLoader (GitHub). Apollo Federation docs. "GraphQL over HTTP" spec.

---

## BÀI 13 — Async API: WebSocket / SSE / realtime
> Phủ: F-WS-001…007

### ① Mục tiêu & vị trí
Khi cần đẩy dữ liệu chủ động (push) thay vì client poll. Học xong bạn biết **khi nào cần realtime**, phân biệt **WebSocket/SSE/long-polling**, cơ chế SSE, vấn đề **scale WS đa instance**, **auth WS/SSE**, **backpressure/slow consumer**, và **không mất message khi reconnect**. Nối Bài 5 (async) & Bài 10 (backpressure).

### ② Giảng cơ bản → nâng cao

**(a) Khi nào realtime thay poll (F-WS-001).** Cần cập nhật **tức thời** (chat, notification, giá live, collab) và/hoặc giảm tải poll (nhiều client poll dày = lãng phí). Nhưng **poll vẫn đơn giản & đủ** cho nhiều ca (cập nhật mỗi 30s) → **không over-engineer** realtime khi không cần.

**(b) WS vs SSE vs long-polling (F-WS-002):**

| | WebSocket | SSE | Long-polling |
|---|---|---|---|
| Chiều | **Full-duplex** (2 chiều) | **Server→client** 1 chiều (text) | Mô phỏng push qua HTTP request giữ lâu |
| Hợp ca | chat/game/collab/2 chiều | notification/feed/log/LLM token stream | fallback khi không có WS/SSE |
| Hạ tầng | protocol riêng (`Upgrade`) | **chạy trên HTTP thường**, auto-reconnect | HTTP thường, tốn connection |
| Độ phức tạp | Cao hơn | **Đơn giản hơn** | Đơn giản nhưng kém hiệu quả |

→ Nếu chỉ cần **một chiều server→client**, **SSE thường thắng** (đơn giản, qua proxy/HTTP dễ, auto-reconnect).

**(c) SSE cơ chế (F-WS-003).** Content-type **`text/event-stream`**, server giữ HTTP connection mở, gửi event dạng `data: ...\n\n`. Trình duyệt `EventSource` **auto-reconnect** và gửi lại **`Last-Event-ID`** để resume. Nhẹ, qua HTTP/2 dễ. **OpenAPI 3.2** đã mô tả được SSE. *(verify)*

**(d) Scale WS đa instance (F-WS-004).** WS connection **có state ở 1 node** (user A nối pod 1). Muốn broadcast cho A từ pod 2 → pod 2 không có socket của A. Giải:
- **Sticky session** (LB ghim user vào 1 pod) — đơn giản nhưng kém co giãn.
- **Pub/sub** (Redis/NATS): pod nào có message publish lên channel; pod giữ socket của A subscribe và đẩy xuống. → chuẩn cho horizontal scaling.
- **Presence** (ai đang online) cũng phức tạp hơn HTTP stateless.

**(e) Auth WS/SSE (F-WS-005).** WS/SSE không tiện gửi `Authorization` header **mỗi message** như REST per-request. Giải: **xác thực ở handshake** (token qua query param / subprotocol / cookie), verify **khi connect**. Xử lý **token hết hạn**: re-auth hoặc đóng connection (đừng để connection "sống mãi" với token đã hết hạn).

**(f) Backpressure / slow consumer (F-WS-006).** Client **chậm** (mạng yếu/tab nền) → server buffer dồn ứ → **memory phình** → có thể OOM. Giải: **drop/coalesce** (gộp event mới nhất), **giới hạn queue**, hoặc **đóng connection chậm**. Nối với backpressure phía Node stream (mục G).

**(g) Không mất message khi reconnect (F-WS-007).** Mạng rớt → reconnect → có thể **mất event giữa chừng**. Giải:
- Server **đánh id sự kiện**; client gửi **`Last-Event-ID`** (SSE) hoặc resume token → server **replay** từ điểm đó.
- Hoặc **message broker + cursor** (lưu offset như Kafka).
- Mô hình: **at-least-once + dedupe phía client** (nối idempotency Bài 8).

### ③ ⚠️ Cũ → mới / lựa chọn

| Hay gặp | Nên cân nhắc | Vì sao |
|---|---|---|
| Mặc định WS cho mọi realtime | SSE nếu chỉ 1 chiều server→client | Đơn giản hơn, qua HTTP dễ |
| WS đa instance không pub/sub | Sticky **hoặc** Redis/NATS pub/sub | Broadcast cross-node |
| Auth per-message | Auth ở **handshake** + xử lý token expiry | WS/SSE không gửi header mỗi message |
| Bỏ qua slow consumer | drop/coalesce/limit queue | Tránh OOM |
| Reconnect = mất event | event id + `Last-Event-ID`/resume token | At-least-once + dedupe |

### ④ Áp dụng + bigtech
- **Slack/Discord** dùng WS cho chat 2 chiều + presence. **OpenAI/Anthropic streaming** dùng **SSE** để stream token LLM (một chiều). **Server giá chứng khoán/notification** thường SSE. Scale WS qua Redis pub/sub là pattern phổ biến. *(pattern; verify chi tiết)*

### ⑤ Code thực hành

```ts
// SSE một chiều + event id để resume (Express)
app.get('/stream', auth, (req, res) => {       // auth ở handshake (token query/cookie)
  res.set({ 'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache', 'Connection': 'keep-alive' });
  const lastId = req.header('Last-Event-ID');   // client resume từ đây
  const sub = bus.subscribe(req.user.id, lastId, (evt) => {
    // backpressure: nếu client chậm, drop event cũ thay vì buffer vô hạn
    if (!res.writableNeedDrain) {
      res.write(`id: ${evt.id}\n`);
      res.write(`data: ${JSON.stringify(evt.payload)}\n\n`);
    }
  });
  req.on('close', () => sub.unsubscribe());     // dọn khi client ngắt
});
```

> ⚠️ **AI hay sai:** generate SSE/WS handler **ghi không kiểm tra backpressure** và **không cleanup `on('close')`** → memory leak + slow-consumer OOM khi scale. Người hiểu phải check `writableNeedDrain`/drop và unsubscribe khi đóng.

### ⑥ Keywords
- 🧠 **HIỂU:** poll vs push; WS vs SSE 1 chiều; state-ful WS khó scale; backpressure; at-least-once + dedupe.
- 📌 **THUỘC:** `text/event-stream`, `Last-Event-ID`, auth ở handshake, sticky vs pub/sub, OpenAPI 3.2 mô tả SSE.
- 🛠️ **LÀM ĐƯỢC:** cài SSE có resume; scale WS qua Redis pub/sub; xử lý slow consumer + cleanup.

### ⑦ Mental model
**"Một chiều → SSE; hai chiều → WS. WS có state nên cần sticky/pub-sub để scale; reconnect không mất message nhờ event id + replay; client chậm thì drop, đừng buffer vô hạn."**

### ⑧ Phỏng vấn & thách đố
1. Khi nào realtime thay poll? → cần tức thời/giảm tải; poll vẫn đủ nhiều ca.
2. WS vs SSE? → full-duplex vs server→client 1 chiều; SSE đơn giản hơn.
3. Scale WS đa instance gặp gì? → state 1 node → sticky/pub-sub.
4. *(khó)* Auth WS khi không gửi header mỗi message? → handshake + xử lý token expiry.
5. *(khó)* Không mất message khi reconnect? → event id + Last-Event-ID/resume + dedupe.

**Thách đố:** "WS chạy ngon ở 1 server, lên 3 pod thì user lúc nhận lúc không." Vì sao? → connection state ghim 1 pod, thiếu pub/sub broadcast cross-node.

### ⑨ Bài tập + tự chấm
- **BT:** Cài notification SSE có auth handshake + resume (`Last-Event-ID`) + xử lý slow consumer. *Đạt khi:* reconnect không sót event, không leak khi client ngắt, drop khi client chậm.

### ⑩ Đọc thêm
MDN Server-Sent Events / WebSocket. "Scaling WebSockets" (Ably/Socket.IO docs). OpenAPI 3.2 streaming section.

---

## BÀI 14 — gRPC + Protobuf (nội bộ)
> Phủ: F-GRPC-001…007

### ① Mục tiêu & vị trí
gRPC là lựa chọn cho giao tiếp **service-to-service** hiệu năng cao. Học xong bạn biết gRPC **là gì**, **REST vs gRPC** (khi nào nội bộ), **Protobuf vs JSON**, **evolve schema** an toàn, **4 kiểu streaming**, **error/deadline model**, và vì sao **LB L4 không balance tốt** gRPC. *(Góc nhìn: quyết định protocol + thiết kế Protobuf — không lặp transport gRPC trong Nest microservices ở mục H.)*

### ② Giảng cơ bản → nâng cao

**(a) gRPC là gì (F-GRPC-001).** RPC framework (Google): **contract bằng Protobuf** (`.proto`), transport **HTTP/2**, payload **binary**, **codegen** client/server đa ngôn ngữ từ `.proto`. Bạn gọi method như gọi hàm local; gRPC lo serialize + network.

**(b) REST vs gRPC (F-GRPC-002).** 
- gRPC: **nhanh** (binary + HTTP/2 multiplex), **typed** (contract chặt), **streaming** native → tốt cho **service-to-service** nội bộ, latency thấp, high-throughput.
- Vì sao ít expose public/browser: **browser không gọi gRPC trực tiếp** (cần **grpc-web** + proxy như Envoy); REST/JSON **phổ biến, debug dễ (curl), human-readable**, tooling rộng → tốt cho **public API**.

**(c) Protobuf vs JSON (F-GRPC-003).**
- Protobuf: **binary nhỏ + nhanh**, **schema chặt**, **backward-compat qua field number**.
- Bất lợi: **không human-readable**, cần **codegen/tooling** để đọc-debug.

**(d) Evolve Protobuf an toàn (F-GRPC-004) — quy tắc field number:**
- **KHÔNG tái dùng / đổi field number** (number là danh tính trên wire).
- **KHÔNG đổi type** của field đang dùng.
- **Thêm field mới** = thêm number mới (optional).
- **`reserved`** cho field/number đã xóa (cấm tái dùng nhầm).
→ Đảm bảo **backward & forward compatibility** (client cũ bỏ qua field lạ; server cũ bỏ qua field mới).

**(e) 4 kiểu gRPC (F-GRPC-005):**

| Kiểu | Mô tả | Hợp ca |
|---|---|---|
| **Unary** | request → response (như REST call) | CRUD thường |
| **Server-streaming** | 1 request → nhiều response | feed/kết quả lớn/tải xuống dần |
| **Client-streaming** | nhiều request → 1 response | upload từng phần/aggregate |
| **Bidirectional** | 2 chiều đồng thời | chat/realtime 2 chiều |

**(f) Error & deadline (F-GRPC-006).** 
- gRPC có **status code riêng** (`OK`, `NOT_FOUND`, `UNAVAILABLE`, `DEADLINE_EXCEEDED`, `PERMISSION_DENIED`...) — **khác HTTP status**.
- **Deadline/timeout** được **propagate xuyên call chain** (A gọi B gọi C — deadline trôi theo); **cancellation** lan truyền (client hủy → server dừng). 
- **Metadata** (key-value) thay cho HTTP header.
→ Deadline propagation là lợi thế lớn của gRPC cho microservices (chống cascading timeout).

**(g) gRPC sau load balancer (F-GRPC-007).** gRPC dùng **HTTP/2 multiplex** trên **một connection long-lived**. **LB L4** (TCP) ghim cả connection vào **1 backend** → mọi request đi 1 nơi, các backend khác "đói". Cần: **LB L7** (hiểu HTTP/2, balance per-request/stream), hoặc **client-side LB**, hoặc **service mesh** (Envoy/Istio/Linkerd).

### ③ ⚠️ Cũ → mới / bẫy

| Bẫy | Đúng | Vì sao |
|---|---|---|
| Expose gRPC cho browser trực tiếp | grpc-web + proxy, hoặc REST cho public | Browser không gọi gRPC native |
| Tái dùng/đổi field number | Number mới + `reserved` cho cái xóa | Giữ wire compatibility |
| Map gRPC status = HTTP status | gRPC có code riêng | Tránh nhầm error model |
| Đặt gRPC sau LB L4 | LB L7 / client-side LB / mesh | HTTP/2 multiplex ghim 1 backend |

### ④ Áp dụng + bigtech
- **Google** dùng gRPC nội bộ khắp nơi. **Netflix/Uber/Square** dùng gRPC service-to-service + Envoy/mesh để LB. **Kubernetes** API nội bộ dùng Protobuf. Public-facing thường vẫn REST/GraphQL, gRPC ở tầng nội bộ. *(pattern; verify chi tiết)*

### ⑤ Code thực hành

```protobuf
// orders.proto — contract + evolve an toàn
syntax = "proto3";
package orders;
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);                 // unary
  rpc WatchOrders(WatchRequest) returns (stream Order);          // server-streaming
}
message Order {
  string id = 1;
  double total = 2;
  // field 3 đã xoá -> reserved để không ai tái dùng number/tên
  reserved 3; reserved "legacy_status";
  string currency = 4;   // field MỚI: number mới, optional -> non-breaking
}
message GetOrderRequest { string id = 1; }
message WatchRequest { string user_id = 1; }
```

```ts
// Client gọi unary có DEADLINE (propagate xuyên call chain)
const deadline = new Date(Date.now() + 2000); // 2s
client.getOrder({ id: '123' }, { deadline }, (err, order) => {
  if (err?.code === grpc.status.DEADLINE_EXCEEDED) { /* xử lý timeout */ }
});
```

> ⚠️ **AI hay sai:** generate `.proto` rồi đổi `total` từ field 2 sang 3 hoặc tái dùng số đã xóa "cho gọn" → **vỡ client cũ trên wire** dù compile ổn. Người hiểu phải giữ field number + `reserved`.

### ⑥ Keywords
- 🧠 **HIỂU:** gRPC vì sao nhanh/nội bộ; Protobuf trade-off; field number = danh tính wire; deadline propagation; L4 vs L7 LB.
- 📌 **THUỘC:** HTTP/2 + binary + codegen; 4 kiểu streaming; gRPC status codes; `reserved`; grpc-web cho browser.
- 🛠️ **LÀM ĐƯỢC:** viết `.proto` evolve an toàn; set deadline; biết cần mesh/L7 LB cho gRPC.

### ⑦ Mental model
**"gRPC = contract Protobuf + HTTP/2 binary cho nội bộ; field number là bất khả xâm phạm; deadline trôi xuyên call; muốn LB phải L7/mesh chứ không L4."**

### ⑧ Phỏng vấn & thách đố
1. gRPC dựa trên gì? → Protobuf + HTTP/2 + binary + codegen.
2. Khi nào gRPC nội bộ, vì sao ít public? → service-to-service nhanh/typed; browser cần grpc-web.
3. Evolve Protobuf không phá client? → giữ field number, thêm field mới, `reserved`.
4. *(khó)* gRPC sau LB L4 sao hỏng? → HTTP/2 multiplex ghim 1 backend; cần L7/mesh.
5. *(khó)* Deadline gRPC khác timeout REST chỗ nào? → propagate + cancellation xuyên call chain.

**Thách đố:** "Deploy gRPC sau ELB (L4), 5 pod mà 1 pod gánh 90% tải." Vì sao? → HTTP/2 long-lived connection ghim 1 backend; cần L7/mesh hoặc client-side LB.

### ⑨ Bài tập + tự chấm
- **BT:** Viết `.proto` cho OrderService (unary + server-stream), thêm field mới + `reserved` cho field xóa, client set deadline. *Đạt khi:* không tái dùng number, có `reserved`, deadline propagate, chọn đúng LB story.

### ⑩ Đọc thêm
grpc.io/docs. Protocol Buffers docs (proto3, "Updating A Message Type"). "gRPC Load Balancing" (grpc.io blog). Envoy/Istio docs.

---

# MODULE 5 — QUYẾT ĐỊNH CẤP TECH LEAD

---

## BÀI 15 — Gateway / BFF / Contract testing / Deprecation / Webhook / Governance
> Phủ: F-TL-001…006

### ① Mục tiêu & vị trí
Bài tổng kết ở tầng **quyết định kiến trúc & tổ chức** — thứ phân biệt Tech Lead với senior. Học xong bạn lý giải được **API Gateway**, **BFF**, **contract testing**, **deprecation an toàn**, **thiết kế webhook**, và **governance đa team**. Tích hợp mọi bài trước (auth/rate limit ở Bài 4/10, deprecation Bài 6, error Bài 7, idempotency Bài 8, OpenAPI Bài 11).

### ② Giảng cơ bản → nâng cao

**(a) API Gateway (F-TL-001).** Một điểm vào trước các service, gánh **cross-cutting concerns**: auth/authn, rate limiting, routing, **request aggregation**, TLS termination, logging/observability, transformation. Vì sao tách khỏi service: **DRY** (không lặp auth/rate limit ở mỗi service), tách concern khỏi business logic, đổi policy một chỗ. ⚠️ Cảnh báo: gateway dễ **phình thành "god component"** ôm cả business logic → khó maintain, single point of failure. Giữ gateway **mỏng** (cross-cutting, không business).

**(b) BFF — Backend for Frontend (F-TL-002).** Một backend **riêng cho từng loại client** (web BFF, mobile BFF). Lý do: client khác nhau cần **shape/aggregation khác nhau**; thay vì nhồi mọi nhu cầu vào một API chung (over/under-fetching), mỗi BFF **tailor** response cho client của nó. Khi nên có: nhiều loại client với nhu cầu data khác biệt rõ. Trade-off: **thêm layer + duplicate logic** giữa các BFF (so với GraphQL — một cách giải khác cho cùng vấn đề flexibility, Bài 12).

**(c) Contract testing (F-TL-003).** Vấn đề: microservices — service B đổi API, **vỡ** consumer A nhưng không ai biết tới khi e2e/production. **Contract testing (vd Pact)**: consumer định nghĩa **contract** (kỳ vọng của nó về producer); producer chạy test verify nó **thỏa contract** trước khi deploy. **Consumer-driven contract**. Chạy như **CI gate** → bắt breaking change **giữa service** mà **không cần e2e đầy đủ** (nhanh, rẻ hơn). Bổ sung cho OpenAPI breaking-detection (Bài 11).

**(d) Deprecate & sunset public endpoint (F-TL-004).** (Mở rộng Bài 6.) Quy trình an toàn: **thông báo trước** (email/changelog/dashboard) → header **`Deprecation`** + **`Sunset`** (RFC 8594, ngày tắt) → **timeline rõ ràng** (vd 6–12 tháng) → **telemetry** biết *ai còn dùng* (đừng tắt mù) → **migration guide**. **Không tắt đột ngột.** Với public API, tắt sai = mất khách + sự cố.

**(e) Webhook — outbound API (F-TL-005).** Bạn gọi *ngược* về consumer. Phải lo:
- **At-least-once + retry/backoff** (consumer có thể tạm down) → consumer **nhận trùng** → cần **event id để dedupe** (nối idempotency Bài 8).
- **HMAC signature** (ký payload bằng shared secret) để consumer **verify nguồn** (chống giả mạo).
- **Không đảm bảo ordering** → consumer phải chịu được event đến lệch thứ tự (gửi timestamp/sequence để consumer tự sắp).
- (Thêm) timeout ngắn cho mỗi delivery, dead-letter sau N lần fail, cho consumer **replay**.

**(f) Governance đa team (F-TL-006).** Nhiều team thiết kế API → mỗi team một kiểu (error format khác, pagination khác, naming khác) → **chi phí tích hợp + học** tăng vọt. Giải:
- **API style guide** (convention chung: naming, error envelope RFC 9457, pagination cursor, auth, versioning).
- **Linter tự động** (vd **Spectral**) ép style guide trong CI — không dựa vào review thủ công.
- **API review / governance board** cho thay đổi lớn.
- **Shared components** (`$ref` chung cho Error/Pagination — Bài 11).
→ Mục tiêu: API toàn org **trông như một người thiết kế**.

### ③ ⚠️ Cũ → mới / bẫy TL

| Bẫy | Đúng | Vì sao |
|---|---|---|
| Gateway ôm business logic | Gateway **mỏng** (cross-cutting only) | Tránh god-component/SPOF |
| Một API chung cho mọi client | BFF / GraphQL khi nhu cầu khác biệt rõ | Tránh over/under-fetching |
| Dựa e2e bắt breaking giữa service | **Contract testing** (Pact) CI gate | Nhanh, rẻ, sớm |
| Tắt endpoint khi "thấy ít dùng" | Deprecation policy + telemetry + Sunset | Không phá client |
| Webhook fire-and-forget | retry/backoff + HMAC + event id + dead-letter | At-least-once, verify, dedupe |
| Mỗi team tự do thiết kế | Style guide + Spectral CI + review | Nhất quán toàn org |

### ④ Áp dụng + bigtech
- **Netflix** tiên phong **BFF** (device-specific). **Kong/Apigee/AWS API Gateway/Envoy** cho gateway. **Pact** cho contract testing (PactFlow). **Stripe/GitHub** có deprecation policy + Sunset + changelog mạnh. **Zalando/Google/Microsoft** publish **API style guide** công khai + linter. *(pattern; verify chi tiết tại từng nguồn)*

### ⑤ Code thực hành

```ts
// Webhook outbound production-ready: HMAC + event id + retry/backoff + dead-letter
async function deliverWebhook(endpoint, event, attempt = 1) {
  const body = JSON.stringify(event);                 // event có id duy nhất + timestamp
  const sig = hmacSHA256(endpoint.secret, body);      // ký để consumer verify nguồn
  try {
    const r = await fetch(endpoint.url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json',
                 'X-Webhook-Id': event.id,            // consumer dedupe theo id
                 'X-Webhook-Signature': sig,
                 'X-Webhook-Timestamp': event.ts },
      body, signal: AbortSignal.timeout(5000),        // timeout mỗi delivery
    });
    if (!r.ok) throw new Error(`status ${r.status}`);
  } catch (e) {
    if (attempt < 6) {                                // retry với exponential backoff
      await queue.enqueueDelayed('webhook', { endpoint, event, attempt: attempt + 1 },
                                 2 ** attempt * 1000);
    } else {
      await deadLetter.save(endpoint, event);         // hết retry -> dead-letter để replay
    }
  }
}
```

```yaml
# .spectral.yaml — governance: ép convention toàn org (CI)
extends: ["@stoplight/spectral/rulesets/oas"]
rules:
  paths-kebab-case: { given: "$.paths[*]~", then: { function: pattern, functionOptions: { match: "^/[a-z0-9-/{}]+$" } } }
  error-uses-problem-json: { description: "Lỗi phải dùng RFC 9457", severity: error, ... }
```

> ⚠️ **AI hay sai (nhiều chỗ):** webhook **không HMAC** (consumer không verify được nguồn → giả mạo), **không event id** (consumer không dedupe được → xử lý trùng), **không retry/dead-letter** (mất event khi consumer tạm down). Người hiểu phải có đủ HMAC + id + retry + dead-letter.

### ⑥ Keywords
- 🧠 **HIỂU:** vì sao tách cross-cutting ra gateway; BFF vs API chung vs GraphQL; consumer-driven contract; governance đa team.
- 📌 **THUỘC:** việc của gateway; deprecation policy (Deprecation/Sunset/telemetry/migration); webhook (at-least-once/HMAC/event id/no-ordering); Pact; Spectral.
- 🛠️ **LÀM ĐƯỢC:** thiết kế webhook production-ready; dựng contract test CI gate; viết style guide + Spectral; deprecate an toàn.

### ⑦ Mental model
**"TL không gõ API — TL bảo vệ *tính nhất quán & khả tiến hóa* của hợp đồng qua thời gian và qua nhiều team: gateway mỏng, contract test chặn breaking, deprecation có telemetry, webhook chịu lỗi, governance bằng linter chứ không bằng niềm tin."**

### ⑧ Phỏng vấn & thách đố
1. API Gateway làm gì, vì sao tách? → cross-cutting tập trung; cảnh báo god-component.
2. BFF khi nào? → nhiều client nhu cầu khác biệt rõ.
3. Contract testing giải quyết gì? → bắt breaking giữa service không cần e2e; consumer-driven.
4. *(khó)* Deprecate public endpoint an toàn? → thông báo + Sunset + timeline + telemetry + migration.
5. *(khó)* Webhook cần lo gì? → at-least-once/retry + HMAC + event id (dedupe) + no-ordering + dead-letter.
6. *(khó)* Governance đa team? → style guide + Spectral CI + review + shared components.

**Thách đố:** "Mỗi team trả error một format, integrate cực khổ." Giải pháp TL? → style guide chung (RFC 9457) + Spectral linter trong CI ép tự động + shared `$ref` Error.

### ⑨ Bài tập + tự chấm
- **BT1:** Thiết kế hệ webhook cho "payment.succeeded". *Đạt khi:* HMAC, event id + dedupe guide, retry/backoff, dead-letter, không phụ thuộc ordering.
- **BT2:** Soạn 5 rule Spectral cho style guide org (naming, error 9457, pagination, versioning header, security). *Đạt khi:* rule chạy được trong CI và bắt được vi phạm.

### ⑩ Đọc thêm
"Backends for Frontends" (Sam Newman / microservices.io). Pact docs (pact.io) + PactFlow. RFC 8594. Zalando RESTful API Guidelines. Spectral docs. Kong/Envoy gateway docs.

---

# ✅ TỔNG KẾT MỤC F & CỔNG HOÀN THÀNH

## Bản đồ 1 trang — mental model toàn mục

| # | Bài | Câu thần chú |
|---|---|---|
| 1 | REST fundamentals | Danh từ ở URL, động từ ở method, ngữ cảnh mỗi request |
| 2 | Status & creation | Số trước, body sau; 201+Location |
| 3 | RMM & HATEOAS | L2 là điểm ngọt; HATEOAS chỉ khi workflow phức tạp |
| 4 | CORS | Luật trình duyệt nới lỏng SOP; cho *đọc* không cho *làm* |
| 5 | Batch & async | Việc lâu → 202+job; batch → báo cáo từng dòng |
| 6 | Versioning | Mặc định additive; bump là vũ khí cuối |
| 7 | Error handling | Status cho máy, code cho business, không leak |
| 8 | Idempotency | Mạng sẽ retry; claim nguyên tử; effectively-once |
| 9 | Pagination | List lớn → keyset; whitelist sort; estimate total |
| 10 | Rate limiting | Đếm tập trung & atomic; per-key; bảo vệ downstream |
| 11 | OpenAPI | Hợp đồng máy-đọc; contract-first; CI lint + breaking-gate |
| 12 | GraphQL | Client chọn shape; đổi lại cache/authz/N+1 khó |
| 13 | Async API | 1 chiều→SSE, 2 chiều→WS; reconnect không mất message |
| 14 | gRPC | Protobuf+HTTP/2 nội bộ; field number bất khả xâm phạm; L7/mesh |
| 15 | TL decisions | Bảo vệ tính nhất quán & khả tiến hóa của hợp đồng |

## Cổng hoàn thành (theo WORKFLOW 2 — Bước E)
"Học xong mục F" = **ĐẠT toàn bộ 87 câu** của `QB_F_apidesign.md` qua phỏng vấn chấm live (Bước D).
- Cách tự kiểm: với mỗi câu, đọc to dòng *"dò cái gì"* và tự trả lời thành tiếng (Feynman). Chạm hết các mốc → tạm đạt; lấn cấn chỗ nào → quay lại bài tương ứng.
- **Spaced repetition:** ôn lại theo lịch **hôm nay → mai → 3 ngày → 1 tuần**. Mỗi lần ôn trộn ngẫu nhiên 1–2 câu cũ đã đạt để chống học vẹt.

## Câu thần chú cuối
> *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*
> Cú pháp (proto field, header name, status number) tra docs 30 giây. **Quyết định** (REST hay gRPC, offset hay keyset, version hay additive, gateway mỏng hay dày) mới là việc của Tech Lead — và là thứ AI hay làm sai cần bạn kiểm tra.

---
*Tài liệu sinh theo WORKFLOW 2 (Bước B + C) từ `QB_F_apidesign.md`. Chuẩn/phiên bản đã verify 6/2026 — verify lại các dòng có dấu (verify) khi đọc về sau.*
