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
Lõi thương mại merchant↔buyer  ███████░░░░░░░░░░░░░ ~40%   (đã de-risk)
Khám phá/discovery (AI search) ████░░░░░░░░░░░░░░░░  ~20%
Trí tuệ (forecast/reco/lending)█░░░░░░░░░░░░░░░░░░░  ~5%   (data sẵn, model chưa)
B2B nhà phân phối               ██░░░░░░░░░░░░░░░░░░  ~10%
VN-thiết-yếu (e-invoice/ship/Zalo/mobile/VietQR) █░░░░░░░  ~5%
Tuân thủ & nền tảng             █████████░░░░░░░  ~55%
─────────────────────────────────────────────────────────
TỔNG (vision đầy đủ)            █████░░░░░░░░░░░░░░░  ~20–25%
```


---

## 1. Nền tảng & Multi-tenant (Foundation)  — 🟢 ~85%

| Tính năng | Trạng thái | % | Mô tả ngắn | Nguồn |
|---|---|---|---|---|
| Multi-tenant isolation (RLS + GUC + FORCE) | 🟢 DONE | 95% | 46 cột `tenant_id`, RLS NULLIF-hardened fail-closed, e2e cross-tenant đỏ-khi-sai | ADR-040, V011-026 |
| RLS 3-pool runtime (icp_app NOBYPASSRLS) | 🟢 DONE | 95% | Tách pool app/migrate/privileged — chống bypass | ADR-040 Am 2026-06-17 |
| Auth (JWT + session + refresh + switch-tenant) | 🟢 DONE | 85% | 10 route auth; thiếu MFA/SSO/lockout (xem §3 auth-hardening) | S-03, ADR-046 |
| RBAC actor taxonomy (buyer/merchant/supplier/staff/admin) | 🟢 DONE | 80% | `tenants.type` + `tenant_memberships` + role-guard; thiếu role auditor/support | ADR-062, V034 |
| Idempotency (Redis SETNX + response cache) | 🟢 DONE | 95% | Chống double-charge/replay; 2-lớp payment_callbacks + CDC inbox | ADR-004 |
| Event backbone (CDC Debezium + Redpanda + outbox) | 🟢 DONE | 85% | Outbox same-txn, CDC slot-aware, exactly-once inbox | ADR-006/058, V017-020 |
| Observability (OTel logs/traces/metrics) | 🟢 DONE | 85% | 170+ log type, trace xuyên 3-service; thiếu vài histogram (Vespa/Kafka) | ADR-011 |
| Contract-first FE↔BE (OpenAPI codegen + Zod SOT) | 🟢 DONE | 95% | CI drift-gate, generated client | ADR-017 |
| CI/CD hard-gate (lint/test/typecheck/coverage/drift) | 🟢 DONE | 90% | 8 job, coverage floor ≥70%, facts-drift guard | S-P0-03/T01 |
| HA/scale (pgbouncer, Vespa cluster, multi-server) | 🔵 PLANNED | 10% | Single-node hiện tại; pooler + Vespa HA = scale-phase | ADR-067, BACKLOG P2 |

---

## 2. Người mua — Buyer Experience (Shopee/Amazon buyer parity)  — 🟡 ~60%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Storefront + PDP (trang chi tiết SP, buyer-scoped) | 🟢 DONE | 90% | Storefront read API field-projected + variant selector | ADR-074, S-INT05/T02a-b |
| Giỏ hàng (add/qty/remove/promo/variant) | 🟢 DONE | 95% | Composite line-key variant, tax, honor-lower pricing | S-INT05, ADR-073 |
| Lưu để sau (save-for-later) | 🟢 DONE | 90% | Bảng riêng RLS, move-to-cart | V055, ADR-073 |
| Checkout + sổ địa chỉ + phí ship | 🟢 DONE | 90% | Address-book, shipping rate, COD/bank | S-INT06, ADR-070 |
| Theo dõi đơn realtime (SSE) | 🟢 DONE | 85% | Order-status stream | S-INT06/T05 |
| Tìm bằng text (AI hybrid Vespa) | 🟡 PARTIAL | 60% | BM25+CLIP chạy; **thiếu facets/sort/pagination/filter** | Intent-03 |
| Tìm/mua bằng giọng nói | 🟡 PARTIAL | 55% | STT+parse chạy; **thiếu review-state, OOS-substitute, TTS-confirm** | Intent-02 |
| Gợi ý bằng ảnh (visual reco) | 🟡 PARTIAL | 55% | Vision + 3-signal blend; **empty-state dead-end, thiếu grid** | Intent-04 |
| Duyệt theo danh mục / browse | 🟡 PARTIAL | 40% | Có category; thiếu trang browse đầy đủ + facet sidebar | ADR-071 |
| Autocomplete / gợi-ý-gõ | 🟠 NEEDED | 0% | Amazon/Shopee baseline | — |
| Đánh giá & xếp hạng (reviews/ratings) | 🟠 NEEDED | 0% | **Cốt lõi niềm tin mua hàng** — chưa có | — |
| Hỏi-đáp sản phẩm (Q&A) | 🟠 NEEDED | 0% | — | — |
| Yêu cầu trả hàng / hoàn tiền (buyer) | 🟠 NEEDED | 0% | Returns/refund request flow | — |
| Mua lại nhanh (reorder) | 🟠 NEEDED | 0% | — | — |
| Wishlist / so sánh / vừa-xem | 🟠 NEEDED | 0% | save-for-later ≠ wishlist công khai | — |
| Thông báo đơn (push/SMS/Zalo) | 🟠 NEEDED | 0% | VN kỳ vọng tin nhắn cập nhật | xem §7,§10 |
| Đánh giá sau mua + ảnh review | 🟠 NEEDED | 0% | UGC = data + trust | — |

---

## 3. Người bán — Merchant / Seller-Central parity  — 🟡 ~55%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Quản lý SP (CRUD + biến thể + ảnh gallery) | 🟢 DONE | 90% | Parent/child variant, gallery object-store/CDN | S-CATALOG, ADR-071/072 |
| Danh mục sản phẩm | 🟢 DONE | 90% | FK category, RLS-scoped | ADR-071, V053 |
| Nhập hàng loạt CSV | 🟢 DONE | 85% | Bulk import → denorm Vespa | S-CATALOG |
| Vòng đời SP (active/inactive/archived/draft) | 🟢 DONE | 90% | Soft-delete đảo được | ADR-071 |
| Tồn kho 1 điểm (atomic decrement) | 🟢 DONE | 90% | CAS atomic, chống oversell | ADR-055 |
| Tồn kho đa điểm/kho (multi-location) | 🔵 PLANNED | 0% | Hiện stock global | ADR-063 §S-C8 |
| Storefront status (bật/tắt gian hàng) | 🟢 DONE | 85% | StorefrontStatusGuard | S-P0 |
| Đa cửa hàng + nhân viên (staff) | 🟢 DONE | 80% | 1 account nhiều shop, role staff | ADR-046/062 |
| Khuyến mãi (promo code, free-ship, free-gift) | 🟡 PARTIAL | 60% | Promo fixture; thiếu campaign/voucher engine | S-INT05 |
| Quản lý đơn (merchant view) | 🟡 PARTIAL | 50% | Có order/transaction; thiếu màn quản-lý-đơn merchant đầy đủ | S-INT06 |
| Dashboard phân tích (analytics) | 🟡 PARTIAL | 50% | Per-product/category matview; action-cards display-only, **thiếu endpoint** | S-INT07 |
| AI hỗ trợ quyết định (restock/price/promo) | 🔵 PLANNED | 15% | MCP stub `suggest_*` có; chưa nối model/UI | ADR-063 §3 |
| Dự báo nhu cầu (forecast) | 🔵 PLANNED | 5% | Chronos-2/AutoGluon — data pipeline sẵn, model chưa | ADR-063 §3 |
| Quản lý trả hàng/hoàn tiền (merchant) | 🟠 NEEDED | 0% | — | — |
| Quản lý đánh giá (review moderation) | 🟠 NEEDED | 0% | — | — |
| Quảng cáo / sponsored listing | 🟠 NEEDED | 0% | Doanh thu ads (Shopee/Lazada monetize) | — |
| Quyết toán/payout cho seller | 🟡 PARTIAL | 30% | Payment settle có; payout-to-merchant flow chưa rõ | ADR-060 |
| Phân tích đối thủ (Shopee price) | 🟡 PARTIAL | 30% | Mock fixture V008; crawler thật chưa code | ADR-039 |

---

## 4. Nhà phân phối — Supplier / B2B  — 🔵 ~10%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Mô hình actor `supplier` (RBAC + tenant.type) | 🟡 PARTIAL | 25% | Khung phân quyền có; chưa có luồng nghiệp vụ | ADR-062 |
| Luồng PO (supplier → merchant → buyer) | 🔵 PLANNED | 0% | Procurement epic | ADR-063 §6 S-C7 |
| **Công nợ B2B (mua-trả-sau)** | 🟠 NEEDED | 0% | Tập quán VN: phân phối↔shop công nợ — **chưa có** | BACKLOG P1 #99 |
| Bảng giá B2B + MOQ + bậc số lượng | 🟠 NEEDED | 0% | — | — |
| Tồn kho multi-echelon (supplier→shop) | 🔵 PLANNED | 0% | SCOT Lagrangian | ADR-063 §3 |
| **Hóa đơn điện tử B2B (e-invoice)** | 🟠 NEEDED | 0% | **NĐ 123/2024 bắt buộc** — chặn hợp pháp hoá B2B | BACKLOG P2 #85 |
| Đối soát/đặt hàng định kỳ (recurring PO) | 🟠 NEEDED | 0% | — | — |

---

## 5. AI & Intelligence (khác biệt cốt lõi)  — 🟡 ~35%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| 6 intent hội thoại (search/voice-buy/voice-analyze/reco/import/cart) | 🟡 PARTIAL | 65% | LangGraph live; 4/6 cần hardening (xem §2,§3) | S-04..S-10 |
| Vespa hybrid search (BM25 + CLIP 512d + cross-encoder) | 🟢 DONE | 85% | 6 rank-profile, denorm 1-RTT | ADR-001/024/036 |
| Pipeline behavior-event (31 type → Postgres → Vespa signal) | 🟢 DONE | 85% | Tracker SDK, batch 5min + realtime | ADR-012/013 |
| Voice AI (TTS/STT + cost metric) | 🟢 DONE | 80% | Gemini speech, webm opus, llm_traces | S-08 |
| Image AI (object-detect + NN search) | 🟢 DONE | 80% | Vision analyze, image_similarity profile | S-07 |
| Reco (item-item + co-purchase + trend) | 🟡 PARTIAL | 45% | Blend 3-signal; **thiếu two-tower + LTR đa-mục-tiêu** | ADR-043, ADR-063 |
| Reco 2-stage (two-tower → multi-objective LTR) | 🔵 PLANNED | 10% | Own-embedding, LTR training loop | ADR-063 §3 |
| **Dự báo nhu cầu (Chronos-2/AutoGluon probabilistic)** | 🔵 PLANNED | 5% | lost-sales, lead-time-variance, quantile | ADR-063 §3 |
| **Lending scorecard (PD/LGD/EAD, %-doanh-số)** | 💡 VISION | 5% | Stub `suggest_loan`; Stripe-Capital model | ADR-063 §5 |
| Trợ lý Gen-AI khám phá (Rufus-style) | 💡 VISION | 0% | — | ADR-063 §3 |
| LLM cost architecture (provider-swap + lite-routing) | 🟡 PARTIAL | 40% | Abstraction + durable trace; route-by-difficulty chưa live | ADR-054 |
| Eval harness (snapshot gate CI + live baseline) | 🟡 PARTIAL | 50% | Mock-gate xanh; harness đầy đủ + canh degrade chưa | ADR-051 #2 |
| PII redaction trước LLM US (cross-border) | 🟢 DONE | 85% | Fail-closed, consent-gated | ADR-068 |
| AI v6 (planner+executor DAG, guardrails, context-budget) | 🔵 PLANNED | 5% | 6 nâng cấp | ADR-051 |
| ML-ops/governance (drift/skew/canary/SHAP/CECL) | 🔵 PLANNED | 0% | Bắt buộc khi ship ML thật | ADR-063 §4 |

---

## 6. Thanh toán & Tài chính (VN rails + parity)  — 🟡 ~70%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| VNPay / Momo / ZaloPay (online) | 🟢 DONE | 85% | Adapter ký-verify, IPN idempotent; **tunnel live-test cuối còn chờ** | ADR-038/060 |
| COD + chuyển khoản ngân hàng | 🟢 DONE | 90% | Offline method | V047 |
| Đối soát + auto-sweep IPN sót | 🟢 DONE | 90% | Reconcile use-case, safety-net | ADR-060 Am 3-4 |
| Hoàn tiền (refund) | 🟡 PARTIAL | 60% | Momo loop sandbox xong; VNPay/ZaloPay e2e chờ tunnel | ADR-060e |
| **VietQR / QR ngân hàng tức thì** | 🟠 NEEDED | 0% | **Rail phổ biến nhất VN hiện nay** — chưa có | — |
| **Hóa đơn điện tử (e-invoice)** | 🟠 NEEDED | 0% | NĐ 123/2024 bắt buộc | BACKLOG P2 #85 |
| Thuế/VAT tự động | 🟠 NEEDED | 0% | Khai báo VAT | BACKLOG P2 #86 |
| Ví/escrow nền tảng | 🟠 NEEDED | 0% | Giữ tiền marketplace | — |
| Payout cho merchant/supplier | 🟡 PARTIAL | 30% | Settle có; payout flow chưa đủ | ADR-060 |
| **Lending / BNPL (%-doanh-số)** | 💡 VISION | 5% | Revenue-lever lớn, gap tín dụng SME VN | ADR-063 §5 |
| Usage metering (billing SaaS per-tenant) | 🟡 PARTIAL | 40% | Bảng `usage_events`; chưa nối billing | ADR-044 |

---

## 7. Logistics & Vận chuyển (VN)  — 🟠 ~10%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Sổ địa chỉ + phí ship cơ bản | 🟢 DONE | 80% | Nhập tay + rate theo method | ADR-070 |
| **Tích hợp GHN/GHTK/J&T/Ahamove** | 🟠 NEEDED | 0% | **Bắt buộc cho fulfillment VN** | BACKLOG P2 #9 |
| In vận đơn / tạo đơn vận chuyển | 🟠 NEEDED | 0% | — | — |
| Đồng bộ trạng thái vận chuyển (tracking) | 🟠 NEEDED | 0% | — | — |
| Đối soát COD với hãng vận chuyển | 🟠 NEEDED | 0% | Tiền COD ↔ carrier | — |
| Tính phí ship realtime theo vùng | 🟠 NEEDED | 0% | API hãng ship | — |
| Đa kho / chọn kho gần nhất | 🔵 PLANNED | 0% | — | ADR-063 |

---

## 8. Marketing & Tăng trưởng  — 🟠 ~15%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Mã giảm giá / free-ship / free-gift | 🟢 DONE | 70% | Promo engine cơ bản | S-INT05 |
| Voucher / campaign engine | 🟠 NEEDED | 0% | Chiến dịch, điều kiện áp | — |
| Flash sale / deal theo giờ | 🟠 NEEDED | 0% | — | — |
| Tích điểm / loyalty | 🟠 NEEDED | 0% | — | — |
| Giới thiệu bạn (referral) | 🟠 NEEDED | 0% | — | — |
| Quảng cáo nội sàn (ads/sponsored) | 🟠 NEEDED | 0% | Doanh thu ads | — |
| Email / SMS / Zalo marketing | 🟠 NEEDED | 0% | — | — |
| Affiliate / KOC | 🟠 NEEDED | 0% | — | — |
| Live commerce (bán hàng live) | 🟠 NEEDED | 0% | Shopee Live >60% engagement VN | — |

---

## 9. Tin cậy, Tuân thủ & Quản trị (VN law + GDPR)  — 🟢 ~85%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Consent (analytics/ai/marketing) | 🟢 DONE | 90% | Per-purpose, tracker consent-gate | ADR-041 |
| DSAR (truy cập Art15 + xoá Art17) | 🟢 DONE | 85% | Export 12-15 bảng, erasure atomic | ADR-041 |
| Chính sách lưu trữ (retention) | 🟢 DONE | 80% | Category-driven; scheduler purge một phần | ADR-041/067 |
| Audit hash-chain bất biến + WORM anchor | 🟢 DONE | 90% | SHA256 chain, Ed25519, epoch ceremony | ADR-042/061 |
| Phát hiện vi phạm + runbook 72h | 🟢 DONE | 80% | breach_incidents register | ADR-069 |
| Redaction PII cross-border LLM | 🟢 DONE | 85% | Fail-closed mask | ADR-068 |
| Vòng đời tenant (soft-close + purge) | 🟢 DONE | 85% | Status machine | ADR-066 |
| DPA/RoPA legal posture (tài liệu) | 🟡 PARTIAL | 50% | Living doc; cần chốt trước go-live | S-C6e-3 |
| Chống gian lận (fraud detection) | 🟠 NEEDED | 0% | Risk-scoring giao dịch | ADR-063 (substrate) |
| Kiểm duyệt nội dung (content moderation) | 🟠 NEEDED | 0% | UGC/review/SP vi phạm | — |
| Validate MIME ảnh upload (security) | 🟠 NEEDED | 0% | **P0 security gap** — base64 chưa check type | BACKLOG P0 #8 |

---

## 10. Kênh & Mobile  — 🟠 ~15%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Web responsive (Next.js, phoneFrame 414px) | 🟢 DONE | 85% | 60 page, mobile-first; a11y/i18n còn thiếu | S-01, ADR-022/023 |
| PWA (cài như app, offline) | 🟠 NEEDED | 0% | Bước đệm rẻ trước native | — |
| **App mobile iOS/Android** | 🟠 NEEDED | 0% | **80%+ TMĐT VN là mobile** | BACKLOG P1 #50 |
| **Zalo (login + OA chat + ZNS thông báo)** | 🟠 NEEDED | 0% | **50M+ user VN** | BACKLOG P1 #77 |
| Thông báo SMS (OTP + đơn hàng) | 🟠 NEEDED | 0% | — | — |
| i18n tiếng Việt (UI strings) | 🟠 NEEDED | 0% | Hiện nhiều trang English | BACKLOG P2 #23 |
| Social commerce (FB/TikTok Shop sync) | 🟠 NEEDED | 0% | — | — |

---

## 11. Vận hành nền tảng (Platform Ops)  — 🟡 ~65%

| Tính năng | Trạng thái | % | Mô tả | Nguồn |
|---|---|---|---|---|
| Observability OTel (logs/traces/metrics) | 🟢 DONE | 85% | LGTM stack | ADR-011 |
| Worker: CDC consumer + payment + housekeeper + audit + retention | 🟢 DONE | 75% | Live + test; còn vài worker skeleton | S-P0-04 |
| Worker: inventory + notification + card-gen + crawler | 🟡 PARTIAL | 20% | Skeleton/docstring | BACKLOG #11 |
| Alerting + SLO error-budget + dashboard-as-code | 🟡 PARTIAL | 30% | Rule có; Alertmanager + 5 dashboard chưa | BACKLOG #7 |
| Admin console (breach/retention/audit/tenant) | 🟡 PARTIAL | 55% | Một số màn admin | S-C6 |
| Backup/DR (Postgres + Vespa rebuild) | 🟠 NEEDED | 10% | Vespa restore chưa validate | BACKLOG P1 |
| k6 load/perf baseline | 🟡 PARTIAL | 50% | REST p95 đo; stream/canary chưa | S-P0-03/T05 |

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
| 1 | Nền tảng & Multi-tenant | 🟢 | ~85% |
| 2 | Người mua (buyer) | 🟡 | ~60% |
| 3 | Người bán (merchant) | 🟡 | ~55% |
| 4 | Nhà phân phối (B2B) | 🔵 | ~10% |
| 5 | AI & Intelligence | 🟡 | ~35% |
| 6 | Thanh toán & Tài chính | 🟡 | ~70% |
| 7 | Logistics VN | 🟠 | ~10% |
| 8 | Marketing & Growth | 🟠 | ~15% |
| 9 | Tin cậy & Tuân thủ | 🟢 | ~85% |
| 10 | Kênh & Mobile | 🟠 | ~15% |
| 11 | Vận hành nền tảng | 🟡 | ~65% |
| — | **TỔNG (vision đầy đủ)** | 🟡 | **~40–45%** |
| — | *(Lõi merchant↔buyer MVP)* | 🟢 | *~80%* |

---

## 📎 Ghi chú cho người đọc

- **Founder/Dev:** đây là backlog-master. Mỗi slice mới (`docs/slices/S-*.md`) nên trỏ về mục tương ứng ở đây và cập nhật % khi đóng. Các mục 🟠 NEEDED và 🔵 PLANNED là nguồn để cắt slice kế.
- **Nhà đầu tư:** đọc §0 (tóm tắt) + bảng dashboard. Cột "Trạng thái" cho biết **đâu là thật (🟢), đâu là lộ trình (🔵/🟠/💡)** — ICP không trộn lẫn hai thứ.
- **Số liệu thị trường (TAM/ARR/LTV):** cố ý KHÔNG đưa vào đây — cần validate bằng dữ liệu thật + traction pilot trước khi lên pitch-deck.

> *Cập nhật lần cuối: 2026-06-23. Tài liệu sống — sửa trực tiếp khi trạng thái đổi.*
