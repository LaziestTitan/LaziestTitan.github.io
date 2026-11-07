---
title: "404 Bank: Internal Penetration Test & Domain Compromise"
date: 2026-11-07
draft: false
tags: ["Active Directory", "AD CS", "Kerberoasting", "Internal Pentest"]
summary: "Internal penetration test simulating an assumed-breach scenario resulting in full Active Directory domain compromise via AD CS ESC4."
---

### Engagement Summary
Assumed-breach assessment against an internal financial network (`404finance.local`). Evaluated Active Directory security controls, internal service isolation, and certificate infrastructure misconfigurations.

* **Key Attack Vectors:** Reverse-engineered hardcoded credentials in internal application, Active Directory ACL permission bloat (chained account takeover), and AD CS ESC4 certificate template escalation.
* **Outcome:** Achieved full Domain Administrator compromise via PKINIT certificate theft.

📄 **[Download / View Full Client Deliverable PDF](/reports/404-bank-report.pdf)**