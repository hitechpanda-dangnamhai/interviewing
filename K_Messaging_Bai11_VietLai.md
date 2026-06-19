# 🎯 BÀI 11 — API Gateway, Service Mesh & Service Discovery

**Phủ:** K-GW-001 → 008

> 💡 **Mục tiêu một câu:** Sau khi đã *tách* service (Bài 10), bài này lo **topology nối chúng lại** — client vào hệ thống qua đâu (**gateway**), service tìm nhau thế nào (**discovery**), và giao tiếp nội bộ an toàn/quan sát được nhờ gì (**service mesh**).

---

## ① Mục tiêu & vị trí trong mạch

Bài 10 *chia* hệ thống thành nhiều service tự trị. Nhưng chia xong thì sinh ra ba câu hỏi topology:
- **Client vào bằng cửa nào?** → **API Gateway**.
- **Service tìm địa chỉ nhau ra sao** khi IP đổi liên tục? → **Service Discovery**.
- **Giao tiếp nội bộ** an toàn + quan sát được nhờ gì? → **Service Mesh**.

> 🔍 Bài này nhìn **gateway ở góc topology** (không lặp lại **F-TL** về *"god component"*). Cross-ref **F** (gateway tổng quát), **M** (**mTLS/zero-trust**), **K-RESIL** (**breaker** ở hạ tầng).

---

## ② Giảng cơ bản → nâng cao

### (a) Service discovery 📘 K-GW-001

> 📇 **Ví dụ trực giác:** số điện thoại của bạn bè *đổi liên tục*. Bạn không khắc cứng số vào tường — bạn tra **danh bạ (registry)**. Container lên/xuống → **IP đổi động** → không thể **hardcode địa chỉ**.

Instance **đăng ký** vào **registry** (**Consul/Eureka/etcd**...), bên gọi **tra cứu**:

| Kiểu | Ai hỏi registry | Đặc điểm |
|---|---|---|
| **Client-side discovery** | **client tự hỏi** rồi tự chọn instance | client **tự load-balance** |
| **Server-side discovery** | **gateway/LB hỏi giúp** | client chỉ gọi **một địa chỉ ổn định** |

> 🔎 *verify:* tên công cụ; trong **Kubernetes**, **DNS + Service object** làm phần lớn việc này.

---

### (b) API Gateway — góc topology 📘 K-GW-002

> 🚪 **Ẩn dụ:** gateway là **cửa trước/lễ tân** của toà nhà — mọi khách (client) vào đều qua đây, được chỉ phòng, kiểm tra giấy tờ, gộp giúp vài việc.

Điểm vào **north-south** (client ↔ hệ thống):
- **routing** tới service,
- **aggregation** (gộp nhiều call thành một response),
- **auth / rate-limit / TLS** tập trung,
- **che cấu trúc nội bộ** khỏi client.

> ⚠️ Cross-ref **F-TL** cho cảnh báo **gateway phình to** (đừng nhồi **business logic** vào). **Trọng tâm của Mục K: vị trí gateway trong sơ đồ**, không phải chi tiết tính năng.

---

### (c) North-south vs east-west 📘 K-GW-003

> 🧭 **Ẩn dụ địa lý:** **bắc-nam** = trục *vào/ra* toà nhà (khách ngoài). **đông-tây** = đi *giữa các phòng* trong nội bộ.

| Trục | Nghĩa | Ai lo |
|---|---|---|
| **North-south** | **client ↔ hệ thống** (biên ngoài) | **API gateway** |
| **East-west** | **service ↔ service** nội bộ | **service mesh** (hoặc gọi trực tiếp): **mTLS, retry, timeout, traffic shaping** |

> 🎯 **Phân vai rõ:** **gateway biên ngoài, mesh nội bộ.**

---

### (d) Service mesh 📘 K-GW-004

> 🛰️ **Ẩn dụ — phiên dịch viên đi kèm:** mỗi nhân viên (service) có một **trợ lý riêng (sidecar proxy)** đi kèm, lo hết chuyện *bảo mật, gọi lại khi rớt, ghi nhật ký* — bản thân nhân viên không phải học mấy việc đó.

**Service mesh** = lớp hạ tầng cho giao tiếp **service-to-service**.

- **Mô hình sidecar:** một **proxy** (**Envoy**) chạy *cạnh mỗi pod*, lo **mTLS, retry, timeout, circuit breaking, traffic split, telemetry** — **không cần sửa code** service.
- **Control plane** (vd **Istiod**) cấu hình các proxy.

> ⚖️ **Đổi lại:** **độ phức tạp vận hành** + **latency proxy** (mỗi call đi qua thêm 1–2 hop proxy).

---

### (e) BFF — Backend for Frontend 📘 K-GW-005

> 📱💻 **Ví dụ trực giác:** app **mobile** cần payload *gọn* (mạng yếu, màn nhỏ); **web** cần payload *đầy đủ*. Một gateway "one-size-fits-all" sẽ hoặc thừa cho mobile hoặc thiếu cho web.

**BFF** = gateway/aggregation **riêng cho từng loại client** (**web/mobile/3rd-party**) → tối ưu **payload** & **số call** theo nhu cầu client; mỗi đội frontend **sở hữu BFF của mình**.

> ⚖️ **Trade-off:** **trùng lặp logic** giữa các BFF.

---

### (f) mTLS & zero-trust nội bộ 📘 K-GW-006

> 🔐 **Ẩn dụ:** **zero-trust** = *"không tin ai chỉ vì họ đã ở trong toà nhà"*. Mọi người gặp nhau đều phải *xuất trình thẻ hai chiều*.

**mTLS** = xác thực & mã hoá **hai chiều** giữa service → **không tin mạng nội bộ mặc định** (**zero-trust**); mesh **tự cấp/luân chuyển cert** + áp mTLS **không cần sửa app**.

> 🎯 **Vì sao quan trọng:** **chống lateral movement** — khi 1 service bị chiếm, kẻ tấn công *không* tự do nhảy sang service khác. (cross-ref **M**.)

---

### (g) Breaker/retry ở hạ tầng — lợi & bất ngờ 📘 K-GW-007

| | Chi tiết |
|---|---|
| **Lợi** | đồng bộ **policy resilience** không sửa code |
| **Bất ngờ 1** | **retry ở mesh + retry ở app = nhân số lần gọi** (**retry amplification**) |
| **Bất ngờ 2** | **breaker mesh trip** mà app **không biết** → hành vi khó hiểu |

```
RETRY AMPLIFICATION — vì sao chồng retry giết downstream:
  app retry 2 lần × mesh retry 2 lần = 2 × 2 = 4 lần gọi cho MỖI request lỗi
  1000 request lỗi → 4000 lần gọi dội vào downstream đang ngộp ❌
```

> 🎯 **Cần phối hợp policy app↔mesh, tránh chồng retry — chỉ retry ở MỘT tầng.** (cross-ref **K-RESIL-005**.)

---

### (h) Mesh không miễn phí 📘 K-GW-008

> 🚨 **Mesh mạnh nhưng đắt:**
> - thêm **sidecar** (**latency**, **RAM/CPU** mỗi pod),
> - **control plane** phải vận hành,
> - **đường cong học dốc**,
> - **debug khó** hơn.

> 🎯 **Chỉ đáng khi:** **nhiều service** + cần **mTLS/observability/traffic-mgmt** thống nhất. **Hệ nhỏ → thư viện resilience** (vd **opossum**) **+ gateway là đủ**. (🔎 *verify*.)

---

## ③ ⚠️ Cũ → Mới (cập nhật quan trọng 2025–2026)

| Cái cũ | Nay | Vì sao |
|---|---|---|
| **Hardcode IP/host** downstream | **Service discovery** (registry/DNS); trong k8s: **Service + DNS** | IP động khi scale |
| **Sidecar mesh** (Envoy mỗi pod) là mặc định | **Istio ambient** (GA 2025): **ztunnel** per-node L4 + **waypoint** per-namespace L7; **Cilium eBPF** (kernel); **Linkerd** giữ **sidecar Rust** nhẹ | Sidecar tốn RAM/latency mỗi pod → *"kỷ nguyên sidecar đang kết thúc"* (🔎 verify) |
| **Một gateway dùng chung** mọi client | **BFF per client** khi nhu cầu khác nhau | Tránh gateway phình + payload thừa |
| **Mạng nội bộ "tin được"** | **Zero-trust + mTLS** (mesh tự áp) | Chống **lateral movement** |
| Nhồi **resilience/auth** vào code mỗi service | Đẩy lên **mesh/gateway** (cẩn thận **retry amplification**) | Đồng bộ policy (🔎 verify) |

> 📌 **Lưu ý cập nhật:** nếu đọc bài *"Istio vs Linkerd"* trước **giữa-2025**, nó **bỏ qua ambient mode** — thay đổi lớn nhất gần đây. Khi học **verify**: **ztunnel** (L4 mTLS, DaemonSet) + **waypoint** (L7 Envoy, optional). **Linkerd** đổi mô hình **license** (Buoyant subscription cho stable). **Cilium** đẩy mesh vào **kernel** qua **eBPF** (không sidecar). (🔎 *verify từng cái*.)

---

## ④ Áp dụng thực tế + bigtech

**Use case:** app **mobile + web** gọi qua **2 BFF riêng** → BFF route vào các service nội bộ; nội bộ chạy **mesh** áp **mTLS + telemetry** tự động; **service discovery** qua **k8s DNS**.

```
SƠ ĐỒ TOPOLOGY:
  [mobile] ──► BFF-mobile ┐
                          ├──► [mesh: mTLS + telemetry] ──► Catalog/Order/Payment...
  [web]    ──► BFF-web    ┘            (east-west)
       (north-south)                discovery qua k8s DNS
```

**Bigtech:** **Istio/Linkerd** phổ biến ở hệ **k8s** lớn; nhiều hệ đang dịch sang **ambient/eBPF** để giảm chi phí sidecar; **BFF** do **Netflix/SoundCloud** phổ biến hoá. (pattern; 🔎 *verify số liệu/đời*.)

---

## ⑤ Code thực hành + cấu hình

> 💡 **Mesh chủ yếu là cấu hình hạ tầng, không phải code app.** Ví dụ ý niệm (🔎 *verify CRD theo đời Istio*):

```yaml
# (ý niệm) bật mTLS strict cho namespace qua mesh — KHÔNG sửa code service
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default, namespace: payments }
spec: { mtls: { mode: STRICT } }   # mọi giao tiếp east-west trong ns này PHẢI mTLS // verify apiVersion
```

```yaml
# (ý niệm) retry ở mesh — CẢNH BÁO: nếu app cũng retry -> amplification
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: catalog }
spec:
  http:
    - route: [{ destination: { host: catalog } }]
      retries: { attempts: 2, perTryTimeout: 1s }  # phối hợp với app: chỉ retry ở MỘT tầng // verify
```

```bash
# Ambient mode (ý niệm) — KHÔNG sidecar, dùng ztunnel/waypoint // verify lệnh theo đời
istioctl install --set profile=ambient
kubectl label namespace payments istio.io/dataplane-mode=ambient
```

> 🚨 **AI hay sai:**
> 1. bật **retry** ở **cả mesh và code** → **2×2 = 4 lần gọi** khi lỗi (**amplification** dìm downstream);
> 2. khuyên dựng **full Istio cho hệ 3 service** (**over-engineering** — dùng **opossum + 1 gateway** là đủ).

---

## ⑥ Keywords

**🧠 Cần HIỂU:**
- **discovery** (**client** vs **server-side**).
- **north-south** vs **east-west**.
- **sidecar** vs **ambient/eBPF**.
- **BFF**; **zero-trust/mTLS**.
- **retry amplification**; **chi phí mesh**.

**📌 Cần THUỘC:**
- **gateway = north-south**; **mesh = east-west**.
- **sidecar (Envoy)** lo **cross-cutting**.
- **ztunnel(L4) + waypoint(L7)**.
- **mesh chỉ đáng khi nhiều service**.

**🛠️ Cần LÀM ĐƯỢC:**
- quyết **có cần mesh không**.
- phân vai **gateway/mesh**.
- tránh **chồng retry app↔mesh**.

---

## ⑦ Mental model

> 🧠 **"Gateway = cửa trước (north-south), mesh = đường nội bộ (east-west, mTLS/retry/telemetry không sửa code).**
> **Discovery thay hardcode IP.**
> **Mesh mạnh nhưng đắt — kỷ nguyên sidecar đang nhường chỗ ambient/eBPF.**
> **Hệ nhỏ: thư viện + gateway là đủ."**

---

## ⑧ Phỏng vấn & thách đố

**(TB)** **Service discovery** giải gì? **Client** vs **server-side**?
→ giải IP động; client tự hỏi registry + tự LB / gateway hỏi giúp, client gọi 1 địa chỉ ổn định.

**(khó)** **North-south** vs **east-west** — gateway lo gì, mesh lo gì?
→ gateway lo biên ngoài (client↔hệ thống); mesh lo nội bộ (service↔service).

**(khó)** **Service mesh** & **sidecar** làm gì? Vì sao tách **cross-cutting** khỏi code?
→ proxy cạnh pod lo mTLS/retry/timeout/telemetry không sửa code → đồng bộ policy, tách mối quan tâm.

**(khó)** **BFF** là gì, khi nào cần?
→ gateway riêng từng loại client; cần khi nhu cầu payload/call khác nhau rõ rệt.

**(rất khó)** **Mesh không miễn phí** — chi phí gì khiến không phải hệ nào cũng nên dùng?
→ sidecar latency/RAM, control plane vận hành, học dốc, debug khó.

> 🧩 **Thách đố:** *"Sau khi cài **Istio**, **p99 latency tăng** và thỉnh thoảng downstream bị **dìm** dù traffic không tăng. Hai nghi phạm?"*
> → **(1)** **sidecar overhead** (Envoy mỗi pod thêm latency/RAM — cân nhắc **ambient/eBPF**); **(2)** **retry amplification** (mesh retry + app retry chồng nhau → nhân số request lên downstream). **Sửa:** gỡ retry ở **một tầng**; cân nhắc **ambient mode**.

---

## ⑨ Bài tập + tiêu chí

**Đề:** Hệ **4 service** trên **k8s** cần: **mTLS nội bộ**, **telemetry**, **web+mobile payload khác nhau**. Đề xuất: **discovery**, **gateway/BFF**, **có nên mesh không** (và **mode** nào).

> ✅ **Đạt khi:**
> - **discovery** qua **k8s DNS/Service**.
> - **BFF** riêng **web/mobile**.
> - cân nhắc **mesh** cho **mTLS+telemetry** nhưng **nêu chi phí** — gợi ý **ambient** để giảm overhead với hệ vừa, hoặc **thư viện resilience** nếu chỉ 4 service.
> - **tránh chồng retry**.

---

## ⑩ Đọc thêm

- **`istio.io`** (**Ambient mesh, dataplane modes**), **`linkerd.io`**, **`cilium.io`**.
- **Sam Newman** về **BFF/gateway**. (🔎 *verify đời — đổi rất nhanh*.)

> 🎯 **Một câu mang về:** *Gateway giữ cửa trước, mesh trải đường nội bộ — nhưng cả hai chỉ đáng tiền khi hệ đủ lớn. Với vài service, một **thư viện resilience + một gateway** đã đủ; đừng kéo cả **Istio** vào để rồi trả giá bằng latency và đêm mất ngủ debug sidecar.*
