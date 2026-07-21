# Cybersecurity Portfolio

Hands-on security projects covering SOC monitoring (custom SIEM), mobile application security testing, web application penetration testing, and full exploitation walkthroughs — built and documented end-to-end.

**Author:** Kirpalsinh Rana
**Focus areas:** SOC Operations · SIEM Monitoring · Offensive Security (Web & Mobile) · OWASP Testing · Privilege Escalation

---

## About this repository

Each folder under `projects/` is a self-contained project with its own write-up: what was built or tested, the tools and methodology used, and what it demonstrates. These are hands-on labs and testing exercises, not passive tutorials.

## Projects

| # | Project | What it demonstrates | Stack |
|---|---|---|---|
| 1 | [NullByte: 1 — SIEM & Boot2Root Pentest](projects/01-nullbyte-siem-pentest) | Full exploitation chain: recon → steganography → brute force → SQLi → hash cracking → root privilege escalation | Nmap, Nikto, Gobuster, Hydra, sqlmap, SUID/PATH hijacking |
| 2 | [MobSF — Mobile Application Security Testing](projects/02-mobsf-mobile-testing) | Static/dynamic analysis of Android APKs — insecure storage, hardcoded secrets, permission misuse | MobSF |
| 3 | [OWASP Web App Pentesting — bWAPP, DVWA, Mutillidae](projects/03-owasp-web-pentest) | Vulnerability assessment across intentionally vulnerable apps — SQLi, XSS, broken authentication | bWAPP, DVWA, Mutillidae |
| 4 | [TryHackMe — Guided Web Security Labs](projects/04-tryhackme-labs) | Structured, mentor-guided labs on OWASP Top 10 exploitation techniques | TryHackMe |
| 5 | [WordPress Penetration Testing](projects/05-wordpress-pentest) | End-to-end exploitation: payload delivery, reverse shell via Netcat, credential extraction, hash cracking with Hashcat | Netcat, Hashcat, Python HTTP server |

*(SIEM NULLBYTE — the custom SIEM tool project — will be added here as a separate entry once documented.)*

## Skills demonstrated across these projects

- **SOC Operations:** Log monitoring, alerting, custom SIEM tooling
- **Offensive Security:** Web application penetration testing, OWASP Top 10 exploitation, SQL injection
- **Mobile Security:** Static & dynamic Android APK analysis
- **Exploitation Techniques:** Steganography extraction, brute forcing, hash cracking, reverse shells, SUID/PATH privilege escalation
- **Tools:** Nmap, Nikto, Gobuster, Hydra, sqlmap, MobSF, Netcat, Hashcat

---

*All testing was performed against intentionally vulnerable, legally provided lab environments (VulnHub, OWASP practice apps, TryHackMe, self-hosted WordPress instance) for educational purposes.*
