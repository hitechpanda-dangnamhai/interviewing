# 🎯 BÀI 7 — Consensus (Raft / Paxos / Quorum / Leader Election)

**Phủ:** I-CONSENSUS-001 → I-CONSENSUS-009 · **Nền móng đứng sau quorum, fencing token, CP system. Tổng kết cả mục I.**

> 💡 **Cách đọc bài này.** Đây là **điểm hội tụ** của cả mục I: **quorum** (Bài 5/6), **fencing token** (Bài 4), **CP system** (Bài 5) — tất cả *quy về một viên gạch nền* gọi là **consensus**. Hai câu cần khắc cốt: ***(1) consensus dựa MAJORITY (nên dùng số node LẺ) để cấm split-brain; (2) consensus ĐÚNG nhưng ĐẮT — chỉ đặt ở control-plane, đừng nhét vào hot path data.***

---

## ① Mục tiêu & vị trí trong mạch

Học xong bài này bạn sẽ làm được 7 việc:

1. **Hiểu bài toán consensus** và *nó cần cho việc gì*.
2. **Phân biệt Paxos vs Raft** ở mức thực dụng.
3. **Nắm quorum/majority** chống **split-brain**.
4. **Mô tả Raft elect leader** bằng trực giác.
5. **Hiểu hành vi khi network partition.**
6. **Biết vì sao consensus là cách "đúng" để sinh fencing token** (nối **Bài 4**).
7. **Hiểu vì sao consensus đắt** → **không đặt vào hot path**.

> 🎯 **Vị trí trong mạch học — đây là điểm hội tụ:**
> - **quorum** (Bài 5/6) → consensus *thực thi* nó.
> - **fencing token** (Bài 4) → consensus *sinh ra* nó đúng cách.
> - **CP system** (Bài 5) → consensus là *cơ chế* khiến nó "từ chối phục vụ phe thiểu số".

**Ẩn dụ định vị:** suốt mục I bạn gặp đi gặp lại từ "majority", "quorum", "for correctness → ZK/etcd". Bài 7 mở nắp xem *bên trong* những thứ đó vận hành ra sao — *làm sao nhiều máy không tin nhau, có máy chết giữa chừng, vẫn đồng ý được một sự thật chung*.

---

## ② Giảng cơ bản → nâng cao

### (a) Consensus là gì — **I-CONSENSUS-001**

> **Ví dụ trực giác:** Một nhóm bạn chia tài sản, *không ai tin tuyệt đối ai*, lại có người *thỉnh thoảng ngất xỉu* (mất liên lạc). Làm sao cả nhóm **thống nhất một con số chung** mà không ai gian lận được, kể cả khi vài người vắng mặt giữa chừng? Đó là bài toán consensus.

**Consensus** = nhiều **node** phải **đồng ý trên một giá trị / một thứ tự** *dù có lỗi* (**crash fault** — node chết, mạng trễ).

**Cần cho:**
- **leader election** (bầu một node làm trưởng),
- **log / state replication** (sao chép nhật ký/trạng thái nhất quán),
- **distributed lock đúng đắn** (nối Bài 4),
- **config / coordination store** (**etcd**).

> 🎯 **Chốt:** Consensus là **"viên gạch" của mọi hệ CP**. Đằng sau mọi thứ "nhất quán mạnh" bạn thấy ở mục I, gần như đều có consensus.

---

### (b) Paxos vs Raft & quorum

#### 🔍 Paxos vs Raft — **I-CONSENSUS-002**

| Tiêu chí | **Paxos** | **Raft** |
|---|---|---|
| Tính đúng | **Đúng-được-chứng-minh** | Đúng |
| Dễ hiểu / implement | **Khó hiểu, khó implement** | **Thiết kế cho dễ hiểu** |
| Cách tiếp cận | Trừu tượng | **Tách rõ** leader election + log replication + safety |
| Vị thế hiện nay | Kinh điển học thuật | **Chuẩn de-facto** (etcd, Consul, nhiều hệ) |

> 💡 **Mẹo phỏng vấn:** **đừng diễn Paxos từng bước** (sa lầy ngay). Nói được *"Raft tách bài toán cho dễ hiểu, là chuẩn thực tế"* là **đủ điểm**.

---

#### 🔍 Quorum / majority — **I-CONSENSUS-003**

> **Ví dụ trực giác:** Muốn ra quyết định trong hội đồng, cần *quá bán* giơ tay. Nếu hội đồng tách làm đôi, *chỉ một nửa có thể đạt quá bán* — nửa kia không thể tự quyết. Nhờ vậy không bao giờ có *hai quyết định mâu thuẫn*.

**Quorum / majority** = chỉ **majority (N/2 + 1)** mới được **elect leader / commit write** → tránh **split-brain** (hai leader cùng lúc).

**Vì sao dùng số LẺ (3/5/7)** — tối ưu khả năng chịu lỗi:

| N (số node) | Chịu được chết | Ghi chú |
|---|---|---|
| **3** | **1** | |
| **5** | **2** | |
| **7** | **3** | |
| **4** | **1** | ❌ Chỉ bằng N=3 → **thừa 1 node vô ích** |

> 🎯 **Chốt:** **4 node cũng chỉ chịu 1 chết như 3** → chọn **số lẻ** để không lãng phí.

---

### (c) Cơ chế Raft (trực giác)

#### 🔍 Elect leader — **I-CONSENSUS-004**

> **Ví dụ trực giác:** Lớp học mất giáo viên. Học sinh nào *chờ một lúc không thấy ai chỉ huy* (election timeout) thì *xung phong ứng cử*, hô to "bầu tôi đi!". Ai được *quá nửa lớp* đồng ý → thành lớp trưởng, bắt đầu *điểm danh đều đặn* (heartbeat) để mọi người biết "tôi còn sống, đừng bầu lại".

**Timeline — Raft leader election:**

```
T0  Tất cả là follower, leader cũ vừa chết (hết heartbeat)
T1  follower-A: không nghe heartbeat trong ELECTION TIMEOUT
       ↓
T2  A → CANDIDATE: tăng TERM (vd term 4→5), xin vote từ các node khác
       ↓
T3  B, C: "term 5 > term mình biết → ok, vote cho A"
       ↓
T4  A nhận MAJORITY vote (≥ N/2+1) → thành LEADER ✅
T5  A phát HEARTBEAT đều → các node biết có leader, không ứng cử nữa
```

> 💡 **Term đóng vai logical clock** (đồng hồ logic): **term cao hơn = thông tin mới hơn** → node nhận ra **leader cũ / thông tin cũ** và bỏ qua.

---

#### 🚨 Split-brain — **I-CONSENSUS-005**

> **Ví dụ trực giác:** Một công ty mà *hai người cùng tự xưng CEO*, mỗi người ký một loạt quyết định trái ngược → hỗn loạn. **Split-brain** là tình trạng đó trong hệ phân tán.

**Split-brain** = hai node cùng tưởng mình leader → **ghi mâu thuẫn**.

**Timeline — majority quorum chặn split-brain:**

```
Cluster 5 node bị chia: phe X = {n1,n2,n3}  |  phe Y = {n4,n5}

phe X (3 node): đạt majority (≥3) → elect leader, COMMIT được ✅
phe Y (2 node): KHÔNG đạt majority (cần 3) → STALL, không commit ❌
       ↓
→ Không thể có HAI majority trong cùng cluster → KHÔNG thể hai leader hợp lệ ✅
```

> 🎯 **Chốt:** Consensus chặn split-brain bằng **majority quorum**: chỉ **phe đa số** mới commit/elect, **phe thiểu số stall**.

---

#### 🔍 Network partition — **I-CONSENSUS-006**

**Timeline — partition rồi heal:**

```
PARTITION:
   phe majority: elect leader mới + tiếp tục phục vụ ✅
   phe thiểu số: không đạt quorum → KHÔNG commit (block / trả stale tùy cấu hình)

HEAL (mạng lành lại):
   leader cũ (phe thiểu số): thấy TERM cao hơn từ phe kia
       ↓
   → STEP DOWN (tự thoái vị) + đồng bộ lại log theo leader mới ✅
```

> 💡 Đây **chính là** cách **CP system "từ chối phục vụ phe thiểu số"** trong **Bài 5**. Giờ bạn thấy *cơ chế bên dưới* của hành vi CP.

---

### (d) Nối với fencing & chi phí

#### 🔍 Consensus sinh fencing token — **I-CONSENSUS-007**

Nhớ lại **Bài 4**: **fencing token** cần **đơn điệu tăng** *+ bền qua node chết*. Vấn đề:

| Cách sinh token | Vấn đề |
|---|---|
| Đếm trên **1 node** | ❌ **Không chịu lỗi** (node chết → mất chuỗi) |
| Nhiều node **tự đếm** | ❌ **Lệch nhau** (mỗi node một dãy số) |
| **Consensus** | ✅ Chuỗi thứ tự **nhất quán + bền vững** |

> 🎯 **Chốt:** Chỉ **consensus** cho được chuỗi thứ tự nhất quán, bền vững → **ZooKeeper (zxid)** / **etcd (revision)** *cấp sẵn* thứ tự này. **Đây là lý do Bài 4 nói "for correctness → ZK/etcd".**

---

#### 🔍 etcd ở hạ tầng thật — **I-CONSENSUS-008**

**etcd** = store **cấu hình / coordination** cho:
- **Kubernetes** (toàn bộ **state cluster**),
- **service discovery**,
- **leader election**,
- **distributed lock**.

Vì **Raft** cho **strong consistency** + cơ chế **watch** (theo dõi thay đổi).

> ⚠️ **(verify landscape.)**

---

#### 🚨 Consensus ĐẮT, đừng đặt vào hot path — **I-CONSENSUS-009** (⭐⭐⭐⭐⭐)

> 🚨 **Vì sao đắt?** Mỗi quyết định cần **round-trip tới majority** + **fsync log** (ghi nhật ký xuống đĩa an toàn) → **latency cao**, **throughput giới hạn**.

> 🎯 **Nguyên tắc TL (vàng):** **Tách control-plane (consensus) vs data-plane (throughput cao).** Chỉ dùng consensus cho **metadata / coordination / leader election**, **KHÔNG cho mọi write data-plane**. Đặt consensus *đúng chỗ*, không nhét vào *đường nóng*.

| | **Control-plane** | **Data-plane** |
|---|---|---|
| Việc | metadata, coordination, leader election | write/read khối lượng lớn của ứng dụng |
| Đặt consensus? | ✅ Có | ❌ Không (sẽ giết throughput) |
| Ưu tiên | Đúng đắn | Throughput cao |

> 📘 **vs** ➕ — *docs gốc roadmap* mở đầu **agentic/coordination**; **toàn bộ consensus chi tiết là phần ➕ chiều sâu TL.**

---

## ③ ⚠️ Kiến thức cũ / cách làm bị thay thế

| Cái cũ / hiểu lầm | Thay bằng (đúng) | Vì sao |
|---|---|---|
| **Tự implement Paxos** | Dùng **Raft** (etcd/Consul) hoặc thư viện đã kiểm chứng | Paxos **khó đúng**; Raft là **chuẩn de-facto** |
| **Số node chẵn** cho cluster (4, 6) | **Số lẻ** (3, 5, 7) | Chẵn **không tăng** khả năng chịu lỗi, tốn tài nguyên |
| Tự đếm **fencing token** trên app/1 node | Lấy thứ tự từ **consensus store** (**zxid / revision**) | 1 node không bền; nhiều node tự đếm lệch |
| Đặt **consensus vào mọi write** (hot path) | Consensus cho **control-plane**; **data-plane** riêng | **Round-trip majority + fsync** → quá chậm |
| *"Cluster vẫn chạy khi mất quá nửa node"* | **Mất majority → không commit được** (đúng theo thiết kế) | **An toàn > sống** khi đã mất quorum |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case — "chỉ một instance là leader xử lý job nhạy cảm":**
Dùng **etcd lease + leader election**: instance **thắng lease** là leader; **mất lease** (chết/partition) thì instance khác lên — *an toàn vì **etcd (Raft)** không cho hai leader*.

**Bigtech pattern (verify landscape):**
- **Kubernetes** lưu **toàn bộ state trong etcd** (**Raft**).
- **Consul** (HashiCorp) dùng **Raft** cho **service mesh coordination**.
- **Kafka** từng dùng **ZooKeeper** cho controller/broker metadata, *nay chuyển sang* **KRaft** (Raft nội bộ, bỏ ZooKeeper).
- **Google Spanner** dùng **Paxos** cho **replication group**.
- **Khuynh hướng:** **Raft thắng** vì *dễ hiểu / dễ vận hành*.

> ⚠️ **(verify — landscape đổi theo phiên bản.)**

---

## ⑤ Code thực hành + cấu hình

> 💡 **Consensus hiếm khi tự viết** — bạn **dùng etcd/ZooKeeper**. "Code" là **dùng API leader election / lease cho đúng**.

```javascript
// ✅ Leader election bằng etcd lease (ý tưởng — verify API client thật)
// "etcd3"/client chính thức: kiểm chứng tại etcd docs
async function runAsLeaderOnly(work: () => Promise<void>) {
  const lease = await etcd.lease(10);                 // lease 10s, tự renew
  const lock = etcd.lock('jobs/leader');
  try {
    await lock.acquire();                              // chỉ 1 instance thắng (Raft đảm bảo)
    // ✅ Đến đây mình là leader DUY NHẤT → làm việc nhạy cảm
    await work();
  } finally {
    await lock.release();
    await lease.revoke();
  }
}
// 🧠 Khác hẳn Redis SET NX (Bài 4): ở đây thứ tự & độc-nhất do CONSENSUS đảm bảo,
//    an toàn cho correctness; đổi lại nặng/chậm hơn → đừng nhét vào hot path.
```

**Vì sao số lẻ — khả năng chịu lỗi `f = ⌊N/2⌋`:**

```
N=3 → f=1   N=5 → f=2   N=7 → f=3
N=4 → f=1 (bằng N=3 nhưng tốn hơn) → CHỌN LẺ
// Quorum để commit/elect = N/2 + 1  (3→2, 5→3, 7→4)
```

> 🔧 **Cấu hình:**
> - **Cluster consensus đặt số lẻ**, **tách khỏi data-plane**.
> - Đặt **election / heartbeat timeout** hợp lý theo độ trễ mạng.
> - **KHÔNG để consensus store gánh write throughput cao** của ứng dụng.
> - **Backup / snapshot định kỳ** — **etcd có thể là single point of failure** cho cả cluster K8s.

---

## ⑥ Keywords cần nhớ (glossary)

**🧠 Cần HIỂU (nắm bản chất):**
- **consensus** (đồng ý dù có **crash fault**).
- **Paxos vs Raft**.
- **quorum / majority**.
- **split-brain**.
- **term-as-logical-clock**.
- **leader election**.
- **step-down** khi heal.
- **control-plane vs data-plane**.
- **"consensus đắt"**.

**📌 Cần THUỘC (nhớ nguyên văn):**
- **majority = N/2 + 1**.
- **số lẻ 3/5/7 chịu 1/2/3**.
- **Raft = election + log replication + safety**.
- **etcd = Raft (K8s)**.

**🛠️ Cần LÀM ĐƯỢC (kỹ năng tay):**
- Dùng **etcd/ZK** cho **leader election**.
- Giải thích hành vi **partition + heal**.
- **Chỉ ra chỗ KHÔNG nên đặt consensus.**

---

## ⑦ Mạch tư duy cần nhớ (mental model)

> 🎯 **Khẩu quyết — gói cả bài (và cả mục I):**
> *"**Consensus = nhiều node đồng ý dù có lỗi, dựa MAJORITY** (nên dùng số lẻ) để **cấm split-brain**. **Raft = Paxos-dễ-hiểu**, là chuẩn thực tế (etcd/K8s). Nó **đúng nhưng ĐẮT** → chỉ đặt ở **control-plane** (coordination/leader/fencing), **tránh hot path data**."*

---

## ⑧ Câu hỏi phỏng vấn & thách đố

**(TB)** **Consensus** là bài toán gì, cần cho việc nào?
→ *Gợi ý:* nhiều node đồng ý một giá trị/thứ tự dù có crash fault; cho leader election, log replication, lock đúng, etcd.

**(TB)** **Paxos vs Raft** khác biệt thực dụng nào? (và *đừng làm gì* trong phỏng vấn)
→ *Gợi ý:* Raft dễ hiểu, chuẩn de-facto; *đừng diễn Paxos từng bước*.

**(khó)** **Quorum** là gì, **vì sao thường số node lẻ**?
→ *Gợi ý:* majority N/2+1; lẻ tối ưu chịu lỗi (4 chỉ bằng 3).

**(khó)** **Raft elect leader** thế nào (trực giác)? **Term** đóng vai gì?
→ *Gợi ý:* election timeout → candidate → xin vote → majority → leader; term = logical clock.

**(rất khó)** Vì sao **consensus là cách "đúng" để sinh fencing token**? Vì sao **đắt**, khi nào **KHÔNG đặt vào hot path**?
→ *Gợi ý:* cho chuỗi đơn điệu bền (zxid/revision); đắt vì round-trip majority + fsync; chỉ ở control-plane.

> 🧩 **Thách đố / trick — AI hay sai ở đây:**
> *"Cluster 5 node mất 3 node, sao không ghi được nữa?"*
> → **mất majority** (cần 3) → **đúng theo thiết kế**, *thà block còn hơn split-brain*.
>
> *"AI đề xuất đẩy mọi write qua etcd cho 'nhất quán mạnh' — duyệt không?"*
> → ❌ **chặn**: consensus có **round-trip majority + fsync**, đặt vào **data-plane** sẽ **giết throughput**; đây là **AI đúng khái niệm nhưng sai chỗ đặt** — chỉ huy **tách control-plane vs data-plane**.

---

## ⑨ Bài tập thực hành + tiêu chí tự chấm

**BT1 — Leader election bằng etcd (hoặc ZooKeeper) cho 3 instance.**
> ✅ **Đạt khi:** đúng **1 leader tại mọi thời điểm**; **kill leader** → instance khác lên trong vài giây; **mô tả được** điều gì xảy ra khi **partition**.

**BT2 — Viết 1 trang "đặt consensus ở đâu trong kiến trúc của tôi".**
> ✅ **Đạt khi:** chỉ rõ phần nào là **control-plane** (đặt consensus) vs **data-plane** (không), kèm **lý do latency/throughput**.

---

## ⑩ Đọc thêm / nguồn chuẩn

- **In Search of an Understandable Consensus Algorithm (Raft)** — Ongaro & Ousterhout.
- **raft.github.io** (visualization tương tác).
- **etcd docs** — *Learning / Why etcd*; **ZooKeeper** — *Internals (ZAB)*.
- **DDIA ch.9** (consensus, total order broadcast, membership).
- **Lamport** — *Paxos Made Simple* (đọc để biết, không cần thuộc).

---

> 🔚 **Hết Bài 7 — và hết mạch mục I.** → **Bước 3 — Chốt ghi nhớ.**
>
> 🎯 **Tâm điểm recall:** **majority/số lẻ chống split-brain**, **term = logical clock**, **consensus đắt → control-plane only**. Gợi ý 3 câu tự hỏi: *(1) Vì sao mất majority thì thà block còn hơn ghi tiếp? (2) Raft bầu leader thế nào, term làm gì? (3) Vì sao "mọi write qua etcd" là sai chỗ đặt?*
>
> 💡 **Nhìn lại cả mục I:** race sinh ở **khoảng trống đọc–ghi** (Bài 1) → bịt bằng **lock** (Bài 2) → chống lặp bằng **idempotency** (Bài 3) → mở rộng ra **distributed lock + fencing** (Bài 4) → đánh đổi **CAP/PACELC** (Bài 5) → chọn **mức consistency** (Bài 6) → và **consensus** (Bài 7) là viên gạch nền làm ra tất cả những đảm bảo "đúng" đó.
