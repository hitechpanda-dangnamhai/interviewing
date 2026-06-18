# ⚙️ Mục N — DevOps / Docker / Kubernetes / CI-CD · Giáo trình đầy đủ để học

> **Sinh theo WORKFLOW 2 — Bước B (dựng giáo trình ngược từ bộ đề) + Bước C (giảng từng bài theo Hợp đồng 10 mục).**
> Nguồn câu hỏi: `QB_N_devops.md` (**98 câu**, 11 mục con). Đây là **tài liệu học để lưu**, không phải phiên phỏng vấn — phỏng vấn live là Bước D, làm sau.
> **Bối cảnh kiến thức: tính tại 6/2026.** Mọi mục có *(verify)* là thứ đổi nhanh (đời K8s, tên tool, license) → kiểm chứng tại docs chính thức khi thực hành.
> Quy ước nguồn: **📘** = có trong roadmap/QB gốc · **➕** = bổ sung của giảng viên · **⬆️** = làm sâu hơn.

---

## 🔎 Bối cảnh hiện hành đã verify (6/2026) — đọc trước

Những mốc này được dùng xuyên suốt tài liệu (đã search + verify tại nguồn chính thức; vẫn nên tự kiểm chứng vì đổi nhanh). **Đây cũng chính là các câu "đính chính" so với mặc định cũ mà interviewer hay dò:**

- **Dockershim đã bị gỡ khỏi Kubernetes từ v1.24** (công bố ở v1.20). Câu "K8s dùng Docker làm runtime" là **lỗi thời** — K8s nói chuyện với container runtime qua **CRI** (containerd, CRI-O). Image build theo chuẩn **OCI** (kể cả build bằng Docker) vẫn chạy bình thường. *(verify: kubernetes.io)*
- **Native sidecar containers** (`initContainers` có `restartPolicy: Always`): feature gate bật mặc định từ **v1.29**, **GA/stable ở v1.33** ("Octarine", 4/2025). K8s đảm bảo sidecar **khởi động trước app, tắt sau app**. Trước đó phải "chế" bằng container thường → vỡ thứ tự start/stop. *(verify: kubernetes.io/docs sidecar-containers)*
- **In-Place Pod Resize** (đổi CPU/memory requests/limits không cần restart pod): alpha v1.27 → beta v1.33 → **GA/stable ở v1.35** (12/2025). *Đính chính QB:* không còn là "verify đời" mơ hồ — nay nói được **GA v1.35**. *(verify: kubernetes.io blog 1.35)*
- **Ingress NGINX (community `kubernetes/ingress-nginx`) đã RETIRED 3/2026** — repo **archived read-only**, không còn release/bugfix/security patch. **Đính chính QB** (QB ghi "~khai tử": nay đã *thật sự* EOL). Hướng kế thừa: **Gateway API** (đã v1.0 GA). Lưu ý: **Ingress API `networking.k8s.io/v1` KHÔNG bị xoá** — chỉ *feature-frozen*; controller thay thế còn-sống: Traefik, Contour, Cilium, **F5/NGINX Inc. controller** (dự án *khác* với community). *(verify: kubernetes.io blog 11/2025)*
- **K8s đang ở vùng ~1.35–1.36** (6/2026; EKS/GKE/AKS trễ hơn upstream vài đời). Đừng khẳng định version cụ thể mà không tra. Vài thay đổi "phá vỡ" gần đây: **cgroup v1 bị gỡ ở 1.35**, **containerd 1.x EOL**, **kube-proxy IPVS bị deprecate** (đang dịch sang **nftables** backend). *(verify)*
- **Terraform đổi license sang BSL (8/2023)** → cộng đồng fork thành **OpenTofu** (Linux Foundation, open source, drop-in tương thích phần lớn HCL). Nhắc IaC nên biết cặp **Terraform / OpenTofu**. *(verify: opentofu.org)*

> 🧭 **Câu thần chú của mục N:** *AI viết đúng cú pháp Dockerfile/YAML nhưng dễ **sai vận hành** — `latest` tag không reproducible, shell-form ENTRYPOINT nuốt SIGTERM, liveness == readiness gây restart loop, rollback code khi schema đã đổi là vỡ. Bắt những chỗ đó — và **biết khi nào KHÔNG nên dùng K8s** — là việc của Tech Lead.*

> ⚠️ **Trọng tâm theo roadmap:** mục N chỉ ~30% tần suất; độ sâu kỳ vọng ở TL backend là *"đủ để bàn deploy/scale trong system design + ra quyết định"*. Ưu tiên **đạt vững**: N-DKR, N-K8S, N-K8SOPS, N-CICD, N-DEPLOY. Còn IaC/Helm/GitOps/Cloud đạt mức *"giải thích + trade-off"*; các câu ⭐⭐⭐⭐⭐ là điểm cộng.

---

# 🧱 BƯỚC B — Giáo trình ngược dựng từ 98 câu hỏi

## Nguyên tắc sắp xếp: **dependency order**, không theo tần suất hỏi

Chuỗi phụ thuộc khái niệm: hiểu **container** là gì (process bị cô lập, không phải VM) trước → **đóng gói/hardening** container → lên **K8s core** (vì sao cần orchestration, kiến trúc, Pod/Service) → **K8s operations** (probe/resource/rollout/scheduling — phần "chạy thật ở prod") → **đóng gói manifest** (Helm/Kustomize) và **dựng hạ tầng** (IaC) và **deploy kiểu GitOps** → ghép tất cả vào **CI/CD pipeline** → chọn **chiến lược deploy** (rolling/blue-green/canary + migration DB) → cuối cùng là **cloud managed services** và **lớp quyết định TL** (khi nào KHÔNG dùng K8s, cost, secret ở scale, supply chain, DR). Đặt K8s-ops **trước** deploy strategy có chủ đích: chiến lược deploy zero-downtime *dựa trên* probe + graceful shutdown + rolling params đã học ở ops.

## Mục lục bài học → ID câu hỏi nó phủ

| Bài | Tên bài | Mục con | ID câu phủ | Số câu |
|---|---|---|---|---|
| **1** | Container vs VM & Docker fundamentals | N-DKR | N-DKR-001 → 011 | 11 |
| **2** | Docker runtime & build sâu (isolation, hardening, registry) | N-DKRADV | N-DKRADV-001 → 008 | 8 |
| **3** | Kubernetes core I — vì sao K8s, kiến trúc, Pod/Service | N-K8S | N-K8S-001 → 008 | 8 |
| **4** | Kubernetes core II — config, namespace, label, etcd | N-K8S | N-K8S-009 → 012 | 4 |
| **5** | K8s ops I — probes, requests/limits/QoS, OOMKilled, HPA | N-K8SOPS | N-K8SOPS-001 → 004 | 4 |
| **6** | K8s ops II — rolling, PDB, workload types, StatefulSet, scheduling | N-K8SOPS | N-K8SOPS-005 → 009 | 5 |
| **7** | K8s ops III — Ingress/Gateway API, graceful, CronJob, sidecar, debug, resize | N-K8SOPS | N-K8SOPS-010 → 015 | 6 |
| **8** | Packaging — Helm & Kustomize | N-HELM | N-HELM-001 → 005 | 5 |
| **9** | Infrastructure as Code — Terraform / OpenTofu | N-IAC | N-IAC-001 → 008 | 8 |
| **10** | GitOps — ArgoCD/Flux, pull vs push, drift/self-heal | N-GITOPS | N-GITOPS-001 → 006 | 6 |
| **11** | CI/CD pipeline | N-CICD | N-CICD-001 → 010 | 10 |
| **12** | Deployment strategies & zero-downtime DB migration | N-DEPLOY | N-DEPLOY-001 → 010 | 10 |
| **13** | Managed cloud services (EKS/RDS/S3/CDN/IRSA/cost) | N-CLOUD | N-CLOUD-001 → 007 | 7 |
| **14** | TL-level cross-cutting (no-K8s, cost, secrets, signing, DR, platform eng) | N-TL | N-TL-001 → 006 | 6 |

**Tổng: 14 bài / 98 câu.** Mỗi bài dưới đây theo **Hợp đồng đầu ra 10 mục**.

---

# 🎓 BƯỚC C — Các bài giảng

---

## Bài 1 — Container vs VM & Docker fundamentals
*Phủ: N-DKR-001 → 011*

### ① Mục tiêu & vị trí trong mạch
Đây là **nền của cả mục N** (Phase đầu của chuỗi DevOps). Học xong bạn **hết coi container là "VM nhỏ"**, hiểu image/layer/cache, viết được Dockerfile production và biết các bẫy (PID 1, shell-form, alpine/musl). Mọi thứ về K8s sau này đều giả định bạn đã nắm bài này — K8s chỉ là *điều phối* các container.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Một **container không phải máy ảo**. Nó là **một process bình thường trên kernel của host**, nhưng bị "bịt mắt" để chỉ thấy một phần hệ thống (qua *namespaces*) và bị "khoá tay" về tài nguyên (qua *cgroups*). VM thì khác hẳn: có **hypervisor** + **guest kernel + guest OS đầy đủ**.

**(b) Cơ chế.**
- **Container vs VM (N-DKR-001):** container chia sẻ kernel host → khởi động **mili-giây**, image cỡ **MB**. VM bootstrap cả guest OS → cỡ **GB**, khởi động **giây**. Đánh đổi: cùng kernel ⇒ **ranh giới cô lập yếu hơn VM** (một lỗ kernel có thể cho "thoát container"); VM có boundary phần cứng ảo nên mạnh hơn nhưng nặng.
- **Image vs Container (N-DKR-002):** *image* = template **read-only**, gồm nhiều **layer** xếp chồng. *Container* = image + **một writable layer (copy-on-write)** ở trên cùng. Chạy 10 container từ 1 image → 10 cái **share chung các layer read-only**, mỗi cái chỉ thêm một writable layer mỏng → **không tốn 10× dung lượng**.
- **Layer caching (N-DKR-003):** mỗi lệnh Dockerfile tạo 1 layer, cache theo nội dung. **Đổi 1 layer → invalidate mọi layer phía sau.** Vì vậy **thứ tự lệnh quyết định tốc độ build** (xem code ⑤).
- **Multi-stage build (N-DKR-004):** tách *stage build* (chứa compiler/devDeps) khỏi *stage runtime* (chỉ artifact + runtime). `COPY --from=builder` chỉ bê output sang → image cuối **nhỏ, ít attack surface, không lộ source/build secret**.
- **Giảm size image (N-DKR-005):** base nhỏ (slim/alpine/**distroless**), **multi-stage**, `.dockerignore`, gộp/giảm layer, dọn cache *trong cùng layer*, chỉ cài prod deps. **Tác động lớn nhất** thường là *base image + multi-stage*. Nhỏ ⇒ pull nhanh, ít CVE, scale nhanh.
- **`.dockerignore` (N-DKR-007):** loại file khỏi **build context** gửi cho daemon → tránh copy `node_modules`/`.git`/`.env` vào image. Bỏ qua nó: build chậm, image to, **lộ secret/.git history**, phá cache.
- **`COPY` vs `ADD` (N-DKR-008):** `COPY` rõ ràng, dự đoán được. `ADD` có "magic" (giải nén tar, tải URL) → dễ bất ngờ/rủi ro. **Mặc định dùng `COPY`**, `ADD` chỉ khi cần giải nén tar local.

**(c) Mép giới hạn & sai lầm thường gặp.**
- **PID 1 (N-DKR-006, N-DKR-010):** trong container, process đầu là **PID 1** — có ngữ nghĩa *init* đặc biệt: mặc định **bỏ qua signal không có handler** và phải **reap zombie** process con. Nếu app làm PID 1 mà **không bắt SIGTERM**, thì `docker stop`/K8s gửi SIGTERM bị **nuốt** → sau timeout bị **SIGKILL** → mất graceful drain. Hơn nữa **shell-form** `CMD node app.js` bị bọc trong `/bin/sh -c` ⇒ `sh` là PID 1, **không forward signal** cho node. Fix: dùng **exec-form** `["node","app.js"]`, và/hoặc init nhẹ (`tini`, cờ `--init`). *(cơ chế bắt signal phía Node → trỏ G-DIAG-009; phía Nest → H-399)*
- **Alpine/musl bẫy (N-DKR-011):** alpine cực nhỏ nhưng dùng **musl libc** (không phải glibc) → **native addon/DNS/locale** có thể lệch; native module build cho glibc có thể **vỡ trên musl**. distroless = không shell/package manager (an toàn, khó debug). slim = glibc, cân bằng. **Đừng chọn theo "nhỏ nhất thắng" — phải test.**

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ (trong docs / phổ biến) | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Tag `latest` ở production | Tag **bất biến** (semver/commit-sha) hoặc pin **`@sha256:digest`** | `latest` mơ hồ → mỗi node/lần pull có thể khác bản → không reproducible, rollback mù *(N-DKRADV-006)* |
| `COPY . .` rồi mới `npm install` | `COPY package*.json` → install → `COPY . .` | sửa code không phá cache deps; sai thứ tự ⇒ mỗi lần sửa code reinstall toàn bộ |
| App chạy thẳng PID 1 không handle signal | Handle SIGTERM trong app **hoặc** `tini`/`--init` | tránh nuốt SIGTERM → mất graceful shutdown, để lại zombie |
| Image "full OS" (ubuntu/node bản đầy) | **multi-stage + distroless/slim** | nhỏ hơn nhiều, ít CVE, pull nhanh |
| `ADD` lấy URL/giải nén tuỳ tiện | `COPY` (mặc định) | dự đoán được, ít rủi ro |

### ④ Áp dụng thực tế + So sánh bigtech
Mọi backend ship lên cloud hôm nay gần như đều đóng gói thành **OCI image**. Use case điển hình: build Node/Go/Java service → image → push registry → K8s/ECS/Cloud Run chạy. Kiến trúc tối thiểu: *Dockerfile multi-stage → CI build → registry (ECR/GCR/GHCR) → orchestrator pull bằng digest*.

Thực hành phổ biến ở các tổ chức lớn (pattern, *không khẳng định tuyệt đối — verify theo từng nơi*): **Google** mở nguồn **distroless** base images (gcr.io/distroless) và dùng rộng nội bộ; **Netflix/Spotify/Uber** chuẩn hoá base image + scan trong pipeline; nhiều nơi pin **digest** thay vì tag để reproducible. Triết lý chung: *base nhỏ + immutable + scan* là mặc định.

### ⑤ Code thực hành + cấu hình

Dockerfile production cho Node.js (N-DKR-009) — minh hoạ multi-stage, cache deps, non-root, exec-form:

```dockerfile
# syntax=docker/dockerfile:1
# --- Stage build: có toolchain/devDeps ---
# Pin theo digest cho reproducible (verify digest tại registry chính thức)
FROM node:22-slim AS builder
WORKDIR /app
# Copy manifest TRƯỚC để tận dụng layer cache (đổi source không reinstall deps)
COPY package.json package-lock.json ./
# npm ci = cài deterministic từ lockfile (khác npm install)
RUN npm ci
COPY . .
RUN npm run build            # ví dụ tsc -> dist/

# --- Stage runtime: chỉ artifact + prod deps ---
FROM node:22-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev        # chỉ prod deps
COPY --from=builder /app/dist ./dist   # chỉ bê output build sang
# Chạy non-root (image node có sẵn user 'node')
USER node
# exec-form -> node là PID 1, nhận trực tiếp SIGTERM (graceful shutdown chạy được)
ENTRYPOINT ["node", "dist/server.js"]
# (Healthcheck thường để K8s lo bằng probe; HEALTHCHECK Dockerfile dùng khi chạy docker thuần)
```

`.dockerignore` đi kèm:

```gitignore
node_modules
npm-debug.log
.git
.env
.env.*
dist            # build lại trong image, không copy bản local
Dockerfile
.dockerignore
```

Build & chạy:

```bash
# BuildKit bật mặc định ở Docker hiện đại
docker build -t myapp:1.4.2 .
# Pin digest khi deploy thật:
docker images --digests myapp
docker run --rm --init -p 8080:8080 myapp:1.4.2   # --init thêm reaper PID1 nếu app chưa handle
```

> ⚠️ **Bảo mật/lỗi thường gặp:** không bake secret vào layer (xem N-DKRADV-007/Bài 2); không hardcode tag `latest`; nhớ `USER node` (chạy root trong container = rủi ro escape). `// version Node/digest cần kiểm chứng tại registry chính thức.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU (giải thích bằng lời):** container = process + namespaces + cgroups · copy-on-write writable layer · layer cache invalidation · multi-stage · PID 1 init semantics · exec-form vs shell-form signal handling · musl vs glibc trade-off.
- 📌 **Cần THUỘC:** `COPY` (mặc định) vs `ADD` · `npm ci --omit=dev` · `USER` non-root · `.dockerignore` · `COPY --from=builder` · `--init` / `tini`.
- 🛠️ **Cần LÀM ĐƯỢC:** viết Dockerfile multi-stage non-root đúng thứ tự cache · đọc một Dockerfile và chỉ ra chỗ phá cache / nuốt signal / chạy root.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Container = process bị cô lập, KHÔNG phải VM nhỏ."** Và: *"Dockerfile tốt = cache deps trước, multi-stage, non-root, exec-form."*

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* Container khác VM ở đâu, vì sao nhẹ hơn? → chia sẻ kernel host, không guest OS; đánh đổi cô lập yếu hơn.
2. *(TB)* Vì sao 10 container từ 1 image không tốn 10× dung lượng? → share layer read-only, mỗi cái 1 writable CoW layer.
3. *(TB)* Sắp xếp Dockerfile sai làm chậm build — cho ví dụ. → `COPY . .` trước `npm install` ⇒ mỗi lần sửa code reinstall.
4. *(khó)* Multi-stage build giải quyết gì? → tách toolchain khỏi runtime → image nhỏ, ít attack surface, không lộ build secret.
5. *(khó)* Vì sao alpine đôi khi không phải lựa chọn tốt cho Node? → musl libc ≠ glibc → native addon/DNS/locale lệch.

**Thách đố / trick:**
- *"Tôi để `CMD node app.js` (shell-form), app có handle SIGTERM mà `docker stop` vẫn mất 10s rồi bị kill — vì sao?"* → shell-form bọc trong `sh -c`; **sh là PID 1**, không forward SIGTERM cho node → node không nhận signal → bị SIGKILL sau grace. Fix: exec-form `["node","app.js"]`.
- *"Image production của tôi có cả `git`, `curl`, compiler — sao lại nguy hiểm?"* → tăng attack surface + size + CVE; build tool nên ở stage build, bị loại khỏi runtime qua multi-stage.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Viết lại một Dockerfile "xấu"** (single-stage, chạy root, `COPY . .` trước install, tag latest) thành production-grade.
   - *Đạt khi:* multi-stage; `COPY package*.json` trước `npm ci`; `--omit=dev` ở runtime; `USER` non-root; exec-form ENTRYPOINT; có `.dockerignore`; image cuối nhỏ hơn rõ rệt (`docker images` so sánh).
2. **Đo cache:** build, sửa 1 dòng source, build lại — quan sát layer nào CACHED.
   - *Đạt khi:* giải thích được vì sao layer deps vẫn CACHED còn layer `COPY . .` trở đi bị rebuild.

### ⑩ Đọc thêm / nguồn chuẩn
docs.docker.com (Dockerfile reference, multi-stage, build cache, `.dockerignore`); kubernetes.io (container runtime / CRI); Google distroless repo (github.com/GoogleContainerTools/distroless).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI sinh Dockerfile thường để **shell-form CMD** và **chạy root** — cú pháp đúng, build chạy, nhưng **graceful shutdown vỡ** và **rủi ro bảo mật**. Hãy kiểm tra: *PID 1 có nhận được SIGTERM không? Có chạy non-root không?*

---

## Bài 2 — Docker runtime & build sâu (isolation, hardening, registry)
*Phủ: N-DKRADV-001 → 008*

### ① Mục tiêu & vị trí trong mạch
Bài 1 dạy *đóng gói*. Bài này đi xuống **cơ chế dưới capo** (namespaces/cgroups), **hardening** container production, **network/volume**, **BuildKit cache & secret**, **immutable tag/digest** và **image scanning**. Đây là nền cho các câu bảo mật vận hành ở K8s và CI/CD sau này.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Container "ảo" được tạo từ **hai nhóm tính năng kernel Linux**: *namespaces* lo **cô lập tầm nhìn** ("anh thấy gì"), *cgroups* lo **giới hạn & đo tài nguyên** ("anh được dùng bao nhiêu"). Không có gì "máy ảo" cả.

**(b) Cơ chế.**
- **Namespaces & cgroups (N-DKRADV-001):** *namespaces* (PID, network, mount/filesystem, UTS, IPC, user) → container "tưởng nó cô đơn". *cgroups* (CPU, memory, I/O, pids) → giới hạn và đo. **Container = process + namespaces + cgroups.**
- **Network mode (N-DKRADV-003):** **bridge** = mạng ảo NAT mặc định (cô lập, map port); **host** = dùng thẳng network stack host (nhanh, mất cô lập, đụng port); **none** = không network; **overlay** = multi-host (swarm). Host hợp khi cần hiệu năng/port động nhưng **phá cô lập**.
- **Volume vs bind mount vs tmpfs (N-DKRADV-004):** writable layer container là **ephemeral** → mất khi xoá container (đây là lý do "data biến mất sau restart"). **volume** (Docker quản lý, portable, backup) cho data bền; **bind mount** map path host (dev/config, phụ thuộc host); **tmpfs** trong RAM (secret tạm, không chạm đĩa).
- **BuildKit (N-DKRADV-005):** build song song, cache thông minh. `--mount=type=cache` giữ thư mục cache deps **giữa các lần build mà không nằm trong image**; `--mount=type=secret` dùng secret lúc build **không leak vào layer**.
- **Immutable tag & digest (N-DKRADV-006):** `latest` = anti-pattern (không reproducible, rollback mù). Dùng tag bất biến (semver/commit-sha) hoặc pin **`@sha256:digest`**. `imagePullPolicy: Always` + latest → drift; pin digest cho xác định.

**(c) Mép giới hạn & sai lầm.**
- **Hardening nhiều lớp (N-DKRADV-002):** *"chạy non-root chưa đủ"*. Cần defense-in-depth: non-root `USER`, **drop capabilities** (`--cap-drop=ALL` + add tối thiểu), `--read-only` rootfs + tmpfs cho ghi tạm, `--security-opt no-new-privileges`, **seccomp/AppArmor** profile, **không mount docker socket**, rootless runtime. Non-root giảm nhưng **không chặn lỗ kernel/escape**. *(threat model sâu → M)*
- **Secret qua ENV var (N-DKRADV-007):** ENV bị lộ qua `docker inspect`, `/proc/<pid>/environ`, log, child process, crash dump. Build secret bake vào layer là **vĩnh viễn**. Thay bằng BuildKit `--mount=type=secret` (build) và file mount / secret manager (run). *(rotation/dynamic → M-SECRET)*
- **Image scanning (N-DKRADV-008):** scan (Trivy/Grype) ở **CI (pre-push) + registry (định kỳ) + admission (deploy)**. *"Sạch CVE hôm nay" ≠ an toàn mãi* — CVE mới công bố liên tục → image cũ thành "có lỗ" dù không đổi → cần **rescan + rebuild định kỳ**. Base nhỏ (distroless) giảm bề mặt. *(SBOM/SLSA/provenance → M-SUPPLY)*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Build cũ (legacy builder) | **BuildKit** (mặc định Docker hiện đại) | song song, cache mount, build secret không leak |
| Secret qua `ENV`/`ARG` lúc build | `--mount=type=secret` (BuildKit) | ENV/ARG bị bake vào layer / lộ qua inspect |
| Tag `latest` + `pullPolicy: Always` | Tag bất biến / `@sha256:digest` | reproducible, rollback xác định |
| "Chạy non-root là đủ an toàn" | Drop caps + read-only + seccomp + no-new-privileges | non-root không chặn escape qua lỗ kernel |
| Quét CVE một lần lúc build | Scan CI + registry **định kỳ** + admission | CVE mới xuất hiện sau khi image đã build |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: ảnh production "khó tấn công" cho service internet-facing. Kiến trúc tối thiểu: *distroless base → drop ALL caps → read-only rootfs → scan gate ở CI → pin digest*. Pattern phổ biến (verify theo nơi): nhiều tổ chức dùng **Trivy** (Aqua) hoặc **Grype** (Anchore) trong pipeline; **Chainguard** bán "low/zero-CVE" base images — phản ánh xu hướng coi *image hygiene* là hạng mục bảo mật chính. Rootless runtime và "không mount docker socket vào container" là khuyến nghị chuẩn để giảm blast radius.

### ⑤ Code thực hành + cấu hình

Hardening lúc `docker run` (N-DKRADV-002):

```bash
docker run -d \
  --user 1000:1000 \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --read-only --tmpfs /tmp \
  --security-opt no-new-privileges \
  --pids-limit 200 \
  myapp@sha256:<digest>          # pin digest, không dùng :latest
```

BuildKit cache mount + secret (N-DKRADV-005/007):

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
# Cache thư mục ~/.npm GIỮA các lần build, không nằm trong image cuối
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
# Dùng secret lúc build mà KHÔNG leak vào layer
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm run build
```

```bash
# Truyền secret an toàn (không lên layer, không lên history)
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp:1.4.2 .

# Scan trước khi push (gate CI)
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:1.4.2   # // verify cờ tại trivy docs
```

> ⚠️ **Bảo mật:** không `--privileged`, không mount `/var/run/docker.sock` vào container app (= cho phép điều khiển toàn host). `// cú pháp Trivy/severity cần kiểm chứng tại docs chính thức.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** namespaces (cô lập tầm nhìn) vs cgroups (giới hạn tài nguyên) · ephemeral writable layer · vì sao ENV-secret rò · "CVE-clean hôm nay ≠ mãi mãi" · defense-in-depth.
- 📌 **Cần THUỘC:** `--cap-drop=ALL` · `--read-only` + `--tmpfs` · `no-new-privileges` · `--mount=type=cache` / `type=secret` · `@sha256:digest`.
- 🛠️ **Cần LÀM ĐƯỢC:** hardening một `docker run` · cấu hình BuildKit cache/secret · đặt scan gate trong pipeline.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Cô lập = namespaces + cgroups; an toàn = nhiều lớp, không chỉ non-root; tin cậy = digest + scan định kỳ."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Namespaces và cgroups mỗi cái lo gì? → cô lập tầm nhìn vs giới hạn tài nguyên.
2. *(TB)* Data container biến mất sau restart vì sao, fix? → writable layer ephemeral; dùng volume.
3. *(khó)* Vì sao non-root chưa đủ để hardening? → không chặn escape qua lỗ kernel; cần drop caps/read-only/seccomp.
4. *(khó)* Vì sao truyền secret qua ENV rủi ro? → lộ qua inspect/proc/log/crash dump.
5. *(khó)* "Image sạch CVE" có nghĩa an toàn mãi? → không; CVE mới xuất hiện, cần rescan/rebuild.

**Thách đố / trick:**
- *"Tôi pin `myapp:1.4.2` rồi, sao hai node lại chạy bản khác nhau?"* → tag *có thể bị đẩy đè* (mutable) ở registry; chỉ **digest `@sha256`** mới thật sự bất biến.
- *"`--mount=type=secret` khác gì truyền `--build-arg TOKEN=...`?"* → build-arg lưu vào image history/layer (leak); secret mount chỉ hiện diện trong RUN đó, không vào layer.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Hardening một container** đang chạy root, full caps.
   - *Đạt khi:* chạy non-root, `--cap-drop=ALL` + add tối thiểu, read-only rootfs + tmpfs, no-new-privileges; app vẫn chạy.
2. **Thêm scan gate** vào pipeline build sao cho CRITICAL chặn push.
   - *Đạt khi:* build fail (exit≠0) khi có CVE CRITICAL; giải thích vì sao vẫn cần rescan định kỳ.

### ⑩ Đọc thêm / nguồn chuẩn
docs.docker.com (BuildKit, build secrets, security); kubernetes.io (Pod Security / securityContext — nối Bài 5–7); Trivy (aquasecurity.github.io/trivy), Grype (github.com/anchore/grype).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI hay đề xuất `--build-arg` để truyền token build — cú pháp chạy, nhưng **token bị bake vào image history**. Hãy chỉ huy dùng `--mount=type=secret`. Đây là *hiểu định hình yêu cầu*, không phải cú pháp.

---

## Bài 3 — Kubernetes core I: vì sao K8s, kiến trúc, Pod/Service
*Phủ: N-K8S-001 → 008*

### ① Mục tiêu & vị trí trong mạch
`docker run` chạy 1 container trên 1 host. Bài này trả lời **vì sao cần K8s**, kiến trúc **control plane / worker node**, và các đối tượng nền: **Pod → ReplicaSet → Deployment**, **Service**, **reconciliation loop**. Là nền cho toàn bộ "K8s ops" (Bài 5–7).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** K8s là **hệ điều hành cho cụm máy**: bạn **khai báo trạng thái mong muốn** ("tôi muốn 3 bản app này chạy"), K8s **liên tục làm cho thực tế khớp mong muốn** — pod chết thì thay, node mất thì xếp lại. Bạn không "chạy lệnh", bạn "tuyên bố ý định".

**(b) Cơ chế.**
- **Vì sao K8s (N-K8S-001):** khi 1 host + tay không đủ — cần **scheduling** nhiều container trên nhiều node, **self-healing**, **rolling update/rollback**, **scaling (HPA)**, **service discovery + load balancing**, config/secret, declarative desired-state. (Không phải "để cho oách".)
- **Control plane (N-K8S-002):** **API server** = cổng giao tiếp duy nhất (REST, validate, auth); **etcd** = kho state (source of truth); **scheduler** = chọn node cho pod chưa gán; **controller-manager** = các control loop kéo thực tế về desired.
- **Worker node (N-K8S-003):** **kubelet** = agent đảm bảo container trong pod chạy đúng spec + báo health; **kube-proxy** = lo networking Service (iptables/IPVS/**nftables** — IPVS đang deprecate, *verify*); runtime qua **CRI** (containerd/CRI-O) — **không còn dockershim**.
- **Pod (N-K8S-004):** đơn vị nhỏ nhất = 1+ container **share network namespace (cùng localhost/IP)** + storage + lifecycle. **Không deploy "bare pod"** vì không tự hồi sinh khi chết/node mất → dùng controller.
- **Pod ↔ ReplicaSet ↔ Deployment (N-K8S-005):** Deployment quản ReplicaSet, ReplicaSet quản số pod. Đổi image → Deployment tạo **ReplicaSet mới**, tăng dần pod mới/giảm pod cũ (rolling), **giữ RS cũ để rollback**.
- **Reconciliation loop (N-K8S-006):** khai báo *desired*, controller liên tục so *actual vs desired* và **hành động khép khoảng cách** — **level-triggered** (nhìn trạng thái), không edge-triggered (bắt sự kiện). Pod chết → tự thay. Khác hẳn script imperative chạy-một-lần rồi mặc kệ drift.
- **Service types (N-K8S-007):** **ClusterIP** (nội bộ, mặc định); **NodePort** (mở port trên mọi node); **LoadBalancer** (xin LB cloud); **headless** (`clusterIP: None`, trả thẳng IP pod — cho StatefulSet/discovery).

**(c) Mép giới hạn & sai lầm.**
- **Service routing khi IP pod đổi liên tục (N-K8S-008):** pod sinh/diệt → IP đổi. Service có **IP ảo ổn định**; **label selector** → controller cập nhật pod khoẻ vào **Endpoints/EndpointSlice**; **kube-proxy** lập trình iptables/IPVS/nftables để LB tới các IP đó. Client luôn gọi 1 địa chỉ Service, không quan tâm pod đổi. *Sai lầm:* tưởng Service "biết" pod qua tên — thực ra qua **label match** + endpoint cập nhật động.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "K8s dùng Docker làm runtime" | Runtime qua **CRI**: containerd/CRI-O | **dockershim gỡ từ v1.24**; image OCI vẫn chạy *(verify)* |
| Deploy **bare Pod** | **Deployment** (qua ReplicaSet) | bare pod không self-heal/scale/rollout |
| kube-proxy **IPVS/iptables** mặc định bàn tới | đang dịch **nftables** backend (IPVS deprecate ~1.35) | hiệu năng + hướng đi upstream *(verify)* |
| `Endpoints` (cũ) | **EndpointSlice** (mở rộng tốt hơn) | scale với nhiều endpoint |
| Tư duy "chạy script tạo hạ tầng" | **declarative desired-state + reconcile** | tự healing, idempotent |

### ④ Áp dụng thực tế + So sánh bigtech
K8s ra đời từ **Borg** của Google; CNCF chủ quản. Use case: chạy hàng chục–trăm microservice, scale theo tải, self-heal, rollout an toàn. Kiến trúc tối thiểu: *Deployment (n replica) → Service (ClusterIP) → Ingress/Gateway (vào từ ngoài)*. Managed control plane: **EKS/GKE/AKS** (Bài 13). Pattern phổ biến (verify): hầu hết tổ chức **không tự vận hành control plane** mà dùng managed; dùng **Deployment cho stateless**, để DB ra **managed service** thay vì StatefulSet (Bài 6/13).

### ⑤ Code thực hành + cấu hình

Deployment + Service tối thiểu (đọc hiểu, không học vẹt):

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }        # Deployment quản pod qua label
  template:
    metadata:
      labels: { app: web }           # pod mang label này
    spec:
      containers:
        - name: web
          image: myrepo/web@sha256:<digest>   # pin digest, không :latest
          ports: [{ containerPort: 8080 }]
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP                    # nội bộ cluster
  selector: { app: web }             # CHỌN pod qua label -> endpoint động
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f deployment.yaml -f service.yaml
kubectl get deploy,rs,pod -l app=web      # thấy Deployment -> ReplicaSet -> Pod
kubectl get endpointslices -l kubernetes.io/service-name=web   # IP pod khoẻ
kubectl rollout status deploy/web
# Mô phỏng self-heal:
kubectl delete pod -l app=web --field-selector=...   # pod bị xoá -> RS tạo lại
```

> ⚠️ `selector.matchLabels` của Deployment là **immutable** sau khi tạo — đổi nó là lỗi thường gặp. `// API version (apps/v1) ổn định; tính năng mới verify tại kubernetes.io.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** declarative desired-state · reconciliation loop (level-triggered) · vì sao Pod là đơn vị nhỏ nhất · Service IP ảo + endpoint động · control plane vs worker.
- 📌 **Cần THUỘC:** API server / etcd / scheduler / controller-manager · kubelet / kube-proxy / CRI · ClusterIP / NodePort / LoadBalancer / headless · Deployment→ReplicaSet→Pod.
- 🛠️ **Cần LÀM ĐƯỢC:** viết Deployment+Service tối thiểu · đọc `kubectl get`/`rollout status` · giải thích self-heal khi xoá pod.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Khai báo desired → controller reconcile về desired."** Và: *"Client gọi Service (IP ổn định); pod đổi IP, endpoint tự cập nhật theo label."*

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* `docker run` đã chạy container, K8s giải thêm vấn đề gì? → scheduling/self-heal/rollout/scale/discovery ở scale.
2. *(TB)* Control plane gồm gì, mỗi cái làm gì? → API server/etcd/scheduler/controller-manager.
3. *(TB)* Vì sao không deploy bare pod? → không self-heal; dùng Deployment.
4. *(khó)* Giải thích reconciliation loop / level-triggered. → so actual vs desired liên tục, khép gap, idempotent.
5. *(khó)* Service định tuyến tới pod đổi IP bằng cách nào? → selector → EndpointSlice → kube-proxy.

**Thách đố / trick:**
- *"Interviewer: K8s dùng Docker đúng không?"* → Đính chính: K8s dùng **CRI runtime** (containerd/CRI-O); **dockershim đã gỡ v1.24**; image build bằng Docker (OCI) vẫn chạy. (Đây là bẫy "trung thực/cập nhật".)
- *"Tôi `kubectl delete pod` mãi mà nó cứ mọc lại?"* → vì **ReplicaSet reconcile** giữ số replica; muốn dừng thật phải scale Deployment về 0 / xoá Deployment, không xoá pod.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Deploy web 3 replica + Service**, rồi xoá 1 pod và quan sát.
   - *Đạt khi:* giải thích được pod mới do **ReplicaSet** tạo lại; Service vẫn route (endpoint cập nhật).
2. **Đổi image** trong Deployment, theo dõi `rollout status`.
   - *Đạt khi:* mô tả Deployment tạo **RS mới**, rolling, giữ RS cũ để rollback.

### ⑩ Đọc thêm / nguồn chuẩn
kubernetes.io/docs/concepts (Pods, Deployments, Services, Controllers); "Kubernetes the Hard Way" (Kelsey Hightower) cho ai muốn hiểu sâu control plane.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI thường sinh manifest với `image: app:latest` và bare Pod cho demo — *cú pháp đúng* nhưng **không reproducible, không self-heal**. Hãy chỉ huy: Deployment + digest.

---

## Bài 4 — Kubernetes core II: config, namespace, label, etcd
*Phủ: N-K8S-009 → 012*

### ① Mục tiêu & vị trí trong mạch
Hoàn thiện "core": cách đưa **config/secret** vào pod, **namespace** cô lập gì, **label** là "chất keo", và **etcd** — bộ não cluster. Nối tiếp Bài 3, chuẩn bị cho ops.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Tách **config** khỏi **image** (12-factor) để cùng 1 image chạy mọi môi trường. Label là **nhãn dán** mà gần như mọi cơ chế K8s dựa vào để "tìm nhau". etcd là **cuốn sổ cái** duy nhất ghi mọi thứ.

**(b) Cơ chế.**
- **ConfigMap vs Secret (N-K8S-009):** ConfigMap = config **phi nhạy cảm**; Secret = dữ liệu nhạy cảm. Cả hai inject qua **env** hoặc **volume mount**.
- **Namespace (N-K8S-010):** phân vùng **logic** tên/resource/quota/RBAC. **Không** tự cô lập network (cần **NetworkPolicy**) hay node/kernel.
- **Label & selector (N-K8S-011):** gán/chọn resource theo nhãn thay tên cứng → linh hoạt. Ví dụ cơ chế dựa hoàn toàn vào label: Service chọn pod, Deployment quản pod (`matchLabels`), node affinity/anti-affinity, NetworkPolicy.
- **etcd (N-K8S-012):** giữ toàn bộ state desired/actual = **source of truth**. Mất etcd = mất "bộ não" (không apply/đọc state, cluster đóng băng dù pod còn chạy). Dùng **Raft** → cần **quorum (đa số)** → số node **lẻ** (3/5) để chịu lỗi mà vẫn bầu leader. *(Raft/consensus → I-345)*

**(c) Mép giới hạn & sai lầm.**
- **Bẫy "Secret được mã hoá" (N-K8S-009):** K8s Secret **mặc định chỉ base64-encode trong etcd — KHÔNG mã hoá**. Phải bật **encryption-at-rest** cho etcd + **RBAC chặt** + (tốt hơn) **external secret manager**. *(kiến trúc → M-SECRET, Bài 14 N-TL-003)*
- **Namespace ≠ biên giới bảo mật cứng (N-K8S-010):** nhiều namespace vẫn chung cluster/node → cần NetworkPolicy + RBAC + (nếu cần cô lập mạnh) node pool/cluster riêng.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "K8s Secret là mã hoá" | base64 + **bật encryption-at-rest** + RBAC + external manager | base64 ≠ mã hoá |
| Hardcode config trong image | ConfigMap/Secret inject runtime | 1 image chạy mọi env (12-factor) |
| Coi namespace là sandbox bảo mật | namespace = logic; thêm **NetworkPolicy**/RBAC | namespace không cô lập network/kernel |
| Secret trần trong Git | **Sealed Secrets / External Secrets / Vault** | commit secret thô là rò *(Bài 14)* |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: cùng image `web` chạy dev/staging/prod chỉ khác ConfigMap/Secret. Pattern phổ biến (verify): production hầu như **không dùng K8s Secret trần** — dùng **External Secrets Operator** đồng bộ từ **Vault/AWS Secrets Manager**, hoặc **Sealed Secrets** để hợp GitOps. etcd luôn chạy **3 hoặc 5 node** (HA) ở managed cluster — cloud lo giúp.

### ⑤ Code thực hành + cấu hình

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: web-config }
data:
  LOG_LEVEL: "info"
  FEATURE_X: "true"
---
apiVersion: v1
kind: Secret
metadata: { name: web-secret }
type: Opaque
stringData:                 # stringData: K8s tự base64 (KHÔNG phải mã hoá!)
  DB_PASSWORD: "change-me"
```

Inject vào pod:

```yaml
    envFrom:
      - configMapRef: { name: web-config }
    env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef: { name: web-secret, key: DB_PASSWORD }
```

```bash
# Chứng minh Secret chỉ base64 (không mã hoá):
kubectl get secret web-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# -> ra plaintext => cần encryption-at-rest + RBAC + external manager
```

> ⚠️ Đừng commit Secret YAML thô vào Git. `// EncryptionConfiguration cho etcd: verify tại kubernetes.io.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** "Secret = base64, không mã hoá" · namespace cô lập logic (không network) · label là chất keo · etcd = source of truth + quorum lẻ.
- 📌 **Cần THUỘC:** ConfigMap vs Secret · `envFrom`/`secretKeyRef` · NetworkPolicy (để cô lập network) · quorum 3/5 lẻ.
- 🛠️ **Cần LÀM ĐƯỢC:** inject config/secret · chứng minh secret là base64 · giải thích vì sao etcd cần số node lẻ.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Config ra khỏi image; Secret không phải mã hoá; label nối mọi thứ; etcd lẻ để có quorum."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* ConfigMap vs Secret, và bẫy "Secret mã hoá"? → base64; cần encryption-at-rest + RBAC.
2. *(TB)* Namespace cô lập gì, không cô lập gì? → logic; không network (cần NetworkPolicy).
3. *(TB)* Vì sao label là "chất keo"? → Service/Deployment/affinity/NetworkPolicy đều dựa label.
4. *(khó)* Mất etcd thì sao, vì sao số node lẻ? → mất bộ não; Raft cần quorum đa số.

**Thách đố / trick:**
- *"Tôi tạo `kind: Secret` nên dữ liệu được mã hoá rồi đúng không?"* → **Sai**: chỉ base64 trong etcd. Phải bật encryption-at-rest + RBAC; tốt hơn dùng external manager.
- *"3 node etcd chịu được mất mấy node?"* → mất **1** (còn 2 = đa số của 3). 5 node chịu mất 2. (Số chẵn không tăng khả năng chịu lỗi mà còn dễ split.)

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Inject** một ConfigMap + Secret vào Deployment Bài 3.
   - *Đạt khi:* pod đọc được biến; bạn `base64 -d` ra plaintext và giải thích rủi ro.
2. **Giải thích quorum:** vẽ 3-node etcd, mất 1 node — còn hoạt động? mất 2?
   - *Đạt khi:* nói đúng đa số (2/3 ok, 1/3 mất quorum) và vì sao số lẻ.

### ⑩ Đọc thêm / nguồn chuẩn
kubernetes.io (ConfigMap, Secret, Namespaces, Labels, encrypting secret data at rest); etcd.io (quorum, Raft).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI sinh manifest `kind: Secret` và *gọi nó là "mã hoá"*. Đây đúng kiểu *cú pháp đúng, mệnh đề sai* — Tech Lead phải bắt và yêu cầu encryption-at-rest/external secret.

---

## Bài 5 — K8s ops I: probes, requests/limits/QoS, OOMKilled, HPA
*Phủ: N-K8SOPS-001 → 004*

### ① Mục tiêu & vị trí trong mạch
Bắt đầu phần **vận hành production** — phần TL bị vặn nhất. Bài này: **health probes**, **resource requests/limits + QoS**, debug **OOMKilled**, và **HPA** (autoscale). Nền cho rolling/zero-downtime ở Bài 6–7 và Bài 12.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** K8s cần biết pod *còn sống* và *sẵn sàng nhận traffic* (probes); cần biết pod *cần bao nhiêu* và *được dùng tối đa bao nhiêu* tài nguyên (requests/limits) để xếp chỗ và bảo vệ node; và cần *tự thêm/bớt* replica theo tải (HPA).

**(b) Cơ chế.**
- **3 probe (N-K8SOPS-001):**
  - **liveness** = "còn sống không?" → fail thì **restart** pod.
  - **readiness** = "sẵn sàng nhận traffic chưa?" → fail thì **gỡ khỏi Service endpoint** (KHÔNG restart).
  - **startup** = che giai đoạn khởi động chậm khỏi liveness (cho app boot lâu).
- **Requests vs limits + QoS (N-K8SOPS-002):**
  - **request** = scheduler dùng để **xếp pod lên node** (đảm bảo tối thiểu).
  - **limit** = **trần runtime**.
  - Vượt **memory limit → OOMKilled** (exit 137). Vượt **CPU limit → throttle** (chậm, không chết).
  - QoS: `request==limit` (cả CPU+mem) → **Guaranteed** (ít bị evict); có cả nhưng lệch → **Burstable**; không set gì → **BestEffort** (bị evict trước).
- **HPA (N-K8SOPS-004):** tăng/giảm replica theo metric so target. Cần **metrics-server** + pod phải có **resource requests** (HPA tính % theo request). Có cool-down chống flapping.

**(c) Mép giới hạn & sai lầm.**
- **Probe sai (N-K8SOPS-001):** liveness quá nhạy → **restart loop**; thiếu readiness → traffic vào pod chưa sẵn sàng → lỗi 5xx; **liveness dùng chung logic readiness** → restart khi chỉ cần ngừng nhận traffic (vd phụ thuộc DB chậm → restart vô ích).
- **OOMKilled debug (N-K8SOPS-003):** xem `kubectl describe`/last state `reason=OOMKilled` + `exit 137`. Phân biệt: **limit quá thấp** vs **memory leak** vs **spike thật** vs **heap setting lệch** (vd Node `--max-old-space-size`/JVM heap không khớp cgroup limit). Fix: **đo thực tế**, set limit hợp lý, sửa leak, để runtime nhận biết cgroup memory. **Không chỉ "tăng limit" mù.**
- **HPA theo CPU không phản ánh tải thật (N-K8SOPS-004):** CPU không bắt được bottleneck **I/O / queue length / latency** → app nghẽn DB nhưng CPU thấp → HPA không scale. Dùng **custom/external metric** (độ dài queue, RPS, p95 latency). *(nguồn metric → O Observability)*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Dùng 1 probe cho mọi thứ / liveness=readiness | Tách **liveness/readiness/startup** đúng vai | tránh restart loop / traffic vào pod chưa sẵn sàng |
| Không set requests/limits | Set request (mem≈limit), CPU cho burst | scheduler xếp đúng, QoS Guaranteed cho service quan trọng |
| HPA chỉ theo CPU | Custom/external metric (queue/latency/RPS) | CPU không phản ánh I/O-bound |
| "OOM thì tăng limit" | Đo + phân biệt leak/spike/heap-cgroup | tăng mù che leak, tốn tiền |
| (mới) đổi requests phải restart pod | **In-Place Pod Resize GA v1.35** | đổi mem/CPU không restart *(Bài 7)* |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: service API phải **không nhận traffic khi đang warm-up** (readiness chờ DB pool sẵn sàng) và **tự restart khi deadlock** (liveness). Pattern phổ biến (verify): đặt **memory request == limit** cho service quan trọng (QoS Guaranteed), **CPU request thấp + cho burst** (không set CPU limit hoặc đặt cao) để tránh throttle oan; HPA theo **custom metric** qua KEDA/Prometheus Adapter cho workload event-driven. Nhiều nơi dùng **VPA** (đề xuất right-size) + giờ là **in-place resize** (Bài 7).

### ⑤ Code thực hành + cấu hình

```yaml
spec:
  containers:
    - name: web
      image: myrepo/web@sha256:<digest>
      resources:
        requests: { cpu: "100m", memory: "256Mi" }
        limits:   { memory: "256Mi" }     # mem request==limit -> Guaranteed-ish; KHÔNG đặt CPU limit để tránh throttle
      startupProbe:                         # cho app boot chậm
        httpGet: { path: /healthz, port: 8080 }
        failureThreshold: 30
        periodSeconds: 5
      readinessProbe:                       # gỡ khỏi endpoint khi chưa sẵn sàng (vd DB pool)
        httpGet: { path: /ready, port: 8080 }
        periodSeconds: 5
      livenessProbe:                        # restart khi treo thật sự — KHÔNG kiểm tra dependency ngoài
        httpGet: { path: /healthz, port: 8080 }
        periodSeconds: 10
        failureThreshold: 3
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }   # % theo request -> cần requests + metrics-server
```

```bash
kubectl describe pod <pod> | grep -A3 'Last State'   # tìm reason: OOMKilled, exit 137
kubectl top pod                                       # cần metrics-server
kubectl get hpa web                                   # xem current/target + replica
```

> ⚠️ **Bẫy chí mạng:** **liveness probe KHÔNG nên kiểm tra dependency ngoài** (DB/Redis) — DB chậm sẽ làm cả fleet restart loop, biến sự cố nhỏ thành cascade. Dependency check để **readiness**. `// API autoscaling/v2 verify tại kubernetes.io.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** liveness(restart) vs readiness(gỡ endpoint) vs startup · request(schedule) vs limit(trần) · mem→OOMKill / cpu→throttle · QoS Guaranteed/Burstable/BestEffort · vì sao HPA-CPU mù với I/O.
- 📌 **Cần THUỘC:** exit 137 = OOMKilled · `metrics-server` cần cho HPA/`top` · request==limit → Guaranteed · `--max-old-space-size`/heap khớp cgroup.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình 3 probe đúng vai · set requests/limits hợp QoS · debug OOMKilled theo describe/logs · cấu hình HPA.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"liveness=restart, readiness=gỡ traffic (đừng nhầm); request=xếp chỗ, limit=trần; mem vượt→chết, cpu vượt→chậm."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Ba probe khác nhau, mỗi cái fail thì sao? → restart / gỡ endpoint / che boot chậm.
2. *(khó)* Vượt mem limit vs cpu limit khác gì? → OOMKilled vs throttle.
3. *(khó)* QoS class quyết bởi gì, ảnh hưởng evict? → request/limit; Guaranteed ít bị evict.
4. *(khó)* HPA cần gì để chạy, sao CPU đôi khi sai? → metrics-server + requests; I/O-bound cần custom metric.

**Thách đố / trick:**
- *"Pod cứ restart mỗi 30s, app không crash log gì — vì sao?"* → **liveness probe** fail (timeout/đường dẫn sai/check dependency chậm) → K8s restart. Kiểm tra cấu hình probe trước khi đổ lỗi app.
- *"Tăng memory limit gấp đôi mà vẫn OOMKilled định kỳ?"* → khả năng **memory leak** (tăng dần tới trần) hoặc heap runtime không đọc cgroup → đo heap theo thời gian, không tăng mù.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Cấu hình 3 probe** cho một app boot 20s, phụ thuộc DB.
   - *Đạt khi:* startup che boot; readiness check DB; **liveness KHÔNG** check DB; giải thích vì sao.
2. **Tạo OOMKilled có chủ đích** (đặt mem limit thấp) rồi debug.
   - *Đạt khi:* tìm ra `reason=OOMKilled`/exit137 qua `describe`, nêu 2 nguyên nhân gốc khả dĩ.

### ⑩ Đọc thêm / nguồn chuẩn
kubernetes.io (Configure Liveness/Readiness/Startup Probes; Resource Management; HPA); KEDA (keda.sh) cho event-driven autoscaling.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI hay generate **liveness probe gọi `/health` mà `/health` lại ping DB** — cú pháp đúng, nhưng DB chậm → **restart loop toàn fleet**. Hãy chỉ huy: dependency check thuộc readiness, không thuộc liveness.

---

## Bài 6 — K8s ops II: rolling, PDB, workload types, StatefulSet, scheduling
*Phủ: N-K8SOPS-005 → 009*

### ① Mục tiêu & vị trí trong mạch
Tiếp ops: **rolling update zero-downtime**, **PodDisruptionBudget**, chọn **workload type** (Deployment/StatefulSet/DaemonSet/Job/CronJob), vì sao **DB trên K8s khó**, và **scheduling** (taints/affinity). Trực tiếp đỡ cho Bài 12 (deploy strategies).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Cập nhật app mà người dùng **không thấy gián đoạn** = thay pod dần dần, chỉ chuyển traffic khi pod mới **thật sự sẵn sàng**. Đồng thời, khi cluster tự bảo trì (drain node), đừng vô tình gỡ hết replica cùng lúc (PDB). Và phải **chọn đúng loại workload** theo bản chất state.

**(b) Cơ chế.**
- **Rolling zero-downtime (N-K8SOPS-005):** `maxSurge` = số pod thêm tạm; `maxUnavailable` = số pod được phép thiếu. Rollout **chỉ chuyển traffic khi pod mới readiness pass**. Phối với graceful shutdown (Bài 7).
- **PDB (N-K8SOPS-006):** giữ **tối thiểu N pod sẵn sàng** khi có **voluntary disruption** (drain node nâng cấp, autoscaler scale-down). **Involuntary** (node crash, OOM) PDB **không chặn được**.
- **Workload types (N-K8SOPS-007):** **Deployment** = stateless thay thế tự do (API service); **StatefulSet** = cần định danh ổn định + storage bền + thứ tự (DB, Kafka, Zookeeper); **DaemonSet** = 1 pod/node (log/metric agent, CNI); **Job** = chạy tới hoàn thành; **CronJob** = Job theo lịch.
- **StatefulSet & "DB trên K8s khó" (N-K8SOPS-008):** pod tên ổn định `app-0,app-1` + DNS riêng (headless Service), **PVC gắn bền** qua restart, tạo/scale/update **theo thứ tự (ordinal)**. DB cần data persistence + cluster membership + ordered bootstrap + failover → **vận hành phức tạp hơn stateless** → thường cân nhắc **managed DB** hoặc **operator**.
- **Scheduling (N-K8SOPS-009):** **taint** đẩy pod ra khỏi node trừ khi pod có **toleration** (node GPU/dedicated chỉ nhận pod được phép). **node affinity** kéo pod tới node có nhãn. **pod anti-affinity** rải replica ra khác node/zone để chịu lỗi. Phân biệt "đẩy" (taint) vs "hút" (affinity).

**(c) Mép giới hạn & sai lầm.**
- **Rolling đặt sai (N-K8SOPS-005):** `maxUnavailable` cao + **không có readiness** → traffic vào pod chưa sẵn sàng → **downtime**; `maxSurge:0 + maxUnavailable:0` → **kẹt** (không tạo nổi pod mới, không gỡ pod cũ).
- **StatefulSet rolling nguy hiểm (N-K8SOPS-008/N-DEPLOY-010):** rolling ẩu DB cluster có thể **đụng quorum/leader** (mất đa số → cụm down) → cần partition rollout, kiểm tra health từng node, để operator lo.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Recreate / kill-all-restart | **Rolling** + readiness + maxSurge/maxUnavailable | zero-downtime |
| Mặc định Deployment cho mọi thứ | Chọn **theo bản chất state** (Stateful/Daemon/Job/Cron) | DB/agent/batch khác stateless |
| Tự chạy DB bằng StatefulSet "cho tiện" | **Managed DB** hoặc **operator** chuyên dụng | stateful trên K8s vận hành khó (failover/ordered) |
| Không có PDB khi drain node | **PDB** giữ tối thiểu pod | tránh gỡ hết replica lúc maintenance |
| Rải replica "hên xui" | **pod anti-affinity** theo node/zone | chịu lỗi node/zone |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: API stateless 20 replica, rolling update không downtime, rải đều 3 zone (anti-affinity), PDB giữ ≥18 pod khi cloud nâng cấp node. Pattern phổ biến (verify): stateless → Deployment + HPA + anti-affinity theo `topology.kubernetes.io/zone`; **DB hầu hết để managed** (RDS/CloudSQL) thay vì StatefulSet; nếu buộc chạy trên K8s thì dùng **operator** (CloudNativePG, Strimzi cho Kafka...). DaemonSet cho agent log/metric (Fluent Bit, node-exporter).

### ⑤ Code thực hành + cấu hình

```yaml
# Rolling + anti-affinity (stateless)
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }   # an toàn: luôn đủ pod cũ tới khi pod mới Ready
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector: { matchLabels: { app: web } }
                topologyKey: topology.kubernetes.io/zone   # rải khác zone
      containers: [{ name: web, image: myrepo/web@sha256:<digest> }]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: web-pdb }
spec:
  minAvailable: 5                # giữ >=5 pod sẵn sàng khi voluntary disruption
  selector: { matchLabels: { app: web } }
```

```bash
kubectl rollout status deploy/web
kubectl rollout undo deploy/web        # rollback về ReplicaSet cũ
kubectl get pdb web-pdb
kubectl drain <node> --ignore-daemonsets   # PDB sẽ chặn nếu vi phạm minAvailable
```

> ⚠️ `maxUnavailable:0` an toàn nhất cho zero-downtime nhưng cần đủ headroom node cho `maxSurge`. `// topologyKey/labels chuẩn verify tại kubernetes.io.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** rolling cần readiness · voluntary vs involuntary disruption · vì sao DB-on-K8s khó · taint(đẩy) vs affinity(hút) vs anti-affinity(rải).
- 📌 **Cần THUỘC:** `maxSurge`/`maxUnavailable` · `minAvailable` PDB · Deployment/StatefulSet/DaemonSet/Job/CronJob · `topology.kubernetes.io/zone`.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình rolling zero-downtime + anti-affinity + PDB · chọn đúng workload type cho một mô tả · `rollout undo`.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Rolling = thay dần, chỉ khi Ready; PDB = đừng gỡ hết khi bảo trì; chọn workload theo state, không mặc định Deployment."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(khó)* Rolling đạt zero-downtime nhờ gì? → maxSurge/maxUnavailable + readiness.
2. *(khó)* PDB giải quyết gì, chặn được loại disruption nào? → voluntary; không involuntary.
3. *(khó)* Khi nào Deployment vs StatefulSet vs DaemonSet vs Job? → theo bản chất state.
4. *(khó)* Vì sao chạy DB trên K8s khó hơn stateless? → persistence + membership + ordered + failover.

**Thách đố / trick:**
- *"Tôi set `maxUnavailable: 50%` để deploy nhanh, sao có lúc 5xx tăng vọt?"* → nửa fleet bị gỡ + nếu readiness yếu/thiếu → traffic vào pod chưa sẵn sàng. Hạ maxUnavailable, siết readiness.
- *"Có PDB rồi sao node crash vẫn mất nhiều pod cùng lúc?"* → PDB chỉ chặn **voluntary** disruption; node crash là **involuntary** — cần anti-affinity rải zone + dư replica.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Cấu hình rolling zero-downtime** + anti-affinity + PDB cho web 6 replica.
   - *Đạt khi:* `maxUnavailable:0`/hợp lý, anti-affinity theo zone, PDB minAvailable; `drain` node bị PDB chặn đúng lúc.
2. **Bảng quyết định workload:** cho 4 mô tả (API, Postgres, log agent, báo cáo hằng đêm) → chọn loại.
   - *Đạt khi:* Deployment / StatefulSet(or managed) / DaemonSet / CronJob, kèm lý do.

### ⑩ Đọc thêm / nguồn chuẩn
kubernetes.io (Deployments rolling update, PodDisruptionBudget, StatefulSets, DaemonSet, Job/CronJob, Assigning Pods to Nodes); operator: CloudNativePG, Strimzi.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI được hỏi "deploy Postgres lên K8s" thường generate **StatefulSet thẳng** — chạy được nhưng **sai kiến trúc vận hành** (không lo failover/backup). TL chỉ huy: managed DB hoặc operator. *Hiểu định hình yêu cầu.*

---

## Bài 7 — K8s ops III: Ingress/Gateway API, graceful, CronJob, sidecar, debug, resize
*Phủ: N-K8SOPS-010 → 015*

### ① Mục tiêu & vị trí trong mạch
Hoàn tất ops: vào-từ-ngoài (**Ingress → Gateway API**), **graceful shutdown** trong rolling (race SIGTERM↔endpoint), **CronJob** không double-run, **native sidecar GA**, **debug** pod (Pending/CrashLoop/ImagePull), **In-Place Pod Resize GA**. Đây là cụm "đời phiên bản" — verify nhiều.

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Service là L4 nội bộ; muốn **L7 routing + TLS từ ngoài** cần Ingress (đang nhường chỗ Gateway API). Khi tắt pod lúc rolling, có một **race** tinh tế khiến request rớt nếu không xử lý. Và phần lớn thời gian "debug K8s" là đọc đúng 3 trạng thái pod.

**(b) Cơ chế.**
- **Ingress vs Service, Gateway API (N-K8SOPS-010):** Service = L4 trong cluster; **Ingress** = L7 routing (host/path) + TLS termination từ ngoài, cần **Ingress controller** hiện thực. **Landscape 2026:** community **Ingress NGINX đã RETIRED 3/2026** (repo archived) → dịch sang **Gateway API** (v1.0 GA, biểu đạt mạnh hơn, role-oriented: GatewayClass/Gateway/HTTPRoute) hoặc controller còn-sống (Traefik/Contour/Cilium/F5-NGINX). Ingress API `networking.k8s.io/v1` **không bị xoá**, chỉ feature-frozen. *(verify)*
- **Graceful pod shutdown & race (N-K8SOPS-011):** khi xoá pod, K8s **song song** gửi **SIGTERM** và cập nhật **endpoint** — **không đảm bảo thứ tự** → kube-proxy/LB có thể **còn gửi traffic** tới pod đang tắt. Tránh rớt request: thêm **`preStop` sleep nhỏ** (hoặc readiness fail trước) để chờ endpoint rút, **rồi** app mới drain; app phải **bắt SIGTERM drain in-flight** (cơ chế → G-DIAG-009 / H-399); `terminationGracePeriodSeconds` đủ dài hơn drain, nếu không bị **SIGKILL**.
- **CronJob không double-run (N-K8SOPS-012):** CronJob → Job → pod. Để ý `concurrencyPolicy` (Forbid/Replace), **idempotency** của job; nếu logic cron tự viết trong app nhiều replica → cần **leader election/lock** (→ I-170). **Không giả định "chỉ 1 instance".**
- **Native sidecar GA (N-K8SOPS-013):** khai báo ở `initContainers` với `restartPolicy: Always` → **khởi động trước app, sống suốt đời pod, tắt sau app**. Trước GA phải "chế" bằng container thường → app có thể start khi sidecar chưa sẵn sàng, hoặc Job không kết thúc vì sidecar mãi chạy. **GA/stable từ v1.33** (gate on từ v1.29). *(verify)* (mesh/sidecar proxy → K-419)
- **In-Place Pod Resize (N-K8SOPS-015):** trước đây đổi requests/limits → **tạo pod mới (restart)**. **In-Place Resize GA ở v1.35** → đổi CPU/mem **không restart** (đính chính QB). Request quá cao = lãng phí/giảm mật độ/đắt; quá thấp = bị xếp lên node quá tải → evict/throttle. Đo (VPA/observe) để chỉnh.

**(c) Mép giới hạn & debug.**
- **3 trạng thái pod (N-K8SOPS-014):** **Pending** = không schedule được (thiếu tài nguyên/affinity/taint/PVC chưa bind) → `describe` xem Events. **CrashLoopBackOff** = container start rồi chết lặp (lỗi app/config/probe) → `logs`/`logs --previous` + exit code. **ImagePullBackOff** = sai tag/registry/credential → kiểm tra image ref + imagePullSecret. **Quy trình vàng: describe → events → logs.**

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **Ingress NGINX** (community) làm mặc định | **Gateway API** (GA) hoặc controller còn-sống | community ingress-nginx **retired 3/2026** *(verify)* |
| Ingress annotation cho mọi routing nâng cao | Gateway API: HTTPRoute (filter/rewrite/split chuẩn) | role-oriented, biểu đạt mạnh, portable |
| "Chế" sidecar bằng container thường | **Native sidecar** (`initContainers`+`restartPolicy: Always`) | đúng thứ tự start/stop; GA v1.33 *(verify)* |
| Đổi requests/limits → restart pod | **In-Place Pod Resize GA v1.35** | không restart *(verify)* |
| App tắt = chỉ cần bắt SIGTERM | + `preStop` chờ endpoint rút + grace đủ dài | tránh race SIGTERM↔endpoint rớt request |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: edge L7 routing + TLS cho nhiều service; rolling không rớt request; cron báo cáo chạy đúng 1 lần. Pattern phổ biến (verify): nhiều tổ chức đang **migrate Ingress → Gateway API** trong 2026 do EOL của ingress-nginx; service mesh (Istio ambient/Cilium) dùng Gateway API làm north-south. Native sidecar GA khiến Istio/Linkerd bỏ "init-container hack". `preStop: sleep` 5–15s là mẹo chuẩn để né race endpoint.

### ⑤ Code thực hành + cấu hình

Graceful shutdown đúng (N-K8SOPS-011):

```yaml
spec:
  terminationGracePeriodSeconds: 45      # > thời gian drain in-flight
  containers:
    - name: web
      image: myrepo/web@sha256:<digest>
      lifecycle:
        preStop:
          exec: { command: ["sh","-c","sleep 10"] }   # chờ endpoint bị rút trước khi app drain
      readinessProbe: { httpGet: { path: /ready, port: 8080 }, periodSeconds: 5 }
      # App PHẢI bắt SIGTERM: ngừng nhận request mới, drain in-flight, đóng DB/queue (cơ chế -> G/H)
```

Gateway API (kế thừa Ingress) — minh hoạ:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: web-route }
spec:
  parentRefs: [{ name: prod-gateway }]
  hostnames: ["app.example.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: / } }]
      backendRefs: [{ name: web, port: 80 }]
# // Gateway/GatewayClass do platform team định nghĩa; verify apiVersion tại gateway-api.sigs.k8s.io
```

Native sidecar (GA v1.33):

```yaml
spec:
  initContainers:
    - name: log-shipper
      image: fluent/fluent-bit@sha256:<digest>
      restartPolicy: Always        # <- biến init container thành SIDECAR (sống suốt đời pod)
  containers:
    - name: web
      image: myrepo/web@sha256:<digest>
```

```bash
# Debug 3 trạng thái:
kubectl describe pod <p>            # Pending: xem Events (taint/PVC/resource)
kubectl logs <p> --previous        # CrashLoopBackOff: log lần chết trước + exit code
kubectl get pod <p> -o yaml | grep -i image   # ImagePullBackOff: soi image ref/secret
# In-place resize (v1.35+):
kubectl patch pod <p> --subresource=resize --patch '{"spec":{"containers":[{"name":"web","resources":{"requests":{"cpu":"200m"}}}]}}'  # // verify cú pháp tại kubernetes.io
```

> ⚠️ Nếu `preStop` + grace không đủ → SIGKILL cắt giữa request. `// đời K8s cho in-place resize (GA 1.35) và Gateway API verify tại docs chính thức.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** Service(L4) vs Ingress/Gateway(L7) · race SIGTERM↔endpoint + cách né · vì sao native sidecar sửa thứ tự lifecycle · 3 trạng thái pod ánh xạ nguyên nhân.
- 📌 **Cần THUỘC:** `preStop` sleep · `terminationGracePeriodSeconds` · `initContainers`+`restartPolicy: Always` (sidecar) · `concurrencyPolicy` CronJob · `describe→events→logs`.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình graceful shutdown chống rớt request · viết HTTPRoute (Gateway API) · debug Pending/CrashLoop/ImagePull.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Tắt pod = chờ endpoint rút (preStop) RỒI mới drain; debug = describe → events → logs."** Và: *"Ingress NGINX hết đời → Gateway API."*

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(khó)* Vì sao có race khi tắt pod, tránh thế nào? → SIGTERM ∥ xoá endpoint; preStop sleep + readiness fail trước.
2. *(TB)* Service vs Ingress, thay đổi landscape gần đây? → L4 vs L7; Ingress NGINX retired → Gateway API.
3. *(khó)* Native sidecar sửa vấn đề gì? → thứ tự start/stop; init+restartPolicy:Always.
4. *(TB)* Pod Pending/CrashLoop/ImagePull gợi gì? → schedule/lỗi app/lỗi image; describe→logs.
5. *(khó)* In-place resize đổi gì so với trước? → đổi resource không restart (GA v1.35).

**Thách đố / trick:**
- *"App đã bắt SIGTERM drain đàng hoàng mà rolling vẫn rớt vài request — vì sao?"* → **race**: pod nhận SIGTERM trước khi bị gỡ endpoint → LB còn gửi traffic. Thêm `preStop: sleep` để endpoint rút trước.
- *"Interviewer: anh deploy bằng Ingress NGINX chứ?"* → cập nhật: community ingress-nginx **retired 3/2026**; hướng đi **Gateway API** / controller còn-maintained. (Bẫy cập nhật.)

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Chống rớt request khi rolling:** thêm preStop + grace + readiness, mô tả luồng tắt.
   - *Đạt khi:* nêu đúng thứ tự (readiness fail/preStop → endpoint rút → drain → exit) và vì sao grace > drain.
2. **Debug 3 pod hỏng** (Pending/CrashLoop/ImagePull giả lập).
   - *Đạt khi:* mỗi cái dùng đúng `describe`/`logs --previous`/soi image và chỉ ra nguyên nhân.

### ⑩ Đọc thêm / nguồn chuẩn
kubernetes.io (Pod lifecycle/termination, Sidecar containers, In-Place Resize, CronJob, Debug Pods); gateway-api.sigs.k8s.io; blog Ingress NGINX retirement (kubernetes.io 11/2025).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI generate graceful shutdown chỉ "bắt SIGTERM" mà **quên preStop** — đúng phần app, nhưng **race endpoint vẫn rớt request**. TL phải bổ sung preStop + grace. Và phải biết **Ingress NGINX đã EOL** để không copy template chết.

---

## Bài 8 — Packaging: Helm & Kustomize
*Phủ: N-HELM-001 → 005*

### ① Mục tiêu & vị trí trong mạch
Khi có nhiều service × nhiều môi trường, copy-paste YAML là địa ngục. Bài này: **Helm** (templating + packaging + release lifecycle) và **Kustomize** (overlay/patch), khi nào chọn cái nào, và giữ chart "lành mạnh". Nối vào GitOps (Bài 10) và CI/CD (Bài 11).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Helm = "**package manager cho Kubernetes**" (như apt/npm): đóng gói nhóm manifest thành **chart**, tham số hoá qua **values**, cài/nâng cấp/gỡ như **một đơn vị (release)** có version. Kustomize = "base + bản vá", không template-engine.

**(b) Cơ chế.**
- **Helm giải gì (N-HELM-001):** tránh copy-paste YAML cho nhiều env/service; templating + values; đóng gói chart cài/upgrade/uninstall như một release, versioned, share qua registry.
- **Cấu trúc chart (N-HELM-002):** `templates/` = manifest có placeholder (Go template); `values.yaml` = giá trị mặc định; `Chart.yaml` = metadata. **Render** = template + values → YAML thật. **Override thứ tự:** defaults `<` `-f file` `<` `--set` (cái sau thắng).
- **upgrade/rollback (N-HELM-003):** mỗi `helm upgrade` tạo **revision** mới; `helm rollback` về revision trước theo manifest đã lưu → tiện hơn tự nhớ YAML cũ. **Nhưng** vẫn dính vấn đề state/DB (Helm không lo schema — xem Bài 12).
- **Helm vs Kustomize (N-HELM-004):** Helm = templating + packaging + release lifecycle (mạnh cho phân phối/app phức tạp nhiều tham số). Kustomize = base + **overlay patch** YAML thuần, **không template-engine** (đơn giản, "không logic trong YAML"), tích hợp `kubectl -k`. Nhiều nơi **phối cả hai** (Helm render → Kustomize patch).

**(c) Mép giới hạn & sai lầm.**
- **Chart phức tạp (N-HELM-005):** YAML + Go template lồng `if/range/whitespace` → khó đọc/test/debug, **lỗi indent âm thầm**. Giữ chart đơn giản, ít logic; dùng `helm lint` + `helm template` review output; `values.schema.json` validate input; tách phần phức tạp. **Ưu tiên rõ ràng hơn "thông minh".**

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Copy-paste YAML mỗi env | **Helm values** hoặc **Kustomize overlay** | DRY, ít lỗi |
| Helm 2 (Tiller server-side) | **Helm 3** (client-only, không Tiller) | bỏ thành phần đặc quyền trong cluster |
| Nhồi logic `if/range` vào template | Chart đơn giản + schema + (đôi khi) Kustomize | dễ test/đọc, tránh indent bug |
| Rollback bằng "nhớ lại YAML cũ" | `helm rollback <revision>` / GitOps revert | có lịch sử revision |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: 1 chart `web` dùng cho dev/staging/prod chỉ khác `values-<env>.yaml`. Pattern phổ biến (verify): Helm để **đóng gói/phân phối** (chart công khai như Bitnami, Prometheus stack); Kustomize cho **tuỳ biến môi trường** trong GitOps (ArgoCD/Flux hỗ trợ cả hai). Nhiều platform team chuẩn hoá một "golden chart" nội bộ + để app team chỉ chỉnh values (mầm platform engineering — Bài 14).

### ⑤ Code thực hành + cấu hình

```bash
helm create web                      # khung chart
helm lint web                        # bắt lỗi sớm
helm template web -f web/values-prod.yaml   # render ra YAML để REVIEW trước khi apply
helm upgrade --install web ./web -f web/values-prod.yaml -n prod   # idempotent
helm history web -n prod             # xem revision
helm rollback web 3 -n prod          # về revision 3
```

Kustomize overlay (không template-engine):

```yaml
# base/kustomization.yaml -> deployment.yaml + service.yaml
# overlays/prod/kustomization.yaml
resources: [ ../../base ]
replicas:
  - name: web
    count: 6
images:
  - name: myrepo/web
    digest: sha256:<digest>      # pin digest cho prod
patches:
  - path: resources-patch.yaml   # patch requests/limits cho prod
```

```bash
kubectl apply -k overlays/prod    # Kustomize tích hợp sẵn kubectl
```

> ⚠️ Luôn `helm template`/`kubectl kustomize` **review diff** trước khi apply prod. `// Helm/Kustomize CLI verify tại helm.sh / kustomize.io.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** Helm = template+package+release vs Kustomize = base+overlay · vì sao "logic trong YAML" nguy hiểm · revision rollback vẫn không lo DB.
- 📌 **Cần THUỘC:** `Chart.yaml`/`values.yaml`/`templates/` · override defaults<`-f`<`--set` · `helm lint`/`template`/`rollback` · `kubectl apply -k`.
- 🛠️ **Cần LÀM ĐƯỢC:** tham số hoá 1 chart cho nhiều env · render review trước apply · chọn Helm vs Kustomize cho một tình huống.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Helm = đóng gói + template; Kustomize = vá overlay; review render trước khi apply; rollback Helm không cứu schema DB."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Helm giải vấn đề gì? → templating + packaging + release lifecycle.
2. *(TB)* values override theo thứ tự nào? → defaults < `-f` < `--set`.
3. *(khó)* Helm vs Kustomize, khi nào dùng cái nào? → packaging/phân phối vs tuỳ biến env đơn giản.
4. *(khó)* Rủi ro chart phức tạp, giữ lành mạnh sao? → indent/logic; lint+template+schema, ít logic.

**Thách đố / trick:**
- *"`helm rollback` về revision cũ là an toàn tuyệt đối?"* → **Không** — Helm chỉ revert *manifest*, **không revert migration DB/schema** đã chạy (xem Bài 12). Rollback "sạch" cần migration backward-compatible.
- *"Template Helm của tôi render ra YAML lỗi indent ngẫu nhiên?"* → whitespace control của Go template (`{{-`/`-}}`); dùng `helm template` soi output, thêm `values.schema.json`.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Đóng gói** Deployment+Service Bài 3 thành chart, chạy dev & prod khác values.
   - *Đạt khi:* 1 chart + 2 values; `helm template` ra đúng khác biệt (replicas/digest/resources).
2. **So sánh** làm lại bằng Kustomize base+overlay.
   - *Đạt khi:* giải thích trade-off (templating vs patch) cho tình huống của bạn.

### ⑩ Đọc thêm / nguồn chuẩn
helm.sh/docs; kubectl.docs.kubernetes.io (Kustomize); ArgoCD/Flux docs (hỗ trợ Helm & Kustomize).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI hay sinh chart "thông minh" đầy `if/range` — render được nhưng **bẫy indent + khó review**. TL chỉ huy: chart đơn giản, `helm template` review, schema validate.

---

## Bài 9 — Infrastructure as Code: Terraform / OpenTofu
*Phủ: N-IAC-001 → 008*

### ① Mục tiêu & vị trí trong mạch
Hạ tầng (cluster/VPC/IAM/DB) cũng nên là **code**. Bài này: vì sao IaC, **state** (và vì sao mất nó là thảm hoạ), `plan` vs `apply` + idempotency, **drift**, declarative vs imperative, **module/env**, **OpenTofu**, và ranh giới **Terraform lo cluster / GitOps lo workload** (consensus 2026). Nối vào GitOps (Bài 10) & cloud (Bài 13).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Thay vì bấm console (không ai nhớ đã bấm gì), bạn **mô tả hạ tầng mong muốn bằng code**, review qua PR, dựng lại được sau thảm hoạ. Terraform giữ một **state** ánh xạ "code ↔ resource thật".

**(b) Cơ chế.**
- **IaC giải gì + snowflake (N-IAC-001):** hạ tầng = code → versioned, review PR, reproducible, tự tài liệu, dựng lại sau thảm hoạ. Tránh **snowflake server** (cấu hình tay không ai biết khác chỗ nào) và config drift. Declarative desired-state.
- **State (N-IAC-002):** state = ánh xạ resource khai báo ↔ resource thật trên cloud. **Mất state → Terraform "tưởng chưa có gì"** → định tạo lại/không quản được cái đang tồn tại. Production cần **remote backend + state locking** (S3+DynamoDB lock / Terraform Cloud / OpenTofu remote) để team chia sẻ + **lock chống 2 người apply đồng thời** làm hỏng state. State **chứa secret** → phải mã hoá.
- **plan vs apply + idempotency (N-IAC-003):** `plan` = tính **diff** desired vs actual, **không đổi gì** → review. `apply` = thực thi diff. Chạy lại apply **không tạo trùng** (idempotent — chỉ làm phần lệch). **Luôn xem plan** để bắt destroy nhầm trước khi chạm prod.
- **Drift (N-IAC-004):** thực tế lệch khỏi state/code do thay đổi **out-of-band** (ai đó sửa tay console). `plan` lộ drift (sẽ "sửa lại" về code). Nguy hiểm: **code không còn là source of truth**; apply sau có thể **đè/đập thứ ai đó sửa gấp**. Kỷ luật: mọi thay đổi qua code, hạn chế quyền sửa tay.
- **Terraform vs Ansible (N-IAC-005):** Terraform = **provisioning** hạ tầng (tạo VM/network/cluster, declarative); Ansible = thiên **configuration management** (cài/cấu hình *bên trong* máy, push step). Thường phối: Terraform dựng, Ansible/cloud-init cấu hình. Container hoá làm phần config nhẹ đi.
- **Module & env (N-IAC-006):** tách **module** tái dùng + truyền biến/workspace per env; **state khổng lồ/monolithic** → apply chậm, **blast radius lớn** (đổi nhỏ động cả prod), khó phân quyền → tách state theo domain/độ-thay-đổi.

**(c) Mép giới hạn & quyết định.**
- **Terraform lo cluster, GitOps lo workload (N-IAC-007):** consensus 2026: hai **tốc độ thay đổi khác nhau** (infra chậm/ít; app deploy liên tục). Terraform `apply` chậm + state lock **không hợp deploy app nhiều lần/ngày**. Tách: **Terraform tới biên cluster (VPC/IAM/EKS)**, **ArgoCD/Flux reconcile workload** từ Git → app rollout nhanh, an toàn, không trộn blast radius infra với app; tránh circular dependency.
- **OpenTofu (N-IAC-008):** HashiCorp đổi Terraform sang **license BSL (2023)** → cộng đồng fork **OpenTofu** (Linux Foundation, open source, tương thích phần lớn HCL). Team cân nhắc license/governance/feature. *Đây là yếu tố license & ecosystem, không phải khác biệt kỹ thuật lớn lúc đầu.* *(verify trạng thái hiện tại tại opentofu.org)*

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Click console / script tay | **IaC declarative** (Terraform/OpenTofu) | versioned, reproducible, chống snowflake |
| Local state file | **Remote backend + state locking** | team share + chống apply đồng thời hỏng state |
| `apply` thẳng không xem plan | **`plan` review** rồi apply | bắt destroy nhầm/drift trước prod |
| Terraform deploy cả app | **Terraform=cluster, GitOps=workload** | hai tốc độ thay đổi; tránh blast radius lẫn lộn |
| Chỉ biết Terraform | Biết cặp **Terraform / OpenTofu** | license BSL 2023 → fork OpenTofu *(verify)* |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: dựng VPC + EKS + RDS + IAM bằng Terraform (state ở S3+DynamoDB lock), còn app deploy bằng ArgoCD. Pattern phổ biến (verify): module hoá (`terraform-aws-modules`), tách state theo môi trường/domain, **policy-as-code** (Sentinel/OPA) trong CI để gate `plan`. Tranh luận Terraform↔OpenTofu vẫn tiếp diễn; nhiều tổ chức/đám mây đã thêm hỗ trợ OpenTofu — *verify lựa chọn hiện tại của team*.

### ⑤ Code thực hành + cấu hình

```hcl
# backend.tf — remote state + locking (sống còn ở prod)
terraform {
  required_version = ">= 1.6"           # // verify: Terraform vs OpenTofu version
  backend "s3" {
    bucket         = "acme-tfstate"
    key            = "prod/eks.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "tf-lock"           # state locking
    encrypt        = true                # state chứa secret -> mã hoá
  }
}
```

```hcl
# Module tái dùng + biến per-env
module "eks" {
  source       = "./modules/eks"
  cluster_name = var.cluster_name
  environment  = var.environment        # dev/staging/prod
  node_count   = var.node_count
}
```

```bash
terraform init
terraform plan -out=tfplan     # XEM diff trước (không đổi gì)
terraform apply tfplan         # chỉ apply đúng plan đã review (idempotent)
terraform plan                 # chạy lại để phát hiện DRIFT (nếu ai sửa tay)
# OpenTofu: thay 'terraform' bằng 'tofu' (tương thích phần lớn) // verify
```

> ⚠️ **Đừng commit state file vào Git** (chứa secret) — dùng remote backend mã hoá. `// license/feature Terraform vs OpenTofu verify tại nguồn chính thức.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** state = map code↔thật; mất state = thảm hoạ · drift & vì sao nguy với "code là source of truth" · provisioning vs config management · Terraform=cluster/GitOps=workload.
- 📌 **Cần THUỘC:** remote backend + **state locking** · `plan` (không đổi) vs `apply` · idempotent · module/workspace · Terraform **BSL 2023** → **OpenTofu**.
- 🛠️ **Cần LÀM ĐƯỢC:** cấu hình remote state+lock · đọc `plan` để bắt destroy nhầm · tách module/env tránh state monolithic.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Hạ tầng = code; state là báu vật (remote+lock+encrypt); luôn plan trước apply; Terraform tới biên cluster, GitOps lo bên trong."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* IaC giải gì so với click console? snowflake là gì? → reproducible/versioned; server cấu hình tay không ai biết.
2. *(khó)* State là gì, vì sao cần remote+lock? → map code↔thật; share + chống apply đồng thời.
3. *(khó)* `plan` vs `apply`, idempotency ở đâu? → diff vs thực thi; chỉ làm phần lệch.
4. *(khó)* Vì sao 2026 tách Terraform=cluster / GitOps=workload? → hai tốc độ thay đổi; tránh state lock cho app.
5. *(TB)* Vì sao có OpenTofu? → BSL 2023 → fork Linux Foundation *(verify)*.

**Thách đố / trick:**
- *"Đồng nghiệp sửa security group trên console cho nhanh; lần `apply` sau của tôi 'mất' thay đổi đó — vì sao?"* → **drift**: code là source of truth; apply đưa về code, đè sửa tay. Phải sửa qua code hoặc import.
- *"Hai người cùng `apply` một lúc thì sao?"* → không có lock → **corrupt state**; bắt buộc remote backend + state locking.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Cấu hình remote state + lock** cho một module nhỏ.
   - *Đạt khi:* backend S3+DynamoDB (encrypt), giải thích vì sao lock; `plan` không đổi gì.
2. **Tạo drift** (sửa tay 1 resource) rồi `plan`.
   - *Đạt khi:* plan lộ drift, giải thích rủi ro "code không còn là source of truth".

### ⑩ Đọc thêm / nguồn chuẩn
developer.hashicorp.com/terraform (state, backends, modules); opentofu.org; "Terraform: Up & Running" (Brikman).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI hay generate Terraform với **local state** và `apply` không plan — chạy được demo nhưng **mất state/đè drift ở team thật**. TL chỉ huy: remote+lock+encrypt, plan-review-gate.

---

## Bài 10 — GitOps: ArgoCD/Flux, pull vs push, drift/self-heal
*Phủ: N-GITOPS-001 → 006*

### ① Mục tiêu & vị trí trong mạch
Kết tinh "deploy theo Git": Git là source of truth, operator trong cluster **reconcile** liên tục. Bài này: nguyên lý GitOps, **pull vs push** (điểm bảo mật hiring manager hay dò), ArgoCD vs Flux, **drift/self-heal**, luồng **CI→GitOps**, rollback. Trực tiếp nối CI/CD (Bài 11).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Thay vì "CI bắn `kubectl apply` vào cluster", bạn **commit trạng thái mong muốn vào Git**; một **operator trong cluster** (ArgoCD/Flux) liên tục so cluster vs Git và **kéo cluster về khớp Git**. Deploy = commit/PR (review, audit, revert).

**(b) Cơ chế.**
- **Nguyên lý (N-GITOPS-001):** desired state khai báo trong Git; operator **continuous reconciliation** so cluster vs Git, kéo về khớp; **không `kubectl apply` tay**.
- **Pull vs push + bảo mật (N-GITOPS-002):** **push** = CI (Jenkins/Actions) chạy `kubectl/helm` đẩy vào cluster → **CI phải giữ cluster credential** (blast radius lớn nếu rò). **pull** = operator *trong cluster* tự kéo từ Git → **CI KHÔNG cần credential cluster**; cluster chủ động → **giảm bề mặt tấn công**, tách CI khỏi quyền deploy. (Đây là điểm hay bị dò.)
- **ArgoCD vs Flux (N-GITOPS-003):** cùng GitOps operator cho K8s. **ArgoCD** có UI + CRD `Application` rõ; **Flux** nhẹ, GitOps Toolkit, CLI-first. **Bẫy trung thực:** đừng phóng đại "thạo cả hai" — nếu sâu ArgoCD, nông Flux thì nói thật (interviewer kiểm tra độ trung thực).
- **Drift/self-heal (N-GITOPS-004):** operator phát hiện cluster lệch Git → đánh dấu **OutOfSync** và (nếu auto-sync/self-heal) **đưa về đúng Git** → thay đổi tay bị "nuốt". Lợi: chống drift, mọi thay đổi qua Git. Lưu ý: sửa khẩn cấp tay sẽ bị revert → hotfix qua Git hoặc **tạm tắt auto-sync có kiểm soát**.
- **Luồng CI→GitOps & tách repo (N-GITOPS-005):** CI build/test/push image → **cập nhật tag image trong deploy repo** (image updater/commit/PR) → operator thấy commit → reconcile. Tách **app repo** (source code) và **deploy/config repo** (source of truth của *trạng thái mong muốn*) → audit/rollback bằng git, tách quyền & lịch sử.

**(c) Mép giới hạn.**
- **Rollback GitOps (N-GITOPS-006):** `git revert` về commit/manifest trước → operator reconcile về trạng thái cũ; có **lịch sử/audit** ai-đổi-gì-khi-nào, không "bấm nút bí ẩn". **Vẫn dính giới hạn state/DB** (rollback schema khó — Bài 12).

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| **Push**: CI `kubectl apply` vào cluster | **Pull**: ArgoCD/Flux reconcile từ Git | CI không giữ cluster credential → ít blast radius |
| `kubectl apply` tay lên prod | Commit/PR → operator sync | audit/review/revert; chống drift |
| Sửa tay trên cluster | Sửa qua Git (self-heal nuốt sửa tay) | Git là single source of truth |
| App & deploy chung 1 repo | Tách **app repo / deploy repo** | tách quyền, lịch sử, rollback bằng git |
| "Bấm nút rollback" mơ hồ | `git revert` (có audit) | minh bạch ai-đổi-gì |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: 50 service, mỗi deploy = PR vào deploy repo; ArgoCD auto-sync về cluster; rollback = `git revert`. Pattern phổ biến (verify): **ArgoCD** rất phổ biến cho UI/multi-cluster; **Flux** mạnh cho GitOps-as-toolkit/automation; nhiều nơi dùng **image updater** (Argo Image Updater / Flux image automation) để tự bump tag sau CI. GitOps là lõi của nhiều **Internal Developer Platform** (Bài 14).

### ⑤ Code thực hành + cấu hình

```yaml
# ArgoCD Application — operator KÉO từ Git và reconcile
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: web, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://git.example.com/acme/deploy.git   # DEPLOY repo (tách khỏi app repo)
    targetRevision: main
    path: apps/web/overlays/prod        # Kustomize/Helm path
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated: { prune: true, selfHeal: true }   # tự xoá thứ thừa + kéo drift về Git
# // apiVersion argoproj verify tại argo-cd.readthedocs.io
```

```bash
# Rollback = revert Git (có audit), KHÔNG kubectl tay:
git revert <bad-commit> && git push     # operator reconcile về trạng thái cũ
argocd app get web                       # xem Synced/OutOfSync
argocd app history web
```

> ⚠️ Với `selfHeal: true`, `kubectl edit` tay trên prod sẽ **bị revert** → hotfix phải qua Git hoặc tạm tắt auto-sync có kiểm soát. `// flags ArgoCD verify tại docs.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** continuous reconciliation · **pull > push về bảo mật** (CI không giữ cluster cred) · self-heal nuốt sửa tay · tách app/deploy repo · rollback = git revert.
- 📌 **Cần THUỘC:** ArgoCD `Application` + `selfHeal/prune` · Flux (toolkit) · OutOfSync · image updater.
- 🛠️ **Cần LÀM ĐƯỢC:** mô tả luồng CI→deploy repo→reconcile · giải thích vì sao pull an toàn hơn · rollback bằng git revert.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Git là sự thật; operator kéo cluster về Git (pull, không push); deploy = commit, rollback = revert."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* GitOps ở mức nguyên lý là gì? → desired state trong Git + reconcile liên tục.
2. *(khó, hay hỏi)* Pull vs push, vì sao pull an toàn hơn? → CI không giữ cluster credential; cluster chủ động kéo.
3. *(TB)* ArgoCD vs Flux + bẫy trung thực? → UI/Application vs toolkit/CLI; nói thật độ thạo.
4. *(khó)* Self-heal làm gì khi `kubectl edit` tay? → đánh OutOfSync → revert về Git.

**Thách đố / trick:**
- *"GitOps thì CI cần quyền gì trên cluster?"* → (pull-based) **gần như không** — CI chỉ push image + commit deploy repo; operator trong cluster mới có quyền apply. Đây là điểm bảo mật cốt lõi.
- *"Tôi fix khẩn cấp bằng `kubectl edit`, lát sau nó tự mất?"* → **self-heal** revert về Git. Hotfix đúng cách = commit vào Git (hoặc tạm tắt auto-sync có kiểm soát).

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Vẽ luồng** CI build image → bump tag ở deploy repo → ArgoCD sync → rollback bằng revert.
   - *Đạt khi:* nêu đúng tách 2 repo + operator pull + CI không giữ cluster cred.
2. **Giải thích self-heal:** chuyện gì khi ai đó scale tay replica trên prod?
   - *Đạt khi:* OutOfSync → bị kéo về số trong Git; cách hotfix đúng.

### ⑩ Đọc thêm / nguồn chuẩn
argo-cd.readthedocs.io; fluxcd.io; opengitops.dev (nguyên lý GitOps); CNCF GitOps WG.

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** Hỏi AI "thiết kế deploy K8s", nó thường mặc định **push** (CI `kubectl apply`) — chạy được nhưng **CI ôm cluster credential** (blast radius). TL chỉ huy pull-based GitOps. *Hiểu định hình kiến trúc.*

---

## Bài 11 — CI/CD pipeline
*Phủ: N-CICD-001 → 010*

### ① Mục tiêu & vị trí trong mạch
Ghép mọi thứ thành **đường ống tự động** từ commit → prod. Bài này: CI vs Delivery vs Deployment, **stage order (fail-fast)**, **build-once-promote**, secret/**OIDC keyless**, cache, **quality gates**, mono/polyrepo, **trunk-based**, **DORA metrics**. Dùng GitOps (Bài 10) làm "phần deploy".

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** CI = "mỗi thay đổi tự build/test". CD = "luôn deploy được" (Delivery: người bấm nút cuối; Deployment: tự bấm nếu qua gate). Pipeline tốt = **fail nhanh & rẻ trước**, build **một artifact** rồi promote.

**(b) Cơ chế.**
- **CI vs Delivery vs Deployment (N-CICD-001):** CI = merge thường + tự build/test. **Continuous Delivery** = luôn ở trạng thái deploy-được nhưng **bấm nút thủ công** lên prod. **Continuous Deployment** = **tự động** lên prod nếu qua gate. Khác nhau ở "ai bấm nút cuối".
- **Stage order fail-fast (N-CICD-002):** lint/format → build → unit test → security/dependency scan → build image → integration test → push registry → deploy staging → smoke/e2e → promote prod. Xếp **rẻ-và-hay-fail trước** (lint/unit) để fail-fast, khâu **đắt sau** (e2e/load). Mỗi stage là **gate**.
- **Build once, promote same artifact (N-CICD-003):** build **1 artifact bất biến** (image theo digest) rồi promote dev→staging→prod. Build lại mỗi env → artifact prod **không phải** cái đã test (deps/base đổi, non-reproducible) → "works in staging, breaks in prod". **Config khác nhau inject lúc runtime**, không rebuild.
- **Secret/OIDC keyless (N-CICD-004):** không hardcode trong YAML/log; dùng secret store của CI; tốt nhất **OIDC**: runner đổi **token ngắn hạn** lấy quyền cloud theo job → **không có long-lived key để rò**; least-privilege per pipeline; mask log; xoay vòng. (Cùng tư duy IRSA — Bài 13.)
- **Cache CI (N-CICD-005):** cache `~/.npm`/`node_modules`/build output + Docker layer cache → bỏ tải/biên dịch lại. Rủi ro: cache **stale/bẩn** (key sai → deps cũ, ẩn lỗi lockfile) → key theo **hash lockfile** + invalidation đúng.

**(c) Mép giới hạn & quyết định.**
- **Quality gates (N-CICD-006):** unit+integration pass, coverage/mutation ngưỡng, lint, security/dependency scan, **contract test "can-i-deploy"**, smoke sau deploy. Gate = điểm chặn; *bản chất từng loại test* → trỏ L. Cân tốc độ feedback vs độ chặt.
- **Mono vs polyrepo (N-CICD-007):** polyrepo = mỗi service pipeline riêng, trigger theo repo. Monorepo = **path filter / affected detection** để chỉ chạy phần đổi (tránh build cả thế giới); shared step tách **reusable workflow**. Trade-off: monorepo dễ refactor xuyên service nhưng cần tooling lọc.
- **CI/CD cho ~50 microservices (N-CICD-008):** quyết mono/poly trước; pipeline per-service (poly) hoặc path-filter (mono); reusable templates; **deploy qua GitOps** (pipeline không giữ cluster credential); versioning/contract giữa service; rollout độc lập; chuẩn hoá để onboard nhanh (mầm platform engineering — Bài 14). *(independent deploy → K)*
- **Trunk-based vs Gitflow (N-CICD-009):** trunk-based = branch ngắn, merge thường, **feature flag** che việc dở → ít conflict, deploy thường, lô nhỏ dễ rollback. Branch sống lâu = merge hell + tích hợp muộn + lô to khó debug. Gitflow nặng release branch hợp release chậm/đóng gói.
- **DORA metrics (N-CICD-010):** **deployment frequency, lead time for changes, change failure rate, MTTR**. Deploy thường = lô nhỏ → ít thay đổi/lần → dễ test/rollback/định vị lỗi → **CFR thấp, MTTR ngắn**. Đo *outcome*, không chỉ hoạt động.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| Build lại image cho mỗi env | **Build once, promote** (digest) | artifact prod = cái đã test |
| Long-lived cloud access key trong CI | **OIDC keyless** (token ngắn hạn) | không có key để rò |
| Gitflow + release branch dài | **Trunk-based** + feature flag | deploy thường, lô nhỏ, ít conflict |
| Test nặng chạy trước | **Fail-fast**: lint/unit trước, e2e sau | feedback nhanh, rẻ |
| Pipeline ôm cluster credential để `kubectl apply` | **Deploy qua GitOps** | tách CI khỏi quyền deploy *(Bài 10)* |
| Đo "số build" | **DORA** (freq/lead/CFR/MTTR) | đo outcome delivery |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: PR → GitHub Actions lint/test/scan/build image → push GHCR → bump tag deploy repo → ArgoCD sync staging → smoke → promote prod. Pattern phổ biến (verify): **GitHub Actions OIDC → AWS/GCP** (không cần lưu key) là chuẩn mới; reusable workflows cho step chung; **monorepo + affected** (Nx/Turborepo/Bazel) ở nhiều big-tech; DORA dùng để đo sức khoẻ delivery (báo cáo DORA/Accelerate). Trunk-based + feature flag (LaunchDarkly/OpenFeature) rất phổ biến.

### ⑤ Code thực hành + cấu hình

GitHub Actions với **OIDC keyless** + build-once (minh hoạ):

```yaml
# .github/workflows/ci.yml
name: ci
on: { push: { branches: [main] } }
permissions:
  id-token: write       # cho OIDC
  contents: read
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }   # cache theo lockfile
      - run: npm ci
      - run: npm run lint && npm test             # fail-fast: rẻ trước
      - run: trivy fs --severity HIGH,CRITICAL --exit-code 1 .   # security gate // verify
      # OIDC: đổi token ngắn hạn lấy quyền cloud (KHÔNG lưu access key)
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/ci-ecr-push
          aws-region: ap-southeast-1
      - run: |
          IMAGE=123.dkr.ecr.ap-southeast-1.amazonaws.com/web:${{ github.sha }}
          docker build -t $IMAGE .          # build MỘT artifact
          docker push $IMAGE
          # -> bump tag (digest) trong DEPLOY repo; ArgoCD sẽ promote (GitOps)
# // tên action/version verify tại github docs
```

> ⚠️ Không in secret ra log; mask token; least-privilege cho role CI. Promote = **trỏ cùng digest** sang env sau, không rebuild.

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** Delivery (người bấm) vs Deployment (tự bấm) · fail-fast ordering · build-once-promote · OIDC keyless · gate · DORA → vì sao deploy-thường an toàn hơn · trunk-based + flag.
- 📌 **Cần THUỘC:** 4 DORA metrics · OIDC vs long-lived key · path filter (monorepo) · cache key theo lockfile · contract "can-i-deploy".
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế stage order · cấu hình OIDC + build-once · đặt quality gates · chọn mono/poly cho một tình huống.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Fail rẻ trước; build một lần rồi promote; deploy thường + lô nhỏ = an toàn (DORA); CI đẩy image, GitOps lo deploy."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(dễ)* CI vs Delivery vs Deployment? → ranh giới ở "ai bấm nút cuối".
2. *(khó)* Vì sao build-once-promote quan trọng? → prod = artifact đã test; rebuild ≠ reproducible.
3. *(khó)* Vì sao OIDC keyless tốt hơn access key? → token ngắn hạn, không có key dài hạn để rò.
4. *(khó)* 4 DORA metrics, vì sao deploy-thường an toàn hơn? → lô nhỏ → CFR thấp, MTTR ngắn.
5. *(khó)* Mono vs polyrepo ảnh hưởng pipeline? → path filter/affected vs pipeline per-repo.

**Thách đố / trick:**
- *"Staging xanh mà prod vỡ — pipeline build image riêng cho từng env, sai ở đâu?"* → **không build-once**: prod image khác staging (deps/base đổi). Build 1 artifact, promote cùng digest, inject config runtime.
- *"CI của tôi lưu AWS access key trong secret — đủ an toàn chưa?"* → long-lived key vẫn rò được; chuyển **OIDC** (token ngắn hạn, least-privilege per job).

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Thiết kế pipeline** backend với đúng thứ tự fail-fast + gates + build-once.
   - *Đạt khi:* lint/unit trước e2e; security scan gate; 1 image promote qua env; deploy qua GitOps.
2. **Chuyển CI sang OIDC** (mô tả) thay vì lưu access key.
   - *Đạt khi:* nêu id-token/role-to-assume + vì sao bớt rủi ro rò key.

### ⑩ Đọc thêm / nguồn chuẩn
DORA / "Accelerate" (Forsgren, Humble, Kim); docs GitHub Actions OIDC; trunkbaseddevelopment.com; martinfowler.com (CI/CD).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** AI hay sinh workflow **build image trong mỗi job/env** và **lưu access key** — chạy được nhưng **non-reproducible + rủi ro key**. TL chỉ huy: build-once-promote + OIDC.

---

## Bài 12 — Deployment strategies & zero-downtime DB migration
*Phủ: N-DEPLOY-001 → 010*

### ① Mục tiêu & vị trí trong mạch
Phần "ra quyết định deploy" — TL hay bị vặn nhất cùng Bài 5–7. Recreate/rolling/**blue-green**/**canary**, **rollback khi đã chạy migration DB**, **expand/contract**, **feature flag**, điều kiện **zero-downtime**, incident khi **rollback cũng fail**, deploy **StatefulSet**. Dựa trên probe/graceful (Bài 5–7) và GitOps/CI (Bài 10–11).

### ② Giảng cơ bản → nâng cao

**(a) Trực giác.** Có nhiều cách "đổi từ bản cũ sang mới": thay hết (recreate), thay dần (rolling), bật cả môi trường mới rồi cắt (blue-green), nhỏ giọt theo % (canary). Cái khó nhất **không phải code** mà là **DB/state** — vì hai bản code thường phải sống chung một lúc.

**(b) Cơ chế.**
- **Recreate vs Rolling (N-DEPLOY-001):** recreate = kill hết cũ rồi dựng mới (**có downtime**, đơn giản); rolling = thay dần (**không downtime**). Recreate cần khi cũ/mới **không chạy đồng thời được** (lock độc quyền, schema không tương thích cùng lúc).
- **Blue-green (N-DEPLOY-002):** dựng môi trường **green** song song **blue**, test xong **cắt toàn bộ traffic** sang green; rollback = trỏ lại blue **tức thì**. Nhược: **tốn gấp đôi tài nguyên** lúc chuyển; **DB/state dùng chung** vẫn phải tương thích cả hai bản.
- **Canary (N-DEPLOY-003):** đẩy bản mới cho **% traffic nhỏ**, tăng dần nếu khoẻ. Khác blue-green (không cắt 100% ngay). **Metric-gated canary** tự promote/rollback theo **error rate/latency/SLO** so baseline → cần **observability + traffic splitting** (mesh/ingress/Argo Rollouts).
- **Chọn chiến lược (N-DEPLOY-004):** rolling = mặc định rẻ, rollback chậm hơn, hai bản cùng tồn tại; blue-green = rollback tức thì nhưng tốn tài nguyên + cutover rủi ro; canary = an toàn nhất cho thay đổi rủi ro cao nhưng cần đo lường + hạ tầng traffic-split. Chọn theo **rủi ro thay đổi + chi phí + khả năng quan sát**.
- **Feature flag (N-DEPLOY-007):** **deploy code (tắt flag) ≠ release tính năng**. Cho canary theo user/%, **kill-switch tức thì**, A/B. Giá: phức tạp test (tổ hợp flag), **flag debt** nếu không dọn, runtime config phải tin cậy.
- **Zero-downtime conditions (N-DEPLOY-008):** readiness đúng, graceful drain (Bài 7), connection draining ở LB, idempotent retry, và **trong lúc rolling hai phiên bản chạy đồng thời** → API/contract/schema/message **tương thích ngược** (client cũ gọi server mới và ngược lại). *(versioning → F)*

**(c) Mép giới hạn — phần khó nhất: DB & incident.**
- **Rollback code vs schema (N-DEPLOY-005):** code rollback nhanh (đổi image), nhưng schema đã đổi (drop/rename cột, đổi type) **không lùi sạch** + dữ liệu bản mới ghi có thể không hợp bản cũ → **rollback code mà schema mới = vỡ**. Phải thiết kế migration **backward-compatible**.
- **Expand/contract (N-DEPLOY-006):** **expand** (thêm cột/bảng mới; code mới ghi cả cũ+mới hoặc đọc cả hai) → deploy code đọc/ghi tương thích → **backfill** dữ liệu → **contract** (xoá cũ) ở release sau. **Không bao giờ "đổi tên cột + đổi code" trong một bước**; mỗi bước hai bản code phải cùng chạy được. *(consistency → I; Saga/Outbox → K)*
- **Rollback cũng fail (N-DEPLOY-009):** **cầm máu trước** (chuyển traffic về env/region khoẻ, bật maintenance/kill-switch) → đánh giá hệ đang ở state nào (giữa chừng? partial?) → **không làm tệ hơn** → khôi phục dịch vụ → **giao tiếp stakeholder** → ổn rồi mới postmortem **không đổ lỗi** + thêm rào chặn tái diễn. *Tư duy incident, không "thử lại bừa".*
- **StatefulSet deploy (N-DEPLOY-010):** update **có thứ tự (ordinal)**, pod giữ identity/PVC; rolling ẩu có thể **đụng quorum/leader** (mất đa số → cụm down) hoặc schema/compat data; cần partition rollout, health từng node, để **operator** lo.

### ③ ⚠️ Kiến thức cũ / bị thay thế

| Cái cũ | Thay bằng (hiện nay) | Vì sao |
|---|---|---|
| "Đổi tên cột + đổi code" 1 bước | **Expand/contract** nhiều release | rollback an toàn, hai bản code cùng sống |
| Rollback = chỉ đổi image | Thiết kế **migration backward-compatible** | schema không lùi sạch |
| Deploy = release | **Feature flag** tách deploy/release | canary/kill-switch/A-B; giảm rủi ro |
| Recreate cho mọi thứ | Rolling (mặc định) / blue-green / canary | zero-downtime, rollback nhanh, an toàn |
| Canary "bằng mắt" | **Metric-gated** (SLO/error/latency) | tự promote/rollback khách quan |
| "Retry deploy" khi sự cố | **Quy trình incident** (cầm máu → khôi phục → postmortem) | không làm tệ hơn |

### ④ Áp dụng thực tế + So sánh bigtech
Use case: đổi tên cột `full_name`→`name` zero-downtime bằng expand/contract; release tính năng rủi ro cao bằng canary metric-gated + flag kill-switch. Pattern phổ biến (verify): **Argo Rollouts / Flagger** cho canary/blue-green tự động trên K8s; **expand/contract** là chuẩn vàng cho migration online (gh-ost/pt-osc cho MySQL, migration tool cho Postgres); feature flag (LaunchDarkly/OpenFeature) tách deploy↔release ở hầu hết tổ chức trưởng thành. Big-tech deploy **rất thường** + lô nhỏ + flag (khớp DORA — Bài 11).

### ⑤ Code thực hành + cấu hình

Expand/contract đổi tên cột (N-DEPLOY-006) — 4 release, không downtime:

```sql
-- RELEASE 1 (EXPAND): thêm cột mới, KHÔNG đụng cột cũ
ALTER TABLE users ADD COLUMN name varchar;            -- nullable, an toàn
```
```js
// Code R1: ghi CẢ HAI cột, đọc ưu tiên cột cũ (cả 2 bản code cùng chạy được)
function saveUser(u){ db.update({ full_name: u.name, name: u.name }); }
function readUser(r){ return r.full_name ?? r.name; }
```
```sql
-- RELEASE 2 (BACKFILL): copy dữ liệu cũ -> mới (chạy nền, idempotent)
UPDATE users SET name = full_name WHERE name IS NULL;
```
```js
// RELEASE 3: code chỉ đọc/ghi cột MỚI (full_name vẫn còn, chưa xoá)
function saveUser(u){ db.update({ name: u.name }); }
function readUser(r){ return r.name; }
```
```sql
-- RELEASE 4 (CONTRACT): khi chắc không còn bản code nào đọc cột cũ
ALTER TABLE users DROP COLUMN full_name;
```

Canary tự động (Argo Rollouts, minh hoạ):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: web }
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5          # 5% traffic
        - pause: { duration: 5m }
        - analysis: { templates: [{ templateName: error-rate-slo }] }  # metric-gated -> rollback nếu vi phạm
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 100
# // apiVersion / analysis template verify tại argoproj.github.io/argo-rollouts
```

> ⚠️ **Quy tắc sống còn:** không bao giờ ghép "đổi schema không tương thích" với "đổi code" trong **một** bước deploy. `// cú pháp migration tuỳ DB; verify.`

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** recreate/rolling/blue-green/canary trade-off · **rollback code ≠ rollback schema** · expand/contract (hai bản code cùng sống) · deploy≠release (flag) · incident-first khi rollback fail.
- 📌 **Cần THUỘC:** maxSurge/maxUnavailable (Bài 6) · metric-gated canary (SLO) · expand→backfill→contract · feature flag/kill-switch · backward-compatible API.
- 🛠️ **Cần LÀM ĐƯỢC:** thiết kế migration zero-downtime (đổi/xoá cột) · chọn chiến lược cho một thay đổi rủi ro · xử lý incident "rollback cũng fail".

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Code lùi dễ, schema lùi khó → expand/contract; deploy ≠ release (flag); khi cháy: cầm máu trước, postmortem sau."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* Blue-green vs canary khác gì? → cắt 100% vs nhỏ giọt %.
2. *(khó)* Vì sao rollback khó khi đã migrate DB? → schema không lùi sạch; data bản mới không hợp bản cũ.
3. *(khó⭐⭐⭐⭐⭐)* Thiết kế đổi tên cột zero-downtime? → expand→deploy tương thích→backfill→contract.
4. *(khó)* Feature flag tách deploy/release thế nào, cái giá? → bật/tắt runtime; flag debt + test tổ hợp.
5. *(khó⭐⭐⭐⭐⭐)* Deploy fail, rollback cũng fail — xử lý? → cầm máu → đánh giá state → khôi phục → giao tiếp → postmortem.

**Thách đố / trick:**
- *"Tôi `helm rollback`/đổi image về bản cũ mà app vẫn 500 hàng loạt — vì sao?"* → migration đã **drop/rename cột**; bản cũ query cột không còn → vỡ. Rollback code không cứu schema; cần expand/contract từ đầu.
- *"Canary 5% xanh, lên 100% thì sập — bug gì?"* → bug chỉ lộ ở **tải/đường dẫn hiếm** (cache/DB connection cạn ở scale). Metric-gated theo SLO + tăng weight chậm + đủ thời gian quan sát.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Viết kế hoạch 4 release** đổi tên cột zero-downtime (expand/contract).
   - *Đạt khi:* mỗi bước hai bản code cùng chạy được; backfill trước contract; không gộp schema+code.
2. **Bảng quyết định chiến lược:** 3 thay đổi (CSS nhỏ / đổi API contract / thuật toán rủi ro) → chọn rolling/blue-green/canary + lý do.
   - *Đạt khi:* khớp rủi ro–chi phí–observability; nêu điều kiện zero-downtime.

### ⑩ Đọc thêm / nguồn chuẩn
argoproj.github.io/argo-rollouts; flagger (flagger.app); martinfowler.com (BlueGreenDeployment, CanaryRelease, ParallelChange/expand-contract); Google SRE book (incident response, blameless postmortem).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** Hỏi AI "đổi tên cột DB", nó thường generate **một** migration `RENAME COLUMN` + sửa code cùng lúc — *cú pháp đúng, sai kiến trúc deploy*: trong lúc rolling, bản cũ vỡ. TL chỉ huy expand/contract. *Hiểu định hình yêu cầu, không phải cú pháp.*

---

## 🎓 BÀI 13 — Managed Cloud Services: ranh giới trách nhiệm & quyết định build-vs-buy

> **Câu hỏi nguồn:** N-CLOUD-001 → 007 · **Tags:** N-CLOUD · **Trọng số:** trung bình–cao (TL phải ra quyết định tiền & lock-in)

### ① Mục tiêu & vị trí trong mạch
Mười hai bài trước dạy bạn **tự vận hành** container/K8s/CI-CD/IaC. Bài 13 lật mặt còn lại: phần lớn thứ đó **cloud provider đã làm sẵn** — và câu hỏi TL không còn là "làm thế nào" mà là "**nên tự làm hay mua**, và khi mua thì mình **còn chịu trách nhiệm phần nào**". Đây là bài về *ranh giới* (shared responsibility) và *kinh tế kỹ thuật* (engineer-time, SLA, lock-in), không phải về cú pháp. Nó nối thẳng vào Bài 14 (quyết định TL-level) và tham chiếu chéo J (CDN/cache), M (IAM/secrets), O (cost observability).

### ② Giảng cơ bản → nâng cao

**a. Trực giác — "managed = họ lo tầng dưới, bạn lo tầng trên".**
Mọi dịch vụ managed đều vẽ một **đường kẻ trách nhiệm**. Trên đường: của bạn (config, data, code, quyền truy cập). Dưới đường: của provider (phần cứng, control-plane, patch hệ điều hành nền). Hiểu sai vị trí đường kẻ = sự cố hoặc hoá đơn bất ngờ.

**b. Cơ chế — đi qua từng dịch vụ lõi:**

- **Managed Kubernetes (EKS / GKE / AKS) — N-CLOUD-001.** Provider quản **control-plane** (API server, etcd, scheduler, controller-manager): vá lỗi, backup etcd, HA control-plane. **Bạn vẫn sở hữu:** node/nodepool (hoặc dùng node tự quản), CNI/networking choices, workload, RBAC, autoscaling config, nâng cấp version cluster (provider cho cửa sổ, bạn bấm & chịu rủi ro app). ⇒ "Managed K8s" **không** = "khỏi lo K8s"; nó bỏ gánh *control-plane*, giữ nguyên gánh *vận hành workload* (toàn bộ Bài 3–7).
- **Managed DB — RDS / Aurora / Cloud SQL — N-CLOUD-002.** Provider lo: backup tự động + **PITR** (point-in-time recovery), failover Multi-AZ, minor patch, replica. Bạn lo: schema, index, query, **migration zero-downtime** (Bài 12), chọn instance size, và **chấp nhận** bạn không có superuser/không sửa được mọi tham số kernel. Trade-off: đắt hơn self-run + nguy cơ **lock-in** (Aurora/Cloud SQL có API & hành vi riêng), đổi lại xoá gần hết gánh ops của DB — thứ mà tự làm cho *đúng* (HA + PITR + test restore) rất tốn người.
- **Object vs Block vs File storage — N-CLOUD-003.** Chọn theo **access pattern**, không theo thói quen:
  - **Object (S3/GCS):** HTTP API, key→blob, gần như vô hạn, 11 số 9 độ bền, **không** phải filesystem (không seek/append rẻ, không POSIX). Cho asset, backup, data-lake, log, ảnh/video.
  - **Block (EBS/PD):** "ổ đĩa ảo" gắn **một** node tại một thời điểm (RWO), IOPS cao, độ trễ thấp. Cho DB tự host, filesystem cần ghi ngẫu nhiên. ↔ Bài 6 PV/StorageClass.
  - **File (EFS/Filestore, NFS):** filesystem chia sẻ **nhiều** node (RWX). Cho legacy cần shared mount, không tối ưu IOPS.
  Sai pattern = đau: nhét DB lên object store, hay mong S3 hành xử như NFS.
- **CDN / Edge — CloudFront / Cloud CDN — N-CLOUD-004.** Cache **ở rìa** (gần user), phục vụ tốt **nội dung tĩnh / cache được**. **Không** cho dynamic per-request (mỗi request một kết quả riêng → edge không cache nổi, chỉ thành proxy thêm hop). **Invalidation** là điểm trick: xoá cache toàn cầu **chậm/tốn**, nên thiết kế **versioned URL / cache-busting** (`app.[hash].js`) thay vì invalidation. ↔ J (caching). Đây là tầng cache **ngoài cùng** trong "thang cache" của J.
- **Workload Identity — IRSA / GKE Workload Identity — N-CLOUD-005.** Pod cần gọi API cloud (đọc S3, gửi SQS) **không** nhét static access key vào env/secret. Thay vào: pod ServiceAccount ↔ IAM role qua **OIDC**, nhận **token ngắn hạn tự xoay vòng**. Đây chính là biến thể của "keyless OIDC" bạn đã gặp ở CI/CD (Bài 11) — *cùng một nguyên lý*: không có bí mật tĩnh để rò. ↔ M (IAM, secret zero).

**c. Mép giới hạn — chỗ "managed" gãy:**
- Managed **không** miễn nhiễm lỗi của bạn: cấu hình sai security group, quên bật Multi-AZ, để public S3 bucket → sự cố là **của bạn**, SLA provider không cứu.
- **SLA ≠ đảm bảo của bạn.** RDS SLA 99.95% là *uptime hạ tầng*; nếu bạn chạy single-AZ thì SLA đó không áp dụng. Đọc kỹ điều kiện.
- **Lock-in là chi phí trả chậm.** Aurora, DynamoDB, Cloud Run… mỗi cái buộc bạn vào API riêng; migration ra ngoài có thể tốn hàng quý kỹ sư. Không phải lý do để né — là biến số phải **định giá**.

### ③ ⚠️ Kiến thức cũ / dễ bị hiểu sai

| Quan niệm cũ / sai | Thực tế hiện nay | Vì sao đổi |
|---|---|---|
| "Managed K8s = không cần biết K8s" | Chỉ bỏ gánh **control-plane**; workload/RBAC/upgrade vẫn của bạn | Tách bạch shared-responsibility |
| "Cứ ném mọi thứ lên S3 cho rẻ" | S3 không phải filesystem; sai access pattern → chậm/đắt/sai | Object≠block≠file |
| "CDN tăng tốc mọi thứ" | Chỉ nội dung cache được; dynamic per-request không hưởng lợi | Bản chất edge cache |
| "Để key cloud trong K8s Secret là đủ" | Secret chỉ base64; dùng **Workload Identity** token ngắn hạn | Bài 14 §secrets, M |
| "Self-host rẻ hơn vì không trả phí managed" | Bỏ sót **chi phí kỹ sư-giờ** + rủi ro vận hành | Build-vs-buy đúng nghĩa |

### ④ Áp dụng thực tế + so sánh bigtech
- **Startup/scale-up:** mặc định **buy** (RDS, managed K8s hoặc thậm chí PaaS — xem Bài 14) để dồn người vào sản phẩm. Self-host chỉ khi có lý do định lượng (chi phí ở quy mô lớn, yêu cầu tuân thủ, hiệu năng đặc thù).
- **Bigtech (Google/Meta/Amazon nội bộ):** thường **build** vì ở quy mô của họ, phí managed × hàng triệu máy > chi phí đội platform chuyên trách; và họ *là* nhà cung cấp. Đây là lý do "vì sao Google tự làm" **không** suy ra được "startup của bạn nên tự làm" — kinh tế khác hẳn quy mô.
- **Quyết định build-vs-buy (N-CLOUD-006)** xoay quanh: (1) **engineer-time** để xây *và duy trì* (không chỉ ngày đầu), (2) **SLA/độ tin cậy** bạn tự đạt được so với mua, (3) **lock-in** & chi phí thoát, (4) sự khác biệt có phải **lợi thế cạnh tranh** không — nếu không, mua. Giá tháng chỉ là một biến nhỏ.

### ⑤ Code thực hành + cấu hình

**IRSA — pod nhận quyền AWS không cần static key (N-CLOUD-005):**
```yaml
# ServiceAccount gắn IAM role qua OIDC (annotation là "dây nối")
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/app-s3-read  # // ARN verify
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: app }
spec:
  template:
    spec:
      serviceAccountName: app-sa     # pod dùng SA này -> token OIDC ngắn hạn auto-mount
      containers:
        - name: app
          image: registry.example.com/app@sha256:<digest>   # pin digest, không :latest
          # KHÔNG có AWS_ACCESS_KEY_ID/SECRET ở đây — đó là cả mục đích
```

**Right-size + autoscale + spot (N-CLOUD-007) — nodepool hỗn hợp GKE (minh hoạ):**
```yaml
# Ý tưởng: workload chịu gián đoạn -> spot/preemptible (rẻ 60-90%); stateful -> on-demand
# (cú pháp tuỳ provider/IaC; đây là sơ đồ quyết định, // verify theo Terraform module)
on-demand-pool:  { node: stateful/DB-adjacent, autoscale: 2..6 }
spot-pool:       { node: stateless/batch,       autoscale: 0..20, taint: spot=true:NoSchedule }
# workload stateless thêm toleration spot=true -> xài pool rẻ; PDB (Bài 6) để chịu eviction.
```

**Đòn bẩy chi phí (N-CLOUD-007) & rủi ro kèm theo:**
```
right-size      : khớp request/limit thực (Bài 5) -> rủi ro: chỉnh quá tay gây OOM/throttle.
autoscale       : HPA/Cluster-Autoscaler co theo tải -> rủi ro: cold-start, thrash nếu metric nhiễu.
spot/preemptible: rẻ -> rủi ro: bị thu hồi bất kỳ lúc -> chỉ cho stateless + PDB + retry.
storage tiering : S3 IA/Glacier cho data lạnh -> rủi ro: phí truy xuất + độ trễ khi cần gấp.
commit/savings  : reserved/CUD 1-3 năm -> rủi ro: cam kết sai dung lượng = trả cho thứ không dùng.
```

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** shared responsibility (đường kẻ trách nhiệm) · build-vs-buy = engineer-time+SLA+lock-in, không chỉ giá · object≠block≠file theo access pattern · edge cache chỉ cho cache-able · workload identity = OIDC token ngắn hạn.
- 📌 **Cần THUỘC:** EKS/GKE managed control-plane (bạn giữ workload) · RDS = backup/PITR/Multi-AZ failover · S3 11×9 độ bền/RWO của EBS/RWX của EFS · IRSA annotation role-arn · spot chỉ cho stateless.
- 🛠️ **Cần LÀM ĐƯỢC:** vẽ bảng build-vs-buy cho một dịch vụ · chọn loại storage theo pattern · cấu hình pod lấy quyền cloud không static key · liệt kê đòn bẩy chi phí + rủi ro.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"Managed dời đường kẻ trách nhiệm xuống, không xoá nó. TL định giá engineer-time + SLA + lock-in — chứ không đọc bảng giá tháng."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(TB)* EKS quản giúp bạn cái gì, bạn còn lo gì? → control-plane vs workload/node/RBAC/upgrade.
2. *(TB)* Khi nào object store, khi nào block? → blob/HTTP/append-rẻ-không vs ổ đĩa RWO IOPS cao.
3. *(khó)* RDS cho gì, đánh đổi gì? → backup/PITR/failover/patch ↔ đắt + lock-in + mất superuser.
4. *(khó)* Pod cần đọc S3, làm sao không nhét key? → IRSA/Workload Identity, token OIDC ngắn hạn.
5. *(khó⭐⭐⭐⭐⭐)* Khung quyết định build-vs-buy của bạn? → engineer-time duy trì + SLA tự đạt + lock-in/chi phí thoát + có phải lợi thế cạnh tranh.

**Thách đố / trick:**
- *"Self-host Postgres trên EC2 rẻ hơn RDS mỗi tháng — chốt self-host nhé?"* → so sai biến. Cộng **kỹ sư-giờ** để đạt **HA + PITR + test-restore định kỳ** cho ngang RDS; thường tổng chi phí + rủi ro lớn hơn. Giá tháng chỉ là phần nổi.
- *"Bật CloudFront trước API động cho nhanh."* → API per-request **không cache được** → CDN chỉ thêm một hop, không nhanh hơn (có khi chậm hơn). CDN cho tĩnh/cache-able; động thì tối ưu ở app/DB/cache nội bộ (J).
- *"Để giảm phí, mình mua Reserved 3 năm tất cả."* → cam kết sai dung lượng = trả cho công suất không dùng; chỉ commit phần **nền ổn định**, để phần co giãn cho on-demand/spot.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Bảng build-vs-buy** cho "message queue" của hệ bạn (SQS/managed Kafka vs tự host): liệt kê engineer-time, SLA, lock-in, chi phí thoát, kết luận.
   - *Đạt khi:* có cả 4 biến + kết luận gắn với "có phải lợi thế cạnh tranh".
2. **Chọn storage** cho 4 nhu cầu: ảnh user upload / DB Postgres self-host / log lưu 1 năm / thư mục chia sẻ giữa 5 pod legacy.
   - *Đạt khi:* object / block / object-tiering / file (RWX) — kèm lý do access pattern.
3. **Sửa cấu hình rò bí mật:** cho một Deployment nhét `AWS_ACCESS_KEY_ID`, viết lại bằng IRSA.
   - *Đạt khi:* bỏ static key, có SA annotation role-arn, giải thích token ngắn hạn.

### ⑩ Đọc thêm / nguồn chuẩn
AWS/GCP/Azure shared responsibility model docs; AWS Well-Architected (Cost Optimization pillar); docs.aws.amazon.com (EKS IRSA, RDS PITR, S3 storage classes); cloud.google.com (Workload Identity, Cloud SQL); FinOps Foundation (finops.org) cho văn hoá chi phí (nối Bài 14).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** Hỏi AI "deploy app cần đọc S3 lên EKS", nó hay generate Deployment với `AWS_ACCESS_KEY_ID`/`SECRET` trong env — *chạy được, nhưng nhét bí mật tĩnh vào cluster*, đúng thứ Bài 14 & M cấm. TL thấy ngay phải là **IRSA/Workload Identity**. Và khi AI nói "self-host rẻ hơn", TL biết hỏi lại *"đã cộng kỹ sư-giờ duy trì HA+PITR chưa?"*. *Hiểu định hình quyết định, không phải cú pháp.*

---

## 🎓 BÀI 14 — TL-level: quyết định, văn hoá & vận hành ở quy mô (CAPSTONE)

> **Câu hỏi nguồn:** N-TL-001 → 006 · **Tags:** N-TL · **Trọng số:** cao nhất về *tư duy chỉ huy* (ít cú pháp, nhiều quyết định)

### ① Mục tiêu & vị trí trong mạch
Đây là bài **gộp mạch**. Mười ba bài trước cho bạn công cụ; bài này hỏi: **khi nào dùng, khi nào KHÔNG, và làm sao để cả tổ chức làm đúng mà không biến DevOps thành cửa ải**. Câu hỏi TL ở đây gần như **không có cú pháp** — chúng đo *khả năng ra quyết định dưới ràng buộc* (đội nhỏ, tiền, rủi ro, tuân thủ). Bài này tham chiếu dày sang M (supply-chain, secrets), và là nơi câu "AI viết đúng cú pháp nhưng sai vận hành/kiến trúc" lên đỉnh: phần lớn sai lầm đắt nhất ở đây là **chọn sai công cụ ngay từ đầu**, không phải sai dòng YAML.

### ② Giảng cơ bản → nâng cao

**a. Trực giác — "TL được trả lương để nói KHÔNG đúng lúc".** Junior hỏi *làm K8s thế nào*; TL hỏi *có nên không*. Giá trị nằm ở việc chặn độ phức tạp không cần thiết và mở "đường ray" để team đi nhanh mà an toàn.

**b. Cơ chế — sáu quyết định lõi:**

- **N-TL-001 — "Startup muốn K8s vì ai cũng dùng": khi nào K8s là SAI.** K8s trả giá bằng **độ phức tạp vận hành** (Bài 3–7): cần người hiểu networking, RBAC, upgrade, observability. Đội nhỏ / tải thấp / ít service ⇒ chi phí đó **lớn hơn** lợi ích. Lựa chọn đúng thường là **PaaS / serverless container**: Cloud Run, ECS Fargate, AWS App Runner, Render, Fly.io — deploy bằng một lệnh, tự scale, không phải nuôi cluster. Quy tắc: **chọn độ phức tạp thấp nhất giải quyết được bài toán**; "lên K8s" là quyết định cần *biện minh bằng nhu cầu*, không phải mặc định theo trend (cargo-cult). Dấu hiệu *nên* K8s: nhiều service, cần lịch trình/scheduling tinh vi, multi-tenant, team đủ lớn để có người chuyên trách, hoặc cần tránh lock-in PaaS một cách có chủ đích.
- **N-TL-002 — Kiểm soát chi phí ở tổ chức: văn hoá + cơ chế, không chỉ con số.** Cắt phí bền vững cần (1) **hiển thị**: gắn nhãn (tag/label) tài nguyên theo team/service → biết tiền chảy đâu (showback/chargeback); (2) **trách nhiệm**: team thấy hoá đơn của chính mình (FinOps); (3) **cơ chế tự động**: budget alert, right-sizing định kỳ, dọn tài nguyên mồ côi, spot cho phù hợp (Bài 13). Chỉ "bảo mọi người tiết kiệm" → thất bại; phải có *dữ liệu + động lực + tự động hoá*.
- **N-TL-003 — Quản secrets ở quy mô.** K8s `Secret` mặc định chỉ **base64** (không phải mã hoá) — đủ để lộ nếu ai đọc được etcd/manifest. Ở quy mô cần: **External Secrets Operator** (đồng bộ từ Vault/AWS Secrets Manager/GCP SM vào cluster), hoặc **Sealed Secrets** (mã hoá để commit an toàn vào Git — nối GitOps Bài 10), hoặc **Vault** (quản tập trung + dynamic secrets + audit). Nguyên tắc: bí mật **không** nằm thô trong Git/manifest; có **xoay vòng** & **audit**. ↔ M-SECRET (đào sâu ở mục M).
- **N-TL-004 — Chỉ image tin cậy được chạy ở prod.** Ba lớp: (1) **Ký** image — cosign/Sigstore tạo chữ ký xác thực nguồn gốc; (2) **Chặn ở cửa** — admission policy **Kyverno / Gatekeeper(OPA)** từ chối pod nếu image không ký / không từ registry duyệt / có CVE nghiêm trọng; (3) **SBOM + scan** — biết image gồm gì (software bill of materials) và quét lỗ hổng trong CI (Bài 11). Kết quả: chuỗi cung ứng có thể **kiểm chứng**, không "tin vì nó nằm trong registry". ↔ M-SUPPLY.
- **N-TL-005 — Multi-region / DR.** Hai mô hình: **active-active** (mọi region phục vụ thật, chịu lỗi tốt nhất, khó nhất vì state/đồng bộ dữ liệu) và **active-passive** (một region chính + dự phòng chờ, đơn giản hơn, RTO cao hơn). Hai số đo: **RTO** (bao lâu khôi phục) & **RPO** (mất tối đa bao nhiêu dữ liệu). **Bẫy lớn nhất:** stateless thì dễ nhân bản, nhưng **DB/state** mới là chỗ khó — replicate đồng bộ vs bất đồng bộ, conflict, failover DB. Và **DR chưa từng test = không có DR**: phải diễn tập (game day) định kỳ, nếu không runbook chỉ là giấy.
- **N-TL-006 — Platform Engineering / IDP: "golden path" thay vì gác cổng.** Mô hình cũ: DevOps là **người gác cổng** — mọi deploy phải qua họ → nghẽn cổ chai. Mô hình mới: đội platform xây **Internal Developer Platform** cung cấp **golden path** (đường ray vàng) — template, pipeline, hạ tầng self-service đã "bọc" sẵn best practice & policy. Dev tự deploy **trong** đường ray an toàn; platform team làm *sản phẩm cho developer*, không phải *trạm kiểm soát*. Mục tiêu: tốc độ **và** an toàn cùng lúc, giảm gánh nhận thức cho dev.

**c. Mép giới hạn — nơi TL dễ sai:**
- **Cargo-cult bigtech:** copy kiến trúc Google/Netflix khi chưa có quy mô/đội ngũ của họ → ôm phức tạp vô ích. "Họ làm vì họ phải; bạn chưa phải."
- **Platform thành gatekeeper trá hình:** dựng IDP nhưng vẫn bắt mọi thứ duyệt tay → mất luôn lợi ích self-service.
- **Policy quá ngặt:** admission policy chặn cả việc chính đáng → dev tìm đường lách → tệ hơn không có policy. Cần "đường ray" dễ đi đúng hơn đi sai.
- **DR trên giấy:** có runbook, chưa test → khi cháy mới biết nó sai.

### ③ ⚠️ Kiến thức cũ / dễ bị hiểu sai

| Quan niệm cũ / sai | Thực tế TL-level | Vì sao đổi |
|---|---|---|
| "Hệ nào cũng nên lên K8s" | Đội nhỏ/tải thấp → PaaS/serverless (Cloud Run, Fargate, App Runner) | K8s = chi phí phức tạp phải biện minh |
| "Bảo team tiết kiệm là đủ" | Cần tag + showback + alert + tự động hoá (FinOps) | Văn hoá phải có cơ chế đỡ |
| "K8s Secret là an toàn" | Chỉ base64; cần ESO/Vault/Sealed Secrets + xoay vòng | base64 ≠ mã hoá |
| "Image trong registry là tin được" | Phải **ký** + admission policy + SBOM/scan | Supply-chain verifiable |
| "Có region thứ 2 là có DR" | DR chưa test = không có; state/DB mới là chỗ khó | RTO/RPO + game day |
| "DevOps duyệt mọi deploy cho an toàn" | Golden path self-service; platform là sản phẩm | Bỏ nghẽn cổ chai |

### ④ Áp dụng thực tế + so sánh bigtech
- **Startup 5 người, 2 service, tải thấp:** **đừng** K8s. Cloud Run/Fargate + RDS + CI/CD đơn giản (Bài 11) + secrets manager của cloud. Dồn 100% người vào sản phẩm. Lên K8s khi *đau thật* (nhiều service, cần scheduling/multi-tenant).
- **Scale-up 50+ kỹ sư, hàng chục service:** lúc này K8s + **IDP/golden path** trả lời được bài toán nghẽn cổ chai; đầu tư platform team, policy-as-code (Kyverno), External Secrets, FinOps tagging.
- **Bigtech:** build platform nội bộ khổng lồ (Google Borg→K8s nguồn gốc; Spotify Backstage→chuẩn IDP mã nguồn mở). Họ là *minh chứng mô hình*, không phải *khuôn để copy nguyên xi* cho đội nhỏ.
- Xuyên suốt: **chọn độ phức tạp tối thiểu khả thi**, rồi *tăng dần khi có bằng chứng nhu cầu* — đó là chữ ký của TL giỏi.

### ⑤ Code thực hành + cấu hình

**Admission policy chặn image chưa ký / sai registry (Kyverno, N-TL-004):**
```yaml
apiVersion: kyverno.io/v1            # // apiVersion verify tại kyverno.io
kind: ClusterPolicy
metadata: { name: only-trusted-images }
spec:
  validationFailureAction: Enforce   # chặn thật, không chỉ cảnh báo
  rules:
    - name: require-registry
      match: { any: [{ resources: { kinds: [Pod] } }] }
      validate:
        message: "Chỉ cho image từ registry nội bộ đã duyệt."
        pattern:
          spec:
            containers:
              - image: "registry.internal.example.com/*"   # ngoài danh sách -> từ chối
    - name: verify-signature
      match: { any: [{ resources: { kinds: [Pod] } }] }
      verifyImages:                  # // cú pháp verifyImages/cosign verify theo bản Kyverno
        - imageReferences: ["registry.internal.example.com/*"]
          attestors:
            - entries: [{ keyless: { subject: "https://github.com/org/*", issuer: "https://token.actions.githubusercontent.com" } }]
              # keyless OIDC: cùng nguyên lý "không khoá tĩnh" như IRSA (Bài 13) & CI OIDC (Bài 11)
```

**External Secrets — kéo bí mật từ Vault/SM vào cluster (N-TL-003):**
```yaml
apiVersion: external-secrets.io/v1   # // group/version verify tại external-secrets.io
kind: ExternalSecret
metadata: { name: db-credentials }
spec:
  secretStoreRef: { name: aws-sm, kind: ClusterSecretStore }
  target: { name: db-credentials }   # tạo K8s Secret này, đồng bộ từ nguồn ngoài
  refreshInterval: 1h                 # xoay vòng định kỳ
  data:
    - secretKey: password
      remoteRef: { key: prod/db, property: password }   # nguồn thật ở AWS SM, KHÔNG ở Git
```

**Khung quyết định "có nên K8s?" (N-TL-001) — checklist:**
```
Lên K8s NẾU đa số đúng:        | Ở lại PaaS/serverless NẾU:
- >~5-10 service liên thuộc    | - 1-3 service, tải thấp/bursty
- cần scheduling/affinity sâu  | - team <~10, không người chuyên K8s
- multi-tenant / cô lập mạnh   | - cần ship nhanh, ops tối thiểu
- có người vận hành chuyên     | - chưa có nhu cầu định lượng rõ
- chủ đích tránh lock-in PaaS  | => Cloud Run / Fargate / App Runner
```

### ⑥ Keywords cần nhớ (glossary)
- 🧠 **Cần HIỂU:** chọn độ phức tạp tối thiểu khả thi · K8s là chi phí phải biện minh, không phải mặc định · platform = sản phẩm (golden path), không phải cổng · DR chưa test = không có DR · cost = văn hoá + cơ chế.
- 📌 **Cần THUỘC:** PaaS thay K8s (Cloud Run/Fargate/App Runner/Render) · K8s Secret chỉ base64 → ESO/Vault/Sealed Secrets · image ký cosign/Sigstore + admission Kyverno/Gatekeeper/OPA + SBOM · active-active vs active-passive, RTO/RPO · IDP/golden path.
- 🛠️ **Cần LÀM ĐƯỢC:** quyết định K8s hay không bằng checklist · thiết kế cơ chế kiểm soát chi phí tổ chức · viết admission policy "chỉ image tin cậy" · phác kế hoạch DR có test · phân biệt platform-as-product vs gatekeeper.

### ⑦ Mạch tư duy cần nhớ (mental model)
**"TL chọn độ phức tạp tối thiểu khả thi, nói KHÔNG với cargo-cult, biến an toàn thành đường ray tự phục vụ — và nhớ: thứ chưa test thì coi như không tồn tại."**

### ⑧ Câu hỏi phỏng vấn & thách đố
1. *(khó⭐⭐⭐⭐⭐)* Startup 4 người đòi K8s "vì ai cũng dùng" — bạn nói gì? → hỏi nhu cầu định lượng; mặc định PaaS/serverless; K8s khi đau thật. Chống cargo-cult.
2. *(khó)* Kiểm soát chi phí cloud toàn tổ chức ra sao? → tag + showback/FinOps + alert + tự động right-size/spot; văn hoá + cơ chế.
3. *(khó)* K8s Secret đủ an toàn chưa, không thì sao? → chỉ base64; ESO/Vault/Sealed Secrets + xoay vòng + audit.
4. *(khó⭐⭐⭐⭐⭐)* Đảm bảo prod chỉ chạy image tin cậy? → ký (cosign) + admission policy (Kyverno/OPA) + SBOM/scan trong CI.
5. *(khó⭐⭐⭐⭐⭐)* Thiết kế multi-region DR; bẫy lớn nhất? → active-active/passive, RTO/RPO; bẫy = state/DB + DR chưa diễn tập.
6. *(khó)* Platform team nên là gì để không thành nghẽn cổ chai? → IDP/golden path self-service, platform-as-product.

**Thách đố / trick:**
- *"Mình bắt chước kiến trúc Netflix cho MVP 3 service cho chuẩn."* → cargo-cult: ôm phức tạp của quy mô bạn chưa có. Netflix làm vì *buộc phải*; MVP cần đơn giản để học thị trường. Tăng phức tạp **theo bằng chứng nhu cầu**.
- *"Dựng IDP rồi, nhưng mọi deploy vẫn bắt platform-team duyệt tay."* → đó vẫn là **gatekeeper**, chỉ đổi tên. Golden path phải *self-service*; policy nằm trong đường ray (tự động), không phải con người duyệt từng lần.
- *"Có cluster ở region B rồi, coi như xong DR."* → có region ≠ có DR. Đã replicate **DB/state** chưa? RTO/RPO bao nhiêu? **Đã diễn tập failover thật chưa?** Chưa test = chưa có.
- *"Cứ để admission policy chặn hết cho chắc."* → quá ngặt → dev lách (chạy ngoài, tắt policy) → tệ hơn. Thiết kế để *đi đúng dễ hơn đi sai*.

### ⑨ Bài tập thực hành + tiêu chí tự chấm
1. **Memo quyết định (1 trang):** "Có nên đưa hệ X lên K8s?" cho một startup giả định — dùng checklist §⑤, kết luận có điều kiện.
   - *Đạt khi:* nêu nhu cầu định lượng, so PaaS, kết luận gắn quy mô/đội ngũ, không cargo-cult.
2. **Thiết kế kiểm soát chi phí:** đề xuất tag schema + 3 cơ chế tự động + cách team thấy hoá đơn của mình.
   - *Đạt khi:* có cả *hiển thị + trách nhiệm + tự động hoá*, không chỉ "kêu gọi tiết kiệm".
3. **Policy supply-chain:** viết (giả mã hoặc Kyverno) chính sách "chỉ image ký từ registry nội bộ"; mô tả phần CI tạo chữ ký + SBOM.
   - *Đạt khi:* đủ 3 lớp ký/chặn/scan; chỉ rõ chỗ nối CI (Bài 11) và M-SUPPLY.
4. **Runbook DR rút gọn + kế hoạch game day:** chọn active-passive cho một hệ có Postgres; nêu RTO/RPO mục tiêu, bước failover, điều cần test.
   - *Đạt khi:* xác định rõ xử lý **state/DB**, có lịch diễn tập, định nghĩa "thành công" của bài test.

### ⑩ Đọc thêm / nguồn chuẩn
Team Topologies (Skelton & Pais) & "Platform as a product"; Backstage (backstage.io) cho IDP; CNCF TAG-App-Delivery; Sigstore/cosign (sigstore.dev), Kyverno (kyverno.io), OPA/Gatekeeper (openpolicyagent.org); External Secrets Operator (external-secrets.io), HashiCorp Vault; Google SRE Workbook (DR, game days); FinOps Foundation (finops.org). Tham chiếu chéo: **M** (secrets/supply-chain), **O** (observability/cost), **R** (xu hướng platform mới).

> 🧪 **"Hiểu để chỉ huy & kiểm tra":** Đây là bài AI sai *đắt nhất* mà *trông đúng nhất*. Hỏi "deploy startup nhỏ thế nào", AI thường dựng nguyên bộ K8s + Helm + Ingress — *mọi dòng YAML đều đúng cú pháp*, nhưng **sai quyết định kiến trúc**: đội 4 người không nuôi nổi cluster, lẽ ra Cloud Run. Hỏi "quản secret", AI cho `kubectl create secret` — *chạy được*, nhưng base64 ≠ an toàn ở quy mô. Hỏi "chạy image", AI kéo `:latest` từ Docker Hub — *được*, nhưng không ký, không scan, không admission. **TL đọc qua phần cú pháp để bắt phần quyết định & vận hành** — và đó là toàn bộ giá trị của vai trò này. *Hiểu định hình yêu cầu và quyết định; cú pháp chỉ là bề mặt.*

---

## 🏁 Kết mạch N — DevOps / Docker / K8s / CI-CD

Bạn đã đi từ **container vs VM** (Bài 1) → **build/runtime sâu** (2) → **K8s core & ops** (3–7) → **đóng gói Helm/Kustomize** (8) → **IaC** (9) → **GitOps** (10) → **CI/CD** (11) → **chiến lược deploy & migration zero-downtime** (12) → **managed cloud & build-vs-buy** (13) → **quyết định/văn hoá/vận hành TL-level** (14).

**Sợi chỉ xuyên suốt cần mang vào phỏng vấn:**
- **Khai báo > mệnh lệnh:** mô tả trạng thái mong muốn (K8s/IaC/GitOps), để hệ tự hội tụ — và biết mặt trái (drift, reconcile).
- **Không có khoá tĩnh:** OIDC keyless lặp lại ở CI (11), workload identity (13), ký image (14) — *cùng một nguyên lý chống rò bí mật*.
- **Rollback code ≠ rollback state:** expand/contract, deploy≠release (12) — chỗ AI sai kiến trúc nhiều nhất.
- **Chọn độ phức tạp tối thiểu khả thi:** đỉnh ở Bài 14 — biết **khi nào KHÔNG** dùng K8s quan trọng ngang biết cách dùng.
- **Thứ chưa test coi như không tồn tại:** probe, PDB, DR, restore — tất cả phải được kiểm chứng.

*Mọi mốc version đều gắn nhãn (verify) — luôn kiểm lại theo đời K8s/đời tài liệu tại thời điểm bạn đọc, vì hệ sinh thái này đổi nhanh.*
