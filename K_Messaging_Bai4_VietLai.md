# 🎯 BÀI 4 — Kafka Internals II: replication, rebalance, retention, EOS, lag

**Phủ:** K-KAFKA-006 → 012

> 💡 **Mục tiêu một câu:** Bài 3 cho bạn "vật lý" của Kafka (partition/offset/group). Bài này cho bạn **độ bền** (message không bay khi server chết), **vận hành** (rebalance, lag), và sự thật phũ phàng về **exactly-once**.

---

## ① Mục tiêu & vị trí trong mạch

Bài này **hoàn thiện Kafka** bằng năm mảnh ghép cuối:
- **độ bền** — **replication / ISR**,
- **vận hành nhóm** — **rebalance**,
- **vòng đời dữ liệu** — **retention** vs **compaction**,
- **exactly-once** — **EOS**,
- **hai lỗi vận hành sống còn** — **hot partition** và **consumer lag**.

> 🎯 **Chốt vị trí:** sau bài này bạn đủ nền cho **Bài 5** (delivery semantics) và **Bài 6** (**Outbox** — vốn dựa trên **producer ack / EOS** của Kafka).

---

## ② Giảng cơ bản → nâng cao

### (a) Replication & ISR 📘 K-KAFKA-007, 008

> 🛡️ **Ẩn dụ:** mỗi **partition** giống một tài liệu quan trọng được **photo ra nhiều bản** cất ở nhiều tủ khác nhau. Một bản là **bản gốc đang dùng (leader)**, các bản còn lại là **bản sao theo sát (follower)**. Cháy một tủ → vẫn còn bản khác.

Cơ chế cụ thể: mỗi **partition** có **1 leader + (N−1) follower** (**N = replication factor**, ký hiệu **RF**). **ISR** (**in-sync replicas**) = *tập replica đang theo kịp leader*.

| Cấu hình | Ý nghĩa | Hệ quả |
|---|---|---|
| **`acks=all`** (producer) | chờ **mọi ISR** ghi xong mới coi message "đã nhận" | bền hơn, nhưng chậm hơn |
| **`min.insync.replicas`** | **ngưỡng ISR tối thiểu** để chấp nhận ghi | ISR tụt dưới ngưỡng → producer **bị từ chối** (chặn mất dữ liệu khi leader chết) |

> ⚖️ **Đánh đổi cốt lõi:** **độ bền ↑ ↔ latency/availability ↓** (chờ nhiều replica hơn thì chậm hơn và dễ bị từ chối ghi hơn).

> 🎯 **Cấu hình kinh điển "bền":** **`RF=3`** + **`min.insync.replicas=2`** + **`acks=all`**. (🔎 *verify*.)
> *Vì sao bộ ba này?* RF=3 cho 3 bản; yêu cầu ít nhất 2 bản ghi xong (min.insync=2) → mất 1 node vẫn còn đủ bản đồng bộ để không mất dữ liệu, mà không cần chờ cả 3 (đỡ kẹt khi 1 node chậm).

---

#### Leader chết 📘 K-KAFKA-008

```
TIMELINE — leader của partition chết:
  T1  leader (bản gốc) đang phục vụ đọc/ghi
  T2  💥 leader chết
  T3  controller bầu MỘT follower trong ISR lên làm leader mới  ✅
  T4  producer/consumer reconnect tới leader mới
  T5  tiếp tục hoạt động  ✅ (mất dữ liệu = 0 nếu cấu hình bền đúng)
```

> 💡 **Sự thật ngược trực giác:** **Kafka cluster vốn HA** → nó là *"available, sometimes consistent"*. Điểm chết *đáng lo hơn* thực ra **KHÔNG phải "Kafka chết"**, mà là **consumer chết** → khi sống dậy xử lý lại từ **last committed offset** (**at-least-once**, gặp lại ở Bài 3 & 5).

---

### (b) Rebalance 📘 K-KAFKA-006

> 👥 **Ví dụ trực giác:** một đội chia nhau quầy. Có người **nghỉ/vào/ngất xỉu**, hoặc số quầy thay đổi → cả đội phải **chia lại quầy**. Trong lúc chia lại, dễ có khoảng *"khoan, ai làm quầy nào?"* — công việc đình trệ.

**Rebalance** kích hoạt khi consumer **join / leave / chết** hoặc đổi **#partition** → group **reassign** partition cho các consumer.

| Kiểu rebalance | Cách làm | Cái đau |
|---|---|---|
| **Eager** (stop-the-world, kiểu cũ) | **revoke TẤT CẢ** partition rồi chia lại từ đầu | consumer **ngừng xử lý** → **lag tăng**; có thể **xử lý lại / đảo thứ tự** nếu commit chưa kịp |
| **Cooperative / incremental** (mới) | chỉ **chuyển phần partition cần thiết**, không revoke tất cả | giảm đau **đáng kể** |

> 🎯 **Cập nhật quan trọng:** **cooperative/incremental rebalance** (**KIP-429** và **KIP-848 GA** ở **Kafka 4.0**) chỉ chuyển phần cần thiết. Consumer **opt-in** bằng **`group.protocol=consumer`**. (🔎 *verify*.)
> ⚠️ Câu cũ *"rebalance = stop-the-world"* nay **phải kèm** *"đang dịch sang incremental"* — đừng trả lời phỏng vấn theo sách cũ.

---

### (c) Retention vs Compaction 📘 K-KAFKA-009

> 🗂️ **Ẩn dụ:** **retention** = *cuốn nhật ký* — ghi hết, cũ quá thì xé bỏ trang cũ. **Compaction** = *bảng danh bạ* — mỗi tên (key) chỉ giữ **số điện thoại mới nhất**, số cũ ghi đè đi.

| Tiêu chí | **Retention** | **Compaction** |
|---|---|---|
| Quy tắc xoá | theo **thời gian / dung lượng** → xoá message cũ quá hạn | giữ **bản mới nhất theo key**, xoá bản cũ cùng key |
| Topic trở thành | một **event-log** đầy đủ lịch sử | một **"snapshot trạng thái hiện tại per key"** |
| Dùng cho | log sự kiện thông thường | **changelog / config / materialize state** |
| Ví dụ | mọi giao dịch trong 7 ngày | trạng thái **mới nhất** của mỗi **`userId`** |

> 🎯 **Chọn theo câu hỏi:** *cần **lịch sử đầy đủ** (retention) hay chỉ **state hiện tại** (compaction)?* (🔎 *verify `cleanup.policy`*.)

---

### (d) Exactly-once (EOS) 📘 K-KAFKA-010

> 🧠 **Mental model:** **EOS** (**exactly-once semantics**) ghép từ **hai mảnh**, và *chỉ thần kỳ trong biên giới Kafka*.

**Mảnh 1 — idempotent producer** (`enable.idempotence=true`, mặc định bật ở đời mới):
- **dedup retry phía producer** bằng **producer id + sequence number** → producer retry mà **không ghi trùng**.

**Mảnh 2 — Transactions:**
- cho phép **read-process-write nguyên tử** *trong* Kafka (ví dụ **Kafka Streams**): hoặc **cả batch commit**, hoặc **không gì cả**.

> 🚨 **Giới hạn cốt lõi (phải khắc cốt):** *EOS chỉ "exactly-once" **trong biên Kafka**.* Khi **side-effect** ra **DB ngoài / API ngoài**, **KHÔNG có phép màu** → vẫn cần **idempotent upsert / Outbox** → chỉ đạt **"effectively-once"** (**Bài 5, 6**).

```
EOS trong vs ngoài biên Kafka:
  ┌─────────── BIÊN KAFKA (transaction lo được) ───────────┐
  │  topic A ──read──► xử lý ──write──► topic B  ✅ atomic   │
  └────────────────────────────────────────────────────────┘
                          │
                          ▼ write ra DB NGOÀI
                  ┌──────────────┐
                  │  Postgres    │  ❌ NẰM NGOÀI transaction Kafka
                  └──────────────┘     → crash/redeliver có thể ghi 2 lần
                  → BẮT BUỘC idempotent upsert / Outbox
```

> 🔎 **KIP-890** (4.x) củng cố **transaction protocol**. (🔎 *verify; cross-ref **I-IDEM***.)

---

### (e) Hai lỗi vận hành

#### Hot partition / data skew 📘 K-KAFKA-011

> 🔥 **Ví dụ trực giác:** 10 quầy nhưng 90% khách dồn vào *một* quầy → quầy đó **cháy việc**, 9 quầy kia **rảnh rang**.

**Hot partition** xảy ra khi **key phân bố lệch** (**data skew**) → 1 partition nhận phần lớn traffic → 1 consumer **quá tải**, số còn lại **rảnh**.

> ⚠️ **`key = tenantId` rất nguy hiểm:** một tenant lớn (ví dụ khách enterprise chiếm 70% đơn) sẽ làm **nóng** đúng partition của nó.

**Cách xử lý:**
- chọn **key phân tán hơn** — **composite key** `tenantId:entityId`,
- **tách riêng** tenant lớn (topic/partition riêng),
- hoặc **đánh đổi ordering** (bỏ key để trải đều).

> 🎯 Nhận diện đây là **trade-off ordering ↔ cân tải**: giữ key thì giữ thứ tự nhưng có thể lệch tải; rải key thì cân tải nhưng mất ordering per-entity.

---

#### Consumer lag 📘 K-KAFKA-012

> ❤️ **Ẩn dụ:** **lag** là *nhịp tim* của consumer. Nhịp tăng đều và không hạ = consumer đang đuối, sắp "ngộp".

> 🎯 **Công thức:** **`lag = latest offset − committed offset`** = *số message chưa xử lý*.

- **lag tăng dần** = consumer **chậm hơn** producer (sản xuất nhanh hơn tiêu thụ).

**Cách xử lý (theo thứ tự cân nhắc):**
- **tăng consumer** (nhưng **≤ #partition**, nhớ quy tắc Bài 3),
- **tăng partition**,
- **tối ưu xử lý** / **batch**,
- **đẩy side-effect nặng ra async**.

> 🚨 **Lag là metric sống còn** của lớp messaging — **phải alert**. (🔎 *verify công cụ đo: `kafka-consumer-groups.sh`, **Burrow**, hoặc **exporter Prometheus***.)

---

## ③ ⚠️ Cũ → Mới

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **"Rebalance luôn stop-the-world"** | **Cooperative/incremental** (**KIP-848 GA 4.0**, `group.protocol=consumer`) | Giảm downtime rebalance (🔎 verify) |
| **"Exactly-once là bất khả / hoặc có sẵn mọi nơi"** | **EOS** có *trong* Kafka (**idempotent producer** + **transactions**); ra ngoài → **effectively-once** | Phân biệt **phạm vi** (🔎 verify) |
| **`enable.idempotence` phải tự bật** | Đời mới **mặc định bật** idempotent producer | Vẫn nên **verify** theo client/đời |
| **ZooKeeper điều phối controller/ISR** | **KRaft controller quorum** | 4.x **KRaft-only** (🔎 verify) |

---

## ④ Áp dụng thực tế + bigtech

| Topic | Cấu hình | Vì sao |
|---|---|---|
| **changelog** "trạng thái tài khoản mới nhất" | **compaction** | không cần lịch sử, cần **state hiện tại** để service khởi động đọc lại |
| **giao dịch tài chính** | **retention dài** + **`RF=3`** + **`acks=all`** | cần **lịch sử** + **bền tuyệt đối** |

**Bigtech:** **Kafka Streams** dùng **compacted topic** làm **state store / changelog**; **EOS** dùng trong **pipeline streaming nội bộ**. (pattern; 🔎 *verify*.)

---

## ⑤ Code thực hành + cấu hình

**Producer bền + idempotent** (KafkaJS / config-style):

```javascript
// producer bền: idempotent + acks=all
const producer = kafka.producer({
  idempotent: true,        // dedup retry phía producer (producerId + sequence) // verify tên option theo client
  maxInFlightRequests: 5,  // idempotent CHO PHÉP >1 in-flight mà vẫn giữ ordering (không cần ép về 1)
  // acks=all được suy ra khi idempotent=true (verify)
});
```

```bash
# topic config (verify theo đời broker)

# Durable log (bền tuyệt đối):
replication.factor=3
min.insync.replicas=2
# acks=all đặt phía producer

# Compacted state topic (chỉ giữ bản mới nhất per key):
cleanup.policy=compact

# Retention log thường (giữ 7 ngày rồi xoá):
retention.ms=604800000   # 7 ngày (= 7*24*3600*1000 ms)
```

```bash
# Đo lag (verify công cụ/cờ theo đời)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group inventory-svc   # cột LAG = latest - current offset
```

> 🎯 **Đọc cột `LAG`:** nếu nó *tăng dần qua các lần chạy* → consumer đang tụt lại sau producer → cần can thiệp (xem K-KAFKA-012).

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **ISR** + **acks** / **min.insync** (độ bền ↔ latency).
- **rebalance** (**eager** vs **cooperative**).
- **retention** vs **compaction**.
- **EOS phạm vi** (trong vs ngoài biên Kafka).
- **hot partition**; **lag**.

**📌 Cần THUỘC:**
- **`RF=3`** + **`min.insync.replicas=2`** + **`acks=all`**.
- **`cleanup.policy=compact`**.
- **`lag = latest − committed`**.
- **idempotent producer** = **producerId + sequence**.

**🛠️ Cần LÀM ĐƯỢC:**
- Cấu hình **topic bền**.
- Chọn **compaction** khi cần **state**.
- Chẩn đoán **lag** tăng.
- Nhận diện & xử lý **hot partition**.

---

## ⑦ Mental model

> 🧠 **"Bền = RF3 + min.insync 2 + acks=all** (**ISR** là người gác).
> **Rebalance** đang chuyển từ **stop-the-world** sang **incremental**.
> **EOS chỉ trong Kafka** — ra ngoài phải **tự idempotent**.
> **Lag là nhịp tim của consumer."**

---

## ⑧ Phỏng vấn & thách đố

**(TB)** **ISR** là gì? **`acks=all`** + **`min.insync.replicas`** bảo vệ điều gì, đổi gì?
→ ISR = replica theo kịp leader; bộ này chặn mất dữ liệu khi leader chết, đổi lại latency/availability.

**(TB)** **Rebalance** kích hoạt khi nào, vì sao nguy hiểm, cải tiến gì? (nhắc **KIP-848**)
→ khi consumer join/leave/chết hoặc đổi #partition; nguy hiểm vì stop-the-world (eager); cải tiến = cooperative/incremental.

**(khó)** **Retention** vs **compaction** — khi nào compaction?
→ khi chỉ cần **state hiện tại per key** (changelog/state store), không cần lịch sử đầy đủ.

**(rất khó)** **Exactly-once** Kafka đạt bằng gì, giới hạn ở đâu khi ghi DB ngoài?
→ idempotent producer + transactions (trong Kafka); ghi DB ngoài nằm ngoài transaction → cần idempotent upsert/Outbox (effectively-once).

**(khó)** **Hot partition** do `key=tenantId` — chẩn đoán & xử lý?
→ 1 partition nóng do tenant lớn; xử lý bằng composite key / tách tenant / đánh đổi ordering.

**(khó)** **Lag** là gì, tăng vô hạn thì làm gì?
→ latest − committed; tăng consumer (≤#partition)/tăng partition/tối ưu/batch/async hoá side-effect nặng.

> 🧩 **Thách đố:** *"Dev bật **Kafka transactions** và tuyên bố 'giờ ghi DB cũng **exactly-once**, bỏ idempotency consumer được rồi'. Sai chỗ nào?"*
> → **EOS** chỉ nguyên tử *trong* Kafka (**read-process-write** giữa các topic). **Ghi ra DB ngoài** nằm **ngoài** transaction Kafka → vẫn có thể chạy **2 lần** khi **redeliver/crash** → **bắt buộc** giữ **idempotent upsert / dedup** (**effectively-once**). *Transactions ≠ miễn idempotency ở biên ngoài.*

---

## ⑨ Bài tập + tiêu chí

**Đề:** Thiết kế config cho topic **`ledger-entries`** (không được mất, cần lịch sử) và topic **`account-balance-latest`** (chỉ cần số dư hiện tại). Nêu **RF, acks, cleanup.policy, lý do**.

> ✅ **Đạt khi:**
> - **`ledger-entries`**: **`RF=3`**, **`min.insync=2`**, **`acks=all`**, **retention dài** / `cleanup.policy=delete`.
> - **`account-balance-latest`**: **`cleanup.policy=compact`**, **`key=accountId`**.
> - Giải thích đúng *"**lịch sử** vs **state**"*.

---

## ⑩ Đọc thêm

- **`kafka.apache.org`**: **Replication**, **Transactions**, **Log Compaction**.
- **KIP-848**, **KIP-890** (đọc bản cập nhật **4.x**). (🔎 *verify*.)

> 🎯 **Một câu mang về:** *Kafka cho bạn **độ bền** và **exactly-once trong nhà nó** — bước ra cửa (DB/API ngoài) thì bạn phải tự cầm theo **idempotency**.*
