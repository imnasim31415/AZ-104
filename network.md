# AZ-104 Azure Networking Hands-On Lab Guide

**Purpose:** Practice Azure networking concepts using Azure CLI in Azure
Cloud Shell, while keeping costs under control.

**Exam alignment:** This guide is aligned with the AZ-104 networking
objectives in Microsoft's current study guide: create/configure VNets
and subnets, VNet peering, public IPs, user-defined routes, troubleshoot
connectivity, configure NSGs/ASGs and effective rules, Azure Bastion,
service endpoints, and private endpoints. It also includes supporting
DNS and hybrid-connectivity concepts useful for understanding networking
scenarios.

Official exam guide:
https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104

------------------------------------------------------------------------

## 0. Cost and safety rules

1.  Use one dedicated resource group per lab.
2.  Prefer configuration-only resources first: VNets, subnets, NSGs,
    ASGs, route tables, and peerings.
3.  Do **not** leave VMs, VPN gateways, NAT Gateway, Bastion, private
    endpoints, or other billable services running after practice.
4.  Deleting a resource group is the easiest cleanup method, but it does
    not delete resources created outside that group.
5.  Cost data can take time to appear. Check Cost Analysis after the
    lab.
6.  A budget sends alerts; it does not automatically stop all spending.
7.  Do not use a work subscription or shared resource group for these
    labs.

### Sign in and select the subscription

Run in Azure Cloud Shell (Bash):

``` bash
az login
az account list --output table
az account show --output table
```

If you have multiple subscriptions, select the intended one:

``` bash
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
az account show --query "{name:name,id:id,tenantId:tenantId}" --output table
```

Check the CLI version if a command returns an unexpected API-version
error:

``` bash
az version
az upgrade
```

Cloud Shell often already has Azure CLI installed. You do not need a VM
just to run these commands.

------------------------------------------------------------------------

## 1. Create a reusable lab environment

Use a region available to your subscription. The examples use
`southeastasia`; change it if that region is unavailable or unsuitable.

``` bash
export RG="rg-az104-network-lab"
export LOC="southeastasia"

export VNET_A="vnet-az104-a"
export VNET_B="vnet-az104-b"

az group create --name "$RG" --location "$LOC" --tags purpose=az104 lab=networking
```

Confirm the resource group exists:

``` bash
az group show --name "$RG" --output table
```

**Important:** Re-run the variable definitions in every new Cloud Shell
session. Shell variables are not necessarily retained between sessions.

## 2. VNet, address spaces, subnets, and IP planning

### Concepts to understand

-   A virtual network (VNet) is a private network boundary in Azure.
-   A subnet divides a VNet address space into smaller networks.
-   VNet address spaces must not overlap when you intend to peer or
    route between them.
-   Azure reserves five IPv4 addresses in each subnet: the first four
    and the last address.
-   A private IP is assigned from a subnet. A public IP is a separate
    Azure resource attached to a supported network interface or service.
-   A subnet can be associated with an NSG and/or route table.

### Create two VNets

``` bash
az network vnet create \
  --resource-group "$RG" \
  --name "$VNET_A" \
  --location "$LOC" \
  --address-prefixes 10.10.0.0/16 \
  --subnet-name snet-web \
  --subnet-prefixes 10.10.1.0/24

az network vnet subnet create \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-app \
  --address-prefixes 10.10.2.0/24

az network vnet subnet create \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-db \
  --address-prefixes 10.10.3.0/24

az network vnet create \
  --resource-group "$RG" \
  --name "$VNET_B" \
  --location "$LOC" \
  --address-prefixes 10.20.0.0/16 \
  --subnet-name snet-spoke \
  --subnet-prefixes 10.20.1.0/24
```

### Inspect VNets and subnets

``` bash
az network vnet list --resource-group "$RG" --output table

az network vnet show \
  --resource-group "$RG" \
  --name "$VNET_A" \
  --query "{name:name,addressSpace:addressSpace.addressPrefixes,subnets:subnets[].{name:name,prefix:addressPrefix}}" \
  --output json

az network vnet subnet list \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --output table

az network vnet list-available-ips \
  --resource-group "$RG" \
  --name "$VNET_A" \
  --output table
```

### Practice tasks

-   Add another subnet with a non-overlapping CIDR.
-   Try to create an overlapping subnet and observe the validation
    error. Do not change the VNet address plan unless you mean to.
-   Explain why `10.10.0.0/16` and `10.20.0.0/16` can be peered but two
    VNets with overlapping ranges cannot be directly peered for normal
    routing.
-   Identify which IP addresses Azure reserves in `10.10.1.0/24`.

------------------------------------------------------------------------

## 3. Public and private IP addresses

### Create a public IP resource

A public IP resource by itself is not the same as giving a VM internet
connectivity; it must be associated with a supported resource such as a
NIC or load balancer.

``` bash
az network public-ip create \
  --resource-group "$RG" \
  --name pip-az104-demo \
  --location "$LOC" \
  --sku Standard \
  --allocation-method Static
```

Inspect it:

``` bash
az network public-ip show \
  --resource-group "$RG" \
  --name pip-az104-demo \
  --query "{name:name,ip:ipAddress,sku:sku.name,allocation:publicIPAllocationMethod}" \
  --output table
```

Delete it when finished:

``` bash
az network public-ip delete --resource-group "$RG" --name pip-az104-demo
```

**Cost note:** Public IP pricing can apply even when it is not attached,
depending on SKU and current pricing. Delete unused public IP resources.

### Check private IP availability

``` bash
az network vnet check-ip-address \
  --resource-group "$RG" \
  --name "$VNET_A" \
  --ip-address 10.10.1.10
```

This checks whether a private IP is available in the VNet address space.
In real deployments, also ensure the address belongs to the intended
subnet and is not reserved or already allocated.

------------------------------------------------------------------------

## 4. Network Security Groups (NSGs) and Application Security Groups (ASGs)

### Concepts

-   An NSG contains inbound and outbound allow/deny rules.
-   Rules are evaluated by priority, with lower numbers evaluated first.
-   Azure has default NSG rules; custom rules do not erase them.
-   NSGs can be associated with subnets or network interfaces.
-   An ASG lets you group NICs by application role so that NSG rules can
    refer to those groups rather than fixed IP addresses.
-   If NSGs are associated at both subnet and NIC levels, traffic must
    pass both layers.
-   An NSG is stateful: return traffic for an allowed connection is
    generally permitted automatically.

### Create NSGs

``` bash
for nsg in nsg-web nsg-app nsg-db; do
  az network nsg create \
    --resource-group "$RG" \
    --name "$nsg" \
    --location "$LOC"
done
```

### Create ASGs

``` bash
for asg in asg-web asg-app asg-db; do
  az network asg create \
    --resource-group "$RG" \
    --name "$asg" \
    --location "$LOC"
done
```

### Create an inbound rule allowing HTTP to the web tier

``` bash
az network nsg rule create \
  --resource-group "$RG" \
  --nsg-name nsg-web \
  --name Allow-HTTP-From-Internet \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80
```

### Create a rule that allows app traffic only from the web ASG

This is a conceptual ASG rule. It becomes useful when a NIC is actually
added to `asg-web` and an app NIC to `asg-app`.

``` bash
az network nsg rule create \
  --resource-group "$RG" \
  --nsg-name nsg-app \
  --name Allow-Web-To-App-8080 \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-asgs asg-web \
  --source-port-ranges '*' \
  --destination-asgs asg-app \
  --destination-port-ranges 8080
```

### Associate NSGs with subnets

``` bash
az network vnet subnet update \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-web \
  --network-security-group nsg-web

az network vnet subnet update \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-app \
  --network-security-group nsg-app

az network vnet subnet update \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-db \
  --network-security-group nsg-db
```

### Inspect rules and effective rules

``` bash
az network nsg list --resource-group "$RG" --output table

az network nsg rule list \
  --resource-group "$RG" \
  --nsg-name nsg-web \
  --include-default \
  --output table

az network nsg show \
  --resource-group "$RG" \
  --name nsg-web \
  --query "securityRules[].{name:name,priority:priority,direction:direction,access:access,protocol:protocol,source:sourceAddressPrefix,destinationPort:destinationPortRange}" \
  --output table
```

The `--include-default` option may not be supported in every Azure CLI
version. If it errors, use `az network nsg show` and inspect
`defaultSecurityRules`.

**Effective security rules:** The Azure portal can show effective
security rules for a VM NIC. CLI inspection of the NSG alone does not
necessarily calculate the combined effective rules across every attached
NSG. For VM-based effective-rule and flow troubleshooting, see the
Network Watcher section.

### Practice tasks

-   Add a deny rule for inbound TCP 22 with a priority higher than the
    default allow rules you want to override.
-   Explain why a rule with priority 100 is processed before a rule with
    priority 200.
-   Explain the difference between source IP prefixes and source ASGs.
-   Compare subnet-level NSG association with NIC-level association.

------------------------------------------------------------------------

## 5. User-defined routes (UDRs) and route selection

### Concepts

Azure uses system routes by default. A route table lets you add
user-defined routes. Route selection uses longest-prefix match; when
prefixes match, route-source preference is also relevant. Common
next-hop types include `VirtualNetwork`, `Internet`, `VirtualAppliance`,
`VirtualNetworkGateway`, and `None` (drop traffic).

### Create a route table and a route

``` bash
az network route-table create \
  --resource-group "$RG" \
  --name rt-web \
  --location "$LOC"

az network route-table route create \
  --resource-group "$RG" \
  --route-table-name rt-web \
  --name Drop-10-30 \
  --address-prefix 10.30.0.0/16 \
  --next-hop-type None
```

Associate it with the web subnet:

``` bash
az network vnet subnet update \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-web \
  --route-table rt-web
```

Inspect:

``` bash
az network route-table show \
  --resource-group "$RG" \
  --name rt-web \
  --output json

az network route-table route list \
  --resource-group "$RG" \
  --route-table-name rt-web \
  --output table
```

Remove the route and association if you want to reset only this lab
component:

``` bash
az network route-table route delete \
  --resource-group "$RG" \
  --route-table-name rt-web \
  --name Drop-10-30

az network vnet subnet update \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-web \
  --remove routeTable
```

**Practice tasks:** Create a route to `10.40.0.0/16` with next hop
`VirtualAppliance` only after you understand that this route type
normally needs a reachable appliance private IP. Do not create a fake
appliance route and assume it will work.

------------------------------------------------------------------------

## 6. VNet peering

### Concepts

-   Peering connects VNets over the Azure backbone.
-   Peering is not transitive by default.
-   Each direction has its own peering resource; create both directions
    for bidirectional connectivity.
-   Address spaces must not overlap.
-   Peering does not automatically make an NSG allow the traffic.
-   Gateway transit and use of a remote gateway are options for certain
    hub-and-spoke designs.

### Get the VNet resource IDs

Do this instead of guessing or hardcoding the IDs:

``` bash
export ID_A=$(az network vnet show \
  --resource-group "$RG" --name "$VNET_A" --query id --output tsv)

export ID_B=$(az network vnet show \
  --resource-group "$RG" --name "$VNET_B" --query id --output tsv)

echo "$ID_A"
echo "$ID_B"
```

If either variable is empty, stop and verify the VNet exists and that
`RG`, `VNET_A`, and `VNET_B` are set correctly.

### Create both peering directions

``` bash
az network vnet peering create \
  --resource-group "$RG" \
  --name a-to-b \
  --vnet-name "$VNET_A" \
  --remote-vnet "$ID_B" \
  --allow-vnet-access

az network vnet peering create \
  --resource-group "$RG" \
  --name b-to-a \
  --vnet-name "$VNET_B" \
  --remote-vnet "$ID_A" \
  --allow-vnet-access
```

Inspect both sides:

``` bash
az network vnet peering list \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --output table

az network vnet peering list \
  --resource-group "$RG" \
  --vnet-name "$VNET_B" \
  --output table
```

Look for `Connected` as the peering state. If creation fails with a
resource-not-found error, verify the VNet names, resource group,
subscription, and resource IDs:

``` bash
echo "RG=[$RG] VNET_A=[$VNET_A] VNET_B=[$VNET_B]"
az network vnet list --resource-group "$RG" --output table
```

Delete peerings individually if needed:

``` bash
az network vnet peering delete \
  --resource-group "$RG" --vnet-name "$VNET_A" --name a-to-b

az network vnet peering delete \
  --resource-group "$RG" --vnet-name "$VNET_B" --name b-to-a
```

### Practice tasks

-   Explain why one-way peering is not the same as creating peerings in
    both directions.
-   Try enabling forwarded traffic and inspect the setting.
-   Explain why a VM in VNet A may still fail to connect to a VM in VNet
    B even when peering is `Connected`: NSGs, guest firewalls, missing
    VMs/NICs, and application listeners can still block the connection.
-   Draw a hub-and-spoke diagram and explain why peering is not
    transitive.

------------------------------------------------------------------------

## 7. Azure DNS and Private DNS zones

### Concepts

-   Azure DNS hosts public DNS zones.
-   Azure Private DNS zones resolve names within linked VNets and
    connected network designs.
-   A VNet link associates a private DNS zone with a VNet.
-   Auto-registration can register VM records in supported scenarios.
-   A private DNS zone does not itself create a network path to the IP
    address.

### Create a private DNS zone and link it to a VNet

``` bash
az network private-dns zone create \
  --resource-group "$RG" \
  --name internal.example.com

az network private-dns link vnet create \
  --resource-group "$RG" \
  --zone-name internal.example.com \
  --name link-vnet-a \
  --virtual-network "$VNET_A" \
  --registration-enabled false
```

Create a test A record:

``` bash
az network private-dns record-set a create \
  --resource-group "$RG" \
  --zone-name internal.example.com \
  --name db

az network private-dns record-set a add-record \
  --resource-group "$RG" \
  --zone-name internal.example.com \
  --record-set-name db \
  --ipv4-address 10.10.3.4
```

Inspect:

``` bash
az network private-dns zone list --resource-group "$RG" --output table

az network private-dns link vnet list \
  --resource-group "$RG" \
  --zone-name internal.example.com \
  --output table

az network private-dns record-set a list \
  --resource-group "$RG" \
  --zone-name internal.example.com \
  --output table
```

**Note:** The A record is only a DNS record. Unless a host actually has
that IP and traffic is allowed, it will not make a service reachable.

For public DNS zones, the `az network dns zone` command group manages
zones and records. Public DNS zones may incur charges; check current
pricing before using them for a lab.

------------------------------------------------------------------------

## 8. Service endpoints for Azure PaaS

### Concepts

-   A service endpoint lets a subnet connect to supported Azure services
    using the Azure backbone and enables the service to identify the
    VNet/subnet for network rules.
-   The service resource's firewall/network settings must also allow the
    subnet.
-   Service endpoints do not give the PaaS resource a private IP in your
    VNet.
-   A private endpoint is different: it creates a private IP in your
    VNet for the supported service.

### Enable a Storage service endpoint on a subnet

This exercise creates a Storage account, which can incur
storage/transaction charges if used. Use it briefly and delete it
afterward.

Storage account names must be globally unique, lowercase alphanumeric,
and 3-24 characters. Generate a name:

``` bash
export STORAGE_NAME="az104net$RANDOM$RANDOM"
echo "$STORAGE_NAME"
```

Enable the endpoint:

``` bash
az network vnet subnet update \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name snet-app \
  --service-endpoints Microsoft.Storage
```

Create a Standard locally redundant Storage account:

``` bash
az storage account create \
  --resource-group "$RG" \
  --name "$STORAGE_NAME" \
  --location "$LOC" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access false
```

Inspect network rules:

``` bash
az storage account show \
  --resource-group "$RG" \
  --name "$STORAGE_NAME" \
  --query "{name:name,publicNetworkAccess:publicNetworkAccess,networkRuleSet:networkRuleSet}" \
  --output json
```

A service endpoint is not the same as a firewall allow rule. To restrict
Storage to selected networks, configure its network rules deliberately
and ensure you retain a way to administer it. Do not lock yourself out
of a real account.

After practice, delete the Storage account and resource group. The whole
lab deletion later removes the endpoint configuration as well.

------------------------------------------------------------------------

## 9. Private endpoints for Azure PaaS

### Concepts

-   A private endpoint uses a private IP address in a VNet to reach a
    supported Azure service privately.
-   It is backed by a Private Link connection.
-   DNS must resolve the service hostname to the private endpoint IP for
    clients to use the private path as intended.
-   Creating a private endpoint can incur hourly and data-processing
    charges.
-   Private DNS zone names depend on the service. For Blob Storage, the
    private DNS zone is commonly `privatelink.blob.core.windows.net`.

### Optional lab warning

Skip this lab if you want a configuration-only, low-cost session.
Private endpoints can be billed while deployed. If you do it, use a
dedicated resource group and delete it immediately afterward.

Create a private endpoint only after you understand the service's
required subnet, service resource ID, target subresource, and DNS zone.
First inspect the available CLI syntax:

``` bash
az network private-endpoint create --help
az network private-endpoint dns-zone-group create --help
az network private-dns zone create --help
```

A typical workflow is:

1.  Create a PaaS resource such as Storage.
2.  Create a private DNS zone appropriate to that service.
3.  Create a private endpoint in a subnet.
4.  Approve the private endpoint connection if required.
5.  Link the private DNS zone to the client VNet.
6.  Configure a DNS zone group on the private endpoint.
7.  Verify DNS resolution from a client inside the VNet.
8.  Delete the private endpoint, DNS zone group, DNS zone link/zone, and
    test PaaS resource, or delete the dedicated resource group.

Do not expect a DNS lookup from a random machine outside the VNet to
prove that private DNS is configured correctly.

------------------------------------------------------------------------

## 10. VPN Gateway, Site-to-Site, Point-to-Site, and gateway transit

### Exam concepts to know

-   **Site-to-Site (S2S):** connects an Azure VNet to an on-premises VPN
    device over an IPsec/IKE VPN.
-   **Point-to-Site (P2S):** connects an individual client to an Azure
    VNet.
-   **VNet-to-VNet VPN:** connects VNets using VPN gateways rather than
    VNet peering.
-   **VPN Gateway:** requires a `GatewaySubnet`, a supported public IP
    configuration, and a gateway SKU.
-   **Gateway transit:** allows a peered spoke VNet to use the VPN
    gateway in a hub VNet when the peerings are configured correctly.
-   **ExpressRoute:** private connectivity through a provider; it is not
    the same as a site-to-site VPN.
-   VPN gateways can be expensive and take time to provision. Prefer
    diagrams, documentation, or a short carefully budgeted lab unless
    you specifically need hands-on gateway deployment.

### Create only the required GatewaySubnet (configuration-only preparation)

``` bash
az network vnet subnet create \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name GatewaySubnet \
  --address-prefixes 10.10.255.0/27
```

This creates the subnet only; it does not create a VPN gateway. A real
gateway lab requires a public IP and gateway resource, which can incur
substantial ongoing costs. If you deploy a gateway, record its name and
delete it as soon as the exercise is done:

``` bash
az network vnet-gateway list --resource-group "$RG" --output table
```

Then, using the exact gateway name:

``` bash
az network vnet-gateway delete \
  --resource-group "$RG" \
  --name "<GATEWAY_NAME>"
```

Delete any gateway public IP, local network gateway, and connection
resources that you created separately, or delete the dedicated resource
group.

------------------------------------------------------------------------

## 11. Azure Bastion

### Concepts

-   Bastion provides browser-based RDP/SSH access to VMs without
    exposing their RDP/SSH ports directly to the internet.
-   Bastion requires a dedicated subnet named `AzureBastionSubnet` with
    a supported address prefix and a Bastion resource.
-   Bastion can incur ongoing hourly charges. It is not suitable to
    leave running for a cheap lab.
-   For exam preparation, learn the design, subnet requirement,
    deployment steps, and how Bastion differs from a VM with a public
    IP.

### Configuration preparation only

A subnet named `AzureBastionSubnet` can be created in a suitable VNet,
but the Bastion service itself is billable:

``` bash
az network vnet subnet create \
  --resource-group "$RG" \
  --vnet-name "$VNET_A" \
  --name AzureBastionSubnet \
  --address-prefixes 10.10.254.0/26
```

Do not deploy Bastion unless you have a specific exercise and have
checked current pricing. If deployed, delete the Bastion resource and
its associated public IP when finished.

------------------------------------------------------------------------

## 12. Load balancing and application delivery concepts

AZ-104 networking scenarios may expect you to understand how network
services fit together, even when the objective is not to build every
type of load balancer from scratch.

  -----------------------------------------------------------------------
  Service                 Layer / purpose         Key distinction
  ----------------------- ----------------------- -----------------------
  Azure Load Balancer     Layer 4 TCP/UDP         Distributes network
                                                  flows to backend
                                                  instances

  Application Gateway     Layer 7 HTTP/HTTPS      HTTP routing, TLS
                                                  termination, web
                                                  application firewall
                                                  options

  Azure Front Door        Global HTTP/HTTPS       Global entry point,
                          delivery                acceleration, routing,
                                                  and WAF capabilities

  Traffic Manager         DNS-based traffic       Returns DNS responses;
                          routing                 it does not proxy
                                                  application traffic

  NAT Gateway             Outbound internet       Provides scalable
                          connectivity            outbound SNAT for
                                                  subnet resources

  Azure Firewall          Managed network         Centralized filtering
                          security                and logging for network
                                                  traffic
  -----------------------------------------------------------------------

These services have different billing models and some incur ongoing
charges. For a low-cost AZ-104 plan, study the differences first and
deploy only when needed. Do not assume that deleting a VM automatically
deletes its public IP, NIC, disks, load balancer, or other separately
created resources.

------------------------------------------------------------------------

## 13. Network Watcher and troubleshooting

### Concepts

Network troubleshooting can involve:

-   Effective security rules
-   Effective routes
-   IP flow verify
-   Connection troubleshoot
-   Next hop
-   Packet capture
-   NSG diagnostics
-   DNS resolution
-   Guest operating system firewall and application listeners

Some tools require a VM/NIC or Network Watcher to be enabled in the
region. Do not create VMs just to run a single test unless you need the
hands-on experience.

### Check whether Network Watcher exists

``` bash
az network watcher list --output table
```

If appropriate for your lab and region, enable it:

``` bash
az network watcher configure \
  --resource-group "$RG" \
  --locations "$LOC" \
  --enabled true
```

Check command support with `az network watcher configure --help` if the
CLI version differs.

### Troubleshooting checklist

When a connection fails, check in this order:

1.  Is the destination resource running and listening on the expected
    port?
2.  Do the source and destination have correct IP addresses and subnet
    masks?
3.  Is there a route to the destination, and is the longest-prefix route
    the expected one?
4.  Does VNet peering show `Connected` in both directions?
5.  Does the NSG permit the flow? Check priority and both subnet/NIC
    associations.
6.  Does the guest OS firewall permit the connection?
7.  Does DNS resolve the hostname to the intended address?
8.  If a private endpoint is involved, is the private DNS zone linked
    and the endpoint connection approved?
9.  If traffic crosses a VPN, are the address spaces, routes, gateway
    settings, and on-premises device configuration correct?

For a VM-based lab, Azure CLI supports Network Watcher commands such as
`az network watcher test-ip-flow`, `az network watcher show-next-hop`,
and `az network watcher test-connectivity`. Run each command's `--help`
to see required arguments for your CLI version.

------------------------------------------------------------------------

## 14. Optional VM-based connectivity lab

A VM is useful for proving actual connectivity, but it incurs compute,
disk, and possibly public IP charges. Skip it if you only want to
practise resource configuration.

If you deploy VMs:

-   Use one small VM for a short session.
-   Prefer private IP connectivity and Bastion only if you have budgeted
    for it.
-   Avoid opening SSH/RDP to the whole internet.
-   Stop and deallocate VMs when you are not using them; stopping from
    inside the OS may not release compute billing.
-   Delete VM, NIC, disks, public IPs, and any related resources after
    practice, or delete the entire dedicated resource group.

Inspect VMs and NICs before cleanup:

``` bash
az vm list --resource-group "$RG" --show-details --output table
az network nic list --resource-group "$RG" --output table
az disk list --resource-group "$RG" --output table
az network public-ip list --resource-group "$RG" --output table
```

------------------------------------------------------------------------

## 15. One-command cleanup

For a dedicated lab, deleting the resource group is the most reliable
cleanup approach.

**First verify the variable points to the intended lab:**

``` bash
echo "Resource group to delete: [$RG]"
az group show --name "$RG" --output table
```

**Then delete:**

``` bash
az group delete \
  --name "$RG" \
  --yes \
  --no-wait
```

**Verify completion:**

``` bash
az group exists --name "$RG"
```

Repeat the check after a short wait until the result is `false`.

If it remains `true` for an unexpectedly long time, inspect the group
and any failed deployments. Do not assume `--no-wait` means deletion
completed; it only returns without waiting for the full operation.

### Check for leftovers outside the lab group

``` bash
az resource list \
  --query "[].{Name:name,Type:type,ResourceGroup:resourceGroup,Location:location}" \
  --output table
```

Review the list carefully. Do not bulk-delete resources from other
resource groups or shared subscriptions.

### Cost check

Open Azure Cost Analysis:
https://portal.azure.com/#view/Microsoft_Azure_CostManagement/Menu/\~/costanalysis

Select the correct subscription and date range. Group by resource or
service. Costs may appear with a delay, and deletion does not remove
charges already incurred.

------------------------------------------------------------------------

## 16. Recommended practice sequence

  -----------------------------------------------------------------------
  Session                 Hands-on focus          Cost profile
  ----------------------- ----------------------- -----------------------
  1                       VNet, address spaces,   Low
                          subnets, IP             
                          availability            

  2                       NSGs, rule priorities,  Low
                          ASGs, subnet            
                          associations            

  3                       Route tables, UDRs,     Low
                          next-hop concepts       

  4                       VNet peering,           Low
                          non-overlapping ranges, 
                          bidirectional state     

  5                       Azure DNS and Private   Check DNS pricing
                          DNS zone links/records  

  6                       Service endpoints and   Storage can incur
                          Storage network rules   charges

  7                       Private endpoints and   Can incur hourly/data
                          private DNS integration charges

  8                       Network Watcher and     Depends on
                          connectivity            tools/resources used
                          troubleshooting         

  9                       Bastion and VPN Gateway Potentially high
                          architecture; optional  ongoing cost
                          short deployment        

  10                      Load Balancer vs        Learn concepts first;
                          Application Gateway vs  services can be
                          Front Door vs Traffic   billable
                          Manager vs NAT Gateway  
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 17. AZ-104 networking self-check

You should be able to answer these without notes:

-   Why should VNet address spaces not overlap when peering?
-   Why is VNet peering non-transitive by default?
-   What does an NSG priority number control?
-   How do ASGs simplify security rules?
-   What is the difference between an NSG and a route table?
-   What is the difference between system routes and UDRs?
-   What does a `None` next hop do?
-   What is the difference between a public IP resource and a private
    IP?
-   What does a service endpoint provide, and what does it not provide?
-   How does a private endpoint differ from a service endpoint?
-   Why is private DNS important for private endpoints?
-   What are S2S, P2S, VNet-to-VNet VPN, and ExpressRoute?
-   What subnet name is required for Azure Bastion? What subnet name is
    required for a VPN gateway?
-   How do you investigate effective routes and effective security
    rules?
-   Why does deleting a resource group not necessarily remove resources
    created elsewhere?
-   Why can a budget alert fail to stop spending automatically?

## Official references

-   AZ-104 study guide:
    https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104
-   VNet peering:
    https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering
-   Azure CLI networking reference:
    https://learn.microsoft.com/en-us/cli/azure/network
-   VPN Gateway CLI reference:
    https://learn.microsoft.com/en-us/cli/azure/network/vnet-gateway
-   Azure Cost Management:
    https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/quick-acm-cost-analysis
-   Azure pricing calculator:
    https://azure.microsoft.com/pricing/calculator/

**Final rule:** create only the resources needed for the day's exercise,
record the resource group name, delete the whole group when finished,
verify deletion, and check Cost Analysis later.
