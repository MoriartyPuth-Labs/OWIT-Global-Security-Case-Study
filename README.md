<div align="center">

# 🔐 OWIT Global — Vulnerability Assessment Penetration Testing
### A Black-Box Security Case Study

</div>

> A sanitized, real-world external vulnerability assessment of a pre-production
> insurance SaaS platform (multi-tenant, Spring Boot + .NET + Keycloak, with an
> embedded AI/LLM agent). Client identity and live hostnames are **redacted** for
> confidentiality. This repo documents the full methodology, findings, evidence,
> risk analysis, and — importantly — the **judgment calls** made during testing.

![Type](https://img.shields.io/badge/Assessment-External%20VA-blue)
![Approach](https://img.shields.io/badge/Approach-Black--Box%20Unauthenticated-lightgrey)
![Findings](https://img.shields.io/badge/Findings-1%20High%20%7C%203%20Med%20%7C%206%20Low%20%7C%203%20Info-orange)
![Standards](https://img.shields.io/badge/Aligned-OWASP%20WSTG%20%7C%20API%20Top%2010%20%7C%20CVSS%20v3.1-green)
![Ethics](https://img.shields.io/badge/Exploitation-Validated%2C%20Not%20Exploited-red)

---

## 📑 Table of Contents
1. [Executive Summary](#1--executive-summary)
2. [Engagement Details](#2--engagement-details)
3. [Scope](#3--scope)
4. [Threat Model & Assumptions](#4--threat-model--assumptions)
5. [Methodology](#5--methodology)
6. [Risk Rating Methodology](#6--risk-rating-methodology)
7. [Findings Summary](#7--findings-summary)
8. [Detailed Findings](#8--detailed-findings)
9. [Positive Observations](#9--positive-observations)
10. [The Ethical Decision: Where I Stopped](#10--the-ethical-decision-where-i-stopped)
11. [Tooling](#11--tooling)
12. [Skills & Takeaways](#12--skills--takeaways)
13. [Recommended Next Steps](#13--recommended-next-steps)

---

## 1. 📌 Executive Summary

This engagement was an **external, black-box, unauthenticated** vulnerability
assessment of a pre-production insurance SaaS platform consisting of two single-page
applications, a Spring Boot core API, and a Keycloak identity provider — all hosted on
AWS behind a shared Kubernetes ingress.

The platform demonstrated **strong baseline hygiene**: modern TLS only, HSTS with
preload, suppressed server banners, restricted HTTP methods, disabled high-risk
Actuator endpoints, no subdomain-takeover exposure, and correct `401` enforcement on
the overwhelming majority of endpoints.

That hygiene was undermined by a **single class of high-impact defect**: several
document-service API endpoints were deployed **outside the authentication filter**,
creating an unauthenticated read/write surface on a system that processes customer
documents. This was compounded by two unauthenticated **information-disclosure** issues
— a full application config endpoint and a complete 347-endpoint OpenAPI specification
— which together hand an attacker a precise map of the backend.

> **Headline takeaway:** Good perimeter engineering can be undone by one unlocked door.
> Security is judged by the weakest authentication boundary, not the average one.

**Findings:** 1 High · 3 Medium · 6 Low · 3 Informational.

---

## 2. 🗂️ Engagement Details

| Attribute | Detail |
|---|---|
| **Engagement type** | External, black-box, unauthenticated Vulnerability Assessment |
| **Target** | Pre-production insurance SaaS platform *(redacted)* |
| **Authorization** | Written authorization / Rules of Engagement (ROE) on file |
| **Test window** | 3-day window *(dates redacted)* |
| **Constraints** | No DoS · No destructive actions · **Confirmed issues not exploited** |
| **Tech stack** | AngularJS 1.8.3 + Angular SPAs · Spring Boot · .NET · Keycloak · AWS EKS |
| **Standards** | OWASP WSTG · OWASP Top 10 (2021) · OWASP API Security Top 10 (2023) · CVSS v3.1 |

---

## 3. 🎯 Scope

| Asset | Role | Status |
|---|---|---|
| `https://app-core.<redacted>/` | Core platform UI (AngularJS SPA) | In scope |
| `https://admin-portal.<redacted>/` | Admin portal (Angular SPA) | In scope |
| `https://api-core.<redacted>/` | Core REST API (Spring Boot) | In scope by dependency |
| `https://login.<redacted>/` | Keycloak identity provider | In scope by dependency |

**Discovered, out of scope:** the unauthenticated config endpoint (F-002) exposed
**~90 additional internal and per-tenant subdomains**. These were documented as an
attack-surface disclosure but **not tested**, as they fell outside the authorized scope.

---

## 4. 🧭 Threat Model & Assumptions

- **Attacker profile:** an unauthenticated, remote actor on the public internet with
  no credentials and no prior knowledge of the application.
- **Trust boundary tested:** everything reachable **before** login.
- **Out of model (acknowledged gaps):** authenticated business logic, IDOR/BOLA across
  tenants, injection on authenticated parameters, and the AI/LLM agent internals — none
  reachable black-box, all flagged for an authenticated follow-up.
- **Assumption:** pre-production mirrors production architecture and is to be treated
  with production-level care (per client instruction).

---

## 5. 🔬 Methodology

Aligned to **OWASP WSTG**, executed in four phases at a deliberately low request rate.

### Phase 1 — Reconnaissance & Fingerprinting
- DNS resolution, CNAME / takeover checks, hosting/infra identification
- TLS protocol & cipher inspection, certificate review
- Technology and framework fingerprinting (SPA frameworks, backend, IdP)
- Client-side config harvesting and endpoint enumeration from JS bundles

### Phase 2 — Configuration & Exposure Review
- HTTP security header analysis (HSTS, CSP, XFO, Referrer-Policy, Permissions-Policy)
- CORS policy evaluation (origin reflection, credentials, preflight handling)
- HTTP method testing (TRACE/PUT/DELETE/PATCH/OPTIONS)
- Management-interface exposure (Spring Boot Actuator, Swagger/OpenAPI)
- Sensitive-file & error-handling checks; information-disclosure testing

### Phase 3 — Access Control & Authentication Review *(unauthenticated surface)*
- Authentication enforcement validation per endpoint (expected `401/403`)
- **Authentication-filter bypass detection** via sibling status-code comparison
- Identity provider surface review (admin console, realms, registration endpoints)
- Email security posture (SPF/DKIM/DMARC)

### Phase 4 — Validation & Risk Rating
- Non-destructive confirmation of each finding
- CVSS v3.1 scoring with environment-aware severity
- Clear separation of **confirmed** vs **potential** impact

> **Guiding principle:** *Confirm the vulnerability, not the breach.* Where confirming
> impact would require attacking real data, testing stopped and escalation was
> recommended instead.

---

## 6. 📐 Risk Rating Methodology

Findings were scored with **CVSS v3.1** and mapped to qualitative bands:

| Band | CVSS Range | Meaning |
|---|---|---|
| 🔴 Critical | 9.0 – 10.0 | Confirmed severe impact (e.g., unauth RCE / mass data exposure) |
| 🔴 High | 7.0 – 8.9 | Serious weakness; high likelihood of significant impact |
| 🟠 Medium | 4.0 – 6.9 | Meaningful weakness; impact gated by conditions |
| 🟡 Low | 0.1 – 3.9 | Limited impact; defence-in-depth / hardening |
| ⚪ Info | N/A | No direct impact; advisory / future-phase note |

> Severity reflects **evidence**, not suspicion. Where the unauthenticated *access* was
> confirmed but the *impact* was not validated (by design), findings were rated on
> confirmed access with the potential ceiling clearly documented.

---

## 7. 📊 Findings Summary

| ID | Severity | CVSS | Finding | Class |
|----|----------|------|---------|-------|
| [F-001](#f-001--unauthenticated-authentication-filter-bypass) | 🔴 High | 8.1* | Unauthenticated auth-filter bypass (upload + view + callback) | CWE-306/434/639 |
| [F-002](#f-002--unauthenticated-application-configuration-disclosure) | 🟠 Medium | 5.3 | Unauthenticated application config & architecture disclosure | CWE-200/538 |
| [F-003](#f-003--unauthenticated-openapi--swagger-exposure) | 🟠 Medium | 5.3 | Unauthenticated OpenAPI/Swagger exposure (347 endpoints) | CWE-200 |
| [F-004](#f-004--verbose-net-exception--inconsistent-authentication) | 🟠 Medium | 5.3 | Verbose .NET exception leak / inconsistent auth | CWE-209/755 |
| [F-005](#f-005--overly-permissive-cors) | 🟡 Low | 4.3 | Overly permissive CORS (`ACAO: *`) | CWE-942 |
| [F-006](#f-006--weak--deprecated-security-headers) | 🟡 Low | 3.7 | Weak / deprecated security headers | CWE-693/1021 |
| [F-007](#f-007--end-of-life-client-side-components) | 🟡 Low | 4.8 | End-of-life client components (AngularJS 1.8.3) | CWE-1104/1035 |
| [F-008](#f-008--spring-boot-actuator-exposed-limited) | 🟡 Low | 3.1 | Spring Boot Actuator exposed (limited) | CWE-16/200 |
| [F-009](#f-009--identity-provider-admin-console--privileged-realm-exposed) | 🟡 Low | 4.3 | IdP admin console & privileged realm exposed | CWE-284/419 |
| [F-010](#f-010--weak-email-anti-spoofing) | 🟡 Low | 4.3 | Weak email anti-spoofing (DMARC `p=none`) | CWE-358 |
| [F-011](#f-011--missing-content-security-policy-admin-portal) | ⚪ Info | – | Missing Content-Security-Policy on admin portal | CWE-693 |
| [F-012](#f-012--third-party-scripts-without-subresource-integrity) | ⚪ Info | – | Third-party scripts without SRI | CWE-829 |
| [F-013](#f-013--aillm-agent-subsystem--attack-surface-note) | ⚪ Info | – | AI/LLM chat-agent subsystem (attack-surface note) | LLM Top 10 |

<sub>*F-001 rated High on confirmed unauthenticated access; potential ceiling is Critical (≈9.1) pending authorized confirmation.</sub>

---

## 8. 🔎 Detailed Findings

### F-001 — Unauthenticated Authentication-Filter Bypass
**Upload + Document View + Pipeline Callback**

| | |
|---|---|
| **Severity** | 🔴 High *(potential Critical)* |
| **CVSS v3.1** | 8.1 — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` |
| **CWE** | CWE-306 (Missing Auth), CWE-434 (Unrestricted Upload), CWE-639 (IDOR) |
| **OWASP** | API1/API2:2023, A01:2021 |
| **Affected** | `api-core.<redacted>` — document service |

**Description.** Several document-service endpoints are processed **outside** the
authentication filter, while their functional siblings correctly return `401`. The
cluster exposes an unauthenticated **write** (file upload), an unauthenticated **read**
(document view), and an unauthenticated **callback** into a processing pipeline.

**Root cause (inferred).** Specific routes appear to be allow-listed/`permitAll()` in
the security configuration — an explicit bypass (the upload endpoint is literally named
`uploadWithoutToken`), rather than a generic misconfiguration. The upload accepts an
attacker-controlled `fileName` (path-traversal vector) and a `customerId` (IDOR/data-
poisoning vector).

**Evidence (sanitized; status-code & `OPTIONS` only — no exploitation).**
```bash
# WRITE path — siblings require auth, the bypass endpoint does not:
401  /core-api/docServ/upload
401  /core-api/docServ/uploadBodFile
500  /core-api/docServ/uploadWithoutToken      # <-- reached WITHOUT auth

$ curl -s -D - -o /dev/null -X OPTIONS .../docServ/uploadWithoutToken
HTTP/1.1 200
Allow: POST,OPTIONS

# READ path — sibling reads require auth, 'view' does not:
401  /core-api/docServ/download
401  /core-api/docServ/filelist
200  /core-api/docServ/view/1/1                # <-- reached WITHOUT auth

# CALLBACK — sibling callback requires auth, this one does not:
401  /core-api/bordereau/preVal-scan-callback
500  /core-api/bordereau/pdf-extraction-callback   (Allow: GET,HEAD,POST,OPTIONS)
```

**Confirmation attempt (non-destructive).** A controlled read-metadata test of
`docServ/view` against sequential IDs returned **empty `200` responses** — no document
body. This indicates the document reference is an **opaque, non-sequential identifier**,
so unauthenticated *document disclosure* was **not demonstrated**. The auth bypass is
confirmed; the impact ceiling was deliberately left unproven (see §10).

**Impact.** Unauthenticated read **and** write surface on a customer-data system:
potential document disclosure (IDOR), anonymous ingestion of malicious/forged files
into downstream pipelines (stored XSS, path traversal, data poisoning, worst-case RCE),
and forged processing callbacks. Discoverability is trivial given F-002 and F-003.

**Remediation.**
- Place all three endpoints behind the same authentication + **per-object
  authorization** as their protected siblings (or remove them).
- `docServ/view`: enforce ownership checks on `{param}`; use unguessable identifiers.
- Uploads: allow-list type/MIME/size, ignore client `fileName`, store outside web root
  with random names, disable execution, AV/CDR-scan, validate `customerId` against the
  authenticated principal, rate-limit.
- **Audit every anonymous route** (`*WithoutToken`, `*-callback`, `open*`) — this was a
  *class* of issue, not a one-off.

**References.** CWE-306, CWE-434, CWE-639 · OWASP API Security Top 10 (2023) · OWASP ASVS V4.

---

### F-002 — Unauthenticated Application Configuration Disclosure

| | |
|---|---|
| **Severity** | 🟠 Medium |
| **CVSS v3.1** | 5.3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` |
| **CWE** | CWE-200, CWE-538 |
| **Affected** | `api-core.<redacted>/api` |

**Description.** `GET /api` returns the full application runtime configuration as JSON
with no authentication (~7.7 KB): internal service base URLs, the identity realm name,
the login endpoint, and a complete allow-origins list enumerating ~90 internal/tenant
hostnames.

**Evidence.**
```bash
$ curl -s https://api-core.<redacted>/api
HTTP/1.1 200 OK   Content-Type: text/plain;charset=UTF-8   (7760 bytes)
{ "adminApiURL":"...", "realm":"<redacted>", "loginApiURL":"...",
  "allowedOrigins":"... ~90 internal/tenant hosts ..." }
```

**Impact.** High reconnaissance value — full backend topology + tenant enumeration.
Hard secrets were masked, capping direct impact at Medium.

**Remediation.** Require authentication on `/api` or serve only minimal client config;
enforce allowed origins **server-side**, never shipped to the browser.

**References.** CWE-200 · OWASP A05:2021.

---

### F-003 — Unauthenticated OpenAPI / Swagger Exposure

| | |
|---|---|
| **Severity** | 🟠 Medium |
| **CVSS v3.1** | 5.3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` |
| **CWE** | CWE-200 · OWASP API9:2023 |
| **Affected** | `api-core.<redacted>/core-api/v3/api-docs`, `/core-api/swagger-ui/` |

**Description.** The complete OpenAPI 3 specification (**347 endpoints**) and a live
Swagger UI are served without authentication, disclosing the entire API contract —
paths, parameters, schemas, data models — including the F-001 bypass endpoints.

**Evidence.**
```bash
$ curl -s -o /dev/null -w "%{http_code} %{size_download}\n" .../core-api/v3/api-docs
200 262488                                  # full spec, unauthenticated
.../core-api/swagger-ui/index.html  -> 200  # live Swagger UI
.../administration/swagger-ui.html  -> 401  # other segments correctly protected
```

**Impact.** Dramatically lowers attacker effort and advertises sensitive/upload
endpoints — a direct force-multiplier for F-001.

**Remediation.** Disable Swagger UI / `v3/api-docs` on internet-facing services or
require authentication; apply consistently across all API segments (core-api was the
outlier).

**References.** CWE-200 · OWASP API9:2023 Improper Inventory Management.

---

### F-004 — Verbose .NET Exception / Inconsistent Authentication

| | |
|---|---|
| **Severity** | 🟠 Medium |
| **CVSS v3.1** | 5.3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` |
| **CWE** | CWE-209, CWE-755 |
| **Affected** | `admin-portal.<redacted>/api/*` |

**Description.** Unauthenticated requests to admin-portal API routes return raw .NET
exception text instead of a generic error — confirming the backend technology and
indicating request processing reaches application logic before an auth decision.

**Evidence.**
```bash
$ curl -s -i .../api/applicationSettings
HTTP/1.1 500 Internal Server Error
{"ErrorMessage":"Object reference not set to an instance of an object."}
```

**Impact.** Technology fingerprinting + internal error leakage; indicator of
inconsistent authentication enforcement at the edge.

**Remediation.** Enforce authentication **before** controller logic (consistent
`401/403`); disable detailed error responses in internet-facing environments; add a
global exception handler that never serializes raw messages.

**References.** CWE-209, CWE-755 · OWASP A05:2021.

---

### F-005 — Overly Permissive CORS

| | |
|---|---|
| **Severity** | 🟡 Low · **CVSS** 4.3 · **CWE** 942 |
| **Affected** | `app-core.<redacted>/` |

**Description.** Returns `Access-Control-Allow-Origin: *` for any origin with a broad
allow-headers list (incl. `Authorization`, `username`, `customerid`).

**Evidence.**
```bash
$ curl -s -D - -H "Origin: https://evil.example" https://app-core.<redacted>/ | grep -i access-control
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization,...,username,customerid,...
```

**Impact / limiting factor.** No `Allow-Credentials`, so credentialed theft isn't
currently possible — but the config is fragile (adding credentials later makes it
exploitable). The API host correctly rejected untrusted preflight (`403`).

**Remediation.** Replace `*` with a server-side explicit allow-list; trim allow-headers.

**References.** CWE-942 · OWASP A05:2021.

---

### F-006 — Weak / Deprecated Security Headers

| | |
|---|---|
| **Severity** | 🟡 Low · **CVSS** 3.7 · **CWE** 693 / 1021 |
| **Affected** | `app-core.<redacted>/` |

**Evidence & analysis.**
```bash
X-Frame-Options: ALLOW-FROM *.<redacted>   # deprecated + invalid wildcard (ignored by browsers)
Permissions-Policy: *                       # invalid; effectively grants all features
X-Xss-Protection: 1; mode=block             # deprecated
Referrer-Policy: origin                      # weaker than recommended
Content-Security-Policy: ... 'unsafe-inline' 'unsafe-eval' ...   # weakens XSS mitigation
```
> Clickjacking is mitigated in practice by a valid CSP `frame-ancestors` directive.

**Remediation.** Rely on CSP `frame-ancestors`; set a valid restrictive
Permissions-Policy; remove deprecated headers; tighten Referrer-Policy; remove
`unsafe-*` from CSP (tied to F-007 migration).

**References.** CWE-693, CWE-1021 · OWASP Secure Headers Project.

---

### F-007 — End-of-Life Client-Side Components

| | |
|---|---|
| **Severity** | 🟡 Low · **CVSS** 4.8 · **CWE** 1104 / 1035 |
| **Affected** | `app-core.<redacted>/` |

**Evidence.**
```bash
$ curl -s https://app-core.<redacted>/ | grep -oiE "(angular.js|bootstrap)/[0-9.]+/"
angular.js/1.8.3/      # End-of-Life since Jan 2022 — no security patches
bootstrap/3.4.1/       # End-of-Life
```

**Impact.** Unfixable, accumulating client-side XSS/DOM risk, amplified by user-content
rendering (markdown/report components).

**Remediation.** Plan migration off AngularJS; upgrade libraries; adopt Software
Composition Analysis (SCA) to track EOL/CVE status.

**References.** CWE-1104, CWE-1035 · OWASP A06:2021.

---

### F-008 — Spring Boot Actuator Exposed (Limited)

| | |
|---|---|
| **Severity** | 🟡 Low · **CVSS** 3.1 · **CWE** 16 / 200 |
| **Affected** | `api-core.<redacted>/actuator*` |

**Evidence.**
```bash
$ curl -s .../actuator
{"_links":{"self":{},"health":{},"info":{},"refresh":{}}}
env / heapdump / threaddump / beans / mappings / loggers  -> 404   # correctly disabled
/actuator/refresh (GET) -> 405   # exists, expects POST — NOT invoked (state-changing)
```

**Impact.** Minor disclosure (confirms Spring Boot + Spring Cloud Config); `refresh`
presence is a hardening concern.

**Remediation.** Restrict `/actuator/**` to internal networks or require auth; do not
expose `refresh`; minimize anonymous `info/health` detail.

**References.** CWE-16, CWE-200.

---

### F-009 — Identity-Provider Admin Console & Privileged Realm Exposed

| | |
|---|---|
| **Severity** | 🟡 Low · **CVSS** 4.3 · **CWE** 284 / 419 |
| **Affected** | `login.<redacted>/auth/admin/...`, `/auth/realms/master/...` |

**Description.** The Keycloak admin console and the privileged `master` realm are
internet-reachable. The console login still requires credentials (not bypassed), but
exposure broadens the attack surface (credential stuffing, admin-console CVEs).

**Evidence.**
```bash
$ curl -s .../auth/admin/master/console/ | grep -i "<title>"
<title>Keycloak Administration Console</title>
.../auth/realms/master/.well-known/openid-configuration -> 200
# positive: anonymous dynamic client registration -> 404 (not exposed)
```

**Remediation.** Restrict the admin console / privileged realm to internal/VPN/allow-
listed networks; verify the IdP version is supported/patched; enforce admin MFA and
brute-force protection.

**References.** CWE-284, CWE-419 · Keycloak hardening guidance.

---

### F-010 — Weak Email Anti-Spoofing

| | |
|---|---|
| **Severity** | 🟡 Low · **CVSS** 4.3 · **CWE** 358 |
| **Affected** | Primary + secondary email domains *(redacted)* |

**Evidence.**
```bash
$ nslookup -type=TXT _dmarc.<redacted-domain>
"v=DMARC1; p=none; rua=mailto:..."        # monitor-only, not enforced
# secondary domain: no SPF / no DMARC returned
```

**Impact.** Enables convincing email spoofing / phishing impersonation of the org.

**Remediation.** Move DMARC to `p=quarantine` → `p=reject` after monitoring; ensure
SPF + DKIM alignment; publish records (or null-MX) for all owned domains.

**References.** CWE-358 · RFC 7489 (DMARC).

---

### F-011 — Missing Content-Security-Policy (Admin Portal)

| | |
|---|---|
| **Severity** | ⚪ Info · **CWE** 693 |
| **Affected** | `admin-portal.<redacted>/` |

**Evidence.**
```bash
$ curl -s -D - -o /dev/null https://admin-portal.<redacted>/ | grep -i content-security-policy
# (absent)  — present: X-Frame-Options: DENY, HSTS, X-Content-Type-Options: nosniff
```

**Remediation.** Add a restrictive CSP (`default-src 'self'`; `frame-ancestors 'none'`).

---

### F-012 — Third-Party Scripts Without Subresource Integrity

| | |
|---|---|
| **Severity** | ⚪ Info · **CWE** 829 |
| **Affected** | `app-core.<redacted>/` |

**Evidence.**
```bash
# CDN scripts loaded without integrity attributes
https://cdn.<reporting-vendor>.com/.../*.min.js
import { v4 as uuidv4 } from 'https://jspm.dev/uuid';   # unpinned module CDN
```

**Impact.** Supply-chain risk — a CDN compromise executes attacker code in-app.

**Remediation.** Add SRI hashes, pin exact versions, prefer self-hosting critical code,
avoid unpinned module CDNs in production.

---

### F-013 — AI/LLM Agent Subsystem — Attack-Surface Note

| | |
|---|---|
| **Severity** | ⚪ Info · **Ref** OWASP Top 10 for LLM Applications |
| **Affected** | AI chat / agent endpoints across all apps |

**Description.** The platform embeds an AI assistant that invokes server-side LLM agents
and tool-calling sub-agents. Endpoints required auth (`401/404`) and were **not**
exercised; documented as a high-value target for the authenticated phase.

**Evidence.**
```bash
$ curl -s -o /dev/null -w "%{http_code}" .../ai-integration/   -> 401
# client JS: aiChatBoxService.sendAgentMessage(agent, input, context, session, files)
# OpenAPI: /ai-generate/generate, /ai-generate/documents/upload, ...
```

**Why it matters.** Prompt injection (direct + via uploaded docs — synergy with F-001),
agent tool-abuse / privilege escalation, SSRF via agent requests, cross-tenant leakage,
and XSS via unsanitized LLM output rendering.

**Remediation.** Test under the OWASP LLM Top 10; sanitize agent output; enforce
per-user/per-tenant authorization on agent tools.

---

## 9. ✅ Positive Observations

Good controls observed (security reporting should credit what works):

- 🔒 **TLS 1.0/1.1 disabled**; TLS 1.2/1.3 with modern ECDHE-GCM ciphers
- 🔒 **HSTS** with `includeSubDomains; preload` on all hosts
- 🔒 Server version banners suppressed
- 🔒 Dangerous HTTP methods rejected (`TRACE/PUT/DELETE/PATCH` → `405`)
- 🔒 High-risk Actuator endpoints disabled (`env/heapdump/threaddump` → `404`)
- 🔒 The **majority** of API endpoints correctly enforce `401`
- 🔒 API CORS preflight rejects untrusted origins (`403`)
- 🔒 **No subdomain-takeover** (hosts CNAME to a live, owned AWS ELB)
- 🔒 **No host-header injection** (spoofed Host → `404`)
- 🔒 No exposed `.git` / `.env` / source-maps (SPA catch-all verified)

---

## 10. 🧭 The Ethical Decision: Where I Stopped

The most important part of this engagement wasn't a finding — it was a **decision**.

Confirming the F-001 document-view IDOR would have meant passing real identifiers and
reading back what could be **real customers' insurance documents (PII)**. With an
unauthenticated IDOR, *the confirmation and the breach are the same action.*

So I stopped at the **authentication-filter bypass** (status code + `OPTIONS`):
- ❌ No file was uploaded
- ❌ No document body was read — response bodies were discarded
- ❌ No identifiers were enumerated or brute-forced

I rated F-001 **High** (confirmed access), explicitly documented its **Critical
potential**, and recommended the impact be confirmed by the asset owner's team via a
**white-box code review** or an authorized, controlled exploitation test under a data-
handling protocol — not by an external party improvising against live data.

> **Why this matters:** Anyone can run a scanner. Professional security work is knowing
> **how far to go, when to stop, and how to escalate responsibly.** Severity is bounded
> by evidence, not by suspicion.

---

## 11. 🧰 Tooling

| Tool | Purpose |
|---|---|
| `curl` | All HTTP request/response testing & header analysis |
| `OpenSSL` | TLS protocol/cipher and certificate inspection |
| `httpx` | Fast HTTP probing & fingerprinting |
| `nslookup` | DNS, CNAME, and SPF/DMARC record enumeration |
| Manual analysis | JS/source review, OpenAPI spec analysis, CORS & header evaluation |

> Intentionally lightweight, low-rate, and non-intrusive — appropriate for a
> production-treated target with a strict "no DoS" ROE.

---

## 12. 🧠 Skills & Takeaways

- Black-box external reconnaissance & attack-surface mapping
- **API security testing** & authentication-enforcement analysis (auth-filter bypass detection)
- Information-disclosure & misconfiguration identification
- TLS, CORS, security-header, and email-spoofing assessment
- **Disciplined, ethical validation** — confirming impact without exploiting or exposing data
- CVSS v3.1 risk rating tied to evidence, with confirmed-vs-potential clearly separated
- Professional reporting & remediation guidance for both technical and executive audiences
- Sound judgment on **when to stop and escalate** (VA → white-box / authorized PT)

---

## 13. ⏭️ Recommended Next Steps

1. **Remediate F-001** — authentication + per-object authorization on all affected
   endpoints; audit the whole anonymous-route class. *(Highest priority.)*
2. **Restrict disclosure** — lock down the unauthenticated `/api` config (F-002) and the
   Swagger/OpenAPI exposure (F-003).
3. **Fix the rest** — CORS, headers, CSP, SRI, Actuator, IdP exposure, DMARC.
4. **Authenticated penetration test** — multi-tenancy isolation (IDOR/BOLA), authenticated
   injection, and the **AI/LLM agent subsystem** (OWASP LLM Top 10).
5. **Retest** after Priority 1–2 remediation to confirm closure.

---

<div align="center">

## 👤 Author

**MoriartyPuth** — Offensive Security

![GitHub](https://img.shields.io/badge/GitHub-MoriartyPuth-181717?logo=github)

</div>

> ⚠️ **Disclaimer.** _This document is a sanitised portfolio artefact produced from an
> authorised penetration test. All identifying data has been redacted or replaced with
> documentation-reserved placeholders. It contains no client data, no live targets and no
> novel exploit code. Techniques shown are standard, publicly documented, and provided for
> educational and defensive purposes only. Do not test any system you do not own or lack
> explicit written authorisation to assess._
