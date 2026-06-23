# ICP — Product Feature Catalog & Roadmap (Master Reference)

> **Mục đích:** tài liệu duy nhất để **founder · kỹ-sư · nhà đầu tư** nhìn toàn bộ hệ thống ICP — mọi tính năng (đã có / đang làm / sẽ làm / cần làm cho thị trường VN + Amazon-parity 100%), kèm **trạng thái + % hoàn thành** và **lộ trình**.

---

## 🔑 Legend (quy ước trạng thái)

| Ký hiệu | Nghĩa | % điển hình |
|---|---|---|
| 🟢 **DONE** | Đã chạy production-grade, có test + cite được ADR/route/migration | 85–100% |
| 🟡 **IN-PROGRESS / PARTIAL** | Đã có khung/chạy một phần, còn thiếu mảng | 30–80% |
| 🔵 **PLANNED** | Có ADR/roadmap, chưa code | 0–15% |
| 🟠 **NEEDED (VN/parity gap)** | Bắt buộc cho VN hoặc Amazon-parity, **chưa nằm trong plan** | 0% |
| 💡 **VISION** | Tầm nhìn dài hạn (có precedent + math), chưa cam kết lịch | 0–5% |


---

## 0. Tóm tắt cho nhà đầu tư (Executive Summary)

**ICP (Intelligent Commerce Platform)** = nền **thương mại đa-tenant SaaS "Amazon cho tiệm nhỏ Việt Nam"**: chủ shop / nhà phân phối / người mua cùng vận hành trên một nền — đăng bán, tìm kiếm bằng **giọng nói · ảnh · text (AI)**, thanh toán nội địa (Momo/ZaloPay/VNPay/COD), và (lộ trình) **dự báo nhập hàng + cho vay theo doanh số**.


**Mức hoàn thành tổng thể** (so với tầm nhìn đầy đủ Amazon-parity + VN + AI + B2B):

```
Lõi thương mại merchant↔buyer  █████░░░░░░░░░░░░░░░  ~27%   (đã de-risk)
Khám phá/discovery (AI search) ████░░░░░░░░░░░░░░░░  ~20%
Trí tuệ (forecast/reco/lending)█░░░░░░░░░░░░░░░░░░░  ~3%    (data sẵn, model chưa)
B2B nhà phân phối               █░░░░░░░░░░░░░░░░░░░  ~3%
VN-thiết-yếu (e-invoice/ship/Zalo/mobile/VietQR) ░░░░░░░░  ~2%
Tuân thủ & nền tảng             ██████░░░░░░░░░░░░░░  ~28%
─────────────────────────────────────────────────────────
TỔNG (vision đầy đủ)            ███░░░░░░░░░░░░░░░░░  ~13–15%
```

> **Thông điệp:** lõi đã vững và thật; phần còn lại là **lộ trình thực-thi rõ ràng**, không phải nghiên cứu rủi ro.

---

## 1. Nền tảng & Multi-tenant (Foundation)  — 🟢 ~28%

| Tính năng | Trạng thái | % | Mô tả ngắn |
|---|---|---|---|
| Multi-tenant isolation (RLS + GUC + FORCE) | 🟢 DONE | 32% | 46 cột `tenant_id`, RLS NULLIF-hardened fail-closed, e2e cross-tenant đỏ-khi-sai |
| RLS 3-pool runtime (icp_app NOBYPASSRLS) | 🟢 DONE | 32% | Tách pool app/migrate/privileged — chống bypass |
| Auth (JWT + session + refresh + switch-tenant) | 🟢 DONE | 28% | 10 route auth; thiếu MFA/SSO/lockout (xem §3 auth-hardening) |
| RBAC actor taxonomy (buyer/merchant/supplier/staff/admin) | 🟢 DONE | 27% | `tenants.type` + `tenant_memberships` + role-guard; thiếu role auditor/support |
| Idempotency (Redis SETNX + response cache) | 🟢 DONE | 32% | Chống double-charge/replay; 2-lớp payment_callbacks + CDC inbox |
| Event backbone (CDC Debezium + Redpanda + outbox) | 🟢 DONE | 28% | Outbox same-txn, CDC slot-aware, exactly-once inbox |
| Observability (OTel logs/traces/metrics) | 🟢 DONE | 28% | 170+ log type, trace xuyên 3-service; thiếu vài histogram (Vespa/Kafka) |
| Contract-first FE↔BE (OpenAPI codegen + Zod SOT) | 🟢 DONE | 32% | CI drift-gate, generated client |
| CI/CD hard-gate (lint/test/typecheck/coverage/drift) | 🟢 DONE | 30% | 8 job, coverage floor ≥70%, facts-drift guard |
| HA/scale (pgbouncer, Vespa cluster, multi-server) | 🔵 PLANNED | 3% | Single-node hiện tại; pooler + Vespa HA = scale-phase |

---

## 2. Người mua — Buyer Experience (Shopee/Amazon buyer parity)  — 🟡 ~20%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Storefront + PDP (trang chi tiết SP, buyer-scoped) | 🟢 DONE | 30% | Storefront read API field-projected + variant selector |
| Giỏ hàng (add/qty/remove/promo/variant) | 🟢 DONE | 32% | Composite line-key variant, tax, honor-lower pricing |
| Lưu để sau (save-for-later) | 🟢 DONE | 30% | Bảng riêng RLS, move-to-cart |
| Checkout + sổ địa chỉ + phí ship | 🟢 DONE | 30% | Address-book, shipping rate, COD/bank |
| Theo dõi đơn realtime (SSE) | 🟢 DONE | 28% | Order-status stream |
| Tìm bằng text (AI hybrid Vespa) | 🟡 PARTIAL | 20% | BM25+CLIP chạy; **thiếu facets/sort/pagination/filter** |
| Tìm/mua bằng giọng nói | 🟡 PARTIAL | 18% | STT+parse chạy; **thiếu review-state, OOS-substitute, TTS-confirm** |
| Gợi ý bằng ảnh (visual reco) | 🟡 PARTIAL | 18% | Vision + 3-signal blend; **empty-state dead-end, thiếu grid** |
| Duyệt theo danh mục / browse | 🟡 PARTIAL | 13% | Có category; thiếu trang browse đầy đủ + facet sidebar |
| Autocomplete / gợi-ý-gõ | 🟠 NEEDED | 0% | Amazon/Shopee baseline |
| Đánh giá & xếp hạng (reviews/ratings) | 🟠 NEEDED | 0% | **Cốt lõi niềm tin mua hàng** — chưa có |
| Hỏi-đáp sản phẩm (Q&A) | 🟠 NEEDED | 0% | — |
| Yêu cầu trả hàng / hoàn tiền (buyer) | 🟠 NEEDED | 0% | Returns/refund request flow |
| Mua lại nhanh (reorder) | 🟠 NEEDED | 0% | — |
| Wishlist / so sánh / vừa-xem | 🟠 NEEDED | 0% | save-for-later ≠ wishlist công khai |
| Thông báo đơn (push/SMS/Zalo) | 🟠 NEEDED | 0% | VN kỳ vọng tin nhắn cập nhật |
| Đánh giá sau mua + ảnh review | 🟠 NEEDED | 0% | UGC = data + trust |

---

## 3. Người bán — Merchant / Seller-Central parity  — 🟡 ~18%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Quản lý SP (CRUD + biến thể + ảnh gallery) | 🟢 DONE | 30% | Parent/child variant, gallery object-store/CDN |
| Danh mục sản phẩm | 🟢 DONE | 30% | FK category, RLS-scoped |
| Nhập hàng loạt CSV | 🟢 DONE | 28% | Bulk import → denorm Vespa |
| Vòng đời SP (active/inactive/archived/draft) | 🟢 DONE | 30% | Soft-delete đảo được |
| Tồn kho 1 điểm (atomic decrement) | 🟢 DONE | 30% | CAS atomic, chống oversell |
| Tồn kho đa điểm/kho (multi-location) | 🔵 PLANNED | 0% | Hiện stock global |
| Storefront status (bật/tắt gian hàng) | 🟢 DONE | 28% | StorefrontStatusGuard |
| Đa cửa hàng + nhân viên (staff) | 🟢 DONE | 27% | 1 account nhiều shop, role staff |
| Khuyến mãi (promo code, free-ship, free-gift) | 🟡 PARTIAL | 20% | Promo fixture; thiếu campaign/voucher engine |
| Quản lý đơn (merchant view) | 🟡 PARTIAL | 17% | Có order/transaction; thiếu màn quản-lý-đơn merchant đầy đủ |
| Dashboard phân tích (analytics) | 🟡 PARTIAL | 17% | Per-product/category matview; action-cards display-only, **thiếu endpoint** |
| AI hỗ trợ quyết định (restock/price/promo) | 🔵 PLANNED | 5% | MCP stub `suggest_*` có; chưa nối model/UI |
| Dự báo nhu cầu (forecast) | 🔵 PLANNED | 2% | Chronos-2/AutoGluon — data pipeline sẵn, model chưa |
| Quản lý trả hàng/hoàn tiền (merchant) | 🟠 NEEDED | 0% | — |
| Quản lý đánh giá (review moderation) | 🟠 NEEDED | 0% | — |
| Quảng cáo / sponsored listing | 🟠 NEEDED | 0% | Doanh thu ads (Shopee/Lazada monetize) |
| Quyết toán/payout cho seller | 🟡 PARTIAL | 10% | Payment settle có; payout-to-merchant flow chưa rõ |
| Phân tích đối thủ (Shopee price) | 🟡 PARTIAL | 10% | Mock fixture V008; crawler thật chưa code |

---

## 4. Nhà phân phối — Supplier / B2B  — 🔵 ~3%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Mô hình actor `supplier` (RBAC + tenant.type) | 🟡 PARTIAL | 8% | Khung phân quyền có; chưa có luồng nghiệp vụ |
| Luồng PO (supplier → merchant → buyer) | 🔵 PLANNED | 0% | Procurement epic |
| **Công nợ B2B (mua-trả-sau)** | 🟠 NEEDED | 0% | Tập quán VN: phân phối↔shop công nợ — **chưa có** |
| Bảng giá B2B + MOQ + bậc số lượng | 🟠 NEEDED | 0% | — |
| Tồn kho multi-echelon (supplier→shop) | 🔵 PLANNED | 0% | SCOT Lagrangian |
| **Hóa đơn điện tử B2B (e-invoice)** | 🟠 NEEDED | 0% | **NĐ 123/2024 bắt buộc** — chặn hợp pháp hoá B2B |
| Đối soát/đặt hàng định kỳ (recurring PO) | 🟠 NEEDED | 0% | — |

---

## 5. AI & Intelligence (khác biệt cốt lõi)  — 🟡 ~12%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| 6 intent hội thoại (search/voice-buy/voice-analyze/reco/import/cart) | 🟡 PARTIAL | 22% | LangGraph live; 4/6 cần hardening (xem §2,§3) |
| Vespa hybrid search (BM25 + CLIP 512d + cross-encoder) | 🟢 DONE | 28% | 6 rank-profile, denorm 1-RTT |
| Pipeline behavior-event (31 type → Postgres → Vespa signal) | 🟢 DONE | 28% | Tracker SDK, batch 5min + realtime |
| Voice AI (TTS/STT + cost metric) | 🟢 DONE | 27% | Gemini speech, webm opus, llm_traces |
| Image AI (object-detect + NN search) | 🟢 DONE | 27% | Vision analyze, image_similarity profile |
| Reco (item-item + co-purchase + trend) | 🟡 PARTIAL | 15% | Blend 3-signal; **thiếu two-tower + LTR đa-mục-tiêu** |
| Reco 2-stage (two-tower → multi-objective LTR) | 🔵 PLANNED | 3% | Own-embedding, LTR training loop |
| **Dự báo nhu cầu (Chronos-2/AutoGluon probabilistic)** | 🔵 PLANNED | 2% | lost-sales, lead-time-variance, quantile |
| **Lending scorecard (PD/LGD/EAD, %-doanh-số)** | 💡 VISION | 2% | Stub `suggest_loan`; Stripe-Capital model |
| Trợ lý Gen-AI khám phá (Rufus-style) | 💡 VISION | 0% | — |
| LLM cost architecture (provider-swap + lite-routing) | 🟡 PARTIAL | 13% | Abstraction + durable trace; route-by-difficulty chưa live |
| Eval harness (snapshot gate CI + live baseline) | 🟡 PARTIAL | 17% | Mock-gate xanh; harness đầy đủ + canh degrade chưa |
| PII redaction trước LLM US (cross-border) | 🟢 DONE | 28% | Fail-closed, consent-gated |
| AI v6 (planner+executor DAG, guardrails, context-budget) | 🔵 PLANNED | 2% | 6 nâng cấp |
| ML-ops/governance (drift/skew/canary/SHAP/CECL) | 🔵 PLANNED | 0% | Bắt buộc khi ship ML thật |

---

## 6. Thanh toán & Tài chính (VN rails + parity)  — 🟡 ~23%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| VNPay / Momo / ZaloPay (online) | 🟢 DONE | 28% | Adapter ký-verify, IPN idempotent; **tunnel live-test cuối còn chờ** |
| COD + chuyển khoản ngân hàng | 🟢 DONE | 30% | Offline method |
| Đối soát + auto-sweep IPN sót | 🟢 DONE | 30% | Reconcile use-case, safety-net |
| Hoàn tiền (refund) | 🟡 PARTIAL | 20% | Momo loop sandbox xong; VNPay/ZaloPay e2e chờ tunnel |
| **VietQR / QR ngân hàng tức thì** | 🟠 NEEDED | 0% | **Rail phổ biến nhất VN hiện nay** — chưa có |
| **Hóa đơn điện tử (e-invoice)** | 🟠 NEEDED | 0% | NĐ 123/2024 bắt buộc |
| Thuế/VAT tự động | 🟠 NEEDED | 0% | Khai báo VAT |
| Ví/escrow nền tảng | 🟠 NEEDED | 0% | Giữ tiền marketplace |
| Payout cho merchant/supplier | 🟡 PARTIAL | 10% | Settle có; payout flow chưa đủ |
| **Lending / BNPL (%-doanh-số)** | 💡 VISION | 2% | Revenue-lever lớn, gap tín dụng SME VN |
| Usage metering (billing SaaS per-tenant) | 🟡 PARTIAL | 13% | Bảng `usage_events`; chưa nối billing |

---

## 7. Logistics & Vận chuyển (VN)  — 🟠 ~3%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Sổ địa chỉ + phí ship cơ bản | 🟢 DONE | 27% | Nhập tay + rate theo method |
| **Tích hợp GHN/GHTK/J&T/Ahamove** | 🟠 NEEDED | 0% | **Bắt buộc cho fulfillment VN** |
| In vận đơn / tạo đơn vận chuyển | 🟠 NEEDED | 0% | — |
| Đồng bộ trạng thái vận chuyển (tracking) | 🟠 NEEDED | 0% | — |
| Đối soát COD với hãng vận chuyển | 🟠 NEEDED | 0% | Tiền COD ↔ carrier |
| Tính phí ship realtime theo vùng | 🟠 NEEDED | 0% | API hãng ship |
| Đa kho / chọn kho gần nhất | 🔵 PLANNED | 0% | — |

---

## 8. Marketing & Tăng trưởng  — 🟠 ~5%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Mã giảm giá / free-ship / free-gift | 🟢 DONE | 23% | Promo engine cơ bản |
| Voucher / campaign engine | 🟠 NEEDED | 0% | Chiến dịch, điều kiện áp |
| Flash sale / deal theo giờ | 🟠 NEEDED | 0% | — |
| Tích điểm / loyalty | 🟠 NEEDED | 0% | — |
| Giới thiệu bạn (referral) | 🟠 NEEDED | 0% | — |
| Quảng cáo nội sàn (ads/sponsored) | 🟠 NEEDED | 0% | Doanh thu ads |
| Email / SMS / Zalo marketing | 🟠 NEEDED | 0% | — |
| Affiliate / KOC | 🟠 NEEDED | 0% | — |
| Live commerce (bán hàng live) | 🟠 NEEDED | 0% | Shopee Live >60% engagement VN |

---

## 9. Tin cậy, Tuân thủ & Quản trị (VN law + GDPR)  — 🟢 ~28%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Consent (analytics/ai/marketing) | 🟢 DONE | 30% | Per-purpose, tracker consent-gate |
| DSAR (truy cập Art15 + xoá Art17) | 🟢 DONE | 28% | Export 12-15 bảng, erasure atomic |
| Chính sách lưu trữ (retention) | 🟢 DONE | 27% | Category-driven; scheduler purge một phần |
| Audit hash-chain bất biến + WORM anchor | 🟢 DONE | 30% | SHA256 chain, Ed25519, epoch ceremony |
| Phát hiện vi phạm + runbook 72h | 🟢 DONE | 27% | breach_incidents register |
| Redaction PII cross-border LLM | 🟢 DONE | 28% | Fail-closed mask |
| Vòng đời tenant (soft-close + purge) | 🟢 DONE | 28% | Status machine |
| DPA/RoPA legal posture (tài liệu) | 🟡 PARTIAL | 17% | Living doc; cần chốt trước go-live |
| Chống gian lận (fraud detection) | 🟠 NEEDED | 0% | Risk-scoring giao dịch |
| Kiểm duyệt nội dung (content moderation) | 🟠 NEEDED | 0% | UGC/review/SP vi phạm |
| Validate MIME ảnh upload (security) | 🟠 NEEDED | 0% | **P0 security gap** — base64 chưa check type |

---

## 10. Kênh & Mobile  — 🟠 ~5%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Web responsive (Next.js, phoneFrame 414px) | 🟢 DONE | 28% | 60 page, mobile-first; a11y/i18n còn thiếu |
| PWA (cài như app, offline) | 🟠 NEEDED | 0% | Bước đệm rẻ trước native |
| **App mobile iOS/Android** | 🟠 NEEDED | 0% | **80%+ TMĐT VN là mobile** |
| **Zalo (login + OA chat + ZNS thông báo)** | 🟠 NEEDED | 0% | **50M+ user VN** |
| Thông báo SMS (OTP + đơn hàng) | 🟠 NEEDED | 0% | — |
| i18n tiếng Việt (UI strings) | 🟠 NEEDED | 0% | Hiện nhiều trang English |
| Social commerce (FB/TikTok Shop sync) | 🟠 NEEDED | 0% | — |

---

## 11. Vận hành nền tảng (Platform Ops)  — 🟡 ~22%

| Tính năng | Trạng thái | % | Mô tả |
|---|---|---|---|
| Observability OTel (logs/traces/metrics) | 🟢 DONE | 28% | LGTM stack |
| Worker: CDC consumer + payment + housekeeper + audit + retention | 🟢 DONE | 25% | Live + test; còn vài worker skeleton |
| Worker: inventory + notification + card-gen + crawler | 🟡 PARTIAL | 7% | Skeleton/docstring |
| Alerting + SLO error-budget + dashboard-as-code | 🟡 PARTIAL | 10% | Rule có; Alertmanager + 5 dashboard chưa |
| Admin console (breach/retention/audit/tenant) | 🟡 PARTIAL | 18% | Một số màn admin |
| Backup/DR (Postgres + Vespa rebuild) | 🟠 NEEDED | 3% | Vespa restore chưa validate |
| k6 load/perf baseline | 🟡 PARTIAL | 17% | REST p95 đo; stream/canary chưa |

---

## 🗺️ Lộ trình đề xuất (sequencing)

> Nguyên tắc: **hoàn thiện lõi đã chạy → lấp must-have VN → bật moat trí-tuệ → mở rộng B2B/lending/scale.**

### Giai đoạn NOW — "bán được thật ở VN" (0–3 tháng)
Hoàn thiện 4 intent dở (facets/grid/voice-review) · **VietQR** · **e-invoice** · **logistics GHN/GHTK** · **Zalo (login+ZNS)** · **PWA/app** · thông báo SMS/Zalo đơn hàng · **reviews/ratings** · returns/refund · auth MFA · validate MIME (P0 security) · tunnel live-test thanh toán.

### Giai đoạn NEXT — "moat + monetize" (3–9 tháng)
**Forecast nhập hàng live** (Chronos-2) · reco two-tower+LTR · merchant-decision AI (restock/price/promo) live · **lending pilot** (%-doanh-số) · ads/sponsored · voucher/campaign engine · loyalty · quản-lý-đơn merchant đầy đủ · payout · ML-ops cơ bản (eval/canary).

### Giai đoạn LATER — "nền tảng & mở rộng" (9–18 tháng)
B2B phân phối đầy đủ (PO + **công nợ** + multi-echelon) · live commerce · HA/scale (pgbouncer + Vespa cluster) · ML-governance (drift/SHAP/CECL) · mở rộng SEA (Thái/Indo/Ấn) · own-embedding · fraud/content-moderation.

---

## 📊 Bảng tổng hợp % theo mảng (dashboard)

| # | Mảng | Trạng thái | % |
|---|---|---|---|
| 1 | Nền tảng & Multi-tenant | 🟢 | ~28% |
| 2 | Người mua (buyer) | 🟡 | ~20% |
| 3 | Người bán (merchant) | 🟡 | ~18% |
| 4 | Nhà phân phối (B2B) | 🔵 | ~3% |
| 5 | AI & Intelligence | 🟡 | ~12% |
| 6 | Thanh toán & Tài chính | 🟡 | ~23% |
| 7 | Logistics VN | 🟠 | ~3% |
| 8 | Marketing & Growth | 🟠 | ~5% |
| 9 | Tin cậy & Tuân thủ | 🟢 | ~28% |
| 10 | Kênh & Mobile | 🟠 | ~5% |
| 11 | Vận hành nền tảng | 🟡 | ~22% |
| — | **TỔNG (vision đầy đủ)** | 🟡 | **~13–15%** |
| — | *(Lõi merchant↔buyer MVP)* | 🟢 | *~27%* |

---

## 📎 Ghi chú cho người đọc

- **Founder/Dev:** đây là backlog-master. Mỗi slice mới (`docs/slices/S-*.md`) nên trỏ về mục tương ứng ở đây và cập nhật % khi đóng. Các mục 🟠 NEEDED và 🔵 PLANNED là nguồn để cắt slice kế.
- **Nhà đầu tư:** đọc §0 (tóm tắt) + bảng dashboard. Cột "Trạng thái" cho biết **đâu là thật (🟢), đâu là lộ trình (🔵/🟠/💡)** — ICP không trộn lẫn hai thứ.
- **Số liệu thị trường (TAM/ARR/LTV):** cố ý KHÔNG đưa vào đây — cần validate bằng dữ liệu thật + traction pilot trước khi lên pitch-deck.

> *Cập nhật lần cuối: 2026-06-23. Tài liệu sống — sửa trực tiếp khi trạng thái đổi.*
