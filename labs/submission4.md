# Lab 4 — SBOM Generation & SCA Comparison (Juice Shop v19.0.0)

## Scope and Setup
- Target image: `bkimminich/juice-shop:v19.0.0`
- Tools: Syft, Grype, Trivy (Dockerized)
- Output directories: `labs/lab4/syft`, `labs/lab4/trivy`, `labs/lab4/analysis`, `labs/lab4/comparison`

Generated artifacts include:
- Syft SBOM: `labs/lab4/syft/juice-shop-syft-native.json`
- Trivy package inventory: `labs/lab4/trivy/juice-shop-trivy-detailed.json`
- Grype findings: `labs/lab4/syft/grype-vuln-results.json`
- Trivy findings: `labs/lab4/trivy/trivy-vuln-detailed.json`
- Analysis summaries: `labs/lab4/analysis/sbom-analysis.txt`, `labs/lab4/analysis/vulnerability-analysis.txt`
- Accuracy comparison: `labs/lab4/comparison/accuracy-analysis.txt`

---

## Task 1 — SBOM Generation with Syft and Trivy

### Package Type Distribution
From `labs/lab4/analysis/sbom-analysis.txt`:
- **Syft package counts**
  - `binary`: 1
  - `deb`: 10
  - `npm`: 1128
- **Trivy package counts**
  - Debian OS packages: 10
  - Node.js packages: 1125

### Dependency Discovery Analysis
- Total discovered package entries are very close between tools, with Syft finding slightly more Node packages in this run.
- Quantified overlap from `labs/lab4/comparison/accuracy-analysis.txt`:
  - Packages detected by both tools: **1126**
  - Packages only detected by Syft: **13**
  - Packages only detected by Trivy: **9**
- Practical interpretation:
  - Both tools provide strong coverage for this image.
  - Minor differences likely come from ecosystem parsers, metadata normalization, and how virtual/transitive dependencies are represented.

### License Discovery Analysis
From `labs/lab4/analysis/vulnerability-analysis.txt`:
- Syft unique license types: **32**
- Trivy unique license types: **28**

From `labs/lab4/analysis/sbom-analysis.txt` and `labs/lab4/trivy/trivy-licenses.json`:
- Most common licenses in dependencies are permissive (MIT, ISC, BSD variants), but copyleft and mixed-expression licenses are present.
- Trivy’s top licenses include: MIT (878), ISC (143), LGPL-3.0-only (19), BSD-3-Clause (14), Apache-2.0 (13).

---

## Task 2 — Software Composition Analysis with Grype and Trivy

### SCA Tool Comparison (Vulnerability Detection)
From `labs/lab4/analysis/vulnerability-analysis.txt`:
- **Grype severities**
  - Critical: 11
  - High: 88
  - Medium: 32
  - Low: 3
  - Negligible: 12
- **Trivy severities**
  - CRITICAL: 10
  - HIGH: 81
  - MEDIUM: 34
  - LOW: 18

From `labs/lab4/comparison/accuracy-analysis.txt`:
- CVEs found by Grype: **95**
- CVEs found by Trivy: **91**
- Common CVEs: **26**

Interpretation:
- Grype and Trivy produce overlapping but non-identical vulnerability sets.
- Cross-validating critical findings across both tools is necessary before prioritizing remediation.

### Critical Vulnerabilities Analysis (Top 5 + Remediation)
Representative critical findings extracted from Trivy and Grype outputs:

1. **CVE-2015-9235** (`jsonwebtoken` 0.1.0) — fixed in `4.2.2`
   - Risk: token verification bypass / auth integrity issues in old JWT libraries.
   - Remediation: upgrade `jsonwebtoken` to `>=4.2.2` (prefer current maintained major).

2. **CVE-2019-10744** (`lodash` 2.4.2) — fixed in `4.17.12`
   - Risk: prototype pollution class vulnerabilities.
   - Remediation: upgrade to patched Lodash (`>=4.17.12`, preferably latest 4.17.x).

3. **CVE-2023-32314** (`vm2` 3.9.17) — fixed in `3.9.18`
   - Risk: sandbox escape.
   - Remediation: upgrade `vm2` at minimum to `3.9.18` (newer safe version preferred).

4. **CVE-2023-37466** (`vm2` 3.9.17) — fixed in `3.10.0`
   - Risk: additional sandbox escape path.
   - Remediation: upgrade `vm2` to `>=3.10.0` and retest sandbox functionality.

5. **CVE-2025-15467** (`libssl3` 3.0.17-1~deb12u2) — fixed in `3.0.18-1~deb12u2`
   - Risk: crypto/TLS library weakness at OS layer.
   - Remediation: rebuild image on updated base OS packages and patch regularly.

### License Compliance Assessment
Observed potentially higher-compliance-risk license families include:
- GPL variants (`GPL-1.0`, `GPL-2.0`, `GPL-3.0`)
- LGPL variants (`LGPL-2.1`, `LGPL-3.0`)
- Mixed expressions (e.g., `(BSD-2-Clause OR MIT OR Apache-2.0)`)

Recommendations:
- Maintain an allowlist/denylist policy per business legal requirements.
- Add CI checks to fail builds on disallowed licenses.
- Require manual legal review for copyleft dependencies and dual-license expressions.

### Additional Security Features (Secrets Scanning)
From `labs/lab4/trivy/trivy-secrets.txt`:
- Trivy secret scanner flagged findings including:
  - `HIGH: AsymmetricPrivateKey (private-key)`
  - `MEDIUM: JWT (jwt-token)`
- Practical handling:
  - Validate whether these are test/demo artifacts or false positives.
  - Treat any real key/token in images as incident-level and rotate/revoke immediately.

---

## Task 3 — Toolchain Comparison: Syft+Grype vs Trivy All-in-One

### Accuracy Analysis (Quantified)
From `labs/lab4/comparison/accuracy-analysis.txt`:
- Package overlap is high: **1126 common packages**.
- Detection deltas are small but non-zero (Syft-only 13, Trivy-only 9).
- Vulnerability overlap is lower than package overlap (common CVEs 26), showing differences in advisory databases, matching logic, and scoring normalization.

### Tool Strengths and Weaknesses
**Syft + Grype (specialized chain)**
- Strengths:
  - Strong SBOM detail and flexible SBOM-first workflows.
  - Good for decoupling inventory generation from vulnerability scanning.
- Weaknesses:
  - Two-tool operational complexity (pipelines, updates, artifact handoff).
  - Additional integration effort for secrets/license scanning.

**Trivy (all-in-one)**
- Strengths:
  - Unified scanner for vulnerabilities, licenses, and secrets.
  - Easy CI/CD integration with a single command surface.
- Weaknesses:
  - DB download overhead for ephemeral/containerized runs if cache is not persisted.
  - Less modular when teams want separate SBOM and vuln tool ownership.

### Use Case Recommendations
- Choose **Syft+Grype** when:
  - You want SBOM-centric processes, reusable SBOM artifacts, and modular security toolchains.
  - You need independent control of inventory and vulnerability scanning stages.
- Choose **Trivy** when:
  - You want simple, fast adoption and one integrated scanner for vuln+license+secret checks.
  - You prioritize developer ergonomics and low onboarding overhead.

### Integration Considerations (CI/CD and Operations)
- Persist scanner caches (`trivy-db`) between CI runs to reduce network time and improve reliability.
- Export SBOM and scan artifacts per build for audit trails.
- Gate PRs/builds on severity thresholds and policy rules.
- Run periodic rescans of released images because advisory databases evolve over time.

---

## Issues Encountered
- Trivy DB refresh can be slow in ephemeral Docker runs; cache persistence is important for stable pipeline timing.
- Scan interruption during DB download causes failed runs; rerun succeeds once scan completes end-to-end.

## Final Checklist
- [x] Task 1 done — SBOM generation with Syft and Trivy + analysis
- [x] Task 2 done — SCA with Grype and Trivy + vulnerability/license/secrets assessment
- [x] Task 3 done — quantitative toolchain comparison and recommendations
- [x] `labs/submission4.md` created
- [x] Supporting artifacts generated under `labs/lab4/`
