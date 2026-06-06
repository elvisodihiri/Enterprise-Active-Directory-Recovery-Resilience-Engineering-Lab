<div align="center">

<img src="https://img.shields.io/badge/Windows%20Server-2022-0078D4?style=for-the-badge&logo=windows&logoColor=white"/>
<img src="https://img.shields.io/badge/Active%20Directory-Enterprise%20Lab-0078D4?style=for-the-badge&logo=microsoft&logoColor=white"/>
<img src="https://img.shields.io/badge/Microsoft%20365-Integrated-D83B01?style=for-the-badge&logo=microsoft-365&logoColor=white"/>

<br/><br/>

# Enterprise Active Directory Recovery & Resilience Engineering Lab

### Dual-Site Windows Server 2022 · Backup & Recovery · FSMO Management · M365 Integration

<br/>

> Production-style on-premises Active Directory environment for **Northwind Data Systems** (`northwinddata.com`) —
> a dual-site domain with controllers in London and Paris, 40 provisioned users, AD Sites and Services,
> and a full hybrid connection to Microsoft 365 via Entra Connect.

<br/>

![Phases](https://img.shields.io/badge/Lab%20Phases-9-2d6a4f?style=flat-square)
![Users](https://img.shields.io/badge/Users%20Provisioned-40-2d6a4f?style=flat-square)
![Sites](https://img.shields.io/badge/Sites-London%20%2F%20Paris-2d6a4f?style=flat-square)
![Issues](https://img.shields.io/badge/Troubleshooting%20Scenarios-7-2d6a4f?style=flat-square)

</div>

---

## Lab Environment

<div align="center">

| Component | Detail |
|:---|:---|
| **Domain** | `northwinddata.com` |
| **DC01** | Windows Server 2022 — London HQ (PDC Emulator, RID Master, Infrastructure Master) |
| **DC02** | Windows Server 2022 — Paris Branch |
| **AD Sites** | `London-HQ` / `Paris-Branch` |
| **Site Link** | `London-Paris-Link` — 15-minute replication interval |
| **Users** | 20 UK staff + 20 France staff across 5 departments each |
| **Cloud** | Microsoft 365 / Entra ID via Entra Connect |
| **Toolset** | ADUC · ADAC · GPMC · AD Sites & Services · WSB · ntdsutil · repadmin |

</div>

<br/>

<div align="center">
<img width="49%" alt="AD Sites and Services" src="https://github.com/user-attachments/assets/938d3185-9b85-405b-a4bb-ce406ad20034"/>
<img width="49%" alt="Users and Computers" src="https://github.com/user-attachments/assets/9d2eeed7-eb5c-408a-86f4-32db568f1b98"/>
</div>

---

## Lab Phases

### Phase 1 — Windows Server Backup

Installed Windows Server Backup on both DCs via Server Manager. Ran a manual System State backup on DC01 and configured a scheduled daily backup at 23:00. The System State captures everything AD needs: `NTDS.dit`, `SYSVOL`, the registry, and boot files.

> **Key constraint:** the backup destination cannot share a volume with the OS.

<div align="center">
<img width="49%" alt="WSB on DC1" src="https://github.com/user-attachments/assets/0c4cc2a5-2a2e-4f73-9077-b28ee27c67d4"/>
<img width="49%" alt="WSB Backup date" src="https://github.com/user-attachments/assets/029e746c-9646-43d0-9a3f-99baba0c34cb"/>
</div>

---

### Phase 2 — Non-Authoritative Restore

Booted DC01 into Directory Services Restore Mode via `msconfig`, restored the System State from backup using WSB, then rebooted normally. DC02 automatically replicated current data back to DC01, bringing it fully up to date.

Verified with `repadmin /replsummary` and Event IDs **1394** and **1168** in the Directory Service log.

<div align="center">
<img width="70%" alt="repadmin replsummary" src="https://github.com/user-attachments/assets/71e2e1b0-f49e-4f32-9811-d408d040504e"/>
</div>

---

### Phase 3 — Authoritative Restore

Simulated a deleted OU disaster — removed `FR_Sales` and all 5 users, then forced replication to DC02 so both DCs confirmed the deletion. Recovery requires two steps:

1. Restore from backup in DSRM
2. Run `ntdsutil` to mark restored objects as authoritative *before* rebooting

This stamps objects with a USN higher than DC02's copy, so they replicate outward rather than being overwritten. Verified `FR_Sales` and all users present on both DCs after reboot.

> **Critical order of operations:** rebooting before running `ntdsutil` causes DC02's deletion to replicate back and wipe the restore.

---

### Phase 4 — AD Recycle Bin

Enabled the AD Recycle Bin through ADAC — the modern recovery path requiring no backup or DSRM entry. Deleted Helene Mercier (France Country Manager) and restored her in seconds from the Deleted Objects container with all attributes intact. Also recovered the full `FR_Sales` OU.

> **Critical order:** always restore the parent OU before restoring objects inside it — otherwise users land in `LostAndFound`.

<div align="center">
<img width="49%" alt="Delete User Helene" src="https://github.com/user-attachments/assets/f4f9d2dc-a36e-4f68-b494-8efbaa6eaa1a"/>
<img width="49%" alt="Restore User Helene" src="https://github.com/user-attachments/assets/4eacad45-aed9-4f6a-a0bd-fd652d39ce1c"/>
</div>

---

### Phase 5 — Fine-Grained Password Policies

Created two Password Settings Objects via ADAC's Password Settings Container:

<div align="center">

| PSO | Precedence | Min Length | Lockout Threshold | Applied To |
|:---|:---:|:---:|:---:|:---|
| `PSO-IT-Management-HighSecurity` | 10 | 16 characters | 3 attempts | IT and Management groups |
| `PSO-StandardStaff` | 20 | 12 characters | 5 attempts | Domain Users |

</div>

> **Lower precedence number wins** when a user falls under multiple PSOs. Verified per-user application using ADAC's *View resultant password settings*.

<div align="center">
<img width="70%" alt="PSO Strong Password IT" src="https://github.com/user-attachments/assets/5c714a26-24e2-46a3-99e3-b90d03b8faef"/>
</div>

---

### Phase 6 — FSMO Role Management

Verified all five FSMO role holders across three tools: ADUC Operations Masters dialog, Active Directory Domains and Trusts, and the Schema MMC snap-in. Transferred the PDC Emulator and Domain Naming Master to DC02 to simulate a maintenance window, then transferred both back.

<div align="center">

| Scenario | Method | Condition |
|:---|:---|:---|
| Planned maintenance | **Transfer** — graceful handoff | Both DCs online |
| DC failure | **Seize** — forced takeover | Source DC unreachable |

</div>

> **If you seize a role, the original DC must never come back online.**

<div align="center">
<img width="45%" alt="RID Master dialog" src="https://github.com/user-attachments/assets/bb25a413-a4ff-40de-9b40-344fa7004f11"/>
</div>

---

### Phase 7 — Replication Monitoring

Forced replication between sites via AD Sites and Services (*Replicate Now* on the NTDS Settings connection object). Used `repadmin` to verify health and push updates across all directory partitions.

```powershell
repadmin /replsummary          # Overall replication health
repadmin /showrepl             # Per-partner replication detail
repadmin /syncall /Ade         # Force sync across all partitions
```

**Replication Event IDs monitored:**

| Event ID | Meaning |
|:---:|:---|
| 1394 | Replication healthy |
| 1311 | Topology calculation error |
| 2042 | Tombstone lifetime exceeded |

<div align="center">
<img width="70%" alt="Replication resolved" src="https://github.com/user-attachments/assets/e4e106ab-0b50-42d3-a53a-4edbf5ed0ef6"/>
</div>

---

### Phase 8 — AD Database Maintenance

Booted into DSRM and ran an integrity check on `NTDS.dit` via `ntdsutil → files → integrity`. Followed with an offline defragmentation to physically shrink the database file.

> **Online defrag** (automatic) reorganises data in-place but does not reduce file size.
> **Offline defrag** compacts `NTDS.dit` to a new path, replacing the original and removing superseded transaction logs.

Rebooted after replacing the file and confirmed replication resumed cleanly.

<div align="center">
<img width="70%" alt="Database Maintenance" src="https://github.com/user-attachments/assets/8d117b84-393a-47d9-b33e-c5aba86ce42c"/>
</div>

---

### Phase 9 — Advanced Group Policy

<div align="center">

| Tool / Technique | What It Solved |
|:---|:---|
| GPO Results Wizard | Identified `FR_UserPolicy` as the source of Control Panel block for a France user |
| GPO Modeling Wizard | Simulated USB restriction policy before live deployment |
| Link Order control | Managed policy precedence across France OUs |
| Block Inheritance on `FR_Management` | Prevented France_Staff restrictions from reaching managers |
| Enforced flag | Overrode Block Inheritance for security-critical GPOs |

</div>

---

### France Branch Office

Renamed the default site to `London-HQ`, created `Paris-Branch`, defined subnets for both, configured the `London-Paris-Link` at a 15-minute interval, and moved DC02 into `Paris-Branch`. Created 20 France users across 5 departments with full Organisation tab attributes — these sync to Microsoft 365 and drive dynamic group membership rules in Entra ID.

Ran the Delegation of Control Wizard to scope France IT Security Analyst rights to France users only, with no access to UK accounts.

<div align="center">
<img width="49%" alt="Site link replication interval" src="https://github.com/user-attachments/assets/75cee0c6-9f6b-4b27-906d-e8cff307cd04"/>
<img width="49%" alt="Users and Computers France" src="https://github.com/user-attachments/assets/c475d253-319d-49a3-a850-b387747690c2"/>
</div>

---

### Microsoft 365 & Entra ID Integration

Updated Entra Connect OU filtering to include `France_Staff`, triggered a manual delta sync, and all 20 France users appeared in the Microsoft 365 Admin Centre with on-premises attributes intact. Configured Address Book Policies in Exchange Online so France and UK users see only their own colleagues in Outlook and Teams. All users sign in with `@northwinddata.com`.

---

## Troubleshooting Scenarios

<div align="center">

| # | Symptom | Root Cause | Fix |
|:---:|:---|:---|:---|
| 1 | WSB backup failing — error `0x80070001` | Destination volume permissions | Re-ran WSB as Administrator |
| 2 | DSRM login rejected | Missing `.\` prefix — DSRM requires a local account, not a domain account | Used `.\Administrator` |
| 3 | Authoritative restore not persisting | Rebooted before running `ntdsutil` — DC02 deletion replicated back | Repeated in correct sequence |
| 4 | PSO not applying to users | PSO linked to an OU — PSOs only apply to group and user objects | Linked PSO to a security group |
| 5 | DC02 unresponsive after site move | Site link not configured with both sites | Fixed `DEFAULTIPSITELINK` properties |
| 6 | GPO not applying despite GPMC showing it linked | Authenticated Users removed from Security Filtering | Re-added via Delegation tab |
| 7 | Recycle Bin users landing in LostAndFound | Users restored before their parent OU | Restored container OU first |

</div>

<div align="center">
<img width="49%" alt="CMD replication troubleshooting" src="https://github.com/user-attachments/assets/5b166d1b-0dab-48f8-9124-8595748365b7"/>
<img width="49%" alt="Replication resolved" src="https://github.com/user-attachments/assets/5ad5c7da-302b-49c4-9e8b-039e1d28e94d"/>
</div>

---

## Certification Alignment

<div align="center">

![AZ-800](https://img.shields.io/badge/AZ--800-Administering%20Windows%20Server%20Hybrid%20Core-0078D4?style=flat-square&logo=microsoft&logoColor=white)
![AZ-801](https://img.shields.io/badge/AZ--801-Configuring%20Hybrid%20Advanced%20Services-0078D4?style=flat-square&logo=microsoft&logoColor=white)
![SC-300](https://img.shields.io/badge/SC--300-Identity%20%26%20Access%20Administrator-0078D4?style=flat-square&logo=microsoft&logoColor=white)
![AZ-104](https://img.shields.io/badge/AZ--104-Azure%20Administrator-0078D4?style=flat-square&logo=microsoft-azure&logoColor=white)

</div>

---

## Skills Demonstrated

<div align="center">

`Active Directory` `Windows Server 2022` `ADUC` `ADAC` `AD Sites & Services` `Windows Server Backup`
`System State` `DSRM` `ntdsutil` `Authoritative Restore` `AD Recycle Bin` `Fine-Grained Password Policies`
`FSMO Roles` `repadmin` `NTDS.dit` `GPMC` `Group Policy Results` `Group Policy Modeling`
`Entra Connect` `Microsoft 365` `Exchange Online` `Address Book Policies` `Hybrid Identity`
`Delegation of Control` `AZ-800` `AZ-801`

</div>

---

<div align="center">

*Part of an ongoing lab portfolio documenting enterprise infrastructure and cloud identity engineering.*
*More projects at [github.com/elvisodihiri](https://github.com/elvisodihiri)*

</div>
