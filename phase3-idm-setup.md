# Phase 3: IDM Primary Server Setup

## 🎯 What Are We Doing in This Phase?

We're installing **Red Hat Identity Management (IDM)** - think of it as the "Linux version of Active Directory". 

**Simple Analogy:**
- Active Directory = Central control for Windows computers
- IDM = Central control for Linux computers

After this phase, you'll have a server that can:
- ✅ Manage Linux users centrally
- ✅ Store sudo rules (who can run what commands)
- ✅ Store HBAC rules (who can access which servers)
- ✅ Provide DNS for your Linux infrastructure
- ✅ Later: Trust Active Directory to share users

---

## 📚 Key Concepts to Understand

### What is IDM/FreeIPA?

```
┌─────────────────────────────────────────────────────────────────┐
│                    IDM Server Components                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   389 DS     │  │   MIT        │  │   BIND       │          │
│  │  (LDAP)      │  │  Kerberos    │  │   (DNS)      │          │
│  │              │  │              │  │              │          │
│  │ Stores:      │  │ Provides:    │  │ Provides:    │          │
│  │ - Users      │  │ - Tickets    │  │ - Name       │          │
│  │ - Groups     │  │ - Auth       │  │   resolution │          │
│  │ - Sudo rules │  │ - SSO        │  │              │          │
│  │ - HBAC rules │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │  Dogtag CA   │  │   Web UI     │                            │
│  │              │  │              │                            │
│  │ Issues:      │  │ Provides:    │                            │
│  │ - Certs      │  │ - GUI        │                            │
│  │ - SSL/TLS    │  │ - Admin      │                            │
│  └──────────────┘  └──────────────┘                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**In Simple Terms:**
- **LDAP (389 DS)**: A database that stores all your users, groups, and rules
- **Kerberos**: The authentication system - proves "you are who you say you are"
- **DNS (BIND)**: Converts names to IP addresses (like a phone book)
- **Certificate Authority**: Issues SSL certificates for secure communication
- **Web UI**: A website to manage everything visually

### What is Kerberos? (Simple Explanation)

Kerberos is like a **movie ticket system**:

1. You show ID at the box office (authenticate with password)
2. You get a ticket (Kerberos ticket)
3. You show the ticket to enter any movie (access any service)
4. No need to show ID again! (Single Sign-On)

```
┌─────────────┐     1. Password      ┌─────────────┐
│    User     │ ─────────────────────►│   IDM/KDC   │
│  (john.l1)  │                       │  (Ticket    │
│             │◄───────────────────── │   Office)   │
└─────────────┘     2. Ticket         └─────────────┘
       │
       │ 3. Show ticket
       ▼
┌─────────────┐
│   Server    │  "Ticket valid! 
│  (SSH/App)  │   Welcome john.l1!"
└─────────────┘
```

### What is a Realm?

A **realm** is like a kingdom in Kerberos. All users and services in the same realm trust each other.

- Our IDM realm: `IDM.DEMO.LOCAL` (always UPPERCASE)
- Our AD realm: `AD.DEMO.LOCAL`
- When we create a "trust", we connect these two kingdoms!

---

## 🔧 Prerequisites

Before starting, make sure:
- ✅ Phase 1 completed (Azure VMs created)
- ✅ Phase 2 completed (AD is working)
- ✅ IDM Server 1 VM is running
- ✅ You can SSH to the IDM server

---

## Step 3.1: Connect to IDM Server

**What we're doing:** SSH into the server where we'll install IDM.

```bash
ssh azureuser@<IDM_SERVER1_PUBLIC_IP>
```
Password: `RedHat@123456!`

**Become root** (administrator):
```bash
sudo -i
```

---

## Step 3.2: Update the System

**What we're doing:** Making sure all software is up-to-date before installing IDM.

**Why:** Old software can have bugs or security issues. Always update first!

> **Note:** Azure RHEL VMs use RHUI (Red Hat Update Infrastructure), so no subscription-manager registration needed.

```bash
dnf update -y
```

This might take 2-5 minutes. Wait for it to complete.

---

## Step 3.3: Set the Hostname

**What we're doing:** Giving the server a proper name (FQDN - Fully Qualified Domain Name).

**Why:** IDM and Kerberos are very strict about hostnames. The hostname MUST match what DNS returns!

```bash
# Set the hostname
hostnamectl set-hostname idm.idm.demo.local

# Verify it's set correctly
hostname -f
```

**Expected output:** `idm.idm.demo.local`

### Understanding the Hostname

```
idm.idm.demo.local
 │   │    │    │
 │   │    │    └── Top-level domain
 │   │    └─────── Second-level domain  
 │   └──────────── IDM domain (idm.demo.local)
 └──────────────── Server name
```

---

## Step 3.4: Configure /etc/hosts

**What we're doing:** Adding entries to the local "phone book" so the server knows how to find other servers.

**Why:** Before DNS is set up, servers use `/etc/hosts` to find each other.

```bash
cat >> /etc/hosts << 'EOF'

# IDM Lab Hosts
10.0.1.4    ad.ad.demo.local ad
10.0.1.5    idm.idm.demo.local idm
10.0.1.6    client.idm.demo.local client
10.0.1.7    idm2.idm.demo.local idm2
EOF
```

**Verify:**
```bash
cat /etc/hosts
```

You should see all four servers listed.

---

## Step 3.5: Configure DNS

**What we're doing:** Telling this server to use specific DNS servers.

**Why:** IDM will become a DNS server itself, but during installation, it needs to find the AD server.

```bash
# Set DNS to Azure DNS (for now) and AD server
nmcli con mod "System eth0" ipv4.dns "10.0.1.4 168.63.129.16"
nmcli con mod "System eth0" ipv4.ignore-auto-dns yes
nmcli con up "System eth0"
```

**Verify:**
```bash
cat /etc/resolv.conf
```

**Test DNS is working:**
```bash
# Should resolve to 10.0.1.4
nslookup ad.demo.local
```

---

## Step 3.6: Configure Time Synchronization

**What we're doing:** Making sure the server's clock is accurate.

**Why this is CRITICAL:** Kerberos tickets have timestamps. If clocks are off by more than 5 minutes, authentication FAILS!

```bash
# Install chrony (time sync service)
dnf install -y chrony

# Enable and start it
systemctl enable chronyd
systemctl start chronyd

# Verify time sync
chronyc sources
timedatectl
```

**Check that time is synchronized:**
```bash
timedatectl
```

Look for: `System clock synchronized: yes`

---

## Step 3.7: Install IDM Packages

**What we're doing:** Installing all the software needed for IDM.

**Why:** IDM is made up of many components (LDAP, Kerberos, DNS, CA, etc.) that all need to be installed.

```bash
# Install IDM server with DNS and AD trust support
dnf install -y ipa-server ipa-server-dns ipa-server-trust-ad

# This takes 2-5 minutes
```

**What gets installed:**
- `ipa-server`: Core IDM/FreeIPA server
- `ipa-server-dns`: DNS server integration
- `ipa-server-trust-ad`: Active Directory trust support

**Verify packages installed:**
```bash
rpm -qa | grep ipa
```

---

## Step 3.8: Configure Firewall

**What we're doing:** Opening the network ports that IDM needs to communicate.

**Why:** Linux firewall blocks all ports by default. We need to open specific ports for IDM services.

```bash
# Start firewall if not running
systemctl enable firewalld
systemctl start firewalld

# Open ports for all IDM services
firewall-cmd --permanent --add-service=freeipa-ldap
firewall-cmd --permanent --add-service=freeipa-ldaps  
firewall-cmd --permanent --add-service=freeipa-replication
firewall-cmd --permanent --add-service=freeipa-trust
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-service=ntp
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=kerberos
firewall-cmd --permanent --add-service=kpasswd
firewall-cmd --permanent --add-service=ldap
firewall-cmd --permanent --add-service=ldaps

# Apply changes
firewall-cmd --reload

# Verify
firewall-cmd --list-all
```

### What Each Port Does

| Port | Service | Purpose |
|------|---------|---------|
| 53 | DNS | Name resolution |
| 80/443 | HTTP/HTTPS | Web UI |
| 88 | Kerberos | Authentication |
| 389/636 | LDAP/LDAPS | Directory queries |
| 464 | kpasswd | Password changes |

---

## Step 3.9: Install IDM Server (The Big Step!)

**What we're doing:** This is the main installation command that sets up ALL of IDM.

**Why:** This single command:
- Creates the LDAP database
- Sets up Kerberos
- Configures DNS
- Generates SSL certificates
- Creates the admin user
- Starts all services

### The Installation Command

```bash
ipa-server-install \
    --hostname=idm.idm.demo.local \
    --domain=idm.demo.local \
    --realm=IDM.DEMO.LOCAL \
    --ds-password='RedHat@123456!' \
    --admin-password='RedHat@123456!' \
    --setup-dns \
    --forwarder=10.0.1.4 \
    --forwarder=168.63.129.16 \
    --reverse-zone=1.0.10.in-addr.arpa. \
    --no-host-dns \
    --unattended
```

### Understanding Each Option

| Option | What It Means |
|--------|---------------|
| `--hostname` | This server's full name |
| `--domain` | The DNS domain IDM will manage |
| `--realm` | The Kerberos realm (always UPPERCASE) |
| `--ds-password` | Password for the LDAP directory |
| `--admin-password` | Password for the IDM admin user |
| `--setup-dns` | Also install DNS server |
| `--forwarder` | Where to forward unknown DNS queries |
| `--reverse-zone` | For reverse DNS lookups |
| `--no-host-dns` | Don't check hostname in DNS (we'll set it up) |
| `--unattended` | Don't ask questions, use the options provided |

**⏱️ This takes 10-15 minutes. Be patient!**

### What Happens During Installation

```
1. Creating Directory Server instance... (LDAP database)
2. Configuring Kerberos KDC... (Authentication server)
3. Configuring kadmin... (Kerberos admin server)
4. Configuring DNS... (Name resolution)
5. Configuring Certificate Server... (SSL certificates)
6. Configuring httpd... (Web interface)
7. Starting all services...
```

---

## Step 3.10: Verify Installation

**What we're doing:** Making sure IDM is running correctly.

### Check All Services

```bash
ipactl status
```

**Expected output (ALL should say RUNNING):**
```
Directory Service: RUNNING
krb5kdc Service: RUNNING
kadmin Service: RUNNING
named Service: RUNNING
httpd Service: RUNNING
ipa-custodia Service: RUNNING
pki-tomcatd Service: RUNNING
ipa-otpd Service: RUNNING
ipa-dnskeysyncd Service: RUNNING
```

### Get a Kerberos Ticket

```bash
# Log in as admin
kinit admin
# Enter password: RedHat@123456!

# Verify you have a ticket
klist
```

**Expected output:**
```
Ticket cache: KCM:0
Default principal: admin@IDM.DEMO.LOCAL

Valid starting       Expires              Service principal
01/06/2026 10:00:00  01/07/2026 10:00:00  krbtgt/IDM.DEMO.LOCAL@IDM.DEMO.LOCAL
```

### Test IDM Commands

```bash
# List users (should show admin)
ipa user-find

# Show IDM configuration
ipa config-show
```

---

## Step 3.11: Access Web UI (Optional)

**What we're doing:** Testing the web interface.

Open a browser and go to:
```
https://<IDM_SERVER_PUBLIC_IP>/ipa/ui/
```

Login with:
- Username: `admin`
- Password: `RedHat@123456!`

> **Note:** You'll get a certificate warning - that's OK for a lab. Click "Advanced" → "Proceed".

---

## Step 3.12: Configure DNS to Point to Itself

**What we're doing:** Now that IDM is installed, make it use its own DNS.

**Why:** IDM is now the DNS server, so it should use itself for lookups.

```bash
# Point DNS to localhost and AD
nmcli con mod "System eth0" ipv4.dns "127.0.0.1 10.0.1.4"
nmcli con up "System eth0"

# Verify
cat /etc/resolv.conf
```

---

## 🔍 Troubleshooting

### Installation Fails with DNS Error

**Symptom:** `DNS check failed: SERVFAIL`

**Fix:** Disable DNSSEC validation:
```bash
# Edit DNS options
vi /etc/named/ipa-options-ext.conf

# Add or change to:
dnssec-validation no;

# Restart DNS
systemctl restart named
```

### Cannot Get Kerberos Ticket

**Symptom:** `kinit: Cannot contact any KDC`

**Fix:** Check if Kerberos is running:
```bash
systemctl status krb5kdc
systemctl start krb5kdc
```

### Time Sync Issues

**Symptom:** `Clock skew too great`

**Fix:** Sync time:
```bash
systemctl restart chronyd
chronyc makestep
```

---

## ✅ Phase 3 Checklist

| Item | Command to Verify |
|------|-------------------|
| Hostname correct | `hostname -f` → `idm.idm.demo.local` |
| All services running | `ipactl status` → All RUNNING |
| Can get Kerberos ticket | `kinit admin` → No errors |
| Can find admin user | `ipa user-find` → Shows admin |
| DNS working | `nslookup idm.idm.demo.local` → 10.0.1.5 |

---

## 🎓 What You Learned

After completing this phase, you now understand:

1. **What IDM is**: A combination of LDAP, Kerberos, DNS, and CA
2. **Why hostnames matter**: Kerberos is strict about names
3. **Why time sync is critical**: Kerberos tickets have timestamps
4. **What a realm is**: A Kerberos trust domain
5. **How to verify IDM**: Using `ipactl status` and `kinit`

---

## Next Steps

Proceed to **Phase 4: AD Trust Configuration** to connect IDM with Active Directory!

