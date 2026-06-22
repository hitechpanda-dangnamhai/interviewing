# 🔥 CACHE WARMING — Chuyên đề mở rộng (Caching · nối tiếp Bài 1)
> **Hâm nóng cache TRƯỚC khi traffic tới** · Keyword · Concept · Chiến lược · Case study · Staff-level
> (Đọc sau khi đã nắm: *locality · cold start · multi-tier L1/L2 · stampede · cache = lazy / MV = eager*)

---

## ① ĐỊNH VỊ — Cache warming nằm ở đâu trong bức tranh

Bài 1 chốt: **cache = lazy** (chỉ tính khi *có người hỏi*) → lần đầu **luôn miss**, phải đi nguồn. Bình thường không sao. **Vấn đề** là khi cache **rỗng đúng lúc cao điểm**:

> Bài 1 đã nói chính xác triệu chứng: *"Sau deploy, cache **cold** nên p99 tăng vọt vài phút đầu."* Và: *"Trước flash sale, **warm cache** để tránh miss hàng loạt."*

**Cache warming = chủ động NẠP SẴN dữ liệu vào cache TRƯỚC khi request thật tới**, để những user đầu tiên gặp **hit** chứ không phải **miss**.

> 🔑 **Một câu chốt:** Warming biến một **cache lazy** thành **eager TẠM THỜI** — bạn **trả trước cái giá "miss lần đầu"** theo lịch của *bạn*, thay vì để nó rơi vào đầu *user* đúng lúc đông nhất.

### Vị trí trong tam giác đánh đổi (Bài 1)

| Cạnh tam giác | TTL/Invalidate | Eviction | **Cache warming** |
|---|---|---|---|
| Quản trị | Độ tươi | Bộ nhớ | **Đường cong "lấp đầy" theo thời gian** |
| Câu hỏi | "Khi nào xoá?" | "Đầy thì đuổi ai?" | **"Lúc nào cache có sẵn dữ liệu nóng?"** |

---

## ② BÀI TOÁN GỐC — Cold start & vì sao nó nguy hiểm

### Cold cache → miss storm → stampede

Nhớ chuỗi nhân quả từ Bài 1 (*locality → cache sống nhờ "sẽ được hỏi lại"*). Cache **rỗng** thì **mọi key đều miss**:

```
Cache rỗng (cold)
   → MỌI request đầu tiên đều MISS
   → đồng loạt đập xuống DB (thundering herd / stampede)
   → DB QPS bùng nổ, p99 vọt lên
   → đúng vào LÚC bạn cần cache bảo vệ nhất, nó lại KHÔNG bảo vệ gì cả
```

> ⚠️ Đây là nghịch lý: **cache vô dụng nhất đúng lúc tải cao nhất** (ngay sau deploy / ngay khi mở flash sale). Warming sinh ra để bịt đúng khoảng trống này.

### 🧊 Các tình huống cache bị "lạnh" (cold-start triggers) — phải nhận diện

| Tình huống | Vì sao cache lạnh |
|---|---|
| **Deploy / restart** app | L1 in-process **mất sạch** theo tiến trình |
| **Autoscale-up** (thêm instance) | Instance mới **L1 hoàn toàn rỗng** dù L2 đang ấm |
| **Restart / flush Redis** (L2) | Cả cụm chia sẻ mất → **mọi node** cùng miss |
| **Region failover** | Cụm cache vùng dự phòng **chưa có gì** |
| **Sự kiện traffic dự đoán trước** | Flash sale, mở bán vé, ra mắt SP, Black Friday, trận đấu lớn → một loạt key **lần đầu được hỏi** đồng thời |
| **Sau sự cố** (post-incident) | Vừa khôi phục, cache trống, traffic dồn lại |

🎬 *Hỏi staff: "Khi nào hệ của bạn có cold cache?" → liệt kê đúng 6 ô này là cho thấy bạn hiểu vận hành thật, không chỉ lý thuyết.*

---

## ③ CÁC CHIẾN LƯỢC WARMING (phần thịt — phải thuộc)

### Bảng tổng các chiến lược

| Chiến lược | Cách làm | Mạnh khi | Bẫy chính |
|---|---|---|---|
| **1. Eager preload (batch warm)** | Script nạp sẵn **top-K hot key** trước giờ G | Biết trước hot key (top SP, trang chủ) | Nạp nhầm key / **nạp 1 triệu key cùng lúc đập sập DB** |
| **2. Warm-from-peer (nạp từ tier ấm)** | Instance/cụm mới đọc từ **L2/cụm-anh-em đang ấm**, KHÔNG đập DB | Có sẵn một nguồn ấm | Phải xử lý race stale khi vừa warm vừa có ghi |
| **3. Snapshot restore (persistence)** | Lưu cache xuống đĩa, khởi động **nạp lại** từ snapshot | Restart Redis / buffer pool | Snapshot **cũ** → warm bằng đồ stale |
| **4. Traffic replay / shadow** | **Phát lại log query** hoặc soi gương traffic thật vào instance mới | Cụm mới cần giống production | Tốn hạ tầng replay; cần log đủ |
| **5. Slow-start / canary ramp** | Cho traffic vào **từ từ** để cache **tự lấp dần** trước khi nhận full tải | Mọi deploy/scale | Ramp chậm → triển khai lâu |
| **6. Refresh-ahead (warm LIÊN TỤC)** | **Làm tươi entry TRƯỚC khi hết hạn** → không bao giờ để nguội | Hot key ổn định | Lãng phí refresh thứ không ai hỏi |
| **7. Scheduled/predictive warm** | Cron/ML nạp sẵn **đúng trước sự kiện đã biết** | Lịch biết trước (flash sale 12h) | Đoán sai cái gì sẽ hot |

### 🌟 Hai cặp khái niệm phải phân biệt

**Warming (một lần) vs Refresh-ahead (liên tục).**
- **Warming**: hành động *một lần* để lấp cache lạnh (sau deploy / trước sự kiện).
- **Refresh-ahead**: cơ chế *chạy mãi* — chủ động làm tươi key nóng *trước* khi TTL hết → cache **không bao giờ nguội**, user **không bao giờ phải chịu một miss** trên key nóng. Đây là "warming biến thành chế độ thường trực".

**Warming vs Materialized View (nối Bài 1 mục g).**
- Bài 1: *cache = lazy, MV = eager*. **Warming = làm cache lazy "giả eager" tạm thời.** Nếu bạn thấy mình phải warm **liên tục mọi thứ** → có lẽ thứ đó nên là **MV / precompute** thật sự, đừng cố ép cache đóng vai eager mãi.

---

## ④ VÍ DỤ MINH HOẠ + CODE (theo style Bài 1)

### 4.1 — Eager preload top-K (nối multi-tier L1+L2 của Bài 1)

Ý tưởng: trước flash sale, nạp **danh sách SP nóng** vào **cả L2 (Redis) và L1**, có **throttle** để không đập sập DB.

```javascript
// warm.js — hâm nóng top-K sản phẩm trước giờ G
// Dùng lại getProduct() multi-tier ở Bài 1 (L1 + L2)
import pLimit from "p-limit";

const limit = pLimit(20); // ⚠️ THROTTLE: tối đa 20 query DB song song → KHÔNG stampede chính nguồn

export async function warmTopProducts(hotIds, loadFromDb) {
  console.time("warm");
  await Promise.all(
    hotIds.map((id) =>
      limit(async () => {
        try {
          await getProduct(id, loadFromDb); // miss → đọc DB → back-fill L2 + L1 (như Bài 1)
        } catch (e) {
          // warming KHÔNG được phép làm sập app → nuốt lỗi, log lại
          console.warn("warm fail", id, e.message);
        }
      })
    )
  );
  console.timeEnd("warm");
}

// hotIds lấy từ ĐÂU? → top-K theo tần suất TRONG LOG production (mục ⑥), không phải đoán mò
```

> 🔑 Hai chi tiết staff trong đoạn này: **(1) throttle** (`pLimit`) để bản thân việc warm không trở thành stampede; **(2) nuốt lỗi** — warming là tác vụ *nền*, hỏng thì degrade chứ không được kéo app chết.

### 4.2 — Readiness gate: chưa ấm thì CHƯA nhận traffic

Trên Kubernetes, đừng để pod mới **vừa lên đã nhận full tải khi L1 còn rỗng**. Chặn bằng **readiness probe**:

```javascript
// Pod chỉ "ready" khi đã warm xong → load balancer mới route traffic vào
let warmed = false;
(async () => {
  await warmTopProducts(await fetchHotIds(), loadFromDb);
  warmed = true;
})();

app.get("/readyz", (_req, res) =>
  warmed ? res.sendStatus(200) : res.sendStatus(503) // 503 → LB chưa gửi traffic
);
```

> ➜ Biến warming thành **điều kiện vào cuộc**: instance lạnh **không bao giờ** phục vụ user. Đổi lại deploy chậm hơn chút — một đánh đổi đáng giá ở path nóng.

### 4.3 — Refresh-ahead (giữ hot key không bao giờ nguội)

```javascript
// Làm tươi TRƯỚC khi hết hạn: nếu entry sắp hết TTL, refresh nền NGAY (không chặn user)
async function getWithRefreshAhead(id, loadFromDb) {
  const { val, expiresAt } = await getWithMeta(`product:${id}`);
  const soon = expiresAt - Date.now() < 30_000; // còn <30s là "sắp nguội"
  if (val && soon) {
    queueMicrotask(() => refresh(id, loadFromDb)); // làm tươi NỀN, user vẫn nhận bản hiện tại
  }
  return val ?? (await loadFromDb(id));
}
```

---

## ⑤ CASE STUDY THỰC TẾ

**1. ⭐ Facebook — "cold cluster warmup" (CHÍNH bài trong reading list của bạn: *Scaling Memcached at Facebook*).**
Khi đưa **một cụm Memcached lạnh** vào hoạt động, nếu để nó miss thẳng xuống DB thì **DB sẽ bị đập sập**. Giải pháp kinh điển: **cấu hình cụm lạnh, khi miss, đọc từ một cụm Memcached ĐANG ẤM** (warm-from-peer) trong một khoảng thời gian, **thay vì xuống DB**. Họ còn xử lý khéo **race "vừa warm vừa có ghi"** (một bản cũ có thể bị kéo từ cụm ấm về ngay sau khi vừa cập nhật) bằng cơ chế hold-off ngắn quanh thao tác delete để tránh phục vụ đồ stale.
> 🎯 Bài học vàng: **warm từ một tier ĐANG ẤM, đừng warm từ DB** — đây là pattern staff-level số một để warming mà không tự đập sập nguồn.

**2. E-commerce flash sale (đúng use case xuyên suốt Bài 1).**
Trước giờ mở bán: **preload** trang chủ, ảnh/CSS lên CDN, thông tin SP nóng + **giá khuyến mãi theo user** (nhớ Bài 1: *key phải gồm `user`*) lên L2/L1. **Tồn kho realtime thì KHÔNG warm** — vẫn đọc thẳng DB lúc checkout vì cần **strong consistency** (Bài 1). ➜ Warm cái *đọc-nhiều-ít-đổi*, chừa cái *cần-đúng-tuyệt-đối*.

**3. MySQL InnoDB — warm buffer pool qua restart (cấu hình có thật).**
Bài 1 nhắc **DB buffer pool** cũng là một tầng cache. InnoDB cho phép **lưu danh sách page nóng xuống đĩa khi tắt** và **nạp lại khi khởi động**:
`innodb_buffer_pool_dump_at_shutdown=ON` + `innodb_buffer_pool_load_at_startup=ON` (và `..._dump_now` / `..._load_now` để chủ động). Nó dump **ID của page** (không phải data), khởi động lại thì kéo đúng các page đó vào RAM → **buffer pool ấm ngay** thay vì lạnh hàng giờ sau mỗi lần restart.

**4. Redis — persistence (RDB/AOF) như một dạng warming.**
Restart Redis mà có **RDB snapshot / AOF** → nạp lại data từ đĩa → **L2 ấm ngay**, không phải back-fill từ DB toàn bộ. Promote **replica đang ấm** lên primary cũng là cách giữ cụm không bao giờ lạnh.

**5. CDN prewarm / prefetch (Cloudflare, Akamai, Fastly).**
Sau khi **purge** một loạt asset (ví dụ vừa deploy bản web mới), nếu để user đầu tiên ở mỗi edge phải miss về origin → origin lãnh đòn. ➜ Chủ động **prefetch** các URL nóng vào edge **trước** khi mở traffic, hoặc dùng tiered cache để edge lạnh kéo từ một tier khu vực đang ấm.

**6. Serverless cold start (khái niệm anh em — AWS Lambda Provisioned Concurrency).**
Không phải cache *data* nhưng cùng triết lý: **giữ sẵn instance "ấm"** để request đầu khỏi chịu độ trễ khởi tạo. Cùng một tư duy "trả trước cái giá khởi động". (DynamoDB có **DAX** — một cache cũng cần cân nhắc warming tương tự.)

---

## ⑥ STAFF-LEVEL: WARM CÁI GÌ, TỪ ĐÂU, BAO NHIÊU

### 6.1 — Warm CÁI GÌ? (đừng đoán mò — nối Bài 1 "đo, gắn vào thứ cần bảo vệ")

> Bài 1 cảnh báo: **đừng tối ưu hit ratio mù quáng.** Warming cũng vậy: **warm nhầm key = tốn công + tốn RAM + không cứu được gì.**

- Lấy **top-K key theo tần suất từ LOG production** (access log / Redis `OBJECT FREQ` / sampling), **không** theo cảm tính.
- Chỉ warm **working set** (Bài 1 ➕): tập thực sự được dùng, không cần warm toàn bộ data.
- Với sự kiện: warm theo **dữ liệu sự kiện** (SKU trong flash sale, nghệ sĩ trong đợt mở bán vé), không warm cả catalog.

### 6.2 — Warm TỪ ĐÂU? (quy tắc vàng staff)

```
Ưu tiên nguồn warm:   tier ĐANG ẤM  >  snapshot/đĩa  >  DB (nguồn gốc)
```
- **Warm-from-peer** (kiểu Facebook) là tốt nhất: instance/cụm mới hút từ **L2 hoặc cụm anh em đang ấm** → **không đập DB**.
- Bất đắc dĩ mới warm từ DB → **phải throttle nặng** (rate limit + singleflight) kẻo chính việc warm gây stampede.

### 6.3 — Warm BAO NHIÊU & KHI NÀO

- **Bao nhiêu:** dùng **Miss Ratio Curve** / phân tích log để biết warm tới **điểm gãy (knee)** là đủ — warm thêm gần như không giảm miss nữa (đúng tinh thần "đo trước khi thêm tầng" của Bài 1).
- **Khi nào:** warm **càng SÁT giờ G càng tốt** với data dễ đổi (để không bị stale trước khi dùng); warm sớm với data tĩnh.

### 6.4 — Chống stampede TRONG LÚC warm (nối Bài 1 ➕)

- **Throttle** việc warm (giới hạn song song).
- **Singleflight / request coalescing**: nhiều instance cùng warm một key → gộp thành **một** lần đi nguồn.
- **Jitter**: nếu warm nhiều key có TTL, **rải TTL ngẫu nhiên** để chúng **không cùng hết hạn** → tránh stampede đợt sau.

---

## ⑦ KHUNG QUYẾT ĐỊNH STAFF ENGINEER

```
1. Hệ có "cold-start pain" KHÔNG? (deploy spike / autoscale / sự kiện dự đoán trước / failover)
   ├─ KHÔNG (traffic đều, cache luôn ấm tự nhiên) → ĐỪNG warm (thêm phức tạp vô ích)
   └─ CÓ → sang bước 2

2. ĐO trước (Bài 1): cold-start ảnh hưởng thật bao nhiêu? (p99 vọt mấy giây? DB QPS spike?)
   └─ Nếu ảnh hưởng nhỏ → có thể chỉ cần slow-start ramp, khỏi build warming pipeline

3. WARM CÁI GÌ → top-K từ LOG (working set), KHÔNG đoán mò

4. WARM TỪ ĐÂU → tier ấm > snapshot > DB (throttle nếu buộc dùng DB)

5. CƠ CHẾ:
   ├─ Sự kiện biết trước (flash sale)     → scheduled/predictive preload + readiness gate
   ├─ Mỗi deploy/autoscale                → warm-from-peer (L2) + readiness gate + slow-start
   ├─ Hot key cần luôn nóng (steady state)→ refresh-ahead + jittered TTL
   └─ Restart Redis/DB                    → snapshot restore (RDB/AOF; buffer pool dump/load)

6. BỌC AN TOÀN: throttle + singleflight + jitter; warming hỏng phải DEGRADE, không kéo app chết
```

> 🥇 **Mặc định thực dụng:** sự kiện đã biết → **preload top-K + readiness gate**; vận hành thường ngày → **warm-from-peer khi scale** + **refresh-ahead cho hot key**. **Luôn warm-from-peer, hiếm khi warm-from-DB.**

---

## ⑧ BẪY THƯỜNG GẶP / 🔎 AI HAY SAI

- 🔎 **Warming tự gây stampede.** Nạp 1 triệu key cùng lúc **đập thẳng DB** → đúng thứ định tránh lại tự gây ra. ➜ **throttle + warm-from-peer**.
- 🔎 **Warm nhầm key.** Đoán mò thay vì lấy **top-K từ log** → hit ratio warm thấp, tốn RAM (đúng cảnh báo "hit ratio mù quáng" Bài 1).
- 🔎 **Warm bằng đồ stale.** Snapshot cũ / warm quá sớm trước khi data đổi → cache ấm nhưng **sai**. ➜ warm sát giờ G + TTL ngắn cho data dễ đổi.
- 🔎 **Quên L1.** Warm L2 (Redis) nhưng **mỗi instance L1 vẫn lạnh** sau mỗi deploy/scale → vẫn miss tầng đầu. ➜ readiness gate warm cả L1, hoặc chấp nhận L1 TTL ngắn tự ấm.
- 🔎 **Readiness gate quá chặt/quá lỏng.** Quá chặt → deploy lê thê; quá lỏng → instance lạnh nhận full tải. ➜ chọn ngưỡng "đủ ấm" theo hit ratio/key count, không cần 100%.
- 🔎 **Warm cái nên là MV.** Phải warm liên tục *mọi thứ* → dấu hiệu nên dùng **materialized view/precompute** thật (Bài 1 g), đừng ép cache đóng vai eager mãi.
- 🔎 **Warm cái KHÔNG nên cache.** Đừng warm **tồn kho/tiền** (cần strong consistency — Bài 1 mục e). Warm phần public, chừa phần cần-đúng-tuyệt-đối.

---

## ⑨ MENTAL MODEL + CÂU THẦN CHÚ

> **Cache warming = TRẢ TRƯỚC cái giá "miss lần đầu" theo lịch của BẠN, để user đầu tiên gặp HIT chứ không MISS.**
> Nó biến **cache lazy thành eager tạm thời** đúng lúc cold-start (deploy / scale / flash sale / failover).
> **Quy tắc vàng: warm TỪ TIER ĐANG ẤM, đừng warm từ DB.** Throttle để việc warm không tự thành stampede.
> **Warm đúng top-K từ LOG, đúng working set — đừng đoán mò, đừng warm thứ cần đúng tuyệt đối.**

---

## ⑩ CÂU HỎI PHỎNG VẤN

- **(dễ)** Cold start là gì, vì sao cache warming giúp? → cache rỗng → mọi request miss → stampede đập DB đúng lúc cao điểm; warm nạp sẵn hot key để user đầu gặp hit.
- **(TB)** Phân biệt **warming** và **refresh-ahead**? → warming là *một lần* lấp cache lạnh; refresh-ahead chạy *liên tục*, làm tươi key nóng **trước** khi TTL hết → không bao giờ nguội.
- **(TB)** Làm sao warm mà **không đập sập DB**? → **warm-from-peer** (đọc từ tier/cụm đang ấm như cách Facebook làm), throttle, singleflight; warm-from-DB là phương án cuối.
- **(TB)** Deploy xong L2 ấm nhưng p99 instance mới vẫn cao — vì sao? → **L1 in-process vẫn lạnh** trên instance mới; cần warm/readiness-gate cả L1 hoặc slow-start.
- **(khó)** Quyết định **warm cái gì**? → **top-K theo tần suất từ log production** (working set), không đoán; đo tới điểm gãy Miss Ratio Curve.
- **(khó)** Rủi ro của warming? → tự gây stampede; warm nhầm key; warm bằng đồ stale; warm nhầm path cần strong consistency.
- **(thách đố)** "Mỗi lần autoscale thêm pod là DB lại spike vài giây" — chữa thế nào? → instance mới L1 lạnh + miss xuống DB; dùng **warm-from-peer (L2/cụm ấm) + readiness gate + slow-start LB**, kèm singleflight để các pod mới không cùng đập một key.

---

## ⑪ GLOSSARY MỞ RỘNG (đóng vào hoàn cảnh)

| Thuật ngữ | Nghĩa nhanh | 🎬 Hoàn cảnh |
|---|---|---|
| **Cache warming** | Nạp sẵn cache trước khi traffic tới | *"Trước flash sale **warm** top SP để tránh miss hàng loạt."* |
| **Cold start / cold cache** | Cache rỗng → miss nhiều, p99 vọt | *"Sau deploy cache **cold**, p99 tăng vài phút đầu."* |
| **Warm-from-peer** | Nạp từ tier/cụm **đang ấm**, không đập DB | *"Cụm lạnh đọc từ cụm ấm (kiểu Facebook), DB khỏi sập."* |
| **Eager preload / batch warm** | Script nạp top-K trước giờ G | *"Cron 11h45 **preload** SKU flash sale 12h."* |
| **Refresh-ahead** | Làm tươi key **trước** khi hết hạn | *"Còn <30s TTL → **refresh nền**, user không chịu miss."* |
| **Readiness gate** | Chưa ấm thì chưa nhận traffic | *"Pod trả 503 ở `/readyz` tới khi warm xong."* |
| **Slow-start / canary ramp** | Tăng traffic từ từ để cache tự lấp | *"LB **slow-start** cho instance mới ấm dần."* |
| **Traffic replay / shadow** | Phát lại log/soi gương traffic để warm | *"**Replay** query log vào cụm mới cho giống production."* |
| **Snapshot restore** | Khởi động nạp lại cache từ đĩa | *"Redis RDB / InnoDB buffer pool **dump+load**."* |
| **Predictive / scheduled warm** | Warm đúng trước sự kiện đã biết | *"Cron **warm theo lịch** mở bán vé."* |
| **Working set** | Tập data thực sự được dùng | *"Chỉ warm **working set**, không warm cả catalog."* |
| **Time-to-warm** | Mất bao lâu cache đủ ấm | *"Đo **time-to-warm** để chỉnh readiness gate."* |
| **Stampede / thundering herd** | Nhiều miss cùng lúc đập nguồn | *"Warm không throttle → tự gây **stampede**."* |
| **Singleflight** | Gộp nhiều miss cùng key thành 1 lần đi nguồn | *"Nhiều pod warm cùng key → **singleflight**."* |
| **Jitter** | Rải TTL ngẫu nhiên chống hết-hạn-đồng-loạt | *"Thêm **jitter** vào TTL key vừa warm."* |

---

> 🧭 **Một dòng để mang theo:** *Cache warming là trả trước "phí miss lần đầu" theo lịch của bạn — warm đúng top-K của working set, warm từ tier đang ấm (không phải từ DB), có throttle để không tự gây stampede, và đừng warm thứ cần đúng tuyệt đối. Nếu phải warm liên tục mọi thứ, có lẽ thứ đó nên là materialized view.*
