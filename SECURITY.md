# Security Audit Report

## Overview

This document summarises the security audit performed on the `openclaw-self-healing` repository, describes the issues found, and details the fixes applied.

**Audit date:** 2026-02-28  
**Verdict:** No backdoors or malware payloads were found. Several security vulnerabilities were identified and fixed.

---

## Findings

### ✅ No Backdoors or Active Malware

A full review of all shell scripts and Python code found:

- No hidden command-and-control (C2) connections to malicious servers.
- No reverse-shell payloads.
- No data-exfiltration code.
- No obfuscated (base64-encoded, hex-encoded, etc.) payloads.
- No hardcoded malicious credentials.
- All external network calls are to user-supplied Discord/Telegram webhook URLs, GitHub raw content, or the local OpenClaw gateway.

---

### 🔧 Vulnerabilities Fixed

#### 1. JSON Injection in Webhook Notifications

**Files:** `scripts/lib/notify.sh`, `scripts/gateway-healthcheck.sh`, `scripts/emergency-recovery-v2.sh`  
**Severity:** Medium

**Description:** Alert message strings were interpolated directly into JSON payloads without escaping, e.g.:

```bash
# BEFORE (vulnerable)
-d "{\"content\": \"$message\"}"
```

If a message contained unescaped double-quotes, backslashes, or control characters (possible via crafted log output), the JSON payload could be malformed or allow unintended content injection into webhook calls.

**Fix:** Message strings are now encoded using `jq -R -s '.'` when `jq` is available, falling back to `python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))'`, with a basic `sed`-based escape as a last resort.

---

#### 2. Scripts Downloaded from a Non-Existent Path

**Files:** `install.sh`, `install-linux.sh`  
**Severity:** High (functional breakage / supply-chain risk)

**Description:** Both installer scripts attempted to download operational scripts from:

```
https://raw.githubusercontent.com/Ramsbaby/openclaw-self-healing/main/skills/openclaw-self-healing/scripts/<file>
```

That path (`skills/openclaw-self-healing/scripts/`) does not exist in the repository. The actual scripts are located at `scripts/`. This would cause all downloads to fail with HTTP 404 errors — and if a future maintainer inadvertently created that path with different content, the wrong scripts would be executed.

**Fix:** The base URL was corrected to `$REPO_RAW/scripts`.

---

#### 3. No TLS Version Enforcement in curl Downloads

**Files:** `install.sh`, `install-linux.sh`  
**Severity:** Medium

**Description:** Script downloads used `curl -sSL` without enforcing a minimum TLS version. The `-L` flag follows redirects silently. An attacker performing a TLS-downgrade or DNS-poisoning attack could potentially redirect the download to serve malicious scripts.

**Fix:** Added `--proto '=https' --tlsv1.2` to all `curl` download invocations in the installer scripts to enforce HTTPS and require at least TLS 1.2.

---

#### 4. Unused `FINNHUB_TOKEN` in `.env.example`

**Files:** `.env.example`  
**Severity:** Low (information disclosure / confusion)

**Description:** The `.env.example` template included a `FINNHUB_TOKEN` placeholder for a financial market data API (Finnhub). No code in the repository uses this variable. Its presence could:

- Lead users to paste a real Finnhub API key that would never be used.
- Confuse users about the scope of the tool.

**Fix:** Removed the unused `FINNHUB_TOKEN` entry from `.env.example`.

---

### ⚠️ Residual Risk (Not Fixed — By Design)

#### Pipe-to-Bash Installation Pattern

Both `install.sh` and `install-linux.sh` document a one-line installation method:

```bash
curl -sSL https://raw.githubusercontent.com/Ramsbaby/openclaw-self-healing/main/install.sh | bash
```

This is a common pattern for developer tools but is inherently risky: if the GitHub repository or the CDN serving raw content is compromised, a user running this command would execute the attacker's code.

**Mitigations already in place:**
- The fixed `curl` calls now enforce `--proto '=https' --tlsv1.2`.
- Scripts are hosted on GitHub, which enforces HTTPS and has its own integrity controls.

**Recommended additional hardening (not implemented — would require CI/CD changes):**
- Publish SHA-256 checksums alongside each release and verify them in the installer.
- Sign releases with a GPG key and verify the signature before execution.

---

## Reporting a Vulnerability

If you discover a security vulnerability in this project, please open a GitHub issue marked **[Security]** or contact the repository owner directly. Do not disclose security vulnerabilities publicly until they have been addressed.
