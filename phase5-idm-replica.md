# Phase 5: IDM Replica Setup (High Availability)

## 🎯 What Are We Doing in This Phase?

We're setting up a **second IDM server** that keeps an exact copy of all data from the first server. If the first server dies, the second one takes over automatically!

**Simple Analogy:**
```
Without Replica:                     With Replica:
┌─────────────┐                     ┌─────────────┐   ┌─────────────┐
│ IDM Server  │                     │ IDM Server 1│◄─►│ IDM Server 2│
│  (Single)   │                     │  (Primary)  │   │  (Replica)  │
└─────────────┘                     └─────────────┘   └─────────────┘
       │                                   │                 │
       ▼                                   ▼                 ▼
    Failure!                          If this fails...  This continues!
  Everything stops!                    
```

**Why This Matters for Banks:**
- ✅ **No Single Point of Failure** - Required for RBI compliance
- ✅ **Zero Downtime** - Authentication keeps working
- ✅ **Disaster Recovery** - Data replicated to both servers
- ✅ **Load Distribution** - 1500 servers can spread load

---

## 📚 Key Concepts to Understand

### Multi-Master Replication

Unlike traditional primary/replica where replica is read-only, IDM uses **multi-master replication**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Multi-Master Replication                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────┐              ┌──────────────────┐        │
│   │  IDM Server 1    │   Two-way    │  IDM Server 2    │        │
│   │  (MASTER)        │◄────────────►│  (MASTER)        │        │
│   │                  │  Replication │                  │        │
│   │  Can:            │              │  Can:            │        │
│   │  ✓ Add users     │              │  ✓ Add users     │        │
│   │  ✓ Create rules  │              │  ✓ Create rules  │        │
│   │  ✓ Authenticate  │              │  ✓ Authenticate  │        │
│   │  ✓ Everything!   │              │  ✓ Everything!   │        │
│   └──────────────────┘              └──────────────────┘        │
│                                                                  │
│   Both servers are EQUAL - changes on either sync to the other  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key Point:** Both servers are "masters" - you can make changes on either one!

### How Clients Use Both Servers

Clients automatically discover and use both IDM servers:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client Discovery                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   RHEL Client asks DNS: "Where are the LDAP servers?"           │
│                                                                  │
│   DNS returns SRV records:                                      │
│   _ldap._tcp.idm.demo.local:                                    │
│     → idm.idm.demo.local (10.0.1.5) - priority 1               │
│     → idm2.idm.demo.local (10.0.1.7) - priority 1              │
│                                                                  │
│   Client: "I'll try both! If one fails, use the other"         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What Gets Replicated?

Everything important syncs between servers:

| Data | Replicated? |
|------|-------------|
| Users & Groups | ✅ Yes |
| Sudo Rules | ✅ Yes |
| HBAC Rules | ✅ Yes |
| DNS Zones | ✅ Yes |
| Certificates | ✅ Yes |
| Host Records | ✅ Yes |
| Trust Settings | ✅ Yes |

---

## 🔧 Prerequisites

Before starting:
- ✅ Phases 1-4 completed
- ✅ IDM Primary Server fully working
- ✅ AD Trust established and working
- ✅ IDM Server 2 VM running (10.0.1.7)

---

## Step 5.1: Connect to IDM Server 2

```bash
ssh azureuser@<IDM_SERVER2_PUBLIC_IP>
```
Password: `RedHat@123456!`

Become root:
```bash
sudo -i
```

---

## Step 5.2: Update System

**What:** Make sure system is up-to-date.

> **Note:** Azure RHEL VMs use RHUI - no subscription-manager needed.

```bash
dnf update -y
```

---

## Step 5.3: Set Hostname

**What:** Give the replica server its proper name.

**Why:** Must be unique and resolvable by the primary server.

```bash
# Set hostname
hostnamectl set-hostname idm2.idm.demo.local

# Verify
hostname -f
```

**Expected:** `idm2.idm.demo.local`

---

## Step 5.4: Configure /etc/hosts

**What:** Add local name resolution for all servers.

```bash
cat >> /etc/hosts << 'EOF'

# IDM Lab Hosts
10.0.1.4    ad.ad.demo.local ad
10.0.1.5    idm.idm.demo.local idm
10.0.1.6    client.idm.demo.local client
10.0.1.7    idm2.idm.demo.local idm2
EOF

# Verify
cat /etc/hosts
```

---

## Step 5.5: Configure DNS to Point to Primary

**What:** The replica MUST be able to reach the primary during installation.

**Why:** Replica install downloads all data from the primary server.

```bash
# Point DNS to the PRIMARY IDM server
nmcli con mod "System eth0" ipv4.dns "10.0.1.5"
nmcli con mod "System eth0" ipv4.ignore-auto-dns yes
nmcli con up "System eth0"

# Verify
cat /etc/resolv.conf
```

**Expected:** `nameserver 10.0.1.5`

**Test it works:**
```bash
# Must resolve the primary
nslookup idm.idm.demo.local
# Expected: 10.0.1.5

# Must also resolve AD
nslookup ad.demo.local
# Expected: 10.0.1.4
```

---

## Step 5.6: Configure Time Synchronization

**What:** Keep clocks in sync (critical for Kerberos!).

```bash
# Install chrony
dnf install -y chrony

# Enable and start
systemctl enable chronyd
systemctl start chronyd

# Verify
chronyc sources
timedatectl
```

---

## Step 5.7: Install IPA Packages

**What:** Install the software needed for the replica.

```bash
# Install IDM server packages
dnf install -y ipa-server ipa-server-dns ipa-server-trust-ad

# Verify
rpm -qa | grep ipa
```

---

## Step 5.8: Configure Firewall

**What:** Open ports for IDM services.

```bash
# Start firewall
systemctl enable firewalld
systemctl start firewalld

# Add all required services
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

# Apply
firewall-cmd --reload
firewall-cmd --list-all
```

---

## Step 5.9: Add DNS Record on Primary

### ⚠️ Do This on IDM SERVER 1 (Primary), Not Server 2!

**What:** Tell Primary's DNS about the new replica.

**Why:** Replica install checks DNS - the record must exist!

```bash
# SSH to primary
ssh azureuser@<IDM_SERVER1_PUBLIC_IP>
sudo -i

# Get admin ticket
kinit admin
# Password: RedHat@123456!

# Add DNS record for the replica
ipa dnsrecord-add idm.demo.local idm2 --a-rec=10.0.1.7

# Verify
ipa dnsrecord-find idm.demo.local | grep idm2
dig idm2.idm.demo.local
```

---

## Step 5.10: Enroll Replica as IPA Client First

### ⚠️ Back on IDM Server 2!

**What:** Before becoming a replica, the server must first enroll as a client.

**Why:** This establishes trust and gets Kerberos configured.

```bash
# Enroll as client
ipa-client-install \
    --domain=idm.demo.local \
    --realm=IDM.DEMO.LOCAL \
    --server=idm.idm.demo.local \
    --principal=admin \
    --password='RedHat@123456!' \
    --mkhomedir \
    --no-ntp \
    --unattended
```

**Expected:** "Client configuration complete. The ipa-client-install command was successful"

**Verify:**
```bash
# Should work now!
kinit admin
klist
```

---

## Step 5.11: Promote to Replica

**What:** Transform this client into a full replica server.

**Why:** This copies all data from primary and sets up replication.

```bash
ipa-replica-install \
    --setup-dns \
    --forwarder=10.0.1.4 \
    --forwarder=168.63.129.16 \
    --setup-ca \
    --unattended
```

**⏱️ This takes 10-15 minutes. Be patient!**

### What Happens During Installation

```
1. Configuring directory server... (copies LDAP data)
2. Setting up Kerberos... (configures KDC)
3. Setting up DNS... (copies zones)
4. Setting up CA... (copies certificates)
5. Setting up replication agreements... (enables sync)
```

---

## Step 5.12: Install AD Trust Components

**What:** Enable AD trust support on the replica.

**Why:** So AD users work on this server too!

```bash
# Get admin ticket first
kinit admin

# Install trust components
ipa-adtrust-install --netbios-name=IDM2 -a 'RedHat@123456!'
```

When prompted:
- **Do you wish to continue?** → `yes`
- **Enable trusted domains support?** → `yes`

---

## Step 5.13: Fix DNSSEC (If Needed)

**What:** Disable DNSSEC validation for AD queries.

```bash
# Check if dnssec-validation is "yes"
cat /etc/named/ipa-options-ext.conf | grep dnssec

# If it shows "yes", change it to "no"
sed -i 's/dnssec-validation yes;/dnssec-validation no;/' /etc/named/ipa-options-ext.conf

# Restart DNS
systemctl restart named

# Test AD resolution
nslookup ad.demo.local localhost
```

---

## Step 5.14: Verify Replica Installation

### Check All Services Running

```bash
ipactl status
```

**Expected (ALL RUNNING):**
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

### Check Replication

```bash
# List replica servers
ipa-replica-manage list

# Show topology
ipa topologysegment-find "domain"
```

**Expected:**
```
idm.idm.demo.local: master
idm2.idm.demo.local: master
```

### Verify Data Replicated

```bash
# IDM users should be visible
ipa user-find

# AD trust should be visible
ipa trust-find

# AD users should resolve
id john.l1@ad.demo.local
id mike.l3@ad.demo.local
```

---

## Step 5.15: Test High Availability

### Test: Stop Primary, Verify Replica Works

**On Primary (IDM Server 1):**
```bash
ipactl stop
```

**On Replica (IDM Server 2):**
```bash
# These should STILL WORK!
kinit admin
ipa user-find
id john.l1@ad.demo.local
```

**If it works:** High availability is working! 🎉

**Restart Primary:**
```bash
# On Primary
ipactl start
ipactl status
```

---

## 🔍 Troubleshooting

### Replica Install Fails: "Cannot resolve primary server"

**Fix:**
```bash
# Check DNS points to primary
cat /etc/resolv.conf
# Should show: nameserver 10.0.1.5

# Fix if needed
nmcli con mod "System eth0" ipv4.dns "10.0.1.5"
nmcli con up "System eth0"
```

### AD Users Not Resolving on Replica

**Fix:**
```bash
# Check DNSSEC
cat /etc/named/ipa-options-ext.conf | grep dnssec
# Should be: dnssec-validation no;

# Clear SSSD cache
sss_cache -E
systemctl restart sssd
```

### Replication Not Working

**Fix:**
```bash
# Check replication status
ipa-replica-manage list-ruv

# Force sync
ipa-replica-manage force-sync --from=idm.idm.demo.local
```

---

## ✅ Phase 5 Checklist

| Item | How to Verify |
|------|---------------|
| Hostname correct | `hostname -f` → `idm2.idm.demo.local` |
| DNS resolves primary | `nslookup idm.idm.demo.local` |
| All services running | `ipactl status` |
| Both servers show as master | `ipa-replica-manage list` |
| Topology segment exists | `ipa topologysegment-find "domain"` |
| AD trust works | `id john.l1@ad.demo.local` |
| HA test passes | Stop primary, verify replica works |

---

## 🎓 What You Learned

After completing this phase, you now understand:

1. **Multi-Master**: Both servers are equal, not primary/secondary
2. **Bi-directional Replication**: Changes sync both ways
3. **Client Discovery**: DNS SRV records help clients find servers
4. **High Availability**: No single point of failure
5. **Why Banks Need This**: RBI compliance requires no SPOF

---

## Architecture After Phase 5

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure VNet                                │
│                                                                  │
│  ┌──────────────┐           ┌──────────────┐                    │
│  │ Windows AD   │           │ IDM Primary  │                    │
│  │ ad.demo.local│◄─────────►│ 10.0.1.5     │                    │
│  │ 10.0.1.4     │   Trust   │              │                    │
│  └──────────────┘           └──────┬───────┘                    │
│                                    │                             │
│                             Bi-directional                       │
│                             Replication                          │
│                                    │                             │
│                             ┌──────┴───────┐                    │
│                             │ IDM Replica  │                    │
│                             │ 10.0.1.7     │                    │
│                             │              │                    │
│                             │ Same data!   │                    │
│                             └──────────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

Proceed to **Phase 6: RHEL Client Enrollment** to enroll Linux clients and test AD user logins!

