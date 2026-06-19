# 🎯 BÀI 14 — NewSQL / Distributed SQL

**Phủ:** E-NEW-001 → 005

> 💡 Đây là **bài capstone** — tổng hợp **consistency** (Bài 7–8) + **replication** (Bài 12) + **sharding** (Bài 13). Mục tiêu không phải "biết tên các sản phẩm hot", mà hiểu **NewSQL hứa gì và đánh đổi gì** để *quyết định kiến trúc*, chứ không chạy theo hype.

---

## ① Mục tiêu & vị trí

Bài này là **capstone** của cả mục E — nơi mọi mảnh ghép trước đó hội tụ:
- **consistency** (**Bài 7–8**) — ACID, isolation,
- **replication** (**Bài 12**) — bản sao, lag,
- **sharding** (**Bài 13**) — chia data nhiều node.

> 🎯 **Chốt định vị:** Hiểu **NewSQL hứa gì và đánh đổi gì** để **quyết định kiến trúc**, không chạy theo hype. Câu hỏi của senior không phải "công nghệ nào mới nhất" mà "bài toán của tôi có *thật sự cần* nó không".

---

## ② Giảng cơ bản → nâng cao

### (a) NewSQL là gì  `E-NEW-001`

> **Ẩn dụ:** Trước đây bạn phải chọn: xe an toàn nhưng chậm (**SQL truyền thống** — **ACID** mạnh nhưng khó scale ngang) hoặc xe nhanh nhưng cắt bớt an toàn (**NoSQL** thuần — scale ngang nhưng hi sinh **transaction**/quan hệ). **NewSQL** hứa hẹn "xe vừa nhanh vừa an toàn" — nhưng (như mọi lời hứa) có cái giá ẩn.

**NewSQL** = **SQL + ACID + scale ngang phân tán** — **tránh phải hi sinh transaction/quan hệ** để đổi lấy scale như **NoSQL** thuần. Nó **tự động** lo **sharding** + **replication** + **distributed transaction**.

**Vì sao đây là điều hấp dẫn?** Vì ở Bài 13, shard tay rất đau (cross-shard, distributed tx thủ công). NewSQL hứa làm việc đó *tự động* dưới tầng hạ tầng.

---

### (b) Landscape  `E-NEW-002`

> 🔎 **Lưu ý quan trọng (verify 6/2026):** Đây là lĩnh vực **thay đổi rất nhanh** — bảng dưới đây phản ánh thời điểm biên soạn, **phải verify lại khi học**.

| Sản phẩm | Tương thích | Đặc trưng | Cơ chế |
|----------|-------------|-----------|--------|
| **CockroachDB** | **Postgres-wire-compat** | **multi-region OLTP**, **Serializable mặc định** | **Raft** |
| **YugabyteDB** | **Postgres-compat** (lớp **YQL** + storage **DocDB**) | so trực tiếp với Cockroach | **Raft** |
| **TiDB** | **MySQL-compat** | **HTAP** (**TiKV** + **TiFlash**), tách compute/storage | **Raft** |
| **Google Spanner** | — | **fully-managed** trên **GCP** | **TrueTime** (atomic clock) + **Paxos** |
| **Amazon Aurora DSQL** | — | **serverless distributed SQL**, tích hợp AWS (mới nổi) | — |

---

### (c) Cơ chế consistency  `E-NEW-003`

> **Ví dụ trực giác:** Làm sao nhiều node cách xa nhau cùng "đồng ý" một sự thật? Như một hội đồng biểu quyết: chỉ cần **đa số** giơ tay đồng ý là quyết định có hiệu lực, kể cả vài thành viên vắng mặt (node chết). Đó là **consensus**.

Đạt **strong consistency** phân tán bằng **consensus**:
- **Raft** (**CockroachDB**/**TiDB**/**YugabyteDB**),
- **Paxos** + **TrueTime** (**Spanner**).

**Replicate qua consensus** → **nhất quán mạnh kể cả khi node chết** (khác **async replication** ở Bài 12 có thể mất data).

```text
Consensus đảm bảo nhất quán dù node chết:
  T1: ghi đến → đề xuất cho các node biểu quyết
       ↓
  T2: ĐA SỐ node xác nhận (vd 2/3) → ghi được coi là "committed"
       ↓
  T3: 💥 một node chết → vẫn còn đa số → data KHÔNG mất, vẫn nhất quán   ✅
       (khác async replication: master chết có thể mất data chưa kịp sang)
```

---

### (d) Đánh đổi  `E-NEW-004`

> 🚨 **Không có bữa trưa miễn phí.** NewSQL mạnh nhưng trả giá đắt.

| Đánh đổi | Vì sao |
|----------|--------|
| **latency cao hơn** cho transaction **cross-node** | phải **đồng thuận qua mạng/region** (mỗi tx chờ đa số node, có thể vượt biển) |
| **vận hành phức tạp** | nhiều node, nhiều region phải nuôi |
| **chi phí cao** | hạ tầng phân tán đắt |

> ⚠️ **Sự thật quan trọng:** **Nhiều cluster thực ra KHÔNG cần multi-region.** Mua NewSQL multi-region cho một app single-region là trả thuế latency/chi phí mà không nhận lại lợi ích.

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "Scale = **NoSQL**, mất **ACID**" | **NewSQL** giữ **SQL + ACID + scale ngang** | **consensus** + **distributed tx** |
| Tự **shard Postgres tay** cho multi-region | **Distributed SQL** tự lo **shard/replica/consistency** | Đỡ gánh **distributed tx** thủ công |
| Mặc định chọn **distributed SQL** "cho hiện đại" | **Single-region → managed Postgres** đơn giản/rẻ hơn nhiều `E-NEW-005` | Distributed có **thuế latency/chi phí** |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify landscape khi học):**
> - **Spanner** sinh ra từ nhu cầu Google (**global**, **strong consistency**).
> - **CockroachDB**/**YugabyteDB** là **"Spanner mã nguồn mở"** cho ai cần multi-region không khóa **GCP**.
> - **TiDB** mạnh ở **HTAP** + **MySQL-compat**.
> - **Aurora DSQL** cho team **AWS-first** muốn **serverless**.

### 🧩 TL quyết định  `E-NEW-005`

> ⭐ **Chọn NewSQL khi cần ĐỒNG THỜI:**
> 1. **strong consistency ĐA VÙNG** (multi-region),
> 2. **scale ghi vượt 1 node**,
> 3. **data residency** theo policy (data phải nằm đúng vùng pháp lý).
>
> **Nếu single-region** → **managed Postgres** (**RDS**/**Aurora**/**Cloud SQL**/**Neon**) **đơn giản/rẻ hơn**.
>
> 🚨 **KHÔNG chọn theo hype.**

---

## ⑤ Code + cấu hình

```sql
-- CockroachDB / YugabyteDB nói "wire-compatible Postgres":
-- phần lớn SQL chạy như Postgres, NHƯNG:
--   - một số stored procedure / pg_catalog / window edge case cần workaround
--   - thiết kế phải nghĩ theo "range/tablet" + data residency
-- Multi-region: pin data theo policy (vd dữ liệu EU ở region EU) để tuân GDPR
-- => verify cú pháp theo từng sản phẩm tại docs chính thức
```

> 💡 **Không có code "chuẩn chung"** — mỗi **distributed SQL** có **cú pháp region/placement riêng**.

> 🎯 **Nguyên tắc:** viết **SQL Postgres-chuẩn**, **tránh feature exotic** (đặc thù hiếm), **test kỹ compatibility**. Càng dùng tính năng "lạ" của Postgres, càng dễ vỡ khi chạy trên engine wire-compatible.

> 🚨 **AI hay sai ở đây:** AI hay đề xuất **distributed SQL "cho hiện đại/scalable"** mà không hỏi nhu cầu **multi-region/residency** thật; hoặc giả định **wire-compatible = 100% giống Postgres** (thực tế có edge case cần workaround).

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **consensus** (**Raft**/**Paxos**) · **TrueTime** · vì sao **cross-node tx chậm hơn** · **khi nào KHÔNG cần distributed SQL**.

> 📌 **THUỘC** (nhớ chính xác — **verify**):
> **Cockroach**/**Yugabyte** (Postgres-compat, **Raft**) · **TiDB** (MySQL, **HTAP**) · **Spanner** (**TrueTime**) · **Aurora DSQL** (serverless).

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Quyết định **NewSQL** vs **"managed Postgres + replica"** theo tiêu chí **multi-region/consistency/residency**.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**NewSQL = SQL + ACID + scale ngang nhờ consensus**. Đắt về **latency/vận hành**. Chỉ chọn khi **thật cần multi-region strong consistency** — **single-region thì managed Postgres**."*

Cách dùng: khi ai đó đề xuất NewSQL, chạy 3 câu hỏi của `E-NEW-005` — *multi-region? scale ghi vượt 1 node? data residency?*. Thiếu cả ba → managed Postgres thắng về đơn giản và chi phí.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(TB)** | **NewSQL** giải mâu thuẫn nào? | **SQL+ACID+scale**, không hi sinh tx như **NoSQL** |
| **(khá)** | Kể vài **distributed SQL** + sweet spot | **Cockroach**/**Yugabyte**/**TiDB**/**Spanner**/**Aurora DSQL** |
| **(khá)** | Đạt **strong consistency** phân tán bằng gì? | **Raft** / **Paxos + TrueTime** |
| **(khó)** | Đánh đổi so với Postgres đơn vùng? | **latency cross-node**, **vận hành**, **chi phí** |

> 🧩 **Thách đố tình huống (TL):**
> *Khi nào NewSQL thay "Postgres + replica + shard ở app"?*
>
> Trả lời chuẩn: khi cần **multi-region strong consistency** + **scale ghi >1 node** + **data residency**; **single-region thì không**.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — Startup single-region, 100k user, đề xuất chuyển CockroachDB "cho scalable"

Phản biện như TL.

> ✅ **Tiêu chí Đạt:**
> - Hỏi **nhu cầu multi-region/residency thật** trước khi đồng ý.
> - Nếu không có → **managed Postgres + replica** rẻ/đơn giản hơn nhiều.
> - ❌ Trượt nếu đồng ý chuyển chỉ vì "nghe scalable".

### BT2 — Hệ tài chính cần data EU ở EU, US ở US, vẫn 1 logic + strong consistency

Đề xuất.

> ✅ **Tiêu chí Đạt:**
> - **distributed SQL** có **geo-partitioning/placement policy** (**Cockroach**/**Yugabyte**/**Spanner**) — **verify**.
> - Giải thích vì sao đây đúng là ca cần NewSQL (đủ cả 3 tiêu chí `E-NEW-005`).
> - ❌ Trượt nếu đề xuất single Postgres (không thỏa data residency đa vùng).

---

## ⑩ Đọc thêm

- **CockroachDB / YugabyteDB / TiDB / Spanner docs** (chính thức)
- **Google Spanner paper** (**TrueTime**); **Kleppmann *DDIA* Ch.9** (Consistency & Consensus)

> 🎯 **Một câu mang về:** NewSQL không phải "Postgres phiên bản xịn hơn" — nó là một đánh đổi *latency và chi phí* để mua *strong consistency đa vùng*. Nếu bài toán của bạn ở một vùng, managed Postgres vẫn là câu trả lời đúng, đơn giản và rẻ hơn. Chọn theo *nhu cầu*, không theo *hype* — đó là tinh thần khép lại cả mục E.
