# Phase 4: AD Trust Configuration

## 🎯 What Are We Doing in This Phase?

We're creating a **trust relationship** between Active Directory and IDM. Think of it as building a bridge between two kingdoms - after this, AD users can access Linux servers managed by IDM!

**Simple Analogy:**
```
Before Trust:                        After Trust:
┌─────────┐    ┌─────────┐          ┌─────────┐    ┌─────────┐
│   AD    │    │   IDM   │          │   AD    │◄──►│   IDM   │
│ Kingdom │ ✗  │ Kingdom │          │ Kingdom │    │ Kingdom │
│         │    │         │          │         │    │         │
│ Users   │    │ Linux   │          │ Users   │    │ Linux   │
│ can't   │    │ servers │          │ CAN     │    │ servers │
│ enter   │    │         │          │ enter!  │    │         │
└─────────┘    └─────────┘          └─────────┘    └─────────┘
```

**After this phase:**
- ✅ AD users (john.l1, jane.l2, mike.l3) visible in IDM
- ✅ AD users can SSH to Linux servers
- ✅ AD users keep their AD passwords (no sync needed!)
- ✅ IDM can create sudo/HBAC rules for AD users

---

## 📚 Key Concepts to Understand

### What is a Trust?

A **trust** is an agreement between two identity systems to recognize each other's users.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Trust Relationship                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────┐         ┌──────────────────┐            │
│   │                  │         │                  │            │
│   │  Active Directory│  TRUST  │       IDM        │            │
│   │  (ad.demo.local) │◄───────►│ (idm.demo.local) │            │
│   │                  │         │                  │            │
│   │  "I vouch for    │         │ "I trust AD's    │            │
│   │   these users"   │         │  users as valid" │            │
│   │                  │         │                  │            │
│   └──────────────────┘         └──────────────────┘            │
│                                                                  │
│   AD says: "john.l1 is a valid user with password X"           │
│   IDM says: "OK, I trust you. john.l1 can access my servers"   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Types of Trust

| Trust Type | Direction | Description |
|------------|-----------|-------------|
| **One-way** | AD → IDM | AD users can access IDM resources |
| **Two-way** | AD ↔ IDM | Both can access each other's resources |

**We're creating:** One-way trust (AD users access IDM-managed Linux servers)

### How Trust Uses DNS

DNS is **critical** for trust! Both sides must be able to find each other.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DNS Resolution Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   IDM asks: "Where is ad.demo.local?"                           │
│        │                                                         │
│        │ DNS Query                                              │
│        ▼                                                         │
│   AD DNS (10.0.1.4) responds: "ad.demo.local is at 10.0.1.4"   │
│                                                                  │
│   AD asks: "Where is idm.demo.local?"                           │
│        │                                                         │
│        │ DNS Query                                              │
│        ▼                                                         │
│   IDM DNS (10.0.1.5) responds: "idm.demo.local is at 10.0.1.5" │
│                                                                  │
│   Both can find each other = Trust can be established!          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What is Kerberos Cross-Realm Trust?

Kerberos "realms" are like kingdoms. Each realm has its own authentication server.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kerberos Cross-Realm Trust                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Realm: AD.DEMO.LOCAL          Realm: IDM.DEMO.LOCAL          │
│   ┌──────────────────┐          ┌──────────────────┐           │
│   │  AD KDC          │   Trust  │  IDM KDC         │           │
│   │  (Ticket Office) │◄────────►│  (Ticket Office) │           │
│   └──────────────────┘          └──────────────────┘           │
│                                                                  │
│   When john.l1@AD.DEMO.LOCAL wants to access a Linux server:   │
│                                                                  │
│   1. john.l1 gets ticket from AD KDC                           │
│   2. AD KDC says "I trust IDM, here's a referral"              │
│   3. john.l1 presents referral to IDM KDC                      │
│   4. IDM KDC gives john.l1 a ticket for the Linux server       │
│   5. john.l1 accesses the Linux server!                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What is SMB/CIFS?

**SMB (Server Message Block)** is a protocol used by Windows for file sharing AND trust establishment.

**Why IDM needs SMB:**
- Trust negotiation uses SMB protocols
- SID (Security ID) lookups use SMB
- AD group membership resolution uses SMB

```bash
# This is why we run ipa-adtrust-install - it sets up Samba on IDM
ipa-adtrust-install  # Installs Samba components for AD trust
```

---

## 🔧 Prerequisites

Before starting:
- ✅ Phase 1-3 completed
- ✅ AD Domain Controller running with users created
- ✅ IDM Primary Server running
- ✅ Both servers can ping each other

---

## Part A: Configure AD DNS to Forward IDM Queries

### 🎓 What We're Doing

Telling AD's DNS server: "If anyone asks about idm.demo.local, forward the question to the IDM server."

### Step 4.1: RDP to Windows AD DC

```
RDP to: <AD_DC_PUBLIC_IP>
Username: AD\azureuser
Password: RedHat@123456!
```

### Step 4.2: Add DNS Conditional Forwarder (GUI Method)

1. Open **Server Manager** → **Tools** → **DNS**
2. Expand your server name
3. Right-click **Conditional Forwarders** → **New Conditional Forwarder**
4. Enter:
   - **DNS Domain:** `idm.demo.local`
   - **IP Address:** `10.0.1.5` (IDM server)
5. Click **OK**

> **Note:** You may see a validation warning (red X) - this is OK, click OK anyway.

### Step 4.3: Verify DNS Forwarding (PowerShell)

```powershell
# Test if AD can resolve IDM domain
nslookup idm.idm.demo.local
nslookup _kerberos._tcp.idm.demo.local
```

**Expected:** Should return 10.0.1.5

---

## Part B: Install Trust Components on IDM

### 🎓 What We're Doing

Installing Samba/CIFS components that allow IDM to communicate with AD using Windows protocols.

### Step 4.4: SSH to IDM Primary Server

```bash
ssh azureuser@<IDM_SERVER1_PUBLIC_IP>
sudo -i
kinit admin
# Password: RedHat@123456!
```

### Step 4.5: Configure DNS Forwarding for AD Domain

**What:** Tell IDM DNS to forward AD domain queries to AD server.

```bash
# Add forwarding zone for AD domain
ipa dnsforwardzone-add ad.demo.local --forwarder=10.0.1.4 --forward-policy=only

# Verify
nslookup ad.demo.local localhost
```

**Expected:** Should return 10.0.1.4

### ⚠️ Fix DNSSEC If Needed

If you see `SERVFAIL` errors, disable DNSSEC validation:

```bash
# Edit DNS config
vi /etc/named/ipa-options-ext.conf

# Add or change this line:
dnssec-validation no;

# Restart DNS
systemctl restart named

# Test again
nslookup ad.demo.local localhost
```

### Step 4.6: Install AD Trust Components

**What:** This command installs Samba and configures IDM to talk to AD.

```bash
ipa-adtrust-install --netbios-name=IDM -a 'RedHat@123456!'
```

**When prompted:**
- **Do you wish to continue?** → `yes`
- **Enable trusted domains support in slapi-nis?** → `yes`

**What this does:**
1. Configures Samba for AD communication
2. Adds trust-related objects to LDAP
3. Sets up SID generation for AD users
4. Enables the CLDAP plugin

**⏱️ Takes about 2-3 minutes.**

---

## Part C: Establish the Trust

### 🎓 What We're Doing

Actually creating the trust relationship between AD and IDM.

### Step 4.7: Create the Trust

```bash
ipa trust-add --type=ad ad.demo.local --admin Administrator --password
```

**When prompted:**
- Enter AD Administrator password: `RedHat@123456!`

> **Note:** If `Administrator` doesn't work, try `azureuser` (the Azure VM admin account).

**Expected output:**
```
Added Active Directory trust for realm "ad.demo.local"
  Realm name: ad.demo.local
  Domain NetBIOS name: AD
  Domain Security Identifier: S-1-5-21-xxxxxxxxxx
  Trust direction: Trusting forest
  Trust type: Active Directory domain
```

### Understanding the Output

| Field | Meaning |
|-------|---------|
| Realm name | The AD domain we're trusting |
| NetBIOS name | Short name for AD domain |
| Security Identifier | AD's unique SID |
| Trust direction | IDM trusts AD (AD users can access IDM resources) |

---

## Part D: Verify the Trust

### Step 4.8: Check Trust Status

```bash
# Show trust details
ipa trust-show ad.demo.local

# List all trusts
ipa trust-find
```

### Step 4.9: Verify AD Users Are Visible

**This is the most important test!**

```bash
# Can IDM see AD users?
id john.l1@ad.demo.local
id jane.l2@ad.demo.local
id mike.l3@ad.demo.local
```

**Expected output:**
```
uid=1948001106(john.l1@ad.demo.local) gid=1948001106(john.l1@ad.demo.local) 
groups=1948001106(john.l1@ad.demo.local),1948000513(domain users@ad.demo.local),
1948001103(linux_l1_team@ad.demo.local)
```

**What this shows:**
- ✅ UID assigned to AD user
- ✅ GID assigned
- ✅ AD group memberships visible (linux_l1_team!)

### Step 4.10: Verify AD Groups

```bash
# Check if AD groups are visible
getent group linux_l1_team@ad.demo.local
getent group linux_l2_team@ad.demo.local
getent group linux_l3_team@ad.demo.local
```

---

## 🔍 Troubleshooting

### Trust Creation Fails with "Authentication Error"

**Symptom:** `CIFS server communication error: code "3221225581"`

**Cause:** Wrong AD admin credentials

**Fix:**
```bash
# Try with Azure admin account
ipa trust-add --type=ad ad.demo.local --admin azureuser --password

# Or reset AD Administrator password on AD server
# Then try again
```

### DNS Not Resolving

**Symptom:** `nslookup ad.demo.local` returns SERVFAIL

**Fix:**
```bash
# Check/disable DNSSEC
cat /etc/named/ipa-options-ext.conf
# Should have: dnssec-validation no;

# If not, add it and restart
echo 'dnssec-validation no;' >> /etc/named/ipa-options-ext.conf
restorecon -v /etc/named/ipa-options-ext.conf
systemctl restart named
```

### AD Users Not Visible

**Symptom:** `id john.l1@ad.demo.local` returns "no such user"

**Fix:**
```bash
# Clear SSSD cache
sss_cache -E
systemctl restart sssd

# Check trust status
ipa trust-show ad.demo.local

# Check DNS both ways
nslookup ad.demo.local localhost
nslookup idm.demo.local 10.0.1.4
```

---

## ✅ Phase 4 Checklist

| Item | How to Verify |
|------|---------------|
| AD DNS forwards to IDM | On AD: `nslookup idm.idm.demo.local` |
| IDM DNS forwards to AD | On IDM: `nslookup ad.demo.local` |
| Trust components installed | `rpm -qa \| grep samba` |
| Trust established | `ipa trust-show ad.demo.local` |
| AD users visible | `id john.l1@ad.demo.local` |
| AD groups visible | `getent group linux_l1_team@ad.demo.local` |

---

## 🎓 What You Learned

After completing this phase, you now understand:

1. **What trust is**: An agreement to recognize each other's users
2. **Why DNS is critical**: Both sides must find each other
3. **Kerberos cross-realm**: How tickets work across domains
4. **SMB/CIFS role**: Windows protocol for trust negotiation
5. **Trust direction**: AD users can access IDM resources (not vice versa)

---

## Architecture After Phase 4

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure VNet                                │
│                                                                  │
│   ┌──────────────────┐              ┌──────────────────┐        │
│   │  Windows AD DC   │    Trust     │  IDM Primary     │        │
│   │  ad.demo.local   │◄────────────►│  idm.demo.local  │        │
│   │  10.0.1.4        │              │  10.0.1.5        │        │
│   │                  │              │                  │        │
│   │  Users:          │              │  Can see:        │        │
│   │  - john.l1       │─────────────►│  - john.l1@ad    │        │
│   │  - jane.l2       │              │  - jane.l2@ad    │        │
│   │  - mike.l3       │              │  - mike.l3@ad    │        │
│   │                  │              │                  │        │
│   └──────────────────┘              └──────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

Proceed to **Phase 5: IDM Replica Setup** for high availability, or skip to **Phase 6: Client Enrollment** to test AD user logins!

