# 🎯 BÀI 12 — Replication & lag

**Phủ:** E-REP-001 → 007

> 💡 Đây là **bước scale đầu tiên** (scale đọc + HA), *trước* khi tới sharding. Trái tim của bài: hiểu **lag** (độ trễ giữa các bản sao) và cái bẫy **read-after-write** — đọc ngay sau khi ghi mà không thấy data mình vừa ghi.

---

## ① Mục tiêu & vị trí

Bài này về **replication** — **bước scale đầu tiên** cho **đọc + HA** (high availability), đứng *trước* **sharding**.

Hai thứ phải nắm: **lag** và bẫy **read-after-write**.

Vị trí: là **nền cho Bài 13** (sharding) và **Bài 14** (consistency phân tán).

> 🎯 **Chốt định vị:** **Replication giải bài toán ĐỌC và HA — KHÔNG giải bài toán GHI.** Hiểu sai điều này là nguồn của vô số quyết định kiến trúc sai.

---

## ② Giảng cơ bản → nâng cao

### (a) Replication là gì  `E-REP-001`

> **Ẩn dụ:** **Replication** giống *photocopy sổ cái ra nhiều bản* đặt ở nhiều phòng. Nhiều người đọc cùng lúc ở các phòng khác nhau (scale đọc); một bản hỏng thì còn bản khác (HA). Nhưng **mọi thay đổi vẫn phải ghi vào sổ gốc** rồi mới chép sang — bản sao không tự ghi được.

**Replication** = **copy data sang node khác** để:
- **scale read** (nhiều replica chia tải đọc),
- **HA/failover** (master chết thì replica thay),
- **tách tải backup/analytics** (chạy báo cáo nặng trên replica, không đụng master).

> 🚨 **Lưu ý cốt tử:** **replication KHÔNG giải bài toán scale ghi** — **write vẫn về master** (một điểm ghi duy nhất). Muốn scale ghi phải đợi **sharding** (Bài 13).

---

### (b) Sync vs async  `E-REP-002`

> **Ví dụ trực giác:** **synchronous** = gửi thư xong *đứng chờ người nhận xác nhận* mới đi tiếp (chắc chắn nhận được, nhưng chậm). **asynchronous** = gửi thư rồi *đi luôn* không chờ (nhanh, nhưng lỡ thư thất lạc thì không biết).

| Tiêu chí | **synchronous** | **asynchronous** |
|----------|-----------------|------------------|
| Cách hoạt động | **chờ replica xác nhận** rồi mới báo commit | báo commit ngay, **không chờ** |
| An toàn data | ✅ **không mất data** | ❌ **có thể mất data khi failover** |
| Tốc độ | ❌ chậm hơn | ✅ nhanh |
| Rủi ro | ❌ **block khi replica chết** | data chưa kịp sang replica |

```text
ASYNC mất data khi failover:
  T1: client ghi master → master báo "ok" ngay (chưa chép sang replica)
  T2: 💥 master chết TRƯỚC khi data kịp sang replica
       ↓
  T3: promote replica làm master mới → replica KHÔNG có data ở T1
       ❌ data vừa ghi đã biến mất
  ✅ sync: T1 đợi replica xác nhận mới báo ok → không mất, nhưng chậm hơn
```

> 🎯 **Chốt:** **synchronous** đổi *tốc độ* lấy *an toàn*; **asynchronous** đổi *an toàn* lấy *tốc độ*. Data sống còn → cân nhắc **sync**.

---

### (c) Lag & read-after-write  `E-REP-003`

> **Ẩn dụ:** Bản photocopy luôn ra *sau* bản gốc một nhịp. Bạn vừa viết vào sổ gốc, chạy sang phòng bên đọc bản copy — bản copy chưa kịp cập nhật → bạn không thấy chữ mình vừa viết. Đó là **lag** gây **read-after-write fail**.

**replica chậm hơn master** (**lag**) → đọc replica **ngay sau khi ghi master** → **không thấy data vừa ghi** (**stale read**, **"read-your-own-write" fail**).

```text
READ-AFTER-WRITE fail do lag:
  T1: client GHI master: profile.name = "An"   ✅ master đã có
       ↓ (replica còn đang chép, chưa kịp)
  T2: client ĐỌC từ replica: profile.name → "Bình" (giá trị CŨ)   ❌ stale
       → user vừa sửa tên mà load lại thấy tên cũ → "app bị lỗi?!"
```

#### 🔍 Xử lý read-after-write  `E-REP-004`

| Cách | Làm gì | Khi nào |
|------|--------|---------|
| **route read về master** | đọc quan trọng đẩy thẳng về master | read cần fresh tuyệt đối |
| **sticky theo session** | trong một phiên, đọc cùng nguồn vừa ghi | đảm bảo user thấy chính data mình ghi |
| **chờ LSN/replay position** | đợi replica replay tới đúng vị trí đã ghi | cần fresh nhưng vẫn muốn dùng replica |
| **đọc từ cache** | lấy data vừa ghi từ cache | có sẵn cache layer |

> 🎯 **Tùy yêu cầu nhất quán TỪNG read** — không phải mọi read đều cần fresh. Báo cáo/danh sách chấp nhận hơi cũ → đọc replica thoải mái.

---

### (d) Mép giới hạn

#### 🚨 Failover & split-brain  `E-REP-005`

> **Ẩn dụ:** **split-brain** = hai người *cùng tưởng mình là sếp* và cùng ra lệnh trái ngược → hỗn loạn. Trong DB: **hai node cùng tưởng mình là master** → cùng nhận ghi → data ghi đôi, mâu thuẫn.

**Failover:** master chết → **promote một replica** lên làm master.

**split-brain** = **hai node cùng tưởng mình là master** → cần **fencing**/**quorum** để tránh **ghi đôi**.

> ⚠️ **fencing** = "cách ly" node cũ để nó không ghi nữa; **quorum** = cần đa số node đồng ý mới được làm master → tránh hai master cùng tồn tại.

#### 🔍 Physical vs logical  `E-REP-006`

| Tiêu chí | **physical** (streaming WAL) | **logical** |
|----------|------------------------------|-------------|
| Sao chép gì | **bản sao byte y hệt** | theo **row/table** |
| Version | **cùng version** | **cross-version** (khác version được) |
| Chọn lọc | toàn bộ | **chọn lọc** (chọn bảng) |
| Dùng cho | **HA**/**read replica** | nền cho **CDC** + **nâng cấp zero-downtime** |

> 🔎 **physical** dựa trên **streaming WAL** (chính cuốn log ở Bài 7) — sao chép từng byte nên buộc cùng version. **logical** hiểu theo *row/table* nên chép được giữa hai version khác nhau → đây là chìa khóa cho **nâng cấp không downtime** và **CDC** (Change Data Capture — bắt thay đổi đẩy đi nơi khác).

---

## ③ ⚠️ Cũ → Mới (kiến thức bị thay thế)

| Quan niệm CŨ ❌ | Hiện nay ✅ | Vì sao |
|----------------|------------|--------|
| "Thêm **replica** = giảm tải mọi thứ" | Replica giảm tải **đọc**, **không giải write/consistency** `E-REP-007` | **Write vẫn về master**, có **lag** |
| Đọc replica vô tư sau khi ghi | **Route read-after-write về master** / **chờ LSN** | Tránh **stale read** |
| **Async** "cho nhanh" mặc định | Cân nhắc **sync** cho data sống còn | Async **mất data khi failover** |

---

## ④ Thực tế + bigtech

> 🔎 **Pattern ngành (verify):**
> - **Read replica** + **đọc-ghi tách** (**write→primary, read→replica**) là pattern chuẩn.
> - **Failover tự động** qua **Patroni**/**managed** (**RDS Multi-AZ**) + **quorum** chống **split-brain**.
> - **Logical replication** + **CDC** (**Debezium**) để đẩy data sang **search/warehouse** và **nâng cấp version không downtime**.

> 🧩 **Chẩn đoán** `E-REP-007` — *"thêm replica mà latency không cải thiện, có chỗ trả sai"*:
> 1. **Kiểm lag** (replica có theo kịp master không).
> 2. Xem **read nào bị route nhầm sang replica** (read cần fresh mà lại đọc replica).
> 3. **Đối chiếu consistency requirement từng read**.
> ⚠️ Nhớ: **replica không giải write** — nếu nghẽn ở ghi thì thêm replica vô ích.

---

## ⑤ Code + cấu hình

```sql
-- Theo dõi replication lag TRÊN replica (verify cú pháp theo version)
-- replay_lag = khoảng thời gian replica đang chậm hơn master
SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;

-- TRÊN primary: xem các replica & độ trễ gửi/ghi/flush/replay
SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;
```

```text
# Routing nguyên tắc:
#   - Ghi                              -> primary
#   - Đọc cần fresh (vừa ghi xong)     -> primary HOẶC chờ replay tới LSN đã ghi
#   - Đọc chấp nhận hơi cũ (báo cáo,   -> replica
#     danh sách)
```

> 🚨 **AI hay sai ở đây:** AI hay coi **"thêm replica = giải mọi vấn đề scale"** (quên replica không scale write), hoặc **route read-after-write sang replica** (gây stale read), hoặc mặc định **async** cho cả data sống còn (rủi ro mất data khi failover).

---

## ⑥ Keywords

> 🧠 **HIỂU** (nắm bản chất):
> **lag** · **read-your-own-write** · **split-brain**/**fencing** · **physical vs logical** · **async mất data**.

> 📌 **THUỘC** (nhớ chính xác):
> **replica scale đọc không scale ghi** · **sync an toàn-chậm** / **async nhanh-rủi ro** · **logical = cross-version/CDC**.

> 🛠️ **LÀM ĐƯỢC** (kỹ năng tay):
> Route read theo **consistency requirement** · đo **lag** · chọn **sync/async** theo data.

---

## ⑦ Mental model

> 🎯 **NGUYÊN TẮC VÀNG CỦA BÀI:**
> *"**Replica giải bài toán ĐỌC và HA, không giải GHI**. Có **lag** → **read-after-write phải về master**. **Async đổi an toàn lấy tốc độ**."*

Cách dùng: khi định "thêm replica", hỏi *"vấn đề là đọc hay ghi?"* — nếu nghẽn ghi thì replica vô dụng. Với mỗi read, hỏi *"cần fresh tuyệt đối không?"* — nếu có thì về master/chờ LSN.

---

## ⑧ Phỏng vấn & thách đố

| Mức | Câu hỏi | Trả lời cốt lõi |
|-----|---------|-----------------|
| **(dễ)** | **Replication** để làm gì? | **scale read** + **HA** |
| **(TB)** | **Sync vs async** trade-off? | an toàn-chậm vs nhanh-mất-data |
| **(khá)** | **Lag** gây vấn đề gì sau khi ghi? | **stale**/**read-your-own-write** fail |
| **(khá)** | Xử lý **read-after-write**? | **master**/**sticky**/**chờ LSN**/**cache** |
| **(TB)** | **Split-brain** là gì? | hai master → **fencing**/**quorum** |
| **(khó)** | **Physical vs logical**? | byte y hệt cùng version vs row chọn lọc cross-version/CDC |

> 🧩 **Thách đố tình huống:**
> *Thêm replica mà latency không cải thiện + trả sai data — chẩn đoán?*
>
> Trả lời chuẩn: kiểm **lag** + **route nhầm** + **consistency từng read**; nhớ **replica không giải write**.

---

## ⑨ Bài tập + tiêu chí tự chấm

### BT1 — User cập nhật profile rồi load lại thấy data cũ

Nguyên nhân + 2 cách sửa.

> ✅ **Tiêu chí Đạt:**
> - Nguyên nhân: **lag** + **đọc replica** ngay sau ghi (**read-after-write fail**).
> - Sửa: **route về master sau ghi** / **chờ LSN** / **sticky session**.
> - ❌ Trượt nếu đổ lỗi "cache" mà không thấy lag replica.

### BT2 — Nâng cấp Postgres 16→18 gần như không downtime

Dùng replication kiểu nào?

> ✅ **Tiêu chí Đạt:**
> - **logical replication** (vì **cross-version**) + **cutover** (chuyển traffic sang node mới).
> - Giải thích vì sao physical không được (buộc cùng version).
> - ❌ Trượt nếu chọn physical (không chép được giữa 2 version).

---

## ⑩ Đọc thêm

- **PostgreSQL — High Availability, Replication** — `postgresql.org/docs/current/high-availability.html`
- **Logical replication & CDC** (**Debezium** docs)

> 🎯 **Một câu mang về:** Replica là *bản sao đọc*, không phải *điểm ghi thứ hai*. Nó cứu bạn ở đọc và HA, nhưng luôn chậm hơn master một nhịp — và cái nhịp đó (lag) chính là nơi bug read-after-write ẩn nấp.
