# Lab 3 — Secure Git Submission

## Task 1 — SSH Commit Signature Verification (Local Repository Config)

### 1.1 Commit Signing Security Benefits
- Commit signing provides **authenticity** (proves who authored the commit).
- It provides **integrity** (detects tampering after commit creation).
- In DevSecOps workflows, signed commits strengthen software supply-chain trust, improve auditability, and reduce spoofed-identity risk in collaborative repositories.

### 1.2 SSH Signing Setup (Local Only)
I configured signing **only for this repository** using `--local` (no global signing settings were used for setup).

Commands used:

```bash
git config --local gpg.format ssh
git config --local commit.gpgSign true
git config --local user.signingkey /home/dart/.ssh/id_ed25519.pub
```

Verification output:

```text
Local signing config:
ssh
true
/home/dart/.ssh/id_ed25519.pub
```

### 1.3 Signed Commit Evidence + Analysis
- Local repository now enforces signed commits via `commit.gpgSign=true`.
- Signed commit created in this repository:

```text
6920a79 docs: add lab3 submission and secure git evidence
```

- Local signature verification configured with:

```bash
git config --local gpg.ssh.allowedSignersFile .git_allowed_signers
```

Verification output:

```text
commit 6920a79b27095275f80803db177950d825880a85
Good "git" signature for oleynik.makc@gmail.com with ED25519 key
```

Why commit signing is critical in DevSecOps:
- Prevents unauthorized identity impersonation in commit history.
- Supports non-repudiation and stronger forensic/audit trails.
- Improves trust in CI/CD pipelines that consume repository commits.
- Helps enforce policy gates (e.g., only verified commits allowed).

GitHub “Verified” badge note:
- Badge confirmation happens on GitHub after pushing signed commits from this repo.
- Include a screenshot of the commit’s **Verified** badge in the PR as final evidence.

---

## Task 2 — Pre-commit Secret Scanning (TruffleHog + Gitleaks)

### 2.1 Hook Setup
Created `.git/hooks/pre-commit` with Dockerized TruffleHog + Gitleaks scanning logic for staged files.

Executable permission evidence:

```text
-rwxrwxr-x 1 dart dart 3325 Feb 22 20:26 .git/hooks/pre-commit
```

### 2.2 Secret Detection Tests

#### Test A — Blocked Commit (secret present)
Test file staged: `labs/lab3-secret-test.txt` containing a private-key pattern.

Observed output (excerpt):

```text
[pre-commit] scanning staged files for secrets…
[pre-commit] Files to scan: labs/lab3-secret-test.txt
[pre-commit] TruffleHog scan on non-lectures files…
[pre-commit] ✓ TruffleHog found no secrets in non-lectures files
[pre-commit] Gitleaks scan on staged files…
[pre-commit] Scanning labs/lab3-secret-test.txt with Gitleaks...
Gitleaks found secrets in labs/lab3-secret-test.txt:
RuleID:      private-key
File:        labs/lab3-secret-test.txt
Line:        1
...
✖ Secrets found in non-excluded file: labs/lab3-secret-test.txt

[pre-commit] === SCAN SUMMARY ===
TruffleHog found secrets in non-lectures files: false
Gitleaks found secrets in non-lectures files: true
Gitleaks found secrets in lectures files: false

✖ COMMIT BLOCKED: Secrets detected in non-excluded files.
Fix or unstage the offending files and try again.
HOOK_EXIT=1
```

Result: ✅ commit correctly blocked.

#### Test B — Successful Commit Check (secret removed)
Test file updated to non-sensitive text and re-staged.

Observed output (excerpt):

```text
[pre-commit] scanning staged files for secrets…
[pre-commit] Files to scan: labs/lab3-secret-test.txt
[pre-commit] TruffleHog scan on non-lectures files…
[pre-commit] ✓ TruffleHog found no secrets in non-lectures files
[pre-commit] Gitleaks scan on staged files…
[pre-commit] Scanning labs/lab3-secret-test.txt with Gitleaks...
[pre-commit] No secrets found in labs/lab3-secret-test.txt

[pre-commit] === SCAN SUMMARY ===
TruffleHog found secrets in non-lectures files: false
Gitleaks found secrets in non-lectures files: false
Gitleaks found secrets in lectures files: false

✓ No secrets detected in non-excluded files; proceeding with commit.
HOOK_EXIT=0
```

Result: ✅ commit allowed after remediation.

### 2.3 Analysis: Why automated secret scanning matters
- Detects leaked credentials early (before remote push/PR exposure).
- Reduces incident response cost by stopping mistakes at commit time.
- Creates consistent, developer-local enforcement independent of reviewer oversight.
- Complements CI scanning by shifting security left into daily workflows.

---

## Final Checklist
- [x] Task 1 completed with local SSH signing configuration and analysis
- [x] Task 2 completed with working pre-commit hook and test evidence
- [x] Blocked and successful scan outcomes documented
- [ ] GitHub “Verified” badge screenshot to be attached after pushing signed commit to PR
- [ ] PR URL to be submitted via Moodle
