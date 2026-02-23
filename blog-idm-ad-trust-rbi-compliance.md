# How We Solved Linux Access Control for a Bank Using Red Hat IDM and Active Directory Trust

Imagine this scenario. You are the IT security head at a bank with 1,500 Linux servers. An RBI auditor asks you a simple question: "Who deleted the production database last Tuesday at 3:45 PM?"

You check the logs. They all say "unixadmin." That is the shared account used by 50 different administrators. You have no idea who actually did it.

This is exactly the problem we solved for a major Indian bank preparing for their RBI compliance audit.

## The Shared Account Problem

Most enterprises with mixed Windows and Linux environments face this challenge. Windows users authenticate through Active Directory with individual accounts. But on Linux, teams often share a single privileged account like "unixadmin" or "root."

This creates three major problems.

> No individual accountability. When everyone uses the same account, audit logs become meaningless. You cannot trace actions back to specific people.

> No role-based access. A junior operator has the same privileges as a senior administrator. There is no way to restrict what different team members can do.

> RBI compliance gaps. The Reserve Bank of India mandates individual traceability, role-based access control, and complete audit trails. Shared accounts fail on all three counts.

The bank we worked with had exactly this setup. Fifty administrators across L1, L2, and L3 support tiers all shared one "unixadmin" account. Their RBI audit was three months away.

## The Solution We Built

We implemented Red Hat Identity Management (IDM) with an Active Directory trust. This elegant solution lets AD users authenticate to Linux servers using their existing AD credentials while IDM controls what they can do.

## How Authentication Actually Works

When an AD user like john.l1@ad.demo.local logs into a Linux server, the request flows through a chain of components.

```
┌─────────────────────────────────────────────────────────────────┐
│                     RHEL Client                                 │
│                                                                 │
│   User (john.l1) ──► SSH ──► PAM ──► SSSD ──► IDM ──► AD       │
│                              │                                  │
│                         pam_sss                                 │
│                       (PAM module)                              │
└─────────────────────────────────────────────────────────────────┘
```

Here is what each component does in the chain.

- SSH receives the connection request from the user
- PAM (Pluggable Authentication Modules) handles authentication using configured modules
- pam_sss is the PAM module that bridges to SSSD
- SSSD (System Security Services Daemon) connects to IDM and caches credentials locally
- IDM validates the user via the Kerberos cross-realm trust
- AD confirms the user identity and password

This chain is important because it shows there is no direct connection between the Linux server and AD. SSSD always goes through IDM, which means IDM controls what the user can do even though AD authenticates who they are.

Here is how it works at a high level.

Active Directory remains the source of truth for user identities. Users keep their existing AD accounts and passwords. Nothing changes on the Windows side.

IDM establishes a Kerberos cross-realm trust with AD. This means IDM trusts AD to verify user identities. When john.l1@ad.demo.local tries to SSH into a Linux server, AD validates the password. No password synchronization needed.

IDM controls Linux-specific access. Even though AD authenticates the user, IDM decides which servers they can access (HBAC rules) and what commands they can run (sudo rules).

Every action gets logged with the individual username. The audit trail shows "john.l1@ad.demo.local ran sudo systemctl restart httpd" instead of "unixadmin ran sudo systemctl restart httpd."

## Implementing Role-Based Access Control

We created three access tiers matching the bank's support structure.

## L1 Operators

These team members can view logs, check system status, and monitor servers. They cannot restart services, edit configurations, or install software. They are typically junior team members learning the ropes. If they make a mistake, it cannot break anything.

## L2 Administrators

These team members can do everything L1 can, plus restart services and edit configuration files. They handle most day-to-day operational tasks. But they still cannot get full root access or install arbitrary software.

## L3 Super Administrators

These team members have full sudo access for emergencies and complex troubleshooting. They can become root when needed. But critically, the logs still show mike.l3@ad.demo.local became root, not just "someone became root."

Here is what this looks like in practice.

```
# L1 operator tries to restart a service
john.l1@ad.demo.local$ sudo systemctl restart httpd
Sorry, user john.l1@ad.demo.local is not allowed to execute '/bin/systemctl restart httpd' as root.

# L1 operator checks service status (allowed)
john.l1@ad.demo.local$ sudo systemctl status httpd
httpd.service - The Apache HTTP Server
   Active: active (running)
```

The L1 operator can see the service is running but cannot restart it. This is the principle of least privilege in action.

## The Group Chain That Makes It Work

Here is a technical detail that tripped us up initially. You cannot directly add AD users to IDM sudo rules. AD users do not exist in IDM's LDAP directory.

The solution is a three-tier group hierarchy.

First, you create an External Group in IDM. This special group type can contain references to AD groups using their Security Identifiers (SIDs). You add the AD group "Linux_L1_Team" to the external group "ad_linux_l1_team."

Second, you create a POSIX Group in IDM. This is a regular Linux group with a GID number. You make the external group a member of this POSIX group.

Third, you create your sudo and HBAC rules referencing the POSIX group.

The chain looks like this: AD User belongs to AD Group, which maps to External Group, which belongs to POSIX Group, which is referenced by access rules.

It sounds complicated, but once set up, it works automatically. When John joins the Linux_L1_Team group in AD, he immediately gets L1 access on all Linux servers. No manual provisioning needed.

## High Availability for Zero Downtime

Banks cannot tolerate authentication outages. If the identity system goes down, nobody can log into any server.

We deployed two IDM servers in a multi-master replication topology. Both servers are equal. Changes made on either one sync to the other. If the primary fails, the replica keeps working automatically.

During our demo, we literally stopped all services on the primary IDM server. Then we logged in with an AD user account. It worked perfectly because SSSD (the client-side daemon) automatically failovered to the replica.

This meets the RBI requirement for no single point of failure.

## The Audit Trail That Satisfies Auditors

After implementing this solution, the same logs that used to say "unixadmin" now tell a complete story.

```
Jan 06 10:15:23 server sshd: Accepted password for john.l1@ad.demo.local from 10.0.1.100
Jan 06 10:15:45 server sudo: john.l1@ad.demo.local : COMMAND=/usr/bin/cat /var/log/messages
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

Three months after implementation, the bank passed their RBI audit with zero findings related to Linux access control. The auditors specifically praised the individual accountability and role-based access enforcement.

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
