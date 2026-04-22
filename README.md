# 📘 CRTP Study Notes — Certified Red Team Professional

> Personal study repository for the **Certified Red Team Professional (CRTP)** certification by [Altered Security](https://www.alteredsecurity.com/).  
> Contains full course notes, topic summaries, cheatsheets, and practice lab notes covering Active Directory attacks and red team tradecraft.

---

## ⚠️ Disclaimer

These notes are intended **solely for educational purposes** and ethical security research.  
All techniques documented here should only be applied in **authorized environments** (lab, CTF, or explicitly scoped engagements).  
The author takes no responsibility for misuse of any content in this repository.

---

## 📁 Contents

### 🗒️ Course Notes

Full notes taken per module, covering theory, context, and practical command usage.

|#|File|Topic|
|---|---|---|
|01|`1 - Active Directory.md`|AD fundamentals and structure|
|02|`2 - PowerShell.md`|PowerShell for red teaming|
|03|`3 - Offensive .NET - Tradecraft.md`|.NET-based offensive tooling and OPSEC|
|04|`4 - Domain Enumeration.md`|Core domain enumeration techniques|
|05|`5 - Domain Enumeration - ACLs.md`|ACL enumeration and abuse paths|
|06|`6 - Domain Enumeration - Group Policy.md`|GPO enumeration and attack surface|
|07|`7 - Domain Enumeration - Trusts.md`|Trust relationship enumeration|
|08|`8 - Domain Enumeration - User Hunting.md`|Locating high-value users across the domain|
|09|`9 - Privilege Escalation.md`|Local privilege escalation techniques|
|10|`10 - Lateral Movement.md`|Lateral movement methods and tooling|
|11|`11 - AD Domain Dominance & Kerberos.md`|Domain dominance and Kerberos attack paths|
|12|`12 - Persistence.md`|Persistence mechanisms in AD environments|
|13|`13 - PrivEsc.md`|Advanced privilege escalation techniques|
|14|`14 - Trust Abuse.md`|Inter-domain and forest trust abuse|
|15|`15 - Introduction to EDR.md`|EDR fundamentals and evasion concepts|

---

### 📋 Cheatsheets

Quick-reference command sheets for use during labs and engagements.

|File|Description|
|---|---|
|`PowerShell Cheat Sheet.md`|Offensive PowerShell one-liners and references|

---

### 🧪 Practice Lab Notes

|File|Description|
|---|---|
|`Practice Lab Notes.md`|Full hands-on lab walkthrough notes (27 KB)|

---

### 📂 Additional Folders

|Folder|Description|
|---|---|
|`Objectives/`|Course learning objectives per module|
|`images/`|Supporting screenshots and diagrams|
|`Exam/`|🔒 Excluded — not tracked by Git|

---

## 🧠 Study Approach

These notes are organized to support a **reference-first, understand-second** methodology:

1. **Course Notes** — Taken during each lab module; include context, purpose, and tool syntax
2. **Cheatsheets** — Pure command references, ready to copy-paste during labs or engagements
3. **Practice Lab Notes** — Continuous running notes from hands-on lab work

---

## 🛠️ Tools & Frameworks Referenced

|Tool|Purpose|
|---|---|
|`PowerView`|AD enumeration via PowerShell|
|`BloodHound` / `SharpHound`|Graph-based AD attack path analysis|
|`Mimikatz`|Credential extraction and ticket abuse|
|`Rubeus`|Kerberos ticket operations|
|`BloodyAD`|ACL-based AD manipulation|
|`Impacket`|Python-based AD/SMB/Kerberos tooling|
|`NetExec` (nxc)|Network-wide authentication testing|
|`PowerShell Remoting`|Lateral movement and command execution|

---

## 🎯 Certification Info

|Field|Detail|
|---|---|
|**Certification**|Certified Red Team Professional (CRTP)|
|**Provider**|Altered Security|
|**Focus Area**|Active Directory Red Teaming|
|**Lab Environment**|Fully interactive Windows AD lab|
|**Exam Format**|24-hour hands-on practical|
|**Official Site**|[alteredsecurity.com](https://www.alteredsecurity.com/)|

---

## 📌 Related Certifications & Resources

- [CRTE — Certified Red Team Expert](https://www.alteredsecurity.com/redteamexpert)
- [CRTO — Certified Red Team Operator (Zero-Point Security)](https://training.zeropointsecurity.co.uk/courses/red-team-ops)
- [HTB CPTS — Certified Penetration Testing Specialist](https://academy.hackthebox.com/preview/certifications/htb-certified-penetration-testing-specialist)
- [BloodHound Docs](https://bloodhound.readthedocs.io/)
- [ired.team](https://www.ired.team/) — Excellent AD attack reference

---

## 👤 Author

**Omar**  
Penetration Tester | Aspiring Red Team Operator

> _"Understand the why, then own the how."_

---

## 📄 License

This repository is for **personal educational use only**.  
No license is granted for commercial use or redistribution.  
Content references publicly available research and course material — all credits to their respective authors.