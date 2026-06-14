# 🐛 AntigraHunter — AI-Powered Security Auditing Framework

AntigraHunter is an AI-driven automated security testing framework. Instead of merely reporting theoretical vulnerabilities, AntigraHunter **only reports vulnerabilities that have been confirmed via actual HTTP requests**. No working POC = No report.

---

## Installation (Loading into Antigravity)

AntigraHunter is a "Skill" designed for your AI Assistant (Antigravity). For the AI to understand this workflow, you need to load the directory containing these markdown files into Antigravity's workspace.

1. **Clone/Download** the entire `AntigraHunter` directory to your local machine.
2. **Open Antigravity** and select **Add Workspace** (or the equivalent command to load a directory).
3. **Point the path** to the downloaded `AntigraHunter` directory.
4. The AI will automatically read the rules and system definitions contained in this directory.

#### *Only support Antigravity 2.0 version 2.0.6,2.0.10*
#### *Good news: testing confirms excellent compatibility with Hermes Agent on DeepSeek-v4-pro.*
---

## Usage

You can trigger AntigraHunter by asking the assistant using the following command format:

```
bugfinding -f <source_path> -url <target_url> [-h <auth_header>] [-h2 <second_user_header>]
```

### Flags

| Flag | Required | Description |
|---|---|---|
| `-f <path>` | ✅ | Path to the source code directory to be scanned |
| `-url <url>` | ✅ | The URL of the running application target |
| `-h <header>` | Optional | Auth header (e.g., `"Cookie: session=abc123"` or `"Authorization: Bearer token"`) |
| `-h2 <header>` | Optional | Auth header of a second user, used for testing IDOR |

### Examples

```bash
# Basic scan
bugfinding -f ./webapp -url http://localhost:3000

# Scan with an authentication header
bugfinding -f ./webapp -url https://target.com -h "Cookie: sessionid=abc123"

# Test for IDOR using 2 separate accounts
bugfinding -f ./webapp -url https://target.com \
  -h "Cookie: sessionid=user1token" \
  -h2 "Cookie: sessionid=user2token"
```

---

## Pipeline Overview

AntigraHunter runs through 4 sequential phases:

```
Phase 0: Recon     — Map the attack surface, routes, sinks, and auth mechanisms
Phase 1: Scan      — Perform inter-procedural taint analysis
Phase 2: Verify    — Send actual HTTP requests to confirm POCs
Phase 3: Report    — Generate a comprehensive HTML report
```

---

## The 6 Pipeline Agents

| Agent | Role |
|---|---|
| **Orchestrator** | Coordinates the entire pipeline, manages the state machine, and enforces the Iron Rule. |
| **Recon Agent** | Maps the attack surface: tech stack, routes, auth, sinks, and capability exposures. |
| **Scanner Agent** | Performs inter-procedural taint analysis, tracing data flow from HTTP source to sink. |
| **Verifier Agent** | Sends actual HTTP requests, analyzes responses, and confirms the POC. |
| **Policy & Sandbox Bypass Agent** | Analyzes sandboxes/WAFs, generates bypass payloads, and detects capability exposures. |
| **Reporter Agent** | Generates a premium HTML report with an Executive Summary and copy-pasteable cURL commands. |

---

## Key Features

### ☑ The Iron Rule
> **No confirmed POC = No report.**

AntigraHunter will never include a vulnerability in the final report that hasn't been verified by an actual HTTP request. Every single finding must pass through the Verifier Agent.

### ☑ RCA Debate Loop
When a POC fails, instead of giving up, AntigraHunter initiates a diagnostic loop:

```
Verifier FAIL → Orchestrator categorizes the root cause:
  - Structural error (wrong param, wrong path...) → Scanner re-reads the code and fixes the POC.
  - Payload filtered/WAF → Policy Agent generates bypass payloads.
→ Verifier retries (maximum 3 rounds).
→ If it still fails: Classified as DISCARD or AUTH_LIMITED.
```

### ☑ IDOR Testing with 2 Accounts
Use the `-h2` flag to provide a second user's auth header. AntigraHunter automatically:
1. Uses `-h` to create/fetch a resource belonging to User 1.
2. Uses `-h2` to attempt accessing that same resource as User 2.
3. Confirms IDOR if the access is successful.

---

## Supported Languages & Frameworks

| Language | Frameworks |
|---|---|
| **Python** | Django, Flask, FastAPI |
| **JavaScript / TypeScript** | Express, NestJS |
| **Java** | Spring Boot |
| **PHP** | Laravel, CodeIgniter |
| **Ruby** | Rails |
| **Go** | Gin, Echo |
| **C#** | ASP.NET Core |
| **Rust** | Axum, Actix-web, Warp |
| **C/C++** | Custom HTTP servers |

**Vulnerability Classes Covered:** SQLi, RCE, SSRF, XSS, SSTI, Path Traversal, Open Redirect, IDOR, Auth Bypass, JWT Confusion, XXE, Deserialization, ReDoS, Prototype Pollution, Mass Assignment, Capability Abuse.

---

## File Structure

```
AntigraHunter/
├── README.md              — This file
├── BUGFINDING.md          — The main execution pipeline (entry point)
├── ANTIGRAHUNTER_RUNNER.md        — Auto-loader: triggers when you type "bugfinding ..."
├── AGENTS.md              — Detailed system prompts for each agent
├── AI_INSTRUCTION.md      — System architecture overview
└── SKILL.md               — Knowledge base: Sink Matrix, Taint Analysis, Sanitizer Bypass
```

## Output

Reports are saved to: `<parent_of_source>/reports/AntigraHunter_report_YYYYMMDD_HHMMSS.html`


