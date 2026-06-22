# 📉 CHUYÊN ĐỀ — AMDAHL'S LAW (Định luật Amdahl) trong Tối ưu hệ thống & Caching
> Đào sâu một concept mở rộng của **Bài 1 — Vì sao cache · Các tầng cache · Khi nào KHÔNG cache**
> Mục tiêu: hiểu *vì sao tối ưu phần không-phải-nút-thắt là vô ích*, biết *tính trần tăng tốc*, và mở rộng tới **USL** (vì sao thêm node có thể làm CHẬM đi) ở mức **staff engineer**.
> (✅ = đã có trong Bài 1 · ➕ = mở rộng thêm)

---

## ① Mục tiêu & vị trí trong mạch

Trong Bài 1, **Amdahl's law** xuất hiện ở **bảng concept mở rộng (B2)**:

> ➕ **Amdahl's law** — *Tối ưu phần không phải nút thắt thì vô ích* — *"Cache DB nhưng nút thắt ở serialize → vô dụng (kiểu Amdahl)."*

Và nó là **lý thuyết đứng sau hai trụ cột kỷ luật của Bài 1**:
- **B1: "Đo trước khi thêm tầng"** — *"p99 không cải thiện ư? Đo xem nút thắt có ở DB không đã."*
- **Thách đố ⑧ Bài 1:** *"Team thêm Redis cache trước DB, p99 vẫn không cải thiện — vì sao?"* → vì **nút thắt không nằm ở DB**.

Các chuyên đề trước cũng đã trỏ tới nó: **Cold Start** (⑪: warming cache vô ích nếu nút thắt không ở DB) và **Working Set** (⑪: sizing chuẩn cũng không cứu p99 nếu tối ưu sai chỗ).

Chuyên đề này đưa Amdahl ra sân khấu chính. Học xong bạn sẽ:

- Thuộc **công thức Amdahl** và tính được **trần tăng tốc** (cú sốc "100× trên 10% = chỉ +11%").
- Hiểu Amdahl là **toán học của "đo trước khi tối ưu"** — phần *không* cải thiện thành **trần** chặn tất cả.
- Nắm **Gustafson** (phản đề lạc quan) và **USL** (vì sao thêm node có thể làm **chậm đi** — nối Cache Coherence).
- Biết quy trình staff: **profile → tấn công phần thống trị → nút thắt DỜI → lặp lại**.

> 💡 **Một câu để gắn vào mạch Bài 1:** Bài 1 dạy *"chỉ cache khi đo được lợi"*. Amdahl cho bạn **biết trước lợi tối đa là bao nhiêu** mà *không cần* thử: nếu DB chỉ chiếm 10% thời gian, thì cache DB — dù xuống 0ms — **tối đa chỉ cải thiện 10%**. Amdahl biến trực giác "đừng tối ưu sai chỗ" thành một **phép tính trần** sắc bén.

---

## ② Trực giác — Amdahl's law nói gì?

### Phát biểu

**Mức tăng tốc tổng thể của một hệ, khi cải thiện MỘT phần, bị GIỚI HẠN bởi tỉ lệ thời gian mà phần đó chiếm.**

Nói thẳng: **phần bạn KHÔNG cải thiện sẽ trở thành nút thắt và chặn trần mọi nỗ lực.**

### Ẩn dụ chuyến đi

Bạn đi từ A đến B: **90 phút lái xe** + **10 phút đi bộ** = 100 phút. Bạn mua một chiếc xe **nhanh gấp 10 lần** (90 phút → 9 phút).

- Tổng mới = 9 (lái) + 10 (đi bộ) = **19 phút**. Tăng tốc = 100/19 ≈ **5.3×**.
- Nhưng nếu xe **nhanh vô hạn** (lái → 0 phút): tổng = 0 + 10 = **10 phút**. Tăng tốc tối đa = 100/10 = **10×**. **Không bao giờ vượt nổi** — vì **10 phút đi bộ** (phần không cải thiện) là **sàn cứng**.

> 🔑 **Insight #1:** Phần không-tối-ưu-được (10 phút đi bộ) **đặt trần** cho toàn bộ. Dù xe nhanh thế nào, bạn **không thể** đi nhanh hơn 10×. Đây là toàn bộ tinh thần Amdahl: **trần tăng tốc = 1 / (phần không cải thiện được)**.

### Công thức

Gọi **p** = tỉ lệ công việc *được* tăng tốc, được tăng **s** lần; phần còn lại **(1−p)** không đổi:

```
                    1
Speedup = ───────────────────────
            (1 − p)  +  p / s

Khi s → ∞ (tăng tốc phần đó vô hạn):

                    1
Speedup_max = ───────────      ← TRẦN, do phần không cải thiện quyết định
                 1 − p
```

Dạng song song kinh điển (N bộ xử lý, p = phần song song hoá được):

```
            1                                    1
S(N) = ───────────────     →    S(∞) = ───────────────
        (1−p) + p/N                        1 − p
```

---

## ③ Cú sốc bằng SỐ — vì sao tối ưu sai chỗ là vô ích

Đây là phần phải "thấm bằng số", không chỉ bằng lời.

### Bảng 1 — Tăng phần nhỏ lên 100× cho kết quả gì?

| Phần được tối ưu (p) | Tăng tốc phần đó (s) | Speedup tổng | Cảm giác |
|---|---|---|---|
| 10% | **100×** | 1/(0.9 + 0.001) ≈ **1.11×** | Đổ công sức 100× → chỉ nhanh hơn **11%** 😱 |
| 10% | **∞** | 1/0.9 ≈ **1.11×** | Dù xuống 0 cũng chỉ **+11%** |
| 50% | 100× | 1/(0.5 + 0.005) ≈ **1.98×** | ~2× |
| 90% | 100× | 1/(0.1 + 0.009) ≈ **9.2×** | Tấn công phần lớn → lời thật |
| 95% | 10× | 1/(0.05 + 0.095) ≈ **6.9×** | — |

> 🔑 **Insight #2 (cú đấm chính):** **Tăng tốc một phần NHỎ một cách ngoạn mục → lợi gần như KHÔNG.** Tối ưu 10% công việc lên *100 lần* chỉ nhanh hơn **11%**. Muốn lời lớn, phải tấn công **phần CHIẾM NHIỀU thời gian nhất** (nút thắt). Đây chính xác là *"cache DB nhưng nút thắt ở serialize → vô dụng"* của Bài 1, viết bằng số.

### Bảng 2 — Trần song song hoá (dù vô hạn lõi)

| Phần song song hoá được (p) | Trần tăng tốc (N→∞) |
|---|---|
| 50% | **2×** |
| 90% | 10× |
| 95% | 20× |
| 99% | 100× |
| 99.9% | 1000× |

> 🔑 **Insight #3:** Chỉ **5% phần tuần tự** đã chặn trần ở **20×** — *dù bạn có một triệu lõi*. Phần tuần tự (serial fraction) là kẻ thù không khoan nhượng của khả năng mở rộng. Đây là lý do "thêm máy/lõi" không tự động cho tốc độ tuyến tính.

---

## ④ Amdahl giải thích thách đố Bài 1 ("thêm Redis, p99 không cải thiện")

Áp công thức vào kịch bản kinh điển. Giả sử một request 100ms gồm:

```
  Network/serialize/deserialize:  60ms   ← nút thắt THẬT (60%)
  Compute (app logic):            30ms   (30%)
  DB query:                       10ms   (10%)   ← thứ bạn định cache
```

Bạn thêm Redis, cache **xoá sạch 10ms DB** (DB → 0):

```
Speedup = 100 / (100 − 10) = 100/90 ≈ 1.11×   →   100ms → 90ms
```

→ p99 chỉ giảm **10%**, gần như **không cảm nhận được**. Bạn đã tối ưu **đúng phần Amdahl cảnh báo** (10%, không phải nút thắt). **Nút thắt thật ở serialize (60%)** — đó mới là chỗ phải đánh.

> 🔑 **Insight #4:** "Đo trước khi thêm tầng" (Bài 1) **không phải lời khuyên đạo đức — nó là hệ quả toán học của Amdahl.** Trước khi thêm cache, hãy **phân rã thời gian theo thành phần**; nếu DB không phải phần thống trị, cache sẽ vô ích *về mặt p99* (dù vẫn có thể hữu ích để giảm **DB QPS / tải nguồn** — một mục tiêu khác, xem ⑤).

> ⚠️ **Lưu ý quan trọng:** cache có **hai mục tiêu khác nhau** (Bài 1 B1): *giảm latency* (Amdahl áp dụng) và *giảm tải nguồn / DB QPS* (Amdahl latency **không** trực tiếp áp). Cache có thể **vô dụng cho p99** mà vẫn **cứu DB khỏi sập**. Phải rõ bạn đang tối ưu **cái nào**.

---

## ⑤ Amdahl tổng quát — "tối ưu cái thống trị, không tối ưu cái dễ"

Amdahl gốc nói về tăng tốc/song song, nhưng tinh thần tổng quát hơn nhiều và áp cho **mọi** quyết định tối ưu:

> **Lợi ích của việc cải thiện một thành phần ≈ (mức cải thiện) × (tỉ trọng thành phần đó trong tổng).** Tỉ trọng nhỏ → lợi nhỏ, bất kể cải thiện lớn cỡ nào.

Hệ quả thực hành:

| Nguyên tắc | Nghĩa |
|---|---|
| **Make the common case fast** (Hennessy-Patterson) | Tối ưu cái *hay xảy ra & chiếm nhiều* — đúng phần p lớn của Amdahl |
| **Theory of Constraints** (Goldratt) | Cả dây chuyền nhanh bằng **mắt xích yếu nhất**; cải thiện mắt-không-phải-nút-thắt = lãng phí |
| **Profile, don't guess** | Amdahl bắt bạn *đo tỉ trọng* trước, vì trực giác hay sai chỗ |

> 🔑 **Insight #5:** Người mới hay tối ưu phần **mình quen / thấy dễ** (viết lại hàm mình hiểu rõ), không phải phần **thống trị thời gian**. Amdahl + profiler là liều thuốc giải: *đo, tìm phần to nhất, đánh đúng đó*.

---

## ⑥ Ví dụ minh hoạ

### Ví dụ 1 — Tối ưu query 2% (kinh điển phí công)

Một dev tối ưu một query từ 20ms → 2ms (nhanh 10×!), nhưng query đó chỉ chiếm **2%** thời gian request 1000ms. Speedup tổng = 1000/(1000−18) ≈ **1.018×** — nhanh hơn **1.8%**. Hai ngày công cho 1.8%. Amdahl đã cảnh báo trước nếu anh ta *đo tỉ trọng*.

### Ví dụ 2 — Nút thắt DỜI sau khi sửa (Amdahl đệ quy)

```
Trước:   DB 60ms | compute 30ms | net 10ms   (tổng 100ms)
Cache DB → DB 5ms:
Sau:     DB 5ms  | compute 30ms | net 10ms    (tổng 45ms)  → 2.2× ✅
   ⇒ Giờ COMPUTE (30/45 = 67%) là nút thắt MỚI.
Tối ưu tiếp phải nhắm compute, KHÔNG phải DB nữa.
```
> **Bài học:** Sửa nút thắt xong → **nút thắt dời chỗ** → **profile lại**. Amdahl áp dụng *lặp lại*, không một lần.

### Ví dụ 3 — Code: phân rã thời gian TRƯỚC khi tối ưu

```js
// Đo tỉ trọng từng phần trước khi quyết cache gì (đây là "Amdahl bằng tay")
async function handleRequest(req) {
  const t = {};
  let s = performance.now();
  const auth = await checkAuth(req);      t.auth = performance.now() - s; s = performance.now();
  const data = await db.query(req.q);     t.db   = performance.now() - s; s = performance.now();
  const out  = renderHeavy(data);         t.render = performance.now() - s;
  // Log tỉ trọng → biết phần nào THỐNG TRỊ → đánh đúng đó (đừng cache db nếu render mới to)
  metrics.timing("breakdown", t);
  return out;
}
```
> Đừng thêm cache theo cảm giác. **Đo breakdown** → tìm phần lớn nhất → mới quyết. Đây là Bài 1 "đo trước khi thêm tầng" thành code.

### Ví dụ 4 — Tail latency: nút thắt ở p99 KHÁC p50 (nối Cold Start)

Ở **p50**, DB nhanh (cache hit) → nút thắt là compute. Nhưng ở **p99**, request rơi vào **cache miss + DB chậm** → DB thành phần thống trị của *đuôi*. ⇒ **Amdahl phải áp theo từng percentile.** Tối ưu cho p50 (compute) có thể chẳng động tới p99 (DB tail). Phải hỏi: *"phần nào thống trị Ở ĐÚNG percentile tôi quan tâm?"*

---

## ⑦ Case study thực tế

> ⚠️ Tóm tắt theo tài liệu/nguyên lý công khai và pattern phổ biến. Verify số liệu cụ thể khi cần.

### CS1 — Vì sao 64 lõi không cho 64× (CPU đa nhân)

Một chương trình có 5% phần tuần tự (lock, khởi tạo, I/O nối tiếp) → trần Amdahl = **20×** dù 64 hay 1024 lõi. Thực tế còn tệ hơn vì **đồng bộ/contention** (xem USL ⑨). Đây là lý do "mua thêm lõi" không tự cứu hiệu năng — phải **giảm phần tuần tự** trước.

### CS2 — Spark/MapReduce: shuffle là phần "tuần tự" giới hạn scaling

Trong xử lý dữ liệu lớn, giai đoạn **shuffle** (trộn dữ liệu giữa các node) đóng vai phần khó-song-song-hoá. Thêm executor giúp phần map/reduce nhưng **shuffle + coordination** chặn trần. Tối ưu hay nhất thường là **giảm shuffle** (đúng tinh thần "đánh phần tuần tự"), không phải thêm máy.

### CS3 — ML training: nút thắt ở data loading, không phải GPU

Đội ML mua GPU mạnh hơn 2× nhưng training chỉ nhanh **chút xíu** — vì nút thắt là **nạp/augment dữ liệu trên CPU** (GPU đói, chờ data). Tăng tốc GPU = tối ưu phần *không* thống trị. Giải đúng: **prefetch / nhiều worker data loader / cache dataset** — tấn công đúng nút thắt. Amdahl thuần.

### CS4 — Kịch bản Bài 1 ở đời thực: thêm cache, latency đứng im

Vô số đội thêm Redis "cho nhanh" rồi ngạc nhiên vì p99 không nhúc nhích — vì latency bị thống trị bởi **serialize JSON khổng lồ / N+1 call / TLS handshake / GC pause**, không phải DB. Bài học lặp đi lặp lại: **profile trước**, cache sau.

### CS5 — Gene Amdahl (1967) bác bỏ "cứ thêm bộ xử lý"

Bài báo gốc của Amdahl ra đời để **phản biện** niềm tin thời đó rằng cứ thêm bộ xử lý song song là giải được mọi bài toán lớn. Ông chỉ ra phần tuần tự đặt trần cứng. Đây là **gốc rễ lịch sử** của tư duy "đo nút thắt trước khi ném tài nguyên vào".

---

## ⑧ Giải pháp staff engineer — quy trình tối ưu đúng

### Quy trình 5 bước (Amdahl-driven)

```
1. ĐO breakdown: phân rã metric bạn quan tâm (p99? throughput?) theo THÀNH PHẦN.
2. TÌM phần thống trị (nút thắt) — ở ĐÚNG percentile/đường mã quan tâm.
3. TÍNH trần: nếu xoá phần đó, lợi tối đa là bao nhiêu? (Amdahl) → đáng làm không?
4. ĐÁNH đúng nút thắt (không phải phần dễ/quen).
5. ĐO LẠI: nút thắt đã dời → quay lại bước 1. Lặp tới khi đủ tốt / chi phí > lợi.
```

### Công cụ tìm nút thắt thật

- **Flame graph / profiler** (CPU, allocation) — thấy phần nào "ăn" thời gian.
- **Distributed tracing** (OpenTelemetry/Jaeger) — thấy span nào thống trị trong chuỗi service.
- **Phân rã theo percentile** — p50 vs p99 breakdown riêng (Ví dụ 4).
- **Little's Law** (L = λ × W) — liên hệ throughput, latency, concurrency để biết tăng cái nào.

### Ba quy tắc vàng

1. **Tính trần TRƯỚC khi tối ưu.** "Phần này chiếm bao nhiêu %? Xoá nó được tối đa bao nhiêu?" — nếu < vài %, **bỏ qua**.
2. **Tối ưu đúng metric.** Cache cho **latency** (Amdahl) khác cache cho **DB QPS / tải nguồn** (không phải Amdahl). Đừng nhầm hai mục tiêu (⑤).
3. **Đúng percentile.** User cảm nhận **p99** (Bài 1); tối ưu phần thống trị *của đuôi*, không phải của trung vị.

---

## ⑨ Giải pháp NÂNG CAO — vượt khỏi Amdahl gốc

### A1. Gustafson's law — phản đề LẠC QUAN (đừng bi quan thái quá)

Amdahl giả định **bài toán cố định** (cùng lượng việc, làm nhanh hơn — *strong scaling*). Nhưng thực tế, có thêm tài nguyên ta thường **làm BÀI TOÁN TO HƠN** (xử nhiều dữ liệu hơn trong cùng thời gian — *weak scaling*). Khi quy mô bài toán tăng theo N, **phần tuần tự co lại tương đối** → speedup có thể tăng **gần tuyến tính**.

```
Amdahl:   "Cố định việc — nhanh hơn bao nhiêu?"   → bi quan (trần 1/(1−p))
Gustafson:"Cố định thời gian — làm thêm bao nhiêu?" → lạc quan (scale ~tuyến tính)
```

> 🔑 **Insight #6:** Hai định luật **không mâu thuẫn** — chúng trả lời **hai câu hỏi khác nhau**. Trước khi bi quan kiểu Amdahl, hỏi: *"Tôi cần làm CÙNG việc nhanh hơn (Amdahl), hay làm NHIỀU việc hơn trong cùng thời gian (Gustafson)?"* Big data / training quy mô lớn thường là thế giới Gustafson.

### A2. Universal Scalability Law (USL) — vì sao thêm node có thể làm CHẬM ĐI

Amdahl bỏ sót **chi phí phối hợp** giữa các node. **USL (Neil Gunther)** thêm vào:

```
                       N
C(N) = ─────────────────────────────────
        1 + α(N−1) + β·N(N−1)

α (contention)  = phần tuần tự / tranh chấp tài nguyên   (giống Amdahl)
β (coherency)   = chi phí PHỐI HỢP / ĐỒNG BỘ giữa các node  ← Amdahl không có
```

- Chỉ có **α**: đường cong **bão hoà** (như Amdahl) — thêm node ngừng giúp.
- Có thêm **β**: đường cong **đạt đỉnh rồi ĐI XUỐNG** (*retrograde*) — **thêm node làm CHẬM ĐI** vì chi phí đồng bộ vượt lợi ích.

> 🔑 **Insight #7 (nối Cache Coherence):** Số hạng **β chính là chi phí coherence** — lưu lượng invalidate/đồng bộ giữa các bản cache (chuyên đề Cache Coherence). Thêm node cache → thêm bản phải giữ đồng nhất → β tăng → có **điểm mà thêm node phản tác dụng**. USL cho bạn *điểm đỉnh* đó. Đây là lý do đôi khi **ít node TO** thắng **nhiều node nhỏ**.

### A3. Critical path — phần "tuần tự" trong một DAG tác vụ

Khi việc là một đồ thị phụ thuộc (DAG), **độ trễ bị chặn bởi đường dài nhất (critical path)** bất kể song song hoá phần khác bao nhiêu. Tối ưu một tác vụ **không nằm trên critical path** = vô ích (Amdahl ở dạng đồ thị). Staff engineer **rút ngắn critical path**, không tối ưu nhánh phụ.

### A4. Fit USL vào dữ liệu load-test → capacity planning

Chạy load test ở nhiều mức N (node/luồng), **fit đường cong USL** để ước lượng α, β → biết **đỉnh scalability** (thêm node tới đâu thì dừng / bắt đầu hại). Đây là cách định lượng "nên scale tới đâu" thay vì đoán.

### A5. Roofline — Amdahl cho kernel tính toán

Với code tính toán nặng (ML, HPC), **roofline model** đặt trần theo `min(đỉnh compute, băng thông bộ nhớ × cường độ số học)`. Biết bạn đang **compute-bound** hay **memory-bound** → tối ưu đúng trần (đừng tối ưu compute khi đang memory-bound — Amdahl phiên bản phần cứng).

---

## ⑩ Trường hợp MỞ RỘNG (Amdahl ở mọi tầng)

| Bối cảnh | "Phần không cải thiện được" / nút thắt |
|---|---|
| **Song song / đa lõi** (gốc) | Phần tuần tự (lock, init, I/O nối tiếp) → trần 1/(1−p) |
| **Distributed scaling** | Coordination/coherence (β của USL) → retrograde |
| **Web request latency (Bài 1)** | Tầng thống trị (serialize/compute/DB) — đo breakdown |
| **Caching** | Cache phần *không* thống trị → vô ích cho p99 |
| **ML training** | Data loading vs GPU compute (CS3) |
| **Big data (Spark)** | Shuffle/coordination (CS2) |
| **CI/CD build** | Bước nối tiếp dài nhất (critical path) |
| **Tổ chức/quy trình** | Trạm nghẽn (Theory of Constraints — Goldratt) |
| **DB query plan** | Operator thống trị (full scan/sort) |
| **Microservice chain** | Span chậm nhất trên critical path |

> 🔑 **Insight #8:** Cùng một nguyên lý — *"đánh phần thống trị, đo trước"* — lặp ở **mọi tầng**, từ kernel (roofline) tới cluster (USL) tới cả tổ chức (Theory of Constraints). Amdahl là **cách tư duy**, không chỉ công thức song song.

---

## ⑪ Kiến thức MỞ RỘNG (kết nối cả cụm)

- ✅ **"Đo trước khi thêm tầng" + "gắn cache vào thứ cần bảo vệ" (B1 Bài 1):** Amdahl là *toán học* đằng sau. Đo breakdown → biết cache có đáng không.
- ✅ **Thách đố Bài 1 (thêm Redis, p99 đứng im):** giải thích trực tiếp bằng Amdahl (⑤).
- ➕ **Cold Start (chuyên đề):** warming vô ích nếu nút thắt không ở DB; tail amplification = Amdahl theo percentile.
- ➕ **Working Set (chuyên đề):** "knee" của đường hit-ratio và "đỉnh" của USL cùng họ *lợi ích giảm dần / âm dần*. Sizing đúng chỗ ≈ tối ưu đúng nút thắt.
- ➕ **Cache Coherence (chuyên đề):** chi phí coherence **chính là β của USL** — thêm bản cache → thêm đồng bộ → có điểm phản tác dụng.
- ➕ **Gustafson / USL / Roofline / Theory of Constraints / Little's Law / Critical path:** bộ công cụ định lượng quanh Amdahl.
- ➕ **Hennessy-Patterson "make the common case fast":** Amdahl ở dạng khẩu quyết kiến trúc máy tính.

---

## ⑫ Anti-patterns — AI & người mới hay sai

| ❌ Sai lầm | Vì sao sai | ✅ Đúng |
|---|---|---|
| Tối ưu phần **mình quen/dễ** | Hiếm khi là phần thống trị | **Profile** → đánh phần to nhất |
| Thêm cache/lõi/node **không đo trước** | Có thể tối ưu phần không-nút-thắt (Bài 1) | Đo breakdown → tính trần Amdahl trước |
| Tin **thêm máy = nhanh tuyến tính** | Phần tuần tự (Amdahl) + coordination (USL β) | Giảm phần tuần tự; biết điểm đỉnh USL |
| Micro-optimize **hotspot 1–2%** | Trần lợi cực nhỏ (Bảng 1) | Bỏ qua; tìm phần ≥ phần lớn |
| Tối ưu **p50** khi user cảm **p99** | Nút thắt p99 khác p50 (Ví dụ 4) | Phân rã & tối ưu theo **đúng percentile** |
| Sửa một nút thắt rồi **dừng** | Nút thắt **dời chỗ** (Ví dụ 2) | **Profile lại**, lặp Amdahl |
| Bi quan kiểu Amdahl cho **mọi** bài toán | Quên Gustafson (bài toán có thể scale) | Hỏi "cùng việc nhanh hơn" hay "nhiều việc hơn"? |
| Cứ thêm **node cache nhỏ** | β (coherence) có thể làm chậm đi | Cân nhắc ít node TO; đo USL |

> 🔎 **AI hay sai (nối Bài 1):** AI mặc định *"thêm cache/thêm máy = nhanh hơn"* mà **không đo phần nào thống trị**. Luôn ép hỏi: *"Phần tôi định tối ưu chiếm bao nhiêu % thời gian (ở percentile nào)? Trần lợi tối đa theo Amdahl là bao nhiêu? Có đáng không?"*

---

## ⑬ Khung quyết định / Playbook Amdahl

### D-AMD-1 — Trước khi tối ưu BẤT CỨ thứ gì
```
1. Metric mục tiêu là gì? (p99 latency? throughput? DB QPS?)
2. Phân rã metric đó theo THÀNH PHẦN (breakdown), ở ĐÚNG percentile.
3. Phần định tối ưu chiếm p = ? % tổng.
4. Trần Amdahl = 1/(1 − p) (nếu xoá hẳn). Tính ra con số.
   → Trần < ~10%?  → BỎ QUA, tìm phần khác to hơn.
   → Trần lớn?     → đáng làm, tiến hành.
5. Sau khi sửa → ĐO LẠI (nút thắt đã dời) → lặp.
```

### D-AMD-2 — "Thêm cache có giúp latency không?"
```
□ DB chiếm bao nhiêu % của p99? (đo, đừng đoán)
   → nhỏ → cache KHÔNG cứu p99 (nhưng có thể cứu DB QPS — mục tiêu khác).
   → lớn → cache đáng giá; ước lợi = phần DB có thể cắt.
□ Rõ mục tiêu: latency (Amdahl) hay tải nguồn (không phải Amdahl)?
```

### D-AMD-3 — "Thêm node có scale không?"
```
□ Phần tuần tự (α) bao nhiêu? → trần Amdahl.
□ Chi phí coordination/coherence (β) có lớn không? → USL retrograde?
□ Bài toán CỐ ĐỊNH (Amdahl) hay SCALE ĐƯỢC (Gustafson)?
□ Fit USL vào load-test → tìm ĐỈNH (thêm node tới đâu thì dừng).
```

---

## ⑭ Mental model (câu thần chú gốc)

> 📉 **Amdahl: phần bạn KHÔNG cải thiện trở thành TRẦN.** Tăng tốc một phần nhỏ một cách ngoạn mục → lợi gần như không (100× trên 10% = chỉ +11%).
>
> ⇒ **Đo trước, đánh phần thống trị, ở đúng percentile.** "Đo trước khi thêm tầng" (Bài 1) là **hệ quả toán học** của Amdahl, không phải lời khuyên suông.
>
> Ba điều phải nhớ:
> 1. **Tính trần `1/(1−p)` TRƯỚC khi tối ưu** — phần nhỏ thì bỏ qua.
> 2. **Nút thắt DỜI sau mỗi lần sửa** — profile lại, lặp.
> 3. **Phân tán: coi chừng β (coherence/coordination)** — thêm node có thể làm CHẬM đi (USL), trừ khi bài toán scale được (Gustafson).

---

## ⑮ Câu hỏi phỏng vấn & thách đố

- **(dễ)** Amdahl's law nói gì? → Tăng tốc tổng bị chặn bởi tỉ lệ thời gian phần được cải thiện; phần không cải thiện đặt trần 1/(1−p).
- **(dễ)** Tối ưu 10% công việc lên 100× thì tổng nhanh hơn bao nhiêu? → ≈ **1.11×** (chỉ +11%) — cú sốc kinh điển.
- **(TB)** 95% song song hoá được, vô hạn lõi → trần? → 1/0.05 = **20×**.
- **(TB)** Vì sao thêm Redis mà p99 không cải thiện (thách đố Bài 1)? → Nút thắt không ở DB; DB chỉ chiếm phần nhỏ → Amdahl chặn trần lợi ở phần nhỏ đó.
- **(TB)** Cache vô dụng cho p99 — vậy có bao giờ vẫn nên thêm? → Có, nếu mục tiêu là **giảm DB QPS / tải nguồn** (khác latency) — hai mục tiêu khác nhau.
- **(khó)** Gustafson khác Amdahl ra sao? → Amdahl: việc cố định, nhanh hơn bao nhiêu (bi quan). Gustafson: thời gian cố định, làm thêm bao nhiêu (lạc quan, scale ~tuyến tính).
- **(khó)** USL thêm gì so với Amdahl, liên hệ caching? → Thêm số hạng **β coherence/coordination** → thêm node có thể **làm chậm đi** (retrograde); β chính là chi phí cache coherence (giữ nhiều bản đồng nhất).
- **(khó)** Tối ưu xong nút thắt thì làm gì? → Profile lại — nút thắt **đã dời**; Amdahl áp dụng lặp.

> 🧩 **Thách đố:** *"Đội bạn scale service từ 10 lên 40 instance, throughput tăng tới ~25 instance rồi BẮT ĐẦU GIẢM. Giải thích & xử."*
> → Đây là **USL retrograde**: số hạng **β (coordination/coherence)** vượt lợi ích — có thể do tranh chấp một tài nguyên dùng chung (DB lock, distributed cache invalidation, lock phân tán, shared counter). Amdahl thuần (chỉ α) chỉ *bão hoà*, không *giảm* → dấu hiệu β lớn. Xử: tìm điểm đồng bộ chung (profile contention), **giảm coordination** (shard, bớt shared state, bớt invalidation chéo, batch), cân nhắc **ít instance TO hơn**; fit USL vào số liệu để biết đỉnh thật. Bài học: **scale không tuyến tính — có điểm thêm node phản tác dụng.**

---

## ⑯ Bài tập + tiêu chí tự chấm

**BT1.** Request 200ms: auth 20ms, DB 40ms, render 120ms, net 20ms. Bạn định cache DB (40→4ms). Tính speedup tổng. Có đáng không? Nên đánh đâu?
> ✅ **Đạt khi:** speedup = 200/164 ≈ **1.22×** (~18%); chỉ ra **render (60%)** mới là nút thắt nên đánh trước.

**BT2.** Phần song song hoá được là 80%. Tính trần với N=4, N=16, N=∞.
> ✅ **Đạt khi:** N=4 → 1/(0.2+0.2)=2.5×; N=16 → 1/(0.2+0.05)=4×; N=∞ → 1/0.2 = **5×**.

**BT3.** Giải thích bằng lời + công thức vì sao "thêm cache, p99 đứng im" và phân biệt với "thêm cache cứu DB QPS".
> ✅ **Đạt khi:** Amdahl (DB chiếm % nhỏ của p99 → trần lợi nhỏ) cho latency; nhưng cache vẫn giảm số query xuống DB → cứu QPS (mục tiêu khác).

**BT4.** Mô tả khi nào dùng tư duy Gustafson thay vì Amdahl, cho một ví dụ ML/big-data.
> ✅ **Đạt khi:** khi bài toán **scale được** (xử thêm dữ liệu trong cùng thời gian); vd training trên dataset lớn hơn khi có thêm GPU → speedup gần tuyến tính.

---

## ⑰ Glossary (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh sử dụng |
|---|---|---|
| **Amdahl's law** | Phần không cải thiện đặt trần tăng tốc | *"Cache DB nhưng nút thắt ở serialize → vô dụng."* |
| **Serial fraction (1−p)** | Phần tuần tự/không tối ưu được | *"5% tuần tự → trần 20× dù vô hạn lõi."* |
| **Trần tăng tốc** | 1/(1−p) khi tối ưu phần đó vô hạn | *"Tính trần trước khi đổ công tối ưu."* |
| **Bottleneck / nút thắt** | Phần thống trị thời gian | *"Profile tìm nút thắt, đánh đúng đó."* |
| **Đo breakdown** | Phân rã metric theo thành phần | *"Đo breakdown trước khi thêm cache."* |
| **Gustafson's law** | Việc scale được → speedup ~tuyến tính | *"Big data là thế giới Gustafson, không bi quan."* |
| **USL** | Amdahl + chi phí coordination (β) | *"Thêm node tới đỉnh rồi chậm đi → USL."* |
| **β (coherency)** | Chi phí đồng bộ/coherence giữa node | *"β = chi phí cache coherence → retrograde."* |
| **Retrograde scaling** | Thêm node làm CHẬM đi | *"Throughput giảm sau 25 node = retrograde."* |
| **Critical path** | Đường phụ thuộc dài nhất | *"Rút critical path, đừng tối ưu nhánh phụ."* |
| **Theory of Constraints** | Dây chuyền nhanh bằng mắt yếu nhất | *"Cải thiện trạm không-nghẽn = lãng phí."* |
| **Make the common case fast** | Tối ưu cái hay xảy ra & chiếm nhiều | *"Khẩu quyết Amdahl của kiến trúc máy tính."* |
| **Little's Law** | L = λ × W | *"Liên hệ throughput–latency–concurrency."* |
| **Roofline** | Trần compute-bound vs memory-bound | *"Memory-bound thì tối ưu compute vô ích."* |

---

## ⑱ Đọc thêm

- **Gene Amdahl, "Validity of the Single Processor Approach…" (1967)** — bài báo gốc.
- **John Gustafson, "Reevaluating Amdahl's Law" (1988)** — phản đề scale-up.
- **Neil Gunther, *Guerrilla Capacity Planning*** & **Universal Scalability Law** — β coherence, retrograde scaling, fit USL.
- **Eli Goldratt, *The Goal*** — Theory of Constraints (Amdahl cho tổ chức/quy trình).
- **Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*** — Amdahl + "make the common case fast" + roofline.
- **Williams, Waterman & Patterson, "Roofline: An Insightful Visual Performance Model"**.
- **Brendan Gregg, *Systems Performance*** — flame graph, tìm nút thắt thực chiến.

---

> 🔗 **Nối lại cả cụm (8 tài liệu):** Bài 1 dạy *"chỉ cache khi đo được lợi"* và *"đo trước khi thêm tầng"*. **Amdahl là toán học của kỷ luật đó** — nó cho bạn **trần lợi ích** trước khi tốn một dòng code, giải thích vì sao "thêm Redis mà p99 đứng im", và (qua **USL**) vì sao thêm node cache có thể **phản tác dụng** khi chi phí **coherence** (chuyên đề Cache Coherence) vượt lợi ích. Nó là **người gác cổng** của cả 7 chuyên đề kia: trước khi lo Cold Start, Working Set, hay consistency, hãy hỏi *"thứ tôi sắp tối ưu có phải nút thắt không?"*. Câu chốt: ***đừng tối ưu cái bạn THẤY — tối ưu cái ĐO ĐƯỢC là thống trị; phần bạn bỏ qua chính là cái sẽ chặn trần bạn.***
