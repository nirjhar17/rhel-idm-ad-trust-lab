# IDM-AD Integration Demo Script
## Enterprise Compliance RBAC Demo

---

## 🔑 Quick Reference - IPs & Credentials

| Server | Public IP | Purpose |
|--------|-----------|---------|
| RHEL Client | `20.51.158.128` | Demo target (login here) |
| IDM Primary | `172.190.242.84` | IDM Server 1 |
| IDM Replica | `20.42.110.153` | IDM Server 2 (HA) |
| AD DC | `172.203.152.176` | Active Directory |

| Credential | Value |
|------------|-------|
| All Passwords | `RedHat@123456!` |
| IDM Web UI | `https://idm.idm.demo.local` |
| IDM Admin | `admin` / `RedHat@123456!` |

| User | Role | Access Level |
|------|------|--------------|
| `john.l1@ad.demo.local` | L1 Operator | View only |
| `jane.l2@ad.demo.local` | L2 Admin | Service management |
| `mike.l3@ad.demo.local` | L3 Super Admin | Full access |

---

# DEMO 1: Web UI - Rules & Trust (5 min)

## Open IDM Web UI

**URL:** `https://idm.idm.demo.local`  
**Login:** `admin` / `RedHat@123456!`

---

## 1.1 Show Sudo Rules

**Navigate:** Policy → Sudo → Sudo Rules

| Click On | What to Show | Say This |
|----------|--------------|----------|
| `sudo_l1_operators` | Command Groups: `l1_view_commands` | *"L1 can only run view commands - systemctl status, df, free"* |
| `sudo_l2_admins` | Command Groups: `l1_view_commands`, `l2_admin_commands` | *"L2 can do L1 stuff PLUS restart services"* |
| `sudo_l3_superadmins` | Command Category: `all` | *"L3 has full sudo - can do everything"* |

---

## 1.2 Show HBAC Rules (Host Access)

**Navigate:** Policy → Host Based Access Control → HBAC Rules

| Click On | What to Show | Say This |
|----------|--------------|----------|
| `hbac_l1_operators` | Host Groups: `nonprod_servers` | *"L1 can ONLY login to non-production servers"* |
| `hbac_l2_admins` | Host Groups: `nonprod_servers`, `prod_servers` | *"L2 can login to both prod and non-prod"* |
| `hbac_l3_superadmins` | Host Category: `all` | *"L3 can login to ALL servers"* |

---

## 1.3 Show Host Groups

**Navigate:** Identity → Groups → Host Groups

| Click On | What to Show | Say This |
|----------|--------------|----------|
| `nonprod_servers` | Member hosts: `client.idm.demo.local` | *"Add servers here for L1/L2 access"* |
| `prod_servers` | Member hosts | *"Add production servers here for L2 access"* |

**Key Message:** *"You just add servers to these groups - rules apply automatically. No per-server configuration!"*

---

## 1.4 Show AD Trust

**Navigate:** IPA Server → Trusts

| What to Show | Say This |
|--------------|----------|
| Trust: `ad.demo.local` | *"This is the trust with your Active Directory. AD users can now login to Linux servers with their existing AD credentials."* |

---

# DEMO 2: L1/L2/L3 User Access (10 min)

## 2.1 L1 User (john.l1) - View Only

```bash
ssh john.l1@ad.demo.local@20.51.158.128
```
**Password:** `RedHat@123456!`

### L1 CAN DO ✅
```bash
whoami                          # Shows: john.l1@ad.demo.local
sudo systemctl status sshd      # ✅ Works - can view status
sudo df -h                       # ✅ Works - can view disk
sudo free -m                     # ✅ Works - can view memory
```

### L1 CANNOT DO ❌
```bash
sudo systemctl restart sshd     # ❌ DENIED
sudo yum install -y vim         # ❌ DENIED
```

**Say:** *"L1 can monitor but cannot make changes. This is your L1 operations team - they can see status but can't break anything."*

```bash
exit
```

---

## 2.2 L2 User (jane.l2) - Service Management

```bash
ssh jane.l2@ad.demo.local@20.51.158.128
```
**Password:** `RedHat@123456!`

### L2 CAN DO ✅
```bash
whoami                          # Shows: jane.l2@ad.demo.local
sudo systemctl status sshd      # ✅ Works
sudo systemctl restart crond    # ✅ Works - CAN restart services!
```

### L2 CANNOT DO ❌
```bash
sudo yum install -y vim         # ❌ DENIED
sudo useradd testuser           # ❌ DENIED
```

**Say:** *"L2 can restart services but cannot install software or manage users. Perfect for your operations team who need to restart services during incidents."*

```bash
exit
```

---

## 2.3 L3 User (mike.l3) - Full Access

```bash
ssh mike.l3@ad.demo.local@20.51.158.128
```
**Password:** `RedHat@123456!`

### L3 CAN DO EVERYTHING ✅
```bash
whoami                          # Shows: mike.l3@ad.demo.local
sudo systemctl restart crond    # ✅ Works
sudo yum install -y vim         # ✅ Works
sudo useradd demouser           # ✅ Works
sudo userdel demouser           # Cleanup
sudo su -                       # ✅ Can become root
whoami                          # Shows: root
exit                            # Exit from root
```

**Say:** *"L3 has full access - but importantly, we know WHO became root. It's mike.l3, not a generic 'Unix admin'. Individual accountability!"*

```bash
exit
```

---

# DEMO 3: Audit Logs - Compliance Win! (5 min) ⭐

**This is the MOST IMPORTANT demo for compliance!**

```bash
ssh azureuser@20.51.158.128
```
**Password:** `RedHat@123456!`

```bash
sudo tail -100 /var/log/secure | grep -E "john.l1|jane.l2|mike.l3"
```

### Expected Output:
```
john.l1@ad.demo.local : COMMAND=/usr/bin/systemctl status sshd
john.l1@ad.demo.local : command not allowed ; COMMAND=/usr/bin/systemctl restart sshd
jane.l2@ad.demo.local : COMMAND=/usr/bin/systemctl restart crond
mike.l3@ad.demo.local : COMMAND=/usr/bin/su -
```

### What to Point Out:

| Log Entry | What It Shows |
|-----------|---------------|
| `john.l1 : COMMAND=systemctl status` | John (L1) checked status |
| `john.l1 : command not allowed` | John (L1) tried to restart - DENIED! |
| `jane.l2 : COMMAND=systemctl restart` | Jane (L2) restarted service |
| `mike.l3 : COMMAND=su -` | Mike (L3) became root |

**Say:** 
> *"Look at this audit trail:*
> - *We know john.l1 tried to restart sshd and was DENIED*
> - *We know jane.l2 restarted crond*  
> - *We know mike.l3 became root*
>
> *Compare this to your current state where logs just show 'Unix admin'. Now every action is traceable to an individual - exactly what auditors require!"*

```bash
exit
```

---

# DEMO 4: High Availability (5 min)

## 4.1 Verify Both IDM Servers Running

**Terminal 1 - Primary:**
```bash
ssh azureuser@172.190.242.84
sudo ipactl status
```
All services should show **RUNNING**

**Terminal 2 - Replica:**
```bash
ssh azureuser@20.42.110.153
sudo ipactl status
```
All services should show **RUNNING**

---

## 4.2 STOP Primary (Simulate Failure)

On **Primary (172.190.242.84)**:
```bash
sudo ipactl stop
```

**Say:** *"I'm now stopping the primary IDM server - simulating a complete server failure."*

---

## 4.3 Prove Users Can STILL Login!

**New Terminal:**
```bash
ssh john.l1@ad.demo.local@20.51.158.128
```
**Password:** `RedHat@123456!`

```bash
whoami                          # Works!
sudo systemctl status sshd      # Works!
exit
```

**Say:** *"Even though the primary IDM server is completely DOWN, users can still login and use sudo. The replica took over automatically - ZERO user impact!"*

---

## 4.4 Restart Primary (Recovery)

On **Primary (172.190.242.84)**:
```bash
sudo ipactl start
sudo ipactl status
```

**Say:** *"Primary is back. For your 1,500 servers, this means no authentication downtime even during maintenance or failures."*

---

# CLOSING SUMMARY

> *"To summarize what you just saw:*
>
> 1. **Individual Login** - AD users login with existing credentials, no shared 'Unix admin' account
>
> 2. **L1/L2/L3 RBAC** - Different privilege levels enforced centrally from IDM
>
> 3. **Audit Trail** - Every action traceable to individual user in /var/log/secure
>
> 4. **Centralized Management** - Rules defined once, apply to all 1,500 servers
>
> 5. **High Availability** - No single point of failure
>
> *This directly addresses your audit requirements for:*
> - *Individual traceability*
> - *Role-based access control*  
> - *Audit logs showing who did what*
> - *Elimination of shared accounts"*

---

# TROUBLESHOOTING

| Issue | Quick Fix |
|-------|-----------|
| User not found | On client: `sudo sss_cache -E && sudo systemctl restart sssd` |
| Sudo not working | On client: `sudo sss_cache -E && sudo systemctl restart sssd` |
| Can't login | Check IDM services: `sudo ipactl status` |
| Web UI not loading | Accept certificate warning in browser |
| Password rejected | Password is `RedHat@123456!` (with @) |

---

**Good Luck with Your Demo! 🎯**
