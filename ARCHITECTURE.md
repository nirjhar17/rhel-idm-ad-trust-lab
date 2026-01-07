# Architecture Guide: IDM + AD Trust

## 🎯 What Problem Are We Solving?

**The Bank's Problem:**
```
Currently at Indusind Bank:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   1500 Linux Servers                                        │
│                                                              │
│   All 50 admins login as:  ──►  unixadmin (shared account)  │
│                                                              │
│   RBI Auditor asks: "Who deleted the database?"             │
│   Answer: "We don't know... could be any of 50 people" 😱   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**What We Built:**
```
After our solution:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   1500 Linux Servers                                        │
│                                                              │
│   Each admin logs in with their OWN AD account:            │
│   - john.l1@ad.demo.local                                   │
│   - jane.l2@ad.demo.local                                   │
│   - mike.l3@ad.demo.local                                   │
│                                                              │
│   RBI Auditor asks: "Who deleted the database?"             │
│   Answer: "john.l1 at 3:45 PM from IP 10.0.1.100" ✅       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🏗️ The Big Picture

Think of our solution like a **city with different departments**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                        THE COMPLETE ARCHITECTURE                             │
│                                                                              │
│  ┌─────────────────┐                    ┌─────────────────────────────────┐ │
│  │                 │                    │                                 │ │
│  │  ACTIVE         │       TRUST        │  RED HAT IDM                    │ │
│  │  DIRECTORY      │◄──────────────────►│  (Identity Management)          │ │
│  │                 │   "We recognize    │                                 │ │
│  │  The "HR Dept"  │    each other's    │  The "Security Dept"            │ │
│  │  - Has all      │    employees"      │  - Controls who can enter       │ │
│  │    employee     │                    │    which buildings              │ │
│  │    records      │                    │  - Controls what they can do    │ │
│  │  - Issues ID    │                    │  - Keeps all logs               │ │
│  │    badges       │                    │                                 │ │
│  │                 │                    │                                 │ │
│  └────────┬────────┘                    └─────────────┬───────────────────┘ │
│           │                                           │                      │
│           │                                           │                      │
│           │         ┌─────────────────────────────────┘                      │
│           │         │                                                        │
│           │         ▼                                                        │
│           │    ┌─────────────────────────────────────────────────────┐      │
│           │    │                                                      │      │
│           │    │              LINUX SERVERS                          │      │
│           │    │              (The Buildings)                        │      │
│           │    │                                                      │      │
│           └───►│  When john.l1 tries to enter:                       │      │
│                │  1. Server asks IDM: "Is this person allowed?"      │      │
│                │  2. IDM checks: "Is john.l1 real?" → Asks AD        │      │
│                │  3. AD says: "Yes, john.l1 works here"              │      │
│                │  4. IDM checks: "Can john.l1 enter THIS building?"  │      │
│                │  5. IDM says: "Yes, but he can only VIEW things"    │      │
│                │  6. Server lets john.l1 in with limited access      │      │
│                │                                                      │      │
│                └─────────────────────────────────────────────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Understanding Each Component

### 1. Active Directory (AD) - "The HR Department"

**What is it?**
Think of AD as the company's HR department. HR knows who all the employees are, their ID numbers, which teams they belong to.

**In simple terms:**
- AD stores all users: john.l1, jane.l2, mike.l3
- AD stores all groups: L1_Team, L2_Team, L3_Team
- AD validates passwords: "Is this the right password for john.l1? Yes/No"

**Real-world analogy:**
```
HR Department (Active Directory)
├── Employee Database
│   ├── John (ID: 1001, Team: L1 Operations)
│   ├── Jane (ID: 1002, Team: L2 Admin)
│   └── Mike (ID: 1003, Team: L3 Super Admin)
│
└── When someone claims to be "John":
    → HR checks their ID badge
    → HR confirms: "Yes, this is really John"
```

---

### 2. Red Hat IDM - "The Security Department"

**What is it?**
IDM is like the company's security department. Security doesn't issue ID badges (that's HR/AD), but security controls:
- Which buildings you can enter
- Which rooms you can access
- What equipment you can use

**In simple terms:**
- IDM doesn't store passwords (AD does that)
- IDM stores ACCESS RULES: "L1 can only view logs"
- IDM stores SERVER RULES: "L1 can only access non-prod servers"

**Real-world analogy:**
```
Security Department (IDM)
├── Access Rules
│   ├── L1 Team: Can enter Building A, Can only use READ-ONLY terminals
│   ├── L2 Team: Can enter Building A & B, Can use ADMIN terminals
│   └── L3 Team: Can enter ALL buildings, Can use ALL equipment
│
└── When John (L1) tries to enter the Production Server Room:
    → Security checks: "Is John's team allowed here?"
    → Security says: "No, L1 can't enter production"
    → Access DENIED
```

---

### 3. SSSD - "The Security Guard at Each Building"

**What is it?**
SSSD (System Security Services Daemon) runs on EACH Linux server. It's like a security guard stationed at the door of each building.

**In simple terms:**
- SSSD is installed on every Linux server
- When someone tries to login, SSSD calls IDM
- SSSD caches information (so it doesn't call IDM for every single request)

**Real-world analogy:**
```
Security Guard (SSSD) at Building A entrance:

Person: "Hi, I'm John, let me in"
Guard:  "Let me call the Security Department (IDM)"
        [Calls IDM]
Guard:  "IDM says John is allowed. Come in."
Guard:  [Writes in notebook: "John entered at 10:00 AM"]

Next time John comes:
Guard:  "I remember you from earlier. Come in."
        [This is caching - faster!]
```

**Technical diagram:**
```
┌─────────────────────────────────────────────────────────────┐
│                    LINUX SERVER                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                      SSSD                             │   │
│  │         (The Guard at this server's door)            │   │
│  │                                                       │   │
│  │  Jobs:                                                │   │
│  │  1. Who is john.l1? → Ask IDM → Get UID, GID, groups │   │
│  │  2. Is password correct? → Ask IDM → IDM asks AD     │   │
│  │  3. Can john.l1 access THIS server? → Ask IDM (HBAC) │   │
│  │  4. What commands can john.l1 run? → Ask IDM (Sudo)  │   │
│  │  5. Cache all this info for next time                │   │
│  │                                                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                           │                                  │
│                           │ Talks to                         │
│                           ▼                                  │
│              ┌─────────────────────────┐                    │
│              │         IDM             │                    │
│              │  (Security Department)  │                    │
│              └─────────────────────────┘                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### 4. Kerberos - "The Ticket System"

**What is it?**
Kerberos is an authentication protocol. Think of it like getting a wristband at a music festival.

**Real-world analogy:**
```
Music Festival (Kerberos)

Step 1: Entry Gate (Login)
┌─────────────────────────────────────────┐
│ You show your ID at the entry gate      │
│ Gate gives you a WRISTBAND (ticket)     │
│ "This wristband is valid for 24 hours"  │
└─────────────────────────────────────────┘

Step 2: VIP Area, Food Court, Backstage (Accessing services)
┌─────────────────────────────────────────┐
│ You DON'T show ID again                 │
│ You just show your WRISTBAND            │
│ Guard checks: "Wristband valid? OK, in" │
└─────────────────────────────────────────┘

This is called Single Sign-On (SSO)!
- Login ONCE
- Get a ticket (wristband)
- Use ticket everywhere
- No need to enter password again and again
```

**Technical flow:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         KERBEROS FLOW                                    │
│                                                                          │
│  ┌─────────┐      1. "I'm john.l1,     ┌─────────┐                      │
│  │         │         here's my         │         │                      │
│  │  User   │         password"         │   KDC   │  KDC = Key           │
│  │ john.l1 │ ─────────────────────────►│ (IDM)   │  Distribution       │
│  │         │                           │         │  Center              │
│  │         │      2. "Password OK!     │         │  (Ticket Office)     │
│  │         │◄─────────────────────────│         │                      │
│  │         │         Here's your       │         │                      │
│  │         │         TICKET"           │         │                      │
│  └────┬────┘                           └─────────┘                      │
│       │                                                                  │
│       │ 3. "Let me into Server A,                                       │
│       │     here's my TICKET"                                           │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────┐                                                            │
│  │         │      4. Server checks ticket with KDC                      │
│  │ Server  │         "Is this ticket valid?"                            │
│  │    A    │         "Yes" → Let user in                                │
│  │         │                                                            │
│  └─────────┘                                                            │
│                                                                          │
│  Result: User logged in WITHOUT typing password again! (SSO)            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 5. Sudo Rules - "What Can You DO?"

**What is it?**
Sudo rules control what COMMANDS a user can run with elevated privileges.

**Real-world analogy:**
```
Office Equipment Access:

L1 Intern:
├── Can USE: Photocopier (view/copy documents)
├── Can USE: Coffee machine
└── CANNOT USE: Server room keycard, Admin passwords

L2 Manager:
├── Can USE: Everything L1 can use
├── Can USE: Server room keycard
├── Can USE: Restart services
└── CANNOT USE: Delete databases, Fire employees

L3 Director:
├── Can USE: EVERYTHING
└── Full access to all systems
```

**Technical example:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SUDO RULES                                       │
│                                                                          │
│  L1 Operator (john.l1):                                                 │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ ALLOWED:                           │ DENIED:                    │    │
│  │ sudo cat /var/log/messages    ✅   │ sudo systemctl restart ❌  │    │
│  │ sudo tail -f /var/log/secure  ✅   │ sudo rm -rf /           ❌  │    │
│  │ sudo df -h                    ✅   │ sudo su -               ❌  │    │
│  │ sudo systemctl status sshd   ✅   │ sudo yum install        ❌  │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  L2 Admin (jane.l2):                                                    │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ ALLOWED:                           │ DENIED:                    │    │
│  │ Everything L1 can do          ✅   │ sudo su -               ❌  │    │
│  │ sudo systemctl restart sshd  ✅   │ sudo rm -rf /           ❌  │    │
│  │ sudo vi /etc/hosts           ✅   │                            │    │
│  │ sudo yum install httpd       ✅   │                            │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  L3 Super Admin (mike.l3):                                              │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ ALLOWED:                                                        │    │
│  │ EVERYTHING - Full root access                              ✅   │    │
│  │ sudo su -                                                  ✅   │    │
│  │ Any command as any user                                    ✅   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 6. HBAC Rules - "WHERE Can You Go?"

**What is it?**
HBAC (Host-Based Access Control) controls which SERVERS a user can access.

**Real-world analogy:**
```
Office Building Access:

L1 Intern:
├── Can ENTER: Training Room, Cafeteria, Non-prod Lab
└── CANNOT ENTER: Production Data Center, Executive Floor

L2 Manager:
├── Can ENTER: Everything L1 can enter
├── Can ENTER: Production Data Center
└── CANNOT ENTER: Executive Floor (only C-level)

L3 Director:
└── Can ENTER: ALL areas including Executive Floor
```

**Technical example:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         HBAC RULES                                       │
│                                                                          │
│  Servers in our environment:                                            │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ Non-Production Servers:    │ Production Servers:               │    │
│  │ - dev-app-01              │ - prod-db-01                      │    │
│  │ - test-web-01             │ - prod-app-01                     │    │
│  │ - staging-api-01          │ - prod-web-01                     │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Access Matrix:                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │           │ Non-Prod Servers │ Prod Servers │                  │    │
│  │───────────┼──────────────────┼──────────────┤                  │    │
│  │ L1 john.l1│       ✅         │      ❌      │ Safe to learn!   │    │
│  │ L2 jane.l2│       ✅         │      ✅      │ Can fix prod     │    │
│  │ L3 mike.l3│       ✅         │      ✅      │ Full access      │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  When john.l1 tries SSH to prod-db-01:                                  │
│  → SSSD asks IDM: "Can john.l1 access prod-db-01?"                     │
│  → IDM checks HBAC rules                                                │
│  → IDM says: "No, L1 cannot access production servers"                 │
│  → Connection REJECTED                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 7. The Trust - "Two Companies Recognizing Each Other's IDs"

**What is it?**
A trust is like two companies agreeing to accept each other's employee ID badges.

**Real-world analogy:**
```
Before Trust:
┌─────────────────┐     ┌─────────────────┐
│  Company A      │     │  Company B      │
│  (Windows AD)   │     │  (Linux IDM)    │
│                 │     │                 │
│  "Who is John?  │     │  "Who is John?  │
│   Never heard   │     │   Never heard   │
│   of him!"      │     │   of him!"      │
└─────────────────┘     └─────────────────┘

After Trust:
┌─────────────────┐     ┌─────────────────┐
│  Company A      │◄───►│  Company B      │
│  (Windows AD)   │TRUST│  (Linux IDM)    │
│                 │     │                 │
│  "John works    │────►│  "OK, if A says │
│   for us"       │     │   John is real, │
│                 │     │   I trust that" │
└─────────────────┘     └─────────────────┘
```

**Technical explanation:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         THE TRUST                                        │
│                                                                          │
│   ┌──────────────────┐           ┌──────────────────┐                   │
│   │                  │           │                  │                   │
│   │  ACTIVE          │   Trust   │  RED HAT IDM     │                   │
│   │  DIRECTORY       │◄─────────►│                  │                   │
│   │                  │           │                  │                   │
│   │  Realm:          │           │  Realm:          │                   │
│   │  AD.DEMO.LOCAL   │           │  IDM.DEMO.LOCAL  │                   │
│   │                  │           │                  │                   │
│   │  Has users:      │           │  Trusts AD's     │                   │
│   │  - john.l1       │           │  users           │                   │
│   │  - jane.l2       │           │                  │                   │
│   │  - mike.l3       │           │  Creates rules   │                   │
│   │                  │           │  for AD users    │                   │
│   └──────────────────┘           └──────────────────┘                   │
│                                                                          │
│   How it works:                                                         │
│   1. john.l1 tries to login to Linux server                            │
│   2. Linux server asks IDM: "Who is john.l1@ad.demo.local?"            │
│   3. IDM says: "I don't have john.l1, but I TRUST AD"                  │
│   4. IDM asks AD: "Is john.l1 real? What's his password?"              │
│   5. AD validates john.l1's password                                    │
│   6. IDM then applies ITS rules (sudo, HBAC) to john.l1                │
│   7. john.l1 is logged in with permissions from IDM!                   │
│                                                                          │
│   KEY INSIGHT:                                                          │
│   - AD handles: WHO you are (identity)                                  │
│   - IDM handles: WHAT you can do (authorization)                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 8. The Group Chain - "From AD to Rules"

**Why do we need this?**
AD groups can't directly be used in IDM rules. We need to create a "chain" to connect them.

**Real-world analogy:**
```
International Company:

US Office (AD):
- Has team: "US_Engineering"
- Uses Employee IDs starting with "US-"

India Office (IDM):
- Needs to give access to US_Engineering team
- But can't read US IDs directly!

Solution: Create a "translation":
1. Create "External Group" that says "US_Engineering = these people"
2. Create "Local Group" that IDM understands
3. Link External Group → Local Group
4. Apply rules to Local Group
```

**Technical diagram:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         THE GROUP CHAIN                                  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   AD Group                    (In Active Directory)              │   │
│  │   "Linux_L1_Team"                                                │   │
│  │   Contains: john.l1, sarah.l1                                   │   │
│  │                                                                  │   │
│  └──────────────────────────────┬──────────────────────────────────┘   │
│                                 │                                       │
│                                 │ MAPPED TO                             │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   IDM External Group          (Bridge between AD and IDM)       │   │
│  │   "ad_linux_l1_team"                                            │   │
│  │   External member: linux_l1_team@ad.demo.local                  │   │
│  │                                                                  │   │
│  │   This is like a "translator" - it understands AD's language   │   │
│  │                                                                  │   │
│  └──────────────────────────────┬──────────────────────────────────┘   │
│                                 │                                       │
│                                 │ IS MEMBER OF                          │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   IDM POSIX Group             (Regular Linux group)             │   │
│  │   "linux_l1_operators"                                          │   │
│  │   GID: 989400004                                                │   │
│  │                                                                  │   │
│  │   This is a normal Linux group that rules can reference        │   │
│  │                                                                  │   │
│  └──────────────────────────────┬──────────────────────────────────┘   │
│                                 │                                       │
│                                 │ USED BY                               │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   SUDO RULES                  HBAC RULES                        │   │
│  │   "sudo_l1_operators"         "hbac_l1_operators"               │   │
│  │                                                                  │   │
│  │   Group: linux_l1_operators   Group: linux_l1_operators         │   │
│  │   Commands: cat, tail, df     Hosts: nonprod_servers            │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  RESULT:                                                                │
│  john.l1 → Linux_L1_Team (AD) → ad_linux_l1_team (External) →          │
│  linux_l1_operators (POSIX) → sudo_l1 + hbac_l1 rules                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Complete Login Flow

Let's trace what happens when john.l1 tries to SSH into a Linux server:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE LOGIN FLOW                                   │
│                                                                          │
│  john.l1 types: ssh john.l1@ad.demo.local@linux-server                  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ STEP 1: SSH Connection                                          │    │
│  │ Linux server receives connection request                        │    │
│  │ "Someone claiming to be john.l1@ad.demo.local wants in"        │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ STEP 2: SSSD - "Who is this person?"                           │    │
│  │ SSSD asks IDM: "Tell me about john.l1@ad.demo.local"           │    │
│  │ IDM says: "I trust AD, let me ask them"                        │    │
│  │ IDM asks AD: "Who is john.l1?"                                 │    │
│  │ AD responds: UID=1948001106, Groups: Linux_L1_Team             │    │
│  │ IDM passes this back to SSSD                                   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ STEP 3: Password Check                                          │    │
│  │ SSSD: "john.l1 entered password, is it correct?"               │    │
│  │ IDM forwards to AD (Kerberos)                                  │    │
│  │ AD validates password                                          │    │
│  │ AD: "Yes, password is correct"                                 │    │
│  │ john.l1 gets a Kerberos ticket                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ STEP 4: HBAC Check - "Can they access THIS server?"            │    │
│  │ SSSD asks IDM: "Can john.l1 SSH to this server?"               │    │
│  │ IDM checks HBAC rules:                                         │    │
│  │   - john.l1 is in linux_l1_operators (via group chain)         │    │
│  │   - linux_l1_operators can access nonprod_servers              │    │
│  │   - This server is in nonprod_servers                          │    │
│  │ IDM: "Yes, access allowed"                                     │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ STEP 5: Login Success!                                          │    │
│  │ - Home directory created: /home/john.l1@ad.demo.local          │    │
│  │ - User logged in                                                │    │
│  │ - Audit log: "john.l1 logged in at 10:00 AM"                   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ STEP 6: john.l1 runs "sudo cat /var/log/messages"              │    │
│  │ SSSD asks IDM: "What sudo commands can john.l1 run?"           │    │
│  │ IDM checks sudo rules:                                          │    │
│  │   - john.l1 is in linux_l1_operators                           │    │
│  │   - linux_l1_operators has sudo_l1_operators rule              │    │
│  │   - sudo_l1_operators allows: cat, tail, df, etc.              │    │
│  │ IDM: "cat is allowed"                                          │    │
│  │ Command runs successfully!                                      │    │
│  │ Audit log: "john.l1 ran sudo cat /var/log/messages"            │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ STEP 7: john.l1 tries "sudo systemctl restart sshd"            │    │
│  │ IDM checks sudo rules:                                          │    │
│  │   - sudo_l1_operators does NOT allow systemctl restart         │    │
│  │ IDM: "Command not allowed"                                     │    │
│  │ ERROR: "Sorry, user john.l1 is not allowed to execute..."      │    │
│  │ Audit log: "john.l1 DENIED sudo systemctl restart sshd"        │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🏛️ Network Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│                        AZURE VIRTUAL NETWORK                            │
│                        (10.0.0.0/16)                                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: 10.0.1.0/24                          │   │
│  │                                                                   │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │   │
│  │  │               │  │               │  │               │        │   │
│  │  │   WINDOWS     │  │   IDM         │  │   IDM         │        │   │
│  │  │   AD DC       │  │   PRIMARY     │  │   REPLICA     │        │   │
│  │  │               │  │               │  │               │        │   │
│  │  │  10.0.1.4     │  │  10.0.1.5     │  │  10.0.1.7     │        │   │
│  │  │               │  │               │  │               │        │   │
│  │  │  Services:    │  │  Services:    │  │  Services:    │        │   │
│  │  │  - AD DS      │  │  - LDAP       │  │  - LDAP       │        │   │
│  │  │  - DNS        │  │  - Kerberos   │  │  - Kerberos   │        │   │
│  │  │  - Kerberos   │  │  - DNS        │  │  - DNS        │        │   │
│  │  │               │  │  - CA         │  │  - CA         │        │   │
│  │  │  Users:       │  │  - Samba      │  │  - Samba      │        │   │
│  │  │  - john.l1    │  │               │  │               │        │   │
│  │  │  - jane.l2    │  │  Rules:       │  │  (Replicated  │        │   │
│  │  │  - mike.l3    │  │  - Sudo       │  │   from        │        │   │
│  │  │               │  │  - HBAC       │  │   Primary)    │        │   │
│  │  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘        │   │
│  │          │                  │                   │                │   │
│  │          │      TRUST       │   REPLICATION     │                │   │
│  │          │◄────────────────►│◄─────────────────►│                │   │
│  │          │                  │                   │                │   │
│  │          │                  │                   │                │   │
│  │          │         ┌────────┴───────────────────┘                │   │
│  │          │         │                                             │   │
│  │          │         ▼                                             │   │
│  │          │  ┌───────────────┐                                   │   │
│  │          │  │               │                                   │   │
│  │          │  │   RHEL        │                                   │   │
│  │          │  │   CLIENT      │                                   │   │
│  │          │  │               │                                   │   │
│  │          │  │  10.0.1.6     │                                   │   │
│  │          │  │               │                                   │   │
│  │          │  │  SSSD talks   │                                   │   │
│  │          └─►│  to IDM       │                                   │   │
│  │             │               │                                   │   │
│  │             │  AD users     │                                   │   │
│  │             │  can SSH      │                                   │   │
│  │             │  here!        │                                   │   │
│  │             │               │                                   │   │
│  │             └───────────────┘                                   │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 📝 Glossary (All Terms Explained Simply)

| Term | Simple Explanation |
|------|-------------------|
| **Active Directory (AD)** | Microsoft's "employee database" - stores users and passwords |
| **IDM (Identity Management)** | Red Hat's "security system" - controls what users can do on Linux |
| **SSSD** | The "security guard" on each Linux server that talks to IDM |
| **Kerberos** | The "ticket system" - login once, use ticket everywhere |
| **LDAP** | The "phone book protocol" - how to look up user information |
| **Trust** | Agreement between AD and IDM to recognize each other's users |
| **Realm** | A Kerberos "kingdom" - AD.DEMO.LOCAL, IDM.DEMO.LOCAL |
| **Sudo Rules** | Rules about what COMMANDS users can run |
| **HBAC Rules** | Rules about which SERVERS users can access |
| **External Group** | IDM group that can contain AD groups |
| **POSIX Group** | Regular Linux group with a GID number |
| **Group Chain** | AD Group → External Group → POSIX Group → Rules |
| **DNS** | Converts names to IP addresses (phone book for computers) |
| **CA (Certificate Authority)** | Issues digital certificates for secure communication |
| **Multi-Master** | Both IDM servers are equal - changes sync both ways |
| **SID** | Security Identifier - unique ID for users/groups in Windows |
| **UID/GID** | User ID / Group ID - unique numbers for Linux users/groups |

---

## 🎯 Summary: What We Achieved

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  BEFORE:                              AFTER:                            │
│  ────────                             ──────                            │
│                                                                          │
│  50 admins share 1 account            Each admin has own AD account     │
│  "unixadmin"                          john.l1@ad.demo.local             │
│                                                                          │
│  Everyone is root                     L1: View only                      │
│                                       L2: Service management            │
│                                       L3: Full access                   │
│                                                                          │
│  Anyone can access any server         L1: Non-prod only                 │
│                                       L2: Prod + Non-prod               │
│                                       L3: All servers                   │
│                                                                          │
│  No audit trail                       Full audit:                        │
│  "Who did this?"                      "john.l1 at 3:45 PM"              │
│  "Don't know" 😱                                                         │
│                                                                          │
│  Single IDM server                    2 IDM servers (HA)                │
│  If it fails = outage                 If one fails = no problem         │
│                                                                          │
│  Manual rule on each server           Centralized rules in IDM          │
│  1500 servers = nightmare             1500 servers = same effort        │
│                                                                          │
│  RBI COMPLIANCE: ❌ FAIL              RBI COMPLIANCE: ✅ PASS           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🙋 Still Have Questions?

If you're still confused about any part, here's how to think about it:

**"What is IDM doing?"**
→ IDM is the security guard. It doesn't know who you are (AD does), but it controls what you can do.

**"What is AD doing?"**
→ AD is HR. It knows all employees and validates their ID badges (passwords).

**"What is SSSD doing?"**
→ SSSD is like a local security guard at each building. It calls the main security office (IDM) when needed.

**"Why the group chain?"**
→ AD speaks Windows language, Linux speaks Linux language. The chain is the translator.

**"Why two IDM servers?"**
→ Same reason you have two keys for your house. If you lose one, you still have the other!

---

**Now you understand the complete architecture! 🎉**


