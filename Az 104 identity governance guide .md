# AZ-104 — Manage Azure Identities and Governance: Deep Dive Guide

> **Domain weight:** 20–25% (joint highest with Compute)
> **Expected exam questions:** ~10–15
> **Why it matters:** Identity is the new control plane. This domain underpins every other domain — RBAC scopes, policy enforcement, and subscription structure show up indirectly in Storage, Compute, Networking, and Monitor questions too.

---

## Table of contents

1. [Microsoft Entra ID fundamentals](#1-microsoft-entra-id-fundamentals)
2. [Users — creation, types, properties](#2-users--creation-types-properties)
3. [Groups — security, M365, dynamic, assigned](#3-groups--security-m365-dynamic-assigned)
4. [Licenses](#4-licenses)
5. [External users (B2B / guest)](#5-external-users-b2b--guest)
6. [Self-Service Password Reset (SSPR)](#6-self-service-password-reset-sspr)
7. [Multi-factor authentication](#7-multi-factor-authentication)
8. [Azure RBAC — the access control model](#8-azure-rbac--the-access-control-model)
9. [Built-in roles you must memorize](#9-built-in-roles-you-must-memorize)
10. [Custom RBAC roles](#10-custom-rbac-roles)
11. [Scope hierarchy and inheritance](#11-scope-hierarchy-and-inheritance)
12. [Interpreting access (effective access, deny assignments)](#12-interpreting-access-effective-access-deny-assignments)
13. [Subscriptions and management groups](#13-subscriptions-and-management-groups)
14. [Resource groups](#14-resource-groups)
15. [Azure Policy](#15-azure-policy)
16. [Resource locks](#16-resource-locks)
17. [Tags](#17-tags)
18. [Cost management](#18-cost-management)
19. [Entra ID roles vs Azure RBAC roles — the most-confused topic](#19-entra-id-roles-vs-azure-rbac-roles--the-most-confused-topic)
20. [Hands-on lab walkthroughs](#20-hands-on-lab-walkthroughs)
21. [Exam question patterns and traps](#21-exam-question-patterns-and-traps)
22. [Cheat sheet](#22-cheat-sheet)

---

## 1. Microsoft Entra ID fundamentals

Microsoft Entra ID (formerly Azure Active Directory / Azure AD — renamed in 2023) is Azure's cloud-based identity and access management service. Every Azure subscription is **trusted** by exactly one Entra tenant.

### Tenant vs subscription vs directory

- **Tenant** = a dedicated, isolated instance of Entra ID. Represents an organization. Identified by a tenant ID (GUID) and a default domain like `contoso.onmicrosoft.com`.
- **Subscription** = a billing/container boundary for Azure resources. A subscription trusts exactly one tenant. One tenant can have many subscriptions.
- **Directory** = the older term, used interchangeably with tenant.

### Entra ID is NOT Active Directory Domain Services (AD DS)

This is one of the highest-frequency exam concepts. They are different products.

| Feature | On-prem AD DS | Microsoft Entra ID |
|---|---|---|
| Protocols | Kerberos, NTLM, LDAP | OAuth 2.0, OIDC, SAML, WS-Fed |
| Structure | OUs, forests, domains, GPO | Flat — no OUs, no GPO |
| Querying | LDAP | REST / Graph API |
| Devices | Domain join, GPO | Entra join, Intune (MDM) |
| Trust | Forest/domain trusts | Federation, B2B |
| Hierarchy | Hierarchical | Flat |

If a question mentions **Group Policy, OUs, LDAP queries, or Kerberos** → it's referring to AD DS, not Entra ID. If you need AD DS features in Azure, you use **Microsoft Entra Domain Services** (managed AD DS) — a separate paid service.

### Entra ID editions (license tiers)

| Edition | Key features |
|---|---|
| **Free** | Basic users/groups, SSO for some apps, MFA via Security Defaults |
| **P1** | Conditional Access, full SSPR with writeback, dynamic groups, advanced group management, hybrid identity |
| **P2** | Everything in P1 + Identity Protection (risky users/sign-ins), Privileged Identity Management (PIM), access reviews |

Exam tip: if the scenario needs **Conditional Access, dynamic groups, group writeback, or SSPR with on-prem writeback**, you need **at least P1**. PIM, risk-based Conditional Access, or access reviews → **P2**.

---

## 2. Users — creation, types, properties

### User types

| Type | Source | Use case |
|---|---|---|
| **Cloud identity** | Created in Entra ID directly | Native cloud-only users |
| **Directory-synchronized** | Synced from on-prem AD via Entra Connect / Cloud Sync | Hybrid identity |
| **Guest user** (B2B) | Invited from another Entra tenant or social account | External collaboration |

### Core user properties (memorize these field names)

- **User Principal Name (UPN):** the sign-in name, format `user@domain.com`. Must be unique tenant-wide.
- **Object ID:** GUID, never changes.
- **Mail / Email address:** can differ from UPN.
- **Usage location:** **REQUIRED** for license assignment. Two-letter country code (e.g., `BD`, `US`). Many candidates miss this — assigning a license without `usageLocation` set will fail.
- **Account enabled:** disable instead of delete to preserve history.

### Bulk user operations

- **Create:** "Bulk create" in the portal → upload a CSV with required headers.
- **Invite:** "Bulk invite" → CSV of guest emails.
- **Delete:** soft-deleted users go to a recycle bin. **Restorable for 30 days**, then permanently purged.

### CLI / PowerShell

```powershell
# PowerShell (Microsoft.Graph module)
New-MgUser -DisplayName "Aisha Khan" -UserPrincipalName "aisha@contoso.com" `
  -MailNickname "aisha" -AccountEnabled `
  -PasswordProfile @{Password="P@ssw0rd!"; ForceChangePasswordNextSignIn=$true}
```

```bash
# Azure CLI
az ad user create \
  --display-name "Aisha Khan" \
  --user-principal-name "aisha@contoso.com" \
  --password "P@ssw0rd!" \
  --force-change-password-next-sign-in true
```

---

## 3. Groups — security, M365, dynamic, assigned

### Group types

| Type | Used for | Notes |
|---|---|---|
| **Security** | Granting access to apps and Azure resources | Most common for RBAC |
| **Microsoft 365** | Collaboration (Teams, SharePoint, Outlook) | Comes with a shared mailbox & site |

### Membership types (this is where dropdowns trip you up)

| Membership type | How members are added | Needs P1? |
|---|---|---|
| **Assigned** (static) | Manually added by admin | No |
| **Dynamic User** | Rule based on user attributes | **Yes — P1** |
| **Dynamic Device** | Rule based on device attributes | **Yes — P1** |

### Dynamic group rule example

```text
(user.department -eq "Engineering") -and (user.country -eq "Bangladesh")
```

Common operators: `-eq`, `-ne`, `-contains`, `-startsWith`, `-match`, `-in`, `-and`, `-or`, `-not`.

**Exam trap:** Dynamic membership groups can be **either user OR device** — not both in the same group. If a question asks for one group containing both → impossible.

### Nested groups

- Security groups **can** be nested.
- M365 groups **cannot** be nested (no group-in-group membership).
- Some Azure RBAC role assignments and license assignments **do not respect nested membership** — assign roles/licenses directly to the relevant group, not to a parent.

---

## 4. Licenses

### Group-based licensing

- Assign a license to a security group → all members inherit it.
- Requires `usageLocation` on every user.
- Removed when user leaves the group.
- Faster than per-user assignment for large orgs.

### License conflicts

If two assigned products share a feature (e.g., Exchange Online), you'll see a conflict. Resolve with the "service plans" toggle inside the license assignment.

---

## 5. External users (B2B / guest)

### B2B vs B2C

| | B2B | B2C |
|---|---|---|
| Purpose | Collaboration with partner orgs | Customer-facing apps |
| Identity store | Guest entries in YOUR tenant | Separate B2C tenant |
| Examples | Inviting a vendor to view a Power BI report | Customer signs up to your e-commerce site |

**AZ-104 mostly covers B2B.** B2C is a separate product (Azure AD B2C / Entra External ID).

### Guest user invitation

- Send invite from "Users → New guest user → Invite".
- They receive an email, click "Accept" → guest account created in your tenant.
- Guest's UPN looks like: `aisha_othercompany.com#EXT#@yourtenant.onmicrosoft.com`.
- Guests have **restricted** default permissions — they can view their own profile and limited directory info.

### Restricting guests further

`External Identities → External collaboration settings`:
- Who can invite (Admins only / Members can / Anyone)
- Guest user access restrictions (Most/Limited/Same as members)

---

## 6. Self-Service Password Reset (SSPR)

Lets users reset their own password without calling IT.

### Three states

- **None** — disabled.
- **Selected** — only members of a specific group.
- **All** — everyone in the tenant.

### Authentication methods

User must register at least the **required number** of methods (default: 1, but admins typically require 2).

- Mobile app notification / code
- Email (alternate, not primary)
- Mobile phone (SMS / call)
- Office phone
- Security questions

### Password writeback

- For **hybrid users** synced from on-prem AD.
- Allows the cloud password change to flow back to on-prem AD.
- **Requires P1** and Entra Connect / Cloud Sync configured.

### Exam scenario pattern

> "Users in the Sales group must be able to reset their own passwords. Hybrid users' passwords must update in on-prem AD."
>
> **Answer:** Enable SSPR scoped to the Sales group + enable password writeback (requires P1).

---

## 7. Multi-factor authentication

### Three ways to enable MFA

| Method | When | Notes |
|---|---|---|
| **Security Defaults** | Tenant-wide, free | All-or-nothing, all users, basic. Disabled if Conditional Access is used. |
| **Per-user MFA** (legacy) | Per user, in `user → Multi-factor authentication` | Old, being phased out |
| **Conditional Access** | Targeted, granular | **P1 required.** Best practice. |

### Conditional Access components

A CA policy = `Assignments` (who, what) + `Access controls` (allow/block, with conditions).

**Assignments:**
- Users / groups (include or exclude)
- Cloud apps (e.g., "All cloud apps" or specific ones)
- Conditions (sign-in risk, device platform, location, client app)

**Access controls:**
- Block access
- Grant access requiring: MFA, compliant device, hybrid Entra-joined, approved client app, T&C, etc.
- Session controls: app-enforced restrictions, sign-in frequency

### Common CA scenario patterns

| Requirement | Policy |
|---|---|
| MFA always | Users: All • Apps: All cloud apps • Grant: Require MFA |
| MFA only off-corporate-network | Add a named location condition for the corporate IP, exclude it |
| Block legacy auth | Conditions: client apps = "exchange ActiveSync, other clients" • Grant: Block |
| MFA for admins | Users: directory roles (Global Admin, etc.) • Grant: Require MFA |

---

## 8. Azure RBAC — the access control model

> **The single most heavily-tested concept in this domain.**

Azure RBAC controls **what someone can do with Azure resources**. (Not directory data — that's Entra roles, see §19.)

### The RBAC formula

> **Role assignment = Security principal + Role definition + Scope**

### 1. Security principal (the WHO)

Any of these can receive a role:
- **User**
- **Group** (preferred — manage membership instead of role assignments)
- **Service principal** (an app's identity)
- **Managed identity** (an Azure resource's identity, system or user-assigned)

### 2. Role definition (the WHAT)

A JSON document with three core sections:
- `Actions` — control-plane operations the role can perform (e.g., `Microsoft.Compute/virtualMachines/start/action`)
- `NotActions` — actions to subtract from `Actions`
- `DataActions` / `NotDataActions` — data-plane operations (reading blob contents, sending to a queue, etc.)
- `AssignableScopes` — where the role can be assigned

### 3. Scope (the WHERE)

Roles inherit downward through this hierarchy:

```
Root (/)
  └─ Management Group
       └─ Subscription
            └─ Resource Group
                 └─ Resource
```

A role assigned at Subscription level applies to **every** RG and resource inside it.

---

## 9. Built-in roles you must memorize

There are 100+ built-in roles. The exam focuses on a small set. **Know these cold.**

### Fundamental (general-purpose)

| Role | Can do | Cannot do |
|---|---|---|
| **Owner** | Everything, including assigning RBAC roles | — |
| **Contributor** | Create/manage all resource types | Cannot grant access (cannot assign roles) |
| **Reader** | View everything | Cannot make any changes |
| **User Access Administrator** | Manage user access to Azure resources only | Cannot manage resources themselves |

> **High-frequency exam trap:** A Contributor **cannot grant other people access**. If a question says "user X needs to manage VMs and grant others access" → they need **Owner** or **User Access Administrator + Contributor**.

### Resource-specific (least-privilege examples)

| Role | Use |
|---|---|
| Virtual Machine Contributor | Manage VMs (cannot access VNet/storage) |
| Virtual Machine Administrator Login | Sign in as admin to a VM |
| Virtual Machine User Login | Sign in as standard user |
| Storage Blob Data Owner / Contributor / Reader | Data plane access to blobs |
| Storage Account Contributor | Manage storage account (NOT data inside) |
| Network Contributor | Manage networks |
| Reader and Data Access | Storage account: list keys + view |
| Key Vault Administrator / Secrets User | Key Vault management vs secret usage |
| Cost Management Reader | View cost data |
| Backup Operator / Backup Contributor | Backup tasks |

### Storage gotcha

- **Storage Account Contributor** = manage the *account* (configure properties). Does **NOT** read/write blob data.
- **Storage Blob Data Reader/Contributor/Owner** = read/write actual blobs. These are "data plane" roles.

If someone needs to upload blobs but not change account settings → **Storage Blob Data Contributor**, not Storage Account Contributor.

---

## 10. Custom RBAC roles

Used when no built-in role fits least-privilege.

### When to create custom roles

- A built-in role grants too much (e.g., VM Contributor can delete; you only want start/stop).
- You need a unique combination of permissions across services.

### Custom role JSON structure

```json
{
  "Name": "VM Operator",
  "Description": "Can start and stop VMs but not create or delete.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Resources/subscriptions/resourceGroups/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/00000000-0000-0000-0000-000000000000"
  ]
}
```

### Limits

- Maximum **5,000 custom roles per tenant** (Azure China: 2,000).
- Custom roles can be created via **Portal, PowerShell, CLI, REST API, ARM template**.

### Create with CLI

```bash
az role definition create --role-definition vm-operator.json
```

---

## 11. Scope hierarchy and inheritance

### The four scope levels

1. **Management group** — collection of subscriptions
2. **Subscription**
3. **Resource group**
4. **Resource**

A role assignment at any level **inherits down**. Inheritance is **additive** — you cannot subtract permissions by assigning a "less" role at a child scope. (Deny assignments do this — see §12.)

### Best practice

- Apply roles at the **highest reasonable scope**.
- Assign roles to **groups** rather than users.
- Use **least privilege** — the most-tested concept on this domain.

### Elevate access (root scope)

A Global Administrator (Entra role) does **NOT** automatically get Azure RBAC permissions. To bootstrap access:

1. Sign in as Global Administrator.
2. `Entra ID → Properties → Access management for Azure resources` → toggle Yes.
3. This grants the **User Access Administrator** role at root scope `/`.
4. Now you can assign yourself Owner.
5. **Disable** the toggle when done.

This is a heavily-tested scenario.

---

## 12. Interpreting access (effective access, deny assignments)

### Reading access in the portal

`Resource → Access control (IAM) → Check access` — pick a user, see all role assignments inherited from every scope.

`View role assignments` — see who has access at this scope (and inherited).

### Deny assignments

- **Cannot be created by you directly.**
- Created automatically by Azure-managed services (e.g., Azure Blueprints — now deprecated, Azure Managed Apps, Azure DevTest Labs).
- A deny assignment **always wins** over an allow.
- Order of evaluation: **Deny → Allow → Default deny.**

### "Effective permissions" gotcha

If a user has Reader at subscription level AND Contributor at RG level → at the RG, they have **Contributor** (the union, since both are allows). RBAC is additive.

---

## 13. Subscriptions and management groups

### Subscription = billing + RBAC + policy boundary

- Each subscription has an Owner who pays the bill.
- A subscription is associated with **one** Entra tenant.
- A subscription can be **moved** between tenants (rare, complex).

### Subscription types you might see

| Type | Notes |
|---|---|
| Free Trial | $200 credit, 30 days, plus 12 months free tier |
| Pay-As-You-Go | Pay per use |
| Enterprise Agreement (EA) | Volume licensing, big enterprises |
| Microsoft Customer Agreement (MCA) | Newer billing model replacing EA |
| CSP | Bought via partner |

### Management groups

- Containers for **subscriptions**.
- Up to **6 levels deep** (excluding root and subscription levels).
- A subscription can belong to **only one** management group.
- Used to apply policy and RBAC consistently across many subs.

```
Root (Tenant Root Group)
  ├─ Production MG
  │    ├─ Subscription A
  │    └─ Subscription B
  └─ Non-Production MG
       └─ Subscription C
```

### Why management groups matter

- Apply Azure Policy at MG level → inherited by all subs and resources.
- Apply RBAC at MG level → "global Reader" effect.

### Tenant Root Group

- The implicit top-level MG.
- By default, only the user who first elevated access can manage it.
- Best practice: don't put assignments at root — use a child MG so you have flexibility.

---

## 14. Resource groups

### What a resource group is

- A logical container for resources that share a lifecycle.
- Every resource belongs to **exactly one** RG.
- A resource group has a **region** (where its metadata is stored) — but **resources inside can be in different regions**.

### Moving resources

- Most resources can be moved between RGs and subscriptions.
- Some can't (e.g., classic resources, certain types).
- A move **does not change the resource's region**.
- Resources that depend on each other (e.g., VM and its disk) usually need to be moved together.

### When you delete an RG

- All resources inside are deleted.
- Operation is **irreversible**.

---

## 15. Azure Policy

> Enforces rules on resources. Not the same as RBAC.
>
> - **RBAC** = "*who* can do *what*" (people-centric)
> - **Policy** = "*what* resources can exist with *what* properties" (resource-centric)

### Components

1. **Policy definition** — the rule (JSON). Describes what to evaluate and the effect.
2. **Initiative** (policy set) — a group of related definitions, treated as one.
3. **Assignment** — applying a definition or initiative to a scope (MG, sub, RG).
4. **Parameters** — values you supply when assigning (e.g., allowed locations).
5. **Exemption** — exclude a specific resource from compliance.

### Effects (memorize)

| Effect | What it does |
|---|---|
| **Deny** | Block creation/update of non-compliant resources |
| **Audit** | Allow but flag in compliance report |
| **AuditIfNotExists** | Audit if a related resource doesn't exist (e.g., diagnostic settings) |
| **DeployIfNotExists** | Auto-deploy a related resource (e.g., enable Backup) |
| **Modify** | Add/change tags or properties |
| **Append** | Add fields to a resource (e.g., a default tag) |
| **Disabled** | Temporarily turn off |

`DeployIfNotExists` and `Modify` need a **managed identity** for the assignment to act.

### Common policy scenarios

- Allowed locations (only `eastus`, `westeurope`)
- Allowed VM SKUs
- Require a tag on every resource
- Inherit a tag from RG
- Audit storage accounts that allow public blob access

### Compliance evaluation

Policies evaluate every **24 hours** on existing resources, and immediately on create/update. You can trigger an on-demand scan with:

```bash
az policy state trigger-scan
```

### Initiatives (policy sets)

- Group related policies (e.g., "ISO 27001 compliance" — 100+ definitions in one initiative).
- Built-in initiatives include: Azure Security Benchmark, NIST SP 800-53, HIPAA-HITRUST, PCI DSS.

### Policy vs RBAC question pattern

If the question says "**enforce** or **prevent** at create time" → Policy with **Deny**.
If the question says "**limit who can create**" → RBAC.

Both can coexist — and in fact should.

---

## 16. Resource locks

Prevent accidental deletion or modification.

### Two types

| Lock type | What it allows | What it blocks |
|---|---|---|
| **CanNotDelete** (Delete) | All operations | Delete |
| **ReadOnly** | Read | Any write or delete |

### Scope and inheritance

- Apply at **subscription, RG, or resource**.
- Locks are **inherited by child resources**.
- The **most restrictive** lock wins. ReadOnly at the RG beats CanNotDelete on a single resource inside.

### Permissions to manage locks

- Owner and User Access Administrator can create/delete locks.
- (More precisely: requires `Microsoft.Authorization/locks/*` permission.)

### Caveats — the "weird side effects"

ReadOnly locks can cause unexpected breakage:
- ReadOnly on a storage account → you can't list keys (which is a POST/list operation, treated as a write).
- ReadOnly on a resource group → you can't restart VMs (restart is a write operation under the hood).

Exam tip: if a scenario says "the admin can no longer X" and there's a ReadOnly lock somewhere → that's the cause.

---

## 17. Tags

Key-value metadata for organizing resources.

### Rules

- Up to **50 tags per resource**.
- Tag name max **512 characters** (storage accounts: 128).
- Tag value max **256 characters**.
- Tags are **case-insensitive on the name**, but stored case-sensitive.
- **Not all resources support tags** (e.g., some classic resources, resource groups themselves DO support tags but **don't inherit to child resources by default**).

### Inheritance

Tags are **NOT** inherited automatically. To force inheritance:
- Use Azure Policy with the **Inherit tag from resource group** built-in policy (Modify effect).

### Why tags matter on the exam

- **Cost reporting:** group costs by `Department`, `Project`, `CostCenter`.
- **Automation:** target VMs with `Environment=Prod` for backup automation.
- **Compliance reporting.**

### Apply tags via CLI

```bash
az tag create --resource-id /subscriptions/.../resourceGroups/rg-1 \
  --tags Environment=Prod CostCenter=1234
```

---

## 18. Cost management

### Tools

| Tool | What it does |
|---|---|
| **Cost analysis** | Visualize current and forecasted spend |
| **Budgets** | Set spend thresholds at sub or RG, trigger alerts |
| **Cost alerts** | Email when budget threshold is hit |
| **Advisor recommendations** | Suggestions to save money (right-size VMs, delete unattached disks) |
| **Pricing calculator** | Estimate before deploying |
| **TCO calculator** | Compare on-prem cost vs Azure |

### Budgets

- Scope: **MG, subscription, RG, or by tag**.
- Reset period: monthly, quarterly, annually.
- Alerts at percentage thresholds (e.g., 50%, 80%, 100%, 110%).
- Actions: send email, trigger an action group (optional).
- Budgets **alert only** — they do **not** stop resources or block spending. Many candidates assume they do.

### Reservations

- Pre-pay for 1- or 3-year terms in exchange for big discounts (up to 72%) on VMs, SQL DB, etc.
- Out of scope for AZ-104 in detail, but know the concept.

---

## 19. Entra ID roles vs Azure RBAC roles — the most-confused topic

These two role systems coexist, and exam questions love to test the boundary.

| | **Microsoft Entra (directory) roles** | **Azure RBAC (resource) roles** |
|---|---|---|
| What they control | Identity & directory operations | Azure resources (VMs, storage, etc.) |
| Examples | Global Administrator, User Administrator, Application Administrator, Helpdesk Administrator | Owner, Contributor, Reader, VM Contributor |
| Scope | Tenant-wide (or admin units in P1+) | MG / Sub / RG / Resource |
| Where assigned | `Entra ID → Roles and administrators` | `Resource → Access control (IAM)` |
| Manages | Users, groups, app registrations, password resets, license assignment | Compute, storage, network, every Azure resource |

### Heuristic

- If the task is "reset a user's password" / "manage applications" / "create users" → **Entra role**.
- If the task is "create a VM" / "read a blob" / "manage a storage account" → **RBAC role**.

### Crossover example

A **Global Administrator** has full Entra ID power but **zero** Azure RBAC by default. To manage Azure resources, they must elevate access (see §11) to grant themselves User Access Administrator at root.

---

## 20. Hands-on lab walkthroughs

### Lab 1 — Build a tenant skeleton

**Goal:** create users, a group, assign a license, give the group RBAC at an RG.

```bash
# 1. Create resource group
az group create -n rg-az104-lab -l eastus

# 2. Create users
az ad user create --display-name "Aisha Khan" \
  --user-principal-name aisha@<yourtenant>.onmicrosoft.com \
  --password 'SecureP@ss123!' --force-change-password-next-sign-in true

az ad user create --display-name "Rahim Hossain" \
  --user-principal-name rahim@<yourtenant>.onmicrosoft.com \
  --password 'SecureP@ss123!' --force-change-password-next-sign-in true

# 3. Create security group
az ad group create --display-name "sg-engineering" --mail-nickname engineering

# 4. Add users to group
az ad group member add --group sg-engineering --member-id $(az ad user show --id aisha@<yourtenant>.onmicrosoft.com --query id -o tsv)

# 5. Assign Contributor at RG scope to the GROUP
GROUP_ID=$(az ad group show -g sg-engineering --query id -o tsv)
RG_ID=$(az group show -n rg-az104-lab --query id -o tsv)
az role assignment create --assignee $GROUP_ID --role "Contributor" --scope $RG_ID
```

**Verify:** `az role assignment list --scope $RG_ID -o table`

### Lab 2 — Custom role: VM start/stop only

```bash
cat > vm-operator.json <<'EOF'
{
  "Name": "VM Operator",
  "Description": "Start and stop VMs only.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "AssignableScopes": [
    "/subscriptions/<your-sub-id>"
  ]
}
EOF

az role definition create --role-definition vm-operator.json
az role assignment create --assignee aisha@<yourtenant>.onmicrosoft.com \
  --role "VM Operator" --scope /subscriptions/<your-sub-id>
```

### Lab 3 — Azure Policy: deny non-East-US deployments

Portal path: **Policy → Definitions → Search "Allowed locations" → Assign**.

Or CLI:

```bash
SUB=$(az account show --query id -o tsv)
az policy assignment create \
  --name 'allowed-locations' \
  --scope "/subscriptions/$SUB" \
  --policy "e56962a6-4747-49cd-b67b-bf8b01975c4c" \
  --params '{ "listOfAllowedLocations": { "value": ["eastus","westeurope"] } }'
```

Now try `az group create -n rg-test -l southindia` — it should be **denied**.

### Lab 4 — Resource lock

```bash
az lock create --name "no-delete" --resource-group rg-az104-lab \
  --lock-type CanNotDelete --notes "Production - do not delete"

# Try to delete - blocked
az group delete -n rg-az104-lab --yes  # fails
```

### Lab 5 — Tag inheritance via Policy

1. Create a tag on the RG: `az tag update --resource-id $RG_ID --operation Merge --tags Department=Engineering`
2. Assign built-in policy **"Inherit a tag from the resource group"** with parameter `tagName=Department` to the RG.
3. Create a new resource — tag automatically applied via Modify effect.

### Lab 6 — Budget with alert

Portal: `Subscription → Budgets → Add → Set $50/month, alert at 80%, email yourself`.

---

## 21. Exam question patterns and traps

The exam delivers 40–60 questions in ~150 minutes with multi-select, drag-and-drop, hotspot, case studies, and live performance-based labs. Below are the patterns that recur in the Identity & Governance domain.

### Pattern 1 — "What is the most efficient way?"

> "User X needs to be able to manage 50 VMs across 5 RGs. What is the most efficient way?"

Trap: assigning role per-VM or per-RG works but is not "most efficient". The **most efficient** answer is usually:
- Place all RGs in a **management group** OR assign at **subscription** scope.
- Assign the role **once** to a **group** that contains the user.

### Pattern 2 — "Least privilege"

> "User X needs to start and stop VMs. What is the least-privilege role?"

Don't pick Contributor — too broad. Don't pick Owner. Look for:
- **Virtual Machine Contributor** (still too broad — can delete)
- **Custom role** with only `start/restart/deallocate` actions ← usually correct
- Or for Linux/Windows sign-in: **Virtual Machine Administrator Login** / **User Login**

### Pattern 3 — "Owner needs to grant access"

> "Sara is a Contributor on the subscription. She needs to grant Bob access to the storage account."

Trap: Contributor cannot assign roles. Sara needs **User Access Administrator** added, **OR Owner**.

### Pattern 4 — "Global Admin can't create resources"

> "Maria is Global Administrator but cannot create VMs."

Answer: Global Administrator is an **Entra role**, not an Azure RBAC role. Maria must elevate access to root (see §11) and assign herself an Azure RBAC role like Owner or Contributor.

### Pattern 5 — "Lock prevents the operation"

> "An admin can no longer list storage account keys."

Look for a **ReadOnly lock** on the storage account or the parent RG. List-keys is a POST/write operation, blocked by ReadOnly.

### Pattern 6 — Policy effect choice

> "All future VMs must have a tag `Environment`. Existing VMs without it must show as non-compliant but not be modified."

- For **future** + **non-compliant report** → **Audit** effect on create/update.
- To **prevent creation** → **Deny**.
- To **add the tag automatically** → **Modify** (needs managed identity).
- To **add backup automatically if missing** → **DeployIfNotExists** (needs managed identity).

### Pattern 7 — Group membership type

> "A user is in the Engineering department. They should automatically be in the Engineering security group."

→ **Dynamic User** group with rule `(user.department -eq "Engineering")` (P1 license required).

### Pattern 8 — Guest user permissions

> "A vendor needs to view a Power BI report. What is the most secure approach?"

→ Invite as **B2B guest user** with the appropriate Reader role at the report scope. Don't create them as a member.

### Pattern 9 — SSPR + writeback

> "Hybrid users need to reset their password from any device, and the change must reach on-prem AD."

→ Enable SSPR + **password writeback** + Entra Connect / Cloud Sync configured. **Requires P1.**

### Pattern 10 — Hot-spot (drag-drop) on scope hierarchy

You'll get a 4-box drag-drop: arrange `Resource group, Resource, Management group, Subscription` from broadest to narrowest. Answer: MG → Subscription → RG → Resource.

### Pattern 11 — Performance-based lab

You may need to live-perform: "Create a user named Aisha, add her to a group named Sales, assign Reader at the RG `rg-prod`."

Practice the **portal click path** for: create user, create group, add member, IAM → Add role assignment. The lab grader checks the resulting state, not how you got there. Both portal and CLI work.

### Time management

- ~2.5 minutes per question average.
- Performance-based labs eat 15–25 minutes — don't burn time on a single MCQ early.
- The exam is **open-book** — you can search Microsoft Docs during the test. Use this for syntax-heavy questions, but don't waste minutes searching obvious facts.
- Flag and skip anything taking more than 3 minutes.

---

## 22. Cheat sheet

> **One-page distilled reference. Print this.**

### The RBAC formula

> **Role assignment = Security principal + Role definition + Scope**
>
> Scope hierarchy (broad → narrow): **Management Group → Subscription → Resource Group → Resource**

### Top 6 built-in roles

| Role | Notes |
|---|---|
| Owner | Full control + can grant access |
| Contributor | Full control, **cannot grant access** |
| Reader | View only |
| User Access Administrator | Manage access only |
| Storage Blob Data Contributor | **Data plane** for blobs (≠ Storage Account Contributor) |
| Virtual Machine Contributor | Manage VMs, not network or storage |

### Entra roles vs RBAC roles

| Task | Use |
|---|---|
| Reset user passwords, manage apps, license users | **Entra role** (e.g., User Administrator) |
| Create/manage Azure resources | **Azure RBAC role** |

Global Admin ≠ Owner. Global Admin must elevate to root before getting Azure RBAC.

### Entra ID editions quick pick

| Need | Edition |
|---|---|
| Conditional Access, dynamic groups, SSPR with writeback | **P1** |
| PIM, Identity Protection, access reviews | **P2** |
| Basic free tier | Free |

### Group rules

- **Security group** for RBAC.
- **M365 group** for collaboration; cannot be nested.
- **Dynamic** = rule-based (P1+); cannot mix users + devices in one group.

### SSPR essentials

- Three states: **None / Selected / All**.
- Default 1 method, recommended 2.
- **Password writeback** = hybrid back to on-prem AD (needs **P1**).

### Custom role JSON keys

`Name`, `Description`, `Actions`, `NotActions`, `DataActions`, `NotDataActions`, `AssignableScopes`.

### Azure Policy effects

| Effect | Use |
|---|---|
| Deny | Block at create/update |
| Audit | Report only |
| AuditIfNotExists | Conditional audit |
| DeployIfNotExists | Auto-remediate (needs MI) |
| Modify | Add/change tag or property (needs MI) |
| Append | Add fields |
| Disabled | Off |

> Policy **enforces resource shape**. RBAC **controls who acts**. They complement each other.

### Locks

| Lock | Blocks |
|---|---|
| CanNotDelete | Delete |
| ReadOnly | Any write + delete (incl. list keys, restart VM) |

Inherit downward; most restrictive wins.

### Tags

- Up to 50 per resource. Names ≤ 512 chars, values ≤ 256.
- **Not inherited by default** — use Policy "Inherit a tag from the resource group".

### Subscription / MG facts

- Subscription trusts **one** Entra tenant.
- Management groups can nest **6 levels deep**.
- Tenant Root Group sits above all MGs.

### Budgets

- Scope: MG / Sub / RG / by tag.
- **Alerts only — never block spending.**

### Bulk operations and limits

- Soft-deleted users restorable for **30 days**.
- Custom roles per tenant: **5,000**.
- Policy compliance scan: every **24 hours** automatically.

### Common CLI snippets

```bash
# List role assignments
az role assignment list --scope /subscriptions/<id> -o table

# Add a user to group
az ad group member add --group <name> --member-id <user-object-id>

# Assign role to group at RG
az role assignment create --assignee <group-id> --role "Contributor" \
  --scope /subscriptions/<id>/resourceGroups/<rg>

# Check policy compliance
az policy state list --resource-group <rg> -o table

# Create a CanNotDelete lock
az lock create --name no-del --resource-group <rg> --lock-type CanNotDelete
```

### PowerShell equivalents

```powershell
# List role assignments
Get-AzRoleAssignment -Scope "/subscriptions/<id>"

# Assign role
New-AzRoleAssignment -ObjectId <id> -RoleDefinitionName "Contributor" `
  -ResourceGroupName "rg-prod"

# Check user effective access
Get-AzRoleAssignment -SignInName user@contoso.com -ExpandPrincipalGroups
```

### Exam tactical reminders

1. **Read every word** — "least privilege", "most efficient", "most secure" change the answer.
2. **Eliminate** Owner first when least privilege is asked.
3. If the answer is **"impossible"** (e.g., dynamic group with both users and devices), it might be a trap option — and might be correct.
4. **Group + role at scope** beats individual user assignments — almost always.
5. When stuck on a syntax question, **search Microsoft Docs** in the open-book exam.
6. **Flag and move on** — don't lose 5 minutes on one MCQ.
7. **Save 25+ minutes** for performance-based lab tasks. Do them last.
8. Pass mark: **700/1000**. You don't need perfection — you need consistency across all five domains.

---

*Good luck. Now go to your Azure free account and break things until everything in this guide feels obvious.*
