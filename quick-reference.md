# Quick Reference - IDM AD Lab

## IP Addresses

| VM | Private IP | Public IP | Hostname |
|----|------------|-----------|----------|
| AD DC | 10.0.1.4 | (from Azure) | ad.ad.demo.local |
| IDM Primary | 10.0.1.5 | (from Azure) | idm.idm.demo.local |
| RHEL Client | 10.0.1.6 | (from Azure) | client.idm.demo.local |
| IDM Replica | 10.0.1.7 | (from Azure) | idm2.idm.demo.local |

## Credentials

| Account | Username | Password |
|---------|----------|----------|
| Azure VMs | azureuser | RedHat@123456! |
| AD Admin | Administrator | RedHat@123456! |
| IDM Admin | admin | RedHat@123456! |
| L1 User | john.l1@ad.demo.local | RedHat@123456! |
| L1 User | sarah.l1@ad.demo.local | RedHat@123456! |
| L2 User | jane.l2@ad.demo.local | RedHat@123456! |
| L3 User | mike.l3@ad.demo.local | RedHat@123456! |

## Domains

| Domain | Type |
|--------|------|
| ad.demo.local | Active Directory |
| idm.demo.local | Red Hat IDM |
| IDM.DEMO.LOCAL | Kerberos Realm |

## Red Hat Subscription

```
Username: njajodia@redhat.com
Password: Newuser@123
```

## SSH Commands

```bash
# AD DC (RDP only)
# Use Remote Desktop to: <AD_DC_PUBLIC_IP>

# IDM Primary
ssh azureuser@<IDM_PRIMARY_PUBLIC_IP>

# IDM Replica
ssh azureuser@<IDM_REPLICA_PUBLIC_IP>

# RHEL Client
ssh azureuser@<RHEL_CLIENT_PUBLIC_IP>

# As AD User
ssh john.l1@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>
ssh jane.l2@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>
ssh mike.l3@ad.demo.local@<RHEL_CLIENT_PUBLIC_IP>
```

## Common IDM Commands

```bash
# Get admin ticket
kinit admin

# List users
ipa user-find

# Show trust
ipa trust-show ad.demo.local

# List sudo rules
ipa sudorule-find

# List HBAC rules
ipa hbacrule-find

# Test HBAC
ipa hbactest --user=john.l1@ad.demo.local --host=client.idm.demo.local --service=sshd

# Check services
ipactl status
```

## SSSD Commands (on Client)

```bash
# Clear cache
sss_cache -E

# Restart SSSD
systemctl restart sssd

# Check user
id john.l1@ad.demo.local

# Check sudo rules
sudo -l -U john.l1@ad.demo.local
```

## Azure Commands

```bash
# Set variables
export RG_NAME="idm-ad-demo-rg"

# List VMs
az vm list -g $RG_NAME -o table --show-details

# Start all VMs
az vm start -g $RG_NAME -n ad-dc --no-wait
az vm start -g $RG_NAME -n idm-server1 --no-wait
az vm start -g $RG_NAME -n idm-server2 --no-wait
az vm start -g $RG_NAME -n rhel-client --no-wait

# Stop all VMs (deallocate)
az vm deallocate -g $RG_NAME -n ad-dc --no-wait
az vm deallocate -g $RG_NAME -n idm-server1 --no-wait
az vm deallocate -g $RG_NAME -n idm-server2 --no-wait
az vm deallocate -g $RG_NAME -n rhel-client --no-wait

# Delete everything
az group delete --name $RG_NAME --yes --no-wait
```

## Port Reference

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 88 | TCP/UDP | Kerberos |
| 123 | UDP | NTP |
| 389 | TCP | LDAP |
| 443 | TCP | HTTPS |
| 445 | TCP | SMB (Trust) |
| 464 | TCP/UDP | Kerberos Password |
| 636 | TCP | LDAPS |
| 3268-3269 | TCP | LDAP Global Catalog |
| 3389 | TCP | RDP |

## RBAC Summary

| Role | Group | Sudo Access |
|------|-------|-------------|
| L1 | linux_l1_operators | View only (cat, less, df, free, systemctl status) |
| L2 | linux_l2_admins | L1 + restart services, edit /etc/* |
| L3 | linux_l3_superadmins | ALL commands |

## Troubleshooting Quick Fixes

```bash
# DNS not resolving
nmcli con mod "System eth0" ipv4.dns "10.0.1.5"
nmcli con up "System eth0"

# User not resolving
sss_cache -E && systemctl restart sssd

# IDM services not running
ipactl restart

# Check logs
tail -f /var/log/secure
tail -f /var/log/sssd/sssd.log
journalctl -u sssd -f
```

## Files to Remember

| File | Purpose |
|------|---------|
| /etc/sssd/sssd.conf | SSSD configuration |
| /etc/krb5.conf | Kerberos configuration |
| /etc/resolv.conf | DNS configuration |
| /var/log/secure | Auth/sudo logs |
| /var/log/sssd/ | SSSD logs |


