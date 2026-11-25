![banner](banner.png)

# Mediloza Hospital — Passive Recon & Enumeration Writeup

**Educational only.** This repository is a practice/lab writeup of passive recon and light enumeration against a demo hospital web app. Nothing here is a real, paid audit, and none of it should be treated as an official security assessment.

**Target:** `https://medirozahospital.com`
**Environment:** Windows PowerShell 5.1
**Date:** November 2024
**Status:** Recon, enumeration, and credential/password work complete

---

## Table of Contents

1. [Disclaimer](#disclaimer)
2. [Methodology](#methodology)
3. [Findings](#findings)
4. [Evidence](#evidence)
5. [Lessons learned](#lessons-learned)
6. [Reproduction](#reproduction)
7. [References](#references)

---

## Disclaimer

- Mediloza is a demo/teaching target. All commands and scripts assume a lab environment.
- No data here is guaranteed current or authoritative.
- Do not point these scripts at production systems without permission.
- Reports and database dumps are included for study, not because they were "leaked."

---

## Methodology

Recon was passive-first, then moved to directory discovery, form enumeration, session handling, and finally password recovery on a downloaded artifact.

| Phase | Technique | Result |
|-------|-----------|--------|
| 1 | `robots.txt` review | Hidden paths (`/patient/`, `/staff/`, `/old/`) |
| 2 | Sitemap retrieval | Full route map |
| 3 | Directory discovery | `/patient/portal.php`, `/patient/download.php` |
| 4 | Form enumeration | Login POST with injection-style payload |
| 5 | Session handling | `cookies.txt` persisted, reused across requests |
| 6 | Authenticated artifact retrieval | `report1..3.pdf` downloaded |
| 7 | Hash inspection | `$pdf$2*3*128*...` (PDF 1.7 / R3 / 128-bit) |
| 8 | Password recovery | Dictionary attack, plaintext recovered |

---

## Findings

### 1. `robots.txt` discloses directory structure
`Disallow: /patient/`, `/staff/`, `/old/` — none were protected, only de-listed. Sitemap published in the same file.

### 2. Login form accepts SQL-style credentials
Submitting `admin'--` with any password succeeded, suggesting the query is string-concatenated and unparameterised. Session cookie was set and honoured.

### 3. Download endpoint is `id`-based and not rate-limited
`/patient/download.php?id=1..3` serves PDFs with predictable, sequential IDs. Sequential IDs make it trivial to walk the whole report set.

### 4. PDFs use weak encryption
Two reports were protected with 128-bit RC4, revision 3, hashed with MD5. Wordlist cracking recovered both in seconds — one in a single guess.

### 5. `old/` contains a full SQL dump
`/old/mediloza_db_backup_2019.sql` (~6.3 KB) exposes a `shareholders` table and — separately — a full staff table with **National ID numbers and salaries in cleartext**.

### 6. SQL dump lists shareholders and share classes
`INSERT INTO shareholders` includes names, share counts, and `Ordinary` / `Preferential` classifications.

### 7. Staff table exposes PII and payroll
24 rows: name, job title, department, mobile number, National ID, annual salary, join date. Sorted descending by salary, the Medical Director and CFO top the list.

---

## Evidence

**robots.txt**
