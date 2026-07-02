# AZ-104: Implement and Manage Storage — Complete Study Guide

> Domain weight on the exam: **Implement and manage storage** is roughly **15–20%** of AZ-104. It's one of the highest-yield sections, so it's worth mastering deeply rather than skimming.

---

## Table of Contents
1. Storage Accounts — Fundamentals
2. Redundancy (Replication) Options
3. Access Tiers & Lifecycle Management
4. Securing Storage — Keys, SAS, Azure AD, RBAC
5. Network Security for Storage
6. Blob Storage Features (versioning, soft delete, immutability, snapshots)
7. Azure Files & File Sync
8. Azure Managed Disks
9. Data Movement Tools (AzCopy, Storage Explorer, Import/Export)
10. Monitoring & Diagnostics
11. Practice Exam Questions (with explanations)
12. Quick-Reference Cheat Sheet
13. Study Strategy Tips

---

## 1. Storage Accounts — Fundamentals

A **storage account** is the top-level namespace for Azure Storage services: Blob, File, Queue, Table, and Disk storage.

### Storage account types
| Type | Supports | Notes |
|---|---|---|
| **Standard general-purpose v2** | Blob, File, Queue, Table, Disk | Default/recommended for most scenarios; supports all redundancy options and access tiers |
| **Premium block blobs** | Blob (block/append) | High-transaction, low-latency workloads; SSD-backed |
| **Premium file shares** | Azure Files only | Low-latency file workloads; SSD-backed |
| **Premium page blobs** | Page blobs only | Used mainly for unmanaged VM disks (legacy) |

> **Exam tip:** General-purpose v2 (GPv2) is the default answer for "which account type should you use" unless the question specifies a premium/latency-sensitive requirement.

### Key naming and scope rules
- Storage account names: **3–24 characters**, lowercase letters and numbers only, must be **globally unique** across all of Azure.
- A storage account belongs to one **region** (or is zone/region-redundant depending on SKU).
- Default limit: **250 storage accounts per region per subscription** (soft limit, can be raised via support request).

### Performance tiers
- **Standard** — HDD-based, backs most workloads.
- **Premium** — SSD-based, lower latency, higher IOPS/throughput, higher cost.

---

## 2. Redundancy (Replication) Options

This is one of the most heavily tested areas. Know exactly what each option protects against.

| SKU | Copies | Protects against | Scope |
|---|---|---|---|
| **LRS** (Locally Redundant Storage) | 3 copies | Server/drive failure | Single datacenter |
| **ZRS** (Zone Redundant Storage) | 3 copies across zones | Datacenter/zone failure | Single region (multiple AZs) |
| **GRS** (Geo Redundant Storage) | 6 copies (3 local + 3 in paired region) | Regional outage | Primary + secondary region (secondary not readable) |
| **RA-GRS** (Read-Access GRS) | Same as GRS | Regional outage | Secondary region is **readable** via `-secondary` endpoint |
| **GZRS** (Geo-Zone Redundant) | ZRS in primary + LRS in secondary | Zone failure + regional outage | Best durability, not read-accessible on secondary |
| **RA-GZRS** | Same as GZRS | Same, plus secondary is readable | Highest availability + durability combo |

### Key exam facts
- **LRS** is the cheapest, least durable; **RA-GZRS** is the most expensive, most durable.
- The **paired region** for GRS/RA-GRS/GZRS/RA-GZRS is fixed by Azure (e.g., East US ↔ West US) — you cannot pick an arbitrary secondary region.
- Failover: You can perform a **customer-managed (unplanned) failover** on GRS/RA-GRS/GZRS/RA-GZRS accounts if the primary region is unavailable. This changes the secondary to primary — some recent writes may be lost (RPO typically < 15 min, not guaranteed).
- Converting LRS → GRS is supported; converting **ZRS ↔ GZRS** in-place has account-type restrictions — check whether the account was created with the desired redundancy or needs migration.
- **Object Replication** (blob-level, asynchronous copy to a different storage account, potentially different region) is different from redundancy — used for latency reduction, DR, or compliance, and requires **blob versioning** and **change feed** enabled on both accounts.

---

## 3. Access Tiers & Lifecycle Management

Applies to **Blob storage** (block blobs in GPv2 or Blob storage accounts).

### Access tiers
| Tier | Use case | Storage cost | Access/transaction cost | Min storage duration |
|---|---|---|---|---|
| **Hot** | Frequently accessed data | Highest | Lowest | None |
| **Cool** | Infrequently accessed, stored ≥30 days | Lower | Higher | 30 days |
| **Cold** | Rarely accessed, stored ≥90 days | Lower still | Higher | 90 days |
| **Archive** | Rarely accessed, stored ≥180 days, flexible latency (hours) | Lowest | Highest, plus rehydration cost | 180 days |

- Tier can be set at the **account level** (default tier for new blobs: Hot or Cool) and **overridden per blob**.
- **Archive tier blobs are offline** — you must **rehydrate** before reading:
  - **Standard priority**: up to 15 hours.
  - **High priority**: often < 1 hour (for blobs under ~10 GB).
- Rehydrate by changing the blob's tier or copying it to an online tier with `Copy Blob`.
- Early deletion from Cool/Cold/Archive before the minimum duration incurs a **pro-rated early deletion penalty**.

### Lifecycle Management Policies
Automate tier transitions and deletion using **JSON rule-based policies**, based on:
- Blob **age** (days since last modified, or since last access if **last access time tracking** is enabled).
- Blob **name prefix/blob index tags** filters.
- Actions: `tierToCool`, `tierToCold`, `tierToArchive`, `delete`, and separately for **snapshots** and **previous versions**.

Example lifecycle rule concept:
```json
{
  "rules": [
    {
      "name": "moveToArchiveAndDelete",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/"] },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

> **Exam tip:** If a question mentions "automatically move blobs to cooler tiers based on age" → answer is **Lifecycle Management**, not Azure Automation or Logic Apps.

---

## 4. Securing Storage — Keys, SAS, Azure AD, RBAC

Four ways to authenticate/authorize access to storage data — know when to use each.

### a) Account keys (Shared Key)
- Two keys (key1/key2) provide **full control** over the entire account.
- Support **key rotation** — regenerate one key while apps use the other, then rotate.
- Least secure/granular — avoid for production if possible; can be **disabled entirely** at the account level (`Allow storage account key access = No`), forcing Azure AD/SAS-only access.

### b) Shared Access Signature (SAS)
Grants **time-limited, scoped** access without sharing account keys.

| SAS Type | Scope | Signed by |
|---|---|---|
| **Account SAS** | Multiple services (Blob/File/Queue/Table), broad permissions | Account key |
| **Service SAS** | Single service (e.g., one blob container) | Account key |
| **User Delegation SAS** | Blob service only | **Azure AD credentials** (most secure — no account key involved) |

- SAS parameters: permissions (r/w/d/l...), start/expiry time, allowed IP range, allowed protocol (HTTPS only recommended), signed resource.
- **Stored access policies** let you group SAS permissions on a container and revoke all associated SAS tokens instantly by deleting/modifying the policy — without waiting for expiry.
- **User Delegation SAS is the recommended/most secure option** and is a common "best practice" exam answer.

### c) Azure AD (Microsoft Entra ID) authentication
- Authenticate via Azure AD identity (user, group, managed identity, service principal) instead of keys.
- Authorization is then handled via **Azure RBAC** data-plane roles.
- Supported for Blob and Queue storage (not natively for Table/File SMB in the same way, though Files supports Azure AD Kerberos/DS auth separately).

### d) Azure RBAC roles for storage data
| Role | Grants |
|---|---|
| **Storage Blob Data Reader** | Read blobs |
| **Storage Blob Data Contributor** | Read/write/delete blobs |
| **Storage Blob Data Owner** | Full control incl. POSIX ACLs (Data Lake) |
| **Storage Queue Data Contributor** | Read/write/delete queue messages |
| **Storage Account Contributor** (mgmt plane) | Manage the account (keys, config) but **not** data access by default |

> **Exam tip — classic trap:** Having the **Owner** or **Contributor** role at the subscription/resource level does **not** automatically grant data-plane access to blobs/queues. You need an explicit **Storage Blob Data \*** role (or a key/SAS) — this is a very common exam distractor.

---

## 5. Network Security for Storage

- **Firewalls and virtual networks** (Storage account → Networking):
  - Default: **Allow access from all networks** (public endpoint, secured by key/SAS/AAD).
  - **Enabled from selected virtual networks and IP addresses**: restrict to specific VNet/subnets (via **service endpoint**) and/or public IP ranges.
  - **Disabled** (private endpoint/private link only) — most restrictive.
- **Service Endpoint** (`Microsoft.Storage`): extends VNet identity to the storage service over the Azure backbone; traffic stays on Azure network but the storage account still has a public IP.
- **Private Endpoint**: creates a **private IP address** inside your VNet for the storage account (via Azure Private Link). Traffic never touches the public internet — this is the most secure option and is required to fully "air-gap" a storage account from the internet.
- Exceptions checkbox: "Allow trusted Microsoft services" lets specific first-party services (e.g., Azure Monitor, Azure Backup) bypass the firewall.

> **Exam tip:** If the requirement is "no public internet access at all" → **Private Endpoint**. If the requirement is "restrict to specific VNets, still uses Azure backbone, simpler/cheaper" → **Service Endpoint**.

---

## 6. Blob Storage Features

### Blob types
- **Block blobs** — general files, up to ~190.7 TiB, composed of blocks (used for most data, backups, media).
- **Append blobs** — optimized for append operations (e.g., logging).
- **Page blobs** — random read/write, used for VHD/VM disks.

### Data protection features
| Feature | Purpose |
|---|---|
| **Soft delete (blobs)** | Recover deleted/overwritten blobs within a retention window (1–365 days) |
| **Soft delete (containers)** | Recover an entire deleted container |
| **Versioning** | Automatically keeps previous versions of a blob on every overwrite |
| **Snapshots** | Manual, point-in-time, read-only copy of a blob |
| **Point-in-time restore** | Restore block blob data to an earlier state (requires versioning, change feed, and soft delete enabled) |
| **Immutable storage (WORM)** | Time-based retention or legal hold — blobs cannot be modified or deleted, used for compliance (e.g., SEC 17a-4) |
| **Change feed** | Ordered, durable log of create/update/delete transactions on blobs |

> **Exam tip:** Versioning ≠ Snapshots. Versioning is **automatic** on every write; Snapshots are **manual/on-demand**. Point-in-time restore needs all three (versioning + change feed + soft delete) enabled together.

### Blob index tags
Key-value metadata attached to blobs, used for filtering/finding blobs without listing entire containers, and as filters in lifecycle policies.

---

## 7. Azure Files & File Sync

### Azure Files basics
- SMB (and NFS for premium) file shares in the cloud, mountable from Windows, Linux, macOS.
- Tiers: **Transaction optimized, Hot, Cool** (standard) and **Premium** (SSD-backed FileStorage account type).
- Share quotas: set max size per share (up to 100 TiB for standard/premium large file shares).
- Access via:
  - SMB with **Storage account key**.
  - **Azure AD Domain Services** or **Azure AD Kerberos (hybrid)** for identity-based SMB access with NTFS permissions.
  - **On-premises AD DS** authentication (via domain-joined VM/servers) for hybrid identity.

### Azure File Sync
Syncs on-prem Windows Server file shares with an Azure file share (cloud as the "hub").
- **Storage Sync Service** → **Sync Group** → **Cloud Endpoint** (Azure file share) + **Server Endpoint** (path on registered server).
- **Cloud tiering**: infrequently used files are replaced on-prem with pointers (stubs); full content stays in Azure and is fetched on-demand when opened.
- Provides **multi-site sync**, **cloud-side backup**, and **fast disaster recovery** for on-prem file servers.

> **Exam tip:** "Free up local disk space on an on-prem file server while keeping full files in Azure" → **Azure File Sync with cloud tiering**.

---

## 8. Azure Managed Disks

- Disk types: **Ultra Disk, Premium SSD v2, Premium SSD, Standard SSD, Standard HDD**.
- Managed disks abstract the underlying storage account — Azure manages placement/scaling.
- **Disk redundancy**: LRS or **ZRS** (zone-redundant) depending on disk type/region.
- Operations to know:
  - **Snapshot** a disk (full or incremental) for backup.
  - **Convert** unmanaged disk → managed disk.
  - **Resize** a disk (VM must typically be deallocated for a decrease; some SKUs allow online resize/increase).
  - **Add a data disk** vs **expand OS disk**.
  - **Shared disks** (for clustering, e.g., Windows Server Failover Clustering) — requires disk-level shared enabled and appropriate SKU.

---

## 9. Data Movement Tools

| Tool | Best for |
|---|---|
| **AzCopy** | Command-line, high-performance copy/sync of blobs and files (local↔cloud or cloud↔cloud) |
| **Azure Storage Explorer** | GUI tool for browsing/managing Blob, File, Queue, Table |
| **Azure Data Box** | Physical device shipped for **offline bulk transfer** of large datasets (TB–PB scale) where network transfer is impractical |
| **Import/Export service** | Ship your own disks to an Azure datacenter for bulk import/export |
| **Azure Migrate / Storage migration tools** | Larger workload migrations |

> **Exam tip:** "Transfer 200 TB of data and internet bandwidth is limited" → **Azure Data Box** (or Data Box Heavy for larger scale). "Automate/script a nightly copy job" → **AzCopy**.

---

## 10. Monitoring & Diagnostics

- **Azure Storage Metrics** (via Azure Monitor): capacity, transactions, availability, latency — available at account and service level.
- **Storage Analytics Logging** (classic) vs **Diagnostic settings → Log Analytics/Storage/Event Hub** (modern, recommended).
- **Azure Monitor Alerts**: set thresholds on metrics (e.g., used capacity, availability %) to trigger action groups.
- **Last access time tracking**: needed for access-time-based lifecycle rules.

---

## 11. Practice Exam Questions

**Q1.** You need to ensure a storage account remains available for read operations even if an entire Azure region goes offline, and reads should be served from the secondary region. Which redundancy option should you choose?
A. LRS
B. ZRS
C. GRS
D. RA-GRS

<details><summary>Answer</summary>
<b>D. RA-GRS</b> (or RA-GZRS). Plain GRS replicates to a secondary region but does not allow you to read from it unless read-access is enabled.
</details>

---

**Q2.** A developer needs temporary, time-limited access to upload files to a single blob container, without receiving the storage account key. What should you provide?
A. Account key
B. Service SAS scoped to the container
C. RBAC Owner role on the storage account
D. Storage account connection string

<details><summary>Answer</summary>
<b>B. Service SAS scoped to the container</b> (ideally a User Delegation SAS signed with Azure AD credentials for best security). This grants scoped, time-limited access without exposing the account key.
</details>

---

**Q3.** Your company has compliance requirements stating that certain financial records stored as blobs must not be modified or deleted for 7 years. What should you configure?
A. Soft delete
B. Blob versioning
C. Immutable storage with a time-based retention policy
D. Lifecycle management delete rule

<details><summary>Answer</summary>
<b>C. Immutable storage (WORM) with a time-based retention policy.</b> This enforces that blobs cannot be modified or deleted until the retention period expires — meeting regulatory (e.g., SEC 17a-4) requirements.
</details>

---

**Q4.** A user has been assigned the **Contributor** role on a storage account at the resource group level. They report they cannot read blob contents through the Azure Storage SDK using their Azure AD identity. What is the most likely cause?
A. The storage account firewall is blocking them
B. They lack a Storage Blob Data role assignment
C. Soft delete is enabled
D. The blob is in the Archive tier

<details><summary>Answer</summary>
<b>B. They lack a Storage Blob Data role assignment.</b> Management-plane roles like Contributor do not grant data-plane access to blob data; a role such as Storage Blob Data Reader/Contributor is required.
</details>

---

**Q5.** You want to automatically move blobs older than 60 days to the Cool tier and delete them after 2 years, without writing custom code. What should you use?
A. Azure Automation runbook
B. Blob lifecycle management policy
C. Azure Data Factory pipeline
D. Logic App with a timer trigger

<details><summary>Answer</summary>
<b>B. Blob lifecycle management policy.</b> This is a built-in, rule-based, no-code feature designed exactly for age-based tiering and deletion.
</details>

---

**Q6.** An on-premises file server is running low on disk space, but users still need to see and open all their files as if they were local. What Azure feature addresses this?
A. Azure Files with SMB mount
B. Azure File Sync with cloud tiering enabled
C. Azure Data Box
D. AzCopy sync

<details><summary>Answer</summary>
<b>B. Azure File Sync with cloud tiering enabled.</b> It keeps a full copy of data in Azure while replacing infrequently used local files with lightweight stubs, freeing local disk space while preserving the illusion of full files.
</details>

---

**Q7.** Which redundancy option protects against the loss of an entire datacenter (availability zone) within a single region, without replicating to another region?
A. LRS
B. ZRS
C. GRS
D. RA-GRS

<details><summary>Answer</summary>
<b>B. ZRS.</b> It synchronously replicates across three availability zones in one region. LRS only protects against a single node/disk failure; GRS/RA-GRS add cross-region protection but at extra cost/complexity.
</details>

---

**Q8.** You need to completely block all public internet access to a storage account, while still allowing access from a specific VNet/subnet over a private IP address. What should you configure?
A. Service endpoint
B. Storage firewall IP allow-list
C. Private Endpoint with public network access disabled
D. Azure AD Conditional Access policy

<details><summary>Answer</summary>
<b>C. Private Endpoint with public network access disabled.</b> This assigns a private IP in your VNet and, combined with disabling public network access, ensures traffic never traverses the public internet.
</details>

---

**Q9.** A blob was accidentally overwritten. Blob versioning was enabled prior to the incident. How do you restore the original content with the least effort?
A. Restore from an Azure Backup vault
B. Promote the previous version of the blob to be the current version
C. Recreate the blob manually from a local copy
D. Use Import/Export to re-upload the blob

<details><summary>Answer</summary>
<b>B. Promote the previous version of the blob to be the current version.</b> With versioning enabled, every overwrite automatically creates a new version, and any prior version can be restored directly.
</details>

---

**Q10.** You need to migrate 150 TB of on-premises data to Azure Blob Storage, and your internet connection would take several weeks to transfer it. What is the best solution?
A. AzCopy over a site-to-site VPN
B. Azure Data Box
C. Azure Storage Explorer
D. Increase ExpressRoute bandwidth temporarily

<details><summary>Answer</summary>
<b>B. Azure Data Box.</b> For large, one-time bulk transfers where network transfer is impractically slow, Data Box (a physical shipped device) is the standard solution.
</details>

---

## 12. Quick-Reference Cheat Sheet

**Redundancy:** LRS (cheapest, single DC) → ZRS (zone) → GRS (region, unreadable secondary) → RA-GRS (region, readable secondary) → GZRS/RA-GZRS (zone + region combined)

**Access tiers:** Hot (frequent) → Cool (30-day min) → Cold (90-day min) → Archive (180-day min, offline, needs rehydration)

**Security layering:** Account Key (full control) > SAS (scoped, time-limited) > User Delegation SAS (Azure AD-signed, most secure) > Azure AD + RBAC (identity-based, no shared secret)

**Network isolation:** Public (default) < Firewall IP/VNet rules < Service Endpoint < Private Endpoint (most isolated)

**Data protection:** Soft delete (undo delete) / Versioning (undo overwrite, automatic) / Snapshot (manual point-in-time copy) / Immutability (compliance WORM)

**Bulk transfer:** AzCopy (network-based) vs Data Box (offline/physical, huge datasets)

---

## 13. Study Strategy Tips

1. **Do it hands-on.** Spin up a free-tier Azure account and actually create a storage account, generate a SAS token, set up a lifecycle policy, and configure a private endpoint. AZ-104 heavily tests "what would you click/configure" scenarios.
2. **Focus on scenario keywords.** The exam rarely asks "what is GRS" — it asks "your company needs X, which option meets the requirement with least cost/effort." Train yourself to map requirement phrases (e.g., "protect against regional outage and allow reads from secondary") to the correct feature.
3. **Know the RBAC vs Shared Key vs SAS distinction cold** — this trips up a large share of candidates.
4. **Memorize the minimum-day thresholds** for Cool (30), Cold (90), and Archive (180) tiers — these show up in early-deletion-penalty style questions.
5. **Practice reading multi-constraint questions** (e.g., cost + compliance + region) and eliminate answers that violate any single constraint first.
6. **Use Microsoft Learn's official AZ-104 learning path** and the free **Microsoft Learn sandbox** labs to reinforce hands-on skills alongside this guide.

---

*Good luck with your AZ-104 prep! If you'd like, I can also generate a similar guide for the other AZ-104 domains (Identity/Governance, Compute, Networking, Monitoring/Backup) or build a set of interactive practice-quiz flashcards.*
