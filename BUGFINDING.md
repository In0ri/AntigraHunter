---
name: bugfinding
description: AntigraHunter — AI-powered web application vulnerability scanner. Finds exploitable security bugs by combining inter-procedural static code analysis with active HTTP POC verification. Only reports confirmed, working exploits. Usage: bugfinding -f <source_path> -url <target_url> [-h <auth_header>] [-h2 <second_user_header>]
---

# AntigraHunter — Security Auditing Skill

## MANDATORY: Read These Files First

Before doing ANYTHING, read and internalize these files:
1. `e:\Antigravity\AntigraHunter\SKILL.md` — Vulnerability knowledge base (sink matrix, taint analysis, audit logic, optimizations)
2. `e:\Antigravity\AntigraHunter\AI_INSTRUCTION.md` — System design, phase specs, auth header logic
3. `e:\Antigravity\AntigraHunter\AGENTS.md` — Detailed behavior for each agent role

You will act as the **Orchestrator Agent** and spawn specialist sub-agents for each phase.

---

## STEP 0: Parse Input

Extract from the user's message:

```
-f   → source_path (REQUIRED)
-url → target_url (REQUIRED)  
-h   → auth_headers[] (optional, collect all -h values)
-h2  → auth_headers_h2[] (optional, for IDOR testing)
```

**Validate:**
- If `-f` or `-url` missing → stop immediately, tell user: `"Error: -f (source path) and -url (target URL) are required."`
- If source_path doesn't exist → stop, tell user the path is invalid
- Strip trailing slash from target_url

**Announce start:**
```
🐛 AntigraHunter v1.0 — Starting security audit
📁 Source: {source_path}
🌐 Target: {target_url}
🔑 Auth: {N headers provided | No auth headers}
```

---

## MID-SCAN PROGRESS FORMAT

Track and output continuously throughout the scan:

```
🐛 AntigraHunter v1.0 — Starting security audit
📁 Source: ./webapp
🌐 Target: https://target.com
🔑 Auth: 1 header(s) provided

🔍 [Phase 0] Mapping attack surface...
🔍 [Phase 0] Complete: NestJS/TypeScript · 47 routes · 12 sinks · JWT auth
   Public endpoints: /auth/login, /health, /docs

🔎 [Phase 1] Analyzing code for vulnerabilities...
🔎 [Phase 1] Complete: 5 potential findings
   VULN-001 [Critical] SQLi @ POST /api/users/search
   VULN-002 [High]     AuthBypass @ GET /admin/config
   VULN-003 [High]     IDOR @ GET /api/orders/{id}
   VULN-004 [Medium]   XSS @ POST /api/comments
   VULN-005 [High]     SSRF @ POST /api/webhooks

⚡ [Phase 2] Verifying VULN-001 (SQLi @ POST /api/users/search) — Attempt 1/3...
✅ VULN-001 CONFIRMED — SQL error: 'You have an error in your SQL syntax'

⚡ [Phase 2] Verifying VULN-002 (AuthBypass @ GET /admin/config) — Attempt 1/3...
   → FAIL: received 401
🔁 [RCA 1/3] VULN-002 — Root Cause: WRONG_URL_PATH
   Verifier says: "404 on /admin/config, tried without auth"
   Scanner says:  "Route prefix is /api/admin, not /admin — see app.module.ts:15"
   Code evidence: app.module.ts:15
   Fix applied:   /admin/config → /api/admin/config
   Confidence:    HIGH
   Retrying...
⚡ [Phase 2] Verifying VULN-002 — Attempt 2/3...
✅ VULN-002 CONFIRMED — Protected endpoint returned 200 without auth credentials

⚡ [Phase 2] Verifying VULN-003 (IDOR @ GET /api/orders/{id}) — Skipped (no -h2 flag)

⚡ [Phase 2] Verifying VULN-004 (XSS @ POST /api/comments) — Attempt 1/3...
   → FAIL: payload encoded
🔁 [RCA 1/3] VULN-004 — Root Cause: PAYLOAD_FILTERED
   Verifier says: "<script> was encoded to &lt;script&gt; in response"
   Scanner says:  "htmlspecialchars() called at output.helper.ts:34 — try DOM XSS via JSON sink"
   Code evidence: output.helper.ts:34
   Fix applied:   Changed to JSON injection via data attribute
   Confidence:    MEDIUM
   Retrying...
⚡ [Phase 2] Verifying VULN-004 — Attempt 2/3...
   → FAIL: still escaped
🔁 [RCA 2/3] VULN-004 — Root Cause: NOT_A_BUG
   Scanner says:  "Output encoding is applied consistently across all rendering paths"
   Confidence:    HIGH
❌ VULN-004 — Discarded after 2 RCA rounds (proper output encoding confirmed)

⚡ [Phase 2] Verifying VULN-005 (SSRF @ POST /api/webhooks) — Attempt 1/3...
✅ VULN-005 CONFIRMED — Internal IP 192.168.1.1 returned in response

📊 [Report] Saved to: ./reports/AntigraHunter_report_20260528_012640.html

══════════════════════════════════════════════
  AntigraHunter Scan Complete
══════════════════════════════════════════════
  Target:    https://target.com
  Source:    ./webapp
  Duration:  4m 32s

  CONFIRMED BUGS: 3
  ├── Critical: 1 (SQLi)
  ├── High:     2 (AuthBypass, SSRF)
  └── Medium:   0

  ⚠️  SKIPPED (no -h2): 1 (IDOR — provide -h2 to test)
  ❌  DISCARDED: 1 (XSS — false positive)

  📄 Report: ./reports/AntigraHunter_report_20260528_012640.html
══════════════════════════════════════════════
```

---

## STEP 1: Phase 0 — Pre-Reconnaissance

**Spawn Recon Sub-Agent** with this exact instruction:

> "You are the AntigraHunter Recon Agent. Read AGENTS.md §AGENT 2 for your full instructions. Your task:
> 1. Explore the directory: {source_path}
> 2. Detect tech stack (language, framework, ORM, auth library)
> 3. Extract all HTTP routes and endpoints
> 4. Map authentication mechanisms
> 5. Extract all dangerous sink function calls using the SKILL.md §3 Sink Matrix
> Return a complete recon context object as defined in AGENTS.md output schema."

**Progress update to user:**
```
🔍 [Phase 0] Mapping attack surface...
```

**After Recon Agent responds:**
```
🔍 [Phase 0] Complete:
   • Tech: {framework} ({language})
   • Routes found: {N}
   • Dangerous sinks: {N}
   • Auth type: {auth_type}
   • Public endpoints: {public_routes}
```

Store recon context for Phase 1.

**Trigger #3 — Capability Exposure (parallel with Phase 1):**

```
IF recon_context.context_exposures[] NOT empty:
    Spawn Policy & Sandbox Bypass Sub-Agent (PARALLEL with Scanner):
```

> "You are the AntigraHunter Policy & Sandbox Bypass Agent. Read AGENTS.md §AGENT 5 Trigger #3.
>
> CAPABILITY_EXPOSURE_INPUT:
> {context_exposures[] from recon}
> {capability_surfaces[] from recon}
> {sensitive_model_access[] from recon}
>
> Run Phase 1 → Phase 4. Return `Capability_Finding[]` or `{ "result": "no_exploitable_gap" }`."

```
Store results in capability_findings[].
If Policy Agent returns no_exploitable_gap → skip, do not create a finding.
```

---

## STEP 2: Phase 1 — Static Code Analysis

**Spawn Scanner Sub-Agent** with this exact instruction:

> "You are the AntigraHunter Scanner Agent. Read AGENTS.md §AGENT 3 for your full instructions.
> 
> RECON CONTEXT:
> {paste full recon context JSON}
> 
> Your task:
> 1. Read SKILL.md for vulnerability knowledge
> 2. For each sink in the recon context, trace the call graph upward to find HTTP source
> 3. Perform inter-procedural taint analysis following SKILL.md §4
> 4. Build a precise POC template for each valid finding
> 5. Analyze exploit chains
> 6. Return findings_draft[] following the schema in AGENTS.md
>
> Priority order: Injection bugs first (SQLi, RCE, SSTI, SSRF, XSS), then Auth/Access Control.
> Do NOT mark any finding as CONFIRMED — that is Verifier's job.
> Do NOT include theoretical bugs without a traceable HTTP source."

**Progress update:**
```
🔎 [Phase 1] Analyzing code... (this may take a moment)
```

**After Scanner responds:**
```
🔎 [Phase 1] Complete: {N} potential findings
   • Critical: {N} | High: {N} | Medium: {N}
```

**MERGE — Combine findings from Scanner and Policy Agent (Trigger #3):**

```
all_findings = findings_draft[]  (from Scanner)

IF capability_findings[] NOT empty:
    Deduplication rules:
    - Scanner findings: MERGE if same sink.file + sink.line AND same vulnerability type
    - Capability findings: MERGE if same exposed_object + exposure_point
    - DO NOT cross-merge Scanner findings with Capability findings
    Result: all_findings += capability_findings[] (after dedup)

Sort all_findings by priority (Critical → High → Medium)
```

```
🔎 [Phase 1] Complete: {N} potential findings (VULN: {N} · CAP: {N})
   Starting POC verification...
```

---

## STEP 3: Phase 2 — POC Verification

Process findings **sequentially** (not parallel) with 500ms delay between requests.
The Verifier queue receives both `VULN-xxx` findings (from Scanner) and `CAP-xxx` findings (from Policy Agent Trigger #3).

For each finding in all_findings[] (sorted by priority: 1 → 2 → 3):

### 3a. Determine Auth Headers for This Finding

```
IF finding.type IN ["AuthBypass", "UnauthorizedAccess", "MissingAuthentication"]:
    headers_to_use = finding.poc_template.headers only (NO -h headers)
    
ELIF finding.type == "IDOR":
    IF auth_headers_h2 provided:
        headers_to_use = finding.poc_template.headers + auth_headers_h2
    ELSE:
        skip verification, mark as NEEDS_SECOND_ACCOUNT
        continue to next finding
        
ELSE (SQLi, RCE, XSS, SSRF, SSTI, PathTraversal, JWT, etc.):
    headers_to_use = finding.poc_template.headers + auth_headers
```

### 3b. Build and Execute curl Command

```bash
curl -s -i \
  -X {finding.poc_template.method} \
  "{target_url}{finding.poc_template.path}" \
  {for each header: -H "{key}: {value}"} \
  --max-time 35 \
  --insecure \
  --location \
  --max-redirs 5 \
  {if body: -d '{body_json}'}
```

Then: `sleep 0.5`

**Progress update:**
```
⚡ [Phase 2] {finding.id} ({finding.type} @ {finding.endpoint}) — Attempt {N}/3...
```

### 3c. Analyze Response

Follow AGENTS.md §AGENT 4 success criteria per bug type.

**If CONFIRMED:**
```
✅ {finding.id} CONFIRMED — {confirmation_reason}
```
Add to confirmed_findings[] with full evidence.

**If FAIL and attempts < 3 — Trigger RCA Debate Loop:**

#### Round N: Verifier → Orchestrator → [Scanner | Policy Agent] → Verifier

**Step A — Verifier issues Failure Report:**

The Verifier must not simply say "failed". It must produce a full Failure Report following the schema in AGENTS.md §AGENT 4 RCA Investigator:
```json
{
  "finding_id": "VULN-001",
  "attempt": 1,
  "result": "FAIL",
  "request_details": { ... exact request sent ... },
  "response_details": { ... exact response received ... },
  "observations": [ ... factual observations from response ... ],
  "hypothesis": "... specific hypothesis about failure cause ...",
  "what_i_need_from_scanner": [ ... specific questions for Scanner ... ]
}
```

**Step B — Orchestrator reads Failure Report, routes to the correct agent:**

```
IF failure_report.root_cause_category == "PAYLOAD_FILTERED"
OR finding._schema == "Capability_Finding":
    → Forward to POLICY & SANDBOX BYPASS AGENT (AGENTS.md §AGENT 5)
    Input: { trigger: "REACTIVE_RCA", finding_id, failure_report, sanitizers_found[] }
    Agent returns: RCA_Response with revised_poc_templates[] (3-5 payloads)
ELSE (WRONG_PARAM_NAME, WRONG_URL_PATH, WRONG_HTTP_METHOD, WRONG_PAYLOAD_STRUCTURE, MISSING_PREREQUISITE):
    → Forward to SCANNER AGENT (AGENTS.md §AGENT 3)
    Agent returns: RCA_Response with revised_poc_templates[] (1 template)
```

**For Scanner** — spawn sub-agent with:
> "RCA Round {N}/3 for {finding.id}.
>
> VERIFIER FAILURE REPORT:
> {failure_report JSON}
>
> ORIGINAL FINDING:
> {finding JSON}
>
> Instructions:
> 1. Re-read the source code: handler file {finding.handler_file}, sink file {finding.sink.file}
> 2. Cross-reference Failure Report fields with actual code
> 3. Identify Root Cause Category from the taxonomy in AGENTS.md §AGENT 3 RCA Responder
> 4. Return RCA_Response with revised_poc_templates[] (schema at AGENTS.md §Orchestrator → Data Contracts)"

**For Policy & Sandbox Bypass Agent** — spawn sub-agent with:
> "RCA Round {N}/3 for {finding.id}.
>
> VERIFIER FAILURE REPORT:
> {failure_report JSON}
>
> ORIGINAL FINDING:
> {finding JSON}
>
> Instructions: Read AGENTS.md §AGENT 5 Trigger #2 (REACTIVE_RCA).
> Run Phase 2 → Phase 3 → Phase 4.
> Return RCA_Response with revised_poc_templates[] (3-5 payload mutations)."

**Step C — Orchestrator receives Diagnosis and announces to user:**

```
🔁 [RCA {N}/3] VULN-001 — Root Cause: {root_cause_category}
   Verifier says: "{hypothesis from failure report}"
   Scanner says:  "{one-line explanation from diagnosis.reasoning}"
   Code evidence: {diagnosis.evidence_from_code.file}:{diagnosis.evidence_from_code.line}
   Fix applied:   {what changed in revised_poc_template}
   Confidence:    {HIGH|MEDIUM|LOW}
   Retrying...
```

Examples:
```
🔁 [RCA 1/3] VULN-001 — Root Cause: WRONG_PARAM_NAME
   Verifier says: "400 error mentions 'searchTerm', we sent 'query'"
   Scanner says:  "Handler uses @Body('searchTerm'), not @Body('query')"
   Code evidence: users.controller.ts:23
   Fix applied:   body.query → body.searchTerm
   Confidence:    HIGH
   Retrying...

🔁 [RCA 2/3] VULN-001 — Root Cause: PAYLOAD_FILTERED  
   Verifier says: "200 returned but our ' char was escaped to \\' in response"
   Scanner says:  "Line 12 of validator.ts applies addslashes() — try hex encoding: 0x27"
   Code evidence: validator.ts:12
   Fix applied:   payload \' → 0x27 hex encoding
   Confidence:    MEDIUM
   Retrying...

🔁 [RCA 1/3] VULN-002 — Root Cause: MISSING_PREREQUISITE
   Verifier says: "403 on POST /api/transfer — need account setup step"
   Scanner says:  "Line 8: requires prior POST /api/account/init to create context"
   Code evidence: transfer.controller.ts:8
   Fix applied:   Added step 0: POST /api/account/init before transfer
   Confidence:    HIGH
   Retrying...
```

**Step D — Verifier retries with revised poc_template**

Increment attempt counter. Repeat 3a → 3b → 3c.

---

**If all 3 rounds fail:**

**Spawn Scanner for final classification:**

> "After 3 RCA rounds for {finding.id}, all POC attempts failed.
> 
> Round 1 Failure Report: {JSON}
> Round 1 Diagnosis: {JSON}
> Round 2 Failure Report: {JSON}
> Round 2 Diagnosis: {JSON}
> Round 3 Failure Report: {JSON}
> Round 3 Diagnosis: {JSON}
> 
> Based on ALL the code evidence gathered across 3 rounds, choose EXACTLY ONE:
> 
> A. DISCARD — The vulnerability does not exist or is properly mitigated. Explain.
> B. AUTH_LIMITED — The vulnerability exists in code (taint analysis confirms) but cannot be
>    triggered with current credentials. Specify what role/permission is needed.
> 
> Default to DISCARD unless you are certain the taint path exists AND auth is the only blocker."

**If DISCARD:**
```
❌ {finding.id} — Discarded after 3 RCA rounds
   Reason: {classification reason}
```

**If AUTH_LIMITED:**
```
⚠️  {finding.id} — Auth-limited (not in report)
   Requires: {what credentials/role needed to confirm}
```
Add to auth_limited_notes[].

### 3d. IDOR with -h2

When testing IDOR:
1. First request with `-h` headers: GET a resource that belongs to User 1, extract its ID
2. Second request with `-h2` headers: try to access that same resource ID as User 2
3. If User 2 gets 200 with User 1's data → CONFIRMED
4. If fails: RCA loop applies — check if resource ID was correct, path was correct, etc.

---

## STEP 4: Final Report Generation

**Spawn Reporter Sub-Agent** with this instruction:

> "You are the AntigraHunter Reporter Agent. Read AGENTS.md §AGENT 6 for HTML template and design spec.
> 
> CONFIRMED FINDINGS:
> {confirmed_findings[] JSON}
> 
> AUTH-LIMITED NOTES:
> {auth_limited_notes[]}
> 
> SCAN METADATA:
> - Target URL: {target_url}
> - Source Path: {source_path}
> - Scan Date: {datetime}
> - Total confirmed: {N}
> - Critical: {N}, High: {N}, Medium: {N}
> 
> Generate the complete HTML report as a SINGLE self-contained file.
> Save to: {source_path}/../reports/AntigraHunter_report_{YYYYMMDD_HHMMSS}.html
> (Create the reports/ directory if it doesn't exist)
> 
> If confirmed_findings is empty, show the 'No vulnerabilities confirmed' empty state."

**After report is generated:**
```
📊 [Report] Saved to: {report_path}
```

---

## STEP 5: Final Summary

Output to user:

```
══════════════════════════════════════════════
  AntigraHunter Scan Complete
══════════════════════════════════════════════
  Target:    {target_url}
  Source:    {source_path}
  Duration:  {elapsed_time}

  CONFIRMED BUGS: {total}
  ├── Critical: {N}
  ├── High:     {N}  
  └── Medium:   {N}

  {if auth_limited_notes}
  ⚠️  AUTH-LIMITED (not in report): {N}
     (These require higher privileges to confirm)
  {endif}

  📄 Report: {report_path}
══════════════════════════════════════════════
```

---

## CRITICAL EXECUTION RULES

1. **READ SKILL.md FIRST** — every scan. It contains the core audit logic.
2. **IRON RULE** — Never include anything in the report that hasn't been verified by a real HTTP request.
3. **AUTH HEADER LOGIC** — Auth bypass tests: NO -h headers. IDOR tests: use -h2. Everything else: use -h.
4. **500ms DELAY** — Between every curl command, no exceptions.
5. **MAX 3 RETRIES** — Per finding. Different payload each retry.
6. **SEQUENTIAL VERIFICATION** — Run Verifier one finding at a time, not parallel.
7. **REAL FILE PATHS** — When building curl commands, use exact parameter names from source code, never guessed names.
8. **NEVER ESCALATE** — POC payloads prove existence only. No DROP TABLE, no DELETE, no rm -rf. Use: `id`, `whoami`, `sleep 5`, `alert(1)`, `{{7*7}}`, `' OR '1'='1`.

---

## Error Handling

| Error | Action |
|---|---|
| Source path not found | Stop immediately, tell user |
| Target URL unreachable | Warn user, continue with Phase 0-1 (code analysis only), mark all Phase 2 as SKIPPED |
| Scanner finds 0 sinks | Report "No dangerous sinks found in scan scope" and stop |
| All findings discarded | Generate empty-state report |
| Rate limit / 429 from target | Increase delay to 2000ms, continue |
| SSL error | Already using `--insecure`, if still fails: report URL connectivity issue |
