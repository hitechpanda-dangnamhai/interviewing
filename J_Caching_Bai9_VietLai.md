# 🎯 BÀI 9 — Redis Scaling (Replica · Cluster · Sentinel · RediSearch)
**Phủ:** J-CLUSTER-001 → J-CLUSTER-006

---

## ① Mục tiêu & vị trí trong mạch

Bài 7 nói: **execution của Redis ~1 core**. Bài này trả lời câu hỏi kế tiếp: **khi một node hết đủ, scale thế nào?**

- **Replica** → scale **đọc** + **HA**.
- **Cluster / hash slots** → scale **ghi** + **dung lượng**.
- **Sentinel** → **failover** (bầu master mới khi master chết).

Cộng thêm: **giới hạn multi-key trong cluster**, **hệ quả của replication async** (cache chịu được nhưng lock thì nguy hiểm), và **RediSearch / modules**.

> 💡 **Ý chính:** Trước khi hỏi *"scale bằng cách nào"*, phải hỏi *"nghẽn ở ĐÂU"* — RAM, CPU, hay network? **Mỗi nghẽn có một hướng scale khác nhau.** Thêm replica để cứu nghẽn RAM ghi là vô ích.

---

## ② Giảng cơ bản → nâng cao

### (a) Khi nào scale & các hướng *(J-CLUSTER-001)*

**Một node "hết đủ" khi:** **vượt RAM một node**, hoặc **throughput > ~1 core / băng thông**.

> 🎯 **Bước 0 BẮT BUỘC — nhận diện nghẽn:** là **memory** (hết RAM), **CPU** (hết ~1 core execution), hay **network** (hết băng thông)? Chẩn đoán sai → chữa sai.

Ba hướng scale:

| Hướng | Giải quyết | Cơ chế |
|---|---|---|
| **Replica** (master-replica) | scale **đọc** + **HA** | đọc từ **replica** |
| **Cluster / sharding** | scale **ghi** + **dung lượng** | chia data ra **nhiều node** |
| **Client-side sharding** | scale ghi/dung lượng | **client tự băm key** tới node |

**Ẩn dụ:** một quán ăn quá tải.
- **Replica** = thuê thêm **người bưng bê đọc thực đơn** cho khách (san tải *đọc*), nhưng **vẫn một bếp**.
- **Cluster** = mở thêm **nhiều bếp**, mỗi bếp lo một nhóm món (san tải *ghi* + *chứa* nhiều hơn).

---

### (b) Redis Cluster — hash slots *(J-CLUSTER-002)*

> Redis Cluster chia không gian khoá thành **16384 hash slot**. Một key được map: **`CRC16(key) mod 16384` → slot → node giữ slot đó**.

**Vì sao là slot chứ không map thẳng key → node?** Vì **reshard** (chia lại) chỉ cần **di chuyển slot giữa các node**, không phải băm lại từng key. Slot là "đơn vị di chuyển" gọn → **kiểm soát phân bố dễ hơn**. *(verify số slot/cơ chế tại redis.io)*

**Ẩn dụ:** thay vì gán mỗi lá thư cho một bưu tá (đổi bưu tá là loạn), bạn chia thành **16384 mã vùng**; mỗi bưu tá phụ trách một dải mã vùng. Thêm/bớt bưu tá = chỉ **chuyển vài mã vùng**, thư không cần đánh lại địa chỉ.

---

### (c) Multi-key trong Cluster & hash tag *(J-CLUSTER-003)*

> 🚨 **Giới hạn cốt lõi:** lệnh đụng **nhiều key** phải nằm **CÙNG slot/node** (**không cross-slot**). → **`MGET` / transaction / Lua đa-key bị giới hạn** trong cluster.

**Vì sao?** Vì mỗi node chỉ giữ một phần slot. Một lệnh đa-key mà các key rải ở các node khác nhau thì *không node nào* thực hiện trọn vẹn được → Redis **từ chối** với lỗi **CROSSSLOT**.

**Giải — hash tag `{...}`:** ép các key liên quan vào **cùng slot** bằng cách chỉ băm **phần trong dấu `{}`**:
```
user:{42}:profile   ┐  chỉ "42" được băm
user:{42}:settings  ┘  → cùng slot → dùng được lệnh đa-key
```

> 🎯 **Hệ quả thiết kế:** **key phải tính trước** — nhóm nào cần thao tác atomic chung thì phải **cùng hash tag** ngay từ lúc thiết kế. *(verify)*

---

### (d) Sentinel vs Cluster *(J-CLUSTER-004)*

Hai thứ giải **hai bài toán khác nhau** — đừng lẫn:

| | **Sentinel** | **Cluster** |
|---|---|---|
| **Giải bài toán** | **HA / failover** cho master-replica (**KHÔNG shard**) | **Sharding + HA** cho data vượt 1 node |
| **Cơ chế** | **Bầu master mới** khi master chết | Chia **16384 slot** + failover **per-shard** |
| **Chọn khi** | Chỉ cần **failover**, data **vừa 1 node** | Cần **shard** (data/ghi vượt 1 node) |

*(verify)*

> 🎯 **Một câu phân biệt:** **Sentinel = người gác** (master chết thì cử người thay, nhưng vẫn *một kho data*). **Cluster = chia kho** (data to quá, xẻ ra nhiều kho, mỗi kho lại có gác riêng).

---

### (e) Replication async — hệ quả *(J-CLUSTER-005)* — ⭐ chỗ phân biệt cache vs lock

> Redis replication là **async** (bất đồng bộ): **write trên master có thể CHƯA kịp sang replica** khi xảy ra **failover** → **mất vài write gần nhất**.

Đây là chỗ **cùng một sự thật kỹ thuật, hai hệ quả trái ngược:**

```
T1: client ghi X=mới lên MASTER → master báo OK
    (X chưa kịp copy sang REPLICA)
T2: MASTER chết → failover → REPLICA lên làm master
    → master mới KHÔNG có X=mới  ❌
─────────────────────────────────────
• Cache: chỉ là 1 lần MISS thêm → đọc lại từ DB → CHẤP NHẬN ĐƯỢC ✅
• Lock : 2 client cùng tưởng mình giữ lock → NGUY HIỂM ❌
```

> 🚨 **Với lock:** sau failover, lock vừa ghi *biến mất* → một client khác chiếm được lock **trong khi client cũ vẫn nghĩ mình đang giữ** → **2 client cùng vào vùng tới hạn**. Đây chính là **caveat Redlock (Kleppmann)**. *(Cross-ref I-DLOCK-011.)* *(verify)*

> 🎯 **Khẩu quyết:** **replication async → cache CHỊU ĐƯỢC, lock thì KHÔNG.**

---

### (f) RediSearch / modules *(J-CLUSTER-006)*

> **Module** (như **RediSearch**) thêm **secondary index / full-text / vector search / aggregation** ngay trên data đã ở Redis → query **phức tạp hơn** so với key-value thuần.

**Khi nào dùng:** cần **search nhanh trên data đang ở Redis**.

> ⚠️ **Đừng lạm dụng:** module **KHÔNG thay full search engine** (**Elasticsearch**) cho nhu cầu search lớn. Redis làm search "vừa phải"; search quy mô lớn vẫn cần engine chuyên dụng. *(verify — Redis 8 đã tích hợp **Redis Query Engine / JSON / etc** vào core dưới tri-license.)*

---

## ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái CŨ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| *"Scale Redis = thêm RAM mãi"* | **Replica** (đọc) + **Cluster** (ghi/dung lượng) | 1 node có **trần RAM/CPU** |
| Multi-key tuỳ tiện trong cluster | **Hash tag `{}`** gom cùng slot | Lệnh đa-key **cần cùng slot** |
| *"Redlock chắc chắn đúng"* | Hiểu **replication async → caveat** (xem I-DLOCK) | Failover **mất write → 2 client giữ lock** |
| Dùng Redis làm **search engine chính** | **RediSearch** cho vừa phải; lớn → **Elasticsearch** | Module **không thay** full search |
| **Sentinel = sharding** | **Sentinel = failover**, **Cluster = sharding** | Hai bài toán **khác nhau** |

---

## ④ Áp dụng thực tế + So sánh bigtech

**Use case:**
- **Đọc nhiều** → thêm **replica** + đọc từ replica (chấp nhận **stale nhẹ** cho cache).
- **Data vượt RAM 1 node** → **Cluster**, thiết kế key có **hash tag** cho nhóm cần cùng slot.
- **Cần failover nhưng data vừa** → **Sentinel**.

**Pattern ở bigtech:**
- **Managed Redis/Valkey** (**ElastiCache / Memorystore**) lo **replica/cluster/failover** giúp.
- Hệ thống cực lớn **shard sớm** + **đọc-từ-replica**; **cẩn thận replication lag** khi đọc replica *(cross-ref E read-from-replica)*. *(verify)*

---

## ⑤ Code thực hành + cấu hình

```javascript
import Redis from "ioredis";

// Redis Cluster client — chỉ cần vài node "seed", client tự khám phá topology
const cluster = new Redis.Cluster([
  { host: "10.0.0.1", port: 6379 },
  { host: "10.0.0.2", port: 6379 },
]);

// Hash tag: ép CÙNG slot để dùng lệnh đa-key / transaction
await cluster.mset(
  "user:{42}:profile",  JSON.stringify(profile),  // {42} quyết định slot
  "user:{42}:settings", JSON.stringify(settings), // cùng slot → MSET hợp lệ ✅
);
// "user:43:profile" + "user:42:profile" KHÁC slot → MGET cross-slot sẽ LỖI ❌
```

```bash
redis-cli --cluster check 10.0.0.1:6379   # kiểm tra slot coverage (verify)
redis-cli CLUSTER NODES | head
```

> ⚠️ **Đừng dựa replica** cho đọc **cần đúng tuyệt đối** (có **lag**).
> ⚠️ **Lock qua Redis cluster:** đọc kỹ **I-DLOCK** (**replication async**).

---

## ⑥ Keywords cần nhớ (glossary)

🧠 **Cần HIỂU:**
- **Nhận diện nghẽn TRƯỚC khi scale**.
- **replica (đọc/HA) vs cluster (ghi/dung lượng) vs sentinel (failover)**.
- **replication async → cache ok / lock nguy hiểm**.
- **module không thay search engine**.

📌 **Cần THUỘC:**
- **16384 hash slot**, **CRC16 mod**.
- **hash tag `{}`** gom slot.
- **Sentinel = failover, Cluster = shard**.
- **multi-key cần cùng slot**.

🛠️ **Cần LÀM ĐƯỢC:** thiết kế **key có hash tag**; chọn **replica/cluster/sentinel** theo nhu cầu; cấu hình **cluster client**.

---

## ⑦ Mental model

> **1 node hết** → **replica** (đọc/HA), **cluster** (ghi/dung lượng, **16384 slot / hash tag**), **sentinel** (failover).
> **Replication ASYNC:** **cache chịu được, lock thì KHÔNG** (xem **I-DLOCK**).

---

## ⑧ Câu hỏi phỏng vấn & thách đố

- **(TB)** Khi nào 1 node hết đủ, các hướng scale? → vượt **RAM/throughput**; **replica/cluster/client-shard**; **nhận diện nghẽn trước**.
- **(khó)** Redis Cluster chia data thế nào? → **16384 hash slot**, **CRC16 mod**, node giữ slot.
- **(khó)** Multi-key trong cluster bị giới hạn gì, hash tag giải sao? → phải **cùng slot**; **`{}` ép cùng slot**.
- **(TB)** Sentinel vs Cluster khác bài toán gì? → **failover (không shard)** vs **sharding + HA**.
- **(rất khó)** Replication async ảnh hưởng cache vs lock thế nào? → cache **chấp nhận mất write**; lock → **2 client giữ → nguy hiểm**.

> **🧩 Thách đố:** *"Bạn `MGET user:1:a user:2:b` trên Redis Cluster và bị lỗi **CROSSSLOT**. Vì sao, fix sao mà VẪN shard được?"*
>
> **Trả lời:** `user:1:a` và `user:2:b` **băm ra slot khác nhau** → lệnh đa-key **cross-slot bị từ chối**. **Fix:** **đừng ép mọi thứ về 1 slot** (sẽ mất shard); thay vào đó **gom theo entity bằng hash tag** *khi thực sự cần atomic đa-key* (`user:{1}:a`, `user:{1}:b`). Còn **lookup nhiều user khác nhau** thì dùng **pipeline nhiều `GET`** thay vì một `MGET`.

---

## ⑨ Bài tập + tiêu chí tự chấm

**BT1.** Thiết kế key cho **"giỏ hàng + tổng tiền"** của 1 user cần cập nhật **atomic** trong cluster.
> ✅ **Đạt khi:** dùng **hash tag** `cart:{userId}:items` + `cart:{userId}:total` → **cùng slot**.

**BT2.** Quyết định **Sentinel hay Cluster** cho: **(a)** 5GB data, cần failover; **(b)** 200GB data.
> ✅ **Đạt khi:** **(a) Sentinel**, **(b) Cluster** + lý do **dung lượng**.

---

## ⑩ Đọc thêm

- **redis.io/docs** — *Scale with Redis Cluster, High availability with Sentinel, Replication*.
- **redis.io** — *Redis Query Engine (RediSearch), Redis 8 modules*.
- **Kleppmann** — *How to do distributed locking* (replication caveat; mục I).

---

> 🔎 **AI hay sai ở đây:** AI viết **`MGET` / transaction đa-key** trên cluster **không nghĩ tới slot** → **CROSSSLOT lỗi runtime**.
> **Luôn tự kiểm:** *các key trong một lệnh đa-key có CÙNG slot (hash tag) không?*
