# AntigraHunter — AI Instruction Document
> **Version:** 0.3 | **Platform:** Antigravity Native Agent | **Last Updated:** 2026-06-02

---

## 🎯 What is AntigraHunter?

AntigraHunter là một **Antigravity-native security skill**. Nó **không phải standalone script** — nó là một tập hợp các sub-agent chạy ngay trong môi trường Antigravity, sử dụng các tools sẵn có (file read, grep, run_command, HTTP requests).

**Cách dùng:** Mày chỉ cần nói chuyện với AI:

```
bugfinding -f ./webapp -url https://target.com -h "Authorization: Bearer eyJ..."
```

AI sẽ tự đọc code, scan lỗ hổng, verify POC, và xuất báo cáo HTML — **không cần API key ngoài, không cần setup thêm bất cứ thứ gì**.

---

## 🔴 ABSOLUTE LAW — THE IRON RULE

> **No working POC = No bug report. Zero exceptions.**

- Không "potential", không "possibly", không "could be exploited".
- Mọi bug trong report cuối **phải** có HTTP request thực tế đã được gửi tới target URL và response chứng minh khai thác thành công.
- Nếu POC fail sau MAX_RETRIES → **Bỏ hẳn, không log, không mention**.
- **Ngoại lệ duy nhất:** Bug UNCONFIRMED do thiếu auth/role cao hơn — xử lý theo quy trình phân loại §Phase 2.4.

---

## 🏗️ Multi-Agent Architecture (Antigravity Native)

AntigraHunter được thiết kế theo mô hình **Orchestrator + Specialist Sub-Agents**, tất cả chạy trong Antigravity:

```
User input: "bugfinding -f ./app -url https://target.com -h 'Bearer xyz'"
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │     ORCHESTRATOR AGENT         │
                    │  (Main AI — điều phối phases)  │
                    │  - Parse CLI input             │
                    │  - Quản lý state machine       │
                    │  - Spawn & manage sub-agents   │
                    │  - Quyết định retry/escalate   │
                    └───────┬───────────────────────┘
                            │ spawn
              ┌──────────────┼───────────────┬───────────────────────┐
              ▼              ▼               ▼                       ▼
        ┌──────────┐  ┌──────────┐  ┌──────────────┐          ┌──────────┐
        │  RECON   │  │ SCANNER  │  │  VERIFIER    │          │ REPORTER │
        │  AGENT   │  │  AGENT   │  │  AGENT       │          │  AGENT   │
        │ Phase 0  │  │ Phase 1  │  │  Phase 2     │          │ (Final)  │
        └──────────┘  └──────────┘  └──────┬───────┘          └──────────┘
                                           │ on PAYLOAD_FILTERED / context_exposures
                                           ▼
                                 ┌───────────────────────┐
                                 │   POLICY & SANDBOX    │
                                 │    BYPASS AGENT       │
                                 │   (On-demand only)    │
                                 └───────────────────────┘
```

### Agent Roles & Tools

| Agent | Nhiệm vụ | Antigravity Tools Sử Dụng |
|---|---|---|
| **Orchestrator** | Parse input, điều phối phases, quản lý RCA Debate Loop, quyết định retry/escalate | `invoke_subagent`, `send_message`, state JSON |
| **Recon Agent** | Tech stack detection, route mapping, auth analysis, sink extraction, capability & context exposure mapping | `list_dir`, `view_file`, `grep_search` |
| **Scanner Agent** | Taint analysis, call graph mapping, POC template construction, RCA Structural Fix | `view_file`, `grep_search` |
| **Verifier Agent** | Gửi HTTP POC requests, phân tích response, soạn Failure Report có cấu trúc | `run_command` (curl/httpx), HTTP response analysis |
| **Policy & Sandbox Bypass Agent** | Boundary/Capability analyst: đánh giá Context Exposure, phân tích Sandbox constraints, sinh payload bypass | `view_file`, `grep_search` |
| **Reporter Agent** | Tổng hợp CONFIRMED findings → HTML report với Executive Summary table | `write_to_file`, HTML generation |

---

## 📋 CLI Input Format

```bash
bugfinding -f <path_to_source_folder> -url <target_web_url> -h <auth_header>
```

### Flags

| Flag | Mô tả | Bắt buộc | Ví dụ |
|---|---|---|---|
| `-f` | Đường dẫn tới thư mục source code cần audit | ✅ | `-f ./webapp/` |
| `-url` | URL gốc của web app đang chạy (cho POC requests) | ✅ | `-url https://staging.myapp.com` |
| `-h` | HTTP Header xác thực (có thể dùng nhiều lần) | ❌ | `-h "Authorization: Bearer eyJ..."` |

### Ví dụ đầy đủ

```bash
# Single auth header
bugfinding -f ./webapp -url https://target.com -h "Authorization: Bearer eyJhbGci..."

# Multiple headers (session + tenant)
bugfinding -f ./webapp -url https://target.com -h "Cookie: session=abc123" -h "X-Tenant-ID: 1"

# No auth (public endpoint scan)
bugfinding -f ./webapp -url https://target.com
```

---

## ⚙️ Phase 0 — Pre-Reconnaissance (Recon Agent)

**Mục tiêu:** Map toàn bộ attack surface trước khi scan sâu.  
**Tools:** `list_dir`, `view_file`, `grep_search`  
**Output:** Context cho Phase 1 (không cần persist ra file)

### 0.1 — Tech Stack Detection

Detect framework từ project files:

| File | Framework/Lang |
|---|---|
| `package.json` | Node.js → check `express`, `nestjs`, `next`, `koa`, `fastify` |
| `composer.json` | PHP → check `laravel`, `symfony`, `slim` |
| `requirements.txt` / `pyproject.toml` | Python → check `django`, `flask`, `fastapi` |
| `pom.xml` / `build.gradle` | Java → check `spring-boot` |
| `go.mod` | Go → check `gin`, `echo`, `fiber`, `chi` |
| `Gemfile` | Ruby → check `rails` |
| `*.csproj` | C# → check `aspnet` |

**Output context:** `{ language, framework, orm, template_engine, auth_library }`

### 0.2 — Route & Endpoint Mapping

Extract tất cả HTTP endpoints dựa vào framework:

| Framework | Pattern Scan |
|---|---|
| **Express** | `app.get\|post\|put\|delete\|patch\|use`, `router.*` |
| **NestJS** | `@Controller`, `@Get`, `@Post`, `@Put`, `@Delete`, `@Patch` |
| **Laravel** | `routes/web.php`, `routes/api.php` — `Route::get\|post\|put\|delete` |
| **Django** | `urls.py` → `urlpatterns`, `path()`, `re_path()` |
| **FastAPI** | `@app.get\|post\|put\|delete`, `@router.*` |
| **Spring Boot** | `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@DeleteMapping` |
| **Rails** | `config/routes.rb` → `get`, `post`, `resources`, `namespace` |

**Output context:** List of `{ method, path, controller_file, middleware[], auth_required }`

### 0.3 — Auth Mechanism Analysis

Identify auth mechanisms và protected routes:
- **JWT:** grep `jwt.verify`, `jwt.decode`, `Bearer`, secret key source
- **Session:** grep `session`, `cookie-parser`, `express-session`
- **OAuth:** detect `/auth/callback`, `/oauth/*`, `passport.*`
- **Custom Middleware:** trace middleware chain trên từng route group

**Output context:** `{ auth_type, protected_routes[], public_routes[], middleware_files[] }`

### 0.4 — Static Sink Extraction

Dùng `grep_search` với patterns từ SKILL.md §3 Sink Matrix, extract tất cả dangerous function calls.

**Ưu tiên scan theo thứ tự:**
1. Routes/Controllers (entry points)
2. Services/Use Cases (business logic)
3. Repositories/DAO (data layer)
4. Utils/Helpers (shared functions)

**BỎ QUA hoàn toàn** (từ SKILL.md §6):
- Directories: `node_modules`, `build`, `out`, `target`, `bin`, `obj`, `vendor`, `docs`, `.github`, `.vscode`, `.git`, `assets`, `.next`, `.nuxt`, `coverage`, `.cache`, `.tmp`
- Extensions: `.md`, `.txt`, `.pdf`, `.png`, `.jpg`, `.svg`, `.ico`, `.lock`, `.sum`, `.exe`, `.dll`, `.pyc`, `.class`, `.map`, `.test.ts`, `.test.js`, `.spec.ts`, `.spec.js`

### 0.5 — Capability & Context Exposure Mapping

Scan cho các tính năng nguy hiểm có thể phơi bày server-side authority cho user:
- **Template/expression engines:** `template.render(`, `from_string(`, `SandboxedEnvironment`, `Environment(`, `render_template_string`
- **Context/global construction:** `context.update(`, `environment.globals`, `filters.update(`, `_context[`
- **ORM/model resolver exposure:** `ObjectType.objects`, `ContentType.objects`, `apps.get_model`, `_default_manager`
- **Capability surfaces:** `export`, `report`, `dashboard`, `formula`, `query_builder`, `workflow`, `automation`, `webhook`

**Output context thêm 3 fields:**
- `capability_surfaces[]` — feature name, route/UI entry, user permission required, server authority used
- `context_exposures[]` — file, line, exposed object/class/function, why dangerous
- `sensitive_model_access[]` — users, sessions, tokens, secrets potentially accessible

> ⚠️ Nếu `context_exposures[]` không rỗng → Orchestrator sẽ spawn **Policy & Sandbox Bypass Agent** sau Phase 1.

---

## 🔍 Phase 1 — Static Code Analysis (Scanner Agent)

**Input:** Phase 0 context + SKILL.md  
**Output:** `findings_draft` — List of suspected vulnerabilities với POC templates

### 1.1 — Call Graph Tracing

Với mỗi sink tìm được, trace ngược:

```
Route Definition
    └── Middleware Chain (auth check? input validation?)
        └── Controller/Handler Function
            └── Service/Business Logic Function
                └── Repository/DB Layer
                    └── SINK (dangerous function call)
```

**Quan trọng:** Không dừng ở Controller. Nếu Controller gọi `UserService.search(data)`, PHẢI mở `UserService` để xem data được xử lý tiếp như nào.

### 1.2 — Taint Analysis (Inter-Procedural)

Thực hiện đúng theo SKILL.md §4:

**1. Source Marking** — Chỉ mark data từ HTTP (params, headers, cookies, body, uploaded files). **BỎ QUA** data từ env vars, config files.

**2. Propagation** — Track variable qua assignments, string concat, function params.

**3. Sanitizer Review** — Với mỗi sanitizer gặp:
- Double encoding bypass?
- Type confusion (String → Array input)?
- Regex thiếu `^$`, `/m`, `/i` flag?
→ Nếu sanitizer yếu, **vẫn coi data là tainted** (tránh false negative)

**4. Sink Check** — Data tainted có reach sink không? Có bypass được các lớp bảo vệ không?

### 1.3 — Vulnerability Classification Schema

```json
{
  "id": "VULN-001",
  "type": "SQLi | XSS | SSRF | RCE | IDOR | AuthBypass | PathTraversal | SSTI | Deserialization | ...",
  "severity": "Critical | High | Medium",
  "endpoint": "POST /api/users/search",
  "vulnerable_param": "q",
  "data_flow": "req.body.q → UserController.ts:45 → UserService.search():23 → db.query():89",
  "sink": {
    "file": "repositories/user.repo.ts",
    "line": 89,
    "code": "db.query(`SELECT * FROM users WHERE name = '${q}'`)"
  },
  "sanitizers_found": [],
  "bypass_analysis": "No sanitizer detected. Direct string interpolation.",
  "poc_template": {
    "method": "POST",
    "path": "/api/users/search",
    "headers": { "Content-Type": "application/json" },
    "body": { "q": "' OR '1'='1'--" }
  },
  "chain_with": null,
  "status": "PENDING_VERIFICATION"
}
```

### 1.4 — Chain Analysis

Sau khi collect tất cả individual findings, tự hỏi:
> "Finding này có thể combine với finding nào khác không?"

| Chain Pattern | Kết quả |
|---|---|
| Open Redirect + OAuth Callback | Account Takeover |
| SSRF + Internal Endpoint | Data Exfiltration |
| XSS + CSRF Token Leak | Authenticated Action |
| Leaked Endpoint + IDOR | Unauthorized Data Access |
| Auth Bypass + Admin Function | Privilege Escalation |

Nếu chain được build → document thứ tự request chính xác trong `poc_template`.

---

## ✅ Phase 2 — POC Verification (Verifier Agent)

**Input:** `findings_draft` từ Phase 1  
**Tool:** `run_command` với `curl` hoặc Python `httpx`  
**Mục tiêu:** Prove it works hoặc prove it doesn't.

### 2.1 — HTTP Request Construction

Với mỗi finding:

```bash
# Auto-build từ poc_template + CLI -h headers + -url base
curl -s -i -X POST \
  "https://target.com/api/users/search" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ..." \
  -d '{"q": "'"'"' OR '"'"'1'"'"'='"'"'1'"'"'--"}'
```

### 2.2 — Success Criteria

| Bug Type | Dấu hiệu CONFIRMED |
|---|---|
| **SQL Injection** | SQL error message trong response, data leakage, boolean difference (khác nhau khi `1=1` vs `1=2`), time delay với `SLEEP(5)` |
| **XSS** | `<script>alert(1)</script>` reflected nguyên vẹn trong response |
| **SSRF** | Response từ internal IP, DNS/HTTP callback nhận được |
| **RCE** | Output của `id`, `whoami`, hoặc time delay khi `sleep 10` |
| **IDOR** | Lấy được data của user khác (khác user_id) |
| **Auth Bypass** | HTTP 200 trên endpoint protected mà không có token |
| **Path Traversal** | Nội dung `/etc/passwd` trong response |
| **Open Redirect** | `Location` header redirect tới attacker domain |
| **SSTI** | `{{7*7}}` → `49` trong response |

### 2.3 — ⚠️ RCA DEBATE LOOP (Retry Protocol)

Khi Verifier fail, **không retry mù**. Orchestrator phân loại nguyên nhân và giao cho đúng agent xử lý:

```
POC FAILED
    │
    ▼
Verifier soạn Failure Report (status code, response body, observations, hypothesis)
    │
    ▼
Orchestrator phân tích Root Cause Category:
    │
    ├── Structural Error:
    │   (WRONG_PARAM_NAME, WRONG_URL_PATH, WRONG_HTTP_METHOD,
    │    WRONG_CONTENT_TYPE, MISSING_PREREQUISITE, WRONG_PAYLOAD_STRUCTURE)
    │       → Giao cho SCANNER để đọc lại code, sửa cấu trúc POC
    │       → Scanner trả về RCA_Response + revised_poc_template
    │
    └── Filter/WAF/Sandbox: (PAYLOAD_FILTERED)
            → Giao cho POLICY & SANDBOX BYPASS AGENT
            → Agent đánh giá Sandbox constraints + sinh payload bypass
            → Trả về RCA_Response + mutated_poc_templates[]
    │
    ▼
Orchestrator announce: "🔁 RCA [Scanner|Bypass]: {one_line_diagnosis}"
    → Verifier retry với revised template (attempt N+1)
    │
    ├── attempt 1 fail → RCA Round 1
    ├── attempt 2 fail → RCA Round 2
    └── attempt 3 fail → Orchestrator classify: DISCARD | AUTH_LIMITED
```

**MAX_RETRIES:** 3 vòng RCA per finding. Mỗi vòng có Failure Report + RCA_Response cụ thể.

### 2.4 — UNCONFIRMED Finding Classification

Sau khi fail MAX_RETRIES, AI **tự chấm điểm** finding vào 2 bucket:

#### ❌ DISCARD (bỏ hoàn toàn, không log, không mention)
- Không có user-controlled input path
- Dead code / không reachable
- Lỗi implementation nhưng không có impact thực tế
- Data source từ env/config (không phải HTTP input)
- Rõ ràng là false positive

#### ⚠️ AUTH_LIMITED (ghi chú nội bộ, KHÔNG đưa vào report chính)
- Lỗ hổng tồn tại trong code (taint analysis confirms)
- POC fail vì cần role/permission cao hơn mức đang có
- Ví dụ: "Endpoint chỉ accessible với admin role, không có admin token để test"
- **Không báo cáo** theo Iron Rule, nhưng Orchestrator ghi chú: "Potential finding, cần retest với admin credentials"

> **Tuyệt đối không có category nào khác.** DISCARD hoặc AUTH_LIMITED. Không có "gray area" được đưa vào report.

---

## 📊 Final Report — HTML Format

Chỉ chứa `CONFIRMED` findings. Output là file HTML có thể mở bằng browser.

### Report Structure

```html
<!-- report.html -->
AntigraHunter Security Report
├── Header (target URL, source path, scan date, severity count)
│
├── Executive Summary Table        ← anchor links xuống finding cards bên dưới
│   ├── [CRITICAL] VULN-001 — SQL Injection      → #vuln-001
│   ├── [HIGH]     VULN-002 — SSRF               → #vuln-002
│   └── [MEDIUM]   VULN-003 — Open Redirect      → #vuln-003
│
├── Finding Card: #vuln-001
│   ├── Vulnerability Path (data flow diagram)
│   ├── Code Evidence (file:line với syntax highlight + vulnerable line highlight)
│   ├── Working POC Request (raw HTTP block)
│   ├── Reproduction Command (cURL — copy/paste ready, đầy đủ headers)
│   ├── Response Evidence (response thực tế, truncate nếu quá dài)
│   └── Remediation Suggestion
├── Finding Card: #vuln-002
└── ...
```

### Report Design (Premium HTML)
- Dark theme, syntax highlighted code blocks
- Severity badges (CRITICAL=red, HIGH=orange, MEDIUM=yellow)
- Collapsible sections per finding card
- Copy-to-clipboard buttons cho mọi code block
- Executive Summary table với anchor links đến từng finding
- Timestamp + target metadata ở header

---

## 🔄 Orchestrator State Machine

```
START
  │
  ▼
[Parse CLI Input]
  │ -f, -url, -h, [-h2 for IDOR]
  ▼
[Phase 0: Recon Agent]
  │ tech_stack, routes, auth_map, sinks[]
  │ capability_surfaces[], context_exposures[]
  ▼
[Phase 1: Scanner Agent]
  │ findings_draft[]
  │ (if finding.sanitizers_found OR context_exposures[] not empty)
  │   → spawn Policy & Sandbox Bypass Agent (Proactive)
  ▼
[Phase 2: Verifier Agent — loop per finding]
  │
  ├── CONFIRMED → add to confirmed_findings[]
  ├── FAIL → RCA Debate Loop (max 3 rounds):
  │     ├── Structural Error → Scanner issues RCA_Response
  │     └── PAYLOAD_FILTERED → Policy & Sandbox Bypass Agent issues RCA_Response
  │         Verifier retries với revised_poc_template
  ├── DISCARD → drop silently
  └── AUTH_LIMITED → note internally, skip from report
  │
  ▼ (all findings processed)
[Reporter Agent]
  │ confirmed_findings[] → report.html (với Executive Summary table)
  ▼
[OUTPUT: AntigraHunter_report_YYYYMMDD_HHMMSS.html]
  │
END
```

---

## 🗂️ Knowledge Base

AntigraHunter sử dụng **[SKILL.md](file:///e:/Antigravity/AntigraHunter/SKILL.md)** làm knowledge base chính:
- §3: Sink Matrix (dangerous functions per language)
- §4: Audit Logic (taint analysis workflow)
- §5: Verification & PoC Standard
- §6: Optimization Rules (files/dirs to skip)

---

## 🔒 Safety & Ethics

- Chỉ dùng trên hệ thống được cấp phép kiểm thử hợp pháp.
- POC payloads chỉ dùng để **chứng minh** lỗi: `id`, `whoami`, `sleep 5`, `alert(1)`, `{{7*7}}`.
- Không xóa data, không escalate sau khi confirm.
- Tất cả HTTP request được log để có audit trail.

---

## 📁 Project File Structure

```
e:\Antigravity\AntigraHunter\
├── SKILL.md                    # ✅ Vulnerability knowledge base (có sẵn)
├── AI_INSTRUCTION.md           # ✅ File này — system design
├── AGENTS.md                   # 🔜 Định nghĩa từng agent (prompt + tools)
├── reports/                    # 🔜 Output HTML reports
│   └── report_YYYYMMDD_HHMMSS.html
└── logs/                       # 🔜 Audit trail (HTTP requests log)
    └── session_YYYYMMDD.log
```

---

## ❓ Open Questions

> [!IMPORTANT]
> **Q3: Scope giới hạn** *(vẫn còn open)*
> Có những loại lỗ hổng nào mày muốn BỎ QUA hoặc ƯU TIÊN hơn không? (VD: chỉ focus vào OWASP Top 10, hay bao gồm cả business logic flaws?)

---

*Các câu hỏi đã được giải quyết:*

| Question | Kết quả |
|---|---|
| **Q1** `-h` scope | Áp dụng cho **tất cả** requests. Ngoại lệ: AuthBypass không gửi auth; IDOR dùng `-h2` cho user thứ 2. |
| **Q2** IDOR multi-user | Đã thêm flag `-h2 <header>` vào input format. IDOR tự động **skip** nếu không có `-h2`. |
| **Q4** Rate limiting | Cố định **500ms delay** (`sleep 0.5`) sau mỗi HTTP request. |
