# 🔺 CHUYÊN ĐỀ — CAP THEOREM (Định lý CAP) trong Caching & Hệ phân tán
> Đào sâu một concept mở rộng của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu *chính xác* CAP (không phải bản "pick 2/3" sai lệch), biết *chọn CP hay AP theo path*, và *thiết kế cho lúc partition* ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **CAP theorem** xuất hiện ở **bảng concept mở rộng (B2)**:

> ➕ **CAP theorem** — *Hệ phân tán chỉ chọn 2/3: Consistency, Availability, Partition-tolerance* — *"Distributed cache buộc bạn đối mặt đánh đổi kiểu CAP."*

Ở chuyên đề **Eventual Consistency** ta đã chạm CAP/PACELC để giải thích *vì sao* eventual tồn tại. Chuyên đề này "mở hộp" chính CAP — và sẽ **đính chính ngay** cái mô tả "chọn 2/3" ở trên, vì đó là cách hiểu phổ biến nhưng **gây hiểu lầm**.

Học xong bạn sẽ:

- Phát biểu **chính xác** CAP (định nghĩa C/A/P thật, không phải bản dân gian) và vì sao **"P không phải lựa chọn"**.
- Hiểu lựa chọn thật sự là **CP vs AP *khi* partition xảy ra**, không phải "bỏ một trong ba".
- Phân biệt **C trong CAP** (linearizability) với **C trong ACID** — hai thứ khác hẳn.
- Biết **PACELC** (đánh đổi cả lúc *không* partition), **consensus (Raft/Paxos)**, **fencing token**, và cách **Spanner "lách" CAP**.
- Áp vào **distributed cache**: vì sao cache gần như luôn **AP**, và khi nào phải đặt một lõi **CP** bên cạnh.

> 💡 **Một câu để gắn vào mạch Bài 1:** Bài 1 nói *"distributed cache buộc bạn đối mặt đánh đổi kiểu CAP"*. Chính xác hơn: **khi mạng giữa các node cache (hoặc giữa cache và nguồn) bị đứt, bạn buộc phải chọn — trả lời bằng đồ có-thể-cũ (AP) hay từ chối trả lời để khỏi sai (CP)?** Cache, theo bản năng thiết kế, gần như luôn chọn **AP**.

---

## ② Trực giác — CAP nói gì?

### Phát biểu hình thức (Gilbert & Lynch, 2002 — chứng minh giả thuyết của Brewer 2000)

> Một **hệ lưu trữ phân tán** **không thể** đồng thời đảm bảo cả ba:
> - **C — Consistency (linearizability):** mọi lần đọc thấy **lần ghi mới nhất** (như thể chỉ có một bản duy nhất).
> - **A — Availability:** mọi request tới một node **còn sống** đều nhận **phản hồi (không lỗi)** — không nhất thiết nhanh.
> - **P — Partition tolerance:** hệ vẫn **tiếp tục hoạt động** dù mạng giữa các node **bị đứt / mất gói tuỳ ý**.

### ⚠️ Đính chính NGAY: "chọn 2 trong 3" là cách hiểu SAI lệch

Cách nói dân gian "*chọn 2 trong 3*" khiến người ta tưởng có thể vứt **P** để giữ cả C và A. **Không.**

> 🔑 **Insight #1 (quan trọng nhất cả bài):** Trong một hệ phân tán thật, **partition CHẮC CHẮN sẽ xảy ra** — cáp đứt, switch hỏng, GC pause, mất gói. Bạn **không được chọn "không có partition"**. Bạn chỉ chọn **làm gì KHI nó xảy ra**. Vì vậy CAP thực chất là:
>
> **"Khi partition xảy ra → chọn C hay A?"** → tức là **CP hay AP**.
>
> "CA" chỉ có nghĩa với hệ **một node / không phân tán** (RDBMS đơn lẻ). Gắn nhãn "CA" cho một hệ phân tán = hiểu sai CAP.

### Ẩn dụ hai chi nhánh ngân hàng bị mất liên lạc

Hai chi nhánh A và B dùng chung sổ tài khoản, nhưng **đường truyền giữa chúng bị đứt** (partition). Khách tới chi nhánh A rút tiền. A **không thể** hỏi B "số dư hiện tại là bao nhiêu". Hai lựa chọn:

| Lựa chọn | Hành xử | Đánh đổi |
|---|---|---|
| **CP** (giữ Consistency) | A **từ chối** cho rút (vì không chắc số dư) | An toàn tuyệt đối nhưng **mất availability** — khách bực |
| **AP** (giữ Availability) | A **vẫn cho rút** theo số dư A biết, hoà giải sau | Luôn phục vụ nhưng **có thể rút quá số dư** (overdraft) |

> Không có lựa chọn "vừa cho rút vừa chắc chắn đúng" trong lúc đứt mạng. Đó **là** CAP. Và lựa chọn đúng **phụ thuộc nghiệp vụ**: tiền → thường CP; giỏ hàng → AP.

---

## ③ C trong CAP ≠ C trong ACID (lỗi hiểu lầm kinh điển #2)

| | **C của CAP** | **C của ACID** |
|---|---|---|
| Nghĩa | **Linearizability** — đọc luôn thấy ghi mới nhất, mọi node thống nhất theo thời gian thực | **Consistency** — giao dịch giữ **bất biến/ràng buộc** của DB (khoá ngoại, check…) |
| Về cái gì | Sự **đồng nhất giữa các bản sao** | Sự **đúng đắn của một transaction** |
| Liên quan | Phân tán, replica | Một DB (kể cả đơn node) |

> 🔑 **Insight #2:** Khi nói "hệ này hi sinh C để được A", **C ở đây là linearizability** (đọc thấy ghi mới nhất xuyên các node), **không phải** ràng buộc toàn vẹn dữ liệu của ACID. Lẫn hai chữ C này là sai lầm phổ biến trong phỏng vấn.

Liên quan: **linearizability ≠ serializability**. Linearizability nói về *một object theo thời gian thực*; serializability nói về *các transaction chạy như thể tuần tự*. "Strong consistency" trong dân gian thường ám chỉ linearizability.

---

## ④ CP vs AP — bản đồ quyết định

Khi partition xảy ra:

```
                 PARTITION xảy ra
                        │
        ┌───────────────┴───────────────┐
        ▼                               ▼
   CHỌN C (CP)                     CHỌN A (AP)
   "Thà không trả lời             "Thà trả lời (có thể cũ)
    còn hơn trả lời sai"           còn hơn không trả lời"
        │                               │
  Bên thiểu số TỪ CHỐI ghi/đọc    Mọi bên VẪN nhận ghi/đọc
  (giữ linearizability)           (phân kỳ → hoà giải sau = eventual)
        │                               │
  VD: Zookeeper, etcd, HBase,     VD: Cassandra, DynamoDB(eventual),
      MongoDB(default), Spanner       Riak, CouchDB, DNS, đa số CACHE
```

| Tiêu chí | **CP** | **AP** |
|---|---|---|
| Lúc partition | **Hi sinh availability** (lỗi/từ chối ở phía không chắc đúng) | **Hi sinh consistency** (phục vụ đồ có thể cũ) |
| Phù hợp | Tiền, khoá phân tán, bầu leader, config, ràng buộc duy nhất | Đếm, feed, gợi ý, **cache đọc**, presence |
| Hoà giải | Không cần (luôn nhất quán) | Cần **conflict resolution** (LWW/merge/CRDT) |
| Bài 1 nói | *"path tiền/tồn kho cần strong"* = chọn **CP/strong** | *"like count eventual được"* = chọn **AP/eventual** |

> 🔑 **Insight #3:** **Cache là một quyết định AP gần như mặc định.** Bạn cache *để* vẫn nhanh và vẫn trả lời được kể cả khi nguồn chậm/đứt — chấp nhận stale. Đó chính là *"chọn A, hi sinh C"*. Bài 1 chỉ đang nói điều này bằng ngôn ngữ đời thường: *"cache = nhanh hơn nhưng rủi ro stale"*.

---

## ⑤ PACELC — vì sao CAP chưa đủ cho cache

CAP chỉ nói về lúc **partition**. Nhưng phần lớn thời gian **không** có partition — vậy lúc đó có đánh đổi gì không? **Có.** Đó là lý do có **PACELC** (Abadi):

> **Nếu P**artition: chọn **A** hay **C**.
> **Else** (lúc bình thường): vẫn chọn **L**atency hay **C**onsistency.

| Hệ | Lúc partition | Lúc bình thường | Nhãn |
|---|---|---|---|
| Dynamo / Cassandra / Riak | A | L (nhanh, eventual) | **PA/EL** |
| MongoDB (mặc định) | C | C | **PC/EC** |
| PNUTS (Yahoo) | C | L | **PC/EL** |
| Spanner | C | C (TrueTime) | **PC/EC** |
| **Distributed cache (điển hình)** | **A** | **L** (đọc nhanh, chấp nhận stale) | **PA/EL** |

> 🔑 **Insight #4:** Cache là **PA/EL** — *ngay cả khi mạng khoẻ mạnh*, nó vẫn chọn **Latency** hơn **Consistency** (đọc RAM thay vì đi nguồn để chắc tươi). Đây chính là **ba thứ đánh đổi của Bài 1** (*tốc độ ↔ tươi ↔ RAM*) phát biểu bằng lý thuyết phân tán. PACELC bắt được phần "EL" mà CAP thuần bỏ sót — và đó mới là phần cache sống ở đó **99% thời gian**.

---

## ⑥ Ví dụ minh hoạ

### Ví dụ 1 — Redis Cluster bị split-brain (rất sát Bài 1)

Redis Cluster có master M và replica R ở hai phía mạng. Mạng đứt:
- Phía client vẫn nối được M cũ → **ghi tiếp vào M**.
- Cluster ở phía kia **promote R thành master mới** → cũng nhận ghi.
- → **Hai master** (split-brain). Khi mạng nối lại, các ghi vào M cũ (trong lúc đứt) **bị mất** (Redis async replication, nghiêng **AP**, *không* linearizable mặc định).

> 🔑 Đây là lý do **đừng dùng Redis làm khoá phân tán "chắc chắn đúng"** mà không có fencing (mục ⑨). Redis là cache tốc độ cao (AP), không phải store đồng thuận (CP).

### Ví dụ 2 — Hai ATM mất mạng (CP vs AP, cụ thể hoá ẩn dụ ②)

- **Ngân hàng chọn CP:** ATM ngoại tuyến → **không cho rút** → khách bực nhưng sổ sách không bao giờ âm.
- **Ngân hàng chọn AP:** ATM **vẫn cho rút tới hạn mức ngoại tuyến** (vd $200) → luôn phục vụ, **chấp nhận rủi ro overdraft** nhỏ, hoà giải khi online lại. (Nhiều hệ ATM đời thực chọn AP có giới hạn — vì mất khách đau hơn rủi ro $200 hiếm gặp.)

> 🔑 **Bài học:** CP/AP **không phải đúng/sai tuyệt đối** — là **quyết định nghiệp vụ + định lượng rủi ro**. Đây là tư duy staff engineer.

### Ví dụ 3 — Bầu leader / config: BẮT BUỘC CP

Service discovery, khoá phân tán, "ai là master" — nếu chọn AP, lúc partition sẽ có **hai bên cùng tưởng mình là leader** → ghi đè nhau → hỏng dữ liệu. Vì vậy **Zookeeper/etcd dùng consensus (Zab/Raft)** và là **CP**: lúc partition, **phía thiểu số ngừng phục vụ** để chỉ một leader hợp lệ tồn tại.

### Ví dụ 4 — Kiến trúc lai (chuẩn thực tế)

```
CP core (nhỏ, đắt, chắc):          AP edge (to, rẻ, nhanh):
  etcd/Zookeeper  → config, lock,    Redis/Memcached → cache đọc
                    leader, metadata  read replicas    → đọc mở rộng
                                      CDN              → asset
   ↑ cần linearizable                 ↑ chịu được stale → AP
```
Hầu hết hệ lớn **không chọn một CAP cho toàn hệ** — họ đặt **lõi CP nhỏ** cho thứ cần đúng tuyệt đối và **vành AP lớn** cho thứ cần tốc độ/quy mô.

---

## ⑦ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến. Verify số liệu cụ thể khi cần trích dẫn.

### CS1 — Amazon Dynamo: chọn AP có chủ đích (SOSP 2007)

Dynamo đặt **"always writable"** (giỏ hàng phải luôn thêm được, kể cả lúc partition) lên trên consistency → **AP**. Hệ quả: phải xử **conflict** khi hội tụ (vector clock + merge giỏ hàng). Triết lý: với Amazon, **một lần không thêm được vào giỏ = mất doanh thu**, đau hơn một lần giỏ hàng phải merge.

> 🎯 **Bài học:** Lựa chọn CAP bắt nguồn từ **đo lường chi phí nghiệp vụ**, không từ "lý thuyết đẹp".

### CS2 — Google Spanner: "lách" CAP bằng kỹ thuật (Brewer, 2017)

Spanner là **CP** về mặt hình thức (lúc một số partition xảy ra, nó **chọn consistency, hi sinh availability**). Nhưng Google đầu tư **mạng riêng + TrueTime (đồng hồ nguyên tử/GPS bounded uncertainty)** khiến partition **cực hiếm** → availability đạt ~5 số 9 → *cảm giác như CA*.

> 🎯 **Bài học:** Spanner **không phá** CAP — nó **làm cho P hiếm tới mức gần như không phải trả giá A**. Đây là chiêu staff/principal: *không né định lý, mà giảm tần suất phải đối mặt với nó* (giống cách giảm tần suất cold start ở chuyên đề trước).

### CS3 — Zookeeper / etcd: CP cho điều phối

Cả hai dùng **consensus** (Zab / Raft) để giữ linearizability cho metadata/khoá/bầu leader. Lúc partition, **phía thiểu số từ chối phục vụ** (mất A) để không bao giờ có hai sự thật. Đây là "trái tim CP" mà các hệ AP dựa vào để điều phối.

### CS4 — Cassandra / DynamoDB: AP nhưng *tunable*

Cho vặn **quorum (R/W/N)** hoặc chọn *strong vs eventual read* **từng request** (đã nói ở chuyên đề Eventual Consistency). ⇒ một hệ có thể **trượt dọc trục CP↔AP theo từng thao tác**, không bị khoá cứng một nhãn.

### CS5 — Redis & tranh luận Redlock

Redis (async replication) nghiêng **AP**, có thể **mất ghi đã ack** lúc failover. Cuộc tranh luận **Redlock** (Kleppmann ⟷ tác giả Redis) chỉ ra: dùng Redis làm **khoá phân tán "đúng tuyệt đối"** là sai chỗ — cần **fencing token** và một store CP. Redis xuất sắc ở vai **cache AP**, không ở vai **lock CP**.

> 🎯 **Bài học:** **Chọn công cụ theo lập trường CAP của nó.** Đừng ép một store AP làm việc của CP. (Bài 1: *"Redis là điểm phụ thuộc → phải HA"* — nhưng HA ≠ linearizable.)

---

## ⑧ Áp dụng — tư duy staff engineer về CAP

### Nguyên tắc gốc: ĐỪNG gắn nhãn CAP cho cả hệ — gắn cho từng *path*

> 🔑 **Insight #5:** Câu hỏi sai: *"Hệ của tôi là CP hay AP?"*
> Câu hỏi đúng: ***"Thao tác/đường dữ liệu NÀY cần linearizable không, và khi partition nó nên làm gì?"***
> Cùng một sản phẩm: **đặt hàng/trừ tiền → CP**; **hiển thị feed/đếm like/đọc cache → AP**. Một hệ chứa cả hai là chuyện bình thường (Ví dụ 4).

### Quy trình quyết định cho mỗi path

1. **Path này có cần đọc-thấy-ghi-mới-nhất (linearizable) không?**
   - Có (tiền, khoá, unique, leader) → **CP**: dùng consensus/transaction, chấp nhận từ chối lúc partition.
   - Không (đếm, feed, cache) → **AP**: phục vụ nhanh, eventual, có conflict resolution.
2. **Lúc partition, mỗi phía làm gì?** (viết ra rõ ràng — đa số sự cố sinh ra vì câu này không ai trả lời.)
3. **Hệ AP hội tụ ra sao?** → LWW/merge/CRDT (chuyên đề Eventual Consistency).
4. **Hệ CP mất availability tới mức nào lúc partition?** → định lượng (phía thiểu số chịu được downtime không?).

### Áp vào distributed cache cụ thể

- Cache (đọc) = **AP**: cứ phục vụ, chấp nhận stale, bound bằng TTL (Bài 1).
- Nguồn (DB) cho **path quyết định** = **CP**: đọc thẳng, transaction, đừng để cache xen vào.
- Khi cache **mất kết nối tới nguồn**: chọn trước — **serve-stale** (AP, ưu tiên availability) hay **fail** (CP, ưu tiên đúng). Phần lớn cache chọn serve-stale + cảnh báo.

---

## ⑨ Giải pháp NÂNG CAO (staff / principal)

### A1. Consensus cho lõi CP (Raft / Paxos / Zab)

Khi cần linearizable (config, lock, leader, metadata), dùng **thuật toán đồng thuận**: chỉ **đa số (quorum)** mới ra quyết định → lúc partition, **phía thiểu số tự ngừng**, không bao giờ có hai sự thật. Raft (etcd/Consul), Zab (Zookeeper), Paxos (Spanner/Chubby). Đây là "viên gạch CP" để xây phần còn lại AP lên trên.

### A2. Fencing token — chống "leader chết đi sống lại" phá dữ liệu

Khi dùng khoá/leader, một node bị **GC pause / partition** có thể **tưởng mình vẫn là leader** sau khi quyền đã chuyển. Giải pháp (Kleppmann): mỗi lần cấp khoá kèm **số tăng dần (fencing token)**; tài nguyên phía sau **từ chối** thao tác mang token **cũ hơn** token đã thấy. ⇒ split-brain không gây hại dù vẫn xảy ra.

```
Leader cũ (đã stale) ghi với token=33  → storage thấy đã có token=34 → TỪ CHỐI
Leader mới ghi với token=34            → CHẤP NHẬN
```

### A3. Quorum tinh chỉnh (R + W > N) + sloppy quorum / hinted handoff

- **R + W > N** → đọc *giao* ghi ≥ 1 bản → đọc thấy ghi mới nhất (nghiêng CP).
- **R + W ≤ N** → nhanh hơn, eventual (nghiêng AP).
- **Sloppy quorum + hinted handoff** (Dynamo): lúc một số node chết, ghi tạm vào node "hàng xóm" + ghi chú để giao lại sau → giữ availability (AP) mà không mất ghi.

### A4. CRDT — cách AP "có nguyên tắc" (Strong Eventual Consistency)

Thay vì AP rồi xử conflict thủ công, dùng **CRDT**: dữ liệu *tự* hội tụ bất kể thứ tự (G-Counter, OR-Set…). Cho **Strong Eventual Consistency**: hai bản nhận cùng tập update → *chắc chắn* về cùng trạng thái, **không cần đồng thuận**. Cực hợp multi-region AP (đếm, presence, collab editing).

### A5. Harvest & Yield (Fox & Brewer) — bỏ tư duy nhị phân "up/down"

Thay vì "available hay không", nghĩ theo:
- **Yield** = % request được trả lời.
- **Harvest** = % dữ liệu *đầy đủ* trong mỗi câu trả lời.

Lúc sự cố, thay vì sập (yield=0), hãy **trả lời một phần** (harvest thấp): ví dụ search bỏ qua một shard chết và trả "kết quả gần đủ". ⇒ một cách **degrade duyên dáng** thay vì CP-cứng "all or nothing". (Nối với *load shedding / graceful degradation* ở chuyên đề Cold Start.)

### A6. Spanner-style: làm P *hiếm* thay vì né định lý (CS2)

Nếu buộc phải CP nhưng cần availability cao, **đầu tư vào hạ tầng để partition cực hiếm** (mạng dư thừa, đồng hồ bounded) → trả giá A gần như không đáng kể. Không phá CAP — **giảm tần suất phải trả giá**.

### A7. Thiết kế *hành vi partition* như một tính năng

Viết rõ runbook: phía thiểu số làm gì, cache serve-stale tới ngưỡng nào thì chuyển sang fail, khi nào tự promote leader (và có fencing chưa). **Diễn tập partition** (chaos engineering: chủ động cắt mạng giữa các node trong staging) — đa số bug CAP chỉ lộ ra khi partition thật, lúc đã quá muộn.

---

## ⑩ Trường hợp MỞ RỘNG (CAP ở mọi tầng)

| Hệ thống | Lập trường điển hình | Ghi chú |
|---|---|---|
| **Distributed cache (Redis/Memcached)** | **AP** | Phục vụ stale để giữ tốc độ/availability |
| **Read replica DB** | **AP** (đọc) | Replication lag = cửa sổ bất nhất |
| **Zookeeper / etcd / Consul** | **CP** | Consensus cho lock/config/leader |
| **Cassandra / DynamoDB / Riak** | **AP** (tunable) | Quorum vặn được CP↔AP |
| **MongoDB / HBase** | **CP** (mặc định) | Ưu tiên consistency lúc partition |
| **Kafka (một partition log)** | **CP-ish** | Với `acks=all` + min ISR: ưu tiên không mất/không sai, có thể mất availability |
| **DNS** | **AP** | Eventual toàn cầu theo TTL |
| **Spanner / CockroachDB** | **CP** | Linearizable, làm P hiếm |
| **Service mesh / discovery** | thường **CP** cho nguồn sự thật, **AP** cho cache local | Lai |
| **Blockchain (PoW)** | thiên **AP** + *eventual finality* | Liên quan FLP/đồng thuận xác suất |

> 🔑 **Insight #6:** Một kiến trúc trưởng thành là **bức tranh khảm CAP**: lõi CP (etcd) + DB chính (CP cho ghi) + replica/cache (AP cho đọc) + CDN (AP). "Hệ là CP hay AP" là câu hỏi sai cấp độ — phải hỏi *từng thành phần*.

---

## ⑪ Kiến thức MỞ RỘNG (kết nối)

- ✅ **Strong consistency (B1) = chọn C (CP).** ➕ **Eventual consistency (chuyên đề trước) = chọn A (AP).** CAP là **khung lý thuyết** đứng sau cặp này.
- ➕ **PACELC:** mở rộng quan trọng nhất — bắt đánh đổi *lúc bình thường* (EL), nơi cache thực sự sống.
- ➕ **FLP impossibility (Fischer-Lynch-Paterson 1985):** trong mạng *bất đồng bộ*, **không** thuật toán đồng thuận nào *vừa đúng vừa chắc chắn kết thúc* nếu có dù chỉ một node lỗi. Khác CAP (CAP về C/A khi partition; FLP về *khả năng đạt đồng thuận*) nhưng cùng họ "giới hạn cơ bản của hệ phân tán". Raft/Paxos lách bằng *timeout* (hi sinh tính chắc-chắn-kết-thúc trong lý thuyết để chạy được thực tế).
- ➕ **Linearizability vs Serializability:** C-của-CAP là linearizability (một object, thời gian thực); serializability là về transaction.
- ➕ **CRDT / vector clock / LWW / quorum:** bộ công cụ hiện thực hoá lựa chọn AP (đã ở chuyên đề Eventual Consistency).
- ➕ **Cache coherence (B2):** ép nhiều bản L1 về gần linearizable bằng pub/sub — *cố kéo AP về phía C* trong nội bộ.
- ➕ **Cold Start & Harvest/Yield:** lúc sự cố, "trả lời một phần" (harvest thấp) là họ hàng của *graceful degradation / load shedding*.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao sai | ✅ Đúng |
|---|---|---|
| "Chọn **CA** cho hệ phân tán" | P không phải lựa chọn — partition sẽ xảy ra | Chọn **CP hay AP** *khi* partition |
| Tưởng **C của CAP = C của ACID** | Hai khái niệm khác hẳn (linearizability vs ràng buộc transaction) | Phân biệt rõ (mục ③) |
| Gắn **một nhãn CAP cho cả hệ** | Thực tế là khảm theo path; nhãn ít khi khớp định nghĩa hình thức | Quyết **theo từng path/thao tác** |
| Dùng **Redis làm lock "đúng tuyệt đối"** | Redis AP, mất ghi lúc failover | Store CP (etcd) + **fencing token** |
| Bỏ qua **hành vi lúc partition** | Split-brain / hai leader phá dữ liệu | Thiết kế + diễn tập partition; fencing |
| Mặc định **LWW** khi hội tụ AP | Mất ghi, phụ thuộc đồng hồ | merge/CRDT theo nghĩa nghiệp vụ |
| Tin "Spanner phá được CAP" | Spanner là CP, chỉ *làm P hiếm* | Hiểu đúng: né tần suất, không né định lý |
| Ép **strong consistency cho mọi path** "cho chắc" | Trả giá latency + availability vô ích ở path chịu được stale | AP cho đọc/đếm/feed; CP chỉ cho path cần |

> 🔎 **AI hay sai (nối Bài 1):** AI vừa hay *"thêm cache cho nhanh"* (ngầm AP cho mọi path), vừa hay *"để chắc thì strong consistency hết"* (CP cho mọi path) — **hai thái cực đều sai**. Luôn ép hỏi: *"Path này cần linearizable không? Lúc partition nó nên trả lời hay từ chối?"*

---

## ⑬ Khung quyết định / Playbook CAP

### D-CAP-1 — Chọn CP hay AP cho một path
```
1. Path cần ĐỌC-THẤY-GHI-MỚI-NHẤT (linearizable) không?
   → CÓ (tiền, lock, unique, leader, hạn mức)  → CP (consensus/transaction)
   → KHÔNG (đếm, feed, cache đọc, presence)     → AP (eventual + conflict resolution)
2. Lúc partition, MỖI PHÍA làm gì? (viết ra — đây là gốc của mọi sự cố)
   → CP: phía thiểu số TỪ CHỐI phục vụ.
   → AP: mọi phía vẫn phục vụ → định nghĩa cách HỘI TỤ.
3. AP thì hội tụ bằng gì? → LWW / merge / CRDT (theo nghĩa nghiệp vụ).
4. CP thì mất availability bao lâu? → định lượng, có chịu được không?
```

### D-CAP-2 — Kiểm tra một kiến trúc
```
□ Đã tách lõi CP (config/lock/leader → etcd/Zookeeper) khỏi vành AP (cache/replica/CDN) chưa?
□ Có store nào đang bị ép sai vai (Redis làm lock CP)? → sửa + fencing token.
□ Mỗi đường dữ liệu đã có quyết định CAP rõ ràng chưa, hay đang mặc định mù?
□ Đã viết & diễn tập hành vi lúc partition (chaos test) chưa?
□ Path tiền/tồn kho/quyền có lỡ đi qua cache (AP) không? → kéo về CP/strong.
```

### D-CAP-3 — Khi cache mất kết nối nguồn (partition cache↔DB)
```
Quyết định TRƯỚC, đừng để runtime tự xử bừa:
  → serve-stale (AP): trả bản cache cũ + cảnh báo + bound bằng TTL.  ← mặc định phổ biến
  → fail-closed (CP): từ chối, buộc đọc nguồn — chỉ cho path cần đúng tuyệt đối.
Và: theo dõi staleness/replication lag như một metric SLA.
```

---

## ⑭ Mental model (câu thần chú gốc)

> 🔺 **CAP không phải "chọn 2/3".** Partition CHẮC CHẮN xảy ra ⇒ thật ra là: **khi mạng đứt, chọn C hay A?** → **CP** (thà từ chối còn hơn sai) hay **AP** (thà trả lời còn hơn im).
>
> **C của CAP = linearizability** (đọc thấy ghi mới nhất xuyên node), **không phải** C của ACID.
> **Cache = AP/EL gần như mặc định:** phục vụ nhanh, chấp nhận stale — chính là *"tốc độ ↔ tươi"* của Bài 1.
> **Đừng gắn một nhãn CAP cho cả hệ** — quyết theo **từng path**: lõi **CP** nhỏ (lock/config/tiền) + vành **AP** lớn (cache/replica/CDN).
> Và **thiết kế cho lúc partition** (fencing, runbook, chaos test) — vì đó là lúc lựa chọn CAP của bạn bị đem ra thử.

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Phát biểu CAP. → Hệ phân tán không thể đồng thời có linearizable-consistency, total-availability, partition-tolerance; khi partition → chọn C hay A.
- **(bẫy)** "Chọn 2 trong 3" đúng không? → Gây hiểu lầm. P không bỏ được (partition sẽ xảy ra) ⇒ thật ra chọn **CP vs AP** *lúc partition*. "CA" chỉ cho hệ đơn node.
- **(bẫy)** C trong CAP có phải C trong ACID? → Không. CAP-C = linearizability (đồng nhất giữa replica); ACID-C = ràng buộc toàn vẹn của transaction.
- **(TB)** Cache thường CP hay AP? Vì sao? → **AP/EL**: phục vụ nhanh + chấp nhận stale để giữ availability/latency; đó là lý do cache tồn tại.
- **(TB)** Cho 2 hệ cần CP và 2 hệ hợp AP. → CP: etcd/Zookeeper (lock/config), DB ghi tiền. AP: Cassandra (đếm/feed), DNS, cache.
- **(khó)** Spanner có phá CAP không? → Không; nó là **CP**, chỉ đầu tư hạ tầng (TrueTime + mạng) để partition cực hiếm → A gần như không phải trả giá.
- **(khó)** Vì sao không nên dùng Redis làm khoá phân tán đúng-tuyệt-đối? → Redis AP, mất ghi lúc failover/split-brain; cần store CP + **fencing token**.
- **(khó)** PACELC bổ sung gì cho CAP? → Đánh đổi *lúc không partition* (Latency vs Consistency) — phần cache sống ở đó 99% thời gian.

> 🧩 **Thách đố:** *"Hệ bạn dùng một cache phân tán làm khoá để chống xử lý đơn hàng trùng. Thỉnh thoảng đơn vẫn bị xử lý 2 lần. Vì sao và sửa thế nào?"*
> → Cache (AP) **mất khoá lúc failover/split-brain** → hai worker cùng tưởng giữ khoá. CAP nói rõ: khoá cần **CP**. Sửa: chuyển khoá sang store **CP** (etcd/Zookeeper, consensus) **+ fencing token** để worker stale bị storage từ chối; và làm bước xử lý **idempotent** để dù trùng cũng vô hại.

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Cho 6 đường dữ liệu (đếm view, trừ số dư ví, đọc giá hiển thị, giữ chỗ tồn kho lúc checkout, bầu leader của job scheduler, cache asset CDN). Gán **CP/AP** + nêu hành vi lúc partition cho mỗi cái.
> ✅ **Đạt khi:** đếm/giá-hiển-thị/CDN→AP (serve-stale); trừ-ví/checkout/leader→CP (từ chối phía thiểu số); giải thích đúng hành vi partition.

**BT2.** Giải thích vì sao "hệ X là CA" là phát biểu đáng nghi với một hệ phân tán, và viết lại cho đúng.
> ✅ **Đạt khi:** chỉ ra P không bỏ được; "CA" chỉ hợp hệ đơn node; viết lại dạng "lúc bình thường có cả C&A, lúc partition là CP hoặc AP".

**BT3.** Thiết kế khoá phân tán an toàn cho việc xử lý đơn hàng. Vẽ vai trò của store CP, fencing token, và idempotency.
> ✅ **Đạt khi:** khoá ở store CP (consensus); mỗi lần cấp khoá kèm token tăng dần; storage từ chối token cũ; bước xử lý idempotent.

**BT4.** Phân biệt CAP và FLP bằng lời. Vì sao Raft "chạy được" dù FLP nói consensus là bất khả thi trong mạng async?
> ✅ **Đạt khi:** CAP về C/A lúc partition; FLP về *khả năng đạt đồng thuận có đảm bảo kết thúc* trong mạng async; Raft dùng timeout (giả định phần đồng bộ) để đảm bảo tiến triển trên thực tế.

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **CAP theorem** | Lúc partition, chọn C hay A | *"Cache phân tán là quyết định CAP — và nó chọn A."* |
| **Partition (P)** | Mạng giữa node bị đứt/mất gói | *"P không bỏ được — chỉ chọn làm gì khi nó xảy ra."* |
| **Consistency (C, CAP)** | Linearizability — đọc thấy ghi mới nhất | *"Path tiền cần C → chọn CP."* |
| **Availability (A)** | Node sống luôn trả lời (không lỗi) | *"Cache chọn A → serve-stale lúc nguồn đứt."* |
| **CP** | Lúc partition, hi sinh A để giữ C | *"etcd là CP: phía thiểu số ngừng phục vụ."* |
| **AP** | Lúc partition, hi sinh C để giữ A | *"Cassandra/Redis/DNS là AP."* |
| **PACELC** | Else-Latency-vs-Consistency ngoài CAP | *"Cache là PA/EL — ưu tiên latency cả lúc bình thường."* |
| **Linearizability** | Nhất quán thời-gian-thực cho 1 object | *"C của CAP là linearizability, không phải ACID-C."* |
| **Consensus (Raft/Paxos/Zab)** | Đồng thuận đa số cho lõi CP | *"Dùng Raft cho lock/leader/config."* |
| **Quorum (R/W/N)** | Núm vặn CP↔AP định lượng | *"R+W>N nghiêng CP; ≤N nghiêng AP."* |
| **Fencing token** | Số tăng dần chặn leader stale | *"Token cũ bị storage từ chối → hết split-brain phá data."* |
| **Split-brain** | Hai phía cùng tưởng mình là master | *"Redis failover → split-brain → mất ghi."* |
| **Harvest / Yield** | Trả lời một phần thay vì sập | *"Bỏ shard chết, trả kết quả gần đủ (harvest thấp)."* |
| **TrueTime (Spanner)** | Đồng hồ bounded-uncertainty | *"Spanner CP nhưng làm P hiếm nhờ TrueTime."* |
| **FLP impossibility** | Consensus async không đảm bảo kết thúc | *"Raft lách FLP bằng timeout."* |

---

## ⑱ Đọc thêm

- **Gilbert & Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services" (2002)** — chứng minh hình thức.
- **Brewer, "CAP Twelve Years Later: How the 'Rules' Have Changed" (2012)** — chính tác giả đính chính "2/3".
- **Brewer, "Spanner, TrueTime and the CAP Theorem" (2017)** — vì sao Spanner là CP mà "như CA".
- **Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (PACELC)**.
- **Martin Kleppmann, "A Critique of the CAP Theorem"** & **"Please stop calling databases CP or AP"** — vì sao nên quyết theo path, không gắn nhãn.
- **Kleppmann, "How to do distributed locking"** — fencing token, Redlock.
- **Fischer, Lynch, Paterson, "Impossibility of Distributed Consensus…" (1985)** — FLP.
- **Fox & Brewer, "Harvest, Yield, and Scalable Tolerant Systems" (1999)**.
- **DeCandia et al., "Dynamo" (SOSP 2007)** — AP thực chiến.
- **Kleppmann, *Designing Data-Intensive Applications*** — chương Consistency & Consensus (bản đồ toàn cảnh).

---

> 🔗 **Nối lại cả cụm (5 tài liệu):** Bài 1 dựng khung *tốc độ ↔ tươi ↔ RAM*. **CAP là khung lý thuyết** giải thích vì sao trục "tươi" tồn tại: khi có khoảng cách + partition, bạn **buộc** phải chọn. **Eventual Consistency** là tên của lựa chọn **AP**; **Strong Consistency (B1)** là lựa chọn **CP**. **Cold Start** và **Working Set** là chuyện *vận hành* cái vành AP đó cho hiệu quả. Câu chốt staff engineer: ***đừng hỏi "hệ tôi CP hay AP", hãy hỏi "path NÀY cần gì, và lúc mạng đứt nó nên trả lời hay im lặng" — rồi đặt lõi CP nhỏ bên cạnh vành AP lớn.***
