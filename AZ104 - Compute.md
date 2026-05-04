# AZ-104 — Deploy and Manage Azure Compute Resources: Deep Dive Guide

> **Domain weight:** 20–25% (joint highest with Identity & Governance)
> **Expected exam questions:** ~10–15
> **Why it matters:** Compute is the broadest domain on the exam. It spans four distinct sub-topics — ARM/Bicep IaC, Virtual Machines, Containers, and App Service — and is the second hardest domain after Networking due to sheer breadth. Questions here blend scenario reasoning, syntax interpretation, and live portal tasks.

---

## Table of contents

1. [The compute service decision tree](#1-the-compute-service-decision-tree)
2. [ARM templates — structure and key concepts](#2-arm-templates--structure-and-key-concepts)
3. [Bicep — the modern IaC syntax](#3-bicep--the-modern-iac-syntax)
4. [Deploying templates — modes, commands, scopes](#4-deploying-templates--modes-commands-scopes)
5. [Virtual machines — creation and configuration](#5-virtual-machines--creation-and-configuration)
6. [VM sizes and series](#6-vm-sizes-and-series)
7. [Managed disks](#7-managed-disks)
8. [Azure Disk Encryption](#8-azure-disk-encryption)
9. [Moving a VM across resource group, subscription, region](#9-moving-a-vm-across-resource-group-subscription-region)
10. [Availability sets](#10-availability-sets)
11. [Availability zones](#11-availability-zones)
12. [VM Scale Sets (VMSS)](#12-vm-scale-sets-vmss)
13. [Azure Container Registry (ACR)](#13-azure-container-registry-acr)
14. [Azure Container Instances (ACI)](#14-azure-container-instances-aci)
15. [Azure Container Apps](#15-azure-container-apps)
16. [Container sizing and scaling](#16-container-sizing-and-scaling)
17. [App Service plans](#17-app-service-plans)
18. [App Service — web apps](#18-app-service--web-apps)
19. [Deployment slots](#19-deployment-slots)
20. [App Service networking](#20-app-service-networking)
21. [App Service backup and restore](#21-app-service-backup-and-restore)
22. [Hands-on lab walkthroughs](#22-hands-on-lab-walkthroughs)
23. [Exam question patterns and traps](#23-exam-question-patterns-and-traps)
24. [Cheat sheet](#24-cheat-sheet)

---

## 1. The compute service decision tree

Before diving into individual services, internalize this decision matrix. The exam loves "which service should you use?" questions.

```
Need a full OS you manage yourself?
  └─ Yes → Azure Virtual Machine (IaaS)

Need to run a web app / API without managing OS?
  └─ Yes → Azure App Service (PaaS)

Need to run a container (short-lived or simple)?
  └─ Yes → Azure Container Instances (ACI) — no orchestration
             └─ unless it needs scaling, ingress, revisions → Azure Container Apps

Need Kubernetes with full control?
  └─ Yes → Azure Kubernetes Service (AKS) — not in AZ-104 scope in depth

Need many identical VMs that scale automatically?
  └─ Yes → VM Scale Sets (VMSS)
```

### IaaS vs PaaS summary

| | Azure VM | App Service | ACI | Container Apps |
|---|---|---|---|---|
| Type | IaaS | PaaS | PaaS | PaaS |
| OS managed by | You | Microsoft | Microsoft | Microsoft |
| Scaling | Manual / VMSS | Plan-based / autoscale | Manual (restart) | Automatic (KEDA) |
| Use case | Any workload | Web apps / APIs | Simple/burst containers | Microservices, event-driven |
| AWS equivalent | EC2 | Elastic Beanstalk | Fargate (simple) | Fargate + ECS |

---

## 2. ARM templates — structure and key concepts

Azure Resource Manager (ARM) templates are JSON files that declaratively define Azure resources. ARM is the **management layer** — every action (portal, CLI, PowerShell, REST) goes through ARM.

### ARM template anatomy

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": { },
  "variables": { },
  "functions": [ ],
  "resources": [ ],
  "outputs": { }
}
```

| Section | Purpose |
|---|---|
| `$schema` | Template version — tells ARM which schema to validate against |
| `contentVersion` | Your versioning string — not enforced, just informational |
| `parameters` | Values supplied at deploy time (environment-specific, secrets) |
| `variables` | Computed values reused across the template |
| `functions` | User-defined template functions (rarely used) |
| `resources` | The actual resources to deploy — **required** |
| `outputs` | Values returned after deployment (e.g., public IP, connection string) |

### Parameters section

```json
"parameters": {
  "vmName": {
    "type": "string",
    "defaultValue": "myVM",
    "minLength": 1,
    "maxLength": 64,
    "metadata": { "description": "Name of the virtual machine" }
  },
  "adminPassword": {
    "type": "securestring"
  },
  "environment": {
    "type": "string",
    "allowedValues": ["dev", "staging", "prod"]
  }
}
```

Parameter types: `string`, `securestring`, `int`, `bool`, `object`, `secureObject`, `array`.

Use `securestring` for passwords and keys — they are masked in deployment logs.

### Variables

```json
"variables": {
  "nicName": "[concat(parameters('vmName'), '-nic')]",
  "location": "[resourceGroup().location]"
}
```

Reference with `[variables('nicName')]`.

### Resources section — key fields

```json
"resources": [
  {
    "type": "Microsoft.Compute/virtualMachines",
    "apiVersion": "2023-03-01",
    "name": "[parameters('vmName')]",
    "location": "[variables('location')]",
    "dependsOn": [
      "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
    ],
    "properties": { }
  }
]
```

| Field | Notes |
|---|---|
| `type` | Resource provider + type (e.g., `Microsoft.Storage/storageAccounts`) |
| `apiVersion` | Date-versioned API — each resource type has its own |
| `name` | Resource name |
| `location` | Azure region |
| `dependsOn` | Explicit dependency — ARM waits for listed resources before deploying this one |
| `properties` | Resource-specific configuration |

### dependsOn — the most-tested ARM concept

ARM deploys resources in **parallel by default**. Use `dependsOn` or implicit references to enforce order.

```json
"dependsOn": [
  "[resourceId('Microsoft.Network/networkInterfaces', 'myNIC')]"
]
```

**Exam pattern:** "The storage account must exist before the VM is created." → Add `dependsOn` referencing the storage account to the VM resource.

**Implicit dependency** (preferred): if you reference a resource using `reference(...)` inside another resource's properties, ARM infers the dependency automatically — no explicit `dependsOn` needed.

### Outputs section

```json
"outputs": {
  "publicIPAddress": {
    "type": "string",
    "value": "[reference(resourceId('Microsoft.Network/publicIPAddresses', 'myPublicIP')).ipAddress]"
  }
}
```

### Template functions (commonly tested)

| Function | Usage |
|---|---|
| `concat()` | Join strings: `[concat('vm-', parameters('env'))]` |
| `resourceGroup()` | Current RG — `.location`, `.id`, `.name` |
| `subscription()` | Current sub — `.subscriptionId`, `.tenantId` |
| `resourceId()` | Get resource ID |
| `reference()` | Get runtime properties of a deployed resource |
| `parameters()` | Read a parameter value |
| `variables()` | Read a variable value |
| `uniqueString()` | Deterministic hash for unique names |
| `if()` | Conditional: `[if(equals(parameters('env'), 'prod'), 'Standard', 'Free')]` |

### Deployment modes

| Mode | Behavior |
|---|---|
| **Incremental** (default) | Resources in template are added or updated. Resources NOT in template are left alone. |
| **Complete** | Resources in template are deployed. Resources NOT in template are **deleted** from the RG. |

**Exam trap:** Complete mode will delete resources in the RG that are not in the template. Use with caution in production.

---

## 3. Bicep — the modern IaC syntax

Bicep is a domain-specific language (DSL) that compiles to ARM JSON. It is not a separate service — it is a transpiler. Every Bicep file becomes ARM JSON before it reaches Azure.

### Why Bicep over ARM JSON?

- Simpler syntax — roughly 30% less code for the same template
- Type checking and IDE support
- No need for `dependsOn` in most cases — inferred automatically
- Native support in Azure CLI

### Bicep file structure

```bicep
// Parameters
param vmName string = 'myVM'
param location string = resourceGroup().location

@secure()
param adminPassword string

@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

// Variables
var nicName = '${vmName}-nic'

// Resources
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'sa${uniqueString(resourceGroup().id)}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource vm 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: vmName
  location: location
  properties: {
    hardwareProfile: { vmSize: 'Standard_D2s_v5' }
    osProfile: {
      computerName: vmName
      adminUsername: 'azureuser'
      adminPassword: adminPassword
    }
    storageProfile: {
      imageReference: {
        publisher: 'Canonical'
        offer: '0001-com-ubuntu-server-focal'
        sku: '20_04-lts'
        version: 'latest'
      }
      osDisk: { createOption: 'FromImage' }
    }
    networkProfile: { networkInterfaces: [] }
  }
}

// Outputs
output storageEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### Bicep decorators (exam-relevant)

```bicep
@description('VM name — must be 1-64 chars')
@minLength(1)
@maxLength(64)
param vmName string

@secure()
param adminPassword string

@allowed(['Standard_LRS', 'Standard_GRS', 'Premium_LRS'])
param storageSku string = 'Standard_LRS'
```

### Conditions in Bicep

```bicep
param deployRedis bool = false

resource redis 'Microsoft.Cache/redis@2023-04-01' = if (deployRedis) {
  name: 'myRedis'
  location: location
  properties: { sku: { name: 'Basic', family: 'C', capacity: 0 } }
}
```

### Loops in Bicep

```bicep
param locations array = ['eastus', 'westeurope']

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for loc in locations: {
  name: 'sa${uniqueString(loc)}'
  location: loc
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}]
```

### ARM to Bicep conversion

```bash
# ARM JSON → Bicep (decompile)
az bicep decompile --file azuredeploy.json

# Bicep → ARM JSON (compile)
az bicep build --file main.bicep
```

**Exam tip:** You do not need to write Bicep from scratch on the exam, but you must read a Bicep file, identify what it deploys, and know how to add a parameter or change a property.

---

## 4. Deploying templates — modes, commands, scopes

### Deployment scopes

| Scope | CLI command |
|---|---|
| Resource group | `az deployment group create` |
| Subscription | `az deployment sub create` |
| Management group | `az deployment mg create` |
| Tenant | `az deployment tenant create` |

Most exam scenarios use resource group scope.

### CLI deployment

```bash
# Deploy ARM template
az deployment group create \
  --resource-group rg-prod \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json \
  --mode Incremental

# Deploy Bicep file
az deployment group create \
  --resource-group rg-prod \
  --template-file main.bicep \
  --parameters vmName=myVM adminPassword='P@ssw0rd!'

# What-if dry run — shows what would change without applying
az deployment group what-if \
  --resource-group rg-prod \
  --template-file main.bicep
```

### PowerShell deployment

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName "rg-prod" `
  -TemplateFile "azuredeploy.json" `
  -TemplateParameterFile "azuredeploy.parameters.json" `
  -Mode Incremental
```

### Exporting a template

- Go to any Resource Group → **Export template** — exports the current state as ARM JSON.
- Exported templates often need cleanup (hardcoded values, missing dependencies).

---

## 5. Virtual machines — creation and configuration

### VM creation checklist

1. **Basics:** Subscription, RG, VM name, region, availability option (zone/set/none), image, size, admin credentials
2. **Disks:** OS disk type, data disks, encryption
3. **Networking:** VNet, subnet, public IP, NSG
4. **Management:** Auto-shutdown, backup, monitoring
5. **Advanced:** Cloud-init / custom data, extensions

### Stop vs. Deallocate — very commonly tested

| Action | Compute billing | IP retained | State |
|---|---|---|---|
| **Stop (OS shutdown)** | **Still billed** | Yes | Stopped |
| **Deallocate** | **Not billed** | Public dynamic IP released | Deallocated |
| **Start** | Billed again | New dynamic IP if released | Running |

**Exam trap:** Stopping a VM from inside the OS does NOT deallocate it. You must deallocate via the portal, CLI, or PowerShell to stop billing.

```bash
az vm deallocate --resource-group rg-prod --name myVM
az vm start     --resource-group rg-prod --name myVM
az vm restart   --resource-group rg-prod --name myVM
```

### Resizing a VM

- Vertical scaling — change to a larger or smaller SKU.
- **Causes a reboot** — plan for downtime.
- If resizing within the same availability set, all VMs in the set must be deallocated if the target size is not on the current hardware cluster.

```bash
az vm resize --resource-group rg-prod --name myVM --size Standard_D4s_v5
```

### Custom script extensions

Run scripts on a VM after deployment:

```bash
az vm extension set \
  --resource-group rg-prod \
  --vm-name myVM \
  --name CustomScriptExtension \
  --publisher Microsoft.Compute \
  --settings '{"commandToExecute": "apt-get install -y nginx"}'
```

---

## 6. VM sizes and series

### Size naming convention

`Standard_D4s_v5` = `[Tier]_[Family][vCPUs][Features]_[Version]`

| Component | Meaning |
|---|---|
| `Standard` | Tier (Basic is legacy, Standard is current) |
| `D` | Family (general purpose) |
| `4` | Number of vCPUs |
| `s` | Feature: supports Premium SSD |
| `v5` | Version of hardware generation |

### Common families

| Family | Optimized for | Examples |
|---|---|---|
| **B** | Burstable (cheap, intermittent workloads) | B1s, B2ms |
| **D** | General purpose — balanced CPU/memory | D2s_v5, D4s_v5 |
| **E** | Memory optimized | E4s_v5 |
| **F** | Compute optimized | F4s_v2 |
| **L** | Storage optimized (high disk throughput) | L8s_v3 |
| **N** | GPU (ML, graphics) | NC6, NV6 |
| **M** | Massive memory (SAP HANA) | M128ms |

Know the families and use cases — "which series is best for a memory-intensive database?" → **E-series**.

---

## 7. Managed disks

### Disk roles

| Disk | Purpose | Notes |
|---|---|---|
| **OS disk** | Boots the OS | Created from image, always one per VM |
| **Temporary disk** | Ephemeral local storage | **Data is lost on reboot/resize** — never store persistent data here |
| **Data disk** | Persistent data storage | Multiple allowed, attach/detach at runtime |

### Disk SKUs (performance order)

Standard HDD < Standard SSD < **Premium SSD** < Premium SSD v2 < Ultra Disk

| SKU | Use case |
|---|---|
| Standard HDD | Dev/test, infrequent access |
| Standard SSD | Light production |
| Premium SSD | Production, I/O-intensive apps |
| Ultra Disk | Sub-millisecond latency databases |

The `s` in the VM size name (e.g., D4**s**_v5) indicates Premium SSD support. A VM without `s` cannot use Premium SSD.

### Snapshots

Point-in-time read-only copy of a disk. Used for backup before risky changes, or to copy a disk to another region.

```bash
az snapshot create \
  --resource-group rg-prod \
  --name mySnapshot \
  --source /subscriptions/.../disks/myOSDisk
```

### Ephemeral OS disk

- OS disk stored on the VM host's local SSD (temporary storage).
- Fastest OS disk performance, no extra cost.
- Data is **lost on reboot/reimage** — fine for stateless VMs.
- Not all VM sizes support it.

---

## 8. Azure Disk Encryption

### Three encryption approaches

| Approach | How | Keys |
|---|---|---|
| **Server-Side Encryption (SSE)** | Always-on, platform-level | Platform-Managed (default) or Customer-Managed (CMK) |
| **Encryption at host** | Encrypts temp disk and disk cache at host level | Platform or CMK |
| **Azure Disk Encryption (ADE)** | Inside the VM — BitLocker (Windows) / DM-Crypt (Linux) | Stored in Key Vault, you control the KEK |

### When to use ADE

- Compliance regulations requiring OS-level encryption.
- Need to encrypt OS and data disks inside the VM guest.
- Keys must be in a customer-controlled Key Vault.

```bash
az vm encryption enable \
  --resource-group rg-prod \
  --name myVM \
  --disk-encryption-keyvault myKeyVault
```

### Exam quick pick

| Requirement | Use |
|---|---|
| Default, everything automatic | SSE with Platform-Managed Keys |
| Customer-managed keys, no OS encryption | SSE with CMK |
| OS + data disk encrypted inside VM, compliance | **ADE** |
| Temp disk encryption | Encryption at host |

---

## 9. Moving a VM across resource group, subscription, region

### Move to another resource group or subscription

- Move the VM, NIC, public IP, OS disk, data disks, and VNet together.
- No downtime — VM keeps running.
- The resource ID changes after the move.

```bash
az resource move \
  --destination-group rg-new \
  --ids /subscriptions/.../Microsoft.Compute/virtualMachines/myVM
```

### Move to another region

Not a direct native operation. Options:

1. **Azure Resource Mover** — GUI tool that orchestrates cross-region moves.
2. Manual: snapshot → copy to target region → create disk → create VM.
3. **Azure Site Recovery** — supports cross-region VM migration.

### What cannot be moved (exam traps)

- Classic (v1) VMs and resources.
- VMs with ADE enabled — remove encryption first.
- VMs with active Azure Backup — suspend backup first.
- VMs in a proximity placement group — complex restrictions apply.

---

## 10. Availability sets

### What they protect against

Hardware failures within a single datacenter (rack, power unit). They do NOT protect against datacenter or zone failures.

### Fault domains and update domains

| Concept | Protects against | Default count |
|---|---|---|
| **Fault domain (FD)** | Shared power/network failure — a "rack" | 2 (max 3) |
| **Update domain (UD)** | Simultaneous patching by Azure | 5 (max 20) |

VMs are spread across FDs and UDs. During platform maintenance, only one UD is updated at a time.

### SLA and constraints

- **2+ VMs in an availability set → 99.95% SLA**
- VMs must be placed in the set **at creation time** — you cannot move an existing VM into a set.
- All VMs in the set should run the same application tier.

---

## 11. Availability zones

### What they protect against

The failure of an entire datacenter within a region. Each zone is physically separate with independent power, cooling, and networking.

### SLA

**2+ VMs in different zones → 99.99% SLA** (vs 99.95% for availability sets)

### Availability set vs availability zone — the most-tested comparison

| | Availability Set | Availability Zone |
|---|---|---|
| Protects against | Rack/hardware failure | Datacenter failure |
| SLA (2+ VMs) | **99.95%** | **99.99%** |
| Scope | Single datacenter | Multiple datacenters in region |
| Recommendation | Legacy apps | **All new workloads** |

**Exam trap:** "Highest availability" always means availability zones (99.99%), never sets (99.95%).

---

## 12. VM Scale Sets (VMSS)

VMSS lets you deploy and manage a group of load-balanced VMs. The number of instances can grow or shrink automatically based on demand or a schedule.

### Orchestration modes

| Mode | Description | Use case |
|---|---|---|
| **Uniform** | All VMs identical, managed as a set | Stateless web tiers, batch |
| **Flexible** | Mix of VM sizes, manage individual VMs in set | Stateful apps, HA across zones — **recommended** |

### Autoscale rule types

| Type | How |
|---|---|
| **Metric-based** | React to real-time metrics (CPU, memory, queue length) |
| **Schedule-based** | Predictable load (scale up at 9 AM, down at 6 PM) |
| **Predictive** | ML-based forecast — preview feature |

### Autoscale rule anatomy

```
Metric:       Average CPU % across all VMs
Time window:  10 minutes
Threshold:    > 75% (scale out)  /  < 25% (scale in)
Action:       Add 2 instances   /  Remove 1 instance
Cooldown:     5 minutes
Min:          2 instances
Max:          20 instances
```

**Cooldown period:** prevents thrashing — the scale rule cannot fire again until cooldown elapses.

### Upgrade policies

| Policy | Behavior |
|---|---|
| **Manual** | You trigger updates to instances |
| **Automatic** | Azure applies new model to all instances immediately |
| **Rolling** | Instances updated in batches — recommended for zero-downtime upgrades |

### Attaching a VM to VMSS (exam trap)

- VM must be in the **same region and same resource group** as the VMSS.
- VMSS must use **Flexible orchestration mode**.
- VM must have a **managed disk**.
- VM must NOT be in a self-defined availability set or proximity placement group.

### VMSS limits

- Up to **1,000 instances** with marketplace images.
- Up to **600 instances** with a custom managed image.

---

## 13. Azure Container Registry (ACR)

ACR is a managed Docker-compatible registry for storing and managing container images.

### SKU tiers

| SKU | Use |
|---|---|
| **Basic** | Dev/test, small teams |
| **Standard** | Production workloads |
| **Premium** | Geo-replication, private link, content trust, tokens |

### Core operations

```bash
# Create a registry
az acr create --resource-group rg-prod --name myRegistry --sku Standard

# Build and push directly from source (no local Docker required)
az acr build --registry myRegistry --image myapp:v1 .

# Log in
az acr login --name myRegistry

# Push a local image
docker tag myapp:latest myregistry.azurecr.io/myapp:v1
docker push myregistry.azurecr.io/myapp:v1

# List repositories
az acr repository list --name myRegistry
```

### Authentication to ACR

| Method | Use case |
|---|---|
| **Admin account** | Dev/test only — disable in production |
| **Service principal** | CI/CD pipelines |
| **Managed identity** | Azure services (ACI, App Service, AKS) — **best practice** |
| `az acr login` | Developer local access via token |

---

## 14. Azure Container Instances (ACI)

ACI runs a container in Azure without managing any infrastructure — no cluster, no VM, no orchestrator.

### Key characteristics

- **Serverless containers** — pay per second of CPU/memory used.
- No persistent orchestration — no auto-healing cluster.
- Best for: **burst workloads, batch jobs, event-driven tasks, dev/test**.
- Not for: long-running, highly available, auto-scaling production services.

### Creating a container instance

```bash
az container create \
  --resource-group rg-prod \
  --name myContainer \
  --image myregistry.azurecr.io/myapp:v1 \
  --registry-login-server myregistry.azurecr.io \
  --registry-username myRegistry \
  --registry-password <password> \
  --cpu 1 --memory 1.5 \
  --ports 80 \
  --dns-name-label myapp-demo \
  --restart-policy OnFailure
```

### Restart policies

| Policy | Behavior |
|---|---|
| `Always` | Always restart (default) — persistent services |
| `OnFailure` | Restart only on non-zero exit code |
| `Never` | Run once and stop — batch jobs |

### Multi-container groups

- ACI supports **container groups**: multiple containers sharing a lifecycle, network, and storage.
- All containers in a group share the same IP and port space — communicate via `localhost`.
- Multi-container groups are **Linux only** — not supported on Windows containers.

---

## 15. Azure Container Apps

Container Apps is a serverless PaaS container platform built on Kubernetes and KEDA — without exposing Kubernetes complexity.

### Key concepts

| Concept | Description |
|---|---|
| **Container App Environment** | Shared boundary; single VNet integration point |
| **Container App** | The application — one or more containers |
| **Revision** | Immutable snapshot of a container app version |
| **Ingress** | HTTP traffic routing — internal or external |
| **Scale rules** | KEDA-based: HTTP traffic, queue length, CPU, custom |

### Revisions and traffic splitting

When you update a container app, a new revision is created. You can split traffic:

```
Revision v1 → 80% of traffic
Revision v2 → 20% of traffic  (A/B testing or blue-green)
```

### Scaling

```yaml
scale:
  minReplicas: 0      # scales to zero when idle — no cost
  maxReplicas: 30
  rules:
    - name: http-rule
      http:
        metadata:
          concurrentRequests: "100"
```

### ACI vs Container Apps vs App Service

| | ACI | Container Apps | App Service (containers) |
|---|---|---|---|
| Scale to zero | No | **Yes** | Yes (Consumption plan) |
| Auto-scale | No | **Yes** | Yes |
| Multi-container group | Yes (Linux only) | Yes | No |
| Best for | Burst/batch | Microservices, event-driven | Web apps, APIs |

---

## 16. Container sizing and scaling

### ACI sizing

Specify CPU (0.1–4 vCPU) and memory (0.1–16 GiB) per container group. No autoscale.

### Container Apps scaling

- Min/max replicas with scale rules.
- Scale to 0 — containers shut down when idle, start on first request (cold start latency).
- KEDA scalers: HTTP, Azure Service Bus, Azure Storage Queue, custom.

---

## 17. App Service plans

An App Service plan defines the **region, OS, size, and pricing tier** that web apps run on. Multiple apps share one plan.

### Tiers

| Category | Tiers | Key features |
|---|---|---|
| **Shared compute** | Free (F1), Shared (D1) | Multi-tenant; CPU minute limits; no custom domains (Free) |
| **Dedicated compute** | Basic (B1–B3), Standard (S1–S3), Premium (P0v3–P3v3) | Dedicated VMs; scaling; custom domains |
| **Isolated** | Isolated (I1–I3), IsolatedV2 | Dedicated VNet injection; compliance |

### Features by tier — memorize these

| Feature | Free | Shared | Basic | Standard | Premium | Isolated |
|---|---|---|---|---|---|---|
| Custom domain | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| SSL/TLS | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| Auto-scale | ✗ | ✗ | ✗ | **✓** | ✓ | ✓ |
| **Deployment slots** | ✗ | ✗ | ✗ | **✓ (5)** | ✓ (20) | ✓ (20) |
| VNet integration | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Zone redundancy | ✗ | ✗ | ✗ | ✗ | ✓ (v2+) | ✓ |

**High-frequency exam fact:** Deployment slots require **Standard tier or higher**. Most common question: "Staging slot unavailable — what do you do?" → Scale up the plan to Standard.

### Scale up vs scale out

| | Scale up (vertical) | Scale out (horizontal) |
|---|---|---|
| What | Change to a bigger tier (B1 → S1) | Increase instance count (1 → 5) |
| When | Need more CPU/RAM/features | Handle more concurrent traffic |
| Auto | Manual only | Auto-scale (Standard+) |

---

## 18. App Service — web apps

### Creating a web app

```bash
# Create a plan first
az appservice plan create \
  --name myPlan \
  --resource-group rg-prod \
  --sku S1 \
  --is-linux

# Create the web app
az webapp create \
  --resource-group rg-prod \
  --plan myPlan \
  --name myWebApp \
  --runtime "NODE:18-lts"
```

### Custom domains

1. Add a CNAME or A record at your DNS registrar pointing to the app's default domain.
2. In the app: **Custom domains → Add custom domain** → verify ownership via TXT record.
3. Bind an SSL certificate to the custom domain.

### TLS/SSL configuration

| Type | Notes |
|---|---|
| Free managed certificate | Auto-renewed; for custom domains |
| App Service Certificate | Purchased through Azure, stored in Key Vault |
| Bring your own certificate | Upload a .pfx |
| SNI SSL | Share IP with other apps (default, cheaper) |
| IP SSL | Dedicated IP (needed for some legacy clients) |

### App settings and connection strings

```bash
az webapp config appsettings set \
  --resource-group rg-prod \
  --name myWebApp \
  --settings NODE_ENV=production DB_HOST=mydb.database.windows.net
```

**Slot settings:** settings that do NOT swap when deployment slots are swapped — they stay with the slot.

---

## 19. Deployment slots

### Rules

- Available on **Standard, Premium, Isolated** tiers only.
- Standard: **5 slots**; Premium/Isolated: **20 slots**.
- Each slot has its own URL: `myapp-staging.azurewebsites.net`.
- Each slot is a fully independent app instance.

### Swap operation — zero-downtime deployment

1. Deploy new code to the staging slot.
2. Azure warms up staging (routes some traffic there first).
3. Swap staging ↔ production.
4. Production now runs new code; staging holds the old code.
5. If problems arise — swap back immediately (instant rollback).

```bash
az webapp deployment slot swap \
  --resource-group rg-prod \
  --name myWebApp \
  --slot staging \
  --target-slot production
```

### Slot settings (sticky settings)

Settings marked as slot settings **stay with the slot**, not the code. Use this for:
- Production database connection string (should not go to staging after a swap)
- Environment-specific API keys

**Exam scenario:** "After a slot swap, the production app connects to the staging database." → The connection string was not marked as a slot setting.

### Auto-swap

Configure a slot to automatically swap when new code is deployed. Enable in **Configuration → General settings → Auto swap**.

---

## 20. App Service networking

### Outbound (app calling external resources)

| Feature | Purpose |
|---|---|
| **VNet Integration** | App makes outbound calls into a private VNet |
| **Hybrid Connections** | App calls specific on-premises endpoints (no VPN needed) |
| **NAT Gateway** | Stable outbound IP address |

### Inbound (traffic to the app)

| Feature | Purpose |
|---|---|
| **Access restrictions** | Allow/deny by IP, VNet, service tag |
| **Private endpoint** | Give the app a private IP in your VNet — fully private inbound |
| **App Service Environment (ASE)** | Fully isolated, injected into VNet — Isolated tier |

---

## 21. App Service backup and restore

### Requirements

- **Standard tier or higher** required.
- Backup stores app content + configured databases to a **storage account** in your subscription.

### Configure backup

```bash
az webapp config backup create \
  --resource-group rg-prod \
  --webapp-name myWebApp \
  --container-url "https://mystorageaccount.blob.core.windows.net/backups?<SAS>" \
  --db-connection-string "..." \
  --db-type SqlAzure \
  --frequency 24 \
  --retain-one true
```

### What is backed up

- App files.
- Database (if configured via backup settings).
- SSL certificates and custom domain bindings are **NOT** backed up.

### Restore

Portal: **Backups → Restore** → select backup → restore to same app or a different slot.

---

## 22. Hands-on lab walkthroughs

### Lab 1 — Deploy and resize a VM

```bash
# Create resource group
az group create -n rg-lab -l eastus

# Create a Linux VM in Zone 1
az vm create \
  --resource-group rg-lab \
  --name myLinuxVM \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --zone 1

# Check current size
az vm show -g rg-lab -n myLinuxVM --query "hardwareProfile.vmSize" -o tsv

# Resize (causes a reboot)
az vm resize -g rg-lab -n myLinuxVM --size Standard_D2s_v5

# Attach a 128 GiB Premium SSD data disk
az vm disk attach -g rg-lab --vm-name myLinuxVM \
  --name myDataDisk --size-gb 128 --sku Premium_LRS --new

# Deallocate to stop billing
az vm deallocate -g rg-lab -n myLinuxVM
```

### Lab 2 — VMSS with CPU-based autoscale

```bash
# Create VMSS across 3 zones with a load balancer
az vmss create \
  --resource-group rg-lab \
  --name myVMSS \
  --image Ubuntu2204 \
  --instance-count 2 \
  --vm-sku Standard_D2s_v5 \
  --zones 1 2 3 \
  --orchestration-mode Flexible \
  --admin-username azureuser \
  --generate-ssh-keys

# Create autoscale profile
az monitor autoscale create \
  --resource-group rg-lab \
  --resource myVMSS \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name myAutoscale \
  --min-count 2 --max-count 10 --count 2

# Scale-out rule: CPU > 75% → add 2 VMs
az monitor autoscale rule create \
  --resource-group rg-lab \
  --autoscale-name myAutoscale \
  --condition "Percentage CPU > 75 avg 5m" \
  --scale out 2

# Scale-in rule: CPU < 25% → remove 1 VM
az monitor autoscale rule create \
  --resource-group rg-lab \
  --autoscale-name myAutoscale \
  --condition "Percentage CPU < 25 avg 5m" \
  --scale in 1
```

### Lab 3 — Deploy Bicep template

```bicep
// main.bicep — deploys a storage account and outputs the blob endpoint
param location string = resourceGroup().location
param storageName string = 'sa${uniqueString(resourceGroup().id)}'

@allowed(['Standard_LRS', 'Standard_GRS', 'Premium_LRS'])
param sku string = 'Standard_LRS'

resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageName
  location: location
  sku: { name: sku }
  kind: 'StorageV2'
}

output blobEndpoint string = storage.properties.primaryEndpoints.blob
```

```bash
# Deploy Bicep
az deployment group create \
  -g rg-lab \
  --template-file main.bicep \
  --parameters sku=Standard_GRS

# What-if before deploying
az deployment group what-if -g rg-lab --template-file main.bicep

# Convert ARM JSON to Bicep
az bicep decompile --file exported-template.json
```

### Lab 4 — App Service with staging slot + swap

```bash
# Create Standard plan + web app
az appservice plan create -n myPlan -g rg-lab --sku S1
az webapp create -n myWebApp -g rg-lab --plan myPlan --runtime "NODE:18-lts"

# Create staging slot
az webapp deployment slot create -n myWebApp -g rg-lab --slot staging

# Deploy code to staging (zip deploy example)
az webapp deploy -n myWebApp -g rg-lab \
  --slot staging --src-path ./myapp.zip --type zip

# Mark DB connection string as slot setting (stays with production slot)
az webapp config connection-string set \
  -n myWebApp -g rg-lab \
  --slot production \
  --settings "SQLDB=Server=myserver.database.windows.net" \
  --connection-string-type SQLAzure \
  --slot-settings SQLDB

# Swap staging → production
az webapp deployment slot swap \
  -n myWebApp -g rg-lab --slot staging --target-slot production
```

### Lab 5 — Build and run container with ACR + ACI

```bash
# Create ACR
az acr create -n myRegistry -g rg-lab --sku Basic --admin-enabled true

# Build nginx image directly from a Dockerfile in current directory
echo 'FROM nginx:latest\nCOPY index.html /usr/share/nginx/html/' > Dockerfile
echo '<h1>Hello from ACI</h1>' > index.html
az acr build --registry myRegistry --image nginx-demo:v1 .

# Get admin password
PASSWORD=$(az acr credential show -n myRegistry --query passwords[0].value -o tsv)

# Deploy to ACI
az container create \
  --resource-group rg-lab \
  --name myContainer \
  --image myregistry.azurecr.io/nginx-demo:v1 \
  --registry-login-server myregistry.azurecr.io \
  --registry-username myRegistry \
  --registry-password $PASSWORD \
  --cpu 1 --memory 1 \
  --ports 80 \
  --dns-name-label my-aci-$(date +%s) \
  --restart-policy OnFailure

# View logs
az container logs -g rg-lab -n myContainer
```

---

## 23. Exam question patterns and traps

### Pattern 1 — Stop vs deallocate billing

> "A VM is shut down from inside Windows. Is compute still being billed?"

Answer: **Yes**. Shutting down from inside the OS = Stopped (not deallocated). Compute billing continues. You must deallocate via portal/CLI to stop billing.

### Pattern 2 — Availability set vs zone

> "You need 99.99% availability for a production application running on 2 VMs."

Answer: **Availability zones** — place each VM in a different zone. Availability sets only give 99.95%.

### Pattern 3 — Deployment slot unavailable

> "You try to create a staging slot for your App Service but the option is greyed out."

Answer: The plan is on a tier lower than Standard (e.g., Basic or Free). **Scale up to Standard** tier first.

### Pattern 4 — Slot swap settings issue

> "After swapping staging to production, the production app is connecting to the staging database."

Answer: The database connection string was **not marked as a slot setting**. It swapped with the code. Mark it as a deployment slot setting so it stays with the production slot.

### Pattern 5 — dependsOn in ARM template

> "A VM deployment fails because the NIC does not exist yet. How do you fix the ARM template?"

Answer: Add `dependsOn` to the VM resource referencing the NIC resource ID. Or use an implicit reference to the NIC's ID inside the VM's network profile — ARM infers the dependency.

### Pattern 6 — Complete deployment mode danger

> "You deploy an ARM template in Complete mode. A storage account in the RG that was not in the template was deleted."

Answer: This is expected behavior. Complete mode **deletes resources not defined in the template**. Use **Incremental** mode to leave existing resources untouched.

### Pattern 7 — Parameterize a Bicep file

> "The Bicep file deploys an App Service plan with `sku: 'F1'`. The production environment needs `P3v3`. How do you make the SKU configurable?"

Answer: Add `param skuName string = 'F1'` and reference it with `sku: { name: skuName }`. Pass the value during deployment with `--parameters skuName=P3v3`.

### Pattern 8 — Premium SSD and VM size

> "A Standard_D4_v5 VM cannot use Premium SSD disks."

Answer: The size is `D4_v5` without the `s` suffix. Only `D4s_v5` supports Premium SSD. Either switch to the `s`-suffixed size or use a lower SSD tier.

### Pattern 9 — VMSS attach constraints

> "You try to add a VM in East US / RG2 to a VMSS in West US / RG1."

Answer: To attach a VM to a Flexible VMSS, the VM must be in the **same region and same resource group** as the VMSS. The operation will fail.

### Pattern 10 — ACI multi-container Linux only

> "You need to run two Windows containers together in an ACI container group."

Answer: Multi-container groups in ACI are **Linux only**. Windows containers do not support multi-container groups.

### Pattern 11 — Container Apps vs ACI

> "You need an application that scales to zero when idle and scales out to 20 replicas under load."

Answer: **Azure Container Apps** — it supports scale-to-zero and KEDA-based autoscaling. ACI cannot autoscale or scale to zero.

### Pattern 12 — Temp disk is ephemeral

> "A developer stored logs on the D: drive (temp disk) of a Windows VM. The logs disappeared after resizing."

Answer: The temporary disk is **ephemeral** — data is lost on reboot, resize, or deallocation. Always use a **data disk** for persistent storage.

### Pattern 13 — What-if before deployment

> "You want to verify what resources will be created without actually deploying."

Answer: `az deployment group what-if` (or `New-AzResourceGroupDeployment -WhatIf` in PowerShell). Shows planned changes without applying them.

### Pattern 14 — ARM to Bicep conversion

> "You have an ARM JSON template exported from the portal. How do you convert it to Bicep?"

Answer: `az bicep decompile --file azuredeploy.json` — produces a `.bicep` file.

### Pattern 15 — ARM hotspot / fill-in

Common blanks in ARM JSON hotspot questions:
- Where a parameter is referenced: `[parameters('vmName')]`
- Where a variable is referenced: `[variables('nicName')]`
- Where a resource ID is referenced: `[resourceId('Microsoft.Network/virtualNetworks', 'myVNet')]`
- The `dependsOn` array entry: `"[resourceId('Microsoft.Network/networkInterfaces', 'myNIC')]"`

---

## 24. Cheat sheet

> **One-page distilled reference. Print this.**

### Compute service quick pick

| Workload | Use |
|---|---|
| Full OS, IaaS control | **Azure VM** |
| Many identical VMs, auto-scale | **VMSS** |
| Web app / API, no OS management | **App Service** |
| Single simple container, burst/batch | **ACI** |
| Auto-scaling containers, microservices | **Container Apps** |
| Container image registry | **ACR** |

### VM stop vs deallocate

| Action | Billed? | IP kept? |
|---|---|---|
| Stop (from inside OS) | **Yes** | Yes |
| Deallocate (portal/CLI) | **No** | Dynamic IP released |

### Availability: set vs zone

| | Availability Set | Availability Zone |
|---|---|---|
| Protects from | Rack/hardware failure | Datacenter failure |
| SLA (2+ VMs) | **99.95%** | **99.99%** |
| Recommendation | Legacy apps | **All new workloads** |

### VM size — Premium SSD rule

VM size must have the **`s` suffix** (e.g., `D4s_v5`) to support Premium SSD. Without it → use Standard SSD or lower.

### Temp disk is always ephemeral

Data on the temp disk (D: on Windows, /mnt on Linux) is **lost on reboot, resize, or deallocation.** Never store data there.

### ARM template quick reference

```
Incremental mode  = safe; leaves unlisted resources alone     (default)
Complete mode     = DELETES resources not in the template     (dangerous)
dependsOn         = explicit deploy ordering
what-if           = dry run, shows planned changes
securestring      = masks value in logs
```

### ARM ↔ Bicep

| Direction | Command |
|---|---|
| ARM JSON → Bicep | `az bicep decompile --file template.json` |
| Bicep → ARM JSON | `az bicep build --file main.bicep` |
| Deploy Bicep | `az deployment group create --template-file main.bicep` |

### Bicep param decorators

```bicep
@secure()               // masks value in logs (passwords, keys)
@allowed(['a','b'])     // restricts valid values
@minLength(n)           // string minimum length
@maxLength(n)           // string maximum length
@description('...')     // documentation
```

### VMSS rules

- **Flexible** orchestration = recommended; supports mixed VMs, zone HA
- **Attach VM to VMSS:** same RG + same region + managed disk + Flexible mode, not in availability set
- Instance limit: 1,000 (marketplace) / 600 (custom image)
- Cooldown = wait time before next autoscale action

### App Service tier quick pick

| Need | Minimum tier |
|---|---|
| Custom domain | Shared |
| SSL/TLS | Basic |
| Auto-scale | **Standard** |
| Deployment slots | **Standard** (5 slots) |
| VNet Integration | Standard |
| Zone redundancy | PremiumV2+ |

### Deployment slots rules

- Standard: 5 slots; Premium/Isolated: 20 slots
- Swap = zero-downtime deploy; old code goes to staging for instant rollback
- **Slot settings (sticky):** stay with the slot, do NOT swap with code
- Auto-swap: automatically swaps on deploy

### Containers quick compare

| | ACI | Container Apps | App Service |
|---|---|---|---|
| Scale to zero | No | **Yes** | Yes (Consumption) |
| Auto-scale | No | **Yes (KEDA)** | Yes |
| Multi-container | Yes (**Linux only**) | Yes | No |
| Best for | Burst/batch | Microservices | Web apps |

### ACR quick auth guide

| Method | When |
|---|---|
| Admin account | Dev only — disable in prod |
| Service principal | CI/CD pipelines |
| Managed identity | Azure services pulling images — **always prefer this** |

### Common CLI snippets — compute

```bash
# VM
az vm create -g rg -n myVM --image Ubuntu2204 --size Standard_D2s_v5 --zone 1
az vm deallocate -g rg -n myVM
az vm resize -g rg -n myVM --size Standard_D4s_v5
az vm disk attach -g rg --vm-name myVM --name disk1 --size-gb 64 --sku Premium_LRS --new

# ARM / Bicep
az deployment group create -g rg --template-file main.bicep --parameters env=prod
az deployment group what-if -g rg --template-file main.bicep
az bicep decompile --file template.json

# VMSS
az vmss create -g rg -n myVMSS --image Ubuntu2204 --instance-count 2 --zones 1 2 3
az vmss scale -g rg -n myVMSS --new-capacity 5

# App Service
az appservice plan create -n myPlan -g rg --sku S1
az webapp create -n myApp -g rg --plan myPlan --runtime "NODE:18-lts"
az webapp deployment slot create -n myApp -g rg --slot staging
az webapp deployment slot swap -n myApp -g rg --slot staging --target-slot production

# ACR + ACI
az acr create -n myRegistry -g rg --sku Standard
az acr build --registry myRegistry --image myapp:v1 .
az container create -g rg -n myACI --image myregistry.azurecr.io/myapp:v1 --cpu 1 --memory 1 --ports 80
az container logs -g rg -n myACI
```

### Exam tactical reminders

1. **Stop ≠ deallocate** — stopped VM is still billed. Deallocate to save money.
2. **"Highest availability"** → always availability zones (99.99%), never sets (99.95%).
3. **Staging slot unavailable** → scale up the App Service plan to Standard.
4. **ARM Complete mode deletes** unlisted resources — use Incremental unless you intend this.
5. **Premium SSD** needs VM with `s` in the size name (e.g., `D4s_v5` not `D4_v5`).
6. **Temp disk is ephemeral** — data is gone after reboot/resize.
7. **ACI multi-container** → Linux only.
8. **Container Apps** scale to zero; ACI cannot autoscale.
9. **Bicep decompile** converts ARM JSON to Bicep (`az bicep decompile`).
10. **Attach VM to VMSS** → Flexible mode, same RG, same region, managed disk required.

---

*Every service in this domain is best learned by deploying it. Lab each section in your free Azure account before moving to the next.*
