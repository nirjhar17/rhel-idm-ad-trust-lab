# Phase 2: Active Directory Setup

## Prerequisites
- Phase 1 completed
- Windows Server 2022 VM (ad-dc) is running
- RDP access to the AD DC

## Connection Information

| Item | Value |
|------|-------|
| Public IP | (from Phase 1) |
| Username | azureuser |
| Password | RedHat@123456! |
| Domain Name | ad.demo.local |

---

## Step 2.1: RDP to Windows Server

1. Open Remote Desktop Connection (mstsc.exe)
2. Enter the public IP of ad-dc
3. Click Connect
4. Enter credentials:
   - Username: `azureuser`
   - Password: `RedHat@123456!`

---

## Step 2.2: Set Static IP and Hostname

### Open PowerShell as Administrator

```powershell
# Set hostname
Rename-Computer -NewName "AD" -Force

# Verify network settings (should already be 10.0.1.4)
Get-NetIPAddress -InterfaceAlias "Ethernet*" | Where-Object {$_.AddressFamily -eq "IPv4"}
```

---

## Step 2.3: Install AD DS Role

### Via PowerShell (Recommended)

```powershell
# Install AD DS Role with management tools
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Verify installation
Get-WindowsFeature AD-Domain-Services
```

**Expected Output:**
```
Display Name                                            Name                       Install State
------------                                            ----                       -------------
[X] Active Directory Domain Services                    AD-Domain-Services         Installed
```

---

## Step 2.4: Promote to Domain Controller

```powershell
# Import AD DS Deployment module
Import-Module ADDSDeployment

# Promote to Domain Controller
Install-ADDSForest `
    -DomainName "ad.demo.local" `
    -DomainNetBIOSName "AD" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDNS:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "RedHat@123456!" -AsPlainText -Force) `
    -Force:$true
```

**⚠️ The server will automatically restart after this command.**

---

## Step 2.5: Reconnect After Restart

1. Wait 3-5 minutes for the server to restart
2. RDP back to the server
3. Login with domain credentials:
   - Username: `AD\azureuser` or `azureuser@ad.demo.local`
   - Password: `RedHat@123456!`

---

## Step 2.6: Verify AD DS Installation

### Open PowerShell as Administrator

```powershell
# Verify domain controller
Get-ADDomainController

# Verify domain
Get-ADDomain

# Verify DNS is running
Get-Service DNS

# Verify AD DS is running
Get-Service NTDS
```

---

## Step 2.7: Create Organizational Units (OUs)

```powershell
# Create OU structure for Linux users
New-ADOrganizationalUnit -Name "Linux" -Path "DC=ad,DC=demo,DC=local"
New-ADOrganizationalUnit -Name "Users" -Path "OU=Linux,DC=ad,DC=demo,DC=local"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=Linux,DC=ad,DC=demo,DC=local"

# Verify OUs
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```

---

## Step 2.8: Create AD Groups for Linux Teams

```powershell
# Create L1 Team Group
New-ADGroup -Name "Linux_L1_Team" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groups,OU=Linux,DC=ad,DC=demo,DC=local" `
    -Description "Linux L1 Operators - Basic monitoring access"

# Create L2 Team Group
New-ADGroup -Name "Linux_L2_Team" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groups,OU=Linux,DC=ad,DC=demo,DC=local" `
    -Description "Linux L2 Admins - Service restart and config access"

# Create L3 Team Group
New-ADGroup -Name "Linux_L3_Team" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groups,OU=Linux,DC=ad,DC=demo,DC=local" `
    -Description "Linux L3 Super Admins - Full root access"

# Verify groups
Get-ADGroup -Filter * -SearchBase "OU=Groups,OU=Linux,DC=ad,DC=demo,DC=local" | Select-Object Name, Description
```

---

## Step 2.9: Create Test Users

### L1 Users (Basic Operators)

```powershell
# Create john.l1
New-ADUser -Name "John Operator" `
    -GivenName "John" `
    -Surname "Operator" `
    -SamAccountName "john.l1" `
    -UserPrincipalName "john.l1@ad.demo.local" `
    -Path "OU=Users,OU=Linux,DC=ad,DC=demo,DC=local" `
    -AccountPassword (ConvertTo-SecureString "RedHat@123456!" -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $true

# Create sarah.l1
New-ADUser -Name "Sarah Operator" `
    -GivenName "Sarah" `
    -Surname "Operator" `
    -SamAccountName "sarah.l1" `
    -UserPrincipalName "sarah.l1@ad.demo.local" `
    -Path "OU=Users,OU=Linux,DC=ad,DC=demo,DC=local" `
    -AccountPassword (ConvertTo-SecureString "RedHat@123456!" -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $true

# Add to L1 group
Add-ADGroupMember -Identity "Linux_L1_Team" -Members "john.l1", "sarah.l1"
```

### L2 Users (Senior Admins)

```powershell
# Create jane.l2
New-ADUser -Name "Jane Admin" `
    -GivenName "Jane" `
    -Surname "Admin" `
    -SamAccountName "jane.l2" `
    -UserPrincipalName "jane.l2@ad.demo.local" `
    -Path "OU=Users,OU=Linux,DC=ad,DC=demo,DC=local" `
    -AccountPassword (ConvertTo-SecureString "RedHat@123456!" -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $true

# Add to L2 group
Add-ADGroupMember -Identity "Linux_L2_Team" -Members "jane.l2"
```

### L3 Users (Super Admins)

```powershell
# Create mike.l3
New-ADUser -Name "Mike Super" `
    -GivenName "Mike" `
    -Surname "Super" `
    -SamAccountName "mike.l3" `
    -UserPrincipalName "mike.l3@ad.demo.local" `
    -Path "OU=Users,OU=Linux,DC=ad,DC=demo,DC=local" `
    -AccountPassword (ConvertTo-SecureString "RedHat@123456!" -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $true

# Add to L3 group
Add-ADGroupMember -Identity "Linux_L3_Team" -Members "mike.l3"
```

---

## Step 2.10: Verify Users and Group Memberships

```powershell
# List all users in Linux OU
Get-ADUser -Filter * -SearchBase "OU=Users,OU=Linux,DC=ad,DC=demo,DC=local" | 
    Select-Object Name, SamAccountName, UserPrincipalName

# Verify L1 group members
Get-ADGroupMember -Identity "Linux_L1_Team" | Select-Object Name, SamAccountName

# Verify L2 group members
Get-ADGroupMember -Identity "Linux_L2_Team" | Select-Object Name, SamAccountName

# Verify L3 group members
Get-ADGroupMember -Identity "Linux_L3_Team" | Select-Object Name, SamAccountName
```

**Expected Output:**
```
L1 Group Members:
- john.l1
- sarah.l1

L2 Group Members:
- jane.l2

L3 Group Members:
- mike.l3
```

---

## Step 2.11: Configure DNS Forwarder for IDM Domain

This step will be completed after IDM is installed. For now, just verify DNS is working:

```powershell
# Verify DNS zones
Get-DnsServerZone

# Verify DNS forwarders (should be Azure DNS or empty)
Get-DnsServerForwarder
```

---

## Step 2.12: Configure Firewall for AD Trust (Optional)

Azure NSG already handles this, but verify Windows Firewall is not blocking:

```powershell
# Check Windows Firewall status
Get-NetFirewallProfile | Select-Object Name, Enabled

# If needed, add rules for IDM trust (usually already open in domain profile)
# Kerberos, LDAP, DNS, etc. are typically open by default on DCs
```

---

## Phase 2 Verification Checklist

| Item | Status | How to Verify |
|------|--------|---------------|
| AD DS Role Installed | ☐ | `Get-WindowsFeature AD-Domain-Services` |
| Domain Controller Promoted | ☐ | `Get-ADDomainController` |
| Domain: ad.demo.local | ☐ | `Get-ADDomain` |
| DNS Running | ☐ | `Get-Service DNS` |
| OU: Linux Created | ☐ | `Get-ADOrganizationalUnit -Filter *` |
| Group: Linux_L1_Team | ☐ | `Get-ADGroup Linux_L1_Team` |
| Group: Linux_L2_Team | ☐ | `Get-ADGroup Linux_L2_Team` |
| Group: Linux_L3_Team | ☐ | `Get-ADGroup Linux_L3_Team` |
| User: john.l1 | ☐ | `Get-ADUser john.l1` |
| User: sarah.l1 | ☐ | `Get-ADUser sarah.l1` |
| User: jane.l2 | ☐ | `Get-ADUser jane.l2` |
| User: mike.l3 | ☐ | `Get-ADUser mike.l3` |

---

## Quick Reference Commands

### List All Users
```powershell
Get-ADUser -Filter * -SearchBase "OU=Users,OU=Linux,DC=ad,DC=demo,DC=local" | 
    Format-Table Name, SamAccountName, Enabled
```

### List All Groups
```powershell
Get-ADGroup -Filter * -SearchBase "OU=Groups,OU=Linux,DC=ad,DC=demo,DC=local" |
    Format-Table Name, GroupScope, GroupCategory
```

### Check Group Membership
```powershell
Get-ADPrincipalGroupMembership john.l1 | Select-Object Name
```

### Reset User Password
```powershell
Set-ADAccountPassword -Identity "john.l1" -NewPassword (ConvertTo-SecureString "NewPassword123!" -AsPlainText -Force) -Reset
```

---

## Troubleshooting

### Issue: Cannot login after DC promotion
- Wait 3-5 minutes for services to start
- Use domain credentials: `AD\azureuser`

### Issue: Users cannot authenticate
```powershell
# Check if user account is enabled
Get-ADUser john.l1 -Properties Enabled

# Enable if needed
Enable-ADAccount -Identity john.l1
```

### Issue: DNS not resolving
```powershell
# Check DNS service
Get-Service DNS | Select-Object Status

# Restart if needed
Restart-Service DNS
```

---

## Users Summary Table

| Username | Full Name | Email | AD Group | Password |
|----------|-----------|-------|----------|----------|
| john.l1 | John Operator | john.l1@ad.demo.local | Linux_L1_Team | RedHat@123456! |
| sarah.l1 | Sarah Operator | sarah.l1@ad.demo.local | Linux_L1_Team | RedHat@123456! |
| jane.l2 | Jane Admin | jane.l2@ad.demo.local | Linux_L2_Team | RedHat@123456! |
| mike.l3 | Mike Super | mike.l3@ad.demo.local | Linux_L3_Team | RedHat@123456! |

---

## Next Steps

Once Phase 2 is complete, proceed to:
- **Phase 3**: IDM Primary Server Setup (see `phase3-idm-setup.md`)


