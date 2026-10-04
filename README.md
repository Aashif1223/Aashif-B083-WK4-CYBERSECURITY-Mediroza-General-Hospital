# Mediroza General Hospital — Penetration Test Report

**Author:** Aashif Rahman
**Program:** Networkwalks Cybersecurity Internship — Batch B082, Week 4
**Target:** `medirozahospital.com` (lab/training environment)
**Status:** Educational project — authorized lab target only

> This repository documents a penetration test performed against a **training lab environment** provided by Networkwalks. No real systems, patients, or data are involved. All techniques here were practiced with explicit permission in a controlled lab and must never be used against systems without written authorization.

## Summary

The engagement moved through recon, authentication bypass, file cracking, metadata analysis, and exposed-backup discovery, chaining several low/medium findings into full compromise of a (simulated) sensitive database.

| # | Vulnerability | Location | Risk |
|---|---|---|---|
| 1 | Username enumeration on login page | `patient/login.php` | Medium |
| 2 | SQL injection login bypass | `patient/login.php` | Critical |
| 3 | Encrypted PDFs accessible after login bypass | `patient/reports/` | High |
| 4 | Weak PDF passwords crackable via wordlist | `patient_report_*.pdf` | High |
| 5 | Sensitive metadata left in patient PDFs | `patient_report_3.pdf` | Medium |
| 6 | Forgotten backup folder, directory listing enabled | `old/` | Critical |
| 7 | Confidential data in plaintext backup | `old/mediroza_db_backup_2019.sql` | Critical |

## Repo structure

```
mediroza-pentest-report/
├── README.md
├── report/
│   └── full-report.md        # Detailed writeup of all milestones
└── screenshots/               # Evidence captures (add your images here)
```

## Methodology (high level)

1. **Recon** — checked `robots.txt`, found disallowed `/patient/` and `/old/` paths.
2. **Auth testing** — found username enumeration, then SQL injection, used to bypass login.
3. **File handling** — downloaded protected PDFs, cracked weak passwords via wordlist attack.
4. **Metadata analysis** — extracted hidden author/comment fields pointing to a backup location.
5. **Exposure discovery** — found directory listing enabled on `/old/`, downloaded exposed DB backup.
6. **Reporting** — documented findings with risk ratings and remediation steps.

See [`report/full-report.md`](report/full-report.md) for the full step-by-step writeup.

## Remediation recommendations

- Return identical error messages for invalid username vs. invalid password.
- Use parameterized queries / prepared statements everywhere.
- Store files outside the web root; enforce per-file access control and strong passwords.
- Strip metadata from distributed files (`exiftool -all=`).
- Disable directory listing; never leave backups in publicly accessible web folders.

## Disclaimer

This work was produced for educational purposes only as part of an authorized training exercise. The target environment belongs to Networkwalks and was explicitly set up for this course. These techniques must never be applied to any system without explicit written permission from its owner.
