# Lab 5 — Security Analysis (SAST + Multi-Tool DAST)

## Scope
- Target application: `bkimminich/juice-shop:v19.0.0`
- SAST tool: Semgrep
- DAST tools: ZAP (unauth + auth), Nuclei, Nikto, SQLmap
- Evidence files generated under `labs/lab5/`

---

## Task 1 — SAST with Semgrep

### 1) SAST Tool Effectiveness
From Semgrep outputs (`labs/lab5/semgrep/semgrep-results.json` and `labs/lab5/analysis/sast-analysis.txt`):
- Files scanned: **1014**
- Findings: **25**
- Rules run: 140 (security-audit + owasp-top-ten)

Vulnerability patterns detected by Semgrep include:
- SQL injection risks in Sequelize query usage
- Open redirect / possible user-input redirects
- Raw HTML formatting / script-tag related unsafe patterns (XSS risk)
- Unquoted template attribute variables in HTML templates
- JWT hardcoded secret pattern
- Express directory-listing / sendFile hardening issues

Coverage assessment:
- High code coverage across backend/frontend TypeScript/JS files.
- Findings are primarily code-level flaws and insecure coding patterns rather than runtime config issues.

### 2) Critical Vulnerability Analysis (Top 5)

1. **SQL Injection (Sequelize tainted query)**
   - File/line: `src/data/static/codefixes/dbSchemaChallenge_1.ts:5`
   - Severity: **ERROR**

2. **SQL Injection (Sequelize tainted query)**
   - File/line: `src/data/static/codefixes/dbSchemaChallenge_3.ts:11`
   - Severity: **ERROR**

3. **SQL Injection (Sequelize tainted query)**
   - File/line: `src/data/static/codefixes/unionSqlInjectionChallenge_1.ts:6`
   - Severity: **ERROR**

4. **SQL Injection (Sequelize tainted query)**
   - File/line: `src/data/static/codefixes/unionSqlInjectionChallenge_3.ts:10`
   - Severity: **ERROR**

5. **Template Injection / XSS Risk (unquoted attribute variable)**
   - File/line: `src/frontend/src/app/navbar/navbar.component.html:17`
   - Severity: **WARNING**

---

## Task 2 — DAST with Multiple Tools

### 1) Authenticated vs Unauthenticated Scanning (ZAP)
From `labs/lab5/analysis/zap-comparison.txt`:

- **Unauthenticated scan**
  - Total alerts: 12
  - Severity: High 0, Medium 2, Low 6, Info 4
  - Unique URLs with findings: 16

- **Authenticated scan**
  - Total alerts: 13
  - Severity: High 1, Medium 4, Low 4, Info 4
  - Unique URLs with findings: 22

Observed authenticated endpoint example:
- `http://localhost:3000/rest/admin/application-configuration`

Why authenticated scanning matters:
- It reaches protected/admin attack surface not visible to anonymous crawling.
- It reveals findings with higher business impact (e.g., auth/session/admin paths).
- It improves practical coverage for real user and admin workflows.

### 2) Tool Comparison Matrix

| Tool | Findings | Severity Breakdown | Best Use Case |
|---|---:|---|---|
| ZAP (authenticated) | 13 alert types | High 1, Medium 4, Low 4, Info 4 | Full web-app assessment, authenticated crawling, active testing |
| Nuclei | 23 matches | Info 21, Medium 1, Unknown 1 | Fast known-issue/template-based checks in CI/CD |
| Nikto | 82 items | Mostly informational/server exposure style findings | Web server/header/misconfiguration discovery |
| SQLmap | 2 injection points | SQLi (boolean-based blind) | Deep SQL injection validation and exploitation |

### 3) Tool-Specific Strengths + Example Findings

- **ZAP** (comprehensive DAST + auth support)
  - Strengths: authenticated spider/AJAX spider, passive+active web testing, consolidated reporting.
  - Example findings: `SQL Injection` (HIGH), `Content Security Policy Header Not Set` (MED).

- **Nuclei** (fast template engine)
  - Strengths: quick broad checks using community templates, easy automation.
  - Example findings: `Public Swagger API - Detect`, `Missing Subresource Integrity`.

- **Nikto** (server hardening checks)
  - Strengths: header issues and exposed resource checks.
  - Example findings: missing `X-XSS-Protection`, unusual header exposure (`x-recruiting`, `feature-policy`).

- **SQLmap** (specialized SQLi)
  - Strengths: precise SQL injection detection, DBMS fingerprinting, extraction support.
  - Example findings: injectable search parameter (`/rest/products/search?q=*`) and injectable JSON login parameter; DBMS identified as SQLite.

---

## Task 3 — SAST/DAST Correlation and Security Assessment

### 1) SAST vs DAST Comparison

Counts from generated outputs:
- SAST findings (Semgrep): **25**
- Combined DAST findings (ZAP + Nuclei + Nikto + SQLmap): **120** (13 + 23 + 82 + 2)

Vulnerability types found mainly/only by **SAST** in this lab:
- ORM/query-construction SQLi code patterns before runtime
- Hardcoded JWT secret pattern
- Template-code unsafe patterns (unquoted variable attributes)

Vulnerability types found mainly/only by **DAST** in this lab:
- Missing/weak HTTP security headers (CSP, X-Content-Type-Options, X-XSS-Protection)
- Cross-domain/CORS-style runtime misconfiguration indicators
- Runtime endpoint exposure and auth-surface issues (admin endpoints, robots/exposed paths)

Why results differ:
- SAST analyzes source code and catches insecure coding patterns even if not directly reachable in a running deployment.
- DAST exercises the running app over HTTP and catches deployment/runtime behavior, headers, routing, auth/session handling, and externally visible exposure.
- The two approaches are complementary; using both substantially increases confidence and coverage.

---

## Security Recommendations

1. **Shift-left SAST in CI**: run Semgrep on every PR, block ERROR-level findings.
2. **Authenticated DAST in staging**: schedule ZAP auth scans regularly, not only anonymous baseline scans.
3. **Header hardening**: enforce CSP, X-Content-Type-Options, anti-clickjacking, and stronger CORS policy.
4. **SQLi prevention**: parameterized queries everywhere, strict input validation, centralized ORM safe patterns.
5. **Risk-based triage**: prioritize exploitable HIGH findings first (e.g., SQLi), then medium config weaknesses.

---

## Evidence Files
- SAST:
  - `labs/lab5/semgrep/semgrep-results.json`
  - `labs/lab5/semgrep/semgrep-report.txt`
  - `labs/lab5/analysis/sast-analysis.txt`
- ZAP:
  - `labs/lab5/zap/zap-report-noauth.json`
  - `labs/lab5/zap/report-noauth.html`
  - `labs/lab5/zap/zap-report-auth.json`
  - `labs/lab5/zap/report-auth.html`
  - `labs/lab5/analysis/zap-comparison.txt`
- Other DAST:
  - `labs/lab5/nuclei/nuclei-results.json`
  - `labs/lab5/nikto/nikto-results.txt`
  - `labs/lab5/sqlmap/localhost/log`
  - `labs/lab5/analysis/dast-summary.txt`
- Correlation:
  - `labs/lab5/analysis/correlation.txt`

---

## Final Checklist
- [x] Task 1 done — SAST Analysis with Semgrep
- [x] Task 2 done — DAST Analysis (ZAP + Nuclei + Nikto + SQLmap)
- [x] Task 3 done — SAST/DAST Correlation
