# 🏢 Enterprise Active Directory Recovery & Resilience Engineering Lab
### Dual-Site Windows Server 2022 | Backup, Recovery, FSMO & M365 Integration | AZ-800 Aligned

<p align="center">
  <img src="https://img.shields.io/badge/Windows%20Server-2022-blue?style=for-the-badge&logo=microsoft&logoColor=white" alt="Windows Server 2022">
  <img src="https://img.shields.io/badge/Microsoft%20Entra-Connected-0078D4?style=for-the-badge&logo=microsoft&logoColor=white" alt="Entra ID">
  <img src="https://img.shields.io/badge/AZ--800%20%7C%20801-Aligned-purple?style=for-the-badge" alt="AZ-800 Aligned">
  <img src="https://img.shields.io/badge/SC--300-Identity-green?style=for-the-badge" alt="SC-300">
</p>

---

## 📌 Executive Summary
An enterprise-grade, dual-site hybrid Active Directory infrastructure simulating real-world disaster recovery, resilience engineering, and identity lifecycle management for a fictional organization, **Northwind Data Systems** (`northwinddata.com`). 

This project bridges the gap between on-premises system administration and modern cloud identity, moving beyond standard deployment into hands-on failure state resolution, database optimization, and cross-platform synchronization.

---

## ⚙️ Lab Architecture & Specifications

<div align="center">

| Component | Technical Specification |
| :--- | :--- |
| **Domain Name** | `northwinddata.com` |
| **DC01 (London HQ)** | Windows Server 2022 — Core Operations Master |
| **DC02 (Paris Branch)** | Windows Server 2022 — Regional Replica |
| **AD Sites Topology** | `London-HQ` / `Paris-Branch` via `London-Paris-Link` (15-min replication) |
| **User Directory** | 40 Multi-Departmental Staff Accounts (20 UK / 20 FR) with full OAB schemas |
| **Hybrid Bridge** | Microsoft 365 / Azure Entra ID Integration via Microsoft Entra Connect |
| **Management Stack** | ADUC, ADAC, GPMC, AD Sites & Services, WSB, `ntdsutil`, `repadmin` |

</div>

<p align="center">
  <img width="49%" alt="Sites server" src="https://github.com/user-attachments/assets/938d3185-9b85-405b-a4bb-ce406ad20034" />
  <img width="49%" alt="Computer and users" src="https://github.com/user-attachments/assets/9d2eeed7-eb5c-408a-86f4-32db568f1b98" />
</p>

---

## 🛠️ Core Engineering Phases

<details>
<summary><b>Phase 1 — Bare-Metal & System State Backup Engineering</b></summary>
<br>

Implemented baseline backup architecture utilizing **Windows Server Backup (WSB)** across both production domain controllers. 
* Designed and scheduled daily incremental System State captures at 23:00 to back up critical stateful data (`NTDS.dit`, `SYSVOL`, registry, boot configurations).
* **Key Constraint Resolved:** Enforced architectural separation ensuring backup targets reside on completely independent physical/logical volumes separate from the operating system volume to prevent single-point-of-failure storage degradation.

<p align="center">
  <img width="49%" alt="WSB DC1" src="https://github.com/user-attachments/assets/0c4cc2a5-2a2e-4f73-9077-b28ee27c67d4" />
  <img width="49%" alt="WSB Backup date" src="https://github.com/user-attachments/assets/029e746c-9646-43d0-9a3f-99baba0c34cb" />
</p>
</details>

<details>
<summary><b>Phase 2 — Non-Authoritative Directory Database Restoration</b></summary>
<br>

Simulated local operating system recovery scenarios without impacting the wider multi-master replication topology.
* Intercepted standard boot sequences to initialize **Directory Services Restore Mode (DSRM)** via `msconfig`.
* Executed local System State rollbacks utilizing the WSB subsystem.
* Post-initialization, verified safe catch-up replication where **DC02** automatically updated **DC01** back to production head via outbound multi-master delta synchronization.
* **Verification:** Logged healthy state utilizing `repadmin /replsummary` and parsed Directory Service Event Logs for Event IDs `1394` and `1168`.

<p align="center">
  <img width="80%" alt="repadmin " src="https://github.com/user-attachments/assets/71e2e1b0-f49e-4f32-9811-d408d040504e" />
</p>
</details>

<details>
<summary><b>Phase 3 — Authoritative Object & OU Disaster Recovery</b></summary>
<br>

Simulated a catastrophic structural deletion where the entire `FR_Sales` Organizational Unit (OU) and its underlying security principals were completely purged and replicated forest-wide.
* Isolated the target DC within **DSRM** environments to perform base recovery.
* Utilized the `ntdsutil` command-line utility to target the specific distinguished name (DN) of the deleted container, executing an **Authoritative Restore**.
* This explicitly increments the USN (Update Sequence Number) attributes of the selected objects past the current epoch of all other domain controllers.
* Forced forest-wide replication outbound upon normal reboot, ensuring the restored objects successfully overrode the deletion states on remaining replica partners.

```cmd
ntdsutil
authoritative restore
restore object "OU=FR_Sales,DC=northwinddata,DC=com"
