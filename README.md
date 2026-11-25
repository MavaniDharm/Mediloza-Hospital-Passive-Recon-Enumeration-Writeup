# Mediroza General Hospital — Black-Box Penetration Test

**Networkwalks Academy — Week 4 Training Project**

> ⚠️ **Disclaimer:** This was a simulated training exercise conducted in a controlled lab environment provided by Networkwalks Academy, with explicit written authorisation for all milestones. `medirozahospital.com` is a fictional training target created for this course — no real hospital, patient, or individual is involved. These techniques must never be used against any system without explicit written permission from its owner.

## Overview

| | |
|---|---|
| **Client** | Mediroza General Hospital (simulated) |
| **Target** | `https://medirozahospital.com` |
| **Engagement Type** | Black-box Penetration Test |
| **Duration** | 5 days |
| **Objective** | Identify and exploit vulnerabilities, retrieve and decrypt confidential patient data, and uncover any further critical exposure on the server |

## Tools Used

- `curl` (PowerShell) — HTTP requests, cookie/session handling, file retrieval
- PowerShell `Select-String` / `Get-Content` — response parsing & pattern extraction
- Networkwalks **Hash Calculator** — PDF encryption hash extraction (pdf2john / hashcat `$pdf$` format)
- Networkwalks **Password Cracker (Dictionary Attack Lab)** — offline dictionary attack
- [Openwall John the Ripper `password.lst`](https://github.com/openwall/john) (1,795,708 entries) — extended wordlist
- Standard PDF viewer — verification of decrypted content

## Summary of Findings

| ID | Finding | Risk |
|----|---------|------|
| F1 | Sensitive path disclosure via `robots.txt` | 🟢 Low |
| F2 | SQL injection — patient portal authentication bypass | 🔴 Critical |
| F3 | Insecure Direct Object Reference (IDOR) — patient report download | 🟠 High |
| F4 | Weak / crackable PDF encryption passwords | 🟡 Medium |
| F5 | Publicly exposed legacy database backup (employee PII, salaries, shareholder data) | 🔴 Critical |

---

## M1 — Initial Access

### Recon: `robots.txt` discloses hidden paths

A request to `/robots.txt` revealed three disallowed paths — `/patient/`, `/staff/`, `/old/` — plus a sitemap. These are not access controls; they just hand an attacker a map of the non-public areas of the site.



### F2: SQL injection — authentication bypass

The login form at `/patient/login.php` had no input sanitisation. Submitting:

```
username=admin'-- &password=x
```

truncated the backend SQL query at the injected comment, bypassing the password check entirely. The resulting session cookie granted full access to the authenticated patient portal (`portal.php`), which also exposed three `download.php?id=N` links.



### F3: IDOR on `download.php` — patient report leakage

`download.php?id=N` used small, sequential, unvalidated IDs with no check that the session was authorised for that specific patient's record. Simply requesting `id=1`, `id=2`, `id=3` returned three different patients' confidential PDF lab reports.



---

## M2 — Data Extraction (Decrypting the PDFs)

Each retrieved PDF was encrypted (PDF Revision 3, 128-bit). A crackable hash (`$pdf$...`, pdf2john/hashcat-compatible) was extracted from each file and attacked offline.

### report1.pdf — cracked instantly (100-word list)



### report2.pdf — cracked instantly (100-word list)



### report3.pdf — required a larger wordlist

The built-in 100-password list failed completely:



A larger wordlist (Openwall John the Ripper `password.lst`, 1,795,708 entries) was downloaded and used instead:



### Recovered contents — confirmed genuine patient data

All three PDFs opened successfully, confirming real (simulated) pathology lab reports:

| File | Password | Patient |
|------|----------|---------|
| report1.pdf | `123456` | Sipho Dlamini (MG-P-10231) |
| report2.pdf | `password` | Priya Reddy (MG-P-10244) |
| report3.pdf | `!@#$%^&` | Emily Thompson (MG-P-10258) |



---

## M3 — Critical Data Exposure

Following up on the `/old/` path flagged in `robots.txt` (F1), a legacy database backup — `mediroza_db_backup_2019.sql` — was found publicly accessible with **no authentication required**.

### F5: Shareholder data exposed

```
curl.exe -s https://medirozahospital.com/old/mediroza_db_backup_2019.sql -o backup.sql
```

Parsing the backup revealed the full `shareholders` table — 10 individual/corporate shareholders and their equity stakes.



### Employee salaries & PII exposed

The same backup also contained the complete `employees` table: name, title, department, phone number, **national ID number**, and salary for all 28 hospital staff.



---

## M4 — Report

The full professional penetration testing report (Executive Summary, Scope & Methodology, Findings & Proof of Exploitation, Risk Ratings, Recommendations) is included in this repo:


## Key Takeaways / Recommendations

- Use **parameterised queries** everywhere — never concatenate user input into SQL.
- Enforce **server-side authorisation** on every object reference, not just authentication (fixes IDOR).
- Never let users set weak, common passwords on documents containing sensitive data; use strong random passphrases or a secure portal instead.
- **Never store backups in a web-accessible path.** `/old/` should never have been reachable from the internet.
- Don't rely on `robots.txt` to hide sensitive directories — it's a public, plain-text file.

---

*Educational project completed as part of the Networkwalks Academy penetration testing course. All testing was performed against an authorised training target only.*

Author

Mr. Mavani Dharm 
A cyber-Security student
