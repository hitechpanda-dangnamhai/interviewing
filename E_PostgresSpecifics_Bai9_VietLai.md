# 🎯 BÀI 9 — PostgreSQL specifics

**Phủ:** E-PG-001 → 008

> 💡 Bài này là **đặc thù riêng của engine Postgres** — phần lớn là **hệ quả trực tiếp của MVCC** (Bài 8). Hiểu **JSONB**, **VACUUM**, **wraparound**, **statistics**, **connection cost** và **chọn version** chính là biết "Postgres cư xử khác lý thuyết SQL chung ở chỗ nào".

---

## ① Mục tiêu & vị trí

Bài này về **đặc thù engine Postgres**. Nhiều thứ ở đây là **hệ quả trực tiếp của MVCC** (**Bài 8**) — MVCC tạo ra rác, nên Postgres cần cơ chế dọn rác riêng.

Vị trí: là **nền cho Bài 10** (tối ưu query) và **Bài 11** (connection pooling).

> 🎯 **Chốt định vị:** Hiểu lý thuyết SQL chung là chưa đủ — mỗi engine có "tính khí" riêng. Phần lớn sự cố production Postgres bắt nguồn từ việc *không hiểu MVCC sinh rác và cần VACUUM*.

---

## ② Giảng cơ bản → nâng cao

### (a) JSONB  `E-PG-001`

> **Ẩn dụ:** **json** (text) giống lưu một tờ giấy *chụp ảnh lại* — muốn tìm gì phải đọc lại cả tờ. **JSONB** giống lưu *đã phân loại sẵn vào ngăn* — tìm và đánh dấu (index) được ngay.

**JSONB** lưu **nhị phân** (khác **json** lưu **text**) → **query + index được**. Hợp với **dữ liệu bán cấu trúc/linh hoạt**.

> 🚨 **KHÔNG nên thay toàn bộ schema quan hệ bằng JSONB.** Vì sao? Vì bạn mất ràng buộc (**constraint**), mất tối ưu của cột thường, và rơi vào **schema drift** (Bài 2). JSONB cho *phần linh hoạt*; cột quan hệ cho *phần ổn định*.

#### 🔍 Index JSONB  `E-PG-002`

**GIN** index (ví dụ **`jsonb_path_ops`**) hỗ trợ phép **containment `@>`** ("chứa key/value này").

> ⚠️ **Hạn chế:** GIN trên JSONB **không nhanh bằng btree** trên cột thường cho **range/sort**. Nếu một thuộc tính hay dùng để lọc dải/sắp xếp → tách ra cột riêng có **btree** sẽ tốt hơn.

---

### (b) MVCC → VACUUM  `E-PG-003`

> **Ẩn dụ:** **MVCC** giữ nhiều phiên bản của mỗi dòng (Bài 8). Mỗi lần update/delete, version cũ không bị xóa ngay mà thành **"rác"** (**dead tuple**) — như giấy nháp vứt đầy bàn. **VACUUM** là người dọn bàn, gom rác để tái dùng chỗ.

**MVCC để lại dead tuples** (version cũ). **VACUUM**:
- **thu hồi space** (cho tái dùng nội bộ),
- **cập nhật visibility map** (giúp **index-only scan** ở Bài 4 hoạt động).
- **autovacuum** chạy nền tự động.

```text
Vì sao MVCC sinh dead tuple → cần VACUUM:
  T1: UPDATE row → tạo version MỚI, version CŨ vẫn nằm đó (cho reader đang xem)
       ↓
  T2: reader cũ kết thúc → version cũ không ai cần nữa = DEAD TUPLE (rác)
       ↓
  T3: dead tuple tích tụ dần → bảng phình
       ↓
  T4: VACUUM dọn → space được tái dùng cho dòng mới   ✅
```

#### 🔍 Bloat & VACUUM FULL  `E-PG-004`

**dead tuples tích tụ → bảng/index phình** (**bloat**).

| | **VACUUM** (thường) | **VACUUM FULL** |
|---|---------------------|------------------|
| Trả space về OS? | ❌ không (chỉ **tái dùng nội bộ**) | ✅ có (**rewrite** bảng) |
| Khóa | nhẹ | 🚨 **khóa độc quyền** (chặn mọi truy cập) |
| Chi phí | rẻ, chạy nền | đắt |
| Khi nào | thường xuyên (autovacuum) | hiếm, **tránh giờ cao điểm** |

> ⚠️ **VACUUM thường KHÔNG trả space về OS** — nó chỉ đánh dấu chỗ trống để tái dùng. Muốn lấy lại đĩa thật phải **VACUUM FULL** (rewrite toàn bảng + khóa độc quyền) → chỉ làm lúc vắng khách.

---

### (c) Mép giới hạn / sống còn

#### 🚨 XID wraparound  `E-PG-005`

> **Ẩn dụ:** **transaction ID** (**XID**) là **32-bit** → giống đồng hồ chỉ có số đếm hữu hạn, đếm hết sẽ **quay vòng** về 0. Nếu quay vòng mà chưa "đóng băng" (freeze) các dòng cũ, Postgres sẽ nhầm dòng cũ thành dòng tương lai → hỏng dữ liệu.

**XID wraparound:** **transaction ID 32-bit quay vòng**; nếu **vacuum freeze không kịp** → nguy cơ wraparound → **Postgres chuyển read-only để tự bảo vệ**.

> 🚨 **Điều tệ nhất:** Postgres *ngừng nhận ghi* (read-only) để tránh hỏng dữ liệu — service tê liệt phần ghi. Phải đảm bảo **autovacuum/freeze chạy đủ**. Đây là "ops sống còn".

#### 🔍 max_connections cao ≠ scale  `E-PG-006`

> **Ví dụ trực giác:** Mỗi **connection** trong Postgres = **một backend process** (một tiến trình riêng, tốn RAM/CPU). Mở 1000 connection như thuê 1000 nhân viên ngồi không chờ việc — tốn lương mà còn chen chỗ làm chậm cả văn phòng.

**max_connections cao ≠ scale.** Quá nhiều connection làm **chậm hơn**, không nhanh hơn.

> 🎯 **Chốt:** Dùng **pooler** (**Bài 11**) thay vì nâng **max_connections** lên 1000. Pooler tái dùng một số ít connection thật cho nhiều client.

#### 🔍 ANALYZE / statistics  `E-PG-007`

> **Ẩn dụ:** **planner** (bộ chọn kế hoạch query) như tài xế dùng bản đồ. **statistics** là bản đồ. Bản đồ cũ (đường đã đổi) → tài xế chọn đường sai. **stats cũ → chọn sai plan**.

**planner chọn plan dựa estimate từ stats**; **stats cũ/lệch → chọn sai plan** (ví dụ **seq scan** thay vì dùng **index**) → **query chậm đột ngột**.

```text
Stats cũ làm planner chọn sai plan:
  T1: bảng có 1.000 dòng → stats nói "nhỏ" → planner chọn seq scan (hợp lý)
  T2: import thêm 10.000.000 dòng    ← nhưng stats CHƯA cập nhật
       ↓
  T3: planner vẫn tưởng bảng nhỏ → vẫn seq scan 10tr dòng   ❌ chậm đột ngột
  ✅ chạy ANALYZE → stats mới → planner chuyển sang index scan
```

> 🎯 Cần **ANALYZE**/**autoanalyze** để cập nhật stats.

#### 🔍 Chọn version (6/2026)  `E-PG-008`

> 🔎 **Lưu ý (verify postgresql.org):**
> - Chọn **17** hoặc **18** (**stable**, **còn vá bảo mật**).
> - **Tránh 14** (**EOL 11/2026**) và **≤13** (**đã EOL** — hết hỗ trợ).
> - **KHÔNG chạy 19 beta** cho production.

> ⚠️ **EOL** (End Of Life) = phiên bản hết được vá bảo mật → chạy production rất rủi ro.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| **json** (text) | **jsonb** (nhị phân, index **GIN**) | **query/index được** |
| Thay **schema quan hệ** bằng **JSONB** | JSONB cho phần **bán cấu trúc**; cột quan hệ cho phần **ổn định** | tối ưu + ràng buộc |
| **max_connections = 1000** | **Pooler** (**PgBouncer**) + max_connections vừa phải | mỗi connection = **process** tốn |
| Chạy **PG 13/14** mới | **PG 17/18** (verify) | **13 EOL**, **14 EOL 11/2026** |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify):**
> - **Managed Postgres** (**RDS**/**Aurora**/**Cloud SQL**/**Neon**) lo hộ **autovacuum**/**backup**/**HA** (high availability).
> - **JSONB** dùng rộng cho **cấu hình**/**feature flag**/**payload sự kiện**.
> - **Cảnh báo wraparound** + **autovacuum tuning** là việc **"ops sống còn"** ở quy mô lớn.

---

## ⑤ Code + cấu hình

```sql
-- JSONB + GIN index + containment
CREATE TABLE event (id bigint, data jsonb);
CREATE INDEX idx_event_data ON event USING gin (data jsonb_path_ops);  -- GIN cho @>
SELECT * FROM event WHERE data @> '{"type":"click"}';                   -- containment: chứa key/value này

-- Theo dõi bloat / vacuum: n_dead_tup = số rác đang tích
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;        -- bảng nhiều dead tuple nhất lên đầu

-- Cập nhật statistics khi query CHẬM ĐỘT NGỘT (nghi phạm số 1 sau import lớn)
ANALYZE "order";

-- Kiểm tuổi XID (cảnh báo wraparound) — verify cú pháp
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;  -- age càng cao càng nguy
```

> 🚨 **AI hay sai ở đây:** AI hay khuyên **"tăng max_connections"** để fix lỗi connection — **sai hướng**; đúng là **thêm pooler** (**Bài 11**).

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **dead tuple**/**bloat** · vì sao **VACUUM** cần thiết · **wraparound** · **stats → plan** · **connection = process**.

> 📌 **THUỘC** (nhớ chính xác):
> **jsonb vs json** · **GIN cho jsonb** (**`@>`**) · **VACUUM FULL khóa độc quyền** · **PG 17/18 ok, 14 EOL 11/2026**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Tạo **GIN index** cho **jsonb** · chẩn đoán **query chậm do stats cũ** (**ANALYZE**) · chọn version cho dự án mới.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**MVCC tạo rác (dead tuple) → VACUUM dọn**. **Connection là process** (đừng nâng **max_connections**). **Stats cũ làm planner mù → ANALYZE**."*

Cách dùng: ba phản xạ chẩn đoán — bảng phình? nghĩ **VACUUM/bloat**. Lỗi quá nhiều connection? nghĩ **pooler**, không nghĩ max_connections. Query bỗng chậm sau import? nghĩ **ANALYZE** trước tiên.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(TB)** | **jsonb** vs **json**? Khi nào dùng? | **nhị phân**/**index được**; phần **bán cấu trúc** |
| **(khá)** | **VACUUM** giải quyết gì, vì sao **MVCC** cần? | thu **dead tuples** |
| **(khó)** | **VACUUM** vs **VACUUM FULL**? | FULL **rewrite** + **khóa độc quyền** |
| **(khó)** | **XID wraparound** — vì sao Postgres ngừng nhận ghi? | **bảo vệ**, chuyển **read-only** |
| **(TB)** | **max_connections=1000** có scale? | không; mỗi connection = **process** → **pooler** |

> 🧩 **Thách đố tình huống (TL):**
> *Chọn version PG vào 6/2026?*
>
> Trả lời chuẩn: **17/18**, **tránh 14/13**, **không beta**. (**verify** postgresql.org)

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Bảng `audit_log` ghi/xóa nhiều, bị phình

Chẩn đoán + xử lý **không downtime**.

> ✅ **Tiêu chí Đạt:**
> - Xem **`n_dead_tup`** (trong `pg_stat_user_tables`) để xác nhận **bloat**.
> - **Tuning autovacuum** cho bảng này.
> - Cân nhắc **partition** + **DROP partition cũ** (**Bài 13**) thay vì **DELETE** (DELETE tạo thêm dead tuple).
> - ❌ Trượt nếu đề xuất **VACUUM FULL** giờ cao điểm (gây downtime).

### BT2 — Query nhanh bỗng chậm sau khi import 10tr dòng

Nghi phạm đầu tiên?

> ✅ **Tiêu chí Đạt:**
> - **stats cũ** → chạy **ANALYZE**.
> - Kiểm **EXPLAIN** (Bài 10) xem có **seq scan** thay vì **index** không.
> - ❌ Trượt nếu nhảy ngay vào "thêm index" mà chưa nghi stats.

---

## ⑩ Đọc thêm

- **PostgreSQL — Routine Vacuuming & Wraparound** — `postgresql.org/docs/current/routine-vacuuming.html`
- **PostgreSQL — JSON Functions, GIN** — `.../functions-json.html`, `.../gin.html`
- **Versioning policy** — `postgresql.org/support/versioning/`

> 🎯 **Một câu mang về:** Postgres mạnh nhưng có "tính khí": MVCC sinh rác cần VACUUM, connection là process đắt đỏ, stats là đôi mắt của planner. Hiểu ba điều này, bạn tránh được hầu hết sự cố production của Postgres.
