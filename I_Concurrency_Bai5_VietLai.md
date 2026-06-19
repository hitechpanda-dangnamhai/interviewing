# 🎯 BÀI 5 — CAP & PACELC

**Phủ:** I-CAP-001 → I-CAP-011 · **Định luật đánh đổi của hệ phân tán.**

> 💡 **Cách đọc bài này.** Bài này *không có nhiều code* — nó là **khung tư duy kiến trúc**. Hai câu cần khắc cốt: ***(1) "Consistency" trong CAP KHÔNG phải chữ "C" trong ACID; (2) đánh đổi consistency là per-operation, không phải nhãn dán cho cả hệ.*** Người mới hay học CAP thành khẩu hiệu; người giỏi dùng nó để *định lượng* đánh đổi cho từng đường dữ liệu.

---

## ① Mục tiêu & vị trí trong mạch

Học xong bài này bạn sẽ làm được 5 việc:

1. **Phát biểu CAP đúng** — và *gỡ hiểu lầm "chọn 2 trong 3"*.
2. **Biết "Consistency" trong CAP là linearizability** (khác **ACID-C**).
3. **Phân biệt CP vs AP** qua *hành vi khi partition*.
4. **Hiểu PACELC thực tế hơn CAP ở chỗ nào.**
5. **Biết khi nào đừng mang CAP vào câu chuyện** (chống over-apply, học vẹt).

> 🎯 **Vị trí trong mạch học.** Bài này là **khung tư duy** cho:
> - **Bài 6** — đào sâu chữ **"C"** (các mức consistency).
> - **Bài 7** — **consensus** thực thi **quorum** (cơ chế *làm ra* được tính nhất quán mạnh).

**Ẩn dụ định vị:** Bài 1–4 dạy bạn *chống race trong tay mình*. Bài 5 lùi ra xa, nhìn cả hệ phân tán như một *bản đồ thời tiết*: khi "bão mạng" (partition) ập tới, bạn buộc phải chọn *cứu cái nào trước*. CAP/PACELC là la bàn cho lựa chọn đó.

---

## ② Giảng cơ bản → nâng cao

### (a) Trực giác + gỡ hiểu lầm — **I-CAP-001**

> **Ví dụ trực giác:** Hai chi nhánh ngân hàng giữ *hai bản sao* cùng một sổ tài khoản. Một hôm **đường truyền giữa hai chi nhánh đứt** (**network partition**). Khách tới chi nhánh A rút tiền. A có hai lựa chọn:
> - **Từ chối phục vụ** ("hệ thống đang bảo trì") → giữ cho hai sổ *không lệch nhau* → ưu **Consistency**.
> - **Vẫn cho rút** dựa trên bản sao của riêng A → *phục vụ được* nhưng có thể *lệch* với chi nhánh B → ưu **Availability**.
>
> Không thể vừa *chắc chắn không lệch* vừa *vẫn phục vụ* khi đường đứt. Phải chọn **một**.

**CAP** phát biểu: khi có **network partition (P)** (mạng bị chia cắt), hệ phân tán **không thể vừa Consistent vừa Available** — phải **chọn một**.

> 🚨 **Gỡ hiểu lầm "chọn 2 trong 3".** Câu này **sai**, vì trong hệ phân tán thật, **P là BẮT BUỘC** — *mạng sẽ chia, sớm muộn*. Bạn không "chọn bỏ P" được. Nên thực chất chỉ **chọn C hay A KHI partition xảy ra**.

> 💡 **Điểm tinh tế:** *Lúc mạng lành* (>99% thời gian), bạn **có thể có CẢ C lẫn A**. CAP chỉ ép chọn *trong khoảnh khắc partition* — đó là lý do PACELC ra đời để nói về phần "mạng lành" còn lại.

---

### (b) Cơ chế & bẫy thuật ngữ

#### 🔍 "Consistency" trong CAP = linearizability — **I-CAP-002**

**Linearizability** (khả tuyến tính hóa) = **read luôn thấy write mới nhất**, *như thể chỉ có một bản sao duy nhất* trên đời. Dù thực tế dữ liệu nằm trên nhiều replica, hệ *hành xử* như một bản duy nhất luôn cập nhật.

> 🔎 Đây **KHÁC** chữ **"Consistency" trong ACID** — và đó là bẫy thuật ngữ tiếp theo.

---

#### 🚨 CAP-C vs ACID-C — **I-CAP-003** (bẫy kinh điển)

> **Ví dụ trực giác:**
> - **CAP-C** giống *"hai bản sao sổ phải khớp nhau"* (các bản đồng ý).
> - **ACID-C** giống *"mỗi sổ phải tuân luật kế toán"* (nợ = có, không âm quỹ).
> Hai chuyện *hoàn toàn khác nhau*: sổ có thể *đúng luật kế toán* mà vẫn *lệch với sổ bên kia*.

| Tiêu chí | **CAP-C** | **ACID-C** |
|---|---|---|
| Nghĩa | **replica agreement** (các bản sao đồng ý) | transaction giữ **invariant / constraint** |
| Ví dụ | Read ở mọi replica thấy giá trị mới nhất | **FK**, **CHECK**, business rule (số dư ≥ 0) |
| Phạm vi | Giữa các bản sao (phân tán) | Trong một transaction |

> 🎯 **Chốt:** Hai khái niệm **trực giao** (orthogonal — *độc lập nhau*). Một hệ có thể **ACID local** nhưng **eventually-consistent global**. **Đừng để interviewer thấy bạn lẫn hai chữ C này** — đó là dấu hiệu chưa nắm chắc.

---

#### 🔍 CP vs AP khi partition — **I-CAP-004**

**Timeline — cùng một partition, hai cách hành xử:**

```
Mạng đứt giữa replica-1 và replica-2. Client hỏi replica-1:

CP (ưu Consistency):
   client → replica-1: "đọc/ghi X"
   replica-1: không chắc mình có dữ liệu mới nhất (mất liên lạc với replica-2)
            → TỪ CHỐI / CHẶN, trả lỗi  ✅ không bao giờ trả data sai

AP (ưu Availability):
   client → replica-1: "đọc/ghi X"
   replica-1: cứ phục vụ bằng bản sao của mình
            → TRẢ data (có thể CŨ / có thể tạo ghi conflict)  ✅ luôn sống, ❌ có thể stale
```

| Phân loại | Hành vi khi partition | DB ví dụ (verify) |
|---|---|---|
| **CP** (ưu C) | **Từ chối / chặn** request (trả lỗi) để giữ nhất quán | **etcd**, **ZooKeeper**, **MongoDB** (majority write concern), **HBase**, **CockroachDB**, **Spanner** |
| **AP** (ưu A) | **Vẫn phục vụ** nhưng có thể trả **data cũ** / ghi **conflict** | **Cassandra** (consistency level thấp), **DynamoDB** (eventually-consistent reads), **CouchDB**, **DNS** |

> ⚠️ **Lưu ý — nhiều DB tunable** (điều chỉnh được): **MongoDB** có thể nghiêng **CP/AP** theo **write concern**; **Cassandra** chọn **per query**. **(verify phân loại — đổi theo cấu hình.)**

---

### (c) PACELC — thực tế hơn — **I-CAP-005** (⭐⭐⭐⭐⭐)

> **Ví dụ trực giác:** CAP chỉ nói chuyện *"khi có bão"*. Nhưng 99% thời gian *trời quang*, mà bạn **vẫn** phải chọn: đi đường tắt (**nhanh** nhưng có thể *thông tin hơi cũ*) hay đi vòng hỏi cho chắc (**đúng** nhưng *chậm*). PACELC nói cả về ngày quang lẫn ngày bão.

**PACELC** = **if Partition → A vs C; Else (không partition) → Latency vs C.**

> 💡 **Điểm sáng:** trade-off **latency ↔ consistency** tồn tại trên **MỖI request bình thường**, *không chỉ lúc sự cố*. **Strong consistency** cần **round-trip tới majority/leader** → **chậm hơn ngay cả khi mạng lành**. **CAP bỏ qua chuyện này; PACELC vá đúng chỗ đó.**

| | **Partition (P)** xảy ra | **Else (E)** — mạng lành |
|---|---|---|
| Phải chọn giữa | **A vs C** (sống vs đúng) | **L vs C** (nhanh vs đúng) |
| Tần suất | Hiếm (sự cố) | >99% thời gian |

#### 🔍 Phân loại PACELC — **I-CAP-006**

| Hệ | Phân loại PACELC | Nghĩa |
|---|---|---|
| **MongoDB** (majority) | ≈ **PC+EC** | Ưu **consistency** ở *cả hai vế* |
| **Cassandra** default **ONE** | ≈ **PA+EL** | Ưu **availability + latency** |
| **Cassandra** với **QUORUM** read+write | → **PC+EC** | Người dùng nâng lên nhất quán mạnh |

> 💡 Người dùng chọn **per-query** = **tunable consistency** (nhất quán điều chỉnh được). **(verify.)**

---

### (d) Sắc thái senior/TL

#### 🚨 "Chọn availability" chưa đủ — **I-CAP-007**

> ⚠️ **Cảnh báo phỏng vấn:** nói *"chúng tôi chọn AP"* là **khẩu hiệu**, chưa đủ. **Interviewer senior muốn CON SỐ:**
> - **staleness window** chấp nhận được *bao lâu*? (dữ liệu cũ tối đa mấy giây?)
> - **conflict rate** dự kiến *bao nhiêu*?
> - cách **resolve conflict**: **LWW** / **vector clock** / **CRDT**?

> 🎯 **Chốt:** **Trade-off cần ĐỊNH LƯỢNG, không khẩu hiệu.**

---

#### 🔍 Conflict resolution khi AP — **I-CAP-010**

**Bối cảnh:** lúc **partition**, *cả hai phía cùng ghi*. Khi mạng **heal** (lành lại), phải **merge** hai phiên bản. Các cách:

| Chiến lược | Cách làm | Đánh đổi |
|---|---|---|
| **Last-write-wins (LWW)** | Giữ bản có **timestamp** mới hơn | Đơn giản — nhưng **rủi ro mất ghi** (bản thua biến mất) |
| **Vector clocks / version vectors** | **Phát hiện concurrent** (hai ghi song song không nhân quả) | Phức tạp hơn, nhưng *biết* có conflict |
| **CRDT** | **Merge tự động đúng đắn** (cấu trúc dữ liệu tự hòa giải) | An toàn nhất cho merge, nhưng giới hạn kiểu dữ liệu |
| **App-level merge** | Để tầng ứng dụng quyết định | Linh hoạt nhất, tốn công code |

> 🎯 **Chốt:** Mỗi cách đánh đổi **đơn giản ↔ rủi ro mất dữ liệu**. **LWW** *âm thầm mất dữ liệu* — chỉ dùng khi mất một ghi *không sao*.

---

#### 🔍 Partition không nhị phân — **I-CAP-008**

> **Ví dụ trực giác:** A gọi điện cho B không bắt máy → A *tuyên bố* "B chết rồi". Nhưng B vẫn sống nhăn, chỉ là *điện thoại B đang bận / sóng yếu*. Giờ A và B **bất đồng** về việc "có đứt liên lạc hay không".

**Cơ chế:** node A coi **timeout** là *"B chết"*, nhưng B **vẫn chạy** → hệ **bất đồng về việc có partition hay không**. **CAP đơn giản hóa quá mức** thực tế (vốn mờ, không rạch ròi "có/không partition").

---

#### 🚨 CAP không áp cho single-node — **I-CAP-009**

> ⚠️ **Bẫy over-apply:** **CAP chỉ về phân tán CÓ REPLICA.** Một **Postgres đơn lẻ** *không "chọn C/A"* theo CAP. **Phần lớn app vừa & nhỏ KHÔNG cần lập luận CAP** — *mang CAP vào mọi câu là dấu hiệu học vẹt*.

---

#### 🔍 Consistency là per-operation, không per-system — **I-CAP-011** (⭐⭐⭐⭐⭐)

> 💡 Một hệ **vừa strong-consistent cho tài chính** **vừa eventually-consistent cho analytics** là **ĐÚNG và NÊN thế**.

> 🎯 **Chốt:** **Tune theo yêu cầu nghiệp vụ thật, KHÔNG gán một nhãn cho cả hệ.** Hỏi "hệ này CP hay AP?" thường là câu hỏi sai mức — phải hỏi *"đường dữ liệu NÀO?"*.

> 📘 **vs** ➕ — *docs gốc roadmap* chạm **CAP cơ bản**; **PACELC**, **CAP-C↔ACID-C**, **conflict resolution**, *"per-operation"* là phần ⬆️/➕.

---

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ / hiểu lầm | Thay bằng (đúng) | Vì sao |
|---|---|---|
| *"CAP = chọn 2 trong 3"* | Trong hệ phân tán **P bắt buộc** → **chọn C hay A khi partition** | Mạng **sẽ chia**, không bỏ P được |
| *"Consistency" CAP = "Consistency" ACID* | **CAP-C = linearizability / replica agreement**; **ACID-C = invariant** | Hai khái niệm **trực giao** |
| Chỉ dùng **CAP** để mô tả hệ | **PACELC** (thêm vế **Else: Latency vs C**) | Trade-off tồn tại **cả khi mạng lành** |
| Gán **một nhãn CP/AP** cho cả hệ | **Consistency per-operation** (tunable) | Mỗi path có nhu cầu khác nhau |
| *"Chúng tôi chọn AP"* (khẩu hiệu) | **Định lượng staleness window + conflict resolution** | Senior cần **con số** |
| **LWW** mặc định để resolve conflict | Cân nhắc **vector clock / CRDT** khi không được mất ghi | **LWW âm thầm mất dữ liệu** |
| Mang **CAP vào hệ single-node** | **CAP chỉ cho phân tán có replica** | **Over-apply = học vẹt** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case — hệ thương mại điện tử toàn cầu:**
- **Số dư ví & trạng thái thanh toán** → chọn **CP** (*thà báo lỗi còn hơn sai tiền*).
- **Feed / đếm like / gợi ý** → chọn **AP** (*lệch tạm chấp nhận, ưu tiên luôn phục vụ*).
- **Cùng một hệ, hai chế độ** — đúng tinh thần **per-operation** (I-CAP-011).

**Bigtech pattern (đã verify mức landscape):**
- **Kubernetes** coordination dùng **etcd** (**CP, Raft**).
- **DynamoDB** là **AP** (*Dynamo paper* — eventually consistent + optional strongly-consistent reads).
- **Cassandra** cho chọn **per-query** (**ONE = EL**, **QUORUM = EC**).
- **Google Spanner / CockroachDB** cố *"vừa C vừa A trên thực tế"* bằng **đồng bộ đồng hồ (TrueTime)** + **private network** để partition *cực hiếm* — **không phá CAP**, chỉ làm **P hiếm đến mức bỏ qua được**.

> ⚠️ **(verify chi tiết khi học.)**

---

## ⑤ Code thực hành + cấu hình

> 💡 **CAP là quyết định kiến trúc / cấu hình, KHÔNG phải đoạn code.** "Code" ở đây là **chọn consistency level đúng cho từng path**.

```javascript
// Ví dụ minh hoạ tunable consistency (Cassandra-style pseudo) — verify driver thật
// Path tài chính: cần đọc thấy ghi mới nhất → QUORUM (PC+EC)
await session.execute(query, { consistency: 'QUORUM' });   // chậm hơn, ĐÚNG hơn

// Path hiển thị feed: chấp nhận hơi cũ → ONE (PA+EL)
await session.execute(query, { consistency: 'ONE' });      // nhanh hơn, có thể STALE

// 🧠 Bài học chỉ-huy-AI: AI dễ set MẶC ĐỊNH một consistency cho MỌI query.
//    Người hiểu phải GÁN đúng mức cho đúng path — sai mức ở path tiền = BUG ẨN.
```

```javascript
// MongoDB: cấu hình readConcern/writeConcern theo path — verify docs chính thức
write tiền:    writeConcern: { w: 'majority' }        // nghiêng CP/EC (đúng, chậm hơn)
đọc dashboard: readConcern: 'local'                   // nhanh, có thể stale
```

> 🔧 **Cấu hình:**
> - **Ghi rõ policy consistency** cho từng nhóm API trong **tài liệu kiến trúc**.
> - **Viết test cho path nhạy cảm** (tiền, tồn kho) để bắt việc *lỡ tay đặt sai mức*.
> - **Giám sát replication lag** để biết **staleness window thực tế**.

---

## ⑥ Keywords cần nhớ (glossary)

**🧠 Cần HIỂU (nắm bản chất):**
- **CAP** (P bắt buộc → chọn C hay A khi partition).
- **linearizability-as-CAP-C** — read thấy write mới nhất như một bản duy nhất.
- **CAP-C vs ACID-C** — replica agreement vs invariant; trực giao.
- **PACELC** — thêm vế Else: Latency vs C.
- **tunable / per-operation consistency** — chọn mức theo từng path.
- **conflict resolution** (**LWW / vector clock / CRDT**).
- **partition không nhị phân** — "B chết?" là phán đoán mờ.

**📌 Cần THUỘC (nhớ nguyên văn):**
- *"P → A vs C; E → L vs C"*.
- **CP** = {etcd, ZK, Mongo-majority, Spanner...}; **AP** = {Cassandra-low, DynamoDB, CouchDB, DNS} **(verify)**.
- **Mongo ≈ PC/EC**, **Cassandra ONE ≈ PA/EL**.

**🛠️ Cần LÀM ĐƯỢC (kỹ năng tay):**
- Chọn **consistency level đúng per-path**.
- **Định lượng staleness window**.
- Chọn **chiến lược resolve conflict**.

---

## ⑦ Mạch tư duy cần nhớ (mental model)

> 🎯 **Khẩu quyết — gói cả bài:**
> *"**CAP**: khi mạng chia, chọn **Đúng (C)** hay **Sống (A)**. **PACELC** thêm: khi mạng lành, vẫn chọn **Nhanh (L)** hay **Đúng (C)**. Và lựa chọn này là **PER-OPERATION**, không phải nhãn dán cho cả hệ."*

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** Phát biểu **CAP**. *"Chọn 2 trong 3"* gây **hiểu lầm** gì?
→ *Gợi ý:* P bắt buộc trong hệ phân tán → thực chất chỉ chọn C hay A khi partition.

**(khó)** **"Consistency" trong CAP** nghĩa chính xác là gì? Khác **ACID-C** ra sao?
→ *Gợi ý:* CAP-C = linearizability/replica agreement; ACID-C = invariant; trực giao.

**(khó)** **CP và AP** hành xử khác nhau thế nào khi partition? Ví dụ DB mỗi loại.
→ *Gợi ý:* CP từ chối/chặn (etcd, ZK); AP vẫn phục vụ data cũ (Cassandra low, DynamoDB).

**(rất khó)** **PACELC** mở rộng CAP thế nào, vì sao thực tế hơn?
→ *Gợi ý:* thêm vế Else (Latency vs C) — trade-off tồn tại cả khi mạng lành.

**(rất khó)** Một hệ **vừa strong cho tài chính vừa eventual cho analytics** — đúng/sai?
→ *Gợi ý:* **đúng** — consistency là per-operation, tune theo nghiệp vụ.

> 🧩 **Thách đố / trick — AI hay sai ở đây:**
> *"Postgres đơn lẻ của tôi là CP hay AP?"*
> → ❌ **câu hỏi sai** — **CAP chỉ áp cho hệ phân tán có replica**; single-node *không "chọn C/A"*.
>
> *"Chúng tôi chọn availability"* — interviewer hỏi lại: *"staleness window bao lâu thì chấp nhận, conflict resolve bằng gì?"*
> → nếu **không trả lời được con số** là **chưa đủ**.

---

## ⑨ Bài tập thực hành + tiêu chí tự chấm

**BT1 — Gán CP/AP cho 6 tính năng.**
Cho: **số dư ví**, **đếm view**, **trạng thái đơn**, **danh sách bạn bè**, **log audit**, **gợi ý sản phẩm** → gán **CP/AP** + **mức consistency** + **lý do nghiệp vụ**.
> ✅ **Đạt khi:** mỗi mục có **lựa chọn** + **định lượng staleness chấp nhận được** + **cách resolve conflict** (nếu AP).

**BT2 — Phân loại PACELC cho 3 hệ bạn đang dùng (verify docs).**
> ✅ **Đạt khi:** gọi đúng **PA/PC** và **EL/EC** kèm **dẫn chứng cấu hình**, *không chép nhãn từ trí nhớ*.

---

## ⑩ Đọc thêm / nguồn chuẩn

- **Brewer** — *CAP Twelve Years Later*.
- **Abadi** — *Consistency Tradeoffs in Modern Distributed Database System Design* (**PACELC**).
- **DDIA ch.5 & 9**.
- **Jepsen.io** — phân tích **consistency thực nghiệm** của DB cụ thể (**verify** từng hệ).
- **Docs chính thức** của DB bạn dùng (**read/write concern**, **consistency level**).

---

> 🔚 **Hết Bài 5.** → **Bước 3 — Chốt ghi nhớ.**
>
> 🎯 **Tâm điểm:** **CAP-C ≠ ACID-C**, **PACELC vế Else**, **per-operation**. Gợi ý 3 câu tự hỏi: *(1) Vì sao "chọn 2 trong 3" là sai? (2) Hai chữ "C" (CAP vs ACID) khác nhau thế nào? (3) PACELC thêm gì mà CAP thiếu, vì sao quan trọng cả khi mạng lành?*
