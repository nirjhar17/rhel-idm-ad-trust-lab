# Phase 6: RHEL Client Enrollment

## 🎯 What Are We Doing in This Phase?

We're enrolling a **Linux server as an IDM client**. After this, the Linux server will:
- ✅ Authenticate users against IDM (including AD users!)
- ✅ Apply sudo rules from IDM
- ✅ Apply HBAC rules from IDM
- ✅ AD users can SSH with their AD passwords

**Simple Analogy:**
```
Before Enrollment:                   After Enrollment:
┌─────────────────┐                 ┌─────────────────┐
│  RHEL Server    │                 │  RHEL Server    │
│                 │                 │                 │
│  Only local     │                 │  IDM users ✓    │
│  users can      │      ───►       │  AD users ✓     │
│  login          │                 │  Sudo rules ✓   │
│                 │                 │  HBAC rules ✓   │
└─────────────────┘                 └─────────────────┘
```

---

## 📚 Key Concepts to Understand

### What is SSSD?

**SSSD** (System Security Services Daemon) is the "translator" between Linux and IDM.

```
┌─────────────────────────────────────────────────────────────────┐
│                    How SSSD Works                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User tries to login: ssh john.l1@ad.demo.local@server         │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                        SSSD                               │  │
│   │                                                           │  │
│   │   1. "Who is john.l1@ad.demo.local?"                     │  │
│   │      → Asks IDM via LDAP                                 │  │
│   │      → IDM asks AD (via trust)                           │  │
│   │      → Returns: UID, GID, groups                         │  │
│   │                                                           │  │
│   │   2. "Is the password correct?"                          │  │
│   │      → Asks IDM via Kerberos                             │  │
│   │      → IDM validates with AD                             │  │
│   │      → Returns: Yes/No                                   │  │
│   │                                                           │  │
│   │   3. "Can this user access this host?" (HBAC)            │  │
│   │      → Asks IDM                                          │  │
│   │      → Returns: Yes/No based on HBAC rules               │  │
│   │                                                           │  │
│   │   4. "What sudo commands can they run?"                  │  │
│   │      → Asks IDM                                          │  │
│   │      → Returns: List of allowed commands                 │  │
│   │                                                           │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Result: User logged in with correct permissions!              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why Enroll as IPA Client?

When you run `ipa-client-install`, it:

| Action | What It Does |
|--------|--------------|
| Configures SSSD | Sets up user/group lookups via IDM |
| Configures Kerberos | Sets up authentication via IDM |
| Configures PAM | Integrates with Linux login system |
| Configures SSH | Enables Kerberos-based SSH |
| Configures sudo | Gets sudo rules from IDM |
| Registers host | Adds this server to IDM's inventory |

### User Login Format

After enrollment, AD users can login using:

```
Full format (recommended):
ssh john.l1@ad.demo.local@server-ip

This means:
- User: john.l1@ad.demo.local (AD user)
- Server: server-ip
```

---

## 🔧 Prerequisites

Before starting:
- ✅ Phases 1-5 completed (or at least 1-4)
- ✅ AD Trust working
- ✅ RHEL Client VM running (10.0.1.6)

---

## Step 6.1: Connect to RHEL Client

```bash
ssh azureuser@<RHEL_CLIENT_PUBLIC_IP>
```
Password: `RedHat@123456!`

Become root:
```bash
sudo -i
```

---

## Step 6.2: Update System

**What:** Ensure system is up-to-date.

> **Note:** Azure RHEL VMs use RHUI - no subscription registration needed.

```bash
dnf update -y
```

---

## Step 6.3: Set Hostname

**What:** Give the client a proper FQDN.

**Why:** IDM tracks hosts by their FQDN - it must be unique and correct!

```bash
# Set hostname
hostnamectl set-hostname client.idm.demo.local

# Verify
hostname -f
```

**Expected:** `client.idm.demo.local`

---

## Step 6.4: Configure /etc/hosts

**What:** Add local name resolution.

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

## Step 6.5: Configure DNS to Use IDM

**What:** Point the client to IDM servers for DNS.

**Why:** Client needs to find IDM servers for authentication!

```bash
# Point to BOTH IDM servers (HA!)
nmcli con mod "System eth0" ipv4.dns "10.0.1.5 10.0.1.7"
nmcli con mod "System eth0" ipv4.ignore-auto-dns yes
nmcli con up "System eth0"

# Verify
cat /etc/resolv.conf
```

**Test DNS:**
```bash
# Should resolve IDM servers
nslookup idm.idm.demo.local
nslookup idm2.idm.demo.local

# Should resolve AD (through IDM's forwarder)
nslookup ad.demo.local
```

---

## Step 6.6: Install IPA Client Package

**What:** Install the software needed to enroll as IDM client.

```bash
# Install IPA client
dnf install -y ipa-client

# Verify
rpm -qa | grep ipa-client
```

---

## Step 6.7: Enroll Client into IDM

**What:** This is the main command that joins the client to IDM.

**Why:** After this, IDM controls authentication and authorization for this server!

```bash
ipa-client-install \
    --domain=idm.demo.local \
    --realm=IDM.DEMO.LOCAL \
    --server=idm.idm.demo.local \
    --server=idm2.idm.demo.local \
    --principal=admin \
    --password='RedHat@123456!' \
    --mkhomedir \
    --no-ntp \
    --unattended
```

### Understanding the Options

| Option | What It Does |
|--------|--------------|
| `--domain` | IDM domain to join |
| `--realm` | Kerberos realm (always UPPERCASE) |
| `--server` | IDM servers to use (both for HA!) |
| `--principal` | IDM admin username |
| `--password` | IDM admin password |
| `--mkhomedir` | Auto-create home directories for AD users |
| `--no-ntp` | Don't configure NTP (Azure handles this) |
| `--unattended` | Don't ask questions |

**Expected output:**
```
Client hostname: client.idm.demo.local
Realm: IDM.DEMO.LOCAL
DNS Domain: idm.demo.local
IPA Server: idm.idm.demo.local

Enrolled in IPA realm IDM.DEMO.LOCAL
Created /etc/ipa/default.conf
Configured /etc/sssd/sssd.conf
...
Client configuration complete.
The ipa-client-install command was successful
```

---

## Step 6.8: Verify Enrollment

### Check SSSD is Running

```bash
systemctl status sssd
```

**Expected:** Active (running)

### Check Client is Registered in IDM

```bash
# Get Kerberos ticket
kinit admin
# Password: RedHat@123456!

# Show this host in IDM
ipa host-show client.idm.demo.local
```

### Check IDM Admin User

```bash
# Should resolve the IDM admin user
id admin
```

---

## Step 6.9: Test AD User Resolution

**This is the critical test!**

```bash
# Test AD users are visible
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
- ✅ AD user is recognized
- ✅ Has a Linux UID
- ✅ AD group memberships are visible

### More User Info

```bash
# Get detailed user info
getent passwd john.l1@ad.demo.local

# Check group memberships
id -Gn john.l1@ad.demo.local
```

---

## Step 6.10: Test SSH Login with AD User

### From Your Local Machine

```bash
ssh john.l1@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>
```

When prompted for password: `RedHat@123456!`

### After Login, Verify

```bash
# Who am I?
whoami
# Expected: john.l1@ad.demo.local

# Where's my home?
pwd
# Expected: /home/john.l1@ad.demo.local

# What groups am I in?
id
```

### Home Directory Created Automatically

```bash
# Check home directory was created
ls -la /home/
```

You should see: `/home/john.l1@ad.demo.local`

---

## Step 6.11: Add Login Banner (Optional but Nice for Demo)

**What:** Show a banner when users login.

**Why:** Shows professionalism and warns about monitoring.

```bash
# Create banner
cat > /etc/issue.net << 'EOF'
************************************************************
*                                                          *
*  INDUSIND BANK - AUTHORIZED ACCESS ONLY                  *
*                                                          *
*  This system is managed by Red Hat Identity Management   *
*  All actions are logged and monitored                    *
*                                                          *
************************************************************
EOF

# Enable banner in SSH
echo "Banner /etc/issue.net" >> /etc/ssh/sshd_config

# Restart SSH
systemctl restart sshd
```

---

## 🔍 Troubleshooting

### Cannot Resolve AD Users

**Symptom:** `id john.l1@ad.demo.local` returns "no such user"

**Fix:**
```bash
# Clear SSSD cache
sss_cache -E
systemctl restart sssd

# Wait a few seconds, try again
sleep 5
id john.l1@ad.demo.local
```

### SSH Login Fails

**Symptom:** "Permission denied" when SSHing as AD user

**Check 1:** Is user resolving?
```bash
id john.l1@ad.demo.local
```

**Check 2:** Check auth logs
```bash
tail -20 /var/log/secure
```

**Check 3:** Is HBAC blocking? (We'll set this up in Phase 7)
```bash
# On IDM server
ipa hbactest --user=john.l1@ad.demo.local --host=client.idm.demo.local --service=sshd
```

### Home Directory Not Created

**Symptom:** User logs in but home directory doesn't exist

**Fix:**
```bash
# Enable oddjobd
systemctl enable oddjobd
systemctl start oddjobd

# Or manually create
mkdir -p /home/john.l1@ad.demo.local
chown john.l1@ad.demo.local:john.l1@ad.demo.local /home/john.l1@ad.demo.local
```

---

## ✅ Phase 6 Checklist

| Item | How to Verify |
|------|---------------|
| Hostname correct | `hostname -f` → `client.idm.demo.local` |
| DNS resolves IDM | `nslookup idm.idm.demo.local` |
| SSSD running | `systemctl status sssd` |
| Host registered in IDM | `ipa host-show client.idm.demo.local` |
| AD users resolve | `id john.l1@ad.demo.local` |
| AD user can SSH | `ssh john.l1@ad.demo.local@<IP>` |
| Home directory created | `ls /home/john.l1@ad.demo.local` |

---

## 🎓 What You Learned

After completing this phase, you now understand:

1. **SSSD Role**: Translates between Linux and IDM
2. **IPA Client**: How enrollment works
3. **User Format**: `user@ad.domain@server`
4. **Automatic Features**: Home directories, Kerberos, sudo integration
5. **Why This Matters**: Centralized authentication for all Linux servers

---

## Architecture After Phase 6

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure VNet                                │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ Windows AD   │    │ IDM Primary  │    │ IDM Replica  │      │
│  │ 10.0.1.4     │◄──►│ 10.0.1.5     │◄──►│ 10.0.1.7     │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│         │                   │                   │               │
│         │                   │                   │               │
│         │            ┌──────┴───────────────────┘               │
│         │            │                                          │
│         │            ▼                                          │
│         │    ┌──────────────────────────────────────┐          │
│         │    │           RHEL Client                 │          │
│         │    │         10.0.1.6                      │          │
│         │    │                                       │          │
│         │    │  SSSD talks to IDM for:              │          │
│         └───►│  • User authentication                │          │
│              │  • Group membership                   │          │
│              │  • Sudo rules                         │          │
│              │  • HBAC rules                         │          │
│              │                                       │          │
│              │  AD users can now SSH here!          │          │
│              └──────────────────────────────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

Proceed to **Phase 7: RBAC Configuration** to set up sudo rules and HBAC rules for L1/L2/L3 access control!

