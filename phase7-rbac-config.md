# Phase 7: RBAC Configuration (Sudo & HBAC Rules)

## 🎯 What Are We Doing in This Phase?

This is the **most important phase for RBI compliance**! We're setting up:

1. **Sudo Rules**: Control what commands users can run with elevated privileges
2. **HBAC Rules**: Control which users can access which servers

**Business Value:**
- ✅ L1 operators can only VIEW (cat, tail, less) - can't break anything
- ✅ L2 admins can restart services but can't get full root
- ✅ L3 super admins have full access for emergencies
- ✅ All actions logged with individual accountability

---

## 📚 Key Concepts to Understand

### What is Sudo?

**Sudo** = "Super User DO" - lets you run commands as another user (usually root).

```
┌─────────────────────────────────────────────────────────────────┐
│                     Without Sudo Rules                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   john.l1 runs: sudo rm -rf /                                   │
│                                                                  │
│   Result: 💥 DISASTER! Server destroyed!                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      With Sudo Rules                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   john.l1 runs: sudo rm -rf /                                   │
│                                                                  │
│   Result: ❌ "Sorry, user john.l1 is not allowed to execute     │
│              '/bin/rm' as root"                                 │
│                                                                  │
│   john.l1 runs: sudo cat /var/log/messages                      │
│                                                                  │
│   Result: ✅ Works! (cat is in allowed commands)                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What is HBAC?

**HBAC** = Host-Based Access Control - controls WHO can access WHICH servers.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Without HBAC                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   L1 operator tries: ssh production-database-server             │
│                                                                  │
│   Result: ✅ Logged in! (No restrictions)                       │
│                                                                  │
│   Risk: L1 might accidentally break production!                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      With HBAC                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   L1 operator tries: ssh production-database-server             │
│                                                                  │
│   Result: ❌ "Access denied - HBAC rule violation"              │
│                                                                  │
│   L1 operator tries: ssh dev-test-server                        │
│                                                                  │
│   Result: ✅ Logged in! (Non-prod allowed for L1)               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Group Chain (Important Concept!)

AD users must be linked to IDM rules through a **group chain**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    The Group Chain                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   AD User                                                        │
│   (john.l1@ad.demo.local)                                       │
│           │                                                      │
│           │ is member of                                        │
│           ▼                                                      │
│   AD Group                                                       │
│   (Linux_L1_Team@ad.demo.local)                                 │
│           │                                                      │
│           │ mapped to                                           │
│           ▼                                                      │
│   IDM External Group                    ◄── Links AD to IDM     │
│   (ad_linux_l1_team)                                            │
│           │                                                      │
│           │ is member of                                        │
│           ▼                                                      │
│   IDM POSIX Group                       ◄── Has GID number      │
│   (linux_l1_operators)                                          │
│           │                                                      │
│           │ assigned to                                         │
│           ▼                                                      │
│   Sudo Rules & HBAC Rules               ◄── Actual permissions  │
│   (sudo_l1, hbac_l1)                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Why so complex?**
- AD groups don't have Linux GID numbers
- External groups "translate" AD SIDs to IDM
- POSIX groups have GID numbers Linux understands
- Rules reference POSIX groups

### Run-As in Sudo (Important!)

When you run `sudo command`, you're running that command **as another user**.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   john.l1 (UID 1001) runs: sudo cat /var/log/messages           │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  WHO is running?      → john.l1 (for audit log)         │   │
│   │  WHAT command?        → cat /var/log/messages           │   │
│   │  RUN AS which user?   → root (UID 0)                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   The file /var/log/messages is owned by root.                  │
│   john.l1 can't read it normally.                               │
│   But sudo runs "cat" AS ROOT, so it can read the file.         │
│                                                                  │
│   AUDIT LOG: "john.l1 ran sudo cat /var/log/messages"          │
│   (Individual accountability preserved!)                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 RBAC Matrix We're Implementing

| Role | Users | Sudo Commands | Server Access |
|------|-------|---------------|---------------|
| **L1 Operators** | john.l1, sarah.l1 | View only: cat, tail, less, df, free, systemctl status | Non-prod only |
| **L2 Admins** | jane.l2 | L1 + restart services, edit /etc/*, dnf/yum | Prod + Non-prod |
| **L3 Super Admins** | mike.l3 | ALL commands (can `sudo su -`) | ALL servers |

---

## 🔧 Prerequisites

Before starting:
- ✅ Phases 1-6 completed
- ✅ AD Trust working (can resolve AD users)
- ✅ SSH to IDM Primary server
- ✅ Logged in as root with kinit admin done

---

## Part A: Create the Group Chain

### 🎓 What We're Doing

We need to create groups in IDM that link AD groups to sudo/HBAC rules.

### Step 7.1: Connect to IDM Primary

```bash
ssh azureuser@<IDM_SERVER1_PUBLIC_IP>
sudo -i
kinit admin
# Password: RedHat@123456!
```

### Step 7.2: Create External Groups

**What:** External groups are special IDM groups that can contain AD groups (by their SID).

**Why:** This is the "bridge" between AD and IDM.

```bash
# Create external group for L1 (links to AD's Linux_L1_Team)
ipa group-add ad_linux_l1_team --external --desc="External group for AD Linux_L1_Team"

# Create external group for L2 (links to AD's Linux_L2_Team)
ipa group-add ad_linux_l2_team --external --desc="External group for AD Linux_L2_Team"

# Create external group for L3 (links to AD's Linux_L3_Team)
ipa group-add ad_linux_l3_team --external --desc="External group for AD Linux_L3_Team"
```

### Step 7.3: Add AD Groups to External Groups

**What:** Now we connect the AD groups to the external groups.

**Why:** This tells IDM "users in this AD group belong to this external group."

```bash
# Link AD L1 group to external group
# (When prompted, just press Enter for each prompt)
ipa group-add-member ad_linux_l1_team --external "linux_l1_team@ad.demo.local"

# Link AD L2 group to external group
ipa group-add-member ad_linux_l2_team --external "linux_l2_team@ad.demo.local"

# Link AD L3 group to external group
ipa group-add-member ad_linux_l3_team --external "linux_l3_team@ad.demo.local"
```

> **Note:** When you see `[member user]:` prompt, just press Enter to skip.

### Step 7.4: Create POSIX Groups

**What:** POSIX groups are regular Linux groups with GID numbers.

**Why:** Sudo and HBAC rules reference these groups (not the external groups directly).

```bash
# Create POSIX group for L1 operators
ipa group-add linux_l1_operators --desc="L1 Linux Operators - View only access"

# Create POSIX group for L2 admins
ipa group-add linux_l2_admins --desc="L2 Linux Admins - Service restart access"

# Create POSIX group for L3 super admins
ipa group-add linux_l3_superadmins --desc="L3 Linux Super Admins - Full access"
```

### Step 7.5: Link External Groups to POSIX Groups

**What:** Make the external groups members of the POSIX groups.

**Why:** This completes the chain: AD User → AD Group → External Group → POSIX Group → Rules

```bash
# L1: External → POSIX
ipa group-add-member linux_l1_operators --groups=ad_linux_l1_team

# L2: External → POSIX
ipa group-add-member linux_l2_admins --groups=ad_linux_l2_team

# L3: External → POSIX
ipa group-add-member linux_l3_superadmins --groups=ad_linux_l3_team
```

### Step 7.6: Verify the Chain

```bash
# Check L1 chain
echo "=== L1 Chain ==="
ipa group-show linux_l1_operators
ipa group-show ad_linux_l1_team --all

# Verify john.l1 resolves through the chain
id john.l1@ad.demo.local
```

**What to look for:**
- `linux_l1_operators` should show `Member groups: ad_linux_l1_team`
- `ad_linux_l1_team` should show `External member: linux_l1_team@ad.demo.local`

---

## Part B: Create Sudo Rules

### 🎓 What We're Doing

Creating rules that define what commands each role can run.

### Understanding Sudo Rule Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    Sudo Rule Components                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   sudo_l1_operators:                                            │
│   │                                                              │
│   ├── WHO can use this rule?                                    │
│   │   └── Group: linux_l1_operators                             │
│   │                                                              │
│   ├── On WHICH hosts?                                           │
│   │   └── Host category: all (any enrolled host)                │
│   │                                                              │
│   ├── WHAT commands can they run?                               │
│   │   └── Command group: l1_view_commands                       │
│   │       (cat, tail, less, df, free, etc.)                     │
│   │                                                              │
│   └── Run AS which user?                                        │
│       └── root                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 7.7: Create L1 Sudo Commands

**What:** Define the specific commands L1 can run.

```bash
# Add view-only commands
ipa sudocmd-add "/usr/bin/cat"
ipa sudocmd-add "/usr/bin/less"
ipa sudocmd-add "/usr/bin/tail"
ipa sudocmd-add "/usr/bin/head"
ipa sudocmd-add "/usr/bin/grep"
ipa sudocmd-add "/usr/bin/df"
ipa sudocmd-add "/usr/bin/free"
ipa sudocmd-add "/usr/bin/top"
ipa sudocmd-add "/usr/bin/ps"
ipa sudocmd-add "/usr/bin/uptime"
ipa sudocmd-add "/usr/bin/journalctl"
ipa sudocmd-add "/usr/bin/systemctl status *"

# Group them together
ipa sudocmdgroup-add l1_view_commands --desc="L1 View-only commands"

# Add commands to the group
ipa sudocmdgroup-add-member l1_view_commands \
    --sudocmds="/usr/bin/cat" \
    --sudocmds="/usr/bin/less" \
    --sudocmds="/usr/bin/tail" \
    --sudocmds="/usr/bin/head" \
    --sudocmds="/usr/bin/grep" \
    --sudocmds="/usr/bin/df" \
    --sudocmds="/usr/bin/free" \
    --sudocmds="/usr/bin/top" \
    --sudocmds="/usr/bin/ps" \
    --sudocmds="/usr/bin/uptime" \
    --sudocmds="/usr/bin/journalctl" \
    --sudocmds="/usr/bin/systemctl status *"
```

### Step 7.8: Create L2 Sudo Commands

**What:** L2 gets L1 commands PLUS service management commands.

```bash
# Add L2-specific commands
ipa sudocmd-add "/usr/bin/systemctl restart *"
ipa sudocmd-add "/usr/bin/systemctl start *"
ipa sudocmd-add "/usr/bin/systemctl stop *"
ipa sudocmd-add "/usr/bin/systemctl reload *"
ipa sudocmd-add "/usr/bin/vi /etc/*"
ipa sudocmd-add "/usr/bin/vim /etc/*"
ipa sudocmd-add "/usr/bin/dnf *"
ipa sudocmd-add "/usr/bin/yum *"

# Group them
ipa sudocmdgroup-add l2_admin_commands --desc="L2 Admin commands"

ipa sudocmdgroup-add-member l2_admin_commands \
    --sudocmds="/usr/bin/systemctl restart *" \
    --sudocmds="/usr/bin/systemctl start *" \
    --sudocmds="/usr/bin/systemctl stop *" \
    --sudocmds="/usr/bin/systemctl reload *" \
    --sudocmds="/usr/bin/vi /etc/*" \
    --sudocmds="/usr/bin/vim /etc/*" \
    --sudocmds="/usr/bin/dnf *" \
    --sudocmds="/usr/bin/yum *"
```

### Step 7.9: Create the Sudo Rules

**What:** Now we create the actual rules that tie groups to commands.

#### L1 Sudo Rule (View Only)

```bash
# Create the rule
ipa sudorule-add sudo_l1_operators \
    --desc="Sudo rule for L1 operators - view only" \
    --hostcat=all

# Add the L1 group to this rule
ipa sudorule-add-user sudo_l1_operators --groups=linux_l1_operators

# Add L1 command group
ipa sudorule-add-allow-command sudo_l1_operators --sudocmdgroups=l1_view_commands

# Allow running as root
ipa sudorule-add-runasuser sudo_l1_operators --users=root

# Verify
ipa sudorule-show sudo_l1_operators --all
```

#### L2 Sudo Rule (Service Management)

```bash
# Create the rule
ipa sudorule-add sudo_l2_admins \
    --desc="Sudo rule for L2 admins - service management" \
    --hostcat=all

# Add the L2 group
ipa sudorule-add-user sudo_l2_admins --groups=linux_l2_admins

# Add BOTH L1 and L2 command groups (L2 can do what L1 can)
ipa sudorule-add-allow-command sudo_l2_admins --sudocmdgroups=l1_view_commands
ipa sudorule-add-allow-command sudo_l2_admins --sudocmdgroups=l2_admin_commands

# Allow running as root
ipa sudorule-add-runasuser sudo_l2_admins --users=root

# Verify
ipa sudorule-show sudo_l2_admins --all
```

#### L3 Sudo Rule (Full Access)

```bash
# Create the rule with ALL commands
ipa sudorule-add sudo_l3_superadmins \
    --desc="Sudo rule for L3 super admins - full access" \
    --hostcat=all \
    --cmdcat=all

# Add the L3 group
ipa sudorule-add-user sudo_l3_superadmins --groups=linux_l3_superadmins

# Allow running as ANY user (can sudo su - oracle, etc.)
ipa sudorule-mod sudo_l3_superadmins --runasusercat=all
ipa sudorule-mod sudo_l3_superadmins --runasgroupcat=all

# Verify
ipa sudorule-show sudo_l3_superadmins --all
```

> **⚠️ Important:** Don't add specific runAs users AND set runasusercat=all. It will fail!

---

## Part C: Create HBAC Rules

### 🎓 What We're Doing

Creating rules that control which servers each role can access.

### Step 7.10: Create Host Groups

**What:** Group servers together (prod vs non-prod).

**Why:** Easier to manage rules - "L1 can access non-prod" vs listing 500 servers.

```bash
# Create non-prod host group
ipa hostgroup-add nonprod_servers --desc="Non-Production Servers"

# Create prod host group
ipa hostgroup-add prod_servers --desc="Production Servers"

# Add our client to non-prod (for testing)
ipa hostgroup-add-member nonprod_servers --hosts=client.idm.demo.local
```

### Step 7.11: Disable Default "allow_all" Rule

**What:** IDM has a default rule that allows EVERYONE access to EVERYTHING.

**Why:** We need to disable it so our rules actually work.

```bash
# Disable the default rule
ipa hbacrule-disable allow_all

# Verify
ipa hbacrule-show allow_all
# Should show: Enabled: False
```

### Step 7.12: Create HBAC Rules

#### L1 HBAC (Non-Prod Only)

```bash
# Create L1 rule
ipa hbacrule-add hbac_l1_operators \
    --desc="HBAC rule for L1 - non-prod only" \
    --servicecat=all

# Add L1 group
ipa hbacrule-add-user hbac_l1_operators --groups=linux_l1_operators

# Add ONLY non-prod servers
ipa hbacrule-add-host hbac_l1_operators --hostgroups=nonprod_servers

# Verify
ipa hbacrule-show hbac_l1_operators --all
```

#### L2 HBAC (Prod + Non-Prod)

```bash
# Create L2 rule
ipa hbacrule-add hbac_l2_admins \
    --desc="HBAC rule for L2 - prod and non-prod" \
    --servicecat=all

# Add L2 group
ipa hbacrule-add-user hbac_l2_admins --groups=linux_l2_admins

# Add BOTH host groups
ipa hbacrule-add-host hbac_l2_admins --hostgroups=nonprod_servers
ipa hbacrule-add-host hbac_l2_admins --hostgroups=prod_servers

# Verify
ipa hbacrule-show hbac_l2_admins --all
```

#### L3 HBAC (All Servers)

```bash
# Create L3 rule with ALL hosts
ipa hbacrule-add hbac_l3_superadmins \
    --desc="HBAC rule for L3 - all servers" \
    --servicecat=all \
    --hostcat=all

# Add L3 group
ipa hbacrule-add-user hbac_l3_superadmins --groups=linux_l3_superadmins

# Verify
ipa hbacrule-show hbac_l3_superadmins --all
```

#### IDM Admin HBAC

```bash
# Allow IDM admin to access all hosts
ipa hbacrule-add hbac_idm_admins \
    --desc="HBAC rule for IDM admins" \
    --servicecat=all \
    --hostcat=all

# Add admin user
ipa hbacrule-add-user hbac_idm_admins --users=admin

# Verify
ipa hbacrule-show hbac_idm_admins --all
```

---

## Part D: Test the Configuration

### Step 7.13: Test HBAC

```bash
# Test L1 access
ipa hbactest --user=john.l1@ad.demo.local --host=client.idm.demo.local --service=sshd

# Test L2 access
ipa hbactest --user=jane.l2@ad.demo.local --host=client.idm.demo.local --service=sshd

# Test L3 access
ipa hbactest --user=mike.l3@ad.demo.local --host=client.idm.demo.local --service=sshd
```

**Expected:** All should show `Access granted: True`

### Step 7.14: Refresh Client Cache

On the RHEL client, refresh SSSD cache to get new rules:

```bash
ssh azureuser@<RHEL_CLIENT_PUBLIC_IP>
sudo -i

# Clear cache
sss_cache -E
systemctl restart sssd

# Test sudo rules
sudo -l -U john.l1@ad.demo.local
```

---

## 🔍 Troubleshooting

### HBAC Returns False

**Check 1:** Is the group chain complete?
```bash
ipa group-show linux_l1_operators --all
# Should show: Member groups: ad_linux_l1_team

ipa group-show ad_linux_l1_team --all
# Should show: External member: linux_l1_team@ad.demo.local
```

**Check 2:** Is allow_all disabled?
```bash
ipa hbacrule-show allow_all
# Should show: Enabled: False
```

**Check 3:** Is the host in a hostgroup?
```bash
ipa hostgroup-show nonprod_servers
# Should show: Member hosts: client.idm.demo.local
```

### Sudo Rules Not Working

**Check 1:** Clear client cache
```bash
sss_cache -E
systemctl restart sssd
```

**Check 2:** Check SSSD is configured for sudo
```bash
grep sudo /etc/sssd/sssd.conf
# Should see: services = nss, pam, ssh, sudo
```

**Check 3:** Check sudo rules are received
```bash
sudo -l -U john.l1@ad.demo.local
```

---

## ✅ Phase 7 Checklist

| Item | How to Verify |
|------|---------------|
| External groups created | `ipa group-find --external` |
| External groups have AD members | `ipa group-show ad_linux_l1_team --all` |
| POSIX groups created | `ipa group-find linux_l` |
| POSIX groups have external groups | `ipa group-show linux_l1_operators` |
| Sudo rules created | `ipa sudorule-find` |
| HBAC rules created | `ipa hbacrule-find` |
| allow_all disabled | `ipa hbacrule-show allow_all` |
| HBAC test passes | `ipa hbactest --user=john.l1@ad.demo.local ...` |

---

## 🎓 What You Learned

After completing this phase, you now understand:

1. **Group Chain**: AD Group → External Group → POSIX Group → Rules
2. **Sudo Rules**: Control what commands users can run
3. **HBAC Rules**: Control which servers users can access
4. **Run-As**: Commands run as root but are logged as the actual user
5. **Why this matters for RBI**: Individual accountability + role-based access

---

## 🏁 Next Steps

Proceed to **Phase 8: Validation and Demo** to test everything end-to-end!

