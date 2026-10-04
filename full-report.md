# Full Report — Mediroza General Hospital Pentest

**Author:** Aashif Rahman
**Batch:** B082 | Week 4 | Networkwalks Cybersecurity Internship
**Target:** `https://medirozahospital.com` (authorized lab target)

---

## Milestone 1 — Initial Access

### Step 1: Recon
Checked `robots.txt` for disallowed paths (a common first recon step, since these often reveal folders not meant for public indexing):

```
curl https://medirozahospital.com/robots.txt
```

Output showed:
```
Disallow: /patient/
Disallow: /staff/
Disallow: /old/
```

`/patient/` became the focus for this milestone; `/old/` became relevant later in Milestone 3.

![robots.txt recon](../screenshots/08-robots-txt-recon-terminal.png)

### Step 2: Locate the login page
`https://medirozahospital.com/patient/login.php`

![Login page - incorrect password](../screenshots/01-login-incorrect-password.png)

### Step 3: Username enumeration
Testing a fake username (`bob`) returned *"Username not found"*, while testing `admin` returned *"Incorrect password"*. Two different error messages for invalid username vs. invalid password confirmed the account `admin` exists — a username enumeration vulnerability.

### Step 4: SQL injection test
Entering a single quote in the username field (`admin'`) triggered a raw MySQL syntax error, confirming the input was being placed directly into a query without sanitization.

![SQL injection error](../screenshots/03-sql-injection-error.png)

### Step 5: Login bypass
Using the classic payload `admin' --` as the username (with any password) bypassed authentication by commenting out the password check in the underlying query. Result: logged into the patient portal, which listed 3 downloadable reports.

![Patient portal - reports list](../screenshots/02-patient-portal-reports-list.png)

### Step 6: Download reports
- `patient_report_1.pdf`
- `patient_report_2.pdf`
- `patient_report_3.pdf`

---

## Milestone 2 — Crack the Encryption

### Step 1: Extract PDF hashes
Each PDF was hashed (`$pdf$...` format) using a hash calculator tool to prepare for an offline dictionary attack.

### Step 2: Crack reports 1 & 2
Using a built-in 100-password common wordlist against each hash:
- `patient_report_1.pdf` → `123456`
- `patient_report_2.pdf` → `password`

### Step 3: Crack report 3
The built-in wordlist failed against report 3. Switching to a larger list (John the Ripper's wordlist) succeeded:
- `patient_report_3.pdf` → `!@#$%^&`

![Password cracker dictionary attack](../screenshots/06-password-cracker-dictionary-attack.png)

**Lesson:** when a small wordlist fails, it doesn't mean the target is secure — it means you need a bigger list. Real attackers maintain multiple wordlists for this exact reason.

### Step 4: Decrypt a working copy
```
qpdf --password='!@#$%^&' --decrypt patient_report_3.pdf report3_open.pdf
```
An unlocked copy was needed so metadata tools could read all fields (encrypted PDFs typically only expose an "Encryption" field).

---

## Milestone 3 — Deep Reconnaissance

### Step 1: Metadata analysis
```
exiftool report3_open.pdf
```
Key fields found:
```
Author   : j.malik
Comments : DB backup moved to /old before site migration, do not delete
```

### Step 2: Cross-reference with recon
The `/old` path had already been flagged in `robots.txt` during initial recon (Milestone 1, Step 1) — a good example of how early recon clues connect to later findings.

### Step 3: Exposed directory
```
https://medirozahospital.com/old/
```
Directory listing was enabled (misconfiguration), exposing:
```
mediroza_db_backup_2019.sql
```
```
wget https://medirozahospital.com/old/mediroza_db_backup_2019.sql
```

### Step 4: Parse the backup
The `.sql` backup contained `INSERT INTO` statements for `staff` and `shareholders` tables — converted into readable tables for the report (name/title/department/salary; shareholder/share %/class).

### Step 5: Trace the chain
`j.malik` (Author field in the PDF) matched `Jameel Malik`, IT Systems Administrator, in the staff table recovered from the backup — confirming he was the one who moved the backup and left the note, closing the loop between the metadata clue and the exposed data.

---

## Milestone 4 — Findings & Remediation

### Findings table

| # | Vulnerability | Location | Risk |
|---|---|---|---|
| 1 | Username enumeration | `patient/login.php` | Medium |
| 2 | SQL injection login bypass | `patient/login.php` | Critical |
| 3 | Encrypted PDFs accessible post-bypass | `patient/reports/` | High |
| 4 | Weak PDF passwords | `patient_report_*.pdf` | High |
| 5 | Sensitive metadata in PDFs | `patient_report_3.pdf` | Medium |
| 6 | Directory listing on backup folder | `old/` | Critical |
| 7 | Plaintext confidential data in backup | `old/mediroza_db_backup_2019.sql` | Critical |

### Recommendations

- **Username enumeration:** return identical error messages regardless of which field (username/password) was wrong.
- **SQL injection:** use parameterized queries / prepared statements; never concatenate raw user input into SQL.
- **PDF access control:** store files outside the web root, enforce access control, require strong unique passwords.
- **Metadata hygiene:** strip metadata before distribution (`exiftool -all= filename.pdf`).
- **Directory listing & backups:** disable directory listing server-wide; never leave DB backups in a publicly accessible web folder.

---

## Appendix — Opened Reports (Evidence)

![Report 1 - opened](../screenshots/04-report1-pathology-dlamini.png)

![Report 2 - opened](../screenshots/05-report2-pathology-reddy.png)

![Report 3 - opened](../screenshots/07-report3-pathology-thompson.png)

> ⚠️ These screenshots contain simulated patient data (lab-generated test fixtures) used only for this training exercise. Consider blurring or redacting names/values before pushing this repo publicly, since GitHub is a public platform.

---

*This report was produced for educational purposes as part of an authorized training exercise. The target environment belongs to Networkwalks and was set up specifically for this course. These techniques must never be applied to any system without explicit written permission from the owner.*
