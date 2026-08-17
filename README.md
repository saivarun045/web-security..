# DVWA Web Application Vulnerability Assessment

A hands-on penetration testing lab exploring common OWASP Top 10 vulnerabilities using **DVWA (Damn Vulnerable Web Application)**, containerized with Docker and tested locally in a fully isolated, legal environment.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Platform](https://img.shields.io/badge/platform-DVWA-green)
![Docker](https://img.shields.io/badge/tool-Docker-blue)
![Security Level](https://img.shields.io/badge/security%20level-Low-red)

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Environment Setup](#-environment-setup)
- [Methodology](#-methodology)
- [Findings Summary](#-findings-summary)
- [Detailed Findings](#-detailed-findings)
  1. [SQL Injection](#1-sql-injection)
  2. [Blind SQL Injection](#2-blind-sql-injection)
  3. [Command Injection](#3-command-injection)
  4. [Reflected XSS](#4-reflected-cross-site-scripting-xss)
  5. [Stored XSS](#5-stored-cross-site-scripting-xss)
  6. [DOM-Based XSS](#6-dom-based-xss)
  7. [CSRF](#7-cross-site-request-forgery-csrf)
  8. [Unrestricted File Upload](#8-unrestricted-file-upload)
  9. [Weak Session IDs](#9-weak-session-ids)
  10. [Brute Force](#10-brute-force-authentication)
- [Conclusion](#-conclusion)
- [Disclaimer](#-disclaimer)
- [References](#-references)

---

## 🎯 Project Overview

| Field | Detail |
|---|---|
| **Target Application** | Damn Vulnerable Web Application (DVWA) v1.10 *Development* |
| **Objective** | Identify, exploit, and document common web application vulnerabilities in a safe, legal, sandboxed environment |
| **Environment** | Local Docker container on macOS (localhost) |
| **DVWA Security Level** | Low |
| **Scope** | `localhost` only — no external or production systems were tested |
| **Standards Referenced** | OWASP Top 10 |

This project was completed for educational purposes to build practical, hands-on experience in identifying and exploiting common web application security flaws, and to understand the corresponding secure coding practices that prevent them.

---

## 🛠 Environment Setup

The environment was provisioned locally using Homebrew and Docker:

1. Installed **Homebrew** package manager.
2. Verified **Java** and **Docker** installations.
3. Pulled the official vulnerable DVWA image:
   ```bash
   docker pull vulnerables/web-dvwa
   ```
4. Ran the container, mapping port `8080` (host) to port `80` (container):
   ```bash
   docker run --rm -it -p 8080:80 vulnerables/web-dvwa
   ```
5. Verified the container was running via `docker ps` and Docker Desktop.
6. Accessed DVWA at `http://localhost:8080`, logged in with default credentials (`admin` / `password`), and used **Setup / Reset DB** to initialize the database.
7. Confirmed **Security Level: Low** and **PHPIDS: disabled** for baseline (unmitigated) testing.

| Screenshot | Description |
|---|---|
| ![Terminal setup 1](screenshots/01-terminal-setup-1.png) | Homebrew, Java, and Docker installation checks |
| ![Terminal setup 2](screenshots/02-terminal-setup-2.png) | `docker pull vulnerables/web-dvwa` completing successfully |
| ![DVWA Home](screenshots/03-dvwa-home.png) | DVWA welcome/home page after successful setup |
| ![Security Level](screenshots/04-security-level-low.png) | DVWA Security page confirming Low security level |

---

## 🔍 Methodology

Each vulnerability was tested manually through the DVWA web interface:

1. Navigate to the relevant DVWA module.
2. Submit a benign/normal input to observe expected behavior.
3. Submit a crafted malicious payload designed to exploit a specific weakness (e.g. lack of input sanitization, missing output encoding, missing anti-CSRF tokens).
4. Capture the application's response as evidence.
5. Analyze the root cause and document the appropriate remediation.

All testing was performed at **DVWA Security Level: Low**, which intentionally has no security controls in place, to clearly demonstrate the underlying vulnerability classes.

---

## 📊 Findings Summary

| # | Vulnerability | OWASP Category | Severity | Status |
|---|---|---|---|---|
| 1 | SQL Injection | A03:2021 – Injection | Critical | ✅ Exploited |
| 2 | Blind SQL Injection | A03:2021 – Injection | High | ✅ Exploited |
| 3 | Command Injection | A03:2021 – Injection | Critical | ✅ Exploited |
| 4 | Reflected XSS | A03:2021 – Injection | Medium | ✅ Exploited |
| 5 | Stored XSS | A03:2021 – Injection | High | ✅ Exploited |
| 6 | DOM-Based XSS | A03:2021 – Injection | Medium | ✅ Exploited |
| 7 | CSRF | A01:2021 – Broken Access Control | Medium | ✅ Exploited |
| 8 | Unrestricted File Upload | A05:2021 – Security Misconfiguration | Critical | ✅ Exploited |
| 9 | Weak Session IDs | A07:2021 – Identification & Auth Failures | Medium | ✅ Observed |
| 10 | Brute Force (Login) | A07:2021 – Identification & Auth Failures | High | ✅ Demonstrated |

---

## 🧨 Detailed Findings

### 1. SQL Injection

**Location:** `/vulnerabilities/sqli/`

**Payload used:**
```sql
1' OR '1'='1' #
```

**Result:** Instead of returning a single user record, the query returned **every row in the users table**, confirming the application concatenates user input directly into a SQL query without sanitization or parameterization.

| Normal Input | Malicious Payload |
|---|---|
| ![Normal SQLi input](screenshots/05-sqli-normal-input.png) | ![SQLi payload result](screenshots/06-sqli-payload-result.png) |

**Root Cause:** User-supplied input is concatenated directly into a SQL query string.

**Remediation:**
- Use parameterized queries / prepared statements.
- Apply the principle of least privilege to the database account.
- Implement strict input validation and output encoding.

---

### 2. Blind SQL Injection

**Location:** `/vulnerabilities/sqli_blind/`

**Payload used:**
```sql
1' AND 1=1 #
```

**Result:** The application returned a generic "User ID exists in the database" message rather than actual data — but by toggling the injected boolean condition (`1=1` vs `1=2`) and observing the change in response, the presence of the SQL injection flaw and underlying data can still be inferred and extracted row-by-row.

![Blind SQL Injection](screenshots/19-sqli-blind.png)

**Root Cause:** Same as standard SQL Injection — unsanitized input reaches the SQL query — but application output does not directly display query results, requiring inferential (boolean/time-based) extraction techniques.

**Remediation:**
- Parameterized queries / prepared statements.
- Generic error handling that does not leak query state.
- Web Application Firewall (WAF) as a defense-in-depth layer.

---

### 3. Command Injection

**Location:** `/vulnerabilities/exec/`

**Payload used:**
```bash
127.0.0.1; whoami
```

**Result:** The application executed the injected `whoami` command in addition to the intended `ping`, returning the web server's process owner (`www-data`) — proof of arbitrary OS command execution on the host.

| Normal Ping | Command Injection |
|---|---|
| ![Normal ping](screenshots/07-cmdi-normal-ping.png) | ![Command injection success](screenshots/08-cmdi-payload-result.png) |

**Root Cause:** User input is passed unsanitized into a system shell command.

**Remediation:**
- Avoid invoking shell commands with user input entirely where possible.
- Use safe APIs/language-native functions instead of shell execution.
- Strictly whitelist and validate input (e.g., IP address format only).
- Run web server processes with minimal privileges.

---

### 4. Reflected Cross-Site Scripting (XSS)

**Location:** `/vulnerabilities/xss_r/`

**Payload used:**
```html
<script>alert('XSS')</script>
```

**Result:** The script payload was reflected back into the page and executed immediately in the browser, proving the `name` parameter is echoed into the HTML response without encoding.

| Input Form | Reflected Output | Alert Triggered |
|---|---|---|
| ![XSS reflected form](screenshots/09-xss-reflected-form.png) | ![XSS reflected result](screenshots/10-xss-reflected-result.png) | ![XSS reflected alert](screenshots/11-xss-reflected-alert.png) |

**Root Cause:** User input is rendered directly into the HTML response without output encoding/escaping.

**Remediation:**
- HTML-encode all user-supplied output.
- Implement a strict Content Security Policy (CSP).
- Use templating engines with automatic contextual escaping.

---

### 5. Stored Cross-Site Scripting (XSS)

**Location:** `/vulnerabilities/xss_s/`

**Payload used:**
```html
<script>alert('Stored XSS')</script>
```

**Result:** The payload was submitted through the guestbook "Message" field and permanently stored in the database. The script now executes automatically for **every user** who views the guestbook page — a significantly higher-impact vulnerability than reflected XSS, since it requires no direct interaction from the victim.

| Payload Submitted | Alert Triggered on Page Load |
|---|---|
| ![Stored XSS form](screenshots/12-xss-stored-form.png) | ![Stored XSS alert](screenshots/13-xss-stored-alert.png) |

**Root Cause:** User input is stored in the database and later rendered without sanitization or encoding.

**Remediation:**
- Sanitize input at time of storage and encode output at time of render.
- Implement CSP to restrict inline script execution.
- Use a well-maintained HTML sanitization library (e.g., DOMPurify) for any rich-text fields.

---

### 6. DOM-Based XSS

**Location:** `/vulnerabilities/xss_d/`

**Payload used:** Injected via the `default` URL parameter, manipulated client-side without any server round-trip.

**Result:** The payload executed purely within the browser's DOM, confirming that client-side JavaScript reads untrusted data (e.g., from `location.href`) and writes it back into the page (e.g., via `innerHTML` or `document.write`) without sanitization.

![DOM XSS alert](screenshots/18-dom-xss-alert.png)

**Root Cause:** Client-side JavaScript uses untrusted data (URL parameters) to modify the DOM without sanitization.

**Remediation:**
- Avoid `innerHTML`/`document.write` with untrusted data — use `textContent` or safe DOM APIs.
- Validate and encode data before DOM insertion.
- Apply CSP with `script-src` restrictions.

---

### 7. Cross-Site Request Forgery (CSRF)

**Location:** `/vulnerabilities/csrf/`

**Test performed:** Submitted the password-change form directly (simulating a forged cross-site request), without any anti-CSRF token being required or validated by the server.

**Result:** The admin password was changed successfully with **no token validation**, proving that a malicious third-party site could trick an authenticated admin into unknowingly changing their own password (or performing other state-changing actions) simply by visiting a crafted page.

![CSRF password changed](screenshots/16-csrf-password-changed.png)

**Root Cause:** The application does not implement anti-CSRF tokens or verify the request's origin for state-changing actions.

**Remediation:**
- Implement unique, unpredictable anti-CSRF tokens on all state-changing forms.
- Validate the `Origin`/`Referer` headers server-side.
- Use the `SameSite` cookie attribute (`Strict` or `Lax`).

---

### 8. Unrestricted File Upload

**Location:** `/vulnerabilities/upload/`

**Test performed:** Uploaded a file named `test.php` through the "Choose an image to upload" form, which is intended to accept only images.

**Result:** The server accepted and stored the PHP file without validating its type or content, confirming that an attacker could upload a **web shell** and achieve remote code execution on the server.

![File upload success](screenshots/15-file-upload-success.png)

**Root Cause:** The application does not validate file type (MIME type, extension, or file content/magic bytes) before storing uploaded files in a web-accessible directory.

**Remediation:**
- Validate file type via content inspection (not just extension).
- Store uploads outside the web root, or disable script execution in the upload directory.
- Rename uploaded files and enforce a strict allow-list of extensions.
- Scan uploads with anti-malware tooling.

---

### 9. Weak Session IDs

**Location:** `/vulnerabilities/weak_id/`

**Test performed:** Generated multiple session cookies (`dvwaSession`) using the "Generate" button and inspected them via browser DevTools → Storage → Cookies.

**Result:** The `dvwaSession` cookie value was observed to be a small, easily guessable/sequential value (in this case a low integer), rather than a long, cryptographically random token — making session hijacking via ID prediction feasible.

![Weak session IDs](screenshots/17-weak-session-ids.png)

**Root Cause:** Session identifiers are generated using a weak or predictable algorithm instead of a cryptographically secure random number generator.

**Remediation:**
- Generate session IDs using a cryptographically secure random number generator (CSPRNG).
- Ensure sufficient entropy/length (128+ bits) in session tokens.
- Set `HttpOnly`, `Secure`, and `SameSite` attributes on session cookies.

---

### 10. Brute Force (Authentication)

**Location:** `/vulnerabilities/brute/`

**Test performed:** Submitted repeated login attempts against the DVWA login form to assess rate-limiting and account lockout controls.

**Result:** The login form accepted unlimited login attempts with no lockout, delay, or CAPTCHA, confirming the endpoint is vulnerable to automated credential brute-forcing (e.g. via Hydra or Burp Intruder).

![Brute force page](screenshots/14-brute-force-page.png)

**Root Cause:** No rate-limiting, account lockout, or CAPTCHA is implemented on the authentication endpoint.

**Remediation:**
- Implement account lockout / exponential backoff after repeated failed attempts.
- Add CAPTCHA after a threshold of failed logins.
- Enforce strong password policies and multi-factor authentication (MFA).
- Log and alert on repeated failed login attempts.

---

## ✅ Conclusion

This project demonstrates practical, hands-on identification and exploitation of ten distinct vulnerability classes from the OWASP Top 10, using a fully containerized and isolated DVWA instance. Each finding was reproduced with a specific payload, evidenced with screenshots, and paired with root-cause analysis and industry-standard remediation guidance. This exercise reinforced the importance of secure coding practices — including input validation, output encoding, parameterized queries, anti-CSRF tokens, and secure session management — in preventing real-world web application attacks.

---

## ⚠️ Disclaimer

This project was conducted **exclusively** against a local, intentionally vulnerable application (DVWA) running in an isolated Docker container on `localhost`, strictly for educational purposes. **No external, third-party, or production systems were accessed or tested.** The techniques documented here should only ever be used in authorized, legal environments (e.g., your own lab, or systems you have explicit written permission to test).

---

## 📚 References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [DVWA GitHub Repository](https://github.com/digininja/DVWA)
- [OWASP Cross-Site Scripting (XSS)](https://owasp.org/www-community/attacks/xss/)
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP CSRF](https://owasp.org/www-community/attacks/csrf)
- [OWASP Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
