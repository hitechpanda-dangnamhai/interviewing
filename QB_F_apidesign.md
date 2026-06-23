# 🧪 QB_F — Ngân hàng câu hỏi: API Design & Best Practices

> **Mục F** trong roadmap Tech Lead Backend (🔴 ⏫ ~65%) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 9 mục con của roadmap + nhóm quyết định cấp TL (gateway/BFF/contract test/deprecation/webhook).
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 87 câu.**

> **Chống trùng (đã đọc `QB_E_database.md`, `QB_G_nodejs.md`, `QB_H_nestjs.md`):**
> - **Rate limiting** ở đây nhìn từ góc *thiết kế API + thuật toán + phân tán* (token bucket, sliding window, Redis, header `429`/`Retry-After`), KHÔNG lặp phần wiring thư viện `rate-limiter-flexible`/hardening ở **G-SEC**.
> - **Error handling** ở đây là *hợp đồng lỗi* (status code semantics, RFC 9457, error envelope, không leak internal), KHÔNG lặp phần cài Exception Filter của Nest ở **H-INT**.
> - **DataLoader/N+1** ở đây là góc *GraphQL resolver batching*, KHÁC N+1 phía database ở **E-NP1**.
> - **gRPC** ở đây là *quyết định protocol + thiết kế Protobuf + REST-vs-gRPC*, KHÔNG lặp phần transport gRPC trong Nest microservices ở **H-MS**.
> - **Auth (JWT/OAuth/OIDC)** thuộc mục **M (Security)** — F chỉ chạm *bề mặt API* (CORS, auth ở handshake WS/SSE) và trỏ sang M cho chiều sâu.

## Cách đọc
- **ID:** `F-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (trade-off, distributed, governance).
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm).

## Bối cảnh phiên bản & chuẩn (tính tại 6/2026 — *verify lại khi học*)
- **RFC 9457 "Problem Details for HTTP APIs"** (7/2023) **obsoletes RFC 7807** — media type `application/problem+json`. → nguồn: rfc-editor.org/rfc/rfc9457.
- **RFC 9110** định nghĩa method semantics: GET/HEAD/PUT/DELETE/OPTIONS **idempotent**, POST/PATCH **không**.
- **OpenAPI 3.2.0** (9/2025) = bản **stable hiện tại** (thêm mô tả native cho SSE/streaming). **OpenAPI 3.1** vẫn dùng rộng rãi. **OpenAPI 4.0 "Moonwalk"** đang phát triển, **chưa có ngày release** → không nói "4.0 hiện hành". → nguồn: openapis.org, spec.openapis.org/oas/v3.2.0.html.
- **`Idempotency-Key`** vẫn là **IETF Internet-Draft** (draft-ietf-httpapi-idempotency-key-header-07, 10/2025) — **chưa thành RFC**; Stripe/Adyen/Dwolla/WorldPay đã dùng theo de-facto. → nguồn: datatracker.ietf.org.
- **`RateLimit` / `RateLimit-Policy` header** cũng là IETF draft (httpapi WG) → khi nhắc tên header cụ thể, ghi *(draft)*.
- ⚠️ Mọi câu chạm tên chuẩn/đời spec/tooling đổi nhanh đều có dấu *(verify)*.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **F-REST** | RESTful principles, resource modeling, method/status, Richardson, HATEOAS, CORS | 12 |
| **F-VER** | API versioning | 7 |
| **F-ERR** | Error handling & response standardization (RFC 9457) | 8 |
| **F-IDEM** | Idempotency | 8 |
| **F-PAG** | Pagination / filtering / sorting | 8 |
| **F-RL** | Rate limiting (thuật toán + phân tán) | 8 |
| **F-OAS** | OpenAPI/Swagger, contract-first, codegen | 7 |
| **F-GQL** | GraphQL vs REST, DataLoader, schema | 9 |
| **F-WS** | Async API: WebSocket / SSE / realtime | 7 |
| **F-GRPC** | gRPC + Protobuf (nội bộ) | 7 |
| **F-TL** | Quyết định API cấp Tech Lead (gateway/BFF/contract test/deprecation/webhook) | 6 |

---

## F-REST — RESTful principles & resource modeling

**F-REST-001** ⭐⭐
"REST 'đúng nghĩa' (theo 6 ràng buộc của Fielding) khác gì với 'REST = JSON over HTTP'? Nhiều API tự nhận RESTful — sai chỗ nào?"
> *Dò cái gì:* nêu được stateless, uniform interface, resource-based, client-server, cacheable; phân biệt RESTful thật vs CRUD-over-HTTP/RPC bị gọi nhầm là REST.

**F-REST-002** ⭐⭐
"Đặt tên URL/resource theo nguyên tắc nào? Thiết kế endpoint cho 'đơn hàng của một user'."
> *Dò cái gì:* danh từ số nhiều, không động từ trong path (`/users/{id}/orders` không `/getUserOrders`); biết khi nào nest, khi nào flatten resource.

**F-REST-003** ⭐⭐⭐
"Phân loại GET/POST/PUT/PATCH/DELETE theo 2 trục **safe** và **idempotent**."
> *Dò cái gì:* GET safe+idempotent; PUT/DELETE idempotent (không safe); POST không cả hai; PATCH thường không idempotent; hiểu định nghĩa safe (không đổi state) vs idempotent.

**F-REST-004** ⭐⭐⭐
"PUT vs PATCH khác nhau thế nào? PATCH gửi gì — JSON Merge Patch hay JSON Patch?"
> *Dò cái gì:* PUT thay toàn bộ resource (idempotent), PATCH sửa một phần; biết JSON Merge Patch (RFC 7386) vs JSON Patch (RFC 6902) là 2 định dạng khác nhau.

**F-REST-005** ⭐⭐⭐
"Chọn status code đúng cho các tình huống: tạo thành công, async nhận xử lý sau, không có body trả về, lỗi validation, chưa đăng nhập vs không đủ quyền, conflict."
> *Dò cái gì:* ánh xạ đúng 201 / 202 / 204 / 400-422 / 401-403 / 409; hiểu 4xx (lỗi client) vs 5xx (lỗi server).

**F-REST-006** ⭐⭐⭐
"POST tạo resource thành công thì trả gì cho đúng chuẩn?"
> *Dò cái gì:* 201 Created + header `Location` trỏ tới resource mới (+ thường kèm body), không trả 200 chung chung.

**F-REST-007** ⭐⭐⭐⭐
"Richardson Maturity Model có 4 level (0→3) — mỗi level là gì, và phần lớn hệ thống production thực tế dừng ở đâu?"
> *Dò cái gì:* L0 RPC-over-HTTP một endpoint; L1 resources; L2 dùng đúng HTTP verb + status code; L3 HATEOAS/hypermedia; nhận ra đa số dừng ở L2.

**F-REST-008** ⭐⭐⭐⭐
"HATEOAS là gì, lợi gì, và vì sao thực tế rất ít API làm L3 đầy đủ?"
> *Dò cái gì:* hypermedia controls (link trong response) cho client tự khám phá/giảm coupling; trade-off phức tạp + client hiếm khi tận dụng → over-engineering với phần lớn ca.

**F-REST-009** ⭐⭐⭐
"Content negotiation hoạt động thế nào? `Accept` vs `Content-Type`, và khi nào trả 406 vs 415?"
> *Dò cái gì:* client dùng `Accept` chọn representation, server đọc `Content-Type` của body; 415 Unsupported Media Type (body sai), 406 Not Acceptable (không đáp ứng được `Accept`).

**F-REST-010** ⭐⭐⭐⭐
"CORS thực chất bảo vệ ai? Preflight (`OPTIONS`) kích hoạt khi nào và gồm header gì? Vì sao 'lỗi CORS' không phải lỗi server?"
> *Dò cái gì:* CORS là cơ chế *browser-side*; simple vs preflighted request; `Access-Control-Allow-Origin/Methods/Headers`; CORS không thay thế authz, khác CSRF.

**F-REST-011** ⭐⭐⭐⭐
"Thiết kế batch/bulk operation trong REST (tạo nhiều resource một request, một số fail) thế nào?"
> *Dò cái gì:* REST thuần khó biểu diễn batch; dùng endpoint `/batch` hoặc `207 Multi-Status`; xử lý **partial success** tường minh thay vì fail-all.

**F-REST-012** ⭐⭐⭐⭐⭐
"Thiết kế API cho thao tác chạy lâu (vd export báo cáo 10 phút). Không thể giữ connection chờ đồng bộ — làm sao?"
> *Dò cái gì:* `202 Accepted` + trả về một **operation/job resource** để client poll status (hoặc webhook/SSE báo xong); tránh blocking request đồng bộ, tránh timeout.

---

## F-VER — API versioning

**F-VER-001** ⭐⭐
"Có những cách version API nào (URI path, query, header, media type)? Ưu nhược mỗi cách?"
> *Dò cái gì:* nêu ≥3 cách; URI versioning phổ biến nhất nhưng 'không RESTful thuần'; header/media-type sạch URL nhưng khó test/cache/khó nhìn.

**F-VER-002** ⭐⭐⭐
"Thay đổi nào là **breaking** vs **non-breaking** với client?"
> *Dò cái gì:* thêm field optional / thêm endpoint = non-breaking; xóa/đổi tên field, đổi kiểu, thêm required param, đổi status code = breaking.

**F-VER-003** ⭐⭐⭐
"Khi nào thực sự phải bump major version, khi nào chỉ cần thêm field?"
> *Dò cái gì:* chỉ bump khi có breaking change; ưu tiên evolution additive để tránh maintain nhiều version song song.

**F-VER-004** ⭐⭐⭐⭐
"Vận hành nhiều version song song tốn kém ra sao? Một deprecation policy đầy đủ gồm gì?"
> *Dò cái gì:* chi phí maintain N version (code + test + bug); cần sunset timeline, header `Deprecation`/`Sunset`, thông báo, và telemetry để biết ai còn dùng.

**F-VER-005** ⭐⭐⭐
"Có người nói HATEOAS/hypermedia giảm nhu cầu version — đúng đến đâu?"
> *Dò cái gì:* hypermedia giảm coupling về URL/navigation nhưng KHÔNG loại bỏ breaking change ở schema/semantics; lập luận hợp lý nhưng không phải 'thuốc tiên'.

**F-VER-006** ⭐⭐⭐⭐
"Stripe version theo ngày và pin per-account, server tự transform request/response cũ. Mô hình này giải quyết gì?"
> *Dò cái gì:* nhận ra pattern 'rolling version + transformation layer' → client cũ không phải đổi code, server map giữa các version; đánh đổi: layer transform phức tạp.

**F-VER-007** ⭐⭐⭐⭐
"GraphQL 'không cần version' — đúng hay sai? Evolve schema thế nào?"
> *Dò cái gì:* GraphQL evolve bằng thêm field + `@deprecated` thay vì version cả API; nhưng breaking vẫn xảy ra (xóa field/đổi type), 'versionless' là về *cách evolve*, không phải miễn nhiễm.

---

## F-ERR — Error handling & response standardization

**F-ERR-001** ⭐⭐
"Vì sao phải chuẩn hóa error response toàn app? Một error body tối thiểu nên có gì?"
> *Dò cái gì:* để client parse nhất quán; tối thiểu cần code máy-đọc + message người-đọc + (option) details/field errors + trace id.

**F-ERR-002** ⭐⭐⭐
"RFC 9457 (Problem Details) là gì, gồm field gì, media type nào? Nó thay chuẩn nào?"
> *Dò cái gì:* `type`/`title`/`status`/`detail`/`instance` + extension; media type `application/problem+json`; biết 9457 **obsoletes 7807**. *(verify)*

**F-ERR-003** ⭐⭐⭐
"Business/application error code khác HTTP status thế nào? Vì sao cần cả hai?"
> *Dò cái gì:* HTTP status = lớp transport, coarse-grained; business code (vd `INSUFFICIENT_FUNDS`) = chi tiết ổn định để client switch logic mà không phụ thuộc message.

**F-ERR-004** ⭐⭐⭐
"Validation error nên trả 400 hay 422? Cấu trúc field-level errors thế nào?"
> *Dò cái gì:* phân biệt malformed/parse-fail (400) vs hợp lệ cú pháp nhưng sai nghiệp vụ (422 — có tranh luận); trả mảng lỗi theo từng field/path để client highlight.

**F-ERR-005** ⭐⭐⭐⭐
"Không được leak gì ra error response? Nhưng vẫn phải debug được production — cân bằng sao?"
> *Dò cái gì:* không lộ stack trace/SQL/internal path/secret; gắn correlation/trace id để map sang log nội bộ; 5xx trả message generic, chi tiết để ở log.

**F-ERR-006** ⭐⭐⭐
"Nên xử lý error tập trung ở đâu thay vì try-catch rải rác khắp nơi?"
> *Dò cái gì:* global error handler/filter map exception → response chuẩn, tránh lặp; (ranh giới: đây là *hợp đồng lỗi*, không phải chi tiết wiring Exception Filter của Nest ở H-INT).

**F-ERR-007** ⭐⭐⭐⭐
"Phân loại lỗi: expected (business) vs unexpected (bug) vs transient — log và xử lý mỗi loại khác nhau ra sao?"
> *Dò cái gì:* business → 4xx, không alert, log info/warn; bug → 5xx + alert; transient → retry/circuit breaker; log level và hành vi khác nhau.

**F-ERR-008** ⭐⭐⭐⭐
"Trả lỗi cho thao tác async/batch (một số item fail) và cho webhook thế nào?"
> *Dò cái gì:* partial success (207 / mảng kết quả per-item), không fail-all; webhook cần retry + consumer dedupe; phân biệt lỗi cả-batch vs lỗi một-item.

---

## F-IDEM — Idempotency

**F-IDEM-001** ⭐⭐
"Idempotency trong HTTP nghĩa là gì? Method nào idempotent theo spec?"
> *Dò cái gì:* lặp cùng request → cùng hiệu ứng lên server; GET/PUT/DELETE/HEAD/OPTIONS idempotent, POST/PATCH không (RFC 9110); phân biệt idempotent vs safe.

**F-IDEM-002** ⭐⭐⭐
"Vì sao POST tạo payment phải idempotent? Chuyện gì xảy ra khi client retry do timeout mạng?"
> *Dò cái gì:* network retry/double-submit → double charge/duplicate record; cần cơ chế chống lặp dù request 'có vẻ' là mới.

**F-IDEM-003** ⭐⭐⭐
"`Idempotency-Key` header hoạt động ra sao?"
> *Dò cái gì:* client sinh key duy nhất/operation; server lưu key→response, lần sau **replay** response cũ thay vì làm lại; biết đây là IETF draft, Stripe/Adyen dùng de-facto. *(verify)*

**F-IDEM-004** ⭐⭐⭐⭐
"Hai request cùng `Idempotency-Key` đến **song song** thì sao? Lưu record thế nào để atomic, tránh race?"
> *Dò cái gì:* dùng unique constraint / `INSERT ... ON CONFLICT` để **claim** key atomic; request thứ 2 nhận 409 'đang xử lý' hoặc chờ; tránh check-then-act race.

**F-IDEM-005** ⭐⭐⭐⭐
"Cùng key nhưng **body khác nhau** thì xử lý sao? Fingerprint dùng để làm gì?"
> *Dò cái gì:* lưu fingerprint/checksum của payload; key tái dùng với payload khác → 422 (key reuse mismatch) để tránh trộn hai thao tác khác nhau.

**F-IDEM-006** ⭐⭐⭐
"Idempotency key sống bao lâu (TTL) và scope theo ai?"
> *Dò cái gì:* có TTL hữu hạn (vd 24h), scope per-endpoint/per-account/per-key; không giữ vĩnh viễn (tốn storage, key có thể trùng nghĩa khác).

**F-IDEM-007** ⭐⭐⭐⭐
"PUT idempotent 'miễn phí' còn POST phải tự làm — vì sao? Có nên cứ đổi POST thành PUT để khỏi xử lý?"
> *Dò cái gì:* PUT idempotent vì client cung cấp toàn bộ state tại id xác định; POST sinh id server-side nên lặp → resource mới; chỉ đổi được khi client biết/sinh được id.

**F-IDEM-008** ⭐⭐⭐⭐⭐
"'Exactly-once delivery' trong distributed system có tồn tại không? Liên hệ với idempotency thế nào?"
> *Dò cái gì:* exactly-once *delivery* là ảo tưởng; thực tế 'at-least-once + idempotent consumer' = **effectively-once**; nối với dedupe/outbox/transactional messaging.

---

## F-PAG — Pagination / filtering / sorting

**F-PAG-001** ⭐⭐
"Offset/limit pagination hoạt động thế nào và hạn chế gì?"
> *Dò cái gì:* `OFFSET n LIMIT m`; chậm khi offset lớn (DB scan & bỏ); **data shift** (trùng/sót) khi có insert/delete giữa các trang.

**F-PAG-002** ⭐⭐⭐
"Cursor/keyset pagination là gì, vì sao tốt hơn offset cho dataset lớn / real-time?"
> *Dò cái gì:* `WHERE (sort_key, id) > cursor LIMIT m`; tận dụng index, không scan bỏ, ổn định khi data thay đổi.

**F-PAG-003** ⭐⭐⭐⭐
"Cursor pagination có nhược điểm gì?"
> *Dò cái gì:* không jump tới 'trang N' bất kỳ; cần sort key ổn định + tie-breaker (id) khi trùng; khó tính total count.

**F-PAG-004** ⭐⭐⭐
"Trả metadata pagination (next/prev/cursor/total) ở đâu cho hợp lý?"
> *Dò cái gì:* envelope `{ data, meta }` hoặc `Link` header (RFC 8288); cân nhắc bỏ `total` vì đắt; cursor opaque, encode điểm dừng.

**F-PAG-005** ⭐⭐⭐
"Thiết kế filtering qua query param (nhiều field, nhiều operator) — rủi ro gì?"
> *Dò cái gì:* `?status=active&created_after=...`; **whitelist** field được filter, không cho client filter tùy ý lên column nhạy cảm; tránh injection.

**F-PAG-006** ⭐⭐⭐⭐
"Sorting động kiểu `?sort=-created_at,name` — implement an toàn ra sao?"
> *Dò cái gì:* parse + whitelist cột sort, ánh xạ sang index; không nối thẳng vào SQL; sort phải **deterministic** (luôn kèm id làm tie-breaker).

**F-PAG-007** ⭐⭐⭐⭐
"Total count quá đắt trên bảng lớn — xử lý thế nào?"
> *Dò cái gì:* count đắt (gần full scan); dùng estimate (vd `reltuples` của Postgres), trả `has_more` thay vì total chính xác, hoặc count tách/cache.

**F-PAG-008** ⭐⭐⭐⭐⭐
"Pagination ổn định khi list đang thay đổi liên tục (feed) — đảm bảo không trùng/sót thế nào?"
> *Dò cái gì:* keyset theo sort key gần-immutable (id giảm dần / created_at); offset gây trùng-sót khi data dịch; cursor encode điểm dừng chính xác.

---

## F-RL — Rate limiting

**F-RL-001** ⭐⭐
"Vì sao cần rate limiting? Phân biệt rate limiting vs quota vs throttling."
> *Dò cái gì:* chống abuse/DoS/cost; rate limit = req/đơn vị thời gian, quota = tổng theo billing period, throttling = làm chậm thay vì chặn hẳn.

**F-RL-002** ⭐⭐⭐
"So sánh token bucket, leaky bucket, fixed window, sliding window."
> *Dò cái gì:* token bucket cho phép **burst**; leaky bucket làm mượt output; fixed window đơn giản nhưng có burst ở ranh giới; sliding window (log/counter) mượt hơn.

**F-RL-003** ⭐⭐⭐
"Vấn đề 'burst ở ranh giới' của fixed window là gì? Sliding window khắc phục sao?"
> *Dò cái gì:* gần mốc reset có thể cho ~2x limit (cuối window này + đầu window sau); sliding window log hoặc weighted counter trải đều theo thời gian thật.

**F-RL-004** ⭐⭐⭐⭐
"Rate limit trong hệ nhiều instance — vì sao đếm in-memory không đủ? Giải pháp?"
> *Dò cái gì:* mỗi instance đếm riêng → tổng vượt limit thật; cần store tập trung (Redis `INCR`+TTL / Lua atomic) hoặc đẩy lên gateway/edge.

**F-RL-005** ⭐⭐⭐
"Limit theo gì — per-IP, per-user/API-key, per-endpoint? Cạm bẫy của per-IP?"
> *Dò cái gì:* per-IP dễ false-positive (NAT/proxy/shared IP, corporate) và dễ bypass; per-API-key/user công bằng hơn; thường tiered + kết hợp.

**F-RL-006** ⭐⭐⭐
"Khi client vượt limit thì trả gì? Những header nào nên đính kèm?"
> *Dò cái gì:* `429 Too Many Requests` + `Retry-After`; (draft) `RateLimit`/`RateLimit-Policy` để client biết hạn mức và khi nào thử lại. *(verify header name)*

**F-RL-007** ⭐⭐⭐⭐
"Đặt rate limiter ở đâu trong kiến trúc — gateway/edge hay app? Trade-off?"
> *Dò cái gì:* edge/gateway (Nginx/Kong/CDN) chặn sớm, tiết kiệm tài nguyên; app-level granular theo business/tenant; thực tế thường kết hợp cả hai.

**F-RL-008** ⭐⭐⭐⭐⭐
"Thiết kế rate limit công bằng để tránh 'noisy neighbor' và bảo vệ downstream (DB)? Kết hợp với cơ chế gì?"
> *Dò cái gì:* kết hợp concurrency limit/bulkhead, load shedding, backpressure; limit per-tenant; tránh một tenant làm cạn tài nguyên dùng chung và sập DB.

---

## F-OAS — OpenAPI / Swagger & codegen

**F-OAS-001** ⭐⭐
"OpenAPI và Swagger khác nhau thế nào? Bản spec hiện hành là gì?"
> *Dò cái gì:* OpenAPI = đặc tả (trước đây tên Swagger), Swagger = bộ tool (UI/Editor/Codegen); biết 3.2.0 là bản stable hiện tại, 4.0 'Moonwalk' chưa ra. *(verify)*

**F-OAS-002** ⭐⭐⭐
"Design-first (contract-first) vs code-first — ưu nhược, khi nào chọn cái nào?"
> *Dò cái gì:* contract-first đồng thuận spec sớm + generate cả client/server + cho phép song song; code-first nhanh cho team nhỏ nhưng spec dễ lệch thực tế.

**F-OAS-003** ⭐⭐⭐
"Codegen từ OpenAPI sinh ra gì? Rủi ro của codegen?"
> *Dò cái gì:* client SDK / server stub / type models đồng bộ với spec; rủi ro: code sinh ra xấu/khó tùy biến, và drift nếu quên regenerate.

**F-OAS-004** ⭐⭐⭐⭐
"Làm sao giữ spec và implementation không bị lệch (drift)?"
> *Dò cái gì:* spec là source-of-truth + contract/runtime validation, hoặc generate spec từ code rồi lint; gắn CI gate so sánh spec với hành vi thật.

**F-OAS-005** ⭐⭐⭐
"`components/schemas` + `$ref` để tái dùng — vì sao quan trọng?"
> *Dò cái gì:* DRY cho model/error/pagination dùng chung; tránh copy-paste; dễ maintain và đảm bảo nhất quán.

**F-OAS-006** ⭐⭐⭐⭐
"OpenAPI mô tả tốt cái gì và KHÔNG mô tả tốt cái gì?"
> *Dò cái gì:* tốt cho REST sync (path/param/schema/status); event-driven dùng AsyncAPI; business invariant/stateful workflow/ordering spec không bắt được.

**F-OAS-007** ⭐⭐⭐⭐
"Dùng OpenAPI trong CI: linting và breaking-change detection làm gì?"
> *Dò cái gì:* lint style guide (vd Spectral), so spec cũ↔mới để chặn breaking trước merge, kết hợp contract test (Pact) phía consumer.

---

## F-GQL — GraphQL vs REST

**F-GQL-001** ⭐⭐
"GraphQL giải quyết vấn đề gì của REST?"
> *Dò cái gì:* over-fetching / under-fetching; client chọn đúng field cần; một request lấy nhiều resource liên quan thay vì nhiều round-trip.

**F-GQL-002** ⭐⭐⭐
"Khi nào nên chọn GraphQL, khi nào REST tốt hơn?"
> *Dò cái gì:* GraphQL hợp client đa dạng/UI đổi nhanh/aggregate nhiều nguồn; REST hợp public API đơn giản, cache HTTP, file/upload, ops đơn giản.

**F-GQL-003** ⭐⭐⭐
"Vì sao cache với GraphQL khó hơn REST?"
> *Dò cái gì:* REST cache theo URL+method (HTTP/CDN); GraphQL thường POST một endpoint → mất HTTP cache, phải cache normalized ở client hoặc dùng persisted query + GET.

**F-GQL-004** ⭐⭐⭐⭐
"N+1 trong GraphQL resolver phát sinh thế nào? DataLoader giải quyết ra sao?"
> *Dò cái gì:* mỗi field resolve gọi DB riêng → N+1; DataLoader **batch + cache trong một request**; phân biệt với N+1 phía database (E-NP1) — đây là tầng resolver.

**F-GQL-005** ⭐⭐⭐⭐
"Query lồng sâu/độc hại có thể gây DoS — phòng thế nào?"
> *Dò cái gì:* depth limit, complexity/cost analysis, persisted/allow-listed queries, timeout, rate limit theo cost thay vì theo số request.

**F-GQL-006** ⭐⭐⭐
"GraphQL thường trả 200 + mảng `errors` — vì sao gây tranh cãi?"
> *Dò cái gì:* khó tận dụng HTTP status (200 cho cả lỗi); có thể partial data + errors cùng response; client/middleware/monitoring phải đọc `errors` field.

**F-GQL-007** ⭐⭐⭐
"Versioning/evolution trong GraphQL làm thế nào?"
> *Dò cái gì:* thêm field + `@deprecated` thay vì remove; mở rộng additive; 'versionless' là qua schema evolution, nhưng remove field vẫn là breaking.

**F-GQL-008** ⭐⭐⭐⭐
"Authorization trong GraphQL khó hơn REST ở chỗ nào?"
> *Dò cái gì:* không gắn per-endpoint; cần field/type-level authz trong resolver; dễ lộ data qua nested traversal nếu chỉ check ở root.

**F-GQL-009** ⭐⭐⭐⭐⭐
"Khi nhiều service muốn ghép thành một GraphQL endpoint — pattern gì, đánh đổi gì?"
> *Dò cái gì:* Apollo Federation / subgraph (hoặc schema stitching); gateway compose schema; trade-off latency fan-out, coupling, độ phức tạp vận hành.

---

## F-WS — Async API (WebSocket / SSE / realtime)

**F-WS-001** ⭐⭐
"Khi nào cần realtime/push thay vì client poll?"
> *Dò cái gì:* cần cập nhật tức thời (chat, notification, giá live), giảm tải poll; cân nhắc poll vẫn đơn giản/đủ cho nhiều ca → không over-engineer.

**F-WS-002** ⭐⭐⭐
"WebSocket vs SSE vs long-polling — khác nhau và chọn khi nào?"
> *Dò cái gì:* WS full-duplex (chat/game/collab); SSE server→client một chiều text (notification/feed, auto-reconnect); long-poll fallback; SSE đơn giản hơn, chạy trên HTTP.

**F-WS-003** ⭐⭐⭐
"SSE hoạt động ra sao và ưu điểm so với WS cho luồng một chiều?"
> *Dò cái gì:* `text/event-stream` giữ HTTP connection; auto-reconnect + `Last-Event-ID`; nhẹ, qua proxy/HTTP/2 dễ; OpenAPI 3.2 đã mô tả được SSE. *(verify)*

**F-WS-004** ⭐⭐⭐⭐
"Scale WebSocket trên nhiều instance gặp vấn đề gì?"
> *Dò cái gì:* connection có state ở 1 node → cần sticky session hoặc pub/sub (Redis) để broadcast cross-node; presence/horizontal scaling phức tạp hơn HTTP stateless.

**F-WS-005** ⭐⭐⭐
"Auth cho WebSocket/SSE thế nào khi không gửi header tiện như REST mỗi request?"
> *Dò cái gì:* gắn token ở handshake (query/subprotocol/cookie), verify khi connect; xử lý token hết hạn (re-auth/đóng); WS không gửi auth header per-message.

**F-WS-006** ⭐⭐⭐⭐
"Backpressure / slow consumer trong streaming xử lý ra sao?"
> *Dò cái gì:* client chậm → buffer server phình to (memory); cần drop/coalesce/limit queue hoặc đóng connection chậm; nối với stream backpressure phía Node (G-STR).

**F-WS-007** ⭐⭐⭐⭐⭐
"Đảm bảo không mất message khi client reconnect (mạng rớt) thế nào?"
> *Dò cái gì:* server đánh id sự kiện; client gửi `Last-Event-ID` (SSE) hoặc resume token để replay; hoặc message broker + cursor; at-least-once + dedupe phía client.

---

## F-GRPC — gRPC + Protobuf (nội bộ)

**F-GRPC-001** ⭐⭐
"gRPC là gì, dựa trên những công nghệ nào?"
> *Dò cái gì:* RPC framework; contract bằng Protobuf; transport HTTP/2; payload binary; codegen client/server đa ngôn ngữ.

**F-GRPC-002** ⭐⭐⭐
"REST vs gRPC: khi nào dùng gRPC nội bộ? Vì sao ít expose ra public/browser?"
> *Dò cái gì:* gRPC nhanh/typed/streaming cho service-to-service; browser không gọi gRPC trực tiếp (cần grpc-web + proxy); REST/JSON tốt cho public/debug/độ phổ biến.

**F-GRPC-003** ⭐⭐⭐
"Protobuf vs JSON — lợi và hại?"
> *Dò cái gì:* binary nhỏ + nhanh + schema chặt + backward-compat qua field number; bất lợi: không human-readable, cần codegen/tooling để đọc-debug.

**F-GRPC-004** ⭐⭐⭐⭐
"Evolve Protobuf schema mà không phá vỡ client cũ — quy tắc gì với field number?"
> *Dò cái gì:* KHÔNG tái dùng/đổi field number, không đổi type; thêm field mới (optional); `reserved` cho field đã xóa; đảm bảo backward/forward compatibility.

**F-GRPC-005** ⭐⭐⭐
"4 kiểu gRPC (unary, server-streaming, client-streaming, bidirectional) — mỗi loại hợp ca nào?"
> *Dò cái gì:* unary = request/response; server-stream = feed/kết quả lớn; client-stream = upload từng phần; bidi = chat/realtime hai chiều.

**F-GRPC-006** ⭐⭐⭐⭐
"Error model của gRPC khác HTTP thế nào? Deadline/cancellation hoạt động ra sao?"
> *Dò cái gì:* gRPC status code riêng (vd `UNAVAILABLE`, `DEADLINE_EXCEEDED`) khác HTTP status; deadline được propagate xuyên call + cancellation; metadata thay cho header.

**F-GRPC-007** ⭐⭐⭐⭐⭐
"gRPC sau load balancer — vì sao LB L4 thông thường không balance tốt? Cần gì?"
> *Dò cái gì:* HTTP/2 multiplex trên connection long-lived → L4 LB ghim hết vào 1 backend; cần L7/client-side LB hoặc service mesh (Envoy/Istio) để balance per-request.

---

## F-TL — Quyết định API cấp Tech Lead

**F-TL-001** ⭐⭐⭐
"API Gateway làm những việc gì? Vì sao tách các việc đó khỏi service?"
> *Dò cái gì:* tập trung cross-cutting (auth, rate limit, routing, aggregation, TLS) ở edge; tách concern khỏi business; cảnh báo gateway phình thành 'god component'.

**F-TL-002** ⭐⭐⭐⭐
"BFF (Backend-for-Frontend) là gì, khi nào nên có?"
> *Dò cái gì:* một backend riêng cho từng loại client (web/mobile) để aggregate + tailor response; tránh nhồi mọi nhu cầu vào một API chung; trade-off: thêm layer/duplicate.

**F-TL-003** ⭐⭐⭐⭐
"Contract testing (vd Pact) giải quyết vấn đề gì giữa consumer và producer?"
> *Dò cái gì:* bắt breaking change giữa service mà không cần e2e đầy đủ; consumer-driven contract; chạy như CI gate trước khi deploy producer.

**F-TL-004** ⭐⭐⭐⭐
"Deprecate và sunset một public endpoint thế nào cho an toàn?"
> *Dò cái gì:* thông báo trước + header `Deprecation`/`Sunset` + timeline rõ + telemetry biết ai còn dùng + migration guide; không tắt đột ngột.

**F-TL-005** ⭐⭐⭐⭐⭐
"Đảm bảo nhiều team thiết kế API nhất quán — governance kiểu gì?"
> *Dò cái gì:* API style guide + linter (Spectral) tự động + review/governance + convention chung cho error/pagination/auth; tránh mỗi team một kiểu gây chi phí tích hợp.

**F-TL-006** ⭐⭐⭐⭐
"Thiết kế webhook (outbound API) cần lo những gì?"
> *Dò cái gì:* at-least-once + retry/backoff; HMAC signature để consumer verify nguồn; gửi event id để consumer dedupe; không đảm bảo ordering → consumer phải chịu được.

---

## ✅ Sau Bước A

- Đây là **bộ đề mục F (87 câu)** — chỉ câu hỏi + tiêu chí "dò cái gì". **Hãy LƯU file này vào project** để các phiên sau dò trùng và để chạy Bước D (phỏng vấn chấm live).
- **Bước tiếp theo (B):** dựng giáo trình ngược từ bộ đề (gom câu theo cụm khái niệm → bài học, sắp cơ bản → nâng cao).
- Nhắc câu thần chú: *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*
