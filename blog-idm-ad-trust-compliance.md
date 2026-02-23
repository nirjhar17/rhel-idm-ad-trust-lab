---
layout: default
title: How We Solved Linux Access Control for a Bank Using Red Hat IDM and Active Directory Trust
---

# How We Solved Linux Access Control for a Bank Using Red Hat IDM and Active Directory Trust

Imagine this scenario. You are the IT security head at a bank with 1,500 Linux servers. An auditor asks you a simple question: "Who deleted the production database last Tuesday at 3:45 PM?"

You check the logs. They all say "unixadmin." That is the shared account used by 50 different administrators. You have no idea who actually did it.

This is exactly the problem we solved for a major bank preparing for their compliance audit.

## The Shared Account Problem

Most enterprises with mixed Windows and Linux environments face this challenge. Windows users authenticate through Active Directory with individual accounts. But on Linux, teams often share a single privileged account like "unixadmin" or "root."

This creates three major problems.

> No individual accountability. When everyone uses the same account, audit logs become meaningless. You cannot trace actions back to specific people.

> No role-based access. A junior operator has the same privileges as a senior administrator. There is no way to restrict what different team members can do.

> Compliance gaps. Regulatory auditors mandate individual traceability, role-based access control, and complete audit trails. Shared accounts fail on all three counts.

The bank we worked with had exactly this setup. Fifty administrators across L1, L2, and L3 support tiers all shared one "unixadmin" account. Their compliance audit was three months away.

## The Solution We Built

We implemented Red Hat Identity Management (IDM) with an Active Directory trust. This elegant solution lets AD users authenticate to Linux servers using their existing AD credentials while IDM controls what they can do.

Here is how the architecture works at a high level.

Active Directory remains the source of truth for user identities. Users keep their existing AD accounts and passwords. Nothing changes on the Windows side.

IDM establishes a Kerberos cross-realm trust with AD. This means IDM trusts AD to verify user identities. When john.l1@ad.demo.local tries to SSH into a Linux server, AD validates the password. No password synchronization needed.

IDM controls Linux-specific access. Even though AD authenticates the user, IDM decides which servers they can access (HBAC rules) and what commands they can run (Sudo rules).

Every action gets logged with the individual username. The audit trail shows "john.l1@ad.demo.local ran sudo systemctl restart httpd" instead of "unixadmin ran sudo systemctl restart httpd."

## How IDM and AD Communicate

Before diving into user flows, it is important to understand how IDM and AD establish trust and communicate. This happens through Kerberos, the authentication protocol used by both systems.

The trust establishment happens once during setup. IDM and AD exchange a shared secret key. This creates a special Kerberos principal that allows tickets from one realm to be trusted by the other. Think of it as both systems agreeing on a secret handshake.

![Kerberos Trust Flow](IMAGE_PLACEHOLDER_KERBEROS_TRUST)

When an AD user needs to access a Linux server, here is what happens behind the scenes.

The user first gets a Ticket Granting Ticket (TGT) from AD. This proves their identity within the AD realm.

When they try to access a Linux server (which is in the IDM realm), AD issues a referral ticket. This is like AD saying "I vouch for this user, IDM should trust them."

The user presents this referral to IDM. IDM validates it using the shared trust key and issues a service ticket for the specific Linux server.

Finally, the user presents the service ticket to the Linux server and gets access.

This entire process happens automatically in milliseconds. The user just types their password once and Kerberos handles the rest.

## How AD Groups Map to IDM Rules

Here is a technical detail that is crucial to understand. You cannot directly add AD users or groups to IDM access rules. AD groups use Security Identifiers (SIDs) which are Windows-specific. Linux needs POSIX groups with Group IDs (GIDs).

The solution is a three-layer group chain.

![Group Chain Diagram](IMAGE_PLACEHOLDER_GROUP_CHAIN)

The first layer is Active Directory. This is where your AD admin creates security groups like Linux_L1_Team, Linux_L2_Team, and Linux_L3_Team. Users like john.l1, jane.l2, and mike.l3 are members of these groups. The AD admin manages all user memberships here.

The second layer is IDM External Groups. These are special IDM groups that reference AD groups by their SID. They have no GID because they are just bridges. You create one external group for each AD group you want to map.

The third layer is IDM POSIX Groups. These are real Linux groups with actual GIDs. The external groups are made members of these POSIX groups. HBAC and Sudo rules reference these POSIX groups.

The chain looks like this.

- AD Group (Linux_L1_Team) contains john.l1
- External Group (ad_linux_l1_team) references the AD group
- POSIX Group (linux_l1_operators) contains the external group
- HBAC and Sudo Rules reference the POSIX group

The beauty of this setup is automation. When the AD admin adds a new user to Linux_L1_Team in Active Directory, that user automatically gets L1 access on all Linux servers. No IDM changes needed. The chain handles everything.

This separation also means clean ownership. The Windows team manages users in AD. The Linux security team manages access policies in IDM. Neither team needs to touch the other's system.

## How Authentication Works

When an AD user like john.l1@ad.demo.local logs into a Linux server, the request flows through a chain of components on the RHEL client.

![Authentication Flow](IMAGE_PLACEHOLDER_AUTHENTICATION)

Here is what each component does.

SSH receives the connection request from the user with their AD credentials.

PAM (Pluggable Authentication Modules) is the Linux framework that handles authentication. It uses configured modules to verify identity.

pam_sss is the specific PAM module that bridges to SSSD. When PAM needs to authenticate an AD user, it calls pam_sss.

SSSD (System Security Services Daemon) is the key component on every Linux client. It connects to IDM, handles the Kerberos ticket exchange, and caches credentials locally for offline access.

IDM receives the authentication request from SSSD. It uses the Kerberos trust to validate with AD.

AD confirms the user identity and password. The response flows back through the chain.

The critical insight here is that there is no direct connection between the Linux server and AD. SSSD always goes through IDM. This means IDM controls access (through HBAC and Sudo rules) even though AD authenticates identity.

## How Authorization Works

Authentication answers "Who are you?" but authorization answers "What can you do?" This is a critical distinction for compliance. AD handles authentication (verifying the password). IDM handles authorization (controlling access).

![Authorization Flow](IMAGE_PLACEHOLDER_AUTHORIZATION)

After a user is authenticated, two checks happen inside IDM.

The first check is HBAC (Host-Based Access Control). This determines which servers a user can access. When John tries to SSH into a production server, SSSD asks IDM: "Is John allowed on this server?" IDM checks the HBAC rules. If John is an L1 operator and the rule says L1 users can only access monitoring servers, the connection is denied before John even gets a shell.

The second check is Sudo Rules. This determines what commands a user can run with elevated privileges. Even if John is allowed on a server, he might not be allowed to run certain commands. When John runs "sudo systemctl restart httpd", the sudo service asks SSSD, which asks IDM: "Can John run this command?" If the rule says L1 users can only run "systemctl status" commands, the restart is denied.

Here is an example of layered authorization in action.

```
# John (L1) logs in - HBAC allows access to this server
john.l1@ad.demo.local$ ssh rhel-client
Welcome to RHEL 9

# John tries monitoring command - Sudo allows it
john.l1@ad.demo.local$ sudo systemctl status httpd
httpd.service - The Apache HTTP Server
   Active: active (running)

# John tries service restart - Sudo denies it
john.l1@ad.demo.local$ sudo systemctl restart httpd
Sorry, user john.l1@ad.demo.local is not allowed to execute '/bin/systemctl restart httpd'

# Audit log captures everything
Jan 06 10:15:45 server sudo: john.l1@ad.demo.local : COMMAND=/usr/bin/systemctl status httpd
Jan 06 10:16:02 server sudo: john.l1@ad.demo.local : command not allowed ; COMMAND=/usr/bin/systemctl restart httpd
```

John has HBAC access to the server (he can log in), but Sudo rules limit what he can do. He can run monitoring commands but cannot restart services. Both the allowed and denied actions are logged with his individual username.

## Implementing Role-Based Access Control

We created three access tiers matching the bank's support structure.

## L1 Operators

These team members can view logs, check system status, and monitor servers. They cannot restart services, edit configurations, or install software. They are typically junior team members learning the ropes. If they make a mistake, it cannot break anything.

Allowed commands include systemctl status, df, free, top, cat for log files.

## L2 Administrators

These team members can do everything L1 can, plus restart services and edit configuration files. They handle most day-to-day operational tasks. But they still cannot get full root access or install arbitrary software.

Additional commands include systemctl restart, systemctl stop, vim for config files.

## L3 Super Administrators

These team members have full sudo access for emergencies and complex troubleshooting. They can become root when needed. But critically, the logs still show mike.l3@ad.demo.local became root, not just "someone became root."

They have sudo ALL access but every command is still logged.

## High Availability for Zero Downtime

Banks cannot tolerate authentication outages. If the identity system goes down, nobody can log into any server.

We deployed two IDM servers in a multi-master replication topology. Both servers are equal. Changes made on either one sync to the other. If the primary fails, the replica keeps working automatically.

During our demo, we literally stopped all services on the primary IDM server. Then we logged in with an AD user account. It worked perfectly because SSSD (the client-side daemon) automatically failed over to the replica.

This meets the compliance requirement for no single point of failure.

## The Audit Trail That Satisfies Auditors

After implementing this solution, the same logs that used to say "unixadmin" now tell a complete story.

```
Jan 06 10:15:23 server sshd: Accepted password for john.l1@ad.demo.local from 10.0.1.100
Jan 06 10:15:45 server sudo: john.l1@ad.demo.local : COMMAND=/usr/bin/systemctl status httpd
Jan 06 10:16:02 server sudo: john.l1@ad.demo.local : command not allowed ; COMMAND=/usr/bin/systemctl restart httpd
Jan 06 10:20:33 server sudo: jane.l2@ad.demo.local : COMMAND=/usr/bin/systemctl restart httpd
```

Now we can answer the auditor's question. Jane (L2) restarted httpd at 10:20. John (L1) tried but was denied. Complete accountability.

## Lessons Learned

DNS is critical. Kerberos authentication depends heavily on DNS. Both AD and IDM need to resolve each other's hostnames. We spent two hours debugging what turned out to be a missing DNS forwarder.

Time synchronization matters. Kerberos tickets have timestamps. If server clocks drift more than five minutes apart, authentication fails with cryptic errors. We use chrony on all servers to keep time in sync.

Test the group chain thoroughly. The external-to-POSIX group mapping is where most mistakes happen. Use the ipa hbactest command to verify access before going live.

Cache clearing is your friend. SSSD caches user and group information aggressively. When troubleshooting, always clear the cache with sss_cache -E and restart sssd.

## The Business Impact

Three months after implementation, the bank passed their compliance audit with zero findings related to Linux access control. The auditors specifically praised the individual accountability and role-based access enforcement.

Beyond compliance, the operations team reported fewer incidents. L1 operators could no longer accidentally restart production services. When issues did occur, the team could immediately identify who did what and why.

The solution also simplified onboarding. New team members get added to the appropriate AD group, and they automatically get the right Linux access. No tickets, no waiting, no manual configuration.

## Getting Started

If you are facing similar challenges, here is how to begin.

Start with a lab environment. We used Azure VMs to build a complete test setup with AD, IDM servers, and a Linux client. This let us make mistakes without impacting production.

Document your access tiers. Before touching any technology, define what each support tier should and should not be able to do. Get sign-off from security and operations.

Plan your group structure. Map out which AD groups will correspond to which Linux access levels. Keep it simple. Three or four tiers usually suffice.

Test with real scenarios. Before going live, have team members from each tier try their typical tasks. Verify L1 cannot do L2 things and vice versa.

The complete lab setup and demo scripts are available on GitHub. They include all the Azure CLI commands, IDM configuration, and sudo rules we used.

## Conclusion

Shared accounts on Linux servers are a compliance liability and an operational risk. With Red Hat IDM and Active Directory trust, you can give your team individual accounts with appropriate privileges while satisfying even the strictest audit requirements.

The setup takes effort, but the payoff is worth it. Your auditors will thank you. Your security team will thank you. And when something goes wrong at 3 AM, you will know exactly who to call.

## About Me

I work on OpenShift, OpenShift AI, and observability solutions, focusing on simplifying complex setups into practical, repeatable steps for platform and development teams.

GitHub: [github.com/nirjhar17](https://github.com/nirjhar17)

LinkedIn: [linkedin.com/in/nirjhar-jajodia](https://linkedin.com/in/nirjhar-jajodia)

## Disclaimer

The views and opinions expressed in this article are my own and do not necessarily reflect the official policy or position of my employer. This guide is provided for educational purposes, and I make no warranties about the completeness, reliability, or accuracy of this information.
