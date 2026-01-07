# Phase 8: Validation and Demo

## 🎯 What Are We Doing in This Phase?

This is the **final phase** - we're validating everything works and preparing demo scenarios for Indusind Bank!

**Goals:**
- ✅ Verify all components are working
- ✅ Test each demo scenario
- ✅ Show audit/accountability
- ✅ Test high availability
- ✅ Document for customer presentation

---

## 📋 Demo Scenarios Overview

| # | Scenario | What It Shows |
|---|----------|---------------|
| 1 | AD User Visible in IDM | Trust is working |
| 2 | Individual Login | No more shared accounts |
| 3 | L1 Restricted Access | Limited sudo commands |
| 4 | L2 Elevated Access | Service management |
| 5 | L3 Full Access | Super admin capabilities |
| 6 | HBAC Restriction | Server access control |
| 7 | Audit Trail | Individual accountability |
| 8 | High Availability | No single point of failure |

---

## 🔧 Pre-Validation Checklist

Before starting demos, verify all systems are running:

### Check AD Domain Controller
```bash
# RDP to AD DC and verify
Get-ADDomainController
Get-ADUser -Filter * -SearchBase "OU=Users,OU=Linux,DC=ad,DC=demo,DC=local"
```

### Check IDM Primary
```bash
ssh azureuser@<IDM_PRIMARY_IP>
sudo ipactl status  # All should be RUNNING
```

### Check IDM Replica
```bash
ssh azureuser@<IDM_REPLICA_IP>
sudo ipactl status  # All should be RUNNING
```

### Check RHEL Client
```bash
ssh azureuser@<RHEL_CLIENT_IP>
sudo systemctl status sssd  # Should be running
```

---

## Demo 1: AD User Visible in IDM

### 🎓 What This Shows
The trust between AD and IDM is working - AD users are visible in IDM.

### On IDM Server

```bash
ssh azureuser@<IDM_PRIMARY_IP>
sudo -i
kinit admin

# Show AD users are visible
echo "=== AD Users Visible in IDM ==="
id john.l1@ad.demo.local
id jane.l2@ad.demo.local
id mike.l3@ad.demo.local

# Show AD groups
echo "=== AD Groups Visible ==="
getent group linux_l1_team@ad.demo.local
getent group linux_l2_team@ad.demo.local
getent group linux_l3_team@ad.demo.local

# Show trust status
echo "=== Trust Status ==="
ipa trust-show ad.demo.local
```

### Key Points for Demo
- ✅ "Notice the AD users have Linux UIDs assigned automatically"
- ✅ "AD group memberships are visible"
- ✅ "No password synchronization needed - users use AD passwords"

---

## Demo 2: Individual Login (No More Shared Accounts!)

### 🎓 What This Shows
Each admin logs in with their OWN AD account - no more shared "unixadmin"!

### Scenario

```bash
# L1 User Login
ssh john.l1@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>
# Password: RedHat@123456!

# After login
whoami
# Output: john.l1@ad.demo.local

pwd
# Output: /home/john.l1@ad.demo.local

id
# Shows individual UID and groups
```

### Key Points for Demo
- ✅ "Each admin has their own account"
- ✅ "Home directory created automatically"
- ✅ "This is john.l1 - we know exactly who logged in"
- ✅ "RBI audit requirement: Individual accountability - CHECK!"

---

## Demo 3: L1 Restricted Access (View Only)

### 🎓 What This Shows
L1 operators can only VIEW - they cannot make changes or run dangerous commands.

### Scenario

```bash
# Login as L1 user
ssh john.l1@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>

# What CAN L1 do?
sudo cat /var/log/messages      # ✅ ALLOWED - view logs
sudo tail -f /var/log/secure    # ✅ ALLOWED - view logs
sudo df -h                      # ✅ ALLOWED - view disk
sudo systemctl status sshd      # ✅ ALLOWED - view status

# What CANNOT L1 do?
sudo systemctl restart sshd     # ❌ DENIED - can't restart
sudo vi /etc/hosts              # ❌ DENIED - can't edit
sudo yum install httpd          # ❌ DENIED - can't install
sudo su -                       # ❌ DENIED - can't become root
```

### Show Sudo Rules
```bash
# Show what L1 can do
sudo -l

# Expected output shows limited commands
```

### Key Points for Demo
- ✅ "L1 can view logs and check status - their job is monitoring"
- ✅ "L1 CANNOT restart services or make changes"
- ✅ "This protects production from accidental changes"
- ✅ "Principle of least privilege - L1 gets only what they need"

---

## Demo 4: L2 Elevated Access (Service Management)

### 🎓 What This Shows
L2 admins can do everything L1 can PLUS restart services and edit configs.

### Scenario

```bash
# Login as L2 user
ssh jane.l2@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>

# What CAN L2 do? (Everything L1 can)
sudo cat /var/log/messages      # ✅ ALLOWED
sudo df -h                      # ✅ ALLOWED

# PLUS service management
sudo systemctl status sshd      # ✅ ALLOWED
sudo systemctl restart sshd     # ✅ ALLOWED - L2 can restart!
sudo systemctl stop firewalld   # ✅ ALLOWED
sudo systemctl start firewalld  # ✅ ALLOWED

# PLUS config editing (careful!)
sudo vi /etc/hosts              # ✅ ALLOWED

# What CANNOT L2 do?
sudo su -                       # ❌ DENIED - still can't become root
sudo rm -rf /                   # ❌ DENIED - can't run arbitrary commands
```

### Key Points for Demo
- ✅ "L2 can restart services to fix issues"
- ✅ "L2 can edit configuration files"
- ✅ "But L2 still cannot become full root"
- ✅ "This is enough for 90% of admin tasks"

---

## Demo 5: L3 Full Access (Super Admin)

### 🎓 What This Shows
L3 super admins have FULL access for emergencies and complex issues.

### Scenario

```bash
# Login as L3 user
ssh mike.l3@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>

# L3 can do EVERYTHING
sudo su -                       # ✅ ALLOWED - become root!
whoami
# Output: root

# Can run any command
yum install httpd -y            # ✅ ALLOWED
systemctl enable httpd          # ✅ ALLOWED
cat /etc/shadow                 # ✅ ALLOWED - full access

# Can become other users
sudo su - oracle                # ✅ ALLOWED (if oracle exists)
```

### Key Points for Demo
- ✅ "L3 has full sudo access for emergencies"
- ✅ "This is equivalent to root access"
- ✅ "But it's STILL tracked - we know mike.l3 did this"
- ✅ "Only senior admins should be L3"

---

## Demo 6: HBAC Restriction (Server Access Control)

### 🎓 What This Shows
Different roles can access different servers - L1 blocked from production!

### Scenario Setup

First, let's verify HBAC rules:

```bash
# On IDM server
kinit admin

# Test L1 access to non-prod (client is in nonprod_servers)
ipa hbactest --user=john.l1@ad.demo.local --host=client.idm.demo.local --service=sshd
# Expected: Access granted: True

# Test L1 access to hypothetical prod server
# (If you had a prod server, L1 would be denied!)
```

### Show Different Access Levels

| User | Non-Prod Servers | Prod Servers |
|------|------------------|--------------|
| L1 (john.l1) | ✅ Allowed | ❌ Denied |
| L2 (jane.l2) | ✅ Allowed | ✅ Allowed |
| L3 (mike.l3) | ✅ Allowed | ✅ Allowed |

### Key Points for Demo
- ✅ "L1 operators work on non-prod - safe for learning"
- ✅ "L2/L3 can access production when needed"
- ✅ "This prevents L1 from accidentally breaking production"
- ✅ "Access is centrally managed in IDM"

---

## Demo 7: Audit Trail (Individual Accountability)

### 🎓 What This Shows
Every action is logged with the INDIVIDUAL user - not a shared account!

### Show Authentication Logs

```bash
# On RHEL Client
sudo tail -50 /var/log/secure | grep -E "john.l1|jane.l2|mike.l3"
```

**Example output:**
```
Jan 06 10:15:23 client sshd[12345]: Accepted password for john.l1@ad.demo.local from 10.0.1.x
Jan 06 10:15:45 client sudo[12346]: john.l1@ad.demo.local : TTY=pts/0 ; PWD=/home/john.l1 ; USER=root ; COMMAND=/usr/bin/cat /var/log/messages
Jan 06 10:20:12 client sshd[12350]: Accepted password for jane.l2@ad.demo.local from 10.0.1.x
Jan 06 10:20:33 client sudo[12351]: jane.l2@ad.demo.local : TTY=pts/1 ; PWD=/home/jane.l2 ; USER=root ; COMMAND=/usr/bin/systemctl restart sshd
```

### What The Audit Shows

| Field | Information |
|-------|-------------|
| Timestamp | When it happened |
| User | WHO did it (individual!) |
| TTY | Which terminal session |
| Command | WHAT they did |
| PWD | WHERE they were |

### Compare: Before vs After

**Before (Shared Account):**
```
Jan 06 10:15:45 sudo: unixadmin : COMMAND=/usr/bin/rm -rf /data
# WHO was unixadmin? Could be any of 50 people! 😱
```

**After (Individual Accounts):**
```
Jan 06 10:15:45 sudo: john.l1@ad.demo.local : COMMAND=/usr/bin/cat /var/log/messages
# We know EXACTLY who did this! ✅
```

### Key Points for Demo
- ✅ "Every login is tracked with individual username"
- ✅ "Every sudo command shows who ran it"
- ✅ "RBI audit: We can trace any action to a specific person"
- ✅ "No more 'who deleted that file?' mystery"

---

## Demo 8: High Availability (No Single Point of Failure)

### 🎓 What This Shows
If one IDM server fails, the other keeps working - no downtime!

### Scenario: Stop Primary, Verify Replica Works

**Step 1: Stop IDM Primary**
```bash
# On IDM Primary (10.0.1.5)
sudo ipactl stop
```

**Step 2: Test on RHEL Client**
```bash
# On RHEL Client - should STILL work!
ssh john.l1@ad.demo.local@<RHEL_CLIENT_IP>

# After login
id john.l1@ad.demo.local   # Still works!
sudo cat /var/log/messages # Still works!
```

**Step 3: Verify on Replica**
```bash
# On IDM Replica (10.0.1.7)
sudo ipactl status         # All RUNNING
kinit admin               # Still works!
ipa user-find             # Still works!
```

**Step 4: Restart Primary**
```bash
# On IDM Primary
sudo ipactl start
sudo ipactl status         # All RUNNING again
```

### Key Points for Demo
- ✅ "We stopped the primary IDM server completely"
- ✅ "Authentication kept working via the replica"
- ✅ "Zero downtime for users"
- ✅ "RBI requirement: No single point of failure - CHECK!"

---

## 📊 Summary Table for Customer

| Requirement | Solution | Status |
|-------------|----------|--------|
| Individual Accountability | Each admin uses AD account | ✅ |
| Role-Based Access (L1/L2/L3) | IDM Sudo Rules | ✅ |
| Server Access Control | IDM HBAC Rules | ✅ |
| Audit Trail | /var/log/secure shows individual user | ✅ |
| High Availability | Multi-master replication | ✅ |
| Use Existing AD | Trust relationship | ✅ |
| No Password Sync | Kerberos cross-realm auth | ✅ |
| Centralized Management | All rules in IDM | ✅ |

---

## 🎤 Demo Script (5-Minute Version)

### Intro (30 seconds)
"Let me show you how Red Hat IDM with AD Trust solves your RBI compliance requirements."

### Demo 1: Trust (30 seconds)
"First, notice that AD users are visible in IDM without password sync."
```bash
id john.l1@ad.demo.local
```

### Demo 2: Individual Login (30 seconds)
"Each admin logs in with their own account - no more shared unixadmin."
```bash
ssh john.l1@ad.demo.local@client
whoami
```

### Demo 3: L1 Restrictions (1 minute)
"L1 operators can view but not change."
```bash
sudo cat /var/log/messages    # Works
sudo systemctl restart sshd   # Denied!
```

### Demo 4: Audit (30 seconds)
"Every action is logged with the individual user."
```bash
sudo tail /var/log/secure | grep john.l1
```

### Demo 5: HA (1 minute)
"And if one IDM server fails, the other keeps working."
[Stop primary, show login still works]

### Close (30 seconds)
"This gives you individual accountability, role-based access, and high availability - all centrally managed."

---

## 🧹 Post-Demo Cleanup (Optional)

If you want to clean up the Azure resources:

```bash
# Delete everything
az group delete --name idm-ad-demo-rg --yes --no-wait

# Verify deletion started
az group list --output table
```

---

## 📝 Quick Reference Card

### IP Addresses
| Server | Private IP | Public IP |
|--------|------------|-----------|
| AD DC | 10.0.1.4 | (from Azure) |
| IDM Primary | 10.0.1.5 | (from Azure) |
| IDM Replica | 10.0.1.7 | (from Azure) |
| RHEL Client | 10.0.1.6 | (from Azure) |

### Credentials
| Account | Username | Password |
|---------|----------|----------|
| Azure VMs | azureuser | RedHat@123456! |
| IDM Admin | admin | RedHat@123456! |
| L1 User | john.l1@ad.demo.local | RedHat@123456! |
| L2 User | jane.l2@ad.demo.local | RedHat@123456! |
| L3 User | mike.l3@ad.demo.local | RedHat@123456! |

### Key Commands
```bash
# Check IDM status
ipactl status

# Check AD trust
ipa trust-show ad.demo.local

# Test HBAC
ipa hbactest --user=john.l1@ad.demo.local --host=client.idm.demo.local --service=sshd

# Clear SSSD cache (on clients)
sss_cache -E && systemctl restart sssd

# View audit logs
tail -f /var/log/secure
```

---

## 🎉 Congratulations!

You've successfully built and validated a complete IDM + AD Trust demo lab!

This demonstrates:
- ✅ Individual accountability (RBI compliance)
- ✅ Role-based access control (L1/L2/L3)
- ✅ Host-based access control
- ✅ High availability
- ✅ Integration with existing AD
- ✅ Centralized management

**Ready for the Indusind Bank demo! 🚀**

