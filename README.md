# Enterprise-Active-Directory-Recovery-Resilience-Engineering-Lab
Enterprise AD lab simulating real enterprise scenarios: System State backup and recovery, authoritative object restore via ntdsutil, FSMO transfer and seizure, AD Recycle Bin, PSO configuration, inter-site replication diagnostics, offline database defragmentation, and advanced GPO modelling across a dual-DC Windows Server 2022 domain.



# 🏢 On-Premises Active Directory Lab
### Dual-Site Windows Server 2022 | Backup, Recovery, FSMO & M365 Integration | AZ-800 Aligned

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

![AD Sites and Services Overview](screenshots/01-sites-services-overview.png)

---

## What I Covered

### Phase 1 — Windows Server Backup

Installed Windows Server Backup on both DCs via Server Manager, ran a manual System State backup on DC01, and configured a scheduled daily backup at 11 PM. The System State captures everything AD needs: NTDS.dit, SYSVOL, registry, and boot files. Key constraint learned: the backup destination cannot be on the same volume as the OS.

![WSB Backup Success](screenshots/03-wsb-backup-success.png)
![WSB Schedule](screenshots/04-wsb-schedule.png)

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

![ADAC Deleted Objects](screenshots/10-adac-deleted-objects.png)
![Helene Mercier Restored](screenshots/11-helene-mercier-restored.png)

---

### Phase 5 — Fine-Grained Password Policies

Created two Password Settings Objects through ADAC's Password Settings Container. **PSO-IT-Management-HighSecurity** (Precedence 10): 16-character minimum, lockout after 3 attempts — applied to IT and Management groups. **PSO-StandardStaff** (Precedence 20): 12-character minimum, lockout after 5 attempts — applied to Domain Users. Lower precedence number wins when a user is covered by multiple PSOs. Verified correct application per user using ADAC's View resultant password settings.

![PSO Container](screenshots/12-pso-container.png)
![Resultant PSO Pierre Dubois](screenshots/13-pso-resultant-pierre.png)
