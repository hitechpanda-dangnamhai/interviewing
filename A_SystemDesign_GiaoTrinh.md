# 🏛️ Giáo trình A — System Design & Software Architecture (Tech Lead Backend)

> **Workflow 2 — Bước B + C** chạy trên ngân hàng `QB_A_systemdesign.md` (89 câu / 16 mục con).
> Tài liệu tự học một file: dựng giáo trình *ngược* từ bộ đề, rồi giảng trọn 16 bài theo **Hợp đồng đầu ra 10 mục**.
> Ngày biên soạn: 18/06/2026 · Ngôn ngữ: tiếng Việt, giữ thuật ngữ tiếng Anh.

**Quy ước đọc tài liệu**
- 📘 = có/ngụ ý trong bộ đề gốc · ➕ = bổ sung của giảng viên · ⬆️ = làm sâu hơn.
- Mỗi bài kết bằng mục ⑧ (phỏng vấn) + ⑨ (bài tập) để bạn tự test — đây là phần "active recall", hãy che đáp án và trả lời trước.
- Số latency trong tài liệu là **bậc độ lớn** (Jeff Dean's "latency numbers"), không phải con số tuyệt đối; dùng để *biện luận*, không để khẳng định SLA.
- Chi tiết nội bộ bigtech (fan-out, ABR, recommendation…) nêu như **pattern phổ biến**, không khẳng định kiến trúc thật của hãng.

**Câu thần chú xuyên suốt:** *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

---

# 📍 BƯỚC B — Giáo trình ngược từ bộ đề (dependency order)

System Design là mục ráp lại Phase 1–4 của roadmap, nên trong roadmap nó học **cuối**. Nhưng *trong nội bộ* mục A, vẫn có thứ tự dependency riêng: phải vững **quy trình + ước lượng + scaling primitive + đánh đổi** trước, rồi mới "ráp" được các **case study** (URL, feed, chat, checkout…). 16 bài dưới đây xếp theo đúng dây phụ thuộc đó — KHÔNG xếp theo tần suất hỏi.

### Bản đồ 5 khối → 16 bài

| Khối | Bài | Tên bài | Phủ ID | Vì sao đặt ở đây |
|---|---|---|---|---|
| **I. Nền tảng tư duy** | 1 | Quy trình & tư duy làm system design | A-PROC-001→007 | Khung làm bài; mọi bài sau đều chạy trong khung này |
| | 2 | Capacity & back-of-envelope estimation | A-CAP-001→006 | Con số dẫn tới quyết định kiến trúc; cần trước khi vẽ |
| **II. Scaling primitives** | 3 | Horizontal scaling & Load balancing | A-LB-001→007 | Đơn vị mở rộng cơ bản nhất của hệ phân tán |
| | 4 | Building blocks lõi (CDN, hashing, bloom, geo, queue, search, cache layer) | A-BB-001→009 | Bộ "lego" để ráp mọi case study |
| | 5 | Bottleneck & failure modes ở production | A-BOT-001→006 | Dựa trên cache/DB/pool ở Bài 3–4; dạy cách *soi* điểm nghẽn |
| **III. Đánh đổi & ranh giới** | 6 | Trade-off articulation, CAP & PACELC | A-TO-001→007 | Kỹ năng *phân biệt senior với TL*; ngôn ngữ chung của mọi quyết định |
| | 7 | Decomposition & service boundaries | A-DEC-001→004 | Monolith↔microservices; ranh giới service |
| | 8 | Serverless & edge computing | A-SVL-001→005 | Một lựa chọn compute; cần hiểu trade-off để chỉ huy |
| **IV. Case studies (ráp tất cả)** | 9 | URL shortener / pastebin | A-URL-001→005 | Case nhập môn: read-heavy, ID generation, cache |
| | 10 | Rate limiter | A-RL-001→006 | Distributed counter, atomicity, fail-open/closed |
| | 11 | News feed & fan-out | A-FEED-001→006 | Fan-out, celebrity problem, eventual consistency |
| | 12 | Video/feed platform quy mô lớn | A-VID-001→005 | CDN + transcoding pipeline + counter + recommendation |
| | 13 | Chat real-time | A-CHAT-001→005 | WebSocket, connection state, ordering, presence |
| | 14 | Notification system đa kênh | A-NOTI-001→004 | Queue + worker + dedup + preference + fan-out |
| | 15 | E-commerce checkout (chống oversell) | A-CHK-001→005 | Strong consistency, atomic reserve, Saga, idempotency |
| **V. Phán đoán TL** | 16 | Hiểu để chỉ huy & kiểm tra | A-RT-001→002 (+ tổng hợp) | "Đúng trên giấy" vs "đúng ở production"; vai trò TL |

### Ánh xạ ngược: mỗi câu nằm ở bài nào
- A-PROC → **Bài 1** · A-CAP → **Bài 2** · A-LB → **Bài 3** · A-BB → **Bài 4** · A-BOT → **Bài 5**
- A-TO → **Bài 6** · A-DEC → **Bài 7** · A-SVL → **Bài 8**
- A-URL → **Bài 9** · A-RL → **Bài 10** · A-FEED → **Bài 11** · A-VID → **Bài 12** · A-CHAT → **Bài 13** · A-NOTI → **Bài 14** · A-CHK → **Bài 15**
- A-RT → **Bài 16**

### Lộ trình học gợi ý
1. Học tuần tự **Bài 1 → 8** (nền). Đừng nhảy vào case study trước khi vững khối I–III — đó chính là lỗi "nhảy vào vẽ box" mà A-PROC-007 cảnh báo.
2. Từ **Bài 9**, mỗi case study là một bài tập "drive the interview": tự đặt requirement, tự ước lượng, tự chỉ bottleneck.
3. **Bài 16** học cuối — nó là lăng kính TL soi lại 15 bài trước.

---

# 📚 BƯỚC C — Giảng từng bài (Hợp đồng đầu ra 10 mục)

---

## Bài 1 — Quy trình & tư duy làm system design
*Phủ: A-PROC-001 → A-PROC-007*

### ① Mục tiêu & vị trí trong mạch
Bài mở màn khối **Nền tảng tư duy**. Học xong bạn có một **khung 6 bước** chạy được trong 45 phút phỏng vấn, và hiểu vì sao *quy trình* được chấm điểm độc lập với *kiến thức*. Mọi case study (Bài 9–15) đều là bài 1 áp vào một đề cụ thể — nắm vững đây thì 7 case study sau chỉ là "điền nội dung vào khung".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Một bài system design giống như được giao "xây một thành phố" trong 45 phút. Người yếu lao ngay vào vẽ từng tòa nhà (box). Người giỏi hỏi trước: *thành phố này cho bao nhiêu dân, ở đâu, ưu tiên giao thông hay nhà ở?* — rồi mới quy hoạch. Câu hỏi đặt ra (clarify) định hình toàn bộ bản vẽ.

**(b) Cơ chế — khung 6 bước (📘 A-PROC-001):**

1. **Clarify requirements** — chia làm **functional** (hệ *làm gì*: post, follow, redirect…) và **non-functional** (scale, latency, availability, consistency, durability). 📘 A-PROC-002: phải chốt non-functional **trước** vì nó định hình kiến trúc — cần strong consistency thì thiết kế *khác hẳn* so với chấp nhận eventual.
2. **Estimate** — back-of-envelope: DAU → QPS, storage, bandwidth (chi tiết ở Bài 2). Chỉ ước lượng con số *dẫn tới quyết định*.
3. **High-level design** — vẽ các khối lớn: client → LB → service → cache → DB → queue. Đặt tên luồng dữ liệu.
4. **API contract & data model** (📘 A-PROC-006) — sau high-level, **trước** scale. Định nghĩa vài endpoint chính + schema cốt lõi để "neo" thiết kế, tránh nói chung chung lơ lửng.
5. **Scale & deep-dive** — chọn 1–2 phần đào sâu (DB scaling, cache, hot key…).
6. **Bottleneck & trade-off** — tự chỉ điểm nghẽn, nêu đánh đổi của lựa chọn.

**Phân bổ thời gian ~45 phút (📘 A-PROC-004):** ~5' clarify, ~5' estimate, ~10' high-level + API, ~15–20' deep-dive 1–2 phần, ~5' trade-off/bottleneck. Nguyên tắc: **không sa lầy một phần**; quản lý thời gian là kỹ năng được chấm.

**(c) Mép giới hạn & sai lầm (📘 A-PROC-007).** Lý do trượt dù *biết nhiều*: nhảy vào chi tiết quá sớm; không hỏi requirement; im lặng (không "think out loud"); không nêu trade-off; over-engineer (xem Bài 6). ➕ A-PROC-005 — **"drive the interview"**: chủ động đề xuất scope, nói giả định thành tiếng, tự chọn phần deep-dive, tự chỉ bottleneck. Ngồi chờ interviewer hỏi mới nói = điểm trừ vì thiếu *ownership*.

> 📘 vs ➕: khung 6 bước và phân bổ thời gian là chuẩn phổ biến (📘). Ý "drive the interview" và "think out loud như tín hiệu senior" là ➕ nhấn mạnh của giảng viên.

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Cách làm hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Vẽ box ngay khi nghe đề | Clarify functional + non-functional trước | Không có non-functional thì không biết thiết kế cho scale nào |
| Liệt kê mọi công nghệ mình biết | Chỉ đưa thành phần *requirement đòi* | Khoe kiến thức = over-engineer = điểm trừ (A-TO-007) |
| Ước lượng mọi con số cho "đủ bài" | Chỉ ước lượng con số *đổi quyết định* | Estimation trang trí lãng phí thời gian (A-CAP-006) |
| Im lặng suy nghĩ rồi mới nói kết quả | Think out loud suốt quá trình | Interviewer chấm *quá trình tư duy*, không chỉ kết quả |

### ④ Áp dụng thực tế + So sánh bigtech
Khung này không chỉ cho phỏng vấn — nó là cách viết **design doc / RFC** thật ở công ty: *Context & Requirements → Goals/Non-goals → Proposed design → Alternatives considered → Trade-offs → Rollout*. Phần "Alternatives considered" và "Trade-offs" trong RFC chính là bước 6 ở trên. ➕ Pattern phổ biến ở các công ty lớn (Google design doc, Amazon "working backwards"/PRFAQ, các template RFC nội bộ): bắt buộc nêu *non-goals* và *alternatives* — đúng tinh thần "clarify scope + nêu trade-off" của bài này.

### ⑤ Code thực hành + cấu hình
Bài này là *quy trình*, không phải code. Thay vào đó dùng **checklist clarify** dán sẵn (mang vào phỏng vấn / mở đầu mọi design doc):

```text
# CLARIFY CHECKLIST (đọc to khi bắt đầu)
FUNCTIONAL
- [ ] Tính năng cốt lõi (must-have) vs nice-to-have?
- [ ] Ai dùng? Luồng chính end-to-end là gì?
NON-FUNCTIONAL (chốt TRƯỚC khi vẽ)
- [ ] Scale: DAU? QPS avg/peak? read:write ratio?
- [ ] Latency budget (p99)? real-time hay async được?
- [ ] Consistency cần: strong hay eventual? (tiền/tồn kho => strong)
- [ ] Availability target? Durability (mất data có chấp nhận)?
- [ ] Đặc thù data: kích thước item? tăng trưởng/năm?
SCOPE
- [ ] Cái gì NẰM NGOÀI phạm vi (non-goals)?
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** functional vs non-functional, "non-functional định hình kiến trúc", drive the interview, think out loud, YAGNI/over-engineering.
- 📌 **Cần THUỘC:** 6 bước (clarify → estimate → high-level → API/data → scale → trade-off); 5 trục non-functional (scale, latency, availability, consistency, durability).
- 🛠️ **Cần LÀM ĐƯỢC:** chạy clarify checklist; phân bổ thời gian 45'; tự chỉ ra bottleneck của thiết kế mình vừa vẽ.

### ⑦ Mental model
> **"Clarify → Estimate → Sketch → Contract → Scale → Trade-off."** Hỏi trước khi vẽ; non-functional vẽ ra kiến trúc.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Kể 6 bước tiếp cận một bài system design theo thứ tự. → **Gợi ý:** clarify → estimate → high-level → API/data model → scale/deep-dive → bottleneck/trade-off.
2. *(TB)* Functional vs non-functional khác gì, vì sao chốt non-functional trước? → functional = làm gì; non-functional = scale/latency/availability/consistency/durability; nó định hình kiến trúc (strong consistency ⇒ thiết kế khác hẳn).
3. *(khó)* "Drive the interview" nghĩa là gì? → chủ động scope, nêu giả định, tự chọn deep-dive & chỉ bottleneck — thể hiện ownership.
4. *(khó)* Phỏng vấn 45', bạn chia thời gian sao? → 5/5/10/20/5; không sa lầy; chừa thời gian trade-off.
5. *(rất khó)* Vì sao có người *biết nhiều* vẫn trượt vòng này? → quy trình kém: nhảy chi tiết sớm, không clarify, không trade-off, im lặng, over-engineer.

**Thách đố (trick):** *Interviewer ném đề rồi im lặng chờ. Bạn làm gì đầu tiên?* → **Bẫy:** lao vào vẽ. **Đúng:** đặt câu hỏi clarify + nói giả định thành tiếng; im lặng đó là *test xem bạn có drive được không*.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Lấy đề "thiết kế Twitter". Viết ra: 3 functional must-have + 5 non-functional (kèm con số giả định) + 2 non-goals — trong 5 phút. *Đạt khi:* có đủ 5 trục non-functional và mỗi cái có con số/giả định cụ thể, không bỏ trống.
**BT2.** Vẽ timeline 45 phút cho đề đó, ghi rõ phút nào làm gì. *Đạt khi:* có ≥1 deep-dive được cấp ≥10', và có ô riêng cho trade-off cuối.

### ⑩ Đọc thêm
- *System Design Interview* (Alex Xu) — chương "A Framework for System Design Interviews".
- Google Eng Practices — *Design Docs*. AWS — *Working Backwards / PRFAQ*.
- `github.com/donnemartin/system-design-primer` (mục "How to approach a system design interview question").

---

## Bài 2 — Capacity & back-of-envelope estimation
*Phủ: A-CAP-001 → A-CAP-006*

### ① Mục tiêu & vị trí trong mạch
Tiếp khối **Nền tảng tư duy**. Bài 1 dạy *khi nào* ước lượng (bước 2); bài này dạy *cách* ước lượng và — quan trọng hơn — **dùng con số để ra quyết định kiến trúc**. Kết nối thẳng tới Bài 3 (có cần scale ngang không?), Bài 4 (có cần CDN/cache không?), Bài 6 (đánh đổi nào).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Estimation không phải để ra con số *đúng*, mà để trả lời câu hỏi nhị phân: *"Có cần shard không? Có cần cache không? 1 hay nhiều datacenter?"* Sai số 2× không sao; sai *bậc độ lớn* (10×) mới chết. Mục tiêu là **bậc độ lớn**, không phải độ chính xác.

**(b) Cơ chế — 4 phép tính lõi:**

- **QPS (📘 A-CAP-001):** `avg QPS = DAU × số request/user/ngày ÷ 86.400`. `peak QPS ≈ 2–3 × avg` (traffic dồn giờ cao điểm). Ví dụ 10M DAU × 10 req/ngày = 100M req/ngày ÷ 86.400 ≈ **~1.150 avg QPS** → peak **~2.300–3.500**. Luôn **nói giả định ra tiếng** ("giả sử mỗi user 10 request/ngày") thay vì bịa số.
- **Storage (📘 A-CAP-002):** `item/năm × kích thước trung bình × hệ số (replication ×2–3 + metadata)`. Ví dụ 500M ảnh/năm × 1MB × 3 (replica) ≈ **1.5 PB/năm**. Dự trù **nhiều năm** tăng trưởng.
- **Bandwidth (📘 A-CAP-005):** `QPS × payload trung bình = bytes/s`. Ra throughput → biết có cần CDN không, egress bao nhiêu (tiền!), chọn instance/network. *Nối ước lượng với quyết định.*
- **Read:write ratio (📘 A-CAP-003):** read ≫ write → **cache + read replica + CDN**; write ≫ read → **sharding + queue + LSM store** (vd Cassandra). Ratio quyết định *đặt nỗ lực tối ưu ở đâu*.

**(c) Latency orders of magnitude (📘⬆️ A-CAP-004) — "nên thuộc bậc độ lớn".** Các con số kinh điển của Jeff Dean (đây là **bậc độ lớn để biện luận**, không phải SLA — đã verify):

| Thao tác | Bậc độ lớn |
|---|---|
| L1 cache | ~0.5 ns |
| Main memory (RAM) | ~100 ns |
| Đọc 1MB tuần tự từ RAM | ~250 µs |
| SSD random read | ~150 µs |
| Đọc 1MB từ SSD | ~1 ms |
| Round-trip trong cùng datacenter | ~0.5 ms |
| HDD seek | ~10 ms |
| Round-trip xuyên lục địa (CA↔EU) | ~100–150 ms |

Main memory reference ~100 ns; round trip within same datacenter ~500 µs; HDD seek ~10 ms; gói tin CA→Netherlands→CA ~150 ms. Quy tắc nhớ: **RAM (ns) ≪ SSD (µs) ≪ disk/network (ms) ≪ cross-region (chục–trăm ms)**. Dùng để biện luận: vì sao cache RAM đáng giá (rẻ hơn DB disk ~1000×), vì sao cross-region call phải cẩn thận, vì sao đặt replica gần user.

**Mép giới hạn (📘 A-CAP-006):** chỉ ước lượng con số *dẫn tới quyết định*. Ước lượng "trang trí" không đổi thiết kế = lãng phí thời gian quý trong phỏng vấn. Hỏi bản thân: *"Con số này khiến tôi chọn khác đi không?"* — nếu không, bỏ.

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Tính ra con số *chính xác* nhiều chữ số | Làm tròn về bậc độ lớn | Mục tiêu là *quyết định*, không phải kế toán |
| Bịa số không nói giả định | Nêu giả định to, rõ | Interviewer chấm *cách suy luận*, số sai-có-cơ-sở vẫn đạt |
| Quên hệ số replication/metadata khi tính storage | ×2–3 cho replica + metadata | Thiếu hệ số ⇒ sai bậc độ lớn |
| Bỏ qua peak, chỉ tính avg | peak ≈ 2–3× avg | Hệ sập vì peak, không vì avg |

### ④ Áp dụng thực tế + So sánh bigtech
Estimation thật xảy ra ở **capacity planning** trước khi launch và ở **cost review** (FinOps). Ví dụ: bandwidth estimation quyết định bài toán *egress cost* — đẩy ảnh/video qua CDN thay vì origin có thể cắt chi phí egress đáng kể, đây là lý do mọi nền tảng media-heavy đều CDN-first (pattern phổ biến). ➕ Ở công ty, con số QPS/storage còn dùng để chọn **instance type** và đặt **autoscaling threshold** — cùng một phép tính bài 2.

### ⑤ Code thực hành + cấu hình
Script ước lượng nhanh để luyện trực giác (chạy được, không cần thư viện ngoài):

```python
# capacity_estimate.py — chạy: python capacity_estimate.py
# Không có dependency ngoài; số ra là BẬC ĐỘ LỚN để biện luận, không phải SLA.

def estimate(dau, req_per_user_per_day, payload_kb,
             item_per_year, item_size_mb, replication=3, peak_factor=3):
    sec_per_day = 86_400
    avg_qps = dau * req_per_user_per_day / sec_per_day
    peak_qps = avg_qps * peak_factor
    bandwidth_mb_s = peak_qps * payload_kb / 1024          # MB/s ở peak
    storage_pb_year = item_per_year * item_size_mb * replication / 1024 / 1024 / 1024  # MB->PB

    print(f"avg QPS   ≈ {avg_qps:,.0f}")
    print(f"peak QPS  ≈ {peak_qps:,.0f}  (x{peak_factor})")
    print(f"bandwidth ≈ {bandwidth_mb_s:,.1f} MB/s ở peak  -> cần CDN nếu lớn")
    print(f"storage   ≈ {storage_pb_year:,.3f} PB/năm (đã x{replication} replica)")
    # Quy tắc quyết định (heuristic, KHÔNG phải luật cứng):
    if peak_qps > 10_000: print("=> peak QPS cao: cân nhắc shard/scale ngang DB")
    if bandwidth_mb_s > 100: print("=> bandwidth lớn: CDN gần như bắt buộc")

if __name__ == "__main__":
    # Ví dụ: 10M DAU, 10 req/ngày, payload 5KB; 500M ảnh/năm @1MB
    estimate(dau=10_000_000, req_per_user_per_day=10, payload_kb=5,
             item_per_year=500_000_000, item_size_mb=1)
```

> ⚠️ Đây là công cụ *luyện trực giác*; trong phỏng vấn bạn nhẩm tay, không gõ script.

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** "estimate để ra quyết định, không để ra con số đúng"; vì sao read:write ratio định hình kiến trúc; latency hierarchy.
- 📌 **Cần THUỘC:** 86.400 s/ngày; peak ≈ 2–3× avg; RAM ~100ns / SSD ~µs / DC round-trip ~0.5ms / cross-region ~100ms.
- 🛠️ **Cần LÀM ĐƯỢC:** nhẩm QPS/storage/bandwidth từ DAU; phát biểu giả định to; rút ra "cần cache/shard/CDN không".

### ⑦ Mental model
> **"DAU → QPS → bytes → quyết định."** Con số chỉ đáng tính nếu nó đổi thiết kế. RAM rẻ hơn disk ~1000× về latency.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* 10M DAU, mỗi user 10 request/ngày — avg & peak QPS? → ~1.150 avg, ~2.300–3.500 peak.
2. *(TB)* Read:write ratio ảnh hưởng kiến trúc thế nào? → read-heavy ⇒ cache+replica+CDN; write-heavy ⇒ shard+queue+LSM.
3. *(TB)* Vì sao RAM cache đáng giá so với đọc DB? → RAM ~100ns vs disk ~ms ⇒ nhanh hơn ~1000× bậc độ lớn.
4. *(khó)* Storage cho 500M ảnh/năm @1MB? → ×3 replica ≈ 1.5PB/năm; dự trù nhiều năm.
5. *(rất khó)* Khi nào estimation là lãng phí? → khi con số không đổi quyết định kiến trúc nào.

**Thách đố (trick):** *Bạn tính ra 1.150 QPS và kết luận "cần 50 server".* Có gì sai? → **Bẫy:** nhảy từ QPS sang số server mà chưa biết *1 server gánh bao nhiêu QPS* (phụ thuộc payload, CPU/IO). Phải ước lượng per-server capacity trước; và phải dùng **peak**, không phải avg.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Ước lượng QPS, storage/năm, bandwidth peak cho "Instagram-lite": 50M DAU, mỗi user xem 50 ảnh + đăng 0.2 ảnh/ngày, ảnh ~1.5MB. *Đạt khi:* tách read (xem) và write (đăng), nêu giả định, ra bậc độ lớn hợp lý, kết luận "cần CDN + object storage".
**BT2.** Cho một con số bạn vừa tính, chỉ ra *quyết định kiến trúc* nó dẫn tới. *Đạt khi:* mỗi con số gắn với đúng một quyết định (cache / shard / CDN / replica).

### ⑩ Đọc thêm
- Jeff Dean — "Latency Numbers Every Programmer Should Know" (gist `jboner/2841832`; bản tương tác `colin-scott.github.io/.../interactive_latency.html`).
- *System Design Interview* (Alex Xu) — chương "Back-of-the-envelope estimation".

---

## Bài 3 — Horizontal scaling & Load balancing
*Phủ: A-LB-001 → A-LB-007*

### ① Mục tiêu & vị trí trong mạch
Mở khối **Scaling primitives**. Đây là đơn vị mở rộng cơ bản nhất của mọi hệ phân tán: thêm máy thay vì máy to hơn. Nắm bài này thì các case study (Bài 9–15) đều bắt đầu từ "đặt LB trước cụm stateless service". Kết nối Bài 5 (LB là bottleneck/SPOF) và Bài 8 (serverless tự lo phần này).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Vertical scaling = thuê một đầu bếp giỏi hơn; horizontal = thuê thêm nhiều đầu bếp. Đầu bếp giỏi có trần (không ai nấu nhanh vô hạn) và nếu ốm thì bếp đóng cửa (SPOF). Nhiều đầu bếp mở rộng gần vô hạn và một người ốm vẫn chạy — nhưng cần *người điều phối order* (load balancer) và các đầu bếp **không được giữ ghi chú riêng** (stateless).

**(b) Cơ chế.**
- 📘 A-LB-001 — **Horizontal > vertical**: vertical có trần phần cứng + SPOF + downtime khi nâng; horizontal mở rộng gần vô hạn + chịu lỗi, đổi lại đòi **stateless + LB + phối hợp**.
- 📘 A-LB-002 — **Stateless service**: instance không giữ session state; *bất kỳ instance phục vụ được request bất kỳ*; state đẩy ra ngoài (Redis/DB). Đây là **điều kiện** để scale ngang (thêm/bớt node tự do).
- 📘 A-LB-003 — **Sticky session** (ghim user vào 1 instance) thường nên **tránh**: cản rebalance, mất session khi node chết, lệch tải. Thay bằng **external session store** (Redis).
- 📘 A-LB-004 — **L4 vs L7 LB**: L4 route theo IP/port (nhanh, không hiểu nội dung); L7 hiểu HTTP (route theo path/header, TLS termination, retry, sticky theo cookie) nhưng tốn hơn. Chọn L7 khi cần routing thông minh / TLS offload; L4 khi cần thuần throughput.
- 📘 A-LB-005 — **Thuật toán LB**: round-robin (request đồng đều, đơn giản); least-connections (request không đều thời lượng); hash theo key (giữ affinity/cache locality, vd shard theo user-id).
- 📘 A-LB-006 — **Health check & graceful drain**: LB ngừng route tới node unhealthy; khi scale-in/rolling deploy phải **drain** connection đang phục vụ trước khi tắt — nếu không sẽ ra 5xx hàng loạt.

**(c) Mép giới hạn (📘 A-LB-007).** Bản thân LB là **SPOF**. Làm tầng LB highly available: nhiều LB + **DNS round-robin / anycast / floating IP / active-passive failover**. Một LB chết không được kéo sập toàn hệ. ➕ Thực tế cloud: managed LB (ALB/NLB, GCLB…) đã HA sẵn ở tầng dưới, nhưng nguyên lý "đừng để 1 điểm chết kéo sập" vẫn phải hiểu để chỉ huy.

> 📘 vs ➕: toàn bộ là kiến thức ổn định trong bộ đề (📘). Phần managed LB và "external session store là mặc định hiện nay" là ➕ cập nhật thực hành.

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Cũ / hay gặp | Nên dùng nay | Vì sao |
|---|---|---|
| Sticky session để giữ login | External session store (Redis/JWT) | Sticky cản scale & mất session khi node chết |
| Lưu state trong RAM của app instance | Đẩy state ra Redis/DB; instance stateless | Mới scale ngang được |
| Scale dọc khi tải tăng | Scale ngang + stateless trước | Vertical có trần + downtime + SPOF |
| Một LB duy nhất | Cụm LB + DNS/anycast/floating IP | LB là SPOF |

### ④ Áp dụng thực tế + So sánh bigtech
Mọi web service production đều theo pattern **DNS → (anycast) → LB → autoscaling group of stateless instances → shared state store**. ➕ Pattern phổ biến: cloud LB managed (AWS ALB L7 / NLB L4, GCP Cloud Load Balancing, Cloudflare LB) lo health check + drain + multi-AZ. Trên Kubernetes, vai trò LB chia thành Service (L4) + Ingress/Gateway (L7); session đẩy ra Redis. Chi tiết sản phẩm có thể đổi — *verify tên/đời khi cần*.

### ⑤ Code thực hành + cấu hình
Minh hoạ **stateless + external session** (Express + Redis). Điểm cốt lõi: state KHÔNG nằm trong process.

```javascript
// app.js — chạy N instance sau LB; session ở Redis nên scale ngang tự do
// npm i express express-session connect-redis redis
const express = require('express');
const session = require('express-session');
const { RedisStore } = require('connect-redis');   // kiểm chứng API tại docs connect-redis (đã đổi qua các đời)
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL }); // KHÔNG hardcode
redisClient.connect();

const app = express();
app.use(session({
  store: new RedisStore({ client: redisClient }),   // <- state ra ngoài, instance stateless
  secret: process.env.SESSION_SECRET,               // từ env/secret manager
  resave: false, saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, maxAge: 3600_000 },
}));

app.get('/healthz', (_req, res) => res.sendStatus(200)); // LB health check + graceful drain dựa vào đây
app.get('/me', (req, res) => {
  req.session.views = (req.session.views || 0) + 1;       // đọc/ghi session ở Redis, không ở RAM instance
  res.json({ views: req.session.views, servedBy: process.env.HOSTNAME });
});
app.listen(3000);
```

```text
# requirements/cấu hình
- ĐẶT REDIS_URL, SESSION_SECRET qua biến môi trường / secret manager (không commit).
- LB cấu hình health check trỏ /healthz; bật connection draining khi scale-in/deploy.
- Bật autoscaling theo CPU/QPS; mọi instance phải qua được test "tắt 1 node, request vẫn ok".
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao stateless là điều kiện scale ngang; L4 vs L7 khi nào; vì sao sticky session hại; LB là SPOF.
- 📌 **Cần THUỘC:** 3 thuật toán LB (round-robin / least-conn / hash); health check + graceful drain; cách HA tầng LB (DNS RR/anycast/floating IP).
- 🛠️ **Cần LÀM ĐƯỢC:** đặt LB + cụm stateless + external session; cấu hình health check; chứng minh "tắt 1 node hệ vẫn chạy".

### ⑦ Mental model
> **"Stateless + LB = scale ngang."** Đẩy state ra ngoài; đừng để node nào (kể cả LB) là điểm chết duy nhất.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Vì sao ưu tiên scale ngang hơn dọc? → vertical: trần + SPOF + downtime; horizontal: gần vô hạn + chịu lỗi (cần stateless+LB).
2. *(TB)* Stateless nghĩa là gì, sao là điều kiện scale ngang? → không giữ session state; instance nào cũng phục vụ được; state ở Redis/DB.
3. *(TB)* L4 vs L7 LB? → L4 IP/port nhanh; L7 hiểu HTTP, route path/header + TLS, tốn hơn.
4. *(khó)* Vì sao tránh sticky session, thay bằng gì? → cản rebalance/mất session/lệch tải; external session store.
5. *(khó)* Graceful drain là gì, vì sao cần khi deploy? → drain connection đang phục vụ trước khi tắt node ⇒ tránh 5xx khi rolling deploy/scale-in.

**Thách đố (trick):** *"Em dùng round-robin nên tải chia đều rồi."* — sai chỗ nào? → **Bẫy:** round-robin chia đều *số request* chứ không đều *công sức*; request dài-ngắn khác nhau ⇒ node nhận toàn request nặng vẫn quá tải. Dùng **least-connections** cho workload không đều.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Vẽ kiến trúc tối thiểu cho web app 5 instance: chọn loại LB, thuật toán, nơi để session, health check. *Đạt khi:* session ngoài process, có health check + drain, giải thích chọn L4 hay L7.
**BT2.** Mô tả điều gì xảy ra khi 1 instance chết giữa lúc đang phục vụ, và LB chết. Nêu cách phòng cả hai. *Đạt khi:* nhắc external session (mất instance không mất session) + HA tầng LB.

### ⑩ Đọc thêm
- NGINX docs — *HTTP Load Balancing*; AWS — *Elastic Load Balancing* (ALB vs NLB).
- Kubernetes docs — *Service / Ingress / Gateway API*.

---

## Bài 4 — Building blocks lõi
*Phủ: A-BB-001 → A-BB-009*

### ① Mục tiêu & vị trí trong mạch
Bài "hộp lego" của khối **Scaling primitives**. Tập hợp các kỹ thuật lõi mà mọi case study sau dùng đi dùng lại: CDN, object storage, consistent hashing, bloom filter, geo index, message queue, full-text search, và các tầng cache. Học xong, khi gặp đề bạn *gọi tên đúng lego* thay vì phát minh lại.

### ② Giảng cơ bản → nâng cao (gộp theo nhóm lego)

**Nhóm phân phối nội dung & lưu trữ lớn**
- 📘 A-BB-001 — **CDN**: cache static/asset gần user ⇒ giảm latency + giảm tải origin. **Đặt lên CDN:** ảnh, JS/CSS, video, asset bất biến. **KHÔNG đặt:** dữ liệu cá nhân/động/nhạy cảm. Cân nhắc **TTL + invalidation** (cache invalidation là phần khó).
- 📘 A-BB-002 — **Object storage (vd S3)** vs database: object storage lưu **file lớn** rẻ/bền/scale; DB chỉ giữ **metadata + URL**. KHÔNG nhồi blob (ảnh/video) vào DB. Phục vụ qua **CDN / presigned URL**. *(verify tên dịch vụ khi cần)*

**Nhóm phân phối key qua node**
- 📘 A-BB-003 — **Consistent hashing**: phân phối key qua node trên một *vòng hash*. Khi thêm/bớt node chỉ **remap phần nhỏ** key, không reshuffle gần như toàn bộ như `hash % N` (modulo: đổi N là gần như mọi key đổi node ⇒ cache miss bão hoà). Nền tảng cho cache cluster / shard cluster (Redis Cluster, Cassandra, DynamoDB partitioning).
- 📘 A-BB-004 — **Virtual nodes**: rải mỗi physical node thành *nhiều điểm* trên vòng ⇒ phân phối đều hơn, tránh hot node, rebalance mượt khi node chết/thêm.

**Nhóm cấu trúc dữ liệu thông minh**
- 📘 A-BB-005 — **Bloom filter**: cấu trúc xác suất trả "**có thể có** / **chắc chắn không**" — có **false positive**, KHÔNG false negative. Dùng để **lọc trước** né đụng DB/cache: chống *cache penetration* (key không tồn tại đập thẳng DB), check tồn tại nhanh (vd username đã dùng chưa, URL đã crawl chưa).
- 📘 A-BB-006 — **Geo / nearby index**: dùng **geohash / quad-tree / S2** biến tọa độ 2D thành key truy vấn được; query theo ô lưới lân cận. KHÔNG quét toàn bảng tính khoảng cách từng điểm.

**Nhóm tách rời & tìm kiếm**
- 📘 A-BB-007 — **Message queue** *(giao với K — đào sâu ở Bài về Messaging)*: thêm vào để **decouple** producer/consumer, **hấp thụ burst** (buffer), xử lý **async + retry**. Đổi lại: thêm latency + eventual consistency + phức tạp vận hành. Tín hiệu nên thêm queue: tác vụ nặng/chậm không cần đồng bộ (gửi mail, transcode, fan-out).
- 📘 A-BB-008 — **Full-text search**: cần tìm văn bản ⇒ thêm **Elasticsearch/OpenSearch** (inverted index + relevance + scale), KHÔNG `LIKE '%x%'` (không dùng được index ⇒ quét tuyến tính chậm). Đổi lại: phải **đồng bộ dữ liệu** từ DB nguồn (qua CDC/dual-write) + vận hành thêm một hệ. *(verify tên/đời)*

**Nhóm tổng hợp**
- 📘⬆️ A-BB-009 — **Cache đặt ở nhiều tầng** *(giao với J — Bài về Caching)*: client → CDN/edge → API gateway → app local (in-process) → distributed (Redis) → DB cache. Mỗi tầng giảm tải tầng dưới nhưng **thêm bài toán invalidation/stale**. Chọn tầng theo *read pattern*: data toàn cục ít đổi ⇒ CDN; per-user nóng ⇒ Redis; cực nóng cục bộ ⇒ in-process (chấp nhận stale ngắn).

> 📘 vs ➕: tất cả lego là kiến thức ổn định (📘). Ý "invalidation là phần khó nhất của cache", "CDC để đồng bộ search" là ➕ nhấn mạnh.

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Cũ / hay gặp | Nên dùng | Vì sao |
|---|---|---|
| `hash % N` để chia key qua cache/shard | **Consistent hashing + virtual nodes** | Đổi N ⇒ remap gần như toàn bộ key (cache miss bão) |
| Lưu ảnh/video trong cột BLOB của DB | Object storage + DB giữ metadata/URL | DB phình, đắt, khó scale; blob nên ở storage rẻ + CDN |
| `LIKE '%keyword%'` cho search | Inverted index (Elastic/OpenSearch) | `%x%` không xài index ⇒ chậm tuyến tính |
| Cache mọi thứ "cho nhanh" | Cache theo read pattern + có chiến lược invalidation | Stale/invalidation sai gây bug khó chịu hơn cả chậm |

### ④ Áp dụng thực tế + So sánh bigtech
- **Media-heavy** (ảnh/video): object storage + CDN là chuẩn ngành (pattern phổ biến) vì cắt egress & latency.
- **Cache/shard cluster**: consistent hashing là nền tảng của Redis Cluster, Cassandra, DynamoDB (partition key + virtual nodes) — *verify chi tiết từng sản phẩm khi cần*.
- **"Nearby"** (ride-hailing, food delivery): geohash/S2 cho truy vấn lân cận; bloom filter hay gặp ở tầng chống cache penetration và dedup crawler.

### ⑤ Code thực hành + cấu hình
**Consistent hashing tối giản** để thấy "thêm node remap ít key":

```python
# consistent_hash.py — chạy: python consistent_hash.py  (chỉ stdlib)
import hashlib, bisect

class ConsistentHash:
    def __init__(self, nodes=(), vnodes=100):   # vnodes = virtual nodes/physical node
        self.vnodes = vnodes
        self.ring = {}            # hash điểm -> node
        self.sorted_keys = []     # vòng hash đã sort
        for n in nodes: self.add(n)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)  # md5 chỉ để phân phối, KHÔNG bảo mật

    def add(self, node):
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)

    def get(self, key):           # key thuộc node nào
        if not self.ring: return None
        h = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

if __name__ == "__main__":
    ring = ConsistentHash(["A", "B", "C"], vnodes=200)
    sample = [f"key-{i}" for i in range(10_000)]
    before = {k: ring.get(k) for k in sample}
    ring.add("D")                                  # thêm node
    after = {k: ring.get(k) for k in sample}
    moved = sum(1 for k in sample if before[k] != after[k])
    print(f"Thêm 1 node (3->4): chỉ ~{moved/len(sample)*100:.1f}% key phải remap")
    # Kỳ vọng ~ 1/4 ~ 25% — so với modulo gần như 100%
```

```text
# Lưu ý cấu hình cho lego thật:
- CDN: đặt TTL hợp lý + cơ chế purge/invalidation khi asset đổi (versioned URL: app.v3.js).
- Object storage: dùng presigned URL cho upload/download trực tiếp, KHÔNG proxy blob qua app.
- Search: dựng pipeline đồng bộ (CDC/Debezium hoặc outbox) DB -> index; KHÔNG dual-write tay dễ lệch.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao modulo hashing tệ khi đổi N; bloom filter "false positive nhưng không false negative"; vì sao blob ra object storage; cache nhiều tầng đánh đổi stale.
- 📌 **Cần THUỘC:** consistent hashing + virtual nodes; geohash/quad-tree/S2; inverted index; presigned URL; các tầng cache (client/CDN/gateway/local/distributed/DB).
- 🛠️ **Cần LÀM ĐƯỢC:** chọn đúng lego cho một yêu cầu; cài consistent hashing; thiết kế pipeline đồng bộ DB→search.

### ⑦ Mental model
> **"Đừng phát minh lại lego."** Blob→object storage+CDN; chia key→consistent hashing; tồn tại?→bloom; nearby→geohash; text→inverted index; nặng/async→queue.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* CDN cache cái gì, KHÔNG cache cái gì? → static/asset gần user; không cache data cá nhân/động/nhạy cảm.
2. *(TB)* Consistent hashing giải quyết gì mà modulo không? → thêm/bớt node chỉ remap phần nhỏ key thay vì gần toàn bộ.
3. *(TB)* Virtual nodes để làm gì? → phân phối đều + tránh hot node + rebalance mượt.
4. *(khó)* Bloom filter dùng ở đâu, đặc tính gì? → lọc trước chống cache penetration/dedup; false positive có, false negative không.
5. *(khó)* Vì sao không `LIKE '%x%'`, thêm gì, trả giá gì? → không xài index ⇒ chậm; thêm inverted index; phải đồng bộ data + vận hành thêm hệ.

**Thách đố (trick):** *Bloom filter báo "có" username `john` ⇒ bạn từ chối đăng ký luôn?* → **Bẫy:** false positive ⇒ "có thể có" không chắc chắn có. Phải **đi xác nhận ở DB** khi bloom nói "có"; chỉ tin tuyệt đối khi bloom nói "**chắc chắn không**".

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Chạy `consistent_hash.py`, đổi `vnodes` 1 → 200, quan sát độ lệch phân phối & % remap. *Đạt khi:* giải thích được vì sao vnodes lớn ⇒ phân phối đều hơn.
**BT2.** Cho yêu cầu "upload + phục vụ ảnh + tìm theo caption + đề xuất ảnh gần vị trí". Liệt kê lego cho từng phần. *Đạt khi:* ảnh→object storage+CDN; caption→inverted index; gần vị trí→geohash; (và metadata→DB).

### ⑩ Đọc thêm
- Karger et al. — *Consistent Hashing and Random Trees* (paper gốc).
- AWS S3 docs (presigned URL); Elastic/OpenSearch docs (inverted index); Uber Eng blog — H3 (geo indexing). *(verify đời/sản phẩm khi trích số liệu)*

---

## Bài 5 — Bottleneck & failure modes ở production
*Phủ: A-BOT-001 → A-BOT-006*

### ① Mục tiêu & vị trí trong mạch
Khép khối **Scaling primitives**. Bài 3–4 cho bạn các thành phần; bài này dạy cách **soi điểm nghẽn** và nhận diện các *failure mode chỉ lộ ở production* (hot key, stampede, pool cạn). Đây là cầu nối tới Bài 16 ("đúng trên giấy" vs "đúng ở production") và giao nhiều với mục O (observability), I (concurrency), J (caching).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Khi interviewer hỏi *"bottleneck của thiết kế anh vừa vẽ là gì?"*, họ kiểm tra xem bạn có *nhìn được nơi tải dồn* không. Bottleneck luôn ở **tầng có tải/đồng thời cao nhất** hoặc **state tập trung** (chỗ mọi request phải đi qua một điểm).

**(b) Cơ chế — soi theo thứ tự.**
- 📘 A-BOT-001 — **Soi từ đâu trước**: tầng có concurrency cao nhất hoặc state tập trung — DB ghi, single LB, cache, hot key/partition. Chỉ rõ *điểm cụ thể + cách giảm*, không nói chung chung "cần scale".
- 📘 A-BOT-002 — **DB thường là bottleneck đầu tiên** (ghi/đọc tập trung). Nấc giảm tải **rẻ → đắt**: `index → cache → read replica → sharding → CQRS/queue`. KHÔNG nhảy thẳng vào shard (shard đắt về vận hành & mất join).
- 📘 A-BOT-003 — **Hot partition / hot key**: 1 shard/key nhận tải lệch (celebrity, popular item, key thiết kế xấu kiểu `country=VN`). Phát hiện qua phân phối lệch (p99 một shard cao). Fix: shard key tốt hơn, **salting** (thêm hậu tố ngẫu nhiên), dedicated cache cho key nóng, replicate read.
- 📘 A-BOT-005 — **Connection/thread pool cạn** *(giao với E/I)*: request xếp hàng chờ connection ⇒ latency tăng, timeout **lan cả hệ** (cascading). Fix: chỉnh pool size theo DB limit, tách pool theo loại tải, **circuit breaker** để cắt sớm thay vì kéo chết dây chuyền.
- 📘 A-BOT-006 — **Thundering herd / cache stampede** *(giao với J)*: nhiều key cùng hết hạn ⇒ request đồng loạt miss dồn vào DB. Fix: **lock/single-flight** (chỉ 1 request đi tính, số còn lại chờ), **jittered TTL** (rải hạn ngẫu nhiên), **stale-while-revalidate** (trả bản cũ trong lúc làm mới), **pre-warm** cache.

**(c) Mép giới hạn — debug đúng cách (📘 A-BOT-004, *giao với O*).** "Hệ chậm ở production" — **đo trước, đừng đoán**: metrics/tracing/p99 → tìm tầng latency cao → soi queue depth / CPU / connection pool / DB slow query. **Data-driven**, không sửa theo cảm tính. Đây là khác biệt giữa "đoán mò vá lung tung" và "TL điều tra có phương pháp".

> 📘 vs ➕: các failure mode đều trong bộ đề (📘). Ý "circuit breaker cắt cascading", "single-flight" là ➕ làm sâu.

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| "Hệ chậm thì thêm server" | Đo trước (p99, tracing) rồi mới sửa đúng tầng | Thêm server không cứu hot key / pool cạn / DB lock |
| Gặp tải cao là shard DB ngay | index → cache → replica → mới shard | Shard đắt; đa số ca giải quyết ở nấc rẻ hơn |
| Đặt TTL bằng nhau cho mọi key | Jittered TTL + single-flight | TTL đồng loạt ⇒ stampede khi hết hạn cùng lúc |
| Pool to vô tội vạ "cho chắc" | Pool theo DB limit + circuit breaker | Pool quá lớn làm sập DB; quá nhỏ gây nghẽn |

### ④ Áp dụng thực tế + So sánh bigtech
- **Celebrity hot key** là failure mode kinh điển ở mạng xã hội (1 user/triệu follower hoặc 1 post viral) — giải bằng dedicated cache + (ở feed) hybrid fan-out (Bài 11).
- **Cache stampede** xảy ra mỗi khi một cache phổ biến hết hạn đồng loạt sau deploy/restart; mọi hệ read-heavy đều phải có chống stampede.
- **Cascading failure do pool cạn** là nguyên nhân nhiều outage lớn; circuit breaker + timeout + bulkhead là pattern phòng vệ phổ biến (Netflix Hystrix là ví dụ lịch sử; nay thường dùng resilience4j / service mesh — *verify đời công cụ*).

### ⑤ Code thực hành + cấu hình
**Single-flight chống cache stampede** (Python, dùng asyncio lock — minh hoạ ý tưởng, không hardcode):

```python
# single_flight.py — gộp các request cùng miss vào 1 lần tính DB
import asyncio, random, time

cache = {}                       # demo in-memory; thực tế là Redis
locks = {}                       # 1 lock / key
inflight_db_calls = 0

async def load_from_db(key):     # giả lập query DB tốn 200ms
    global inflight_db_calls
    inflight_db_calls += 1
    await asyncio.sleep(0.2)
    return f"value-of-{key}"

async def get(key, ttl=5):
    now = time.time()
    item = cache.get(key)
    if item and item["exp"] > now:
        return item["val"]                       # cache hit
    lock = locks.setdefault(key, asyncio.Lock())
    async with lock:                              # single-flight: chỉ 1 coroutine vào DB
        item = cache.get(key)                     # double-check sau khi giành lock
        if item and item["exp"] > now:
            return item["val"]
        val = await load_from_db(key)
        jitter = random.uniform(0, ttl * 0.2)     # jittered TTL chống hết hạn đồng loạt
        cache[key] = {"val": val, "exp": now + ttl + jitter}
        return val

async def main():
    await asyncio.gather(*[get("hot-key") for _ in range(100)])  # 100 request cùng miss
    print(f"100 request đồng thời nhưng chỉ {inflight_db_calls} lần gọi DB")  # kỳ vọng: 1

asyncio.run(main())
```

```text
# Cấu hình chống các failure mode (checklist):
- TTL: thêm jitter (±10–20%); bật stale-while-revalidate nếu hệ chịu được data hơi cũ.
- Pool: maxConnections theo (DB max_connections × % an toàn); tách pool đọc/ghi; đặt acquire-timeout ngắn.
- Circuit breaker: mở khi error-rate/latency vượt ngưỡng; half-open thử lại; tránh cascading.
- Observability: bật tracing + p99 dashboard + alert trên queue depth & pool saturation (giao với O).
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** "đo trước khi sửa"; cascading failure từ pool cạn; vì sao hot key phá sharding; vì sao TTL đồng loạt gây stampede.
- 📌 **Cần THUỘC:** nấc giảm tải DB (index→cache→replica→shard→CQRS); fix hot key (salting/dedicated cache); fix stampede (single-flight/jitter/SWR/pre-warm); circuit breaker.
- 🛠️ **Cần LÀM ĐƯỢC:** chỉ đúng bottleneck của một sơ đồ; debug data-driven; viết single-flight + jittered TTL.

### ⑦ Mental model
> **"Bottleneck nằm ở chỗ state tập trung / tải dồn."** Đo trước, sửa nấc rẻ trước, chặn cascading bằng circuit breaker.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* "Bottleneck của thiết kế này là gì?" — soi từ đâu? → tầng concurrency cao nhất / state tập trung; chỉ điểm cụ thể + cách giảm.
2. *(TB)* DB là bottleneck — các nấc giảm tải theo thứ tự? → index → cache → read replica → sharding → CQRS/queue.
3. *(khó)* Hot key là gì, fix sao? → 1 key/shard tải lệch; salting, dedicated cache, shard key tốt hơn, replicate read.
4. *(khó)* Cache stampede phòng sao? → single-flight/lock, jittered TTL, stale-while-revalidate, pre-warm.
5. *(rất khó)* "Hệ chậm ở production" — debug từ đâu? → đo p99/tracing trước; tìm tầng latency cao; soi queue/CPU/pool/slow query; data-driven.

**Thách đố (trick):** *Thêm cache xong hệ vẫn sập mỗi sáng 8h khi cache nguội.* Vì sao, fix? → **Bẫy:** nghĩ cache là xong. Thực ra cache cold + nhiều key hết hạn cùng lúc ⇒ **stampede** + miss dồn DB. Fix: pre-warm trước giờ cao điểm + jittered TTL + single-flight.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Chạy `single_flight.py`; tắt lock (cho mỗi request tự load) và so số lần gọi DB. *Đạt khi:* giải thích vì sao có lock thì 100 request ⇒ 1 lần DB.
**BT2.** Cho sơ đồ "client → LB → app → Postgres" read-heavy bị chậm lúc cao điểm. Liệt kê 4 nghi phạm bottleneck và cách đo từng cái. *Đạt khi:* nêu được DB slow query, pool saturation, cache miss/stampede, hot key — kèm metric để xác nhận.

### ⑩ Đọc thêm
- Google SRE Book — chương *Addressing Cascading Failures*.
- AWS Builders' Library — *Avoiding fallback in distributed systems* & *timeouts/retries/backoff with jitter*.
- resilience4j docs (circuit breaker, bulkhead). *(verify đời công cụ)*

---

## Bài 6 — Trade-off articulation, CAP & PACELC
*Phủ: A-TO-001 → A-TO-007*

### ① Mục tiêu & vị trí trong mạch
Mở khối **Đánh đổi & ranh giới** — và là bài *phân biệt senior với TL thật*. Mọi quyết định ở case study (Bài 9–15) đều phải phát biểu được bằng ngôn ngữ trade-off. Học xong bạn có **khung biện luận** để trả lời thuyết phục câu kinh điển: *"Vì sao A mà không B, anh đánh đổi gì?"*

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Trong system design **không có "tốt nhất", chỉ có "phù hợp nhất với requirement"**. Mọi lựa chọn đều cắt một thứ để được thứ khác. TL giỏi không khoe "A tốt hơn" — họ nói "với requirement X, tôi chọn A, chấp nhận mất Y".

**(b) Cơ chế.**
- 📘 A-TO-001 — **CAP theorem** & vì sao "chọn 2 trong 3" gây hiểu lầm: P (network partition) là thứ **luôn có thể xảy ra** trong hệ phân tán, không phải lựa chọn. Nên thực chất: **khi CÓ partition, phải chọn C hay A**. Lúc *không* partition vẫn có đánh đổi C/A. ➕ **PACELC** mở rộng đúng hơn: *if Partition then (C or A), Else (Latency or C)* — kể cả lúc bình thường, strong consistency vẫn đánh đổi latency. Không phải "vứt hẳn 1 cái vĩnh viễn".
- 📘 A-TO-002 — **Strong vs eventual consistency**: strong cho **tiền/tồn kho/đặt vé** (phải đúng ngay); eventual cho **feed/like/đếm view** (trễ vài giây vô hại). Đánh đổi: strong ⇒ ↑latency, ↓availability khi partition; eventual ⇒ nhanh/sẵn sàng nhưng có thể đọc data cũ. **Phải nêu ví dụ cụ thể**, không nói lý thuyết suông.
- 📘 A-TO-004 — **Tứ giác latency/throughput/cost/consistency**: cải thiện cái này thường hy sinh cái kia (strong consistency ↑latency; cache ↓latency nhưng stale; replica ↑availability nhưng ↑cost & ↑độ phức tạp đồng bộ). TL chọn theo **ưu tiên nghiệp vụ**, không mơ "tốt mọi mặt".
- 📘 A-TO-005 — **SQL vs NoSQL** *(giao với E)*: quyết theo **access pattern + yêu cầu consistency + mô hình dữ liệu + scale ghi**, KHÔNG theo "NoSQL nhanh hơn". Postgres mặc định an toàn cho phần lớn ca; chọn NoSQL khi có lý do cụ thể (scale ghi cực lớn, schema linh hoạt, key-value đơn giản).
- 📘 A-TO-006 — **Monolith vs microservices**: micro thêm phức tạp network/vận hành/consistency xuyên service. "Microservices mặc định" là **sai** — chia khi CÓ lý do (scale độc lập, team boundary). Modular monolith / monolith hợp team nhỏ & giai đoạn sớm (đào sâu Bài 7).

**(c) Khung trả lời trade-off thuyết phục (📘 A-TO-003).** Bốn bước:
1. **Nêu tiêu chí** đang cân (latency / cost / consistency / complexity / operability).
2. **So 2 phương án theo từng tiêu chí** (không chỉ nói chung).
3. **Gắn với requirement đã chốt** ("vì ta cần strong consistency cho tồn kho nên…").
4. **Thừa nhận nhược điểm** của phương án chọn ("đổi lại latency cao hơn ~Xms, chấp nhận được vì…").
KHÔNG bao giờ nói "A tốt hơn B" trống rỗng.

**Mép giới hạn — over-engineering (📘 A-TO-007).** Thêm shard/microservice/queue/đa vùng khi *chưa cần* = điểm trừ ở phỏng vấn TL: chi phí phức tạp > lợi ích. Nguyên tắc **YAGNI**: bắt đầu đơn giản, scale *khi requirement đòi*. Thể hiện **phán đoán**, không khoe kiến thức.

> 📘 vs ➕: CAP, strong/eventual, SQL/NoSQL, monolith/micro, over-engineering đều 📘. **PACELC** và khung 4 bước articulation là ➕ làm sâu.

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Quan niệm sai phổ biến | Đúng hơn | Vì sao |
|---|---|---|
| "CAP: chọn 2 trong 3" | Khi có P thì chọn C/A; dùng **PACELC** (kể cả lúc không partition) | P không phải lựa chọn — nó luôn có thể xảy ra |
| "NoSQL nhanh hơn SQL" | Chọn theo access pattern/consistency/scale ghi | "Nhanh hơn" tuỳ workload; Postgres an toàn mặc định |
| "Microservices = hiện đại = nên dùng" | Chia khi CÓ lý do; mặc định modular monolith | Micro thêm chi phí phân tán lớn |
| "A tốt hơn B" (không nêu tiêu chí) | Nêu tiêu chí → so → gắn requirement → nhận nhược điểm | Trade-off trống rỗng = red flag |

### ④ Áp dụng thực tế + So sánh bigtech
- **PACELC phân loại hệ thật**: DynamoDB/Cassandra nghiêng **PA/EL** (ưu availability/latency, eventual); hệ kiểu Spanner nghiêng **PC/EC** (ưu consistency, chấp nhận latency cao hơn). Đây là cách đọc một datastore qua lăng kính trade-off (pattern phổ biến — *verify chi tiết sản phẩm*).
- **Monolith-first** là khuyến nghị phổ biến (Martin Fowler "MonolithFirst"): nhiều công ty lớn tách microservices *sau khi* seam đã rõ, không phải từ ngày đầu.

### ⑤ Code thực hành + cấu hình
Bài này là *tư duy*, không code. Dùng **bảng quyết định** điền khi gặp lựa chọn (mang vào phỏng vấn):

```text
# TRADE-OFF DECISION TABLE (điền cho mỗi quyết định lớn)
Quyết định: <SQL vs NoSQL? strong vs eventual? mono vs micro?>
Requirement liên quan đã chốt: <...>
| Tiêu chí       | Phương án A | Phương án B |
|----------------|-------------|-------------|
| Latency        |             |             |
| Consistency    |             |             |
| Cost           |             |             |
| Complexity/Ops |             |             |
| Scale (đọc/ghi)|             |             |
=> CHỌN: <A/B> vì requirement <...>; CHẤP NHẬN mất: <nhược điểm>.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** CAP đúng nghĩa (P luôn có thể xảy ra); PACELC; vì sao mọi lựa chọn là đánh đổi; YAGNI/over-engineering.
- 📌 **Cần THUỘC:** ví dụ strong (tiền/tồn kho/vé) vs eventual (feed/like/view); 4 trục (latency/throughput/cost/consistency); khung 4 bước articulation.
- 🛠️ **Cần LÀM ĐƯỢC:** trả lời "vì sao A không B" theo khung; phân loại một datastore theo PACELC; nhận ra over-engineering.

### ⑦ Mental model
> **"Không có tốt nhất, chỉ có phù hợp nhất."** Nêu tiêu chí → so sánh → gắn requirement → nhận nhược điểm. PACELC > CAP "2 trong 3".

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* CAP nói gì, vì sao "chọn 2 trong 3" gây hiểu lầm? → khi có partition mới phải chọn C/A; P không phải lựa chọn; dùng PACELC.
2. *(TB)* Strong vs eventual — ví dụ mỗi loại? → strong: tồn kho/thanh toán; eventual: feed/like/view.
3. *(khó)* SQL vs NoSQL quyết theo gì? → access pattern + consistency + mô hình data + scale ghi; không phải "nhanh hơn".
4. *(khó)* Vì sao "microservices mặc định" là sai? → thêm chi phí phân tán; chia khi có lý do (scale/team boundary).
5. *(rất khó)* Khung trả lời "vì sao A không B"? → tiêu chí → so sánh → gắn requirement → thừa nhận nhược điểm.

**Thách đố (trick):** *Ứng viên nói "em chọn microservices + Kafka + đa region cho MVP 100 user."* Phản biện? → **Bẫy:** khoe kiến trúc lớn. Đây là **over-engineering**: 100 user không cần phân tán; chi phí vận hành/consistency > lợi ích. TL chọn modular monolith + 1 DB, scale khi requirement đòi.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Cho yêu cầu "đếm view video tỷ lượt". Dùng decision table chọn strong vs eventual consistency. *Đạt khi:* chọn eventual, gắn lý do (trễ vài giây vô hại, ưu latency/availability), nêu nhược điểm (đếm xấp xỉ).
**BT2.** Phân loại Postgres, DynamoDB, Cassandra theo PACELC (P_/E_). *Đạt khi:* giải thích mỗi cái nghiêng C hay A/L và vì sao.

### ⑩ Đọc thêm
- Brewer — *CAP Twelve Years Later*; Abadi — *Consistency Tradeoffs in Modern Distributed Database System Design (PACELC)*.
- Martin Fowler — *MonolithFirst*, *MicroservicePremium*.

---

## Bài 7 — Decomposition & service boundaries
*Phủ: A-DEC-001 → A-DEC-004*

### ① Mục tiêu & vị trí trong mạch
Tiếp khối **Đánh đổi & ranh giới**. Bài 6 nói *khi nào* chia (monolith vs micro); bài này nói *chia theo đâu* và *dấu hiệu chia sai*. Quan trọng cho các case study lớn (checkout Bài 15, feed Bài 11) khi phải đặt ranh giới service. Giao với K (messaging/event) khi mỗi service có DB riêng.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Chia service như chia phòng ban trong công ty: chia theo **chức năng nghiệp vụ** (kế toán, kho, bán hàng) thì mỗi phòng tự chủ; chia theo **tầng kỹ thuật** (phòng "viết SQL", phòng "viết API") thì làm gì cũng phải họp cả 3 phòng. Ranh giới đúng = ít phải "họp" (gọi chéo) khi thay đổi.

**(b) Cơ chế.**
- 📘 A-DEC-001 — **Chia theo business capability / volatility**, KHÔNG theo layer kỹ thuật (controller/service/db). Tức chia theo **bounded context** (DDD) — *cái gì thay đổi cùng nhau thì ở cùng một service*. Giảm coupling khi thay đổi; mỗi service tự chủ một nghiệp vụ end-to-end.
- 📘 A-DEC-002 — **Dấu hiệu boundary vẽ SAI**: 2 service luôn deploy/đổi cùng nhau; gọi chéo liên tục (**chatty**); transaction xuyên service nhiều; chia sẻ DB. ⇒ nên **gộp lại** hoặc vẽ lại ranh giới.
- 📘 A-DEC-003 — **Database-per-service** *(giao với K)*: lợi = tự chủ + deploy độc lập (mỗi service đổi schema không ảnh hưởng service khác). Kéo theo = **mất join** xuyên service + cần đồng bộ dữ liệu (**event / CDC**) + consistency xuyên service phức tạp (phải Saga thay vì transaction — xem Bài 15).
- 📘 A-DEC-004 — **Modular monolith**: ranh giới module rõ *trong một deploy*. Lấy được lợi ích boundary (code tách bạch, ít coupling) mà **chưa gánh chi phí phân tán** (network, distributed transaction, ops). Tách ra microservice **sau** khi seam đã rõ và có lý do thật.

> 📘 vs ➕: toàn bộ là kiến thức ổn định (📘). Liên hệ DDD/bounded context và "Conway's law" (cơ cấu hệ phản chiếu cơ cấu tổ chức) là ➕ làm sâu.

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Chia service theo tầng (API service, DB service…) | Chia theo bounded context/nghiệp vụ | Theo tầng ⇒ đổi gì cũng phải sửa nhiều service |
| Nhiều service share chung một DB | DB-per-service + đồng bộ qua event/CDC | Share DB ⇒ coupling ngầm, mất tự chủ |
| Distributed transaction/2PC xuyên service | Saga + compensating action | 2PC mong manh & chậm trong hệ phân tán |
| Tách micro ngay từ đầu | Modular monolith trước, tách khi seam rõ | Tránh chi phí phân tán khi chưa cần |

### ④ Áp dụng thực tế + So sánh bigtech
- **Conway's Law**: tổ chức nhiều team độc lập ⇒ hệ tự nhiên tách theo team — nên ranh giới service thường khớp ranh giới team (pattern phổ biến).
- **Strangler Fig pattern**: cách tách microservice từ monolith dần dần — bọc monolith bằng facade, chuyển từng capability ra service mới rồi "siết" dần. Đây là cách an toàn hơn "viết lại từ đầu".

### ⑤ Code thực hành + cấu hình
Không phải bài code; dùng **bài tập vẽ ranh giới**. Ví dụ minh hoạ ranh giới đúng (modular monolith với module rõ):

```text
# Modular monolith — cấu trúc thư mục thể hiện bounded context
src/
  catalog/     # module Catalog: domain + service + repo, KHÔNG truy cập bảng của module khác
    domain/  api/  repo/
  ordering/    # module Ordering: tự chủ, gọi catalog QUA interface công khai, không qua DB
    domain/  api/  repo/
  inventory/   # module Inventory
    domain/  api/  repo/
  shared-kernel/   # type/contract dùng chung TỐI THIỂU
# Quy tắc: module chỉ gọi nhau qua public API (interface), không đụng bảng/đối tượng nội bộ nhau.
# Khi cần tách micro: mỗi module đã là một seam sẵn sàng -> bê ra thành service.
```

```text
# Heuristic kiểm tra boundary (tự hỏi):
- Hai service này có BAO GIỜ deploy riêng không? Nếu luôn cùng nhau -> gộp.
- Một thay đổi nghiệp vụ điển hình chạm mấy service? >2 thường -> boundary sai.
- Có transaction nào BẮT BUỘC xuyên 2 service không? Nhiều -> cân nhắc gộp/đổi ranh giới.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** chia theo business capability/volatility vs theo layer; vì sao share DB là anti-pattern; vì sao Saga thay 2PC; modular monolith.
- 📌 **Cần THUỘC:** dấu hiệu boundary sai (chatty / deploy cùng nhau / txn xuyên service / share DB); DB-per-service đánh đổi; Strangler Fig; Conway's Law.
- 🛠️ **Cần LÀM ĐƯỢC:** vẽ ranh giới cho một domain; chỉ ra boundary sai; thiết kế modular monolith có seam.

### ⑦ Mental model
> **"Chia theo cái thay đổi cùng nhau, không theo tầng kỹ thuật."** Bắt đầu modular monolith; tách micro khi seam rõ + có lý do.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Chia service theo gì, vì sao không theo layer? → theo bounded context/nghiệp vụ; theo layer ⇒ coupling cao khi đổi.
2. *(TB)* Dấu hiệu một boundary bị chia sai? → chatty, deploy cùng nhau, txn xuyên service, share DB.
3. *(khó)* DB-per-service lợi/hại gì? → tự chủ + deploy độc lập; nhưng mất join + cần event/CDC + consistency phức tạp.
4. *(TB)* Modular monolith là gì, vì sao là điểm khởi đầu tốt? → boundary rõ trong 1 deploy; lợi ích tách mà chưa gánh chi phí phân tán.
5. *(khó)* Tách micro từ monolith an toàn thế nào? → Strangler Fig: bọc facade, chuyển từng capability, siết dần.

**Thách đố (trick):** *Hai microservice "Order" và "OrderItem" luôn được sửa và deploy cùng nhau.* Có gì sai? → **Bẫy:** chia quá nhỏ theo *entity* thay vì *capability*. Đây là boundary sai — chúng thuộc cùng một bounded context "Ordering", nên **gộp** lại; tách ra chỉ tạo chatty calls + distributed transaction vô ích.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Cho domain e-commerce (catalog, cart, order, payment, inventory, shipping, notification). Nhóm thành 4–6 service và giải thích ranh giới. *Đạt khi:* chia theo capability, chỉ ra cặp nào nên gộp/tách và vì sao, nêu chỗ cần event giữa service.
**BT2.** Chỉ ra 3 dấu hiệu cho biết bạn đã chia service sai trong một thiết kế giả định. *Đạt khi:* dùng đúng các tín hiệu (chatty / deploy cùng / txn xuyên / share DB).

### ⑩ Đọc thêm
- Sam Newman — *Building Microservices* (chương boundaries) & *Monolith to Microservices* (Strangler Fig).
- Eric Evans — *Domain-Driven Design* (Bounded Context).

---

## Bài 8 — Serverless & edge computing
*Phủ: A-SVL-001 → A-SVL-005*

### ① Mục tiêu & vị trí trong mạch
Khép khối **Đánh đổi & ranh giới**. Một *lựa chọn compute* khác với cụm container/k8s ở Bài 3. Mục tiêu: hiểu **khi nào hợp / không hợp** và các bẫy đặc thù (cold start, connection, stateless) — để **chỉ huy** quyết định "serverless hay container", không chạy theo hype.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Serverless (FaaS) như thuê taxi theo cuốc: không có cuốc thì không trả tiền, lên cuốc đầu hơi chờ xe tới (cold start), không hợp để chở hàng đường dài liên tục (tải đều cao thì tự nuôi xe rẻ hơn). Edge computing như đặt quầy phục vụ ngay cạnh khách (gần user) cho việc nhẹ.

**(b) Cơ chế.**
- 📘 A-SVL-001 — **Khi nào FaaS hợp**: tải **gai/không đều**, **event-driven**, muốn ít vận hành (scale-to-zero). **Không hợp**: tải đều cao (cost cao hơn container), latency cực nhạy, long-running/stateful. Trade-off: **cold start + vendor lock-in**.
- 📘 A-SVL-002 — **Cold start**: lần gọi đầu/sau idle phải khởi tạo runtime ⇒ tăng latency. Giảm bằng **provisioned concurrency / keep-warm / runtime nhẹ**. ➕ (verify) — nay còn **SnapStart** (snapshot môi trường đã init, giảm cold start ~90%, miễn phí ở runtime hỗ trợ) là lựa chọn rẻ hơn provisioned concurrency. Vấn đề chủ yếu với hệ **p99 latency-sensitive**; tải async (S3 event, cron, batch) thường chịu được cold start.
- 📘 A-SVL-003 — **Stateless & ephemeral** ⇒ hệ quả tới **DB connection**: mỗi invocation có thể mở 1 connection ⇒ **bùng nổ connection** đập sập DB. Giải: dùng **connection proxy** (verify: **Amazon RDS Proxy** quản pool dùng chung cho Lambda) hoặc external store; KHÔNG để mỗi function tự mở pool lớn (double pooling). Thiết kế khác hẳn service thường trú.
- 📘 A-SVL-004 — **Edge computing / CDN compute**: chạy logic **gần user** (auth nhẹ, A/B test, URL rewrite, personalization cache) ⇒ giảm round-trip tới origin. KHÔNG đặt logic nặng/stateful/data-heavy ở edge (edge giới hạn CPU/thời gian, xa DB nguồn).

**(c) Khung quyết định (📘 A-SVL-005).** "Serverless hay container/k8s?" — quyết theo: **pattern tải** (gai vs đều), **latency budget**, **độ phức tạp vận hành**, **cost ở tải dự kiến**, **statefulness**, **vendor lock-in**. Quyết theo **requirement**, không theo hype.

> 📘 vs ➕: nguyên lý FaaS/cold start/edge là 📘. **SnapStart** và chi tiết RDS Proxy là ➕ (đã verify tại docs AWS — *kiểm chứng đời/region khi triển khai*).

### ③ ⚠️ Kiến thức cũ / hay bị làm sai

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Mỗi Lambda tự mở connection pool tới DB | Dùng RDS Proxy / external pool; NullPool ở app | Bùng nổ connection / double pooling đập sập DB |
| "Serverless luôn rẻ hơn" | Tính cost ở **tải dự kiến**; tải đều cao ⇒ container rẻ hơn | FaaS rẻ ở tải gai, đắt ở tải đều cao liên tục |
| Provisioned concurrency cho mọi cold start | Cân nhắc **SnapStart** (miễn phí) / chịu cold start nếu async | Provisioned concurrency tốn tiền cố định 24/7 |
| Đặt business logic nặng ở edge | Edge chỉ logic nhẹ; logic nặng ở origin | Edge giới hạn CPU/time, xa DB |

### ④ Áp dụng thực tế + So sánh bigtech
- **FaaS hợp**: webhook handler, ảnh thumbnail on-upload, cron/ETL, glue giữa service — tải gai, event-driven (pattern phổ biến: AWS Lambda, GCP Cloud Functions, Azure Functions, Cloudflare Workers).
- **Edge**: auth token check, A/B routing, geo-personalization, cache ở biên (Cloudflare Workers / CDN compute). Bigtech đặt **logic nhẹ + cache** ở edge, **data + logic nặng** ở region — *verify chi tiết sản phẩm/đời*.

### ⑤ Code thực hành + cấu hình
**Lambda + DB qua RDS Proxy đúng cách** (Node.js): khai báo client ở global scope nhưng validate trước khi dùng; để proxy lo pool.

```javascript
// handler.js — Lambda; ĐÃ verify RDS Proxy là cách quản connection cho serverless (docs AWS)
// Quy tắc: KHÔNG mở pool lớn trong từng invocation; trỏ tới RDS Proxy endpoint.
const { Client } = require('pg');                 // pg client đơn (proxy lo pooling)
let client;                                       // global scope -> tái dùng giữa các invocation warm

async function getClient() {
  if (client && client._connected) return client; // validate trước khi dùng (tránh stale conn)
  client = new Client({
    host: process.env.RDS_PROXY_ENDPOINT,         // <- RDS Proxy, KHÔNG nối thẳng DB
    user: process.env.DB_USER,
    database: process.env.DB_NAME,
    // dùng IAM auth token thay vì password tĩnh nếu có thể (an toàn hơn)
    ssl: { ca: process.env.RDS_CA }, password: process.env.DB_PASSWORD,
  });
  await client.connect();
  client._connected = true;
  return client;
}

exports.handler = async (event) => {
  const db = await getClient();
  const { rows } = await db.query('SELECT id, name FROM users WHERE id = $1', [event.userId]);
  return { statusCode: 200, body: JSON.stringify(rows[0] ?? null) };
};
```

```text
# Cấu hình serverless (checklist):
- DB: dùng RDS Proxy (MaxConnectionsPercent ~80-90%); ở app dùng NullPool/không pool nội bộ.
- Cold start: ưu tiên SnapStart (nếu runtime hỗ trợ) trước khi mua provisioned concurrency.
- Secrets: dùng secret manager + IAM auth, KHÔNG hardcode password.
- Edge: chỉ deploy logic nhẹ (auth check/rewrite/AB); giữ data-heavy ở origin.
- // kiểm chứng tên dịch vụ/đời/region tại docs AWS — có thể đổi.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** khi nào FaaS hợp/không; vì sao serverless gây bùng nổ connection; vì sao edge chỉ cho logic nhẹ; cost crossover (gai vs đều).
- 📌 **Cần THUỘC:** cold start + cách giảm (provisioned concurrency / SnapStart / keep-warm); RDS Proxy cho connection; vendor lock-in; tiêu chí "serverless vs container".
- 🛠️ **Cần LÀM ĐƯỢC:** quyết serverless hay container theo requirement; cấu hình Lambda + RDS Proxy đúng; chọn logic đặt ở edge.

### ⑦ Mental model
> **"FaaS = trả theo cuốc, hợp tải gai."** Stateless ⇒ connection là bài toán (proxy); edge chỉ cho logic nhẹ; quyết theo requirement không theo hype.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Khi nào serverless hợp, khi nào không? → hợp tải gai/event-driven/ít ops; không hợp tải đều cao/latency nhạy/long-running/stateful.
2. *(TB)* Cold start là gì, giảm sao? → init runtime lần đầu/sau idle ⇒ ↑latency; provisioned concurrency / SnapStart / keep-warm / runtime nhẹ.
3. *(khó)* Serverless ảnh hưởng DB connection thế nào? → mỗi invocation 1 connection ⇒ bùng nổ; dùng RDS Proxy / external pool.
4. *(TB)* Edge computing đặt logic gì, KHÔNG đặt gì? → auth nhẹ/AB/rewrite/personalization cache; không logic nặng/stateful/data-heavy.
5. *(khó)* "Serverless hay k8s?" quyết theo gì? → pattern tải, latency, ops, cost ở tải dự kiến, statefulness, lock-in.

**Thách đố (trick):** *Lambda của bạn chạy tốt khi test nhưng sập DB khi traffic spike.* Vì sao? → **Bẫy:** quên ephemeral connection. Spike ⇒ hàng nghìn invocation đồng thời mở connection ⇒ vượt `max_connections` của DB ⇒ DB từ chối/treo. Fix: **RDS Proxy** + giới hạn reserved concurrency + NullPool.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Cho 3 workload (webhook ingest gai; API user-facing tải đều 5k QPS; cron ETL đêm). Quyết serverless hay container cho từng cái + lý do. *Đạt khi:* webhook→FaaS, API tải đều→container (cost+latency), ETL→FaaS (async chịu cold start).
**BT2.** Liệt kê 3 bẫy production của serverless và cách phòng. *Đạt khi:* nêu cold start (SnapStart), connection blow-up (RDS Proxy), vendor lock-in (abstraction/đa cloud cân nhắc).

### ⑩ Đọc thêm
- AWS docs — *Using AWS Lambda with Amazon RDS* & *RDS Proxy connections*; *Lambda SnapStart*.
- AWS Well-Architected — *Serverless Application Lens*. Cloudflare — *Workers* (edge compute). *(verify đời/region)*

---

## Bài 9 — Case study: URL shortener / pastebin
*Phủ: A-URL-001 → A-URL-005*

### ① Mục tiêu & vị trí trong mạch
Case study **nhập môn** của khối IV — ráp Bài 1 (quy trình), 2 (read-heavy), 4 (cache/object storage). Đề đơn giản nên là nơi tốt nhất để luyện *drive the interview* end-to-end mà không bị ngợp. Học xong bạn xử được mọi biến thể "tạo mã ngắn → tra ngược nhanh".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** URL shortener = một **từ điển khổng lồ** ánh xạ `short_key → long_url`. Bài toán thật không phải "tạo key" mà là *tra ngược cực nhanh, cực nhiều lần* (read ≫ write).

**(b) Cơ chế.**
- **Clarify**: functional = tạo short URL, redirect, (optional: custom alias, expiration, analytics). Non-functional = read ≫ write (📘 A-URL-002), latency redirect thấp, availability cao, không trùng key.
- 📘 A-URL-002 — **Read-heavy** ⇒ kiến trúc tối ưu *đường đọc*: **cache redirect nóng** (Redis) + **read replica** + (tùy) CDN. Tạo 1 lần, redirect hàng triệu lần.
- 📘 A-URL-001 — **Sinh short key**: hai hướng đánh đổi.
  - *base62 của ID tăng dần*: gọn, tuần tự — nhưng **đoán được** (lộ thứ tự, enumerable).
  - *hash + xử lý collision*: ngẫu nhiên (khó đoán) nhưng phải **check trùng**.
  - Độ dài key ↔ không gian địa chỉ: base62 với 7 ký tự ≈ 62⁷ ≈ 3.5 nghìn tỷ key.
- 📘 A-URL-003 — **Unique ID ở quy mô phân tán** (tránh auto-increment 1 DB thành bottleneck):
  - *counter trung tâm / range allocation*: mỗi node xin một dải ID (vd 1000 số) rồi cấp local.
  - *Snowflake-like*: `timestamp + machine_id + sequence` ⇒ ID 64-bit, tuần tự theo thời gian, không cần phối hợp toàn cục.
  - *UUID / KGS (key generation service)*: pre-generate key sẵn.
  - Đánh đổi: tính **tuần tự** (tốt cho index) vs **phân tán/khó đoán**.
- 📘 A-URL-004 — **301 vs 302 redirect**: 301 permanent ⇒ browser cache redirect ⇒ **mất lần đếm** các lần sau (nhanh hơn cho user, tốt cho SEO). 302 temporary ⇒ mỗi lần qua server ⇒ **đếm được** analytics. Chọn theo nhu cầu tracking.

**(c) Mép giới hạn (📘 A-URL-005).** Thêm tính năng làm phức tạp ở đâu: **custom alias** ⇒ check unique (đụng key sinh tự động); **expiration** ⇒ TTL + cleanup job (xóa key hết hạn); **analytics** ⇒ đẩy event **async qua queue** để KHÔNG làm chậm redirect (redirect phải nhanh, đếm là việc phụ).

### ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| `AUTO_INCREMENT` 1 DB sinh ID | Range allocation / Snowflake / KGS | 1 DB là bottleneck + SPOF ở scale lớn |
| Đếm analytics đồng bộ trong redirect | Đẩy event async qua queue | Redirect phải nhanh; đếm chậm làm hỏng UX |
| Dùng 301 rồi than "không đếm được click" | 302 nếu cần tracking | 301 bị browser cache ⇒ mất count |
| Hash rồi không check collision | Check trùng (hoặc dùng counter→base62) | Collision ⇒ ghi đè URL người khác |

### ④ Áp dụng thực tế + So sánh bigtech
Pattern này dùng ở mọi link shortener (bit.ly, t.co…) và mọi hệ cần **short id** (mã giảm giá, mã mời, share link). ➕ Mở rộng thực tế: short URL thường phục vụ redirect ở **edge/CDN** để latency cực thấp toàn cầu; analytics pipeline (Kafka → warehouse) tách hẳn khỏi đường redirect (pattern phổ biến).

### ⑤ Code thực hành + cấu hình
**Sinh key bằng counter → base62 + cache redirect** (Python, minh hoạ; redis cho cache):

```python
# url_shortener.py — pip install redis ; cấu hình REDIS_URL, DB qua env
import string
ALPHABET = string.digits + string.ascii_lowercase + string.ascii_uppercase  # base62

def to_base62(n: int) -> str:
    if n == 0: return ALPHABET[0]
    s = []
    while n:
        n, r = divmod(n, 62)
        s.append(ALPHABET[r])
    return "".join(reversed(s))            # ID tăng dần -> key gọn, tuần tự

# --- Giả lập tầng lưu trữ ---
class Store:
    def __init__(self, redis, db):
        self.redis = redis                 # cache redirect nóng (đường đọc)
        self.db = db                       # nguồn sự thật (short_key -> long_url, meta)

    def create(self, long_url, alias=None, ttl=None):
        if alias:                          # custom alias: PHẢI check unique (A-URL-005)
            if self.db.exists(alias): raise ValueError("alias đã dùng")
            key = alias
        else:
            new_id = self.db.next_id()     # range allocation/Snowflake để tránh 1-DB bottleneck
            key = to_base62(new_id)
        self.db.put(key, long_url, ttl)    # ttl -> expiration + cleanup job dọn sau
        return key

    def resolve(self, key):                # ĐƯỜNG ĐỌC: cache trước, DB sau
        url = self.redis.get(f"u:{key}")
        if url: return url
        url = self.db.get(key)
        if url: self.redis.setex(f"u:{key}", 3600, url)   # cache nóng
        return url
# Redirect handler: trả 302 nếu cần đếm click; đẩy event click vào QUEUE (async), KHÔNG đếm inline.
```

```text
# Cấu hình:
- ID: dùng range allocation (mỗi instance xin dải 10k id) hoặc Snowflake; KHÔNG auto-increment đơn DB.
- Redirect: 302 nếu cần analytics; 301 nếu ưu tiên tốc độ/SEO và không cần đếm.
- Analytics: click -> Kafka/queue -> aggregate offline; redirect không chờ analytics.
- Cleanup: cron quét key hết hạn theo TTL (hoặc TTL native của Redis cho cache).
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao read-heavy định hình kiến trúc; base62-counter vs hash đánh đổi; 301 vs 302 ảnh hưởng count; vì sao analytics phải async.
- 📌 **Cần THUỘC:** base62 (62⁷≈3.5T với 7 ký tự); cách sinh distributed unique ID (range/Snowflake/KGS); 301=cache, 302=đếm được.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế end-to-end URL shortener; chọn cơ chế ID; tách analytics khỏi đường redirect.

### ⑦ Mental model
> **"Tạo 1 lần, đọc triệu lần."** Tối ưu đường đọc (cache+replica); ID phân tán không-collision; đếm click async; 302 nếu cần tracking.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Hệ này read- hay write-heavy, suy ra gì? → read≫write ⇒ cache redirect + replica + (CDN).
2. *(TB)* Sinh short key: base62-counter vs hash — đánh đổi? → counter gọn/tuần tự/đoán được; hash ngẫu nhiên/cần check collision.
3. *(khó)* Sinh unique ID phân tán không đụng nhau? → range allocation / Snowflake (ts+machine+seq) / KGS; tránh 1-DB auto-increment.
4. *(TB)* 301 vs 302 ảnh hưởng analytics/cache thế nào? → 301 browser-cache mất count; 302 mỗi lần qua server, đếm được.
5. *(khó)* Thêm custom alias + expiration + analytics phức tạp ở đâu? → alias check unique; expiration TTL+cleanup; analytics async qua queue.

**Thách đố (trick):** *"Em dùng `UPDATE clicks = clicks+1` trong handler redirect cho chính xác."* Sao không nên? → **Bẫy:** ghi DB đồng bộ trong đường nóng. Mỗi redirect +1 lần ghi vào 1 row ⇒ **hot row + chậm + bottleneck** ở scale lớn. Đẩy event async, aggregate sau (chấp nhận đếm trễ/xấp xỉ).

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Thiết kế full URL shortener cho 100M URL, 10:1 read:write. Vẽ luồng tạo + redirect, chọn ID scheme, đặt cache ở đâu. *Đạt khi:* tối ưu đường đọc rõ ràng, ID không-collision phân tán, analytics async.
**BT2.** Ước lượng storage cho 100M URL (mỗi record ~500B) và QPS redirect nếu mỗi URL được click 50 lần/đời trong 1 năm. *Đạt khi:* dùng Bài 2 ra bậc độ lớn hợp lý.

### ⑩ Đọc thêm
- *System Design Interview* (Alex Xu) — "Design A URL Shortener". Twitter Snowflake (ID generation). `system-design-primer` — pastebin/URL shortener.

---

## Bài 10 — Case study: Rate limiter
*Phủ: A-RL-001 → A-RL-006*

### ① Mục tiêu & vị trí trong mạch
Case study về **distributed counter + atomicity** — ráp Bài 4 (consistent hashing), 5 (hot key), 6 (fail-open/closed là trade-off), giao mạnh với I (concurrency) và M (security/abuse). Đây là bài bộc lộ rõ ai *thật sự hiểu hệ phân tán* (race condition, state trung tâm).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Rate limiter = người gác cửa đếm "anh đã vào mấy lần trong phút này". Vấn đề khó: khi có **50 cửa (server)**, mỗi cửa đếm riêng thì tổng vượt giới hạn — phải có **một cuốn sổ chung** (Redis) và **ghi sổ không bị tranh** (atomic).

**(b) Cơ chế — thuật toán (📘 A-RL-001):**
- **Token bucket**: bình chứa token, đổ đều theo thời gian; mỗi request lấy 1 token. Cho phép **burst** (đến dung tích bình) + smoothing average rate. Phổ biến nhất.
- **Fixed window**: đếm trong cửa sổ cố định (vd mỗi phút reset). Đơn giản nhưng **boundary burst** (📘 A-RL-002): dồn request cuối phút này + đầu phút kế ⇒ tới ~2× limit trong khoảng ngắn.
- **Sliding window log**: lưu timestamp mọi request ⇒ chính xác tuyệt đối nhưng **tốn memory**.
- **Sliding window counter**: dung hòa — nội suy giữa 2 cửa sổ, gần chính xác, rẻ memory.

**(c) Phân tán & race (cốt lõi).**
- 📘 A-RL-003 — **State cục bộ mỗi server là SAI**: mỗi server chỉ thấy phần traffic của nó ⇒ tổng vượt limit. Cần **state trung tâm** (Redis) HOẶC local counter + **sync định kỳ**. Nhận ra đây là **bài toán consistency**.
- 📘 A-RL-004 — **Race condition** *(giao với I)*: 2 server cùng read-modify-write 1 counter ⇒ lost update. Fix: **atomic op** — Redis `INCR`, **Lua script** (đọc-kiểm-ghi nguyên tử), hoặc transaction. KHÔNG read-modify-write rời rạc. (Thừa nhận race = điểm cộng; lờ đi = red flag.)
- 📘 A-RL-005 — **Redis chết** ⇒ **fail-open hay fail-closed?** Đây là trade-off có chủ đích (Bài 6): *fail-open* (cho qua, ưu availability — đa số API public) vs *fail-closed* (chặn, ưu bảo vệ — endpoint tài chính/security). Trả lời "tùy" + nêu tiêu chí, KHÔNG cứng nhắc.

**(d) Mép giới hạn (📘 A-RL-006).** Ở throughput cực lớn, **chính Redis cũng thành bottleneck / hot key**. Giảm tải: **local in-memory counter sync Redis mỗi ~100ms** (chấp nhận sai số nhỏ), **shard key qua slot**, **per-user dedicated key** cho super-user, **consistent hashing** để phân tán. (Pattern phổ biến — *verify chi tiết*.)

### ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Counter cục bộ mỗi app server | State trung tâm (Redis) hoặc local+sync | Mỗi server thấy 1 phần ⇒ tổng vượt limit |
| `GET` rồi `SET count+1` rời rạc | `INCR` / Lua script (atomic) | Read-modify-write bị race ⇒ lost update |
| Fixed window cho giới hạn chặt | Sliding window (counter) | Fixed window cho boundary burst ~2× |
| "Redis chết thì hệ tự xử" (không định nghĩa) | Quyết fail-open/closed có chủ đích | Hành vi khi lỗi phải là quyết định, không phải tai nạn |

### ④ Áp dụng thực tế + So sánh bigtech
Rate limiting là lớp phòng vệ chuẩn ở **API gateway** (chống abuse, bảo vệ backend, fair-use, chống DDoS lớp ứng dụng). ➕ Pattern phổ biến: limit nhiều tầng (per-IP, per-user, per-endpoint, global); **adaptive rate limiting** (tự siết khi backend quá tải); thuật toán token bucket được dùng rộng (nginx `limit_req`, cloud API gateway, Stripe…). *(verify chi tiết hãng)*

### ⑤ Code thực hành + cấu hình
**Token bucket atomic bằng Redis Lua** (an toàn race, A-RL-004):

```python
# rate_limiter.py — pip install redis ; REDIS_URL qua env
import redis, time

# Lua chạy nguyên tử trên Redis: đọc token + thời gian, refill, trừ, ghi lại — KHÔNG bị race.
TOKEN_BUCKET = """
local key      = KEYS[1]
local rate     = tonumber(ARGV[1])   -- token/giây đổ vào
local capacity = tonumber(ARGV[2])   -- dung tích bình (cho phép burst)
local now      = tonumber(ARGV[3])
local tokens   = tonumber(redis.call('hget', key, 'tokens') or capacity)
local last     = tonumber(redis.call('hget', key, 'ts') or now)
local refill   = math.min(capacity, tokens + (now - last) * rate)  -- đổ token theo thời gian trôi
local allowed  = refill >= 1
if allowed then refill = refill - 1 end
redis.call('hset', key, 'tokens', refill, 'ts', now)
redis.call('expire', key, math.ceil(capacity / rate) * 2)
return allowed and 1 or 0
"""

class RateLimiter:
    def __init__(self, url, rate=10, capacity=20):  # 10 req/s, burst tới 20
        self.r = redis.from_url(url)
        self.sha = self.r.script_load(TOKEN_BUCKET)
        self.rate, self.capacity = rate, capacity

    def allow(self, user_id) -> bool:
        try:
            return bool(self.r.evalsha(self.sha, 1, f"rl:{user_id}",
                                       self.rate, self.capacity, time.time()))
        except redis.RedisError:
            return True   # FAIL-OPEN (A-RL-005): ưu availability. Đổi thành False nếu endpoint nhạy cảm.
```

```text
# Cấu hình & scale:
- Atomicity: luôn dùng Lua/INCR; KHÔNG read-modify-write ở app.
- Fail mode: chọn fail-open (public API) hay fail-closed (payment/auth) — ghi rõ trong design.
- Hot key/throughput cao (A-RL-006): thêm local counter sync mỗi ~100ms + shard key + dedicated key cho super-user.
- Đặt limiter ở API gateway/edge để chặn sớm, giảm tải backend.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao local counter sai; vì sao cần atomic; boundary burst; fail-open vs fail-closed là trade-off; Redis cũng có thể là hot key.
- 📌 **Cần THUỘC:** 4 thuật toán (token bucket / fixed / sliding log / sliding counter) + đặc tính; `INCR`/Lua atomic.
- 🛠️ **Cần LÀM ĐƯỢC:** cài token bucket atomic; quyết fail mode; giảm tải Redis ở throughput cực lớn.

### ⑦ Mental model
> **"Một cuốn sổ chung, ghi không tranh."** State trung tâm + atomic op; chọn fail-open/closed có chủ đích; Redis nóng thì local+sync.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Token bucket vs fixed vs sliding window — đặc tính? → bucket cho burst+smoothing; fixed đơn giản nhưng boundary burst; sliding log chính xác/tốn RAM; sliding counter dung hòa.
2. *(TB)* Boundary burst là gì? → dồn cuối+đầu 2 cửa sổ ⇒ tới ~2× limit; sliding window khắc phục.
3. *(khó)* Rate limiter trên 50 server: vì sao local sai, fix sao? → mỗi server 1 phần traffic ⇒ tổng vượt; cần Redis trung tâm hoặc local+sync.
4. *(khó)* Race khi 2 server +1 cùng counter — xử lý? → atomic `INCR`/Lua/transaction; không read-modify-write rời.
5. *(khó)* Redis chết thì làm gì? → fail-open (public, ưu availability) vs fail-closed (nhạy cảm); nêu trade-off.

**Thách đố (trick):** *"Em để mỗi server giữ counter trong RAM cho nhanh, đỡ gọi Redis."* Sai ở đâu? → **Bẫy:** tối ưu latency phá đúng đắn. 50 server × limit-mỗi-server ⇒ tổng cho qua gấp ~50× ý định. Nếu muốn local cho nhanh thì phải **sync về Redis** (chấp nhận sai số nhỏ), không để mỗi server đếm độc lập.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Cài `rate_limiter.py`, test 30 request liên tiếp với rate=10/s, capacity=20: quan sát burst rồi bị chặn. *Đạt khi:* giải thích vì sao 20 request đầu qua (burst) rồi nhỏ giọt theo rate.
**BT2.** Thiết kế rate limiter cho API gateway phục vụ 1M QPS toàn cục, có super-user. Nêu cách tránh Redis thành hot key. *Đạt khi:* dùng local+sync, shard key, dedicated key cho super-user, đặt ở edge.

### ⑩ Đọc thêm
- *System Design Interview* (Alex Xu) — "Design a Rate Limiter". Redis docs — `INCR`, Lua scripting. Stripe blog — *Scaling your API with rate limiters*.

---

## Bài 11 — Case study: News feed & fan-out
*Phủ: A-FEED-001 → A-FEED-006*

### ① Mục tiêu & vị trí trong mạch
Case study **fan-out** kinh điển — ráp Bài 2 (read:write), 5 (hot key/celebrity), 6 (eventual consistency), giao với F (pagination/cursor). Đây là bài dạy rõ nhất *vì sao một thiết kế "đúng" phải đổi khi gặp celebrity*.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Feed = "khi A đăng bài, làm sao những người follow A thấy nó nhanh?". Hai cách trái ngược: **đẩy** (push) bài vào hộp thư từng follower ngay lúc đăng, hay **kéo** (pull) gom bài lúc follower mở app. Đẩy = đọc nhanh/ghi nặng; kéo = ghi nhẹ/đọc chậm.

**(b) Cơ chế.**
- 📘 A-FEED-001 — **Fan-out-on-write vs on-read**:
  - *on-write (push)*: lúc post, ghi vào feed của **mọi follower**. Đọc cực nhanh (feed precomputed). Đổi lại: **ghi nặng + tốn storage** (nhân bản).
  - *on-read (pull)*: lúc đọc, gom bài từ những người user follow. Ghi nhẹ. Đổi lại: **đọc chậm** (phải gom + sort runtime).
  - Chọn theo **read:write ratio & phân bố follower**.
- 📘 A-FEED-002 — **Celebrity problem** *(giao Bài 5)*: user triệu follower phá on-write — 1 post phải ghi triệu feed ⇒ **bão ghi + độ trễ**. Giải: **hybrid** — on-write cho user thường + **pull cho celebrity**, **merge lúc đọc**. (Đây là câu phân biệt senior.)
- 📘 A-FEED-003 — **Feed lưu ở đâu**: precomputed feed trong **cache/Redis** theo user; chỉ lưu **ID post** rồi *hydrate* nội dung sau (tiết kiệm RAM + nội dung đổi không phải rewrite feed); **giới hạn độ dài** feed (vài trăm mục gần nhất).
- 📘 A-FEED-004 — **Ranking** (không chỉ theo thời gian): cần feature/score ⇒ không thể chỉ sort timestamp; thường tách **ranking service**, tính **async/precompute**; trade-off realtime vs chất lượng xếp hạng.
- 📘 A-FEED-005 — **Mức consistency**: thấy post trễ vài giây **vô hại** (khác tiền/vé) ⇒ ưu **availability/latency** hơn strong consistency (eventual là đủ).
- 📘 A-FEED-006 — **Pagination cuộn vô tận** *(giao F)*: dùng **cursor** (timestamp/id) chứ không **offset** — offset chậm dần ở list lớn + lệch/nhảy khi có post mới chèn; cursor ổn định + hiệu quả realtime.

### ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| On-write cho TẤT CẢ user (kể cả celebrity) | Hybrid: on-write thường + pull celebrity | Celebrity ⇒ bão ghi triệu feed |
| Lưu full nội dung post trong mỗi feed | Lưu ID post, hydrate khi đọc | Đỡ RAM + nội dung sửa không phải rewrite feed |
| Offset pagination cho feed | Cursor (id/timestamp) | Offset chậm + nhảy khi có post mới |
| Strong consistency cho feed | Eventual consistency | Trễ vài giây vô hại; ưu latency/availability |

### ④ Áp dụng thực tế + So sánh bigtech
Hybrid fan-out là pattern phổ biến ở mạng xã hội lớn (Twitter/X, Instagram…): đa số user dùng push, tài khoản đông follower dùng pull, merge lúc đọc; feed precomputed ở store nhanh (Redis-like); ranking tách thành tầng ML riêng. ➕ "For You" recommendation feed (Bài 12) thêm tầng candidate generation + ranking trên nền fan-out này. *(pattern phổ biến — verify nội bộ hãng)*

### ⑤ Code thực hành + cấu hình
**Hybrid fan-out** (Python, minh hoạ logic push thường + pull celebrity + merge):

```python
# feed.py — minh hoạ; redis cho feed store, queue cho fan-out async
CELEB_THRESHOLD = 100_000   # ngưỡng follower coi là "celebrity" (tinh chỉnh theo data)

def on_post(post_id, author_id, followers_count, redis, queue, db):
    if followers_count < CELEB_THRESHOLD:
        # FAN-OUT ON WRITE: đẩy post_id vào feed từng follower (async qua queue, KHÔNG inline)
        queue.publish("fanout", {"post_id": post_id, "author_id": author_id})
    else:
        # CELEBRITY: KHÔNG fan-out; chỉ lưu vào timeline tác giả, follower sẽ PULL lúc đọc
        db.append_author_timeline(author_id, post_id)

def fanout_worker(msg, redis, db):           # worker xử lý queue
    for follower in db.followers(msg["author_id"]):
        redis.lpush(f"feed:{follower}", msg["post_id"])    # lưu ID, không lưu nội dung
        redis.ltrim(f"feed:{follower}", 0, 799)            # giới hạn ~800 mục gần nhất

def get_feed(user_id, cursor, redis, db):    # ĐỌC: merge push-feed + pull từ celebrity đang follow
    pushed = redis.lrange(f"feed:{user_id}", 0, 50)        # phần precomputed (user thường)
    celeb_posts = []
    for celeb in db.celebrities_followed_by(user_id):      # pull celebrity timeline
        celeb_posts += db.author_timeline(celeb, after=cursor, limit=50)
    merged = sorted(set(pushed) | set(celeb_posts), reverse=True)[:50]  # merge + sort
    posts = db.hydrate(merged)               # hydrate nội dung từ ID (A-FEED-003)
    next_cursor = posts[-1].id if posts else None          # CURSOR pagination (A-FEED-006)
    return posts, next_cursor
```

```text
# Cấu hình:
- Fan-out: luôn async qua queue; KHÔNG fan-out đồng bộ trong request post.
- Feed store: Redis list/sorted-set theo user, lưu ID, ltrim giới hạn độ dài.
- Pagination: cursor (id/timestamp giảm dần), KHÔNG offset.
- Ranking (nếu có): tách ranking service, precompute score, merge khi serve.
- Consistency: eventual (chấp nhận trễ vài giây).
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** fan-out on-write vs on-read đánh đổi; celebrity problem & hybrid; vì sao lưu ID không lưu nội dung; vì sao cursor không offset; vì sao eventual đủ.
- 📌 **Cần THUỘC:** push=đọc nhanh/ghi nặng, pull=ghi nhẹ/đọc chậm; hybrid merge lúc đọc; cursor pagination.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế feed hybrid; xử celebrity; chọn cursor; tách ranking.

### ⑦ Mental model
> **"Push cho dân thường, pull cho ngôi sao, merge lúc đọc."** Lưu ID + hydrate; cursor không offset; eventual là đủ.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Fan-out on-write vs on-read khác gì? → write: đẩy lúc post, đọc nhanh/ghi nặng; read: gom lúc đọc, ghi nhẹ/đọc chậm.
2. *(khó)* Celebrity problem phá on-write sao, fix? → 1 post ghi triệu feed ⇒ bão ghi; hybrid push-thường + pull-celebrity, merge.
3. *(TB)* Feed lưu thế nào để đọc nhanh? → precomputed cache theo user, lưu ID post, hydrate sau, giới hạn độ dài.
4. *(TB)* Vì sao eventual consistency chấp nhận được? → trễ vài giây vô hại; ưu latency/availability.
5. *(khó)* Vì sao cursor chứ không offset cho cuộn vô tận? → offset chậm + nhảy khi có post mới; cursor ổn định/hiệu quả.

**Thách đố (trick):** *"Hybrid: user thường push, celebrity pull. Vậy một bài của celebrity, follower có thấy trong feed precomputed không?"* → **Bẫy:** quên merge. Bài celebrity **không** nằm trong push-feed (vì không fan-out); phải **pull + merge lúc đọc**. Nếu chỉ đọc push-feed, follower sẽ *thiếu* post của celebrity ⇒ bug feed.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Thiết kế feed cho 200M user, phân bố follower lệch (đa số <1k, một số >10M). Quyết push/pull/hybrid + ngưỡng. *Đạt khi:* dùng hybrid, giải thích ngưỡng celebrity, mô tả merge lúc đọc.
**BT2.** Cho feed dùng offset pagination bị "nhảy item" khi có post mới. Giải thích nguyên nhân & sửa. *Đạt khi:* chỉ ra offset dịch khi insert + chuyển sang cursor.

### ⑩ Đọc thêm
- *System Design Interview* (Alex Xu) — "Design a News Feed". `system-design-primer` — feed. Bài blog kỹ thuật về timeline/fan-out (verify nguồn/đời).

---

## Bài 12 — Case study: Video/feed platform quy mô lớn
*Phủ: A-VID-001 → A-VID-005*

### ① Mục tiêu & vị trí trong mạch
Case study **media-heavy** — ráp Bài 2 (bandwidth/storage khổng lồ), 4 (CDN + object storage + queue), 11 (feed/recommendation), 5 (counter hot row). Đây là bài "đắt nhất" về hạ tầng; mục tiêu là hiểu **pipeline upload→serve** và **vì sao mọi thứ phải async + CDN-first**.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Phục vụ video cho hàng triệu người = bài toán **băng thông**, không phải compute. Không thể stream mọi video từ một origin (sập băng thông + latency cao toàn cầu). Giải pháp xương sống: **đẩy nội dung ra rìa mạng (CDN)** + **chuẩn bị sẵn nhiều phiên bản** để client chọn theo mạng.

**(b) Cơ chế.**
- 📘 A-VID-001 — **CDN + Adaptive Bitrate (ABR)**: không stream từ origin mà qua **CDN** (gần user ⇒ giảm latency + giảm tải/băng thông gốc). **ABR** (HLS/DASH): video được chia thành **nhiều độ phân giải/bitrate**, cắt thành **segment nhỏ**; client tự chọn bitrate theo băng thông hiện tại (mạng yếu ⇒ hạ chất lượng thay vì buffer). **Pre-encode** sẵn nhiều bitrate. *(verify chi tiết hãng)*
- 📘 A-VID-002 — **Pipeline upload → serve** (vì sao **async**): `upload tới object storage → transcoding nhiều bitrate/format qua queue + worker → đẩy CDN → cập nhật metadata "ready"`. Transcoding **nặng & lâu** (phút–giờ) ⇒ KHÔNG chặn user; user thấy "đang xử lý" rồi nhận thông báo khi xong.
- 📘 A-VID-003 — **"For You" vs follow-feed** *(giao Bài 11)*: recommendation-driven (không chỉ theo follow) ⇒ cần **candidate generation + ranking + serving** riêng; mix precompute/realtime; **nặng phần ML serving**. (Pattern phổ biến.)
- 📘 A-VID-004 — **View/like count tỷ lượt**: KHÔNG `UPDATE` trực tiếp mỗi lượt (ghi nóng vào 1 row ⇒ lock/bottleneck). Dùng **gom batch / approximate counter / đẩy qua stream rồi aggregate**; chấp nhận đếm **xấp xỉ + trễ**.
- 📘 A-VID-005 — **Thumbnail/preview**: sinh **offline lúc upload** (nhiều size), lưu **object storage**, phục vụ qua **CDN**; KHÔNG sinh on-the-fly mỗi request (lãng phí compute + chậm).

> 📘 vs ➕: pipeline, CDN, ABR, counter, thumbnail đều 📘. Liên hệ ML-serving cho recommendation là ➕ (mức pattern).

### ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Stream video thẳng từ origin | CDN + ABR (HLS/DASH) | Origin sập băng thông; latency toàn cầu cao |
| Transcode đồng bộ trong request upload | Async qua queue + worker | Transcode lâu (phút–giờ); chặn user = hỏng UX |
| `UPDATE views=views+1` mỗi lượt xem | Stream + aggregate / approximate counter | Hot row ⇒ lock/bottleneck ở tỷ lượt |
| Sinh thumbnail on-the-fly | Sinh offline + lưu storage + CDN | Tiết kiệm compute, phục vụ nhanh |

### ④ Áp dụng thực tế + So sánh bigtech
Đây là xương sống của mọi nền tảng video (YouTube/Netflix/TikTok — pattern phổ biến): object storage cho master file, transcoding farm (queue + worker) sinh ABR ladder, CDN phân phối segment, recommendation tách thành hệ ML riêng. Netflix nổi tiếng với CDN đặt sâu trong mạng ISP (Open Connect) — *verify chi tiết khi trích*. View counter dùng stream processing (Kafka/Flink-like) aggregate thay vì ghi DB trực tiếp.

### ⑤ Code thực hành + cấu hình
**Upload → enqueue transcoding (async)** + **approximate view counter**:

```python
# video_pipeline.py — minh hoạ; object storage + queue + redis counter
def handle_upload(file, user_id, storage, queue, db):
    master_key = storage.put_master(file)                  # 1) lưu master vào object storage
    video_id = db.create_video(user_id, master_key, status="processing")
    # 2) enqueue transcoding cho từng bitrate — ASYNC, không chặn response upload
    for profile in ["240p", "480p", "720p", "1080p"]:
        queue.publish("transcode", {"video_id": video_id, "master_key": master_key, "profile": profile})
    return {"video_id": video_id, "status": "processing"}   # trả ngay; user nhận noti khi "ready"

def transcode_worker(msg, storage, cdn, db):               # worker (scale ngang)
    out_key = transcode(msg["master_key"], msg["profile"]) # nặng/lâu -> chạy nền
    storage.put(out_key); cdn.push(out_key)                # 3) đẩy CDN
    if db.all_profiles_done(msg["video_id"]):
        db.set_status(msg["video_id"], "ready")            # 4) cập nhật metadata khi xong

# View count xấp xỉ: tăng ở Redis, flush batch về DB định kỳ (A-VID-004)
def record_view(video_id, redis):
    redis.incr(f"views:{video_id}")                        # ghi nóng vào Redis, KHÔNG vào DB row
def flush_views(redis, db):                                # cron mỗi ~10s: gom batch -> DB
    for key in redis.scan_iter("views:*"):
        vid = key.split(":")[1]; cnt = int(redis.getset(key, 0))
        if cnt: db.increment_views(vid, cnt)               # 1 lần ghi gộp thay vì N lần
```

```text
# Cấu hình:
- Object storage cho master + variants; CDN phục vụ segment (HLS/DASH manifest + .ts/.m4s).
- Transcoding: queue + autoscaling worker farm; idempotent theo (video_id, profile).
- Counter: Redis incr + flush batch; chấp nhận trễ/xấp xỉ.
- Thumbnail: job offline lúc upload, lưu storage, phục vụ CDN.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao CDN-first; ABR giải bài băng thông client; vì sao pipeline phải async; vì sao counter không ghi DB trực tiếp.
- 📌 **Cần THUỘC:** pipeline upload→transcode→CDN→ready; HLS/DASH ABR; approximate counter; thumbnail offline.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế pipeline video; tách counter khỏi hot row; bố trí CDN + object storage.

### ⑦ Mental model
> **"Master ở object storage, biến thể ở CDN, nặng thì async, đếm thì xấp xỉ."** Bài toán video là băng thông + pipeline, không phải compute đồng bộ.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Vì sao không stream từ origin, ABR là gì? → CDN gần user giảm latency/băng thông; ABR chia nhiều bitrate, client chọn theo mạng.
2. *(khó)* Pipeline upload→serve gồm gì, vì sao async? → storage→transcode(queue+worker)→CDN→ready; transcode nặng/lâu nên không chặn user.
3. *(khó)* View count tỷ lượt — vì sao không UPDATE trực tiếp? → hot row lock; stream/aggregate/approximate counter.
4. *(TB)* "For You" khác follow-feed ở kiến trúc nào? → recommendation: candidate gen + ranking + ML serving riêng.
5. *(TB)* Thumbnail quy mô lớn sinh/serve sao? → offline lúc upload, object storage, CDN.

**Thách đố (trick):** *"Upload xong em trả response sau khi transcode để chắc chắn video sẵn sàng."* Vấn đề? → **Bẫy:** đồng bộ hoá việc nặng. Transcode 1080p có thể mất nhiều phút ⇒ request timeout, user chờ vô lý. Đúng: trả ngay status "processing", transcode nền, báo "ready" qua notification (Bài 14).

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Ước lượng storage + bandwidth cho nền tảng 1M video mới/ngày, mỗi video 100MB master × 4 bitrate, 10M view/ngày @ 5MB/view. *Đạt khi:* dùng Bài 2, tách storage (master+variant) và egress (view), kết luận CDN bắt buộc.
**BT2.** Thiết kế counter view chịu tỷ lượt/ngày không làm nóng DB. *Đạt khi:* Redis incr + flush batch (hoặc stream aggregate), chấp nhận xấp xỉ/trễ, không ghi 1 row/lượt.

### ⑩ Đọc thêm
- HLS / MPEG-DASH specs (ABR). Netflix Tech Blog — Open Connect / encoding. *System Design Interview Vol 2* (Alex Xu) — YouTube/video. *(verify đời/hãng khi trích số liệu)*

---

## Bài 13 — Case study: Chat real-time
*Phủ: A-CHAT-001 → A-CHAT-005*

### ① Mục tiêu & vị trí trong mạch
Case study về **stateful connection** — khác hẳn các bài stateless trước. Ráp Bài 3 (scale stateful khó hơn stateless), 4 (Redis pub/sub + registry), 11 (group fan-out), giao với K (ordering/at-least-once). Mục tiêu: hiểu vì sao **connection là state phải quản lý** và scale nó thế nào.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Chat khác request-response thông thường: server phải **chủ động đẩy** tin tới client *bất cứ lúc nào*. Polling (client hỏi liên tục "có tin mới không?") tốn và trễ. **WebSocket** mở một "đường ống mở sẵn" 2 chiều — nhưng đường ống này **dính vào một server cụ thể** (stateful), gây khó khi scale.

**(b) Cơ chế.**
- 📘 A-CHAT-001 — **WebSocket vs polling**: kết nối 2 chiều bền, **push realtime**, ít overhead hơn long-poll. Đổi lại: connection **stateful** ⇒ khó scale/LB, cần **connection registry**.
- 📘 A-CHAT-002 — **Scale tầng connection** (triệu WebSocket đồng thời): nhiều **gateway** giữ connection; **map user→server** (registry/pub-sub qua Redis); route message tới **đúng server đang giữ connection** của người nhận. Connection là **state phải quản lý** (khác stateless app server).
- 📘 A-CHAT-003 — **Lưu & phân phối message** (thứ tự + không mất): **persist trước khi ack** (ghi DB rồi mới báo "đã gửi"); **sequence/timestamp** cho ordering trong room; offline ⇒ **lưu rồi push khi online**; **at-least-once + dedup** (chấp nhận gửi lặp, khử trùng bằng message-id).
- 📘 A-CHAT-004 — **Presence (online/typing)**: tốn kém vì cập nhật **liên tục + fan-out** tới nhiều người. Làm nhẹ: **heartbeat + TTL trong Redis** (không nhận heartbeat ⇒ coi offline), **throttle** cập nhật, chấp nhận **trễ/xấp xỉ** (presence sai vài giây vô hại).
- 📘 A-CHAT-005 — **Group chat vs 1-1**: group = 1 message phải tới **N thành viên** (fan-out như feed); nhóm rất lớn ⇒ cân nhắc **pull/giới hạn**; vẫn cần **ordering theo room**.

### ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Polling để lấy tin mới | WebSocket (push 2 chiều) | Polling trễ + tốn; push realtime hiệu quả hơn |
| Coi WebSocket server như stateless | Connection registry (user→server) | Connection dính server cụ thể, phải route đúng |
| Ack trước khi persist | Persist trước rồi ack | Ack sớm ⇒ mất tin nếu crash trước khi ghi |
| Broadcast presence mỗi thay đổi | Heartbeat+TTL + throttle | Presence fan-out liên tục ⇒ quá tải |

### ④ Áp dụng thực tế + So sánh bigtech
Pattern phổ biến cho chat (WhatsApp/Slack/Messenger): tầng **connection gateway** stateful tách khỏi tầng **business logic** stateless; **Redis pub/sub** (hoặc message broker) làm "bảng tin trung gian" để server A gửi message tới user đang nối ở server B; message **persist** ở store bền (Cassandra-like cho write-heavy + ordering theo room). Presence dùng heartbeat+TTL. *(verify chi tiết hãng)*

### ⑤ Code thực hành + cấu hình
**Routing message qua connection registry + Redis pub/sub** (Node.js, minh hoạ):

```javascript
// chat_gateway.js — mỗi gateway giữ 1 phần WebSocket; Redis làm registry + pub/sub
// npm i ws ioredis
const WebSocket = require('ws');
const Redis = require('ioredis');
const pub = new Redis(process.env.REDIS_URL);
const sub = new Redis(process.env.REDIS_URL);
const GATEWAY_ID = process.env.HOSTNAME;
const local = new Map();                       // userId -> ws (connection LOCAL của gateway này)

const wss = new WebSocket.Server({ port: 8080 });
wss.on('connection', (ws, req) => {
  const userId = authenticate(req);            // xác thực
  local.set(userId, ws);
  pub.set(`conn:${userId}`, GATEWAY_ID, 'EX', 60);   // REGISTRY: user này đang ở gateway nào (TTL=heartbeat)
  ws.on('message', (raw) => routeMessage(JSON.parse(raw)));
  ws.on('pong', () => pub.expire(`conn:${userId}`, 60));  // heartbeat gia hạn presence (A-CHAT-004)
  ws.on('close', () => { local.delete(userId); pub.del(`conn:${userId}`); });
});

async function routeMessage(msg) {
  await persistMessage(msg);                   // PERSIST TRƯỚC khi ack (A-CHAT-003)
  for (const to of recipients(msg)) {          // 1-1 hoặc fan-out group (A-CHAT-005)
    const gw = await pub.get(`conn:${to}`);    // người nhận đang ở gateway nào?
    if (gw === GATEWAY_ID) deliverLocal(to, msg);          // cùng gateway -> đẩy thẳng
    else if (gw) pub.publish(`gw:${gw}`, JSON.stringify({ to, msg }));  // khác gateway -> qua pub/sub
    else queueOffline(to, msg);                // offline -> lưu, push khi online
  }
}
sub.subscribe(`gw:${GATEWAY_ID}`);             // nhận message gửi tới user nối ở gateway này
sub.on('message', (_ch, raw) => { const { to, msg } = JSON.parse(raw); deliverLocal(to, msg); });
```

```text
# Cấu hình & scale:
- Connection gateway tách khỏi logic; LB hỗ trợ WebSocket (sticky theo connection là chấp nhận được ở đây).
- Registry user->gateway trong Redis với TTL = chu kỳ heartbeat; hết hạn = offline.
- Message: persist trước ack; message-id để dedup (at-least-once); sequence/room cho ordering.
- Group lớn: cân nhắc pull cho thành viên ít hoạt động; throttle presence/typing.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao WebSocket hơn polling; vì sao connection là state khó scale; vì sao persist trước ack; vì sao presence tốn kém.
- 📌 **Cần THUỘC:** connection registry (user→server); at-least-once+dedup; sequence/ordering theo room; heartbeat+TTL cho presence.
- 🛠️ **Cần LÀM ĐƯỢC:** route message xuyên gateway qua pub/sub; thiết kế presence nhẹ; xử group fan-out.

### ⑦ Mental model
> **"Đường ống mở sẵn, dính server — nên cần sổ địa chỉ (registry) để gửi đúng chỗ."** Persist trước ack; presence bằng heartbeat+TTL; group = fan-out.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Vì sao chat dùng WebSocket thay polling, đổi gì? → 2 chiều/push/ít overhead; nhưng stateful, khó scale, cần registry.
2. *(khó)* Triệu WebSocket đồng thời scale sao? → nhiều gateway + registry user→server (Redis pub/sub) + route đúng server.
3. *(TB)* Đảm bảo thứ tự + không mất tin? → persist trước ack, sequence/timestamp theo room, offline lưu+push, at-least-once+dedup.
4. *(khó)* Presence tốn kém vì sao, làm nhẹ sao? → cập nhật liên tục + fan-out; heartbeat+TTL, throttle, chấp nhận xấp xỉ.
5. *(TB)* Group chat khác 1-1 ở đâu? → 1 message tới N (fan-out); nhóm lớn cân nhắc pull; vẫn ordering theo room.

**Thách đố (trick):** *Server A nhận message gửi cho user X, nhưng X đang nối ở server B. A gửi thẳng cho X được không?* → **Bẫy:** quên connection stateful. A **không giữ** socket của X. Phải tra **registry** (X ở server B) rồi route qua **pub/sub** tới B; B mới đẩy xuống socket của X. Đây chính là lý do cần registry + pub/sub.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Thiết kế chat 10M concurrent connection: bố trí gateway, registry, message store, đảm bảo ordering + không mất tin. *Đạt khi:* tách gateway/logic, registry user→server, persist-trước-ack, dedup, ordering theo room.
**BT2.** Thiết kế presence cho 10M user không làm sập hệ. *Đạt khi:* heartbeat+TTL Redis, throttle, chấp nhận trễ; không broadcast mỗi thay đổi.

### ⑩ Đọc thêm
- *System Design Interview Vol 2* (Alex Xu) — "Design a Chat System". WebSocket RFC 6455. Redis pub/sub docs.

---

## Bài 14 — Case study: Notification system đa kênh
*Phủ: A-NOTI-001 → A-NOTI-004*

### ① Mục tiêu & vị trí trong mạch
Case study **queue + worker + reliability** — ráp Bài 4 (queue), 5 (retry/DLQ/dedup), 11 (fan-out), giao với K (messaging). Đây là "hệ ống dẫn" điển hình; mục tiêu: hiểu cách tách **policy** (gửi cho ai, khi nào) khỏi **delivery** (đẩy ra provider) và đảm bảo **không trùng, không mất, không spam**.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Notification = bưu điện đa kênh: nhận yêu cầu "gửi tin này cho user", chọn kênh (email/push/SMS), gọi nhà cung cấp (provider), thử lại nếu hỏng, tôn trọng "đừng làm phiền" của user. Chìa khóa: **không gửi trùng, không mất, không spam, không chặn hệ chính**.

**(b) Cơ chế.**
- 📘 A-NOTI-001 — **Kiến trúc tối thiểu**: `ingestion → queue → per-channel worker/provider adapter → retry/DLQ`. Tách: **template service** (render nội dung), **user preference**, **observability** (gửi/nhận/bounce). Mỗi kênh có adapter riêng (email/push/SMS) vì API provider khác nhau.
- 📘 A-NOTI-002 — **Dedup + retry**: **idempotency key / dedup store** (cùng notification không gửi 2 lần); provider lỗi ⇒ **exponential backoff + DLQ** (dead-letter queue cho cái thất bại mãi); **at-least-once** nên **cần dedup** ở consumer; theo dõi **delivery status**.
- 📘 A-NOTI-003 — **Throttle + user preference** (opt-out, quiet hours): áp **TRƯỚC khi đẩy provider** — tôn trọng preference + chống spam + giới hạn **quota provider**. **Tách policy khỏi delivery** (quyết "có gửi không" tách khỏi "gửi thế nào").
- 📘 A-NOTI-004 — **Fan-out sự kiện triệu user** (vd "live started"): đẩy event vào **queue**, **worker scale ngang**, **batch theo kênh**, **throttle provider**; KHÔNG gửi **đồng bộ inline** trong request (sẽ timeout + đập provider).

### ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Hay gặp (sai) | Nên làm | Vì sao |
|---|---|---|
| Gọi provider đồng bộ trong request | Đẩy queue + worker async | Provider chậm/lỗi sẽ chặn/timeout request chính |
| Không dedup (at-least-once) | Idempotency key + dedup store | Retry/redelivery ⇒ user nhận trùng |
| Bỏ qua preference/quiet hours | Áp policy trước khi gửi | Spam ⇒ user opt-out / provider phạt |
| Retry vô hạn khi provider hỏng | Exponential backoff + DLQ | Tránh bão retry; cô lập lỗi để xử riêng |

### ④ Áp dụng thực tế + So sánh bigtech
Mọi hệ noti (push của app, email marketing, OTP) đều theo pattern queue→worker→provider adapter với retry/DLQ + preference + dedup (pattern phổ biến). ➕ Thực tế: provider adapter trừu tượng hoá nhiều vendor (vd nhiều SMS gateway để failover), rate limit theo quota từng provider, và pipeline observability theo dõi delivery/bounce/open-rate. Fan-out lớn dùng batch + throttle để không vượt quota provider.

### ⑤ Code thực hành + cấu hình
**Pipeline noti: enqueue → worker dedup + preference + retry/DLQ** (Python, minh hoạ):

```python
# notification.py — minh hoạ; queue + redis dedup + provider adapter
def notify(user_id, event, payload, queue):
    # idempotency key: cùng (user,event,payload-hash) chỉ tạo 1 noti (A-NOTI-002)
    idem = f"{user_id}:{event}:{hash(frozenset(payload.items()))}"
    queue.publish("notifications", {"user_id": user_id, "event": event,
                                    "payload": payload, "idem": idem})

def worker(msg, redis, prefs, providers, dlq):
    # 1) DEDUP: nếu idem đã xử lý -> bỏ qua (at-least-once nên phải khử trùng)
    if not redis.set(f"noti:idem:{msg['idem']}", 1, nx=True, ex=86400):
        return
    pref = prefs.get(msg["user_id"])
    # 2) POLICY trước DELIVERY (A-NOTI-003): opt-out, quiet hours, throttle
    if pref.opted_out(msg["event"]):      return
    if pref.in_quiet_hours():             return queue_for_later(msg)
    if pref.over_rate_limit(msg["event"]): return   # chống spam + quota provider
    # 3) DELIVERY theo từng kênh, có retry/backoff + DLQ
    for channel in pref.channels(msg["event"]):      # email / push / sms
        try:
            providers[channel].send(msg["user_id"], render(channel, msg))  # adapter per-channel
        except ProviderError as e:
            if msg.get("attempt", 0) < 5:
                requeue_with_backoff(msg, channel)   # exponential backoff
            else:
                dlq.put(msg, reason=str(e))          # DLQ sau khi hết lượt thử
```

```text
# Cấu hình:
- Ingestion -> queue (Kafka/SQS); worker scale ngang theo kênh.
- Dedup: idempotency key trong Redis (SET NX EX); xử lý at-least-once an toàn.
- Policy: preference service (opt-out/quiet hours/rate) áp TRƯỚC delivery.
- Retry: exponential backoff + jitter; DLQ cho lỗi vĩnh viễn; alert khi DLQ tăng.
- Fan-out lớn: batch theo kênh + throttle theo quota provider; KHÔNG gửi inline.
```

### ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao async qua queue; vì sao at-least-once cần dedup; tách policy khỏi delivery; vì sao fan-out lớn phải batch+throttle.
- 📌 **Cần THUỘC:** ingestion→queue→worker→provider; idempotency key; backoff+DLQ; quiet hours/opt-out/rate limit.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế pipeline noti reliable; cài dedup; xử fan-out triệu user.

### ⑦ Mental model
> **"Queue ở giữa, policy trước delivery, dedup chống trùng, DLQ giữ lỗi."** Không bao giờ gọi provider đồng bộ trong request.

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Kiến trúc tối thiểu noti đa kênh? → ingestion→queue→per-channel worker/adapter→retry/DLQ + template + preference + observability.
2. *(khó)* Đảm bảo không trùng + retry khi provider lỗi? → idempotency/dedup store; backoff+DLQ; at-least-once nên dedup.
3. *(TB)* Throttle + preference đặt ở đâu? → trước khi đẩy provider; tách policy khỏi delivery.
4. *(khó)* Fan-out "live started" triệu user chịu tải sao? → event→queue, worker scale ngang, batch+throttle, không inline.
5. *(TB)* Vì sao không gọi provider đồng bộ? → provider chậm/lỗi chặn request + đập quota.

**Thách đố (trick):** *Hệ retry khi provider timeout, nhưng user phàn nàn nhận 3 email giống nhau.* Nguyên nhân? → **Bẫy:** retry mà không dedup. Provider có thể đã gửi thành công *rồi* mới timeout phản hồi ⇒ retry gửi lại. Cần **idempotency key** ở provider (nếu hỗ trợ) + **dedup store** + theo dõi delivery status để không gửi lại cái đã gửi.

### ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Thiết kế hệ noti cho event "đơn hàng đã giao" gửi email+push, tôn trọng quiet hours. *Đạt khi:* queue+worker, dedup, policy trước delivery, retry+DLQ.
**BT2.** Xử "live started" cho streamer 5M follower mà không đập provider. *Đạt khi:* event→queue, worker scale ngang, batch theo kênh, throttle quota.

### ⑩ Đọc thêm
- *System Design Interview Vol 2* (Alex Xu) — "Design a Notification System". AWS — SQS DLQ; idempotency patterns.

---

# 🟦 BÀI 15 — E-commerce Checkout: Chống Oversell & Thanh Toán Idempotent

> **Khối IV — Case study. Phủ câu hỏi:** A-CHK-001 → A-CHK-005.
> **Giao với:** Bài 4 (transaction/lock), Bài 6 (consistency), Bài 11 (fan-out đọc). Đây là case study **strong-consistency** đối lập với feed/chat (thiên eventual).

## ① Mục tiêu & vị trí trong mạch
Checkout là nơi **không được sai về tiền và tồn kho**. Sai một ly là oversell (bán quá số hàng có), trừ tiền hai lần, hoặc giữ hàng vĩnh viễn. Bài này dạy bạn ba thứ cốt lõi: (1) **trừ tồn kho atomic** chống race, (2) **idempotency** cho thanh toán chống trùng, (3) **Saga + Outbox** thay vì 2PC để giữ nhất quán giữa nhiều service. Đây là bài "kết tinh" của khối Đánh đổi: bạn sẽ thấy vì sao ở đây ta **chọn strong consistency**, ngược với feed.

## ② Giảng: cơ bản → nâng cao

### 📘 Vì sao "đọc rồi trừ" (read-modify-write) là SAI — race condition
Naive code:
```
qty = SELECT stock FROM product WHERE id=1   -- đọc được 1
if qty > 0:
    SELECT ... process ...
    UPDATE product SET stock = qty - 1 WHERE id=1  -- ghi đè bằng 0
```
Hai request chạy song song **cùng đọc qty=1**, cùng thấy `>0`, cùng trừ → bán 2 món dù chỉ có 1. Đây là **lost update / race condition** kinh điển. Khoảng trống giữa *đọc* và *ghi* là nơi tai nạn xảy ra.

### 📘 Ba cách trừ tồn kho atomic (đóng khoảng trống đọc–ghi)

**Cách 1 — Atomic conditional UPDATE (đơn giản & mạnh nhất cho 1 dòng):**
```sql
-- DB tự khóa dòng trong câu UPDATE; điều kiện qty>0 nằm NGAY trong câu ghi
UPDATE product
SET stock = stock - 1
WHERE id = 1 AND stock > 0;
-- Kiểm tra affected_rows: =1 là giữ được hàng, =0 là hết hàng (KHÔNG cho mua)
```
Không có khoảng trống đọc–ghi: điều kiện và phép trừ là **một câu lệnh atomic**. Đây là lựa chọn mặc định, rẻ và đúng.

**Cách 2 — SELECT ... FOR UPDATE (pessimistic lock, khi cần đọc nhiều dòng rồi quyết định):**
```sql
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;  -- khóa dòng tới hết transaction
-- ... logic phức tạp: kiểm nhiều sản phẩm, tính combo ...
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;  -- nhả khóa
```
Dùng khi logic cần "khóa rồi suy nghĩ". Đánh đổi: **giữ khóa lâu = giảm throughput**, dễ thành bottleneck (Bài 5) khi hot product.

**Cách 3 — Redis atomic (cho flash sale tải cực cao, giảm tải DB):**
```
DECR stock:product:1     -- atomic, trả về giá trị mới
-- nếu kết quả < 0 => hết hàng => INCR trả lại + từ chối
```
Redis single-thread nên `DECR` atomic tự nhiên. Cực nhanh, nhưng phải **đồng bộ ngược về DB** (source of truth) và xử lý khi Redis chết.

### 📘 Reservation có TTL — vì sao cần "giữ chỗ tạm"
Người dùng bấm "mua" nhưng chưa trả tiền. Nếu trừ hẳn → bỏ giỏ thì hàng kẹt. Nếu không trừ → oversell lúc thanh toán. Giải pháp: **reservation (giữ chỗ) có thời hạn (TTL)**.
```
1. Reserve: trừ available, ghi reservation{order, qty, expire_at = now+15p}
2. Trả tiền trong 15p  -> commit: chuyển reserved thành sold
3. Quá hạn / hủy       -> release: cộng trả available (compensating action)
```
TTL biến "giữ vĩnh viễn" thành "giữ tạm". Job quét hết hạn (hoặc Redis key TTL) tự nhả hàng.

### ⬆️ Idempotency cho thanh toán — chống trừ tiền 2 lần
Mạng timeout, user bấm "Pay" lại, client retry → nguy cơ **charge 2 lần**. Cách chặn: **idempotency key**.
```
- Client sinh 1 key duy nhất cho mỗi ý định trả tiền (vd uuid), gửi kèm request.
- Server: trước khi charge, SET key vào store (Redis/DB) với điều kiện "chưa tồn tại".
  + Nếu key MỚI    -> thực hiện charge, lưu kết quả gắn với key.
  + Nếu key ĐÃ CÓ  -> trả lại KẾT QUẢ CŨ, KHÔNG charge lần nữa.
```
Mọi payment gateway nghiêm túc (Stripe, PayPal…) đều nhận `Idempotency-Key`. Quy tắc TL: **request làm thay đổi tiền/tồn kho bắt buộc idempotent**.

### ⬆️ Saga + Compensating action + Outbox — thay cho 2PC
Checkout chạm nhiều service: Inventory, Payment, Order, Shipping. Muốn "tất cả cùng thành công hoặc cùng hủy". **2PC (two-phase commit)** làm được trên giấy nhưng **khóa tài nguyên xuyên service, chậm, kẹt khi coordinator chết** → gần như không ai dùng ở quy mô lớn.

Thay bằng **Saga**: chuỗi bước cục bộ, mỗi bước có **compensating action** (hành động bù trừ) nếu bước sau hỏng:
```
reserve_inventory  --hỏng-->  (không cần bù, chưa làm gì)
charge_payment     --hỏng-->  release_inventory          (bù bước trước)
create_order       --hỏng-->  refund_payment + release_inventory
confirm_shipping   --hỏng-->  cancel_order + refund + release
```
Để **không mất event giữa "ghi DB" và "bắn message"**, dùng **Outbox pattern**: ghi event vào bảng `outbox` **trong cùng transaction** với thay đổi nghiệp vụ; một relay đọc outbox và publish lên message bus → đảm bảo *atomic* giữa state và event (chống mất/trùng nửa vời). At-least-once + consumer idempotent (Bài 14).

### ⬆️ Flash sale — khi 100k người giành 1000 món trong 1 giây
Vấn đề: hot key trên 1 product (Bài 16), DB sập vì khóa. Pattern phổ biến:
```
1. Waiting room / queue: chặn dòng người ngay cửa, thả vào từ từ (token).
2. Preload tồn kho vào Redis: DECR atomic ở Redis, KHÔNG đụng DB mỗi request.
3. Ai DECR ra >=0 mới được "giữ chỗ"; số còn lại trả "hết hàng" ngay (fail fast).
4. Ghi nhận người thắng -> đẩy async xuống DB qua queue (làm phẳng burst).
5. Reconcile: định kỳ đối soát Redis <-> DB; Redis chết thì có cơ chế khôi phục.
```
Tinh thần: **đẩy điểm tranh chấp ra Redis atomic + làm phẳng ghi xuống DB bằng queue**, fail fast cho người thua.

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Sai lầm hay gặp | Vì sao sai | Cách đúng |
|---|---|---|
| `SELECT qty; if>0; UPDATE qty-1` | Race: 2 request cùng đọc, cùng trừ → oversell | `UPDATE ... WHERE stock>0` atomic / `FOR UPDATE` |
| Trừ hẳn tồn kho lúc add-to-cart | Bỏ giỏ → hàng kẹt vĩnh viễn | Reservation **có TTL**, hết hạn tự nhả |
| Không có idempotency key cho Pay | Retry/double-click → charge 2 lần | Idempotency key, trả kết quả cũ nếu trùng |
| Dùng 2PC cho cross-service | Khóa lâu, kẹt khi coordinator chết, không scale | **Saga + compensating action + Outbox** |
| Ghi DB xong mới bắn message (2 bước rời) | Crash giữa chừng → mất/trùng event | **Outbox** trong cùng transaction |
| Flash sale đập thẳng DB mỗi request | Hot key + khóa → DB sập | Redis atomic DECR + queue làm phẳng + fail fast |
| Coi checkout như feed (eventual OK) | Tiền/tồn kho sai là không chấp nhận được | Checkout = **strong consistency** có chủ đích |

## ④ Áp dụng thực tế + so sánh bigtech (pattern phổ biến)
- **Sàn TMĐT lớn** thường tách *available / reserved / sold*, reservation TTL, và **Saga** điều phối Inventory–Payment–Order. (Mô tả ở mức pattern, không khẳng định chi tiết nội bộ.)
- **Payment gateway** (Stripe/PayPal) công khai hỗ trợ **Idempotency-Key** — đây là chuẩn ngành, không phải nội bộ.
- **Flash sale** (kiểu Xiaomi/loại "giật" hàng) phổ biến dùng **waiting room + Redis preload + atomic decrement** — pattern được chia sẻ rộng rãi.
- Quy tắc TL chốt: ở **checkout chọn strong consistency**; ở **feed/notification chấp nhận eventual**. Biết *khi nào nghiêng bên nào* mới là năng lực thật (nối Bài 6).

## ⑤ Code thực hành + cấu hình

**(a) Trừ tồn kho atomic + reservation TTL (Python, psycopg) — KHÔNG hardcode secret:**
```python
import os, uuid, psycopg2
# Cấu hình từ ENV, không bao giờ hardcode chuỗi kết nối/mật khẩu
DB_DSN = os.environ["DB_DSN"]  # ví dụ: postg:// user:pass @host/db (đặt ở ENV/secret manager)

def reserve_stock(product_id: int, qty: int, order_id: str, ttl_minutes: int = 15) -> bool:
    """Giữ chỗ tồn kho atomic. Trả True nếu giữ được, False nếu hết hàng."""
    with psycopg2.connect(DB_DSN) as conn, conn.cursor() as cur:
        # Điều kiện available>=qty NẰM TRONG câu UPDATE => atomic, không có khoảng trống đọc-ghi
        cur.execute(
            """
            UPDATE inventory
               SET available = available - %s,
                   reserved  = reserved  + %s
             WHERE product_id = %s AND available >= %s
            """,
            (qty, qty, product_id, qty),
        )
        if cur.rowcount == 0:
            return False  # hết hàng -> từ chối, KHÔNG cho mua

        # Ghi reservation kèm hạn dùng; job/cron sẽ quét cái quá hạn để release
        cur.execute(
            """
            INSERT INTO reservation (order_id, product_id, qty, expire_at)
            VALUES (%s, %s, %s, now() + (%s || ' minutes')::interval)
            """,
            (order_id, product_id, qty, ttl_minutes),
        )
        return True  # commit tự động khi thoát "with conn"
```

**(b) Thanh toán idempotent (Redis SET NX) — chống charge 2 lần:**
```python
import os, json, redis
r = redis.Redis.from_url(os.environ["REDIS_URL"])  # URL lấy từ ENV

def charge_once(idem_key: str, order_id: str, amount_cents: int) -> dict:
    """Idempotent charge: cùng idem_key chỉ charge 1 lần, lần sau trả kết quả cũ."""
    cache_key = f"charge:idem:{idem_key}"
    # SET NX: chỉ set nếu CHƯA tồn tại; giữ 24h để hứng mọi retry hợp lý
    acquired = r.set(cache_key, "PENDING", nx=True, ex=24 * 3600)
    if not acquired:
        prev = r.get(cache_key)
        if prev and prev != b"PENDING":
            return json.loads(prev)          # đã có kết quả -> trả lại, KHÔNG charge
        raise RuntimeError("charge đang xử lý, hãy thử lại sau")  # tránh double in-flight

    result = call_payment_gateway(order_id, amount_cents)  # gọi gateway (cũng nên gửi Idempotency-Key)
    r.set(cache_key, json.dumps(result), ex=24 * 3600)      # lưu kết quả gắn với key
    return result
```

**(c) Outbox pattern (event không lạc khỏi transaction):**
```python
def place_order_with_outbox(cur, order: dict, event: dict):
    """Ghi đơn + ghi outbox trong CÙNG transaction => state và event atomic với nhau."""
    cur.execute(
        "INSERT INTO orders (id, user_id, total_cents, status) VALUES (%s,%s,%s,'CREATED')",
        (order["id"], order["user_id"], order["total_cents"]),
    )
    # Cùng transaction: nếu rollback thì cả đơn lẫn event đều biến mất -> không lệch
    cur.execute(
        "INSERT INTO outbox (id, topic, payload, status) VALUES (%s,%s,%s,'NEW')",
        (str(uuid.uuid4()), "order.created", json.dumps(event)),
    )
    # Một relay riêng đọc outbox status='NEW' -> publish message bus -> đánh dấu 'SENT'
    # Consumer phía sau idempotent (Bài 14) vì giao hàng at-least-once.
```

**(d) Cấu hình Postgres giảm kẹt khóa khi tải cao (gợi ý, tinh chỉnh theo tải thật):**
```ini
# postgresql.conf — đặt thời gian chờ để fail fast thay vì treo vô hạn khi tranh khóa
lock_timeout = '2s'                 # chờ khóa quá 2s thì bỏ, tránh kẹt dây chuyền
idle_in_transaction_session_timeout = '10s'  # giết transaction "ngậm khóa" rồi ngồi im
statement_timeout = '5s'            # chặn truy vấn chạy quá lâu giữ khóa
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao read-modify-write race; vì sao reservation cần TTL; vì sao Saga thay 2PC; vì sao Outbox cần cùng transaction; vì sao checkout nghiêng strong consistency.
- 📌 **Cần THUỘC:** `UPDATE...WHERE stock>0`; `SELECT FOR UPDATE`; Redis `DECR` atomic; idempotency key; compensating action; available/reserved/sold.
- 🛠️ **Cần LÀM ĐƯỢC:** viết trừ tồn kho atomic; cài idempotent charge; phác Saga có bù trừ; thiết kế flash sale Redis preload + queue.

## ⑦ Mental model
> **"Đóng khoảng trống đọc–ghi (atomic), giữ chỗ có hạn (TTL), trả tiền một lần (idempotent), nhiều service thì bù trừ (Saga) chứ đừng khóa chéo (2PC)."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Vì sao "đọc tồn kho, nếu >0 thì trừ" gây oversell? Sửa thế nào? → race giữa đọc và ghi; sửa bằng `UPDATE...WHERE stock>0` atomic hoặc `FOR UPDATE`.
2. *(TB)* Reservation TTL giải quyết vấn đề gì? → giữ hàng tạm cho người đang thanh toán mà không kẹt hàng khi bỏ giỏ; hết hạn tự nhả (compensating).
3. *(khó)* Chống charge 2 lần khi client retry? → idempotency key: lần đầu charge & lưu kết quả, lần sau trả kết quả cũ; gateway cũng nhận Idempotency-Key.
4. *(khó)* Vì sao không 2PC mà dùng Saga? Outbox để làm gì? → 2PC khóa chéo/kẹt coordinator/không scale; Saga = bước cục bộ + bù trừ; Outbox giữ event atomic với state.
5. *(TB)* Flash sale 100k giành 1000 món chịu tải sao? → waiting room + Redis preload + `DECR` atomic + fail fast + queue làm phẳng ghi xuống DB.

**Thách đố (trick):** *Team cho rằng "dùng transaction DB là đủ chống oversell, khỏi cần Redis hay reservation".* Phản biện? → **Bẫy:** transaction đúng cho **một DB**, nhưng (a) hot product 1 dòng bị khóa nối đuôi → throughput sụp ở flash sale; (b) tiền nằm ở **service khác** (payment) nên một transaction DB không bao trùm → cần Saga/idempotency; (c) "giữ chỗ trong lúc user trả tiền" là vấn đề **thời gian/TTL**, transaction không giải được. Transaction là *điều kiện cần*, không phải *đủ*.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Thiết kế luồng checkout chống oversell cho sản phẩm thường (không flash sale). *Đạt khi:* trừ tồn kho atomic, reservation TTL, idempotent payment, Saga có compensating cho payment-fail.
**BT2.** Nâng cấp cho **flash sale** 1000 món / 100k người / 1 giây. *Đạt khi:* Redis preload + atomic decrement, fail fast, queue làm phẳng ghi DB, có reconcile Redis↔DB, nêu xử lý Redis chết.
**BT3.** Vẽ Saga 4 bước (reserve→charge→order→ship) kèm compensating action từng bước và chỉ rõ chỗ đặt Outbox. *Đạt khi:* mỗi bước có hành động bù đúng thứ tự ngược, Outbox nằm cùng transaction với state.

## ⑩ Đọc thêm
- *Microservices Patterns* (Chris Richardson) — Saga, Outbox, transactional messaging.
- Stripe Docs — *Idempotent requests*. AWS — *Saga orchestration với Step Functions*.
- *Designing Data-Intensive Applications* (Kleppmann) — ch. weak isolation & race conditions (lost update, write skew).


---

# 🟪 BÀI 16 — Phán Đoán Tech Lead: Hiểu Để Chỉ Huy & Kiểm Tra

> **Khối V — Lăng kính tổng hợp. Phủ câu hỏi:** A-RT-001, A-RT-002 + soi lại toàn bộ Bài 1→15.
> **Đây không phải bài "thêm kiến thức" mà là bài "đổi vai":** từ người *trả lời đúng* thành người *phán đoán đúng* — biết cái gì sai trước khi nó sai, biết "đúng trên giấy" khác "sống ở production".

## ① Mục tiêu & vị trí trong mạch
Mười lăm bài trước cho bạn **vũ khí**. Bài này dạy **khi nào rút vũ khí nào** và **làm sao biết người khác đang chọn sai**. Một Tech Lead không cần gõ lại từng dòng — họ cần (1) **phản biện một lựa chọn kiến trúc** bằng câu hỏi đúng, và (2) **dự đoán cái gì sẽ vỡ ở production** dù thiết kế "nhìn đẹp". Đây là nơi câu thần chú của cả mục A kết tinh:
> *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

## ② Giảng: hai năng lực phán đoán cốt lõi

### 📘 Năng lực 1 — Phản biện lựa chọn công nghệ (A-RT-001)
**Tình huống mẫu:** Có người đề xuất *fine-tune một LLM* để làm chatbot trả lời theo **tài liệu nội bộ hay thay đổi**. Nghe "AI xịn", nhưng TL phải thấy ngay đây là **lựa chọn sai công cụ**.

**Vì sao sai:**
- Tài liệu **hay đổi** ⇒ fine-tune là "nướng" kiến thức vào trọng số model; đổi tài liệu là phải **train lại** → tốn tiền, tốn thời gian, chậm cập nhật.
- Fine-tune **dễ hallucinate** dữ kiện và **khó truy nguồn** (không chỉ ra "câu trả lời lấy từ trang nào").
- Không kiểm soát được phiên bản tri thức; rủi ro nói sai thông tin nội bộ.

**Lựa chọn đúng: RAG (Retrieval-Augmented Generation) / lookup.**
- Để tài liệu ở ngoài (vector store / search); khi hỏi thì **truy hồi đoạn liên quan** rồi nhét vào prompt cho model trả lời.
- Đổi tài liệu = cập nhật index, **không train lại**; trả lời **trích nguồn được**; giảm hallucinate.
- Fine-tune chỉ hợp khi cần **đổi phong cách/định dạng/giọng** hoặc kỹ năng ổn định, **không phải để nhồi tri thức hay đổi**.

> **Khung phản biện tổng quát (áp cho MỌI đề xuất công nghệ):**
> 1. **Bản chất bài toán là gì?** (tri thức hay-đổi? hay phong cách cố định?)
> 2. **Công cụ này khớp bản chất đó không?** (fine-tune hợp "kỹ năng/giọng", RAG hợp "tri thức động")
> 3. **Chi phí thay đổi & vận hành?** (train lại tốn gì? cập nhật ra sao?)
> 4. **Có cách đơn giản hơn đạt 80% kết quả không?** (đừng dùng búa tạ đập đinh ghim)
> 5. **Đo bằng gì để biết đúng/sai?** (không có metric thì không phải kỹ thuật, là niềm tin)

### 📘 Năng lực 2 — Dự đoán cái "đúng trên giấy nhưng vỡ ở production" (A-RT-002)
Thiết kế trên whiteboard giả định **tải đều, mạng hoàn hảo, dữ liệu phân bố đẹp**. Production thì **ngược lại**. TL giỏi là người **nhìn sơ đồ đẹp và chỉ ra điểm sẽ vỡ**. Danh mục "tử huyệt" hay gặp — đây là checklist vàng:

| Tử huyệt production | "Trên giấy" trông sao | Thực tế vỡ thế nào | Truy về bài |
|---|---|---|---|
| **Hot key / partition lệch** | "Sharding chia đều tải" | 1 key/1 shard nóng (celeb, sản phẩm flash sale) ôm phần lớn traffic | Bài 11, 15, 5 |
| **Cold start** | "Serverless tự co giãn, rẻ" | Burst → loạt cold start → p99 latency vọt; pool DB cạn | Bài 8 |
| **Cache stampede** | "Có cache là nhanh" | Key hết hạn đồng loạt → ngàn request cùng đập DB | Bài 4, 12 |
| **Connection pool cạn** | "Mỗi instance nối DB là xong" | Scale ngang × pool/instance > max_connections DB → DB từ chối | Bài 8, 5 |
| **Network partition** | "Các node nói chuyện với nhau" | Mạng đứt → split-brain / phải chọn C hay A (CAP) | Bài 6 |
| **Thundering herd / retry storm** | "Lỗi thì retry" | Lỗi lan → mọi client retry đồng loạt → tự DDoS chính mình | Bài 14, 3 |
| **Chi phí ở tải thật** | "Kiến trúc này chạy được" | Đúng nhưng hóa đơn gấp 10 lần; serverless/egress/cross-AZ ăn tiền | Bài 8, 2 |
| **Thiếu observability** | "Chạy là biết" | Vỡ mà không có metrics/log/trace → mù, không rollback nổi | Toàn mục |
| **Thiếu rollback / migration nguy hiểm** | "Deploy là xong" | Schema change khóa bảng giờ cao điểm; không có đường lùi | Bài 4, 5 |
| **Clock skew / thời gian** | "Lấy timestamp là xong" | Đồng hồ lệch giữa node → ID trùng, ordering sai | Bài 4, 13 |

### ⬆️ Tư duy "soi lại 15 bài qua lăng kính TL"
Mỗi bài trước giờ trở thành **một câu hỏi kiểm tra** bạn ném vào bất kỳ thiết kế nào:
- *(Bài 1)* Đã làm rõ scope/giả định/SLA chưa, hay nhảy vào vẽ box ngay?
- *(Bài 2)* Con số tải/throughput/dung lượng có ước lượng không, hay "chắc ổn"?
- *(Bài 3,4,5)* Điểm nghẽn nằm đâu? LB, cache, queue đặt đúng chỗ chưa? Cái gì là single bottleneck?
- *(Bài 6)* Chỗ nào cần strong, chỗ nào eventual? Có nhầm lẫn không?
- *(Bài 7,8)* Cắt service theo nghiệp vụ hay theo tầng kỹ thuật? Serverless có hợp workload không?
- *(Bài 9→15)* Mỗi case study có đúng pattern lõi (atomic, idempotent, fan-out, dedup…) không?

> **Mental shift:** người mới hỏi *"thiết kế này làm thế nào?"*; Tech Lead hỏi *"thiết kế này sai ở đâu, đắt ở đâu, vỡ khi nào, và đo bằng gì?"*.

## ③ ⚠️ Cũ → mới / sai lầm hay gặp

| Sai lầm tư duy | Vì sao nguy hiểm | Tư duy TL đúng |
|---|---|---|
| Chọn công nghệ vì "hot/xịn" (fine-tune mọi thứ) | Sai công cụ cho bản chất bài toán | Khớp công cụ với **bản chất** (RAG cho tri thức động) |
| Tin "thiết kế đẹp = chạy tốt" | Production có hot key, cold start, partition | Luôn hỏi "vỡ ở đâu khi tải thật/mạng đứt?" |
| Bỏ qua chi phí | Kiến trúc đúng nhưng phá sản vì hóa đơn | Đưa **cost** thành tiêu chí thiết kế |
| Không có observability/rollback | Vỡ mà mù, không lùi được | Coi metrics/log/trace/rollback là **bắt buộc** |
| "Retry là xử lý lỗi" | Retry storm tự DDoS | Backoff + jitter + circuit breaker + DLQ |
| Phán đoán không kèm metric | Tranh luận theo cảm tính | Mọi quyết định gắn **số đo** |

## ④ Áp dụng thực tế + so sánh bigtech (pattern phổ biến)
- Văn hóa **design review / RFC** ở các công ty lớn chính là thể chế hóa hai năng lực trên: bắt người đề xuất trả lời "vỡ ở đâu, đo bằng gì, chi phí bao nhiêu".
- Thực hành **chaos engineering** (cố tình gây lỗi để lộ tử huyệt trước khi production lộ) là biểu hiện của tư duy "đúng trên giấy chưa đủ".
- **RAG vs fine-tune** giờ là chủ đề phổ biến: cộng đồng đồng thuận RAG cho tri thức hay-đổi, fine-tune cho phong cách/kỹ năng — đúng như khung phản biện ở trên.
- Quy tắc TL chốt mục A: **bạn không cần thuộc mọi con số; bạn cần biết câu hỏi nào vạch ra điểm yếu** và tra cứu phần chi tiết khi cần.

## ⑤ Code/Checklist thực hành
Bài này "phán đoán" nên "code" là **checklist review** bạn dán lên mỗi thiết kế:

```text
# DESIGN REVIEW CHECKLIST (lăng kính Tech Lead) — dùng cho mọi đề xuất

[ Bài toán & lựa chọn ]
□ Bản chất bài toán đã nêu rõ? (tri thức động? phong cách? throughput? consistency?)
□ Công cụ đề xuất KHỚP bản chất? (vd: tri thức hay-đổi -> RAG, KHÔNG fine-tune)
□ Có phương án đơn giản hơn đạt ~80% kết quả?

[ Quy mô & số liệu ]
□ Ước lượng tải/QPS/dung lượng/băng thông? (Bài 2)
□ Hot key / partition lệch đã tính? Ai là "celeb/flash-sale" của hệ này? (Bài 11,15)

[ Điểm vỡ production ]
□ Cold start / pool DB cạn khi scale ngang? (Bài 8)
□ Cache stampede / thundering herd / retry storm? (Bài 4,14)
□ Network partition -> chọn C hay A? split-brain? (Bài 6)
□ Strong vs eventual đặt đúng chỗ? (Bài 6)

[ Vận hành ]
□ Observability: có metrics/log/trace để biết khi nào vỡ?
□ Rollback / migration an toàn (không khóa bảng giờ cao điểm)?
□ Chi phí ở TẢI THẬT (không chỉ tải demo)? egress/cross-AZ/serverless?

[ Đo lường ]
□ Mỗi quyết định gắn 1 metric để biết đúng/sai?
```

## ⑥ Keywords cần nhớ
- 🧠 **Cần HIỂU:** vì sao RAG ≠ fine-tune theo bản chất bài toán; vì sao "đúng trên giấy" hay vỡ (hot key, cold start, partition, stampede, cost); vì sao mọi quyết định cần metric.
- 📌 **Cần THUỘC:** khung phản biện 5 câu hỏi; checklist tử huyệt production; RAG cho tri thức động / fine-tune cho phong cách.
- 🛠️ **Cần LÀM ĐƯỢC:** phản biện một đề xuất kiến trúc trong 2 phút; chỉ ra ≥3 điểm sẽ vỡ ở production cho một sơ đồ "đẹp"; gắn metric cho mỗi quyết định.

## ⑦ Mental model
> **"Người mới hỏi 'làm thế nào'; Tech Lead hỏi 'sai ở đâu, đắt ở đâu, vỡ khi nào, đo bằng gì'. Hiểu để chỉ huy — tra cứu phần còn lại."**

## ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Có người muốn fine-tune LLM cho chatbot tra cứu tài liệu nội bộ hay đổi. Bạn phản biện sao? → sai công cụ; tài liệu động nên dùng **RAG/lookup** (cập nhật index không train lại, trích nguồn, ít hallucinate); fine-tune chỉ hợp phong cách/kỹ năng.
2. *(khó)* Kể 5 thứ "đúng trên giấy nhưng vỡ ở production" và cách phát hiện sớm. → hot key, cold start, cache stampede, pool cạn, network partition / retry storm; phát hiện qua load test, chaos, observability, ước lượng tải.
3. *(TB)* Vì sao mọi quyết định kiến trúc nên gắn metric? → tránh tranh luận cảm tính; có cơ sở rollback; biết khi nào giả định sai.
4. *(khó)* Một sơ đồ microservices "đẹp" — bạn hỏi 3 câu nào đầu tiên để tìm điểm yếu? → (a) hot key/điểm nghẽn đơn ở đâu? (b) strong/eventual đặt đúng chỗ chưa? (c) chi phí & observability ở tải thật ra sao?
5. *(TB)* Khi nào fine-tune thực sự hợp lý? → khi cần đổi **phong cách/định dạng/giọng** hoặc kỹ năng ổn định, dữ liệu ít đổi — không phải để nhồi tri thức hay-đổi.

**Thách đố (trick):** *Sếp nói: "thiết kế này đã chạy ổn ở môi trường staging 1 tuần, cứ lên production."* Bạn — Tech Lead — phản biện thế nào? → **Bẫy:** staging **không tái hiện** production: (a) **tải staging nhỏ** → hot key/partition lệch/pool cạn chưa lộ; (b) **không có traffic thật** → cold start, cache stampede, retry storm chưa xảy ra; (c) **dữ liệu staging sạch/nhỏ** → query chậm trên bảng lớn chưa thấy; (d) **chưa trải mạng đứt** → CAP chưa bị thử. "Chạy ổn ở staging" chỉ chứng minh *không có lỗi hiển nhiên*, **không** chứng minh chịu được tải/sự cố thật. Đề xuất: load test theo tải đỉnh dự kiến, canary/rollout từng phần, có rollback + observability trước khi full production.

## ⑨ Bài tập + tiêu chí tự chấm
**BT1.** Lấy MỘT case study bất kỳ (Bài 9→15), đóng vai TL viết **design review** chỉ ra ≥4 điểm có thể vỡ ở production + cách đo từng điểm. *Đạt khi:* dùng được checklist tử huyệt, mỗi điểm có cách phát hiện/metric.
**BT2.** Viết phản biện 1 trang cho đề xuất "fine-tune LLM cho FAQ sản phẩm cập nhật hàng tuần". *Đạt khi:* nêu đúng bản chất (tri thức động), đề xuất RAG, so chi phí cập nhật & khả năng trích nguồn, nêu khi nào fine-tune mới hợp.
**BT3.** Tự ráp **"thẻ phán đoán"**: 10 tử huyệt production, mỗi cái 1 dòng "dấu hiệu sớm + cách phát hiện". *Đạt khi:* phủ hot key, cold start, stampede, pool, partition, retry storm, cost, observability, rollback, clock skew.

## ⑩ Đọc thêm
- *The Pragmatic Programmer* — tư duy phản biện & "đừng lập trình theo trùng hợp".
- Google SRE Book — *observability, rollout, postmortem culture*; Netflix — *chaos engineering*.
- AWS/Microsoft Well-Architected Framework — 5 trụ (vận hành, bảo mật, độ tin cậy, hiệu năng, **chi phí**) như một checklist review.
- *RAG vs fine-tuning* — tài liệu giới thiệu khi nào dùng cái nào (tri thức động vs phong cách).

---

## 🎓 Kết mục A — System Design & Architecture

Bạn vừa đi hết **16 bài**, dựng ngược từ **89 câu** trong ngân hàng đề, phủ đủ 16 mục con từ quy trình → ước lượng → scaling primitive → đánh đổi → 7 case study → phán đoán Tech Lead.

**Lộ trình học đề xuất (tuần tự, không nhảy cóc):**
1. **Khối I (Bài 1–2)** — nền tảng: quy trình & ước lượng. Học trước vì mọi bài sau đều giả định bạn biết "hỏi scope" và "ước lượng tải".
2. **Khối II (Bài 3–5)** — scaling primitives: LB, building blocks, bottleneck. Đây là "linh kiện" lắp vào mọi case study.
3. **Khối III (Bài 6–8)** — đánh đổi & ranh giới: CAP/consistency, decomposition, serverless. Học cách *chọn*, không chỉ *biết*.
4. **Khối IV (Bài 9–15)** — 7 case study: URL shortener → rate limiter → feed → video → chat → notification → checkout. Mỗi bài ráp lại linh kiện ở Khối II–III.
5. **Khối V (Bài 16)** — đổi vai thành Tech Lead: phản biện lựa chọn & dự đoán điểm vỡ. Học cuối vì nó *soi lại* toàn bộ 15 bài.

**Câu thần chú mang theo vào phòng phỏng vấn:**
> *"Tôi không nhớ để gõ, tôi hiểu để chỉ huy — và tra cứu phần còn lại."*

*Biên soạn: 18/06/2026. Nội dung kiến trúc là pattern phổ biến, ổn định; vài con số latency (Bài 2) và chi tiết serverless (Bài 8) đã được verify và chú nguồn; các tham chiếu bigtech nêu ở mức pattern, không khẳng định chi tiết nội bộ.*
