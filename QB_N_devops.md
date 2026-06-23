# ⚙️ QB_N — Ngân hàng câu hỏi: DevOps / Docker / Kubernetes / CI-CD

> **Mục N** trong roadmap Tech Lead Backend (🟢 ~30% — nhưng là "đủ để bàn deploy/scale trong system design", và lên cao thì hỏi sâu hơn) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 8 mục con của roadmap (Docker cơ bản · Kubernetes cơ bản · CI/CD pipeline · Deploy strategy · Terraform · Helm · GitOps ArgoCD/Flux · Cloud services EKS/RDS/S3/CloudFront) + lớp **TL-level** (khi nào KHÔNG dùng K8s, cost control, secrets ở scale, image supply chain, DR/multi-region, platform engineering) mà phỏng vấn Tech Lead hay vặn.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 98 câu.**

## 🚧 Chống trùng (đã đọc `QB_G_nodejs.md`, `QB_H_nestjs.md`, `QB_E_database.md`, `QB_F_apidesign.md`, `QB_I_concurrency.md`, `QB_J_caching.md`, `QB_K_messaging.md`, `QB_L_testing.md`, `QB_M_security.md`, `QB_Q_designpatterns.md`)

DevOps đụng nhiều mục. Quy ước phân ranh để **không lặp**:

- **`G-DIAG` (Node graceful shutdown — G-DIAG-009)** đã phủ *cơ chế trong Node*: bắt SIGTERM/SIGINT, drain in-flight, đóng DB/queue, timeout cưỡng bức. Và **G cluster (G-226/229/230)** đã phủ *cluster vs để K8s scale pod*.
  → **N KHÔNG lặp cơ chế Node.** N làm phần **K8s-side**: `preStop` hook, `terminationGracePeriodSeconds`, **race giữa SIGTERM và việc xoá endpoint** (N-K8SOPS-011), và PID 1 trong container (N-DKR-010). Khi đụng cơ chế process → trỏ về G.
- **`H-399` (`enableShutdownHooks()`+SIGTERM trong Nest)** đã phủ phía framework. → N cross-ref, không lặp.
- **`K` (Messaging/Microservices)** sở hữu: định nghĩa microservice ("không phải vì dùng Docker" — K-367), **service mesh / Istio / sidecar proxy** (K-419/436), independent deploy, distributed monolith.
  → **N KHÔNG lặp service mesh.** N chỉ chạm K8s networking primitive (Service/Ingress/Gateway API) và **native sidecar container GA của K8s** (N-K8SOPS-013) ở góc *cơ chế lifecycle pod*, rồi trỏ K cho mesh.
- **`L` (Testing)** sở hữu *bản chất* smoke/regression/contract/performance test và "can-i-deploy". → N chỉ làm **vị trí của các gate đó trong pipeline** (N-CICD-006), không lặp "test đó là gì".
- **`I` (Concurrency)** sở hữu *khái niệm* leader election / "cron chạy 1 lần dù nhiều instance" (I-170) và **etcd/Raft** (I-345). → N chỉ trỏ tới khi nói K8s `CronJob` (N-K8SOPS-012) và etcd (N-K8S-012).
- **`M` (Security)** sở hữu chiều sâu **secrets management mức kiến trúc** (M-SECRET) và **supply chain SBOM/SLSA/provenance** (M-SUPPLY). → N lấy *góc DevOps*: secret trong container/K8s vận hành thế nào, image scanning trong pipeline, image signing ở admission — và trỏ M cho threat model.
- **`F` (API Design)** giữ versioning/backward-compat *ở góc API*. → N dùng ở góc *deploy* (rolling cần API backward-compatible — N-DEPLOY-008), cross-ref F.
- **`O` (Observability)** là **mục riêng, CHƯA có file QB.** → N **không** đào Prometheus/Grafana/tracing/logging. N chỉ chạm health probe (primitive của K8s) và "HPA cần metrics"; phần monitoring sâu để dành cho O.
- **`J` (Caching)** giữ CDN ở góc *cache strategy*. → N chỉ nhắc CloudFront như *managed CDN/edge* (N-CLOUD-004), trỏ J cho invalidation/TTL.

---

## Cách đọc
- **ID:** `N-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (trade-off, vận hành production, debug, governance).
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm live ở Bước D).

## Bối cảnh phiên bản & landscape (tính tại 6/2026 — *verify lại khi học*)

> ⚠️ Phần này đổi nhanh → mọi câu chạm tên/đời phiên bản đều nên kiểm chứng tại docs chính thức (kubernetes.io, docs.docker.com, developer.hashicorp.com, argo-cd.readthedocs.io) trước khi tin.

- **Dockershim đã bị gỡ khỏi Kubernetes từ v1.24** (công bố ở v1.20). Câu "K8s dùng Docker làm runtime" là **lỗi thời** — K8s nói chuyện với runtime qua **CRI** (containerd, CRI-O); image build bằng Docker (OCI) vẫn chạy bình thường. *(verify tại kubernetes.io)*
- **Native sidecar containers GA** (`initContainers` có `restartPolicy: Always`) — feature gate bật mặc định từ **v1.29**, **stable từ v1.33**. Trước đó phải "chế" sidecar bằng container thường → vấn đề thứ tự khởi động/tắt. *(verify)*
- **In-Place Pod Resize** đã **GA** (đổi requests/limits không cần restart pod) — đổi cách right-size tài nguyên. *(verify đời K8s)*
- **Ingress NGINX (upstream) bị khai tử (~3/2026)** → hệ sinh thái dịch về **Gateway API** (đã GA) hoặc ingress controller bên thứ ba. Câu hỏi Ingress nay nên biết Gateway API là kế thừa. *(verify)*
- **Terraform đổi license sang BSL (8/2023)** → cộng đồng fork thành **OpenTofu** (Linux Foundation), drop-in tương thích phần lớn. Nhắc tên IaC nên biết cặp **Terraform / OpenTofu**. *(verify)*
- K8s đang ở vùng **~1.33–1.36** (tùy cloud; EKS/GKE trễ hơn upstream). Đừng khẳng định version cụ thể mà không tra.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **N-DKR** | Docker fundamentals (container vs VM, image/layer, multi-stage, Dockerfile, PID 1) | 11 |
| **N-DKRADV** | Docker runtime & build sâu (namespaces/cgroups, security, network, volume, registry, scan) | 8 |
| **N-K8S** | Kubernetes core (architecture, Pod/Deployment/Service, ConfigMap/Secret, reconcile, etcd) | 12 |
| **N-K8SOPS** | K8s operations (probes, requests/limits/QoS, HPA, rolling, PDB, workload types, scheduling, Ingress, graceful, debug) | 15 |
| **N-CICD** | CI/CD pipeline (stages, artifact promotion, secrets/OIDC, gates, mono/polyrepo, DORA) | 10 |
| **N-DEPLOY** | Deployment strategies (rolling/blue-green/canary, rollback, DB migration, feature flag, zero-downtime) | 10 |
| **N-IAC** | Infrastructure as Code (Terraform state/drift/plan, declarative vs imperative, modules, OpenTofu) | 8 |
| **N-HELM** | Packaging (Helm chart/values/release, upgrade/rollback, vs Kustomize) | 5 |
| **N-GITOPS** | GitOps (pull vs push, ArgoCD/Flux, drift/self-heal, image→deploy flow) | 6 |
| **N-CLOUD** | Managed cloud services (EKS/GKE, RDS, S3, CDN, IRSA/workload identity, cost) | 7 |
| **N-TL** | TL-level (khi nào KHÔNG dùng K8s, cost control, secrets ở scale, image signing, DR/multi-region, platform eng) | 6 |

---

## N-DKR — Docker fundamentals

**N-DKR-001** ⭐⭐
"Container khác Virtual Machine ở đâu? Vì sao container 'nhẹ' hơn — và đánh đổi gì về ranh giới cô lập/bảo mật?"
> *Dò cái gì:* container **chia sẻ kernel host** (chỉ là process bị giới hạn tầm nhìn qua namespaces + giới hạn tài nguyên qua cgroups), không có guest OS riêng → khởi động ms, image MB; VM có hypervisor + guest kernel đầy đủ → boundary mạnh hơn nhưng nặng (GB, giây). Trade-off: cùng kernel → cô lập yếu hơn VM (lỗ kernel = thoát container).

**N-DKR-002** ⭐⭐
"Phân biệt **image** và **container**. Vì sao chạy 10 container từ 1 image không tốn 10 lần dung lượng?"
> *Dò cái gì:* image = template read-only, layered; container = instance đang chạy = image + **một writable layer (copy-on-write)** trên cùng; nhiều container share layer read-only, mỗi cái chỉ thêm writable layer riêng → tiết kiệm.

**N-DKR-003** ⭐⭐⭐
"Layer caching của Docker hoạt động ra sao? Vì sao **thứ tự lệnh trong Dockerfile** quyết định build nhanh hay chậm? Cho ví dụ sắp xếp sai làm vỡ cache."
> *Dò cái gì:* mỗi lệnh = 1 layer cache theo nội dung; đổi 1 layer làm **invalidate mọi layer sau**; nên copy `package.json` + install deps TRƯỚC khi `COPY . .` → đổi source code không phải cài lại deps; sai: `COPY . .` trước `npm install` → mỗi lần sửa code đều reinstall.

**N-DKR-004** ⭐⭐⭐
"Multi-stage build giải quyết vấn đề gì? Cho một ví dụ (build tool/compiler không nên có trong image production)."
> *Dò cái gì:* tách stage build (có toolchain: compiler, devDeps) khỏi stage runtime (chỉ artifact + runtime) → image cuối nhỏ, ít attack surface, không lộ source/build secret; `COPY --from=builder` chỉ bê output sang.

**N-DKR-005** ⭐⭐⭐
"Liệt kê các cách giảm kích thước Docker image cho một app backend và cách nào tác động lớn nhất."
> *Dò cái gì:* base image nhỏ (slim/alpine/**distroless**), **multi-stage**, `.dockerignore`, gộp/giảm layer, dọn cache trong cùng layer, chỉ cài prod deps; lớn nhất thường là base image + multi-stage. Hiểu *vì sao* nhỏ tốt (pull nhanh, ít CVE, scale nhanh).

**N-DKR-006** ⭐⭐⭐
"`ENTRYPOINT` vs `CMD` khác nhau thế nào? Và vì sao **exec form** (`["node","app.js"]`) khác **shell form** (`node app.js`) về xử lý tín hiệu?"
> *Dò cái gì:* ENTRYPOINT = lệnh cố định, CMD = tham số mặc định (có thể override); exec form → process là **PID 1** nhận trực tiếp SIGTERM; shell form → bị bọc trong `/bin/sh -c` (sh là PID 1, không forward signal) → graceful shutdown vỡ.

**N-DKR-007** ⭐⭐
"`.dockerignore` để làm gì? Bỏ qua nó gây hậu quả gì ngoài 'image to'?"
> *Dò cái gì:* loại file khỏi **build context** gửi cho daemon; tránh copy `node_modules`/`.git`/secret/.env vào image → build nhanh hơn, image nhỏ, **không lộ secret/.git history**, không phá cache.

**N-DKR-008** ⭐⭐
"`COPY` vs `ADD` — khi nào dùng cái nào, và vì sao khuyến nghị mặc định là `COPY`?"
> *Dò cái gì:* COPY chỉ copy file/dir (rõ ràng, dự đoán được); ADD thêm 'magic' (giải nén tar, tải URL) → dễ gây bất ngờ/rủi ro; best practice: COPY mặc định, ADD chỉ khi cần giải nén tar local.

**N-DKR-009** ⭐⭐⭐⭐
"Viết/đánh giá một Dockerfile *production* cho app Node.js: anh để ý những gì (user, deps, cache, signal)?" *(cross-ref G cho phía process)*
> *Dò cái gì:* base pin theo digest, multi-stage, `npm ci --omit=dev` (deterministic từ lockfile), copy `package*.json` trước để cache, chạy **non-root `USER`**, exec-form ENTRYPOINT/PID-1 đúng, healthcheck, không bake secret. Hiểu *vì sao* mỗi mục chứ không chép template.

**N-DKR-010** ⭐⭐⭐⭐
"Vấn đề 'PID 1' trong container là gì? Vì sao app làm PID 1 có thể **không tắt sạch** và để lại zombie process? Cách xử lý?"
> *Dò cái gì:* PID 1 có ngữ nghĩa init đặc biệt (mặc định bỏ qua signal không có handler; phải reap zombie con); app không handle SIGTERM → `docker stop`/K8s SIGTERM bị nuốt → bị SIGKILL sau timeout (mất graceful drain); fix: handle signal trong app (trỏ G) hoặc init nhẹ (`tini`, `--init`). *(liên hệ N-K8SOPS-011)*

**N-DKR-011** ⭐⭐⭐⭐
"Alpine vs distroless vs slim — trade-off? Một 'bẫy' kinh điển khi đổi Node app sang Alpine là gì?"
> *Dò cái gì:* alpine cực nhỏ nhưng dùng **musl libc** (không phải glibc) → native module/DNS/locale có thể lệch; distroless = không shell/package manager (an toàn, khó debug); slim = glibc, cân bằng. Bẫy: native addon build cho glibc vỡ trên musl; cần test, không chọn theo 'nhỏ nhất thắng'.

---

## N-DKRADV — Docker runtime & build sâu

**N-DKRADV-001** ⭐⭐⭐
"Cụ thể **namespaces** và **cgroups** mỗi cái lo gì trong việc tạo nên một container?"
> *Dò cái gì:* namespaces **cô lập tầm nhìn** (PID, network, mount/filesystem, UTS, IPC, user) → container 'tưởng' nó cô đơn; cgroups **giới hạn & đo tài nguyên** (CPU, memory, I/O, pids); container = process + namespaces + cgroups, không phải 'máy ảo nhỏ'.

**N-DKRADV-002** ⭐⭐⭐⭐
"Hardening một container chạy production gồm những lớp gì? Vì sao 'chạy non-root' chưa đủ?"
> *Dò cái gì:* non-root `USER`, **drop capabilities** (`--cap-drop=ALL` + add tối thiểu), `--read-only` rootfs + tmpfs cho ghi tạm, `no-new-privileges`, seccomp/AppArmor profile, không mount docker socket, rootless runtime; non-root giảm nhưng không chặn lỗ kernel/escape → cần defense-in-depth. *(threat model sâu → M)*

**N-DKRADV-003** ⭐⭐⭐
"Các network mode của Docker (bridge/host/none/overlay) khác gì? Khi nào dùng host network và rủi ro?"
> *Dò cái gì:* bridge = mạng ảo NAT mặc định (cô lập, map port); host = dùng thẳng network stack host (nhanh, nhưng mất cô lập + đụng port); none = không network; overlay = multi-host (swarm/cluster). Host hợp khi cần hiệu năng/port động nhưng phá cô lập.

**N-DKRADV-004** ⭐⭐⭐
"Volume vs bind mount vs tmpfs — khác nhau và khi nào dùng? Dữ liệu container 'biến mất' sau restart là vì sao?"
> *Dò cái gì:* writable layer của container là ephemeral → mất khi xoá container; **volume** (Docker quản lý, portable, backup được) cho data bền; **bind mount** map path host (dev, cấu hình) — phụ thuộc host; **tmpfs** trong RAM (secret tạm, không ghi đĩa).

**N-DKRADV-005** ⭐⭐⭐
"BuildKit / build cache mount giúp gì so với build cũ? Làm sao cache `npm install`/`go mod` qua các lần build mà không bake vào layer?"
> *Dò cái gì:* BuildKit = build song song, cache thông minh, secret build-time không lưu layer; `--mount=type=cache` giữ thư mục cache deps giữa build mà không nằm trong image; `--mount=type=secret` để dùng secret lúc build mà không leak vào layer.

**N-DKRADV-006** ⭐⭐⭐
"Vì sao tag `latest` là anti-pattern ở production? 'Immutable tag' và **image digest** giải quyết gì? `imagePullPolicy` liên quan thế nào?"
> *Dò cái gì:* `latest` mơ hồ → mỗi node/lần pull có thể khác bản → không reproducible, rollback mù; nên tag bất biến (semver/commit-sha) hoặc pin theo **`@sha256:digest`**; `pullPolicy: Always` với latest gây drift, pin digest cho xác định.

**N-DKRADV-007** ⭐⭐⭐
"Tại sao truyền secret qua **ENV var** vào container bị coi là rủi ro? Lựa chọn an toàn hơn lúc build và lúc run?" *(kiến trúc secret → M-SECRET)*
> *Dò cái gì:* env bị lộ qua `docker inspect`, `/proc/<pid>/environ`, log, child process, crash dump; build secret bake vào layer là vĩnh viễn; thay bằng BuildKit `--mount=type=secret` lúc build, và file mount/secret manager (Docker/K8s Secret, Vault) lúc run. Trỏ M cho rotation/dynamic secret.

**N-DKRADV-008** ⭐⭐⭐⭐
"Quét bảo mật image (Trivy/Grype) đặt ở đâu trong vòng đời, và 'image sạch CVE hôm nay' có nghĩa nó an toàn mãi không?" *(supply chain sâu → M-SUPPLY)*
> *Dò cái gì:* scan ở CI (pre-push) + ở registry (định kỳ) + admission lúc deploy; CVE mới công bố liên tục → image cũ thành 'có lỗ' dù không đổi → cần rescan + rebuild định kỳ; base nhỏ (distroless) giảm bề mặt. Trỏ M cho SBOM/SLSA/provenance.

---

## N-K8S — Kubernetes core

**N-K8S-001** ⭐⭐
"`docker run` đã chạy được container, vậy Kubernetes giải quyết thêm vấn đề gì mà container runtime đơn lẻ không lo nổi?"
> *Dò cái gì:* chạy nhiều container trên nhiều node ở scale: **scheduling**, self-healing (restart/replace pod chết), **rolling update/rollback**, scaling (HPA), service discovery + load balancing, config/secret, declarative desired-state; không phải 'để cho oách' — là khi 1 host + tay không đủ.

**N-K8S-002** ⭐⭐⭐
"Các thành phần **control plane** gồm gì và mỗi cái làm gì (API server, etcd, scheduler, controller-manager)?"
> *Dò cái gì:* API server = cổng giao tiếp duy nhất (REST, validate, auth); **etcd** = kho lưu state cluster (source of truth); scheduler = chọn node cho pod chưa gán; controller-manager = các control loop đưa thực tế về desired. Hiểu vai trò, không học vẹt tên.

**N-K8S-003** ⭐⭐⭐
"Trên mỗi **worker node** có gì? kubelet và kube-proxy khác vai trò ra sao? Runtime nói chuyện với K8s qua đâu?"
> *Dò cái gì:* kubelet = agent đảm bảo container trong pod chạy đúng spec + báo cáo health; kube-proxy = lo networking Service (iptables/IPVS/eBPF); container runtime qua **CRI** (containerd/CRI-O) — *không còn dockershim*. *(verify)*

**N-K8S-004** ⭐⭐⭐
"Pod là gì và vì sao nó (chứ không phải container) là đơn vị nhỏ nhất? Vì sao **không** nên deploy 'bare pod' trực tiếp?"
> *Dò cái gì:* pod = 1+ container share **network namespace (cùng localhost/IP)** + storage + lifecycle; bare pod không tự hồi sinh khi chết/node mất → phải dùng controller (Deployment) để có self-healing + scaling + rollout.

**N-K8S-005** ⭐⭐⭐
"Quan hệ Pod ↔ ReplicaSet ↔ Deployment? Khi anh `kubectl apply` một Deployment mới image, chuyện gì xảy ra theo thứ tự?"
> *Dò cái gì:* Deployment quản lý ReplicaSet, ReplicaSet quản lý số pod; đổi image → Deployment tạo **ReplicaSet mới**, tăng dần pod mới/giảm pod cũ (rolling), giữ ReplicaSet cũ để rollback. Tầng controller, không tự tay quản pod.

**N-K8S-006** ⭐⭐⭐⭐
"Giải thích **reconciliation loop** / mô hình declarative của K8s. Vì sao 'khai báo desired state' khác hẳn 'chạy script imperative'?"
> *Dò cái gì:* mình khai báo *trạng thái mong muốn*; controller liên tục so sánh actual vs desired và **hành động để khép khoảng cách** (level-triggered, không edge-triggered); pod chết → tự thay; tự healing/idempotent; khác script chạy-một-lần rồi mặc kệ drift.

**N-K8S-007** ⭐⭐⭐
"Các loại Service (ClusterIP / NodePort / LoadBalancer / headless) khác gì và dùng khi nào?"
> *Dò cái gì:* ClusterIP = nội bộ cluster (mặc định); NodePort = mở port trên mọi node; LoadBalancer = xin LB cloud bên ngoài; **headless** (`clusterIP: None`) = trả thẳng IP pod (StatefulSet/discovery). Chọn theo ai gọi (trong/ngoài cluster).

**N-K8S-008** ⭐⭐⭐⭐
"Một **Service** định tuyến tới đúng các pod bằng cách nào khi pod sinh/diệt liên tục và IP đổi? Vai trò selector, EndpointSlice, kube-proxy?"
> *Dò cái gì:* Service có IP ảo ổn định; **label selector** → controller cập nhật danh sách pod khoẻ vào **Endpoints/EndpointSlice**; kube-proxy lập trình iptables/IPVS để LB tới các IP đó; pod đổi → endpoint cập nhật, client vẫn gọi 1 địa chỉ Service.

**N-K8S-009** ⭐⭐⭐
"ConfigMap vs Secret? Và 'cái bẫy' khiến nhiều người tưởng Secret được mã hoá nhưng thực ra không?"
> *Dò cái gì:* ConfigMap = config phi nhạy cảm; Secret = dữ liệu nhạy cảm, nhưng **mặc định chỉ base64-encode trong etcd (KHÔNG mã hoá)** → phải bật encryption-at-rest cho etcd + RBAC chặt + (tốt hơn) external secret manager; cả hai inject qua env hoặc volume mount. *(kiến trúc → M-SECRET)*

**N-K8S-010** ⭐⭐
"Namespace cô lập cái gì và **không** cô lập cái gì?"
> *Dò cái gì:* namespace = phân vùng *logic* tên/resource/quota/RBAC; **không** tự cô lập network (cần NetworkPolicy) hay node/kernel; nhiều namespace vẫn chung cluster/node → không phải biên giới bảo mật cứng.

**N-K8S-011** ⭐⭐⭐
"Label & selector — vì sao nói chúng là 'chất keo' nối gần như mọi thứ trong K8s? Cho 2 ví dụ cơ chế dựa hoàn toàn vào label."
> *Dò cái gì:* gán/chọn resource theo nhãn thay vì tên cứng → linh hoạt; ví dụ: Service chọn pod qua selector, Deployment quản pod qua matchLabels, node affinity/anti-affinity, network policy đều dựa label; đổi label = thay đổi cách wiring.

**N-K8S-012** ⭐⭐⭐⭐
"Vai trò của **etcd** với cluster? Mất etcd (hoặc split-brain) thì sao, và vì sao nó cần số node lẻ (quorum)?" *(cross-ref I-345 Raft)*
> *Dò cái gì:* etcd giữ toàn bộ state desired/actual = source of truth; mất etcd = mất 'bộ não' (không apply/đọc state được, cluster đóng băng dù pod còn chạy); dùng Raft → cần quorum (đa số) → số lẻ (3/5) để chịu lỗi mà vẫn bầu leader. Trỏ I cho Raft/consensus.

---

## N-K8SOPS — Kubernetes operations

**N-K8SOPS-001** ⭐⭐⭐⭐
"Phân biệt **liveness / readiness / startup probe**. Cho một ví dụ mỗi loại bị cấu hình sai gây hậu quả gì ở production."
> *Dò cái gì:* liveness = 'còn sống không' → fail thì **restart** pod; readiness = 'sẵn sàng nhận traffic chưa' → fail thì **gỡ khỏi Service** (không restart); startup = che giai đoạn khởi động chậm khỏi liveness; sai: liveness quá nhạy → restart loop; thiếu readiness → traffic vào pod chưa sẵn sàng → lỗi; liveness == readiness logic → restart khi chỉ cần ngừng nhận traffic.

**N-K8SOPS-002** ⭐⭐⭐⭐
"**Requests vs limits** dùng cho việc gì khác nhau? Điều gì xảy ra khi pod vượt memory limit vs vượt CPU limit? QoS class ảnh hưởng gì?"
> *Dò cái gì:* **request** = scheduler dùng để xếp pod lên node (đảm bảo tối thiểu); **limit** = trần runtime; vượt memory → **OOMKilled**; vượt CPU → **throttle** (chậm, không chết); request=limit (memory) → QoS **Guaranteed** (ít bị evict); chỉ có limit hoặc lệch → Burstable/BestEffort dễ bị evict trước. Best practice: set memory limit≈request, CPU cho phép burst.

**N-K8SOPS-003** ⭐⭐⭐⭐
"Pod liên tục bị **OOMKilled**. Quy trình debug và các nguyên nhân gốc anh nghĩ tới?"
> *Dò cái gì:* xem `kubectl describe`/last state reason=OOMKilled + exit 137; phân biệt limit quá thấp vs memory leak vs spike thật vs heap setting (vd Node `--max-old-space-size` không khớp limit); fix: đo thực tế, set limit hợp lý, sửa leak, để runtime nhận biết cgroup memory; không chỉ 'tăng limit' mù.

**N-K8SOPS-004** ⭐⭐⭐⭐
"HPA scale dựa trên gì và cần điều kiện gì để hoạt động? Vì sao HPA theo CPU đôi khi không phản ánh đúng tải thật?"
> *Dò cái gì:* HPA tăng/giảm replica theo metric (CPU/memory/custom) so target; cần **metrics-server** + pod phải có **resource requests** (tính % theo request); CPU không bắt được bottleneck I/O/queue → dùng custom/external metric (độ dài queue, latency, RPS); có cool-down tránh flapping. *(HPA cần metrics → liên hệ O cho nguồn metric)*

**N-K8SOPS-005** ⭐⭐⭐⭐
"Rolling update đạt **zero-downtime** nhờ những tham số/cơ chế nào (`maxSurge`, `maxUnavailable`, readiness)? Đặt sai gây gì?"
> *Dò cái gì:* maxSurge = số pod thêm tạm; maxUnavailable = số pod được phép thiếu; rollout chỉ chuyển traffic khi pod mới **readiness pass**; sai: maxUnavailable cao + không readiness → traffic vào pod chưa sẵn sàng → downtime; surge 0 + unavailable 0 → kẹt. Phối với graceful shutdown (N-K8SOPS-011).

**N-K8SOPS-006** ⭐⭐⭐⭐
"**PodDisruptionBudget** giải quyết vấn đề gì? Phân biệt voluntary vs involuntary disruption."
> *Dò cái gì:* PDB giữ tối thiểu N pod sẵn sàng khi có **voluntary disruption** (drain node để nâng cấp, autoscaler scale-down) → tránh gỡ hết replica cùng lúc; involuntary (node crash, OOM) PDB không chặn được. Quan trọng khi cluster tự upgrade/rebalance.

**N-K8SOPS-007** ⭐⭐⭐⭐
"Khi nào dùng Deployment vs **StatefulSet** vs **DaemonSet** vs **Job/CronJob**? Cho mỗi loại một workload điển hình."
> *Dò cái gì:* Deployment = stateless thay thế tự do (API service); StatefulSet = cần định danh ổn định + storage bền + thứ tự (DB, Kafka, Zookeeper); DaemonSet = 1 pod/node (log/metric agent, CNI); Job = chạy tới hoàn thành; CronJob = Job theo lịch. Chọn theo *bản chất state/đặc tính*, không mặc định Deployment.

**N-K8SOPS-008** ⭐⭐⭐⭐
"StatefulSet cung cấp gì mà Deployment không (định danh ổn định, ordered, storage)? Vì sao chạy database trên K8s lại 'khó' hơn stateless?"
> *Dò cái gì:* pod tên ổn định `app-0,app-1` + DNS riêng, PVC gắn bền qua restart, tạo/scale/update **theo thứ tự**; DB cần data persistence + cluster membership + ordered bootstrap + failover → vận hành phức tạp hơn stateless (thường cân nhắc managed DB/operator).

**N-K8SOPS-009** ⭐⭐⭐⭐
"Taints/tolerations và node affinity/anti-affinity dùng để làm gì? Cho ví dụ tách workload nhạy cảm khỏi workload thường."
> *Dò cái gì:* **taint** đẩy pod ra khỏi node trừ khi pod có **toleration** (vd node GPU/dedicated chỉ nhận pod được phép); **affinity** kéo pod tới node có nhãn nhất định; **pod anti-affinity** rải replica ra khác node/zone để chịu lỗi. Phân biệt 'đẩy' (taint) vs 'hút' (affinity).

**N-K8SOPS-010** ⭐⭐⭐⭐
"Service vs **Ingress** khác vai trò gì? Ingress controller là gì, và một thay đổi landscape gần đây anh nên biết (Ingress NGINX/Gateway API)?"
> *Dò cái gì:* Service = L4 trong cluster; Ingress = L7 routing (host/path) + TLS termination từ ngoài vào, cần **Ingress controller** (Nginx/Traefik...) hiện thực; nên biết **Ingress NGINX upstream bị khai tử (~3/2026)** → dịch sang **Gateway API** (kế thừa, biểu đạt mạnh hơn). *(verify)*

**N-K8SOPS-011** ⭐⭐⭐⭐⭐
"Mô tả vòng đời tắt pod khi rolling update: SIGTERM, `preStop`, `terminationGracePeriodSeconds`. Vì sao có **race** giữa 'pod nhận SIGTERM' và 'bị gỡ khỏi endpoint', và làm sao tránh rớt request?" *(cơ chế trong app → G-DIAG-009 / H-399)*
> *Dò cái gì:* khi xoá pod, K8s **song song** gửi SIGTERM và cập nhật endpoint — không đảm bảo thứ tự → kube-proxy/LB có thể còn gửi traffic tới pod đang tắt; thêm `preStop` sleep nhỏ (hoặc readiness fail trước) để chờ endpoint rút, rồi mới drain; app phải bắt SIGTERM drain in-flight (trỏ G); grace period đủ dài hơn drain, nếu không bị SIGKILL.

**N-K8SOPS-012** ⭐⭐⭐
"K8s `CronJob` chạy trên cluster nhiều node — làm sao đảm bảo một job **chỉ chạy một lần**, không double-run?" *(khái niệm dedup/leader → I-170)*
> *Dò cái gì:* CronJob tạo Job → Job tạo pod; cần để ý `concurrencyPolicy` (Forbid/Replace), idempotency của job, và nếu logic cron tự viết trong app nhiều replica thì cần leader election/lock (trỏ I), **không giả định 'chỉ 1 instance'**.

**N-K8SOPS-013** ⭐⭐⭐⭐
"**Native sidecar container** (GA) khác container thường trong pod thế nào, và nó sửa vấn đề gì về thứ tự khởi động/tắt?" *(service mesh/sidecar proxy → K-419)*
> *Dò cái gì:* khai báo ở `initContainers` với `restartPolicy: Always` → **khởi động trước app và sống suốt đời pod**, tắt sau app; trước GA phải 'chế' bằng container thường → app có thể start khi sidecar chưa sẵn sàng, hoặc Job không kết thúc vì sidecar mãi chạy; GA từ ~v1.29, stable v1.33. *(verify)* Trỏ K cho mesh.

**N-K8SOPS-014** ⭐⭐⭐⭐
"Pod ở trạng thái `Pending` / `CrashLoopBackOff` / `ImagePullBackOff` — mỗi cái gợi nguyên nhân gì và anh debug từ đâu?"
> *Dò cái gì:* Pending = không scheduler được (thiếu tài nguyên/affinity/taint/PVC chưa bind) → `describe` xem events; CrashLoopBackOff = container start rồi chết lặp (lỗi app/config/probe) → xem `logs`/`logs --previous` + exit code; ImagePullBackOff = sai tag/registry/credential → kiểm tra image ref + imagePullSecret. Quy trình: describe → events → logs.

**N-K8SOPS-015** ⭐⭐⭐⭐
"Right-sizing tài nguyên pod: trước đây đổi requests/limits phải làm gì, và **In-Place Pod Resize** (GA) đổi điều đó ra sao? Rủi ro đặt request quá cao vs quá thấp?"
> *Dò cái gì:* trước đây sửa resource → tạo pod mới (restart); **in-place resize** cho đổi mà không restart (verify đời K8s); request quá cao = lãng phí + giảm mật độ/đắt; quá thấp = bị xếp lên node quá tải, evict/throttle; cần đo (VPA/quan sát) để chỉnh.

---

## N-CICD — CI/CD pipeline

**N-CICD-001** ⭐⭐
"Phân biệt CI, Continuous **Delivery** và Continuous **Deployment**. Ranh giới giữa delivery và deployment nằm ở đâu?"
> *Dò cái gì:* CI = merge thường xuyên + tự build/test mỗi thay đổi; Delivery = luôn ở trạng thái *deploy được*, nhưng **bấm nút thủ công** để lên prod; Deployment = tự động lên prod nếu qua gate (không cần người duyệt). Khác nhau ở 'ai bấm nút cuối'.

**N-CICD-002** ⭐⭐⭐
"Một pipeline backend điển hình gồm các stage gì, theo thứ tự nào và vì sao thứ tự đó (fail nhanh)?"
> *Dò cái gì:* lint/format → build → unit test → (security/dependency scan) → build image → integration test → push registry → deploy (staging) → smoke/e2e → promote prod; xếp khâu **rẻ-và-hay-fail trước** (lint/unit) để fail-fast, khâu đắt (e2e/load) sau; mỗi stage là gate.

**N-CICD-003** ⭐⭐⭐⭐
"Vì sao nguyên tắc 'build once, promote the same artifact' quan trọng? Build lại image cho mỗi môi trường sai ở đâu?"
> *Dò cái gì:* build 1 artifact bất biến (image theo digest) rồi **promote qua dev→staging→prod**; build lại mỗi env → artifact prod *không phải* cái đã test (deps/base đổi, non-reproducible) → 'works in staging, breaks in prod'; config khác nhau inject lúc runtime, không rebuild.

**N-CICD-004** ⭐⭐⭐⭐
"Quản lý secret/credential trong CI thế nào cho an toàn? Vì sao **OIDC keyless** (federated identity) tới cloud tốt hơn lưu access key dài hạn?"
> *Dò cái gì:* không hardcode trong YAML/log; dùng secret store của CI; tốt nhất **OIDC**: runner đổi token ngắn hạn lấy quyền cloud theo job → **không có long-lived key để rò**; least-privilege per pipeline; mask log; xoay vòng. *(threat model → M-SUPPLY)*

**N-CICD-005** ⭐⭐⭐
"Cache trong CI tăng tốc bằng cách nào (dependency cache, Docker layer cache)? Một rủi ro của cache sai?"
> *Dò cái gì:* cache `~/.npm`/`node_modules`/build output + layer cache → bỏ tải/biên dịch lại; rủi ro: cache 'bẩn'/stale (key sai → kéo deps cũ, ẩn lỗi lockfile) → cần key theo hash lockfile + invalidation đúng.

**N-CICD-006** ⭐⭐⭐
"Đặt các 'quality gate' nào trong pipeline để chặn code xấu lên prod? (test/coverage/scan/contract)" *(bản chất từng loại test → L)*
> *Dò cái gì:* gate chặn merge/deploy: unit+integration pass, coverage/mutation ngưỡng, lint, security/dependency scan, **contract test 'can-i-deploy'**, smoke sau deploy; hiểu *gate là điểm chặn*, còn từng loại test làm gì thì trỏ L. Cân tốc độ feedback vs độ chặt.

**N-CICD-007** ⭐⭐⭐⭐
"Monorepo vs polyrepo ảnh hưởng thiết kế pipeline thế nào? Làm sao chỉ build/deploy service *thực sự* đổi?"
> *Dò cái gì:* polyrepo = mỗi service pipeline riêng, trigger theo repo; monorepo = **path filter / affected detection** để chỉ chạy phần đổi (tránh build cả thế giới); shared step (lint/scan) tách reusable workflow; trade-off: monorepo dễ refactor xuyên service nhưng cần tooling lọc.

**N-CICD-008** ⭐⭐⭐⭐⭐
"Thiết kế CI/CD cho hệ ~50 microservices: anh quyết những gì (repo layout, trigger, deploy, credential)?"
> *Dò cái gì:* quyết mono/poly trước; pipeline per-service (poly) hoặc path-filter (mono); reusable templates cho step chung; **deploy qua GitOps** để pipeline không giữ cluster credential; versioning/contract giữa service; rollout độc lập; chuẩn hoá để team mới onboard nhanh (mầm platform engineering). Trỏ K cho independent deploy.

**N-CICD-009** ⭐⭐⭐
"Trunk-based development vs Gitflow ảnh hưởng tần suất & rủi ro deploy thế nào? Vì sao branch sống lâu hại CI?"
> *Dò cái gì:* trunk-based = branch ngắn, merge thường, feature flag che việc dở → ít merge conflict, deploy thường, lô nhỏ dễ rollback; branch sống lâu = merge hell + tích hợp muộn + lô to khó debug; gitflow nặng release branch hợp release chậm/đóng gói.

**N-CICD-010** ⭐⭐⭐⭐
"Bốn **DORA metrics** là gì và mỗi cái đo sức khoẻ delivery ở khía cạnh nào? Vì sao 'deploy thường xuyên' lại đi cùng 'an toàn hơn'?"
> *Dò cái gì:* deployment frequency, lead time for changes, change failure rate, MTTR; deploy thường = lô nhỏ → ít thay đổi/lần → dễ test/rollback/định vị lỗi → CFR thấp, MTTR ngắn; đo *outcome* chứ không chỉ hoạt động.

---

## N-DEPLOY — Deployment strategies

**N-DEPLOY-001** ⭐⭐⭐
"**Recreate** vs **Rolling** deployment khác gì? Khi nào buộc phải Recreate dù có downtime?"
> *Dò cái gì:* recreate = kill hết bản cũ rồi dựng bản mới (có downtime, đơn giản); rolling = thay dần, không downtime; recreate cần khi bản cũ/mới không thể chạy đồng thời (vd lock tài nguyên độc quyền, schema không tương thích cùng lúc).

**N-DEPLOY-002** ⭐⭐⭐⭐
"**Blue-green** deployment hoạt động ra sao? Ưu (rollback tức thì) và nhược (chi phí, state/DB) là gì?"
> *Dò cái gì:* dựng môi trường mới (green) song song bản cũ (blue), test xong **chuyển toàn bộ traffic** sang green; rollback = trỏ lại blue tức thì; nhược: tốn gấp đôi tài nguyên lúc chuyển, và **DB/state dùng chung** vẫn phải tương thích cả hai bản.

**N-DEPLOY-003** ⭐⭐⭐⭐
"**Canary** deployment là gì và khác blue-green chỗ nào? 'Metric-gated canary' tự động rollback dựa trên gì?"
> *Dò cái gì:* canary = đẩy bản mới cho **một % traffic nhỏ**, tăng dần nếu khoẻ; khác blue-green (không chuyển 100% ngay); gate dựa error rate/latency/SLO so baseline → tự promote hoặc rollback; cần observability + traffic splitting (mesh/ingress/Argo Rollouts).

**N-DEPLOY-004** ⭐⭐⭐⭐
"Khi nào chọn blue-green, khi nào canary, khi nào rolling? Trade-off chính của mỗi cái?"
> *Dò cái gì:* rolling = mặc định rẻ, rollback chậm hơn, hai bản cùng tồn tại; blue-green = rollback tức thì nhưng tốn tài nguyên + cutover rủi ro; canary = an toàn nhất cho thay đổi rủi ro cao nhưng cần đo lường + hạ tầng traffic-split; chọn theo rủi ro thay đổi + chi phí + khả năng quan sát.

**N-DEPLOY-005** ⭐⭐⭐⭐⭐
"Vì sao 'rollback' nói thì dễ làm thì khó khi bản mới đã **chạy migration DB**? Phân biệt rollback code và rollback schema."
> *Dò cái gì:* code rollback nhanh (đổi image), nhưng schema đã đổi (drop/rename cột, đổi type) thì không lùi sạch + dữ liệu bản mới ghi có thể không hợp bản cũ → rollback code mà schema mới = vỡ; phải thiết kế migration **backward-compatible** (xem N-DEPLOY-006) để rollback an toàn.

**N-DEPLOY-006** ⭐⭐⭐⭐⭐
"Thiết kế migration DB **backward-compatible** (expand/contract) để deploy zero-downtime: các pha là gì? Cho ví dụ đổi tên cột."
> *Dò cái gì:* **expand** (thêm cột/bảng mới, code mới ghi cả cũ+mới hoặc đọc cả hai) → deploy code đọc/ghi tương thích → **backfill** dữ liệu → **contract** (xoá cũ) ở release sau; không bao giờ 'đổi tên cột + đổi code' trong một bước; mỗi bước hai bản code phải cùng chạy được. *(transaction/consistency → liên hệ I, K Saga/Outbox)*

**N-DEPLOY-007** ⭐⭐⭐⭐
"Feature flag tách 'deploy' khỏi 'release' thế nào? Lợi ích và cái giá (flag debt) là gì?"
> *Dò cái gì:* deploy code (tắt flag) ≠ bật tính năng cho user; cho phép canary theo user/%, kill-switch tức thì, A/B; giá: phức tạp test (tổ hợp flag), **flag debt** nếu không dọn, runtime config phải tin cậy; flag là quyết định runtime, deploy là kỹ thuật.

**N-DEPLOY-008** ⭐⭐⭐⭐
"Liệt kê điều kiện để một service đạt **zero-downtime deploy**. Vì sao API phải backward-compatible trong lúc rolling?" *(versioning → F)*
> *Dò cái gì:* readiness probe đúng, graceful drain (N-K8SOPS-011), **trong lúc rolling hai phiên bản chạy đồng thời** → API/contract/schema/message phải tương thích ngược (client cũ gọi server mới và ngược lại); connection draining ở LB; idempotent retry. Trỏ F cho versioning.

**N-DEPLOY-009** ⭐⭐⭐⭐⭐
"Tình huống: deploy prod fail, anh bấm rollback nhưng **rollback cũng fail**. Anh xử lý theo thứ tự nào?"
> *Dò cái gì:* **cầm máu trước** (chuyển traffic về env/region khoẻ nếu có, bật maintenance/kill-switch) → đánh giá hệ đang ở state nào (giữa chừng? partial?) → không làm tệ hơn → khôi phục dịch vụ → **giao tiếp stakeholder** → sau khi ổn mới postmortem không đổ lỗi + thêm rào chặn tái diễn. Tư duy incident, không 'thử lại bừa'.

**N-DEPLOY-010** ⭐⭐⭐⭐
"Deploy một StatefulSet (vd DB cluster) khác deploy stateless ở đâu? Vì sao rolling 'thường' có thể nguy hiểm?"
> *Dò cái gì:* StatefulSet update **có thứ tự** (ordinal), pod giữ identity/PVC; rolling ẩu có thể đụng quorum/leader (mất đa số → cụm down), hoặc schema/compat của data; cần partition rollout, kiểm tra health từng node, để ý leader election; thường để **operator** lo (vd Postgres/Kafka operator).

---

## N-IAC — Infrastructure as Code

**N-IAC-001** ⭐⭐⭐
"IaC (Terraform/OpenTofu) giải quyết vấn đề gì so với click console/chạy script tay? 'Snowflake server' là gì?"
> *Dò cái gì:* hạ tầng mô tả bằng code → versioned, review qua PR, reproducible, tự tài liệu, dựng lại được sau thảm hoạ; tránh **snowflake** (server cấu hình tay không ai biết khác chỗ nào) và config drift; declarative desired state.

**N-IAC-002** ⭐⭐⭐⭐
"Terraform **state** là gì và vì sao mất nó là thảm hoạ? Vì sao production cần **remote state + state locking**?"
> *Dò cái gì:* state = ánh xạ resource khai báo ↔ resource thật trên cloud; mất state → Terraform 'tưởng chưa có gì' → định tạo lại/không quản được cái đang tồn tại; remote backend (S3+DynamoDB lock/Terraform Cloud) để team chia sẻ + **lock tránh hai người apply đồng thời** làm hỏng state; state chứa secret → cần mã hoá.

**N-IAC-003** ⭐⭐⭐
"`plan` vs `apply` khác gì, và tính **idempotency** thể hiện ở đâu? Vì sao luôn xem plan trước khi apply?"
> *Dò cái gì:* plan = tính diff desired vs actual, **không đổi gì** → review; apply = thực thi diff; chạy lại apply không tạo trùng (idempotent — chỉ làm phần lệch); plan để bắt thay đổi ngoài ý (destroy nhầm) trước khi chạm prod.

**N-IAC-004** ⭐⭐⭐⭐
"Config **drift** là gì (ai đó sửa tay trên console) và Terraform phát hiện/xử lý ra sao? Vì sao drift nguy hiểm với GitOps tư duy?"
> *Dò cái gì:* thực tế lệch khỏi state/code do thay đổi out-of-band; `plan` lộ drift (sẽ 'sửa lại' về code); nguy hiểm: code không còn là source of truth, apply sau có thể ghi đè/đập thứ ai đó sửa gấp; kỷ luật: mọi thay đổi qua code, hạn chế quyền sửa tay.

**N-IAC-005** ⭐⭐⭐
"Terraform (declarative) vs Ansible (thiên imperative/config mgmt) khác mục đích thế nào? Provisioning vs configuration management."
> *Dò cái gì:* Terraform = **provisioning** hạ tầng (tạo VM/network/cluster, declarative desired state); Ansible = thiên **configuration management** (cài/cấu hình bên trong máy, push step); thường phối: Terraform dựng, Ansible/cloud-init cấu hình; container hoá làm phần config nhẹ đi.

**N-IAC-006** ⭐⭐⭐
"Terraform **module** và biến môi trường (dev/staging/prod) tổ chức ra sao để DRY mà không copy-paste? Rủi ro của 'một state khổng lồ'?"
> *Dò cái gì:* tách module tái dùng + truyền biến/workspace per env; state to/monolithic → apply chậm, blast radius lớn (đổi nhỏ động cả prod), khó phân quyền; nên tách state theo domain/độ-thay-đổi; tránh hardcode giá trị môi trường.

**N-IAC-007** ⭐⭐⭐⭐⭐
"Vì sao 2026 consensus là **Terraform lo cluster/IAM/network, GitOps lo workload trên cluster** thay vì Terraform deploy cả app? Tách ranh giới này lợi gì?"
> *Dò cái gì:* hai tốc độ thay đổi khác nhau (infra chậm/ít, app deploy liên tục); Terraform apply chậm + state lock không hợp deploy app nhiều lần/ngày; tách: Terraform tới biên cluster, **ArgoCD/Flux reconcile workload** từ Git → app rollout nhanh, an toàn, không trộn blast radius infra với app; tránh circular dependency (cluster vs resource trong cluster).

**N-IAC-008** ⭐⭐⭐
"Vì sao có **OpenTofu** bên cạnh Terraform, và điều đó ảnh hưởng lựa chọn của team thế nào?" *(verify)*
> *Dò cái gì:* HashiCorp đổi Terraform sang license BSL (2023) → cộng đồng fork **OpenTofu** (open source, Linux Foundation), tương thích phần lớn; team cân nhắc license/governance/feature; hiểu đây là *yếu tố license & ecosystem*, không phải khác biệt kỹ thuật lớn lúc đầu. *(verify trạng thái hiện tại)*

---

## N-HELM — Packaging

**N-HELM-001** ⭐⭐⭐
"Helm giải quyết vấn đề gì khi quản lý manifest K8s? 'Package manager cho Kubernetes' nghĩa là gì?"
> *Dò cái gì:* tránh copy-paste YAML cho nhiều env/service; **templating** + values để tham số hoá, đóng gói thành **chart** cài/nâng cấp/gỡ như một đơn vị (release), versioned, share qua registry; quản lý nhóm resource liên quan như một thực thể.

**N-HELM-002** ⭐⭐⭐
"Cấu trúc một Helm chart (templates, `values.yaml`, Chart.yaml) và cơ chế render? `values` override theo thứ tự nào?"
> *Dò cái gì:* templates = manifest có placeholder (Go template); values = giá trị mặc định; render = template + values → YAML thật; override: defaults < `-f file` < `--set` (cái sau thắng); tách config khỏi cấu trúc.

**N-HELM-003** ⭐⭐⭐
"`helm upgrade` và `helm rollback` làm gì? 'Release revision' giúp rollback ra sao? Khác với rollback Deployment thuần?"
> *Dò cái gì:* mỗi upgrade tạo revision mới; rollback về revision trước theo manifest đã lưu; tiện hơn nhớ lại YAML cũ; nhưng vẫn dính vấn đề state/DB như mọi rollback (Helm không lo schema).

**N-HELM-004** ⭐⭐⭐⭐
"Helm vs Kustomize — triết lý khác nhau (templating vs overlay/patch) và khi nào chọn cái nào?"
> *Dò cái gì:* Helm = templating + packaging + release lifecycle (mạnh cho phân phối/app phức tạp, nhiều tham số); Kustomize = base + overlay patch YAML thuần, không template-engine (đơn giản, không 'logic trong YAML'), tích hợp `kubectl`; chọn theo cần đóng gói/chia sẻ (Helm) hay chỉ tuỳ biến môi trường (Kustomize); nhiều nơi phối cả hai.

**N-HELM-005** ⭐⭐⭐⭐
"Rủi ro của chart Helm phức tạp (logic/điều kiện nhồi trong template YAML) là gì? Là TL anh giữ chart 'lành mạnh' thế nào?"
> *Dò cái gì:* YAML + Go template lồng if/range/whitespace → khó đọc/test/debug, lỗi indent âm thầm; giữ chart đơn giản, ít logic, dùng `helm lint`/`helm template` review output, schema `values.schema.json` validate input, tách phần phức tạp; ưu tiên rõ ràng hơn 'thông minh'.

---

## N-GITOPS — GitOps

**N-GITOPS-001** ⭐⭐⭐
"GitOps là gì ở mức nguyên lý? 'Git là single source of truth' và 'continuous reconciliation' nghĩa là gì?"
> *Dò cái gì:* trạng thái mong muốn của cluster khai báo trong Git; một operator liên tục so sánh cluster vs Git và **kéo cluster về khớp Git**; deploy = commit/PR (có review, audit, revert); không `kubectl apply` tay.

**N-GITOPS-002** ⭐⭐⭐⭐⭐
"Phân biệt **push** vs **pull** deployment model, và vì sao pull-based GitOps là một cải thiện về **bảo mật**?"
> *Dò cái gì:* push = CI (Jenkins/Actions) chạy `kubectl/helm` đẩy vào cluster → **CI phải giữ cluster credential** (blast radius lớn nếu rò); pull = operator *trong cluster* (ArgoCD/Flux) tự kéo từ Git → **CI không cần credential cluster**, cluster chủ động → giảm bề mặt tấn công, tách CI khỏi quyền deploy. Đây là điểm hiring manager hay dò.

**N-GITOPS-003** ⭐⭐⭐
"ArgoCD vs Flux khác gì ở mức cao? Một câu hỏi 'bẫy về trung thực' interviewer hay gài quanh chủ đề này là gì?"
> *Dò cái gì:* cùng GitOps operator cho K8s; ArgoCD có UI/CRD Application rõ, Flux nhẹ/GitOps Toolkit/CLI-first; **bẫy:** đừng phóng đại 'thạo cả hai' — nếu sâu ArgoCD, nông Flux thì nói thật (interviewer kiểm tra độ trung thực, không phải bắt thuộc).

**N-GITOPS-004** ⭐⭐⭐⭐
"Drift reconciliation / self-heal của GitOps hoạt động ra sao? Ai đó `kubectl edit` tay trên prod thì chuyện gì xảy ra?"
> *Dò cái gì:* operator phát hiện cluster lệch Git → đánh dấu OutOfSync và (nếu auto-sync/self-heal) **đưa về đúng Git** → thay đổi tay bị 'nuốt'; lợi: chống drift, mọi thay đổi qua Git; lưu ý: sửa khẩn cấp tay sẽ bị revert → cần hotfix qua Git hoặc tạm tắt auto-sync có kiểm soát.

**N-GITOPS-005** ⭐⭐⭐⭐
"Luồng từ 'CI build image' tới 'GitOps deploy' nối với nhau thế nào? Vì sao thường tách **app repo** và **config/deploy repo**?"
> *Dò cái gì:* CI build/test/push image → cập nhật tag image trong **deploy repo** (qua image updater/commit/PR) → operator thấy commit → reconcile cluster; tách repo: deploy repo = source of truth của *trạng thái mong muốn* (audit/rollback bằng git), app repo lo source code; tách quyền và lịch sử rõ ràng.

**N-GITOPS-006** ⭐⭐⭐
"Rollback trong GitOps làm thế nào và vì sao 'sạch' hơn cách truyền thống?"
> *Dò cái gì:* `git revert` về commit/manifest trước → operator reconcile về trạng thái cũ; có lịch sử/audit ai-đổi-gì-khi-nào, không 'bấm nút bí ẩn'; vẫn dính giới hạn state/DB (rollback schema vẫn khó — N-DEPLOY-005).

---

## N-CLOUD — Managed cloud services

**N-CLOUD-001** ⭐⭐⭐
"Managed K8s (EKS/GKE/AKS) lo giúp anh phần nào, anh vẫn phải lo phần nào? Trade-off so với tự dựng cluster?"
> *Dò cái gì:* cloud quản **control plane** (API server, etcd, scheduler — HA, patch); bạn lo node/worker (hoặc dùng node tự quản/serverless node), workload, networking/policy, cost, upgrade version; trade-off: bớt gánh vận hành control plane nhưng kém kiểm soát + chi phí + theo nhịp version của cloud.

**N-CLOUD-002** ⭐⭐⭐
"Khi nào dùng **RDS/managed DB** thay vì tự chạy Postgres (trên VM/K8s)? Cái gì anh đánh đổi khi chọn managed?"
> *Dò cái gì:* managed lo backup/PITR, replica/failover, patch, scaling storage → giảm gánh vận hành DB (vốn khó, stateful); đánh đổi: chi phí, ít quyền tinh chỉnh (extension/superuser hạn chế), lock-in; tự chạy DB trên K8s khó vì stateful (N-K8SOPS-008) → mặc định nghiêng managed cho prod.

**N-CLOUD-003** ⭐⭐⭐
"S3/object storage hợp cho dữ liệu gì, KHÔNG hợp cho gì? Khác block storage và file storage ra sao?"
> *Dò cái gì:* object = blob lớn truy cập qua API/HTTP, bền, rẻ, scale vô hạn (ảnh/video/backup/static asset/log) nhưng **không** cho random write/filesystem semantic/latency thấp như DB; block (EBS) cho ổ gắn VM; file (EFS/NFS) cho chia sẻ POSIX; chọn theo access pattern.

**N-CLOUD-004** ⭐⭐⭐
"CloudFront/CDN đặt ở đâu trong kiến trúc và tăng tốc bằng cách nào? Khi nào nó **không** giúp?" *(invalidation/TTL → J)*
> *Dò cái gì:* CDN = cache + serve nội dung ở **edge gần user** → giảm latency + tải origin cho asset tĩnh/cacheable; không giúp (ít) cho nội dung động/cá nhân hoá per-request hoặc API ghi; cần chiến lược cache-control/invalidation (trỏ J).

**N-CLOUD-005** ⭐⭐⭐⭐
"Pod cần gọi dịch vụ cloud (S3/SQS) — vì sao **không** nhét access key tĩnh vào pod? IRSA / Workload Identity giải quyết thế nào?"
> *Dò cái gì:* key tĩnh = secret dài hạn dễ rò/khó xoay, quyền rộng; **IRSA (EKS) / Workload Identity (GKE)** gắn ServiceAccount K8s với IAM role → pod nhận **token ngắn hạn tự xoay**, least-privilege per workload, không lưu key; cùng tư duy OIDC keyless ở CI (N-CICD-004). *(secret kiến trúc → M-SECRET)*

**N-CLOUD-006** ⭐⭐⭐⭐
"Managed service vs tự host (build-vs-buy) — là TL anh quyết dựa trên những yếu tố nào, không chỉ 'giá tháng'?"
> *Dò cái gì:* tổng chi phí gồm **engineer-time vận hành**, độ tin cậy/SLA, time-to-market, năng lực team, lock-in/exit cost, compliance, scale dự kiến; thường managed thắng cho thứ stateful/khó (DB, queue) trừ khi quy mô đủ lớn để tự host rẻ hơn; quyết theo bối cảnh, không tín điều.

**N-CLOUD-007** ⭐⭐⭐⭐
"Kiểm soát chi phí cloud/K8s: các đòn bẩy chính (right-sizing, autoscaling, spot, storage tier) và rủi ro mỗi cái?"
> *Dò cái gì:* right-size requests/limits (N-K8SOPS-015), cluster autoscaler + HPA, **spot/preemptible** cho workload chịu gián đoạn (rủi ro bị thu hồi → cần PDB/đa loại node), storage tiering/lifecycle, tắt env không dùng, theo dõi cost theo namespace/tag; cân chi phí vs độ tin cậy, không cắt mù.

---

## N-TL — TL-level (cross-cutting)

**N-TL-001** ⭐⭐⭐⭐⭐
"Một startup 4 service, 3 kỹ sư, muốn 'lên Kubernetes vì ai cũng dùng'. Là TL anh hỏi/quyết gì? Khi nào K8s là sai lựa chọn?"
> *Dò cái gì:* không cargo-cult — hỏi tải/SLA/đội ngũ/độ phức tạp thật; K8s mang **chi phí vận hành lớn** (đường cong học, upgrade, debug) → đội nhỏ/tải thấp thường hợp PaaS/serverless (Cloud Run, ECS Fargate, App Runner, Render) hoặc vài VM + container; chọn K8s khi đã thật sự cần orchestration nhiều service/scale/đa team. Quyết theo nhu cầu, không theo hype.

**N-TL-002** ⭐⭐⭐⭐
"Cost control ở mức tổ chức: anh thiết lập 'văn hoá + cơ chế' gì để chi phí cloud không phình âm thầm?"
> *Dò cái gì:* visibility (cost theo team/namespace/tag, dashboard, alert ngân sách), right-sizing định kỳ + autoscaling, dọn tài nguyên mồ côi, spot cho phù hợp, gắn cost vào review kiến trúc, owner chịu trách nhiệm; biến cost thành tín hiệu kỹ thuật, không phải hoá đơn cuối tháng.

**N-TL-003** ⭐⭐⭐⭐
"Quản lý secret cho nhiều service/môi trường trên K8s ở scale: vì sao K8s Secret 'trần' chưa đủ, và các pattern (External Secrets Operator / Vault / Sealed Secrets) lo gì?" *(kiến trúc/threat → M-SECRET)*
> *Dò cái gì:* K8s Secret base64 + cần encryption-at-rest + RBAC, khó rotation/audit ở scale; **External Secrets Operator** đồng bộ từ Vault/AWS Secrets Manager vào cluster; **Vault** cho dynamic/short-lived secret + rotation; **Sealed Secrets** cho phép commit secret đã mã hoá vào Git (hợp GitOps); chọn theo nhu cầu rotation/GitOps. Trỏ M cho chi tiết.

**N-TL-004** ⭐⭐⭐⭐⭐
"Đảm bảo 'chỉ image tin cậy mới chạy ở prod': anh dùng những lớp gì (signing, SBOM, admission policy)?" *(supply chain → M-SUPPLY)*
> *Dò cái gì:* ký image (**cosign/Sigstore**) + verify lúc admission (policy controller như Kyverno/Gatekeeper/OPA) → chặn image chưa ký/không từ registry tin cậy; SBOM + scan gate ở CI; pin theo digest; least-privilege registry; hiểu **provenance chứng minh pipeline chứ không chứng minh code an toàn** (trỏ M-SUPPLY).

**N-TL-005** ⭐⭐⭐⭐⭐
"Thiết kế cho **multi-region / DR**: phân biệt active-active vs active-passive, và RTO/RPO định hình lựa chọn thế nào? Bẫy lớn nhất?"
> *Dò cái gì:* RPO = chấp nhận mất bao nhiêu data, RTO = bao lâu phải khôi phục → quyết kiến trúc + chi phí; active-passive (failover, rẻ hơn, RTO cao hơn) vs active-active (phức tạp, cần data replication + xử lý conflict/consistency); **bẫy**: DB/state là phần khó nhất (replication lag, split-brain), và DR không test = DR không tồn tại → phải diễn tập restore. *(consistency → I)*

**N-TL-006** ⭐⭐⭐⭐
"Platform Engineering / Internal Developer Platform (IDP) và 'golden path' giải quyết vấn đề gì trong team nhiều service? Khác 'một team DevOps làm hộ' chỗ nào?" *(liên hệ R Modern Trends)*
> *Dò cái gì:* cung cấp **self-service** (template pipeline, chuẩn deploy, paved road) để dev ship độc lập mà không cần thành chuyên gia K8s → giảm cognitive load + chuẩn hoá; khác mô hình DevOps-as-gatekeeper (nghẽn cổ chai); 'golden path' = con đường mặc định tốt, vẫn cho phép lệch khi cần; mục tiêu là *enable*, không *kiểm soát tập trung*.

---

## Ghi chú dùng bộ đề (cho Bước B–E)

- **Dựng giáo trình ngược (Bước B):** gom cụm theo *dependency*: **Docker (N-DKR/N-DKRADV) → K8s core (N-K8S) → K8s ops (N-K8SOPS) → packaging/IaC/GitOps (N-HELM/N-IAC/N-GITOPS) → CI/CD & deploy (N-CICD/N-DEPLOY) → cloud & TL (N-CLOUD/N-TL)**. Học container trước orchestration trước deploy strategy.
- **Trọng tâm theo roadmap:** mục N chỉ ~30% tần suất và độ sâu kỳ vọng ở TL backend là *"đủ để bàn deploy/scale trong system design + ra quyết định"* — nên ưu tiên **N-DKR, N-K8S, N-K8SOPS, N-CICD, N-DEPLOY** (đạt vững), còn IaC/Helm/GitOps/Cloud đạt mức *"giải thích + trade-off"*, các câu ⭐⭐⭐⭐⭐ là điểm cộng.
- **Khi chấm live (Bước D):** mọi câu chạm **đời phiên bản** (sidecar GA, dockershim, Ingress NGINX retirement, in-place resize, OpenTofu, version K8s) → yêu cầu người học nói rõ "cái này cần verify ở docs" thay vì khẳng định số/đời; đúng tinh thần *"hiểu để chỉ huy & kiểm tra, phần còn lại tra cứu"*.
