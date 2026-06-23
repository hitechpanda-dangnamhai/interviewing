# 🔒 QB_M — Ngân hàng câu hỏi: Security

> **Mục M** trong roadmap Tech Lead Backend (🟡 ~40%, cao ở fintech/healthcare) · Sinh theo **WORKFLOW 2 — Bước A**.
> Đây là **mục LỚN** → bộ đề phủ HẾT 6 mục con của roadmap (JWT/Session/OAuth · OAuth2/OIDC flow · OWASP Top 10 · Secrets management · Supply chain · Zero-trust/mTLS) + lớp **TL-level** (threat modeling, secure SDLC, data protection/compliance, crypto fundamentals) mà phỏng vấn Tech Lead hay vặn.
> **Chỉ câu hỏi — KHÔNG kèm đáp án.** Mỗi câu có dòng *"dò cái gì"* = tiêu chí ĐẠT, dùng để chấm **live** ở Bước D.
> **Tổng: 100 câu.**

## 🚧 Chống trùng (đã đọc `QB_G_nodejs.md`, `QB_H_nestjs.md`, `QB_F_apidesign.md`, `QB_E_database.md`, `QB_L_testing.md`, `QB_Q_designpatterns.md`)

Security đụng chạm nhiều mục khác. Quy ước phân ranh để **không lặp**:

- **`G-SEC` (Node hardening, 9 câu)** đã phủ: helmet, rate-limiter-flexible (wiring), CSRF *cơ bản*, SQLi/NoSQLi *prevention ở Node*, bcrypt/argon2 *chọn cái nào*, secrets *env/.env cơ bản*, supply-chain npm *cơ bản (npm audit/lockfile)*, prototype pollution, Node Permission Model.
  → **M KHÔNG lặp phần "Node wiring".** M đào sâu hơn ở: **tham số argon2id/pepper/migration** (M-PWD), **CSRF + SameSite + CSP như một hệ** (M-XSS), **secrets ở mức kiến trúc — Vault dynamic/KMS/envelope/rotation** (M-SECRET), **supply chain mức mối đe doạ — SBOM/SLSA/provenance/Shai-Hulud** (M-SUPPLY).
- **`H-GRD` (NestJS guards & authorization, 6 câu)** đã phủ: `canActivate`, `@Roles`+Reflector+RolesGuard, AuthGuard(passport) vs RolesGuard, vị trí authn-trước-authz *trong Nest*, ABAC/policy *bằng CASL/OPA trong Nest*.
  → **M KHÔNG lặp cơ chế framework.** M làm phần **khái niệm độc lập framework**: RBAC vs ABAC vs ReBAC *trade-off*, **IDOR/BOLA/BFLA** (OWASP A01), least privilege, multi-tenant isolation.
- **`F` (API Design)** chỉ chạm *bề mặt API* (CORS như header, auth ở handshake WS/SSE) và trỏ sang M. → M làm chiều sâu auth/authz/transport security.
- **`E` (Database)** giữ phần index/transaction; M chỉ lấy góc *encryption at rest / PII / injection như rủi ro bảo mật*, không lặp tuning DB.

---

## Cách đọc
- **ID:** `M-<tag>-<số>`. Tag mục con liệt kê ở mục lục dưới.
- **Độ khó:** ⭐ định nghĩa cơ bản → ⭐⭐⭐⭐⭐ phân biệt senior với TL thật (trade-off, distributed, governance, threat modeling).
- **"Dò cái gì":** điều người trả lời PHẢI chạm tới mới tính ĐẠT (không phải đáp án — chỉ là mốc chấm live ở Bước D).

## Bối cảnh phiên bản & chuẩn (tính tại 6/2026 — *verify lại khi học*)

> ⚠️ Phần này đổi nhanh → mọi câu chạm tên chuẩn/đời spec/đời tấn công đều nên kiểm chứng tại nguồn chính thức trước khi tin.

- **OWASP Top 10:2025** là bản **hiện hành** (công bố 11/2025 tại Global AppSec DC, bản final 1/2026 — bản đầu tiên kể từ 2021). Thay đổi đáng nhớ: **A01 Broken Access Control** vẫn #1 và **nuốt cả SSRF**, bao trùm **BOLA/BFLA**; **A02 Security Misconfiguration** nhảy #5→#2; **2 hạng mục MỚI**: *Software Supply Chain Failures* và *Mishandling of Exceptional Conditions*. *(KHÔNG có "OWASP Top 10 2024" — đừng nói nhầm.)* → nguồn: owasp.org/Top10/2025, blog.qualys.com, parasoft.com.
- **OAuth 2.1** vẫn là **IETF Internet-Draft** (draft-ietf-oauth-v2-1-15, 3/2026), **chưa thành RFC**, nhưng đã được áp dụng rộng. Nội dung chốt: **PKCE bắt buộc cho MỌI client**, **bỏ Implicit flow**, **bỏ ROPC** (password grant), **exact redirect_uri matching**, **cấm bearer token trong URL**. Gắn với **RFC 9700 — OAuth 2.0 Security BCP** (1/2025). → nguồn: oauth.net/2.1, descope.com, aembit.io.
- **Token storage** (đồng thuận hiện nay): access token để **in-memory**; refresh token trong **HttpOnly + Secure + SameSite cookie** do backend phát; **KHÔNG để localStorage** (XSS đọc được). → nguồn: codercops.com, oneuptime.com.
- **Password storage** (OWASP Password Storage Cheat Sheet): ưu tiên **Argon2id** baseline `m=19 MiB, t=2, p=1` (hoặc `m=46 MiB, t=1`); không có argon2 → **scrypt** `N=2^17, r=8, p=1`; legacy → **bcrypt** cost ≥10 (giới hạn 72 byte); cần FIPS → **PBKDF2-HMAC-SHA256** ≥600k vòng; cân nhắc **pepper** (defense-in-depth). Argon2 chuẩn hoá ở **RFC 9106** (9/2021). **SHA-256 thuần KHÔNG dùng cho mật khẩu.** → nguồn: cheatsheetseries.owasp.org/Password_Storage.
- **npm supply chain** đã bước sang thời "worm tự nhân bản": **Shai-Hulud** (9/2025), **Shai-Hulud 2.0** (11/2025), **Mini Shai-Hulud / TeamPCP** (4–5/2026) — đánh cắp credential dev/CI, dùng OIDC token + Sigstore/Fulcio để **ký provenance SLSA Build L3 hợp lệ cho package độc hại**. Bài học: **provenance chứng minh pipeline, KHÔNG chứng minh code an toàn.** → nguồn: unit42.paloaltonetworks.com, snyk.io, wiz.io, tenable.com.

---

## Mục lục mục con (ID prefix → số câu)

| Tag | Mục con | Số câu |
|---|---|---|
| **M-AUTHN** | Authentication fundamentals (authn vs authz, MFA, brute-force/credential stuffing, passkeys/WebAuthn, account recovery) | 7 |
| **M-PWD** | Credential storage chuyên sâu (argon2id params, salt/pepper, timing-safe, migration) | 5 |
| **M-JWT** | JWT chuyên sâu (cấu trúc, ký, `alg=none`, expiry, revocation, JWKS, claims) | 10 |
| **M-SESS** | Session vs token, cookie security, token storage | 7 |
| **M-OAUTH** | OAuth2 / OIDC flows (auth code + PKCE, OAuth 2.1, client types, scopes, introspection) | 11 |
| **M-AUTHZ** | Authorization models (RBAC/ABAC/ReBAC, IDOR/BOLA/BFLA, least privilege, multi-tenant) | 7 |
| **M-OWASP** | OWASP Top 10:2025 + injection/SSRF/misconfig | 8 |
| **M-XSS** | XSS · CSRF · CSP · web-injection (góc hệ thống) | 6 |
| **M-CRYPTO** | Crypto fundamentals (hash vs encrypt vs encode, đối xứng/bất đối xứng, TLS, HMAC, key mgmt) | 8 |
| **M-SECRET** | Secrets management mức kiến trúc (Vault dynamic, KMS, envelope, rotation, scanning) | 6 |
| **M-SUPPLY** | Supply chain mức mối đe doạ (SBOM, SLSA, provenance, lockfile, lifecycle scripts) | 7 |
| **M-NET** | Network / zero-trust / mTLS / WAF / DDoS / API gateway | 6 |
| **M-DATA** | Data protection & compliance (PII, GDPR, PCI-DSS, encryption at rest, audit log) | 6 |
| **M-TL** | TL-level: threat modeling, secure SDLC, SAST/DAST/SCA, incident response, security governance | 6 |

---

## M-AUTHN — Authentication fundamentals

**M-AUTHN-001** ⭐⭐
"Phân biệt rạch ròi authentication và authorization. Cho một ví dụ mà hệ thống authn đúng nhưng authz sai (và ngược lại)."
> *Dò cái gì:* authn = "anh là ai", authz = "anh được làm gì"; ví dụ authz sai = đã đăng nhập đúng nhưng truy cập được tài nguyên người khác (IDOR); thứ tự authn trước authz.

**M-AUTHN-002** ⭐⭐⭐
"Brute-force, credential stuffing và password spraying khác nhau ra sao? Mỗi loại phòng bằng biện pháp gì?"
> *Dò cái gì:* brute-force = thử nhiều pass 1 account; credential stuffing = tái dùng creds rò rỉ; spraying = 1 pass phổ biến cho nhiều account né lockout; phòng: rate-limit theo IP+account, MFA, breached-password check, CAPTCHA, lockout mềm.

**M-AUTHN-003** ⭐⭐⭐⭐
"MFA: phân biệt các yếu tố (something you know/have/are). Vì sao SMS OTP bị coi là yếu? TOTP vs push vs WebAuthn khác gì về sức chống phishing?"
> *Dò cái gì:* 3 nhóm yếu tố; SMS bị SIM-swap/SS7; TOTP chống được nhiều nhưng vẫn phishing được; **WebAuthn/passkey = phishing-resistant** vì gắn origin + khoá phần cứng.

**M-AUTHN-004** ⭐⭐⭐⭐
"Passkey/WebAuthn hoạt động theo nguyên lý nào và vì sao 'không có gì để đánh cắp trên server'? Khác mật khẩu ở đâu?"
> *Dò cái gì:* public-key crypto (cặp khoá), private key ở authenticator/thiết bị, server chỉ giữ public key; challenge–response ký theo origin → chống phishing/replay; server breach không lộ secret đăng nhập.

**M-AUTHN-005** ⭐⭐⭐⭐
"Luồng 'quên mật khẩu' thiết kế sao cho an toàn? Nêu ít nhất 3 sai lầm phổ biến biến nó thành lỗ hổng account takeover."
> *Dò cái gì:* token reset ngẫu nhiên-đủ-mạnh, single-use, hết hạn ngắn, gửi qua kênh đã xác thực, không tiết lộ email tồn tại hay không; sai lầm: token đoán được, không hết hạn, lộ qua referer, không vô hiệu session cũ.

**M-AUTHN-006** ⭐⭐⭐⭐⭐
"User enumeration là gì? Nó rò rỉ ở những điểm nào (login, signup, reset) và vì sao 'thông báo lỗi thân thiện' lại phản tác dụng về bảo mật?"
> *Dò cái gì:* attacker biết account nào tồn tại qua khác biệt response/timing; điểm rò: message khác nhau, status code, response time; cân bằng UX vs security; trả lời đồng nhất + timing đều.

**M-AUTHN-007** ⭐⭐⭐⭐
"Step-up authentication (xác thực nâng cấp) là gì và khi nào TL nên áp? Cho ví dụ ở fintech."
> *Dò cái gì:* yêu cầu xác thực mạnh thêm cho hành động nhạy cảm (chuyển tiền, đổi email) dù đã đăng nhập; risk-based; ví dụ: re-auth/MFA trước giao dịch lớn.

---

## M-PWD — Credential storage chuyên sâu

> *(Bổ sung cho G-SEC-007 vốn chỉ hỏi "chọn bcrypt hay argon2". M đào tham số, pepper, timing, migration.)*

**M-PWD-001** ⭐⭐⭐
"OWASP khuyến nghị tham số argon2id nào làm baseline hiện nay, và 3 tham số (memory/iterations/parallelism) đánh đổi điều gì? Vì sao memory cost quan trọng hơn time cost trước GPU/ASIC?"
> *Dò cái gì:* baseline ~`m=19 MiB, t=2, p=1` (verify); memory-hard làm GPU/ASIC tốn RAM khó song song hoá → đắt hơn; benchmark đạt ~100–250ms/hash trên hardware của mình. *(verify số tại OWASP Cheat Sheet.)*

**M-PWD-002** ⭐⭐⭐
"Salt và pepper khác nhau ở đâu? Salt lưu cùng hash có sao không? Pepper lưu ở đâu và tại sao là 'defense-in-depth' chứ không thay được salt?"
> *Dò cái gì:* salt = unique/record, lưu cùng hash, chống rainbow table; pepper = secret chung lưu NGOÀI DB (app config/HSM), thêm 1 lớp nếu DB rò mà secret store còn an toàn; pepper không thay salt.

**M-PWD-003** ⭐⭐⭐⭐
"Vì sao SHA-256 (hay MD5) thuần không dùng cho mật khẩu dù 'cũng là hash'? Phân biệt fast hash vs slow/adaptive hash."
> *Dò cái gì:* fast hash → tỷ phép thử/giây trên GPU; mật khẩu cần slow + memory-hard + có salt + tunable work factor để theo kịp phần cứng; SHA-256 hợp cho integrity, không cho password.

**M-PWD-004** ⭐⭐⭐⭐
"So sánh hash khi đăng nhập phải 'timing-safe' nghĩa là gì? Khi nào timing attack thực sự khai thác được và khi nào lo thừa?"
> *Dò cái gì:* dùng constant-time compare cho secret/token; với argon2/bcrypt thì so sánh nội tại đã an toàn, nhưng so sánh token/HMAC thủ công bằng `==` có thể rò; nhận diện đâu là vector thực.

**M-PWD-005** ⭐⭐⭐⭐⭐
"Bạn cần đổi thuật toán hash (bcrypt → argon2id) cho 50 triệu user MÀ không bắt mọi người reset password. Chiến lược migration thế nào?"
> *Dò cái gì:* lưu kèm metadata thuật toán/params trong chuỗi hash; rehash khi user đăng nhập thành công (lazy upgrade); hoặc wrap hash cũ trong hash mới (peppered/double-hash) để migrate ngay; versioning rõ ràng.

---

## M-JWT — JWT chuyên sâu

**M-JWT-001** ⭐⭐
"Cấu trúc một JWT gồm mấy phần, mỗi phần chứa gì? Phần payload có được mã hoá không?"
> *Dò cái gì:* header.payload.signature (base64url); payload **chỉ encode chứ KHÔNG mã hoá** → ai cũng đọc được, đừng để secret trong claim; signature đảm bảo integrity, không đảm bảo confidentiality.

**M-JWT-002** ⭐⭐⭐
"JWT vs session-cookie truyền thống: trade-off chính là gì? Khi nào stateless JWT *không* phải lựa chọn tốt?"
> *Dò cái gì:* JWT stateless, scale ngang dễ, không cần lookup; nhưng **khó thu hồi**, payload phình, đồng bộ logout khó; session server-side thu hồi tức thì nhưng cần store chia sẻ; chọn theo nhu cầu revocation/scale.

**M-JWT-003** ⭐⭐⭐⭐
"Tấn công `alg=none` và confusion `HS256`↔`RS256` là gì? Vì sao thư viện cũ dính, và cách phòng?"
> *Dò cái gì:* `alg:none` bỏ chữ ký; RS256→HS256 confusion = dùng public key làm HMAC secret để giả chữ ký; phòng: **pin/whitelist alg cụ thể** phía verify, không tin `alg` trong header, không dùng cùng key cho 2 mục đích.

**M-JWT-004** ⭐⭐⭐⭐
"Symmetric (HS256) vs asymmetric (RS256/ES256) ký JWT: khác nhau gì và khi nào bắt buộc dùng asymmetric?"
> *Dò cái gì:* HS = 1 secret chia sẻ (issuer & verifier cùng giữ) → rủi ro khi nhiều service; RS/ES = private ký / public verify → nhiều resource server verify mà không giữ secret ký; multi-service/3rd-party verify → asymmetric.

**M-JWT-005** ⭐⭐⭐⭐⭐
"JWT 'không thu hồi được' là vấn đề kinh điển. Liệt kê các chiến lược revocation và đánh đổi của từng cái (vì sao mỗi cái làm mất bớt tính stateless)."
> *Dò cái gì:* access token ngắn hạn + refresh; blacklist theo `jti` (re-introduce state); token version/`tokenVersion` trong DB; rotate signing key; tất cả đều cần lookup state → mất bớt stateless; chọn theo SLA revocation.

**M-JWT-006** ⭐⭐⭐⭐
"Refresh token rotation + reuse detection hoạt động ra sao? Vì sao 'one-time refresh token' phát hiện được token bị đánh cắp?"
> *Dò cái gì:* mỗi lần refresh phát token mới + vô hiệu cái cũ; nếu token cũ được dùng lại → nghi rò rỉ → revoke cả family; family/lineage tracking; chống replay refresh.

**M-JWT-007** ⭐⭐⭐⭐
"Khi verify JWT, ngoài chữ ký bạn PHẢI kiểm tra những claim nào? Bỏ sót cái nào dẫn tới lỗ hổng gì?"
> *Dò cái gì:* `exp` (hết hạn), `nbf`, `iat`, `iss`, `aud`, `sub`, đúng `alg`; bỏ `aud` → token của service A dùng được ở B; bỏ `exp` → token sống mãi; bỏ `iss` → chấp nhận issuer giả.

**M-JWT-008** ⭐⭐⭐⭐
"JWKS endpoint và key rotation: resource server làm sao verify khi issuer xoay khoá ký mà không downtime?"
> *Dò cái gì:* issuer publish public keys ở JWKS (`kid` trong header); verifier fetch + cache theo `kid`, support nhiều key cùng lúc trong giai đoạn rotate; refresh JWKS khi gặp `kid` lạ.

**M-JWT-009** ⭐⭐⭐
"Opaque token vs structured (JWT) token khác gì? Khi nào dùng token introspection thay vì self-validation?"
> *Dò cái gì:* opaque = chuỗi tham chiếu, phải gọi introspection để biết nội dung (dễ revoke, kín claim); JWT self-contained verify cục bộ (nhanh, khó revoke); chọn theo revocation tức thì vs latency.

**M-JWT-010** ⭐⭐⭐⭐⭐
"Đặt access token vào JWT bao to là 'vừa'? Nhồi nhiều role/permission vào claim gây vấn đề gì ở quy mô lớn?"
> *Dò cái gì:* token to → mọi request mang header lớn (băng thông, header limit), permission đóng băng tới khi token hết hạn (đổi quyền không hiệu lực ngay); cân nhắc claim tối thiểu + lookup authz, hoặc giảm TTL.

---

## M-SESS — Session, cookie & token storage

**M-SESS-001** ⭐⭐⭐
"Các thuộc tính cookie bảo mật `HttpOnly`, `Secure`, `SameSite` mỗi cái chặn lớp tấn công nào?"
> *Dò cái gì:* HttpOnly = JS không đọc được → giảm trộm cookie qua XSS; Secure = chỉ gửi qua HTTPS; SameSite (Lax/Strict/None) = giảm CSRF bằng chặn gửi cookie cross-site.

**M-SESS-002** ⭐⭐⭐⭐
"Lưu access token ở localStorage vs HttpOnly cookie vs in-memory: phân tích trade-off XSS/CSRF của từng nơi. Khuyến nghị hiện nay là gì?"
> *Dò cái gì:* localStorage đọc được bởi mọi script (XSS lấy token); cookie chống XSS-đọc nhưng phải lo CSRF (SameSite); **đồng thuận: access token in-memory, refresh token HttpOnly+Secure+SameSite cookie**.

**M-SESS-003** ⭐⭐⭐
"Session fixation là gì và biện pháp chuẩn để phòng?"
> *Dò cái gì:* attacker ép victim dùng session id biết trước; phòng: **regenerate session id sau khi đăng nhập thành công**, không nhận session id từ URL.

**M-SESS-004** ⭐⭐⭐⭐
"Server-side session ở hệ phân tán nhiều instance lưu ở đâu? Trade-off sticky session vs shared session store?"
> *Dò cái gì:* shared store (Redis) để mọi instance đọc được; sticky session đơn giản nhưng mất khi node chết + lệch tải; *(liên hệ A-System Design về stateless scaling — ở đây nhìn từ góc session security).*

**M-SESS-005** ⭐⭐⭐⭐
"Idle timeout vs absolute timeout của session khác gì? Vì sao cần cả hai, và logout 'thật sự' phải làm gì phía server?"
> *Dò cái gì:* idle = hết hạn nếu không hoạt động; absolute = trần tuổi tối đa dù đang dùng; logout phải **invalidate phía server** (xoá session/blacklist token), không chỉ xoá cookie client.

**M-SESS-006** ⭐⭐⭐⭐⭐
"Thiết kế 'logout khỏi mọi thiết bị' (global logout) cho hệ dùng JWT stateless. Vì sao bài này khó và bạn giải thế nào?"
> *Dò cái gì:* JWT stateless không revoke tức thì → cần state: `tokenVersion`/`sessionEpoch` trong DB tăng khi logout-all; verify so version; hoặc short TTL + refresh-token store; nêu trade-off latency.

**M-SESS-007** ⭐⭐⭐⭐
"Khi nào CSRF token là cần thiết và khi nào *không*? Vì sao API stateless dùng Bearer header thường không cần CSRF token nhưng auth-by-cookie thì cần?"
> *Dò cái gì:* CSRF khai thác cookie tự gửi; Authorization header KHÔNG tự gửi cross-site → API Bearer ít rủi ro CSRF; cookie-based cần CSRF token / SameSite. *(đào sâu hơn G-SEC-004.)*

---

## M-OAUTH — OAuth2 / OIDC flows

**M-OAUTH-001** ⭐⭐⭐
"OAuth2 giải quyết bài toán gì? Phân biệt rõ 4 vai: Resource Owner, Client, Authorization Server, Resource Server."
> *Dò cái gì:* **delegated authorization** (cấp quyền truy cập tài nguyên không đưa mật khẩu); ánh xạ đúng 4 vai; OAuth = authorization, không phải authentication.

**M-OAUTH-002** ⭐⭐⭐⭐
"Mô tả Authorization Code flow từng bước. Vì sao token đi qua back-channel (server-to-server) an toàn hơn để token trả thẳng về browser?"
> *Dò cái gì:* redirect tới AS → user consent → AS trả **authorization code** về redirect_uri → backend đổi code (+secret/PKCE) lấy token ở token endpoint; token không lộ qua browser/history.

**M-OAUTH-003** ⭐⭐⭐⭐
"PKCE giải quyết lỗ hổng gì của Authorization Code flow? Giải thích code_verifier/code_challenge. Vì sao OAuth 2.1 bắt buộc PKCE cho *mọi* client kể cả confidential?"
> *Dò cái gì:* chống **authorization code interception/injection** (public client không giữ secret); verifier (random) → challenge (hash) gửi lúc authorize, verifier gửi lúc đổi code; 2.1 mở rộng cho mọi client vì code interception xảy ra cả ở confidential.

**M-OAUTH-004** ⭐⭐⭐⭐⭐
"Implicit flow và ROPC (password grant) từng tồn tại để làm gì, vì sao bị loại bỏ trong OAuth 2.1? Hệ thống cũ còn dùng thì migrate sang đâu?"
> *Dò cái gì:* implicit trả token qua URL fragment (lộ history/referer, không refresh) → bỏ; ROPC app cầm trực tiếp password → bỏ; **migrate sang Authorization Code + PKCE**. *(verify: 2.1 vẫn là draft nhưng đã de-facto.)*

**M-OAUTH-005** ⭐⭐⭐
"Client Credentials flow dùng khi nào? Khác Authorization Code ở chỗ nào về 'ai là chủ tài nguyên'?"
> *Dò cái gì:* machine-to-machine / service-to-service, không có user; client tự xác thực bằng credential của chính nó; không có resource owner người dùng tham gia.

**M-OAUTH-006** ⭐⭐⭐⭐
"OIDC khác OAuth2 ở đâu? Phân biệt `id_token` vs `access_token` — cái nào gửi cho API, cái nào để client đọc danh tính?"
> *Dò cái gì:* OIDC = lớp identity trên OAuth2; **id_token** (JWT, claims danh tính, cho **client** consume); **access_token** để gửi tới **resource server/API**; không gửi id_token làm access token.

**M-OAUTH-007** ⭐⭐⭐⭐
"Vì sao `redirect_uri` phải pre-register và 'exact match'? Open redirect / redirect_uri manipulation dẫn tới tấn công gì?"
> *Dò cái gì:* nếu redirect_uri tuỳ ý → AS gửi code/token tới site attacker → chiếm phiên; exact string match (OAuth 2.1) chặn wildcard/partial; liên hệ authorization code hijack.

**M-OAUTH-008** ⭐⭐⭐⭐
"Scope trong OAuth là gì và vì sao 'least privilege scope' quan trọng? Cho ví dụ scope quá rộng gây rủi ro."
> *Dò cái gì:* scope giới hạn quyền của access token; cấp tối thiểu cần thiết; ví dụ xin `repo` full thay vì read-only → token rò rỉ thiệt hại lớn hơn; consent rõ ràng.

**M-OAUTH-009** ⭐⭐⭐
"Confidential client vs public client khác gì? Cái nào được giữ client_secret và vì sao SPA/mobile không thuộc nhóm đó?"
> *Dò cái gì:* confidential = backend giữ secret an toàn; public = SPA/mobile không giữ secret được (code lộ ở client); public client buộc dùng PKCE thay vì dựa client_secret.

**M-OAUTH-010** ⭐⭐⭐⭐⭐
"Token introspection (RFC 7662) vs verify cục bộ JWT: ở kiến trúc microservices nhiều resource server, bạn chọn cách nào và đánh đổi gì?"
> *Dò cái gì:* introspection = gọi AS mỗi request (revoke tức thì, latency + AS thành bottleneck); JWT local verify (nhanh, khó revoke); hybrid: JWT TTL ngắn + introspection cho hành động nhạy cảm.

**M-OAUTH-011** ⭐⭐⭐⭐
"State parameter trong OAuth flow để làm gì? Bỏ qua nó dẫn tới tấn công gì?"
> *Dò cái gì:* `state` chống **CSRF trên callback** (gắn request với phiên người dùng), kiểm tra khi quay về; bỏ qua → attacker ép code của họ vào phiên victim (login CSRF).

---

## M-AUTHZ — Authorization models

> *(H-GRD đã làm RBAC/ABAC *bằng cơ chế NestJS*. M-AUTHZ làm *khái niệm độc lập framework* + lỗ hổng authz.)*

**M-AUTHZ-001** ⭐⭐⭐
"RBAC, ABAC và ReBAC khác nhau thế nào? Cho một yêu cầu mà RBAC thuần KHÔNG biểu diễn được, buộc lên ABAC/ReBAC."
> *Dò cái gì:* RBAC theo vai; ABAC theo thuộc tính (user/resource/env); ReBAC theo quan hệ (vd 'owner of', kiểu Google Zanzibar); ví dụ RBAC thua: "chỉ chủ tài liệu HOẶC member cùng team được sửa".

**M-AUTHZ-002** ⭐⭐⭐⭐
"IDOR / BOLA (Broken Object Level Authorization) là gì? Vì sao đây là lỗ hổng API #1 thực tế và đứng đầu OWASP A01? Cách phòng?"
> *Dò cái gì:* truy cập object người khác bằng cách đổi id (`/orders/123`) mà server không kiểm chủ sở hữu; phòng: **kiểm authz theo từng object** dựa trên user hiện tại, không tin id client gửi; không dùng id đoán được làm "bảo mật".

**M-AUTHZ-003** ⭐⭐⭐⭐
"BFLA (Broken Function Level Authorization) khác BOLA chỗ nào? Cho ví dụ endpoint admin bị lộ cho user thường."
> *Dò cái gì:* BFLA = truy cập *chức năng/endpoint* không được phép (vd gọi `DELETE /admin/users` bằng token user thường); kiểm quyền ở mức function/route, không chỉ ẩn nút UI.

**M-AUTHZ-004** ⭐⭐⭐⭐⭐
"Multi-tenant SaaS: làm sao đảm bảo tenant A không bao giờ đọc được dữ liệu tenant B? Nêu các tầng phòng thủ và sai lầm 'quên WHERE tenant_id'."
> *Dò cái gì:* tenant scoping ở mọi query (row-level security/PG RLS, hoặc middleware ép filter), tách schema/DB nếu cần; defense-in-depth; nguy hiểm nhất là dựa vào dev nhớ thêm điều kiện thủ công.

**M-AUTHZ-005** ⭐⭐⭐⭐
"Centralized policy engine (OPA/Cedar) vs authz rải rác trong code: khi nào TL nên tách policy ra ngoài? Trade-off gì?"
> *Dò cái gì:* tách policy → nhất quán, audit được, đổi luật không deploy code; chi phí: thêm hạ tầng, latency, độ phức tạp; phù hợp khi authz phức tạp/đa service.

**M-AUTHZ-006** ⭐⭐⭐
"Principle of least privilege áp ở những tầng nào của một backend (DB user, service account, token scope, IAM role)?"
> *Dò cái gì:* nêu ≥3 tầng: DB account quyền tối thiểu, service-to-service scope hẹp, cloud IAM role hẹp, token scope nhỏ; lý do giảm bán kính nổ (blast radius).

**M-AUTHZ-007** ⭐⭐⭐⭐
"Confused deputy problem là gì trong ngữ cảnh service gọi service? Cho ví dụ và cách phòng."
> *Dò cái gì:* service có quyền cao bị lừa thực hiện thay attacker (không kiểm quyền *người gọi gốc*); phòng: truyền/kiểm danh tính người dùng cuối (token passthrough/on-behalf-of), không dùng quyền service làm mặc định.

---

## M-OWASP — OWASP Top 10:2025 & injection

**M-OWASP-001** ⭐⭐⭐
"OWASP Top 10 là gì và KHÔNG phải là gì? Vì sao nó là 'awareness doc' chứ không phải checklist đầy đủ để 'pass' bảo mật?"
> *Dò cái gì:* danh sách rủi ro phổ biến/đồng thuận cộng đồng để nâng nhận thức + ưu tiên; không phải tiêu chuẩn audit đầy đủ; dùng làm điểm khởi đầu, không phải đích.

**M-OWASP-002** ⭐⭐⭐⭐
"Bản OWASP Top 10:2025 có gì mới so với 2021? Nêu ≥2 thay đổi đáng nhớ."
> *Dò cái gì:* 2 hạng mục MỚI (Software Supply Chain Failures, Mishandling of Exceptional Conditions); A01 Broken Access Control vẫn #1 và nuốt SSRF; Security Misconfiguration lên #2. *(verify owasp.org/Top10/2025; biết "không có bản 2024".)*

**M-OWASP-003** ⭐⭐⭐⭐
"Vì sao Broken Access Control nhiều năm đứng #1? Nó gồm những lỗi cụ thể nào (kể ≥3)?"
> *Dò cái gì:* IDOR/BOLA, BFLA, thiếu kiểm quyền ở server (tin client), privilege escalation, CORS lỏng, force browsing; phổ biến + dễ khai thác + khó test tự động.

**M-OWASP-004** ⭐⭐⭐
"SQL injection: cơ chế và cách phòng *đúng bản chất*. Vì sao 'escape thủ công' kém hơn parameterized query?"
> *Dò cái gì:* attacker chèn SQL qua input → server build query string; phòng: **parameterized/prepared statement** tách data khỏi code; escape thủ công dễ sót; *(G-SEC-005 hỏi góc Node — ở đây hỏi bản chất/OWASP.)*

**M-OWASP-005** ⭐⭐⭐⭐
"SSRF (Server-Side Request Forgery) là gì? Vì sao đặc biệt nguy hiểm trên cloud, và phòng thế nào? (2025 SSRF được gộp vào đâu?)"
> *Dò cái gì:* server bị lừa gửi request tới đích nội bộ (metadata endpoint cloud, internal service); cloud → đánh cắp IAM credential qua `169.254.169.254`; phòng: allowlist đích, chặn IP nội bộ, IMDSv2; 2025 gộp vào A01.

**M-OWASP-006** ⭐⭐⭐⭐
"Security Misconfiguration (lên #2/2025) gồm những gì? Cho ví dụ cụ thể trong một backend NestJS/Node thật."
> *Dò cái gì:* default creds, debug mode bật ở prod, verbose error lộ stack/path, header thiếu, cloud bucket public, CORS `*`, port quản trị mở; ví dụ thực tế.

**M-OWASP-007** ⭐⭐⭐⭐
"Cryptographic Failures (A02:2021) thường gặp ở backend là gì? Phân biệt 'data in transit' và 'data at rest' về biện pháp."
> *Dò cái gì:* dùng thuật toán yếu/lỗi thời, không TLS, lưu nhạy cảm dạng plaintext, key cứng; in transit = TLS; at rest = encryption + KMS; phân loại dữ liệu nhạy cảm trước.

**M-OWASP-008** ⭐⭐⭐⭐⭐
"Hạng mục mới 'Mishandling of Exceptional Conditions' (2025) cảnh báo điều gì? Lỗi xử lý exception sao lại thành lỗ hổng bảo mật?"
> *Dò cái gì:* xử lý lỗi/edge-case sai → fail-open (lỗi nhưng vẫn cho qua), rò thông tin qua message, trạng thái không nhất quán, bỏ kiểm tra khi catch; nguyên tắc **fail securely / fail closed**. *(verify owasp.org/Top10/2025.)*

---

## M-XSS — XSS · CSRF · web-injection

**M-XSS-001** ⭐⭐⭐
"Phân biệt stored, reflected và DOM-based XSS. Mỗi loại khác nhau ở 'payload sống ở đâu'?"
> *Dò cái gì:* stored = lưu ở server rồi phục vụ lại; reflected = phản hồi ngay từ request; DOM = JS client tự ghi input vào DOM; gốc chung: input không tin được render thành code.

**M-XSS-002** ⭐⭐⭐⭐
"Cách phòng XSS *đúng bản chất* là gì? Vì sao 'context-aware output encoding' quan trọng hơn 'lọc input'?"
> *Dò cái gì:* encode/escape theo ngữ cảnh đầu ra (HTML/attr/JS/URL), dùng framework auto-escape, tránh `innerHTML`/`dangerouslySetInnerHTML`; lọc input dễ bypass, encode tại điểm xuất mới chắc.

**M-XSS-003** ⭐⭐⭐⭐
"Content Security Policy (CSP) làm gì và vì sao nó là 'lớp phòng thủ thứ hai' chứ không thay được fix XSS? Nêu directive quan trọng."
> *Dò cái gì:* CSP giới hạn nguồn script/style được chạy → giảm tác hại khi XSS lọt; không chữa gốc; `script-src`, `default-src`, nonce/hash, tránh `unsafe-inline`.

**M-XSS-004** ⭐⭐⭐
"CSRF cơ chế ra sao và vì sao chỉ ảnh hưởng tới auth dựa trên thứ 'trình duyệt tự gửi' (cookie)?"
> *Dò cái gì:* trình duyệt tự đính cookie vào request cross-site → thực hiện hành động thay user; token Bearer trong header không tự gửi → không dính; *(liên hệ M-SESS-007.)*

**M-XSS-005** ⭐⭐⭐⭐
"Các biện pháp chống CSRF hiện đại xếp theo độ mạnh: SameSite cookie, synchronizer token, double-submit. Mỗi cái hoạt động và giới hạn ra sao?"
> *Dò cái gì:* SameSite=Lax/Strict chặn phần lớn; synchronizer token (server giữ + so khớp); double-submit (cookie+header trùng); hiểu giới hạn từng cái (vd SameSite=None cần token).

**M-XSS-006** ⭐⭐⭐⭐⭐
"Vì sao XSS làm vô hiệu gần như mọi biện pháp bảo vệ token phía client? Lập luận này ảnh hưởng thế nào tới quyết định lưu token ở đâu?"
> *Dò cái gì:* JS bị chèn chạy cùng quyền app → đọc localStorage/memory/gọi API thay user; HttpOnly cookie chặn *đọc* token nhưng XSS vẫn gọi API được → ưu tiên diệt XSS gốc + giảm bán kính (TTL ngắn, CSP).

---

## M-CRYPTO — Cryptography fundamentals

**M-CRYPTO-001** ⭐⭐⭐
"Phân biệt encoding, hashing và encryption. Cho mỗi cái 1 ví dụ dùng đúng và 1 lỗi dùng sai phổ biến."
> *Dò cái gì:* encoding (base64) = biểu diễn, KHÔNG bảo mật; hashing = một chiều (password, integrity); encryption = hai chiều có key (confidentiality); lỗi: coi base64 là 'mã hoá', encrypt password thay vì hash.

**M-CRYPTO-002** ⭐⭐⭐
"Symmetric vs asymmetric encryption khác gì về key và hiệu năng? Vì sao TLS dùng cả hai?"
> *Dò cái gì:* symmetric 1 key nhanh; asymmetric cặp public/private chậm hơn nhưng giải bài toán trao key; TLS dùng asymmetric để trao đổi/ký rồi chuyển sang symmetric cho dữ liệu.

**M-CRYPTO-003** ⭐⭐⭐⭐
"Mô tả ở mức trực giác TLS handshake làm gì. 'mTLS' thêm điều gì so với TLS thường?"
> *Dò cái gì:* xác thực server qua certificate, thương lượng key, thiết lập kênh mã hoá; **mTLS** = client cũng trình cert → xác thực hai chiều (dùng cho service-to-service zero-trust).

**M-CRYPTO-004** ⭐⭐⭐⭐
"HMAC dùng để làm gì và khác hash thuần ở đâu? Cho một use case backend (vd verify webhook)."
> *Dò cái gì:* HMAC = hash + secret key → đảm bảo integrity *và* authenticity; chống ai cũng tạo được hash; ví dụ: verify chữ ký webhook (Stripe), so sánh timing-safe.

**M-CRYPTO-005** ⭐⭐⭐⭐
"Encryption at rest và in transit khác nhau gì, và 'encryption at rest' bảo vệ trước mối đe doạ nào — KHÔNG bảo vệ trước cái gì?"
> *Dò cái gì:* in transit = TLS trên đường truyền; at rest = mã hoá ổ đĩa/DB; at rest chống mất ổ cứng/backup rò, **không** chống app bị chiếm (app có quyền giải mã); cần thêm field-level/tokenization cho dữ liệu cực nhạy.

**M-CRYPTO-006** ⭐⭐⭐⭐⭐
"Envelope encryption và KMS hoạt động ra sao? Vì sao không mã hoá trực tiếp bằng master key?"
> *Dò cái gì:* data mã hoá bằng **data key (DEK)**; DEK được mã hoá bằng **key encryption key (KEK/master)** giữ trong KMS/HSM; master key không rời KMS → rotate/audit dễ, giảm phơi bày key chính.

**M-CRYPTO-007** ⭐⭐⭐
"Vì sao 'tự viết thuật toán mã hoá' (roll your own crypto) là điều cấm kỵ? TL nên hướng team dùng gì?"
> *Dò cái gì:* crypto khó đúng, dễ lỗ hổng tinh vi, cần kiểm chứng học thuật; dùng thư viện/chuẩn đã được kiểm định (libsodium, AEAD như AES-GCM/ChaCha20-Poly1305), không tự chế.

**M-CRYPTO-008** ⭐⭐⭐⭐
"AEAD (vd AES-GCM) cung cấp gì mà 'mã hoá thuần' thiếu? Nonce reuse trong GCM gây hậu quả gì?"
> *Dò cái gì:* AEAD = confidentiality + integrity/authenticity cùng lúc; chống sửa ciphertext; **nonce reuse trong GCM phá huỷ bảo mật** (lộ key/authentication) → nonce phải unique/đếm cẩn thận.

---

## M-SECRET — Secrets management (kiến trúc)

> *(G-SEC-008 đã hỏi "env/.env + secret manager cơ bản". M làm phần kiến trúc/vận hành.)*

**M-SECRET-001** ⭐⭐⭐
"Vì sao `.env` + biến môi trường là 'đủ tốt cho dev' nhưng chưa đủ cho production? Nêu giới hạn của env vars."
> *Dò cái gì:* env dễ rò qua log/crash dump/`/proc`, không rotate được tập trung, không audit ai đọc, dễ commit nhầm; prod cần secret manager có versioning/audit/rotation.

**M-SECRET-002** ⭐⭐⭐⭐
"HashiCorp Vault 'dynamic secrets' là gì và nó giảm rủi ro thế nào so với secret tĩnh dài hạn (vd DB password cố định)?"
> *Dò cái gì:* Vault sinh credential ngắn hạn theo yêu cầu (vd DB user tạm), tự thu hồi khi hết lease; giảm cửa sổ phơi bày, không còn 'một password dùng mãi cho mọi service'.

**M-SECRET-003** ⭐⭐⭐⭐
"Secret rotation: vì sao cần, và làm sao rotate mà không downtime? Nêu pattern dual-secret/overlap."
> *Dò cái gì:* giới hạn thiệt hại khi rò; rotate cần giai đoạn chấp nhận cả key cũ+mới (grace window) rồi mới gỡ cũ; tự động hoá thay vì thủ công.

**M-SECRET-004** ⭐⭐⭐⭐⭐
"Workload làm sao lấy secret an toàn mà KHÔNG có 'secret zero' (secret để lấy secret) hardcode? Nêu hướng giải (workload identity / OIDC)."
> *Dò cái gì:* bài toán bootstrap trust; giải bằng workload identity (cloud IAM role gắn vào instance/pod, OIDC federation cho CI) → đổi danh tính nền tảng lấy secret, không nhúng credential gốc.

**M-SECRET-005** ⭐⭐⭐
"Secret scanning trong CI/repo: nó chặn loại sự cố nào? Nếu secret đã lỡ commit vào git history thì xử lý đúng là gì?"
> *Dò cái gì:* phát hiện key/token vô tình commit (gitleaks/trufflehog/GitHub secret scanning); **đã lộ thì phải REVOKE/rotate ngay**, xoá history chỉ là thứ yếu (coi như đã rò).

**M-SECRET-006** ⭐⭐⭐⭐
"Trong Kubernetes, vì sao 'K8s Secret' mặc định *không phải* mã hoá thật, và TL nên làm gì để bảo vệ secret ở tầng cluster?"
> *Dò cái gì:* K8s Secret chỉ base64 (không mã hoá) lưu trong etcd; cần bật **encryption at rest cho etcd**, RBAC hạn chế đọc secret, hoặc external secret store (Vault/CSI driver, External Secrets Operator).

---

## M-SUPPLY — Supply chain (mức mối đe doạ)

> *(G-SEC-006 hỏi "npm audit/lockfile/Snyk cơ bản". M làm mức threat model + chuẩn hiện đại + bài học worm 2025–2026.)*

**M-SUPPLY-001** ⭐⭐⭐
"Vì sao OWASP 2025 nâng 'Software Supply Chain Failures' thành hạng mục riêng? Phạm vi nó rộng hơn 'dependency lỗi thời' chỗ nào?"
> *Dò cái gì:* không chỉ thư viện cũ mà cả build tool bị chiếm, CI/CD pipeline, dependency độc hại, registry; tấn công gián tiếp qua 'mắt xích yếu nhất'; ví dụ SolarWinds/npm worm.

**M-SUPPLY-002** ⭐⭐⭐⭐
"Mô tả lớp tấn công 'self-replicating worm' kiểu Shai-Hulud (2025–2026). Nó lan thế nào và vì sao nguy hiểm hơn typosquatting đơn lẻ?"
> *Dò cái gì:* worm đánh cắp token dev/CI rồi tự publish phiên bản độc cho các package khác của nạn nhân → lan theo cấp số nhân, biến CI thành hạ tầng phát tán; khác typosquatting ở tính tự nhân bản. *(verify unit42/snyk.)*

**M-SUPPLY-003** ⭐⭐⭐⭐⭐
"Mini Shai-Hulud (2026) tạo được provenance SLSA Build L3 *hợp lệ* cho package độc hại. Bài học cốt lõi về 'provenance chứng minh điều gì' là gì?"
> *Dò cái gì:* provenance attest **pipeline/identity đã build**, KHÔNG attest **code an toàn**; token OIDC bị trộm → Sigstore ký thật cho artifact xấu; verify provenance là cần nhưng không đủ; phải bảo vệ branch/commit/CI trust.

**M-SUPPLY-004** ⭐⭐⭐⭐
"Lifecycle scripts (`preinstall`/`postinstall`) của npm là vector tấn công thế nào? TL làm gì trong CI để giảm rủi ro?"
> *Dò cái gì:* script chạy mã tuỳ ý khi cài → thực thi payload trên máy dev/CI; giảm: `--ignore-scripts` trong CI, review postinstall, sandbox build, hạn chế egress mạng từ runner.

**M-SUPPLY-005** ⭐⭐⭐
"`npm install` vs `npm ci` khác gì về bảo mật/độ tái lập? Vì sao lockfile + `npm ci` quan trọng trong pipeline?"
> *Dò cái gì:* `npm ci` cài đúng lockfile, deterministic, fail nếu lệch; tránh kéo version mới bất ngờ; lockfile khoá integrity hash → phát hiện tamper.

**M-SUPPLY-006** ⭐⭐⭐⭐
"SBOM là gì và nó giúp ích lúc nào trong một sự cố supply-chain (vd 'package X vừa bị nhiễm — chúng ta có dùng không')?"
> *Dò cái gì:* Software Bill of Materials = danh mục mọi thành phần/version; khi có CVE/incident → tra ngay tổ chức có dính không + ở đâu; tăng tốc phản ứng & vá.

**M-SUPPLY-007** ⭐⭐⭐⭐⭐
"Thiết kế chính sách dependency cho team (TL góc nhìn): nêu ≥4 biện pháp giảm rủi ro supply chain mà KHÔNG làm team tê liệt."
> *Dò cái gì:* pin version + lockfile, SCA tự động (Snyk/Dependabot) có triage, tối thiểu hoá deps, kiểm maintainer/độ phổ biến, short-lived scoped publish token + MFA, mirror/private registry, vendoring khi cần.

---

## M-NET — Network / zero-trust / mTLS

**M-NET-001** ⭐⭐⭐
"Mô hình 'zero-trust' khác mô hình 'perimeter (castle-and-moat)' truyền thống ở triết lý nào?"
> *Dò cái gì:* không tin mặc định dù ở trong mạng nội bộ; xác thực/uỷ quyền mọi request, mọi service; 'never trust, always verify'; vs vành đai chỉ chặn ở rìa.

**M-NET-002** ⭐⭐⭐⭐
"mTLS giữa các service giải quyết gì mà API key/JWT không hoàn toàn giải? Service mesh (Istio) đỡ phần nào?"
> *Dò cái gì:* xác thực danh tính *cả hai phía* ở tầng transport + mã hoá; mesh tự cấp/xoay cert + áp mTLS mà không sửa code app (sidecar); tách concern bảo mật mạng khỏi business logic.

**M-NET-003** ⭐⭐⭐
"WAF là gì và nó chặn/không chặn loại tấn công nào? Vì sao WAF không thay được fix code?"
> *Dò cái gì:* lọc HTTP độc hại (SQLi/XSS pattern, rate abuse) ở rìa; là lớp phòng thủ bổ sung, dễ bypass/false positive, không sửa lỗ hổng gốc.

**M-NET-004** ⭐⭐⭐⭐
"Phân biệt rate limiting (góc abuse/DoS) với throttling theo nghiệp vụ. Ở tầng nào nên đặt (gateway vs app) và vì sao chia sẻ state khi scale ngang?"
> *Dò cái gì:* rate limit chống lạm dụng/DoS, nên ở gateway/edge + app; store dùng chung (Redis) khi nhiều instance để giới hạn toàn cục; *(thuật toán token bucket/sliding window đã ở F-RL — ở đây góc security/abuse).*

**M-NET-005** ⭐⭐⭐⭐⭐
"Tấn công DDoS ở các tầng (L3/4 vs L7) khác nhau gì? TL chuẩn bị phòng thủ thế nào (kể cả thứ không tự làm được)?"
> *Dò cái gì:* L3/4 = flood băng thông/gói; L7 = request hợp lệ làm cạn tài nguyên app; phòng: CDN/scrubbing (Cloudflare/AWS Shield), rate limit, autoscale, graceful degradation; nhận ra giới hạn tự host.

**M-NET-006** ⭐⭐⭐⭐
"API Gateway đảm nhận các trách nhiệm bảo mật nào (authn tập trung, rate limit, mTLS termination)? Rủi ro 'trust ngầm sau gateway' là gì?"
> *Dò cái gì:* gateway tập trung authn/rate-limit/TLS; rủi ro: service nội bộ tin mù request đã qua gateway → nếu bypass gateway thì toang; zero-trust vẫn cần authz ở từng service.

---

## M-DATA — Data protection & compliance

**M-DATA-001** ⭐⭐⭐
"PII là gì? TL nên phân loại dữ liệu thế nào trước khi quyết định biện pháp bảo vệ?"
> *Dò cái gì:* thông tin định danh cá nhân (tên, email, SSN, vị trí...); phân loại theo độ nhạy (public/internal/confidential/restricted) → áp biện pháp tương xứng; biết dữ liệu ở đâu (data mapping).

**M-DATA-002** ⭐⭐⭐⭐
"Tokenization vs encryption khác gì khi bảo vệ dữ liệu thẻ/PII? Vì sao PCI-DSS thích tokenization để giảm phạm vi?"
> *Dò cái gì:* encryption = mã hoá có thể giải; tokenization = thay bằng token vô nghĩa, ánh xạ giữ ở vault tách biệt; giảm hệ thống chạm dữ liệu thật → thu hẹp scope audit.

**M-DATA-003** ⭐⭐⭐
"GDPR đặt ra nghĩa vụ kỹ thuật nào ảnh hưởng trực tiếp tới thiết kế backend (right to erasure, data minimization, breach notification)?"
> *Dò cái gì:* xoá được dữ liệu user (kể cả backup/log), thu thập tối thiểu, lưu có mục đích/thời hạn, báo cáo vi phạm trong hạn; ảnh hưởng schema, retention, audit.

**M-DATA-004** ⭐⭐⭐⭐
"Data minimization và retention policy ảnh hưởng thiết kế thế nào? Vì sao 'lưu mọi thứ mãi mãi' là rủi ro bảo mật chứ không chỉ chi phí?"
> *Dò cái gì:* dữ liệu không lưu = không thể bị trộm; đặt TTL/retention, purge tự động, không log dữ liệu nhạy cảm; giảm bán kính nổ khi breach.

**M-DATA-005** ⭐⭐⭐⭐
"Audit logging cho hệ nhạy cảm: log cái gì, KHÔNG log cái gì, và làm sao đảm bảo log không bị sửa (tamper-evident)?"
> *Dò cái gì:* log ai-làm-gì-khi-nào trên hành động nhạy cảm; **không log** secret/PII thô/password/token; append-only/WORM, ký/hash chuỗi, tách quyền ghi-đọc; phục vụ forensics.

**M-DATA-006** ⭐⭐⭐⭐⭐
"Field-level / application-level encryption khác encryption at rest của DB ra sao? Khi nào fintech/healthcare buộc dùng, và đánh đổi gì (query, index)?"
> *Dò cái gì:* mã hoá từng field ở app trước khi vào DB → DBA/DB breach không đọc được; at-rest của DB không chống app/DBA có quyền; đánh đổi: mất khả năng query/index/sort trực tiếp trên field mã hoá.

---

## M-TL — TL-level: SDLC, threat modeling, governance

**M-TL-001** ⭐⭐⭐⭐
"Threat modeling là gì? Giải thích nhanh STRIDE và cho ví dụ áp vào một API login."
> *Dò cái gì:* tìm mối đe doạ có hệ thống trước khi code; STRIDE = Spoofing/Tampering/Repudiation/Info disclosure/DoS/Elevation; áp vào login: spoof identity, brute-force (DoS), token tampering...

**M-TL-002** ⭐⭐⭐⭐
"SAST, DAST, SCA, secret scanning khác nhau gì và đặt ở đâu trong CI/CD pipeline? Vì sao cần nhiều loại?"
> *Dò cái gì:* SAST = quét source (sớm), DAST = test app đang chạy (runtime), SCA = quét dependency/CVE, secret scanning = lộ key; mỗi loại bắt lớp khác nhau → defense-in-depth, gate ở pipeline.

**M-TL-003** ⭐⭐⭐⭐⭐
"'Shift-left security' nghĩa là gì và TL áp dụng cụ thể ra sao mà không biến security thành nút thắt cổ chai làm chậm team?"
> *Dò cái gì:* đưa kiểm tra bảo mật sớm (design review, pre-commit, CI gate) thay vì pentest cuối; tự động hoá + triage ưu tiên theo rủi ro; security as enablement, không chỉ gatekeeper.

**M-TL-004** ⭐⭐⭐⭐
"Một thư viện bạn đang dùng vừa công bố CVE nghiêm trọng (vd Log4Shell/npm worm). Quy trình phản ứng (incident response) của bạn theo bước nào?"
> *Dò cái gì:* xác định có dùng không + ở đâu (SBOM), đánh giá khả năng khai thác/exposure, mitigate tạm (WAF/disable feature), patch/rollback, rotate credential nếu nghi rò, hậu kiểm + postmortem.

**M-TL-005** ⭐⭐⭐⭐⭐
"Đặt 'security gate' trong pipeline: chặn deploy khi quét ra lỗ hổng. Làm sao tránh tình huống alert fatigue / team disable luôn cái gate?"
> *Dò cái gì:* phân ngưỡng theo severity (block critical, warn medium), bỏ false positive, có exception process minh bạch, đo & giảm noise; cân bằng an toàn vs vận tốc; ownership rõ.

**M-TL-006** ⭐⭐⭐⭐
"'Secure by default' nghĩa là gì ở cấp thiết kế? Cho 2 ví dụ một thiết lập mặc định an toàn vs nguy hiểm trong backend."
> *Dò cái gì:* mặc định an toàn để dev khó làm sai; ví dụ: deny-by-default authz, TLS bật sẵn, CORS đóng, không log secret, validation bật; vs default permissive (open CORS, debug on).

---

## 🔑 Gợi ý dùng tiếp (sau Bước A)

- **Bước B (dựng giáo trình ngược):** gom 90 câu này thành các BÀI HỌC theo cụm — gợi ý thứ tự dependency: *Crypto fundamentals → Authn/Password → JWT/Session → OAuth/OIDC → Authz models → OWASP/XSS → Secrets → Supply chain → Network/Data → TL governance.*
- **Bước D (phỏng vấn live):** mỗi đợt 10–15 câu trộn dễ→khó; **hỏi xong DỪNG, chờ trả lời** rồi mới chấm theo dòng *"dò cái gì"*.
- **Spaced repetition:** chèn lại các câu ⭐⭐⭐⭐⭐ (M-JWT-005, M-OAUTH-004/010, M-SUPPLY-003, M-AUTHZ-004, M-TL-003/005) ở đợt sau.

> ⚠️ Trước khi học/chấm: **verify lại** mọi mốc đánh dấu *(verify)* — đặc biệt OWASP Top 10:2025 (owasp.org/Top10/2025), OAuth 2.1 (oauth.net/2.1), OWASP Password Storage Cheat Sheet, và tình hình npm supply-chain — vì đây là phần đổi nhanh nhất của mục M.
