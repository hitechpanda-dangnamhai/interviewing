# 🔗 CHUYÊN ĐỀ — CACHE COHERENCE (Tính nhất quán giữa các bản cache)
> Đào sâu một concept mở rộng của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu *coherence là gì* (và khác *consistency* ra sao), từ **MESI phần cứng** tới **pub/sub phân tán**, biết *chọn cơ chế* và *tránh false sharing / invalidation storm* ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Cache coherence** xuất hiện ở **bảng concept mở rộng (B2)**:

> ➕ **Cache coherence** — *Giữ nhiều bản sao đồng nhất với nhau* — *"L1 ở mỗi node lệch nhau → vấn đề **coherence**, xử bằng pub/sub."*

Và Bài 1 mục (d) đã nêu đúng vấn đề thực tế:

> *"Vì L1 của mỗi node là độc lập, khi dữ liệu ở L2/DB thay đổi, các bản L1 ở những node khác nhau có thể stale khác nhau (node A đã cập nhật, node B vẫn giữ đồ cũ)."* → xử bằng **TTL ngắn** hoặc **pub/sub broadcast invalidate**.

Chuyên đề này "mở hộp" khái niệm đó. Học xong bạn sẽ:

- Phân biệt **coherence vs consistency** — hai thứ hay bị lẫn nhất trong hệ phân tán.
- Hiểu **gốc phần cứng** (MESI, snooping vs directory, false sharing) — chuẩn vàng của coherence.
- Hiểu **distributed cache coherence** chỉ là **bản xấp xỉ rẻ tiền** của coherence phần cứng (TTL, pub/sub, versioning, CDC).
- Nắm các pattern staff-level: **leases**, **directory/callback**, **CDC invalidation pipeline**, **chống invalidation storm & false sharing**.

> 💡 **Một câu để gắn vào mạch Bài 1:** Khi bạn dùng **multi-tier (L1 mỗi node + L2 chung)**, bạn tạo ra **nhiều bản sao của cùng một dữ liệu**. Coherence là **cơ chế giữ những bản sao đó không lệch nhau quá lâu**. Phần cứng làm việc này *mạnh và tự động*; phần mềm phân tán thì *yếu hơn và phải tự xây* — và đó chính là lý do có pub/sub invalidate, TTL ngắn, versioning.

---

## ② Trực giác — Coherence là gì? Và ĐỪNG nhầm với Consistency

### Định nghĩa

**Cache coherence = mọi bản sao của CÙNG MỘT mục dữ liệu, ở các cache khác nhau, đều thống nhất về giá trị (cuối cùng / tức thời) của mục đó.**

Nói cách khác: nếu cùng một key `x` nằm trong L1 của node A, L1 của node B và L2 chung, thì coherence lo việc **chúng không kể ba câu chuyện khác nhau về `x`**.

### ⚠️ Coherence ≠ Consistency (phân biệt then chốt)

Đây là chỗ lẫn lộn phổ biến nhất. Hai khái niệm **khác cấp độ**:

| | **Coherence (nhất quán bản sao)** | **Consistency (mô hình nhất quán / memory model)** |
|---|---|---|
| Về cái gì | **MỘT mục** dữ liệu, nhiều bản sao | **THỨ TỰ** các thao tác trên **NHIỀU mục** khác nhau |
| Câu hỏi | "Các bản của `x` có khớp nhau không?" | "Ghi `x` rồi ghi `y` — node khác có thấy đúng thứ tự đó không?" |
| Đảm bảo | Mọi đọc `x` cuối cùng thấy ghi `x` mới nhất | Linearizable / causal / eventual… (đã học ở chuyên đề trước) |
| Ví dụ vi phạm | L1 node B vẫn giữ `x` cũ sau khi `x` đổi | Node thấy `y=2` nhưng vẫn thấy `x=cũ` dù `x` được ghi trước `y` |

> 🔑 **Insight #1:** **Coherence là về một-ô-nhớ; consistency là về thứ-tự-giữa-nhiều-ô-nhớ.** Một hệ có thể **coherent** (mỗi ô cuối cùng đồng nhất) mà vẫn có **memory model yếu** (thứ tự giữa các ô lộn xộn). Coherence là *điều kiện cần* nhưng *không đủ* cho strong consistency. Khi Bài 1 nói "L1 lệch nhau → vấn đề coherence", nó đang nói đúng về *một key*; còn "đọc thấy ghi mới nhất" (B1) là chuyện *consistency*.

### Ẩn dụ nhiều cuốn sổ tay

Bài 1 dùng ẩn dụ "tờ giấy dán màn hình". Giờ tưởng tượng **cả phòng, mỗi người một tờ giấy** chép cùng một con số từ thư viện.

- **Coherence** = lo việc *khi con số trong sách đổi*, mọi tờ giấy trong phòng được cập nhật / xé bỏ, không ai ôm số cũ mãi.
- **Cơ chế coherence** = cách bạn báo cho cả phòng: hét lên (pub/sub broadcast), hoặc hẹn "cứ 5 phút tự đi kiểm tra lại" (TTL), hoặc "ai sửa sách phải gọi điện cho từng người đã mượn" (directory/callback).

---

## ③ Gốc PHẦN CỨNG — chuẩn vàng của coherence (MESI)

Coherence ra đời ở **CPU đa nhân**: mỗi core có **L1/L2 riêng**. Nếu core 0 ghi `x=1` vào cache của nó mà core 1 vẫn đọc `x=0` từ cache của mình → chương trình **sai**. Phần cứng *bắt buộc* phải giữ coherence — và nó làm rất **mạnh, đồng bộ, tự động**.

### Hai bất biến coherence (Sorin/Hill/Wood)

1. **Single-Writer–Multiple-Reader (SWMR):** tại một thời điểm, **hoặc** một core được ghi `x`, **hoặc** nhiều core cùng đọc `x` — không bao giờ vừa-ghi-vừa-đọc-loạn.
2. **Data-Value:** một lần đọc `x` trả về **giá trị của lần ghi `x` mới nhất**.

### Giao thức MESI — 4 trạng thái của mỗi cache line

| Trạng thái | Nghĩa |
|---|---|
| **M — Modified** | Bản DUY NHẤT, đã sửa (bẩn, khác bộ nhớ chính). Phải ghi xuống trước khi ai khác đọc. |
| **E — Exclusive** | Bản DUY NHẤT, sạch (khớp bộ nhớ). Có thể ghi mà không cần báo ai. |
| **S — Shared** | CÓ THỂ nằm ở nhiều cache, sạch, chỉ đọc. |
| **I — Invalid** | Vô hiệu — không được dùng, phải nạp lại. |

**Quy tắc cốt lõi (invalidation-based):** khi một core muốn **GHI** một line đang **Shared**, nó **phát tín hiệu invalidate** → mọi bản ở cache khác chuyển sang **I** → core đó độc quyền (M) rồi mới ghi. Đọc lại ở core khác = miss → nạp bản mới.

> Biến thể: **MOESI** (thêm **O**wned — AMD) và **MESIF** (thêm **F**orward — Intel) tối ưu để giảm lưu lượng đọc bộ nhớ chính.

### Hai kiểu "nghe ngóng"

| Cơ chế | Cách hoạt động | Quy mô |
|---|---|---|
| **Snooping (bus)** | Mọi cache **rình** một bus chung, thấy ai ghi thì tự invalidate | Tốt cho ít core; **không scale** khi nhiều core (bus nghẽn) |
| **Directory-based** | Một **danh bạ** ghi rõ "line này đang ở những cache nào", chỉ báo cho đúng nơi giữ | Scale cho **nhiều core / NUMA** lớn |

> 🔑 **Insight #2:** Snooping (broadcast cho tất cả) ↔ directory (chỉ báo nơi giữ) là **đúng cùng một cặp đánh đổi** xuất hiện lại ở tầng phân tán: **pub/sub broadcast** (như snooping) ↔ **theo dõi ai giữ key rồi báo riêng** (như directory/callback). Học một lần ở phần cứng, áp lại được ở hệ phân tán.

### Cạm bẫy phần cứng: FALSE SHARING (chia sẻ giả) — nối với spatial locality (B1)

Đơn vị coherence là **cache line** (thường **64 byte**), không phải từng biến. Nếu **hai biến độc lập** nằm tình cờ **cùng một line**, thì hai core sửa hai biến đó vẫn **giành nhau cả line** → invalidate qua lại liên tục (**ping-pong**) → chậm thảm hại, dù logic chẳng liên quan.

```
Cache line 64B:  [ counterA | counterB | ... ]
Core 0 tăng counterA  → invalidate line ở Core 1
Core 1 tăng counterB  → invalidate line ở Core 0
→ line bị giật qua giật lại mỗi lần tăng, dù A và B chẳng dính gì nhau
```

> 🔑 **Insight #3:** **Spatial locality (B1) là con dao hai lưỡi.** Nó giúp cache (đọc dòng 1 → có sẵn dòng 2,3), nhưng cũng đẻ ra **false sharing** khi ghi đồng thời. Sửa bằng **padding/căn lề** để hai biến nóng nằm khác line (`@Contended` trong Java, `alignas(64)` trong C++).

---

## ④ Distributed cache coherence — bản XẤP XỈ rẻ tiền của MESI

Ở tầng app, ta cũng có "nhiều bản của một key" (L1 mỗi node + L2 chung). Nhưng ta **không thể** bê MESI nguyên xi: phần cứng coherence dựa vào **khoảng cách ngắn + bus/đồng bộ nhanh**; còn giữa các node là **network (chậm, có partition)** — giữ coherence *đồng bộ, tức thời* sẽ giết chết tốc độ (chính lý do ta cache!).

⇒ Ta **xấp xỉ** coherence bằng các cơ chế yếu hơn, theo độ chặt tăng dần:

| Cơ chế | Tương ứng phần cứng | Độ chặt | Đánh đổi |
|---|---|---|---|
| **TTL ngắn** (Bài 1) | "tự kiểm tra định kỳ" | Yếu — cửa sổ lệch = TTL | Rẻ nhất, không cần báo nhau; lệch trong TTL |
| **Pub/sub invalidate** (Bài 1) | Snooping (broadcast) | Khá — hội tụ nhanh | Cần message bus; **best-effort, có thể mất tin** |
| **Versioning / CAS** | Data-Value invariant | Chặn ghi-đè-đồ-cũ | Cần lưu version, so sánh |
| **Directory / callback (lease)** | Directory-based | Chặt | Phức tạp, server phải theo dõi ai giữ |
| **CDC invalidation pipeline** | Snooping từ "nguồn sự thật" | Đáng tin (không sót) | Hạ tầng binlog→bus |
| **Bỏ L1, chỉ dùng L2 chung** | — | Mạnh nhất (1 bản) | Mất tốc độ L1, +network hop |

> 🔑 **Insight #4:** Distributed cache coherence là **một phổ**, không phải bật/tắt. Bạn chọn **mức chặt vừa đủ cho path** (giống chọn nấc nhất quán ở chuyên đề Eventual Consistency). Like-count: TTL là đủ. Profile của chính user: cần pub/sub + versioning. Cấu hình feature-flag tác động tới quyền: gần như cần CDC + TTL backstop.

> 🔑 **Insight #5 (nối CAP):** Pub/sub invalidate là **nỗ lực kéo một hệ AP (L1 các node) về gần linearizable**. Nhưng vì network có partition và pub/sub best-effort, bạn **không bao giờ đạt coherence tuyệt đối như phần cứng** — nên **luôn cần TTL backstop** làm lưới an toàn cho lúc message bị mất.

---

## ⑤ Ví dụ minh hoạ

### Ví dụ 1 — L1 hai node lệch nhau (đúng kịch bản Bài 1 (d))

```
Node A (L1: user:42 = "Alan")        Node B (L1: user:42 = "Alan")
   │                                     │
User đổi tên → DB = "Alice", L2 xoá     │
A xử lý request đổi tên → A xoá L1 A    │  (B KHÔNG biết gì)
   │                                     │
A đọc lại → miss → nạp "Alice" ✅        B đọc → HIT L1 cũ → trả "Alan" ❌
```
→ **Incoherent** giữa A và B. Hai cách Bài 1 nêu: **TTL ngắn ở L1** (B tự hết hạn sau vài giây) hoặc **pub/sub** (A phát "invalidate user:42" → B xoá ngay).

### Ví dụ 2 — Code: pub/sub invalidate + TTL backstop

```js
import Redis from "ioredis";
import { LRUCache } from "lru-cache";

const redis = new Redis(process.env.REDIS_URL);
const sub   = new Redis(process.env.REDIS_URL);     // kết nối riêng để subscribe
const L1 = new LRUCache({ max: 10_000, ttl: 5_000 }); // ⬅ TTL 5s = LƯỚI AN TOÀN nếu mất tin

// Mọi node lắng nghe kênh invalidate
sub.subscribe("cache:invalidate");
sub.on("message", (_ch, key) => L1.delete(key));      // ⬅ coherence kiểu "snooping"

export async function invalidate(key) {
  await redis.del(key);                               // xoá L2 (nguồn chia sẻ)
  await redis.publish("cache:invalidate", key);       // báo MỌI node xoá L1
  L1.delete(key);                                     // xoá L1 của chính mình
}

export async function get(key, loadFromDb) {
  const local = L1.get(key);
  if (local !== undefined) return local;
  const shared = await redis.get(key);
  if (shared !== null) { const v = JSON.parse(shared); L1.set(key, v); return v; }
  const v = await loadFromDb(key);
  if (v != null) { await redis.set(key, JSON.stringify(v), "EX", 300); L1.set(key, v); }
  return v;
}
```

> Hai trụ cột coherence ở đây: **pub/sub** (hội tụ nhanh khi tin tới) **+ TTL 5s** (chặn lệch vĩnh viễn nếu tin pub/sub *mất* — điều chắc chắn xảy ra đôi lúc).

### Ví dụ 3 — False sharing (phần cứng) trong code đa luồng

```java
// ❌ Hai counter cạnh nhau → cùng cache line → ping-pong giữa 2 thread
class Counters { volatile long a; volatile long b; }

// ✅ Padding để a và b nằm KHÁC cache line
class Counters {
    @jdk.internal.vm.annotation.Contended volatile long a;
    @jdk.internal.vm.annotation.Contended volatile long b;
}
```
Throughput có thể tăng *nhiều lần* chỉ nhờ tách line — một lỗi coherence vô hình mà profiler hay bỏ sót.

### Ví dụ 4 — Invalidation storm (nối Cold Start)

Một hot key `homepage:banner` nằm ở L1 của **cả 200 node**. Bạn invalidate nó → **200 node cùng miss đồng loạt** → 200 cú đọc L2/DB cùng lúc = **stampede** (chính là cold start cục bộ cho một key). ⇒ coherence "đúng" nhưng vẫn cần **singleflight/coalescing** để không đập nguồn (xem chuyên đề Cold Start).

---

## ⑥ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/bài báo công khai và pattern phổ biến. Verify số liệu cụ thể khi cần trích dẫn.

### CS1 — Facebook Memcached: invalidation pipeline từ binlog (mcsqueal, NSDI 2013)

Facebook giữ coherence cho hàng nghìn Memcached server **xuyên vùng** bằng cách: các daemon **đọc commit log (binlog) của MySQL**, suy ra những key cần xoá, rồi **phát các lệnh delete** tới các tier cache. Vì invalidation **bắt nguồn từ chính nguồn sự thật (DB)** thay vì trông chờ app nhớ xoá, nó **không bị sót** — kể cả khi nhiều đường ghi khác nhau.

> 🎯 **Bài học:** Đây là **CDC-based coherence** ở quy mô khổng lồ. Pattern vàng: **đừng tin app invalidate ở mọi chỗ ghi (dễ quên) — bắt thay đổi từ binlog và phát invalidate**. (Cùng kỹ thuật CDC ở chuyên đề Eventual Consistency.)

### CS2 — Andrew File System (AFS): coherence kiểu directory/callback

AFS cho client cache file, và **server giữ "callback promise"**: nếu file bị sửa, server **chủ động gọi lại (callback)** mọi client đang cache để báo vô hiệu. Đây là **directory-based coherence qua mạng** — server *biết ai đang giữ gì* và *báo đúng nơi*, thay vì broadcast mù.

> 🎯 **Bài học:** Khi cần coherence chặt hơn pub/sub best-effort, **theo dõi người giữ (directory) + callback** cho độ tin cao hơn — đổi lại server phải lưu trạng thái "ai giữ gì".

### CS3 — Leases (Gray & Cheriton, 1989): cầu nối TTL ⟷ callback

**Lease = một lời hứa có thời hạn rằng dữ liệu sẽ không đổi trong khoảng đó.** Trong thời hạn lease, client cache **coherent mà không cần hỏi lại**; muốn ghi, writer phải **đợi lease hết** hoặc **thu hồi (revoke)** nó. **TTL chính là một lease suy biến** (lease không thể thu hồi). Lease cho phép coherence chặt mà vẫn chịu lỗi (nếu mất liên lạc, chỉ cần đợi lease hết hạn).

> 🎯 **Bài học:** TTL và callback **không phải hai phe** — chúng là hai đầu của *cùng một ý tưởng lease*. Chọn thời hạn lease = chọn cửa sổ incoherence tối đa.

### CS4 — CDN coherence: purge lan truyền + tiered

Đổi một asset → **purge** phải lan tới mọi edge. Vì hàng trăm edge, coherence là **eventual + tiered** (origin shield làm tầng trung gian). "Instant purge" rút cửa sổ xuống mili-giây nhưng vẫn là *lan truyền*, không tức thời tuyệt đối.

### CS5 — HTTP revalidation (ETag) — coherence kiểu PULL (Bài 1 C4)

Trình duyệt cache với **ETag**. Lần sau nó hỏi server *"bản tôi giữ (ETag abc) còn mới không?"* → server trả **304 Not Modified** (còn coherent) hoặc bản mới. Đây là **coherence do người đọc chủ động kiểm (pull/validate)**, ngược với pub/sub (push). (Bài 1 C4 đã có ETag/Last-Modified.)

> 🎯 **Bài học:** Có **hai hướng** giữ coherence: **PUSH** (nguồn báo khi đổi — pub/sub, callback) và **PULL** (người đọc hỏi "còn mới không?" — ETag, revalidate). Push nhanh nhưng cần biết ai giữ; pull đơn giản nhưng tốn một vòng hỏi.

---

## ⑦ Giải pháp staff engineer — chọn cơ chế coherence theo path

### Khung chọn nhanh (theo độ tươi cần & chi phí chấp nhận)

| Path | Cần coherence tới đâu | Cơ chế đề xuất |
|---|---|---|
| Đếm/feed/trending | Lỏng (lệch vài giây ok) | **TTL** đơn thuần |
| Profile/catalog đọc nhiều, đổi thưa | Vừa | **TTL + pub/sub invalidate** |
| Dữ liệu user-tự-ghi | Người ghi phải thấy ngay | + **versioning (read-your-writes)** |
| Quyền/feature-flag/cấu hình ảnh hưởng bảo mật | Chặt, không được sót | **CDC pipeline + TTL backstop** (CS1) |
| Tiền/tồn kho-checkout | Tuyệt đối | **Bỏ cache cho path quyết định** (đọc nguồn/CP) |

### Bốn quy tắc vàng

1. **Luôn có TTL backstop** kể cả khi đã có pub/sub — vì pub/sub **best-effort, có thể mất tin** (Insight #5). TTL là lưới an toàn cho coherence.
2. **Thứ tự invalidate đúng:** ghi nguồn **trước**, invalidate **sau**; và chống **stale-set race** bằng **versioning/CAS** (đã học ở chuyên đề Eventual Consistency — coherence và consistency gặp nhau ở đây).
3. **Invalidate đáng tin → CDC**, đừng rải `cache.del()` khắp app (sẽ quên một đường ghi → incoherent lén lút).
4. **Hot key → kèm chống stampede** (singleflight) khi invalidate, để coherence không đẻ ra invalidation storm (Ví dụ 4).

### Khi nào BỎ L1 cho gọn

Nếu một loại dữ liệu **đổi thường xuyên + cần coherent chặt**, chi phí giữ L1 các node đồng bộ có thể **lớn hơn lợi ích tốc độ**. Lúc đó: **bỏ L1, chỉ dùng L2 chung** (một bản → coherent tự nhiên), chấp nhận thêm 1 network hop. *Đừng* cố nhồi L1 cho mọi thứ.

---

## ⑧ Giải pháp NÂNG CAO (staff / principal)

### A1. Directory/callback cho coherence chặt (CS2)

Theo dõi **node nào đang giữ key nào** (một "danh bạ"), khi key đổi thì **báo đúng các node giữ** thay vì broadcast cho tất cả. Giảm lưu lượng invalidate ở hệ rất nhiều node + tăng độ tin (biết chắc đã báo ai). Đổi lại: phải lưu & cập nhật danh bạ (giống directory-based MESI).

### A2. Leases có thể thu hồi (CS3)

Cấp lease cho bản L1: trong hạn, node tin bản của mình coherent; muốn ghi, nguồn **revoke** lease (hoặc đợi hết hạn). Cho phép **coherence mạnh mà vẫn chịu lỗi** (mất liên lạc → tự hết hạn). Là nền của nhiều hệ file/cache phân tán.

### A3. Reliable invalidation: CDC + log có thứ tự + at-least-once

Pub/sub best-effort là điểm yếu. Nâng cấp: phát invalidate qua **log bền (Kafka)** từ **CDC (binlog)**, đảm bảo **at-least-once + theo thứ tự per-key**. Node tiêu thụ idempotent (xoá hai lần vô hại). ⇒ coherence **không sót, đúng thứ tự** — gần mức callback nhưng dễ vận hành hơn ở quy mô lớn (CS1).

### A4. Versioning/generation để coherence + read-your-writes hợp nhất

Gắn **version** cho mỗi key; L1/L2 chỉ chấp nhận bản **version ≥** bản đang giữ. Vừa **chống stale-set race** (consistency) vừa **cho người vừa ghi thấy bản mới** (read-your-writes) vừa giúp **invalidate trễ/loạn thứ tự** không gây hại (bản cũ tới sau bị bỏ qua). Một cơ chế, ba lợi ích.

### A5. Chống invalidation storm (nối Cold Start)

Khi invalidate **hot key trên nhiều node**:
- **Singleflight/coalescing** ở mỗi node để miss trùng key chỉ đi nguồn 1 lần.
- **Back-fill từ L2** thay vì DB (L2 ấm "đỡ" cho L1 vừa bị xoá).
- **Jitter** nếu invalidate hàng loạt key (đừng để mọi node miss đúng cùng mili-giây).

### A6. Chống false sharing ở hot path (phần cứng)

Trong code concurrency nóng (counter, ring buffer, lock-free), **căn lề cache line** (`@Contended`, `alignas(64)`, padding) để biến nóng không chung line. Đo bằng perf counter (`perf c2c` trên Linux phát hiện false sharing). Đây là tối ưu staff-level mà ít người để ý.

### A7. Hiểu memory model để coherence "đủ dùng" đúng cách

Coherence đảm bảo *mỗi ô* hội tụ, nhưng **thứ tự giữa các ô** do **memory model** + **memory barrier** quyết định. Lập trình lock-free cần **happens-before** đúng (acquire/release, `volatile`/`atomic`) — coherence một mình **không** cứu bạn khỏi reordering. (Insight #1 thành code thực.)

---

## ⑨ Trường hợp MỞ RỘNG (coherence ở mọi tầng)

| Bối cảnh | "Bản sao" là gì | Cơ chế coherence |
|---|---|---|
| **CPU đa nhân** | L1/L2 mỗi core | **MESI/MOESI/MESIF** — snooping/directory (mạnh, phần cứng) |
| **NUMA / many-core** | Cache nhiều socket | **Directory-based** (snooping không scale) |
| **CPU ⟷ GPU/accelerator** | Cache rời nhau | **CXL** (Compute Express Link) cho coherent access |
| **Distributed app cache (Bài 1)** | L1 mỗi node + L2 | **TTL + pub/sub + CDC** (xấp xỉ, eventual) |
| **Distributed file system** | File cache ở client | **AFS callback / NFS close-to-open + revalidate** |
| **CDN** | Edge cache | **Purge lan truyền + tiered** (eventual) |
| **Browser** | HTTP cache | **ETag/Last-Modified revalidate** (pull) |
| **Microservices** | Service A cache data của B | Coherence **xuyên ranh giới service** — event/CDC từ B |
| **Read replica DB** | Replica đọc | Replication (eventual, lag = cửa sổ incoherence) |

> 🔑 **Insight #6:** Cùng một bài toán — "nhiều bản, giữ đừng lệch" — lặp ở **mọi tầng**, chỉ khác *khoảng cách* nên khác *chi phí coherence*: phần cứng (gần) → mạnh/đồng bộ; phân tán (xa) → yếu/bất đồng bộ. **Khoảng cách quyết định bạn được phép coherent tới đâu** (đúng tinh thần "khoảng cách quyết định lựa chọn CAP" ở chuyên đề trước).

---

## ⑩ Kiến thức MỞ RỘNG (kết nối cả cụm)

- ✅ **Spatial locality (B1):** đơn vị coherence là **cache line** → locality giúp đọc nhưng đẻ **false sharing** khi ghi. Coherence cho thấy *mặt trái* của locality.
- ✅ **Multi-tier L1/L2 & pub/sub (Bài 1 d):** chính là bài toán coherence + một cơ chế xử lý nó.
- ➕ **Eventual / Strong consistency (chuyên đề trước):** coherence là **cơ chế**; consistency model là **đảm bảo** bạn đạt được. Pub/sub coherence + versioning là cách *đẩy* eventual về gần strong cho từng key.
- ➕ **CAP (chuyên đề trước):** L1 các node = hệ AP nội bộ; coherence (pub/sub) là nỗ lực kéo về C — nhưng partition + best-effort khiến không bao giờ tuyệt đối → cần TTL backstop.
- ➕ **Invalidation patterns / write-through (C4 Bài 1):** write-through giữ coherence chặt hơn (ghi cache+DB cùng lúc) nhưng ghi chậm; cache-aside lỏng hơn (dính stale-set race).
- ➕ **CDC / leases / directory:** bộ công cụ coherence chặt-dần.
- ➕ **Memory model / happens-before / barrier:** coherence *không* thay được ordering — cần barrier cho lock-free.
- ➕ **Cold Start:** invalidate hot key đồng loạt = invalidation storm = cold cục bộ → cần singleflight.

---

## ⑪ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao sai | ✅ Đúng |
|---|---|---|
| **Lẫn coherence với consistency** | Coherence (một ô) ≠ memory model (thứ tự nhiều ô) | Phân biệt rõ; coherence là *cần* không *đủ* cho strong |
| Chỉ dùng **pub/sub, không TTL backstop** | Pub/sub best-effort — mất tin → lệch **vĩnh viễn** | Luôn kèm **TTL ngắn** làm lưới an toàn |
| **L1 TTL dài** "cho đỡ miss" | Cửa sổ incoherence giữa các node kéo dài | TTL L1 ngắn (vài giây) — Bài 1 đã cảnh báo |
| Rải **`cache.del()`** khắp app | Quên một đường ghi → incoherent lén lút | **CDC** từ binlog (không sót) |
| **Broadcast giá trị mới** (update) cho mọi node | Tốn băng thông với value lớn | **Invalidate** (xoá) thường rẻ hơn — như MESI |
| Bỏ qua **false sharing** ở counter nóng | Ping-pong cache line giết throughput | Padding/`@Contended`; đo bằng `perf c2c` |
| Invalidate hot key mà không chống stampede | 200 node miss đồng loạt → đập nguồn | **Singleflight + back-fill từ L2** |
| Nhồi **L1 cho dữ liệu đổi-liên-tục** | Chi phí giữ coherent > lợi ích tốc độ | Bỏ L1, dùng L2 chung (1 bản, coherent tự nhiên) |
| Tin **coherence = không cần barrier** | Reordering vẫn xảy ra giữa các ô | Dùng acquire/release/`volatile`/`atomic` đúng |

> 🔎 **AI hay sai (nối Bài 1):** AI hay đề xuất L1 cache khắp nơi *"cho nhanh"* mà **quên rằng nhiều bản L1 phải được giữ coherent** — rồi sinh bug "node này thấy mới, node kia thấy cũ". Luôn ép hỏi: *"Dữ liệu này có bao nhiêu bản? Giữ chúng đồng nhất bằng cơ chế gì? Mất tin pub/sub thì sao?"*

---

## ⑫ Khung quyết định / Playbook coherence

### D-COH-1 — Chọn cơ chế cho một loại dữ liệu
```
1. Có bao nhiêu BẢN của dữ liệu này? (L1 mỗi node? CDN edge? replica?)
   → 1 bản (chỉ L2 chung)  → coherent tự nhiên, xong.
   → nhiều bản             → tiếp.
2. Cần coherent tới đâu?
   → lỏng (đếm/feed)            → TTL.
   → vừa (catalog/profile)      → TTL + pub/sub invalidate.
   → người-tự-ghi-thấy-ngay     → + versioning (read-your-writes).
   → không-được-sót (quyền)     → CDC pipeline + TTL backstop.
   → tuyệt đối (tiền/tồn kho)   → bỏ cache path quyết định.
3. Hot key? → kèm singleflight chống invalidation storm.
4. Code concurrency nóng? → kiểm false sharing (padding).
```

### D-COH-2 — Kiểm tra một hệ multi-tier
```
□ L1 TTL có đủ NGẮN không? (vài giây — không để lệch lâu)
□ Có pub/sub invalidate cho key đổi-có-ý-nghĩa không?
□ Có TTL BACKSTOP phòng khi pub/sub mất tin không?  ← hay bị quên nhất
□ Invalidate có đáng tin không (CDC) hay rải rác dễ sót?
□ Thứ tự ghi→invalidate đúng + versioning chống stale-set race?
□ Invalidate hot key có chống stampede chưa?
□ Hot path concurrency đã loại false sharing chưa?
```

---

## ⑬ Mental model (câu thần chú gốc)

> 🔗 **Coherence = giữ nhiều bản của CÙNG MỘT ô dữ liệu đừng lệch nhau.** (Khác **consistency** = thứ tự giữa NHIỀU ô.)
>
> **Phần cứng (MESI)** làm coherence *mạnh, đồng bộ, tự động* vì khoảng cách ngắn.
> **Phân tán** chỉ *xấp xỉ* được bằng cơ chế yếu hơn vì network xa + có partition: **TTL → pub/sub → versioning → CDC → directory/lease**, chặt dần, đắt dần.
>
> Bốn câu phải thuộc:
> 1. **Luôn có TTL backstop** — pub/sub có thể mất tin.
> 2. **Invalidate đáng tin → CDC**, đừng rải `del()` khắp app.
> 3. **Invalidate hot key → kèm singleflight** kẻo storm.
> 4. **Coherence ≠ ordering** — lock-free vẫn cần barrier.

---

## ⑭ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Cache coherence là gì? → Giữ mọi bản sao của cùng một mục dữ liệu (ở các cache khác nhau) đồng nhất về giá trị.
- **(bẫy)** Coherence khác consistency thế nào? → Coherence: một ô, nhiều bản, đừng lệch. Consistency/memory model: thứ tự thao tác giữa nhiều ô. Coherence là *cần* không *đủ* cho strong consistency.
- **(dễ)** MESI là gì? → 4 trạng thái cache line: Modified/Exclusive/Shared/Invalid; ghi vào line Shared → invalidate các bản khác.
- **(TB)** Snooping vs directory? → Snooping broadcast trên bus (ít core); directory theo dõi ai giữ rồi báo đúng nơi (nhiều core/NUMA). Tương ứng pub/sub vs callback ở phân tán.
- **(TB)** Bài 1: L1 hai node lệch nhau — xử sao? → TTL ngắn (tự hết hạn) + pub/sub invalidate (báo xoá), kèm TTL backstop vì pub/sub best-effort.
- **(TB)** False sharing là gì, sửa sao? → Hai biến độc lập chung cache line → ping-pong giữa core; sửa bằng padding/căn lề khác line.
- **(khó)** Vì sao pub/sub một mình chưa đủ giữ coherence? → Best-effort, có thể mất tin → lệch vĩnh viễn; cần TTL backstop hoặc log bền (CDC at-least-once).
- **(khó)** Lease liên hệ TTL và callback ra sao? → Lease = lời hứa có hạn dữ liệu không đổi; TTL = lease không thu hồi được; callback = thu hồi lease chủ động. Cùng một phổ.

> 🧩 **Thách đố:** *"Sau khi thêm L1 in-process cho mọi node, user báo: đổi cài đặt ở trang này, sang trang kia (load-balance vào node khác) thấy cài đặt CŨ, vài giây sau mới đúng. Phân tích đầy đủ."*
> → **Incoherent L1 giữa các node** (Ví dụ 1): node xử lý ghi đã xoá L1 của *nó*, node khác vẫn giữ bản cũ tới khi TTL hết. Sửa theo lớp: (1) **pub/sub invalidate** để các node xoá ngay; (2) **TTL L1 ngắn** làm backstop phòng mất tin; (3) **versioning** để người vừa ghi luôn thấy ≥ version mình tạo (read-your-writes); (4) nếu cài-đặt-ảnh-hưởng-quyền thì **CDC** để không sót đường ghi nào. Nếu dữ liệu này đổi nhiều mà cần chặt → cân nhắc **bỏ L1, dùng L2 chung**.

---

## ⑮ Bài tập + tiêu chí tự chấm

**BT1.** Phân biệt coherence và consistency bằng một ví dụ có `x` và `y` (hai ô), chỉ ra một hệ *coherent* nhưng *không* linearizable.
> ✅ **Đạt khi:** ví dụ cho thấy mỗi ô cuối cùng đồng nhất (coherent) nhưng node thấy `y` mới trước `x` mới (sai thứ tự → memory model yếu).

**BT2.** Cho hệ 50 node, mỗi node L1, dữ liệu profile (đọc nhiều, đổi thưa, người-tự-ghi cần thấy ngay). Thiết kế cơ chế coherence đầy đủ.
> ✅ **Đạt khi:** TTL ngắn + pub/sub invalidate + TTL backstop + versioning cho read-your-writes; nêu lý do mỗi lớp.

**BT3.** Viết lại đoạn counter Java bị false sharing thành bản có padding, và nêu cách *đo* để chứng minh cải thiện.
> ✅ **Đạt khi:** tách hai biến khác cache line (`@Contended`/padding) + nhắc `perf c2c` hoặc benchmark throughput.

**BT4.** Giải thích vì sao pub/sub-một-mình không đủ và đề xuất nâng cấp lên invalidation đáng tin.
> ✅ **Đạt khi:** nêu best-effort/mất tin → cần TTL backstop; nâng cấp CDC + log bền at-least-once + tiêu thụ idempotent.

---

## ⑯ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Cache coherence** | Nhiều bản của một mục đừng lệch nhau | *"L1 các node lệch → vấn đề coherence."* |
| **Coherence vs consistency** | Một ô ↔ thứ tự nhiều ô | *"Coherent nhưng memory model yếu là có thật."* |
| **MESI** | 4 trạng thái cache line phần cứng | *"Ghi vào line Shared → invalidate bản khác."* |
| **Snooping / directory** | Broadcast bus ↔ theo dõi người giữ | *"Nhiều core thì directory, ít core thì snooping."* |
| **Cache line** | Đơn vị coherence (~64B) | *"Hai biến chung line → false sharing."* |
| **False sharing** | Biến độc lập chung line → ping-pong | *"Padding để counter nóng khác line."* |
| **Pub/sub invalidate** | Báo mọi node xoá L1 (push) | *"Đổi data → publish invalidate key."* |
| **TTL backstop** | TTL ngắn làm lưới an toàn coherence | *"Pub/sub mất tin → TTL vẫn cứu."* |
| **CDC invalidation** | Bắt binlog → phát invalidate | *"mcsqueal đọc binlog, xoá cache (không sót)."* |
| **Callback (AFS)** | Server báo client khi data đổi (directory) | *"Server gọi lại mọi client đang giữ file."* |
| **Lease** | Lời hứa có-hạn data không đổi | *"TTL là lease không thu hồi được."* |
| **Invalidation storm** | Invalidate hot key → miss đồng loạt | *"Xoá hot key 200 node → stampede."* |
| **Write-invalidate / write-update** | Xoá bản khác / phát giá trị mới | *"Invalidate rẻ hơn broadcast value lớn."* |
| **Memory barrier / happens-before** | Ép thứ tự cho lock-free | *"Coherence không thay barrier."* |
| **CXL** | Coherence CPU⟷accelerator | *"CXL cho GPU truy cập coherent."* |

---

## ⑰ Đọc thêm

- **Sorin, Hill & Wood, *A Primer on Memory Consistency and Cache Coherence*** — chuẩn mực; phân biệt coherence vs consistency, MESI/MOESI, snooping/directory.
- **Scaling Memcached at Facebook (NSDI 2013)** — invalidation pipeline từ binlog (mcsqueal), coherence xuyên vùng.
- **Gray & Cheriton, "Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency" (1989)**.
- **Howard et al., "Scale and Performance in a Distributed File System" (AFS)** — callback-based coherence.
- **Martin Kleppmann, *Designing Data-Intensive Applications*** — coherence/consistency/replication trong một bức tranh.
- **Ulrich Drepper, "What Every Programmer Should Know About Memory"** — cache line, false sharing, MESI thực chiến.
- **Linux `perf c2c`** — công cụ phát hiện false sharing / cache-line contention.
- **CXL Consortium** — coherence CPU↔accelerator hiện đại.

---

> 🔗 **Nối lại cả cụm (6 tài liệu):** Bài 1 dựng khung *tốc độ ↔ tươi ↔ RAM* và cảnh báo *"L1 các node lệch → coherence"*. Chuyên đề này chỉ ra coherence là **cơ chế** giữ bản sao đồng nhất — từ **MESI** (phần cứng, mạnh) tới **pub/sub/CDC** (phân tán, xấp xỉ). Nó là tầng *dưới* của **Eventual/Strong Consistency** (coherence + versioning đẩy eventual về gần strong), là chiến trường nội bộ của **CAP** (L1=AP, pub/sub cố kéo về C), và invalidation storm của nó là họ hàng **Cold Start**; còn *cái gì đáng giữ coherent* thì lại do **Working Set** quyết định. Câu chốt: ***mỗi bản sao bạn tạo ra để lấy tốc độ là một lời hứa phải giữ nó đồng nhất — hãy biết bạn giữ bằng cơ chế nào, chặt tới đâu, và chuyện gì xảy ra khi lời báo bị mất.***
