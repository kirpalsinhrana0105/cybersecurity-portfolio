# MobSF Static Analysis — DIVA (Damn Insecure and Vulnerable App)

![Security Score](https://img.shields.io/badge/Security%20Score-36%2F100-red)
![Trackers](https://img.shields.io/badge/Trackers-0%2F432-brightgreen)
![Classification](https://img.shields.io/badge/Classification-Internal%20%2F%20Confidential-lightgrey)

Static security assessment of the Android application **DivaApplication.apk** (`jakhar.aseem.diva`), performed with [Mobile Security Framework (MobSF)](https://github.com/MobSF/Mobile-Security-Framework-MobSF).

> **DIVA** (Damn Insecure and Vulnerable App) is an intentionally vulnerable Android training app. The findings below are **by design** — this repo doubles as a worked example of how to structure a MobSF static scan into a formal report.

---

## Executive Summary

The scan produced an overall MobSF **Security Score of 36/100**, reflecting a high number of high- and warning-severity issues across the manifest, code, certificate signing, and native shared libraries. No third-party trackers were detected (0/432).

**Key risk areas:**

- Signed with a debug certificate and vulnerable to the **Janus vulnerability** (v1-only signature scheme)
- `android:debuggable=true` and `android:allowBackup=true` in a production-style build
- Multiple exported `Activities` and a `ContentProvider` accessible to other apps without protection
- Raw SQL queries → potential **SQL Injection**
- Native libraries (arm64, mips64, x86_64) missing **NX**, **stack canary**, and **FORTIFY** hardening
- `minSdkVersion=15` — no longer receives security patches

## Application Information

| Property | Value |
|---|---|
| File Name | `DivaApplication.apk` |
| Package Name | `jakhar.aseem.diva` |
| Main Activity | `jakhar.aseem.diva.MainActivity` |
| File Size | 1.43 MB |
| Target / Min / Max SDK | 23 / 15 / — |
| MD5 | `82ab8b2193b3cfb1c737e3a786be363a` |
| SHA1 | `27e849d9d7b86a3a3357fb3e980433a91d416801` |
| SHA256 | `5cefc51fce9bd760b92ab2340477f4dda84b4ae0c5d04a8c9493e4fe34fab7c5` |
| Security Score | 36 / 100 |
| Trackers Detected | 0 / 432 |

**Exported components:** 2 of 17 Activities · 0 of 0 Services · 0 of 0 Receivers · **1 of 1 Providers (exported)**

## Certificate & Signing

| Attribute | Value |
|---|---|
| Signature Schemes | v1: ✅ &nbsp;\| v2: ❌ &nbsp;\| v3: ❌ &nbsp;\| v4: ❌ |
| Subject / Issuer | `C=US, O=Android, CN=Android Debug` |
| Signature Algorithm | `rsassa_pkcs1v15` |
| Validity | 2015-11-02 → 2045-10-25 |
| Serial Number | `0x218330df` |

| Finding | Severity |
|---|---|
| Signed with a debug certificate | 🔴 High |
| Vulnerable to Janus vulnerability (v1-only) | 🔴 High |
| Signed with a code signing certificate | ℹ️ Info |

## Permissions

| Permission | Status | Notes |
|---|---|---|
| `android.permission.INTERNET` | ℹ️ Info | Full network access |
| `android.permission.READ_EXTERNAL_STORAGE` | 🟠 Warning | Dangerous — reads external storage |
| `android.permission.WRITE_EXTERNAL_STORAGE` | 🟠 Warning | Dangerous — read/write/delete external storage |

## Manifest Analysis

| Issue | Severity |
|---|---|
| Installable on vulnerable, unpatched Android (`minSdk=15`) | 🔴 High |
| Debug enabled (`android:debuggable=true`) | 🔴 High |
| Data backup allowed (`android:allowBackup=true`) | 🟠 Warning |
| `APICredsActivity` not protected (exported) | 🟠 Warning |
| `APICreds2Activity` not protected (exported) | 🟠 Warning |
| `NotesProvider` not protected (exported) | 🟠 Warning |

## Code Analysis

| Issue | Severity | Standards | File(s) |
|---|---|---|---|
| Debug configuration enabled | 🔴 High | CWE-919, OWASP M1, MSTG-RESILIENCE-2 | `BuildConfig.java` |
| Raw SQL queries — SQL Injection risk | 🟠 Warning | CWE-89, OWASP M7 | `InsecureDataStorage2Activity.java`, `NotesProvider.java`, `SQLInjectionActivity.java` |
| External storage read/write | 🟠 Warning | CWE-276, OWASP M2, MSTG-STORAGE-2 | `InsecureDataStorage4Activity.java` |
| Sensitive data written to temp file | 🟠 Warning | CWE-276, OWASP M2, MSTG-STORAGE-2 | `InsecureDataStorage3Activity.java` |
| Sensitive data logged | ℹ️ Info | CWE-532, MSTG-STORAGE-3 | — |

## Native Library Hardening (`libdivajni.so`)

Checked across **arm64-v8a**, **mips64**, and **x86_64** — all three share identical results.

| Protection | Status | Notes |
|---|---|---|
| NX (non-executable stack) | 🔴 High | Not set |
| Stack Canary | 🔴 High | Not built with `-fstack-protector-all` |
| FORTIFY | 🟠 Warning | `-D_FORTIFY_SOURCE=2` not used |
| PIE | ℹ️ Info | Built as DSO with `-fPIC`; not applicable |
| RELRO | ℹ️ Info | Full RELRO enabled |
| RPATH / RUNPATH | ℹ️ Info | Neither set |
| Symbols | ℹ️ Info | Stripped |

## Network / Server Location

A single server location was identified in the **United States (west coast)**, with no communication to OFAC-sanctioned countries.

## Findings Summary

| Category | High | Warning | Info |
|---|---|---|---|
| Certificate Analysis | 2 | 0 | 1 |
| Manifest Analysis | 2 | 4 | 0 |
| Code Analysis | 1 | 3 | 1 |
| Shared Library Analysis (×3 libs) | 2 each | 1 each | 5 each |

## Recommendations

- [ ] Re-sign the release build with a proper release keystore (not the Android Debug cert); enable v2/v3 signature schemes to fix the Janus vulnerability
- [ ] Set `android:debuggable="false"` and `android:allowBackup="false"` in the release manifest
- [ ] Restrict `APICredsActivity`, `APICreds2Activity`, and `NotesProvider` — set `android:exported="false"` or add explicit permission protection
- [ ] Replace raw SQL string concatenation with parameterized queries / prepared statements
- [ ] Avoid writing sensitive data to external storage or temp files — use `EncryptedSharedPreferences` / Android Keystore
- [ ] Remove or gate verbose logging in production builds
- [ ] Raise `minSdkVersion` to a currently supported API level (29+)
- [ ] Rebuild native libraries with NX, stack-canary (`-fstack-protector-all`), and FORTIFY (`-D_FORTIFY_SOURCE=2`) enabled

## Scope & Limitations

This report is derived entirely from an automated MobSF static analysis pass — **no dynamic analysis, manual penetration testing, or business-logic review** was performed. DIVA is intentionally vulnerable for training purposes, so these findings are expected/by-design rather than defects in a production system.

---

*Report date: July 22, 2026 · Classification: Internal / Confidential · Tooling: [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF)*
