# 🏢 Enterprise-Active-Directory-Recovery-Resilience-Engineering-Lab

### Dual-Site Windows Server 2022 | Backup, Recovery, FSMO & M365 Integration | AZ-800 Aligned
Enterprise AD lab simulating real enterprise scenarios: System State backup and recovery, authoritative object restore via ntdsutil, FSMO transfer and seizure, AD Recycle Bin, PSO configuration, inter-site replication diagnostics, offline database defragmentation, and advanced GPO modelling across a dual-DC Windows Server 2022 domain.




---

## What I Built

I built a production-style on-premises Active Directory environment for a fictional organisation, **Northwind Data Systems** (`northwinddata.com`) — a full dual-site setup with domain controllers in London and Paris, 40 users, AD Sites and Services, and a complete connection to Microsoft 365 via Entra Connect.

This covers what real sysadmin work actually looks like: backing up and restoring a DC, recovering deleted objects, managing FSMO roles, monitoring replication between sites, and troubleshooting Group Policy. Every phase was hands-on — I broke things and fixed them.

---

## Lab Environment

| Component | Detail |
|---|---|
| Domain | northwinddata.com |
| DC01 | Windows Server 2022 — London HQ |
| DC02 | Windows Server 2022 — Paris Branch |
| AD Sites | London-HQ / Paris-Branch |
| Site Link | London-Paris-Link — 15-minute replication interval |
| Users | 20 UK staff + 20 France staff |
| Cloud | Microsoft 365 / Entra ID via Entra Connect |
| Tools | ADUC, ADAC, GPMC, AD Sites & Services, WSB, ntdsutil, repadmin |

---

<img width="1021" height="728" alt="Sites server" src="https://github.com/user-attachments/assets/938d3185-9b85-405b-a4bb-ce406ad20034" />
<img width="1025" height="726" alt="Computer and users" src="https://github.com/user-attachments/assets/9d2eeed7-eb5c-408a-86f4-32db568f1b98" />


---

## What I Covered

### Phase 1 — Windows Server Backup

Installed Windows Server Backup on both DCs via Server Manager, ran a manual System State backup on DC01, and configured a scheduled daily backup at 11 PM. The System State captures everything AD needs: NTDS.dit, SYSVOL, registry, and boot files. Key constraint learned: the backup destination cannot be on the same volume as the OS.

<img width="1024" height="724" alt="WSB DC1" src="https://github.com/user-attachments/assets/0c4cc2a5-2a2e-4f73-9077-b28ee27c67d4" />
<img width="1022" height="729" alt="WSB Backup date" src="https://github.com/user-attachments/assets/029e746c-9646-43d0-9a3f-99baba0c34cb" />


---

### Phase 2 — Non-Authoritative Restore

Booted DC01 into Directory Services Restore Mode via `msconfig`, restored the System State from backup using WSB, then rebooted normally. DC02 automatically replicated current data back to DC01, bringing it fully up to date. Verified with `repadmin /replsummary` and Event IDs 1394 and 1168 in the Directory Service log.

![msconfig DSRM](screenshots/05-msconfig-dsrm.png)
![WSB Recovery Wizard](screenshots/06-wsb-recovery-wizard.png)
![repadmin Output](screenshots/07-repadmin-replsummary.png)

---

### Phase 3 — Authoritative Restore

Simulated a deleted OU disaster — removed `FR_Sales` and all 5 users, forced replication to DC02 so both DCs confirmed the deletion. Recovery is two steps: restore from backup in DSRM, then run `ntdsutil` to mark restored objects as authoritative before rebooting. This stamps them with a version number higher than DC02's, so they replicate outward instead of getting overwritten. Verified FR_Sales and all users were back on both DCs after reboot.

![ntdsutil Authoritative Restore](screenshots/08-ntdsutil-auth-restore.png)
![FR_Sales Restored on DC02](screenshots/09-dc02-fr-sales-restored.png)

---

### Phase 4 — AD Recycle Bin

Enabled the AD Recycle Bin through ADAC — the modern way to recover deleted objects without touching a backup or DSRM. Deleted Helene Mercier (France Country Manager) and restored her in seconds from the Deleted Objects container with all attributes intact. Also recovered the full FR_Sales OU. Critical order: restore the parent OU before restoring the objects inside it, or users land in LostAndFound.

<img width="1026" height="727" alt="Delete User Helene" src="https://github.com/user-attachments/assets/f4f9d2dc-a36e-4f68-b494-8efbaa6eaa1a" />
<img width="1022" height="727" alt="Restore User Helene" src="https://github.com/user-attachments/assets/4eacad45-aed9-4f6a-a0bd-fd652d39ce1c" />

---

### Phase 5 — Fine-Grained Password Policies

Created two Password Settings Objects through ADAC's Password Settings Container. **PSO-IT-Management-HighSecurity** (Precedence 10): 16-character minimum, lockout after 3 attempts — applied to IT and Management groups. **PSO-StandardStaff** (Precedence 20): 12-character minimum, lockout after 5 attempts — applied to Domain Users. Lower precedence number wins when a user is covered by multiple PSOs. Verified correct application per user using ADAC's View resultant password settings.

<img width="1023" height="724" alt="PSO Strong password IT" src="https://github.com/user-attachments/assets/5c714a26-24e2-46a3-99e3-b90d03b8faef" />


---

### Phase 6 — FSMO Roles

Verified all five FSMO role holders across three tools: ADUC Operations Masters, Active Directory Domains and Trusts, and the Schema MMC snap-in. Transferred the PDC Emulator and Domain Naming Master to DC02 to simulate a maintenance window, then transferred both back. Understood the key distinction — Transfer (graceful, both DCs live) versus Seize (emergency only; if you seize, the old DC must never come back online).

![PDC Emulator Transfer](screenshots/14-fsmo-pdc-transfer.png)
![Domain Naming Master](screenshots/15-fsmo-domain-naming.png)

---

### Phase 7 — Replication Monitoring

Forced replication between sites using AD Sites and Services (Replicate Now on the NTDS Settings connection object). Ran `repadmin /replsummary`, `/showrepl`, and `/syncall /Ade` to verify health and push updates across all partitions. Filtered the Directory Service event log for replication Event IDs — 1394 (healthy), 1311 (topology error), 2042 (tombstone lifetime exceeded).

![repadmin Output](screenshots/16-repadmin-output.png)
![Directory Service Log](screenshots/17-event-viewer-directory-service.png)

---

### Phase 8 — AD Database Maintenance

Booted into DSRM and ran an integrity check on NTDS.dit via `ntdsutil → files → integrity`. Followed with an offline defragmentation using `compact to C:\NTDS-Compact` to physically shrink the database file — online defrag reorganises data but doesn't reduce file size. Replaced the original file, removed superseded transaction logs, rebooted, and confirmed replication resumed cleanly.

![NTDS Folder](screenshots/18-ntds-folder.png)
![ntdsutil Compact](screenshots/19-ntdsutil-compact.png)

---

### Phase 9 — Group Policy Advanced

Used the **GPO Results Wizard** to identify exactly which policy was blocking Control Panel for a France user — pinpointed to FR_UserPolicy in seconds. Used the **GPO Modeling Wizard** to simulate a new USB restriction policy before deployment. Practised Link Order control and configured Block Inheritance on FR_Management so restrictive France_Staff policies didn't affect managers. Used Enforced to override inheritance blocks for security-critical GPOs.

![GPO Results](screenshots/20-gpo-results-pierre.png)
![GPO Link Order](screenshots/21-gpo-link-order.png)
![Block Inheritance](screenshots/22-block-inheritance.png)

---

### France Branch Office

Renamed the default site to London-HQ, created Paris-Branch, defined subnets for both, set up the London-Paris-Link at a 15-minute interval, and moved DC02 into Paris-Branch. Created 20 France users across 5 departments with full Organisation tab attributes set — these sync to Microsoft 365 and power dynamic group rules in Entra ID. Ran the Delegation of Control Wizard to give the France IT Security Analyst rights over France users only, with no access to UK accounts. Linked FR_PasswordPolicy at domain root and FR_UserPolicy to the France_Staff OU.

![AD Sites France Setup](screenshots/01-sites-services-overview.png)
![France OU Structure](screenshots/02-aduc-ou-tree.png)

---

### Microsoft 365 & Entra ID Integration

Updated Entra Connect OU filtering to include France_Staff, triggered a manual sync, and all 20 France users appeared in the Microsoft 365 Admin Centre with on-prem attributes intact. Configured Address Book Policies in Exchange Online so France and UK users see only their own colleagues in Outlook and Teams. Assigned licences and confirmed full access to Teams, Outlook, SharePoint, and OneDrive — all signing in with `@northwinddata.com`.

![M365 France Users](screenshots/23-m365-france-users.png)
![Entra Connect Sync](screenshots/24-entra-connect-sync.png)

---

## Troubleshooting Scenarios Worked Through

- WSB backup failing with 0x80070001 — destination volume permissions; fixed by running WSB as Administrator
- DSRM login rejected — missing `.\` prefix; DSRM requires a local account login, not a domain account
- Authoritative restore not sticking — rebooted before running ntdsutil; DC02's deletion replicated back and wiped the restore; repeated in correct order
- PSO not applying — PSO was linked to an OU, not a security group; PSOs only work on group and user objects
- DC02 unresponsive after site move — site link not configured with both sites; fixed in DEFAULTIPSITELINK properties
- GPO not applying despite showing in GPMC — Authenticated Users removed from Security Filtering; re-added via Delegation tab
- Recycle Bin users landing in LostAndFound — restored users before restoring their parent OU; always restore the container first
<img width="1021" height="593" alt="CMD replication troubleshooting" src="https://github.com/user-attachments/assets/5b166d1b-0dab-48f8-9124-8595748365b7" />
<img width="1015" height="727" alt="replication resolved screenshot" src="https://github.com/user-attachments/assets/5ad5c7da-302b-49c4-9e8b-039e1d28e94d" />

---

## Key Skills Demonstrated

`Active Directory` · `Windows Server 2022` · `ADUC` · `ADAC` · `AD Sites & Services` · `Windows Server Backup` · `System State` · `DSRM` · `ntdsutil` · `Authoritative Restore` · `AD Recycle Bin` · `Fine-Grained Password Policies` · `FSMO Roles` · `repadmin` · `NTDS.dit` · `GPMC` · `Group Policy Results` · `Group Policy Modeling` · `Entra Connect` · `Microsoft 365` · `Exchange Online` · `Hybrid Identity` · `AZ-800` · `AZ-801`

---

## Certification Alignment

- **AZ-800** — Administering Windows Server Hybrid Core Infrastructure
- **AZ-801** — Configuring Windows Server Hybrid Advanced Services
- **SC-300** — Microsoft Identity and Access Administrator
- **AZ-104** — Microsoft Azure Administrator (hybrid identity sections)

---

*Built as part of my transition into IT infrastructure and cloud identity engineering. More lab projects at [github.com/elvisodihiri](https://github.com/elvisodihiri)*
