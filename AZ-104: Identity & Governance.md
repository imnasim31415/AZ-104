# AZ-104: Identity & Governance — Complete Study Guide

> Domain weight: ~20-25% of the exam. This is one of the highest-weighted domains, so know it cold.

---

## 1. Azure AD / Microsoft Entra ID Fundamentals

**Important naming note:** Azure Active Directory was renamed **Microsoft Entra ID**. The exam may use either name — treat them as the same service.

### Key Concepts
- **Tenant**: A dedicated, isolated instance of Entra ID created when you sign up for an Azure/Microsoft 365 service. One organization = usually one tenant (though multiple are possible).
- **Directory**: Synonymous with tenant — holds users, groups, apps, devices.
- A single Azure subscription trusts **exactly one** Entra ID tenant at a time, but one tenant can be associated with **multiple subscriptions**.

### Entra ID Editions
| Edition | Key Features |
|---|---|
| Free | Basic user/group management, on-prem sync |
| Microsoft 365 Apps | Adds some premium features for M365 users |
| Premium P1 | Dynamic groups, group-based access, Hybrid Join, Conditional Access, SSPR with on-prem writeback |
| Premium P2 | Everything in P1 + **Identity Protection**, **Privileged Identity Management (PIM)**, Access Reviews |

**Common exam trap:** PIM and Identity Protection require **Premium P2** — a very frequently tested fact.

### Users
- **Cloud-only identity**: Created directly in Entra ID.
- **Directory-synchronized identity**: Synced from on-prem AD via **Azure AD Connect / Entra Connect (Cloud Sync)**.
- **Guest users (B2B)**: Invited from external tenants/organizations; shown as `#EXT#` in UPN.
- Bulk user creation is done via **CSV upload** in the portal or `New-AzADUser` / `az ad user create` in PowerShell/CLI.

### Groups
| Type | Description |
|---|---|
| **Security group** | Used to assign permissions/access to resources |
| **Microsoft 365 group** | Used for collaboration (shared mailbox, calendar, files) |
| **Assigned membership** | Admin manually adds/removes members |
| **Dynamic membership (Users or Devices)** | Rule-based auto membership — **requires Premium P1** |

**Common exam trap:** Dynamic group membership rules only work for **security groups** and **Microsoft 365 groups** — and require **Premium P1 or P2** licensing.

### Administrative Units (AUs)
- Used to **delegate admin permissions** over a subset of users/groups/devices (e.g., a regional IT admin who can only manage users in their region).
- Scopes role assignments to a smaller portion of the directory instead of the whole tenant.

### External Identities
- **B2B collaboration**: Invite external users as guests into your tenant.
- **B2B direct connect**: Mutual trust between two Entra tenants (e.g., Teams shared channels).
- **B2C**: Customer-facing identity for consumer apps (separate product, mostly historical on the exam now, often replaced by "External ID").

### Self-Service Password Reset (SSPR)
- Lets users reset their own password without helpdesk.
- Requires configuring **authentication methods** (phone, email, security questions, Authenticator app).
- **On-prem password writeback** requires Premium P1.

### Common Exam Questions — Section 1
1. *"You need users to be automatically added to a group based on department attribute — what do you configure?"* → **Dynamic group membership rule** (needs P1).
2. *"Which license is required for PIM?"* → **Entra ID Premium P2**.
3. *"How do you delegate password reset rights for only the Sales department users?"* → **Administrative Unit** + scoped role assignment.
4. *"An external partner needs temporary access to a SharePoint site."* → **B2B guest invite**.

---

## 2. Authentication & Security

### Multi-Factor Authentication (MFA)
- Adds a second verification factor: SMS, call, Authenticator app (push/OTP), FIDO2 key.
- Can be enabled via:
  - **Security defaults** (free, simple, all-or-nothing, no granularity)
  - **Conditional Access policies** (granular, requires Premium P1)
  - Per-user MFA (legacy method, rarely correct answer now)

**Common exam trap:** You **cannot** enable both **Security Defaults** and **Conditional Access MFA policies** at the same time — Security Defaults must be turned off to use Conditional Access.

### Conditional Access (CA)
- **Requires Premium P1.**
- If-then policy engine: *if* (user, location, device, app, risk) *then* (grant/block, require MFA, require compliant device, etc.)
- Common signals: user/group, cloud app, location (IP ranges/named locations), device platform, client app, **sign-in risk** (needs Identity Protection/P2).
- Common controls: Require MFA, require compliant device (Intune), require hybrid Azure AD joined device, block access, require approved client app.

### Identity Protection
- **Requires Premium P2.**
- Detects **sign-in risk** (e.g., impossible travel, anonymous IP) and **user risk** (leaked credentials).
- Feeds into Conditional Access policies (risk-based CA).

### Privileged Identity Management (PIM)
- **Requires Premium P2.**
- Provides **Just-In-Time (JIT)** privileged access: users are **eligible** for a role and must **activate** it (with optional MFA/approval/justification) for a time-boxed period, rather than having it permanently assigned.
- Key terms: **Eligible assignment** vs **Active assignment**, **Activation**, **Access Reviews**.
- Applies to both **Entra ID roles** (e.g., Global Administrator) and **Azure resource roles** (e.g., Owner on a subscription).

### Common Exam Questions — Section 2
1. *"You want to reduce standing access to the Global Administrator role."* → **PIM eligible assignment**.
2. *"Block sign-ins from a specific country unless MFA is satisfied."* → **Conditional Access policy** with location condition.
3. *"Enable baseline security quickly at no extra cost."* → **Security Defaults**.
4. *"Detect and respond to leaked credentials automatically."* → **Identity Protection** (user risk policy).

---

## 3. Role-Based Access Control (RBAC)

### Core Concept
RBAC answers: **"Who can do what, on which resource."**
An RBAC role assignment = **Security principal** + **Role definition** + **Scope**.

### Security Principals
- User, Group, **Service Principal** (identity for an app), **Managed Identity** (auto-managed credential for Azure resources).

### Scope Hierarchy (broadest → narrowest)
```
Management Group
   └── Subscription
        └── Resource Group
             └── Resource
```
- Permissions are **inherited downward**. A role assigned at Management Group level flows down to all subscriptions, resource groups, and resources inside it.

### Built-in Roles (memorize these!)
| Role | Can manage access? | Can manage resources? |
|---|---|---|
| **Owner** | ✅ Yes (full access + assign roles) | ✅ Yes |
| **Contributor** | ❌ No | ✅ Yes (all resource types) |
| **Reader** | ❌ No | 👁 View only |
| **User Access Administrator** | ✅ Yes (only manages access, not resources) | ❌ No |

**Common exam trap:** Contributor can create/manage almost everything **except** grant access to others. Only **Owner** and **User Access Administrator** can assign roles.

### Custom Roles
- Created when built-in roles don't fit. Defined in JSON with `Actions`, `NotActions`, `DataActions`, `NotDataActions`, and `AssignableScopes`.
- `NotActions` **subtracts** from `Actions` — it does not deny at a security level, it just narrows what's granted (important distinction from Deny assignments).

### RBAC vs Azure Policy vs Deny Assignments
| Feature | Purpose |
|---|---|
| **RBAC** | Controls **what actions a user CAN perform** |
| **Azure Policy** | Controls **what resources/configurations are allowed to exist**, regardless of who creates them |
| **Deny Assignment** | Explicitly **blocks** an action even if RBAC grants it (mostly system-generated, e.g., by Blueprints/Managed Apps) |

### Common Exam Questions — Section 3
1. *"A user should be able to deploy VMs but not grant others access."* → **Contributor** role.
2. *"A user needs to manage access for a resource group but not modify resources."* → **User Access Administrator**.
3. *"Where should you assign a role so it applies to all subscriptions under a department?"* → **Management Group** level.
4. *"An app running on a VM needs to access Key Vault without storing credentials."* → **Managed Identity** + RBAC role assignment on Key Vault.
5. *"Roles assigned at parent scope — can child scope override with a Deny?"* → No, standard RBAC is additive only (no deny via roles); use Azure Policy or Deny Assignments instead.

---

## 4. Governance: Management Groups, Subscriptions, Resources

### Hierarchy Limits (know the numbers!)
- Up to **6 levels** of management groups (not counting the Root and subscription level).
- A management group or subscription can have only **one parent**.
- Root management group cannot be moved or deleted; created automatically.

### Subscriptions
- Billing boundary + scale unit for resource limits/quotas.
- Moving a subscription between management groups **does not** change resource group/resource locations — it's purely an org/governance change.
- Changing the **Entra ID tenant** associated with a subscription **removes all RBAC assignments** on that subscription (a very common gotcha question).

### Resource Groups
- Logical container; a resource can belong to **only one resource group**.
- Resources **can be moved** between resource groups and even subscriptions (with some service restrictions — check "move limitations" for the resource type if asked).
- Resource group **location** = where its metadata is stored, not necessarily where resources live.
- Deleting a resource group deletes **all resources inside it**.

### Common Exam Questions — Section 4
1. *"How many levels of management groups can you nest?"* → **Up to 6**.
2. *"What happens to RBAC assignments if you move a subscription to a different Entra tenant?"* → **They are all removed** and must be recreated.
3. *"Can a resource group span two subscriptions?"* → **No.**

---

## 5. Azure Policy

### Purpose
Enforces **organizational standards** and assesses **compliance** at scale — e.g., "only allow specific VM SKUs," "require a tag on all resources," "restrict allowed regions."

### Key Components
- **Policy definition**: The rule (JSON) — condition + effect.
- **Initiative (Policy Set)**: A group of policy definitions bundled together for a common goal (e.g., "ISO 27001 compliance").
- **Assignment**: Applying a definition/initiative to a scope (management group, subscription, resource group).
- **Exclusions**: Exempt a narrower scope from a broader assignment.
- **Exemptions**: Formally exempt a resource from a policy with an expiration date and reason.

### Policy Effects (memorize!)
| Effect | Behavior |
|---|---|
| **Deny** | Blocks the resource creation/update |
| **Audit** | Allows but flags as non-compliant in reports |
| **Append** | Adds fields (e.g., tags) to the request before creation |
| **Modify** | Adds/updates/removes properties on existing resources (needs managed identity) |
| **DeployIfNotExists (DINE)** | Deploys a related resource if a condition isn't met (needs managed identity) |
| **Disabled** | Turns off the policy without deleting it |

**Common exam trap:** `DeployIfNotExists` and `Modify` effects require a **managed identity** with sufficient RBAC permissions assigned to the policy assignment itself.

### Policy vs RBAC (repeat — heavily tested)
- RBAC = "Can this **person** do this action?"
- Policy = "Is this **resource configuration** allowed to exist?"
- They work **together**, not as substitutes.

### Common Exam Questions — Section 5
1. *"Prevent creation of VMs outside East US and West Europe."* → **Azure Policy** with `allowed locations` definition, Deny effect.
2. *"Ensure a cost-center tag exists on all new resource groups, auto-adding if missing."* → Policy with **Append** or **Modify** effect.
3. *"Report on resources that don't use approved SKUs, without blocking them."* → **Audit** effect.
4. *"Temporarily allow one resource to bypass an org-wide policy with a documented reason."* → **Exemption**.

---

## 6. Resource Locks & Tags

### Resource Locks
| Lock Type | Effect |
|---|---|
| **CanNotDelete (Delete lock)** | Resource can be read/modified but not deleted |
| **ReadOnly** | Resource cannot be modified or deleted — effectively frozen |

- Locks apply to the resource and **inherit down** through the hierarchy (Management Group → Subscription → RG → Resource).
- **Locks override RBAC permissions** — even an Owner cannot delete a locked resource until the lock is removed.
- Locking a **Storage Account** with ReadOnly can break apps because it also blocks things like key rotation/listing keys, since those are technically write operations against control plane in some cases.

### Tags
- Key-value pairs for **organizing, cost tracking, and automation** — NOT for access control.
- Apply to most resource types (some exceptions, e.g., classic resources).
- Can be applied/enforced via **Azure Policy** (Append/Modify effects).
- Useful for **cost management reports**, filtering in Azure Monitor, and automation scripts.

### Common Exam Questions — Section 6
1. *"Prevent accidental deletion of a production VNet but still allow config changes."* → **CanNotDelete lock**.
2. *"An Owner cannot delete a resource — why?"* → A **ReadOnly or CanNotDelete lock** is applied (locks override RBAC).
3. *"How do you track costs per department across resources?"* → **Tags** (e.g., `CostCenter`, `Department`).

---

## 7. Azure AD Connect / Hybrid Identity

- **Password Hash Sync (PHS)**: Syncs a hash of the password hash to Entra ID. Simplest, most common, supports leaked-credential detection.
- **Pass-through Authentication (PTA)**: Validates passwords against on-prem AD directly via lightweight agents — password never leaves on-prem.
- **Federation (AD FS)**: On-prem AD FS servers handle authentication; most complex, used for advanced scenarios (smart card auth, etc.).
- **Seamless SSO**: Pairs with PHS or PTA to auto-sign-in domain-joined users without re-prompting.

**Common exam trap:** Know which method to pick when the requirement is "passwords must never leave the on-prem network" → **Pass-through Authentication**.

### Common Exam Questions — Section 7
1. *"Requirement: authentication must occur on-premises, no password hashes stored in the cloud."* → **Pass-through Authentication**.
2. *"Simplest method with least infrastructure that still enables Identity Protection leaked-credential checks."* → **Password Hash Sync**.

---

## 8. Quick-Reference Cheat Sheet

- **PIM, Identity Protection** → require **Entra ID Premium P2**
- **Conditional Access, Dynamic Groups, SSPR writeback** → require **Entra ID Premium P1**
- **Owner** = full control + can assign roles
- **Contributor** = full control, **cannot** assign roles
- **User Access Administrator** = only manages access, not resources
- **RBAC** = controls actions by identities; **Policy** = controls resource configuration/compliance
- **Locks override RBAC** — always check for locks if a "permission denied on delete" question appears
- **Moving subscription to new tenant** → wipes RBAC assignments
- Management group nesting → **max 6 levels**
- **DeployIfNotExists / Modify** policy effects → need a **managed identity**
- Tags = organization/cost, **never** access control

---

## 9. Suggested Hands-On Practice

1. Create a user + a dynamic security group based on a department attribute.
2. Set up a Conditional Access policy requiring MFA for admins only.
3. Assign a custom RBAC role at a resource group scope and test inheritance.
4. Create an Azure Policy that enforces an allowed-locations restriction, then test deployment in a disallowed region.
5. Apply a `CanNotDelete` lock on a resource group and try deleting a resource inside it as an Owner.
6. Move a subscription between two management groups and verify policy inheritance changes.

---

*Good luck with your AZ-104 exam! Review this guide alongside Microsoft Learn's official AZ-104 learning paths for the most current portal screenshots and any recent feature updates.*
