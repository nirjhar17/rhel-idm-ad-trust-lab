# Red Hat IDM + Active Directory Integration Lab in Azure

## Purpose
Demo environment for Indusind Bank - RBI Compliance RBAC Implementation

## Lab Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure VNet                                │
│                      10.0.0.0/16                                 │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Subnet: 10.0.1.0/24                         │    │
│  │                                                          │    │
│  │  ┌──────────────┐           ┌──────────────┐            │    │
│  │  │ Windows      │           │ RHEL 9       │            │    │
│  │  │ Server 2022  │           │ IDM Server 1 │            │    │
│  │  │ (AD DC)      │◄─────────►│ (Primary)    │            │    │
│  │  │              │   TRUST   │              │            │    │
│  │  │ 10.0.1.4     │           │ 10.0.1.5     │            │    │
│  │  │ ad.demo.local│           │idm.demo.local│            │    │
│  │  └──────────────┘           └──────────────┘            │    │
│  │                                    │                     │    │
│  │                                    │ Replication         │    │
│  │                                    ▼                     │    │
│  │  ┌──────────────┐           ┌──────────────┐            │    │
│  │  │ RHEL 9       │           │ RHEL 9       │            │    │
│  │  │ Client       │           │ IDM Server 2 │            │    │
│  │  │              │           │ (Replica/HA) │            │    │
│  │  │ 10.0.1.6     │           │ 10.0.1.7     │            │    │
│  │  │client.idm.   │           │idm2.idm.     │            │    │
│  │  │demo.local    │           │demo.local    │            │    │
│  │  └──────────────┘           └──────────────┘            │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Reference

| VM | Hostname | Private IP | Role | VM Size |
|----|----------|------------|------|---------|
| Windows Server 2022 | ad.ad.demo.local | 10.0.1.4 | AD Domain Controller | Standard_B2ms |
| RHEL 9 | idm.idm.demo.local | 10.0.1.5 | IDM Primary Server | Standard_B2ms |
| RHEL 9 | idm2.idm.demo.local | 10.0.1.7 | IDM Replica (HA) | Standard_B2ms |
| RHEL 9 | client.idm.demo.local | 10.0.1.6 | Test Client | Standard_B2s |

## Credentials (Demo Only)

| Account | Username | Password |
|---------|----------|----------|
| Azure VMs | azureuser | RedHat@123456! |
| AD Domain Admin | Administrator | RedHat@123456! |
| IDM Admin | admin | RedHat@123456! |
| AD Test Users | john.l1, jane.l2, mike.l3 | RedHat@123456! |

## Estimated Cost

- Running 24/7: ~$230-275/month
- Tip: Deallocate VMs when not in use (pay only for storage ~$25-30/month)

---

# Phase 1: Azure Infrastructure Setup

## Step 1.1: Verify Azure Login

```bash
az account show --output table
```

If not logged in:
```bash
az login
```

**Expected Output:**
```
EnvironmentName    HomeTenantId    IsDefault    Name         State    TenantId
-----------------  --------------  -----------  -----------  -------  -----------
AzureCloud         xxxxx-xxxxx     True         your-sub     Enabled  xxxxx-xxxxx
```

## Step 1.2: Create Resource Group

```bash
az group create \
  --name idm-ad-demo-rg \
  --location eastus \
  --tags "Project=IDM-AD-Demo" "Purpose=Indusind-Bank-Demo"
```

**Expected Output:**
```json
{
  "id": "/subscriptions/.../resourceGroups/idm-ad-demo-rg",
  "location": "eastus",
  "name": "idm-ad-demo-rg",
  "properties": {
    "provisioningState": "Succeeded"
  }
}
```

## Step 1.3: Create Virtual Network and Subnet

```bash
az network vnet create \
  --resource-group idm-ad-demo-rg \
  --name idm-ad-vnet \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name idm-ad-subnet \
  --subnet-prefixes 10.0.1.0/24
```

**Verify:**
```bash
az network vnet subnet list \
  --resource-group idm-ad-demo-rg \
  --vnet-name idm-ad-vnet \
  --output table
```

## Step 1.4: Create Network Security Group

```bash
az network nsg create \
  --resource-group idm-ad-demo-rg \
  --name idm-ad-nsg
```

## Step 1.5: Add NSG Rules

### SSH Access (from Internet)
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-SSH" \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 22 \
  --source-address-prefixes "*" \
  --destination-address-prefixes "*"
```

### RDP Access (from Internet - for Windows AD)
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-RDP" \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 3389 \
  --source-address-prefixes "*" \
  --destination-address-prefixes "*"
```

### DNS (TCP and UDP)
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-DNS-TCP" \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 53 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-DNS-UDP" \
  --priority 201 \
  --direction Inbound \
  --access Allow \
  --protocol Udp \
  --destination-port-ranges 53 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

### Kerberos (TCP and UDP 88, 464)
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-Kerberos-88-TCP" \
  --priority 210 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 88 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-Kerberos-88-UDP" \
  --priority 211 \
  --direction Inbound \
  --access Allow \
  --protocol Udp \
  --destination-port-ranges 88 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-Kerberos-464-TCP" \
  --priority 212 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 464 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-Kerberos-464-UDP" \
  --priority 213 \
  --direction Inbound \
  --access Allow \
  --protocol Udp \
  --destination-port-ranges 464 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

### LDAP and LDAPS
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-LDAP" \
  --priority 220 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 389 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-LDAPS" \
  --priority 221 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 636 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

### SMB (Required for AD Trust)
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-SMB" \
  --priority 230 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 445 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

### HTTP and HTTPS (IDM Web UI)
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-HTTP" \
  --priority 240 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80 \
  --source-address-prefixes "*" \
  --destination-address-prefixes "*"
```

```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-HTTPS" \
  --priority 241 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --source-address-prefixes "*" \
  --destination-address-prefixes "*"
```

### NTP (Critical for Kerberos)
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-NTP" \
  --priority 250 \
  --direction Inbound \
  --access Allow \
  --protocol Udp \
  --destination-port-ranges 123 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

### LDAP Global Catalog
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-LDAP-GC" \
  --priority 260 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 3268-3269 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "*"
```

### Allow All VNet Internal Traffic
```bash
az network nsg rule create \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --name "Allow-VNet-Internal" \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --protocol "*" \
  --destination-port-ranges "*" \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-address-prefixes "10.0.0.0/16"
```

## Step 1.6: Associate NSG with Subnet

```bash
az network vnet subnet update \
  --resource-group idm-ad-demo-rg \
  --vnet-name idm-ad-vnet \
  --name idm-ad-subnet \
  --network-security-group idm-ad-nsg
```

**Verify NSG Rules:**
```bash
az network nsg rule list \
  --resource-group idm-ad-demo-rg \
  --nsg-name idm-ad-nsg \
  --output table
```

## Step 1.7: Create Public IPs

```bash
az network public-ip create \
  --resource-group idm-ad-demo-rg \
  --name "ad-dc-pip" \
  --allocation-method Static \
  --sku Standard
```

```bash
az network public-ip create \
  --resource-group idm-ad-demo-rg \
  --name "idm-server1-pip" \
  --allocation-method Static \
  --sku Standard
```

```bash
az network public-ip create \
  --resource-group idm-ad-demo-rg \
  --name "idm-server2-pip" \
  --allocation-method Static \
  --sku Standard
```

```bash
az network public-ip create \
  --resource-group idm-ad-demo-rg \
  --name "rhel-client-pip" \
  --allocation-method Static \
  --sku Standard
```

**Verify and note down the Public IPs:**
```bash
az network public-ip list \
  --resource-group idm-ad-demo-rg \
  --output table
```

## Step 1.8: Create Network Interfaces with Static Private IPs

### AD DC NIC (10.0.1.4)
```bash
az network nic create \
  --resource-group idm-ad-demo-rg \
  --name "ad-dc-nic" \
  --vnet-name idm-ad-vnet \
  --subnet idm-ad-subnet \
  --private-ip-address "10.0.1.4" \
  --public-ip-address "ad-dc-pip"
```

### IDM Server 1 NIC (10.0.1.5)
```bash
az network nic create \
  --resource-group idm-ad-demo-rg \
  --name "idm-server1-nic" \
  --vnet-name idm-ad-vnet \
  --subnet idm-ad-subnet \
  --private-ip-address "10.0.1.5" \
  --public-ip-address "idm-server1-pip"
```

### IDM Server 2 NIC (10.0.1.7)
```bash
az network nic create \
  --resource-group idm-ad-demo-rg \
  --name "idm-server2-nic" \
  --vnet-name idm-ad-vnet \
  --subnet idm-ad-subnet \
  --private-ip-address "10.0.1.7" \
  --public-ip-address "idm-server2-pip"
```

### RHEL Client NIC (10.0.1.6)
```bash
az network nic create \
  --resource-group idm-ad-demo-rg \
  --name "rhel-client-nic" \
  --vnet-name idm-ad-vnet \
  --subnet idm-ad-subnet \
  --private-ip-address "10.0.1.6" \
  --public-ip-address "rhel-client-pip"
```

**Verify NICs:**
```bash
az network nic list \
  --resource-group idm-ad-demo-rg \
  --output table \
  --query "[].{Name:name, PrivateIP:ipConfigurations[0].privateIPAddress}"
```

## Step 1.9: Create Windows Server 2022 VM (AD DC)

```bash
az vm create \
  --resource-group idm-ad-demo-rg \
  --name ad-dc \
  --nics "ad-dc-nic" \
  --image "MicrosoftWindowsServer:WindowsServer:2022-datacenter-azure-edition:latest" \
  --size Standard_B2ms \
  --admin-username azureuser \
  --admin-password 'RedHat@123456!' \
  --os-disk-size-gb 128 \
  --output table
```

**Expected Output:**
```
ResourceGroup    PowerState    PublicIpAddress    PrivateIpAddress
---------------  ------------  -----------------  ------------------
idm-ad-demo-rg   VM running    xx.xx.xx.xx        10.0.1.4
```

## Step 1.10: Create RHEL 9 VMs

### Create IDM Server 1
```bash
az vm create \
  --resource-group idm-ad-demo-rg \
  --name idm-server1 \
  --nics "idm-server1-nic" \
  --image "RedHat:RHEL:9-lvm-gen2:latest" \
  --size Standard_B2ms \
  --admin-username azureuser \
  --admin-password 'RedHat@123456!' \
  --os-disk-size-gb 128 \
  --output table
```

### Create IDM Server 2
```bash
az vm create \
  --resource-group idm-ad-demo-rg \
  --name idm-server2 \
  --nics "idm-server2-nic" \
  --image "RedHat:RHEL:9-lvm-gen2:latest" \
  --size Standard_B2ms \
  --admin-username azureuser \
  --admin-password 'RedHat@123456!' \
  --os-disk-size-gb 128 \
  --output table
```

### Create RHEL Client
```bash
az vm create \
  --resource-group idm-ad-demo-rg \
  --name rhel-client \
  --nics "rhel-client-nic" \
  --image "RedHat:RHEL:9-lvm-gen2:latest" \
  --size Standard_B2s \
  --admin-username azureuser \
  --admin-password 'RedHat@123456!' \
  --os-disk-size-gb 64 \
  --output table
```

## Step 1.11: Verify All VMs

```bash
az vm list \
  --resource-group idm-ad-demo-rg \
  --output table \
  --show-details
```

**Expected Output:**
```
Name          ResourceGroup    PowerState    PublicIps         PrivateIps
------------  ---------------  ------------  ----------------  ------------
ad-dc         idm-ad-demo-rg   VM running    xx.xx.xx.xx       10.0.1.4
idm-server1   idm-ad-demo-rg   VM running    xx.xx.xx.xx       10.0.1.5
idm-server2   idm-ad-demo-rg   VM running    xx.xx.xx.xx       10.0.1.7
rhel-client   idm-ad-demo-rg   VM running    xx.xx.xx.xx       10.0.1.6
```

## Step 1.12: Get Public IPs for SSH/RDP Access

```bash
echo "=== VM Connection Information ==="
echo ""
echo "AD DC (RDP):"
az vm show -d -g idm-ad-demo-rg -n ad-dc --query publicIps -o tsv
echo ""
echo "IDM Server 1 (SSH):"
az vm show -d -g idm-ad-demo-rg -n idm-server1 --query publicIps -o tsv
echo ""
echo "IDM Server 2 (SSH):"
az vm show -d -g idm-ad-demo-rg -n idm-server2 --query publicIps -o tsv
echo ""
echo "RHEL Client (SSH):"
az vm show -d -g idm-ad-demo-rg -n rhel-client --query publicIps -o tsv
```

## Step 1.13: Test Connectivity

### SSH to RHEL VMs
```bash
ssh azureuser@<PUBLIC_IP>
```
Password: `RedHat@123456!`

### RDP to Windows AD DC
- Open Remote Desktop Connection
- Enter the public IP of ad-dc
- Username: `azureuser`
- Password: `RedHat@123456!`

---

## Phase 1 Verification Checklist

| Item | Status | How to Verify |
|------|--------|---------------|
| Resource Group Created | ☐ | `az group show -n idm-ad-demo-rg` |
| VNet/Subnet Created | ☐ | `az network vnet show -g idm-ad-demo-rg -n idm-ad-vnet` |
| NSG with All Rules | ☐ | `az network nsg rule list -g idm-ad-demo-rg --nsg-name idm-ad-nsg -o table` |
| AD DC VM Running | ☐ | RDP to public IP |
| IDM Server 1 Running | ☐ | SSH to public IP |
| IDM Server 2 Running | ☐ | SSH to public IP |
| RHEL Client Running | ☐ | SSH to public IP |
| Static IPs Correct | ☐ | Check private IPs match 10.0.1.4/5/6/7 |

---

## Troubleshooting Phase 1

### Issue: Cannot SSH to RHEL VMs
```bash
az vm get-instance-view -g idm-ad-demo-rg -n idm-server1 --query instanceView.statuses[1].displayStatus

az network nsg rule show -g idm-ad-demo-rg --nsg-name idm-ad-nsg -n Allow-SSH

az vm restart -g idm-ad-demo-rg -n idm-server1
```

### Issue: Cannot RDP to Windows VM
```bash
az vm get-instance-view -g idm-ad-demo-rg -n ad-dc --query instanceView.statuses[1].displayStatus

az network nsg rule show -g idm-ad-demo-rg --nsg-name idm-ad-nsg -n Allow-RDP
```

### Issue: VMs Cannot Ping Each Other
- This is normal - Azure VNet doesn't allow ICMP by default
- Use other tests like: `nc -zv 10.0.1.4 22` (from Linux)

---

## Cost Management Tips

### Stop VMs When Not in Use
```bash
az vm deallocate -g idm-ad-demo-rg -n ad-dc --no-wait
az vm deallocate -g idm-ad-demo-rg -n idm-server1 --no-wait
az vm deallocate -g idm-ad-demo-rg -n idm-server2 --no-wait
az vm deallocate -g idm-ad-demo-rg -n rhel-client --no-wait
```

### Start VMs When Needed
```bash
az vm start -g idm-ad-demo-rg -n ad-dc --no-wait
az vm start -g idm-ad-demo-rg -n idm-server1 --no-wait
az vm start -g idm-ad-demo-rg -n idm-server2 --no-wait
az vm start -g idm-ad-demo-rg -n rhel-client --no-wait
```

---

## Next Steps

Once Phase 1 is complete, proceed to:
- **Phase 2**: Active Directory Setup (see `phase2-ad-setup.md`)

---

## Cleanup (Delete Everything)

**⚠️ WARNING: This will delete ALL resources permanently!**

```bash
az group delete --name idm-ad-demo-rg --yes --no-wait
```
