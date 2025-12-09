# F5 BIG-IP Configuration Standardization – Phase 1
## Complete Jira Ticket Sections & Parameters Breakdown

**Project:** Infrastructure Automation / Network Operations  
**Objective:** Define standard configuration parameters for all 300 F5 BIG-IP Load Balancers  
**Status:** Phase 1 – Documentation & Definition  
**Version:** 1.0  
**Last Updated:** December 2025

---

## Overview

This document defines **12 major configuration sections** that map to Jira Epic/Story templates. Each section contains specific **parameters** that must be standardized and validated. Every parameter has a **standard value**, **acceptable range**, **notes**, and **validation criteria**.

Each section becomes either:
- A **single Jira Story** (if parameters can be bulk-validated)
- **Multiple Jira Stories** (for complex domains like profiles, virtual servers)
- **Jira Tasks** underneath parent Stories

---

## Section 1: Platform & System Baseline

**Jira Ticket:** `EPIC: BIG-IP Platform & System Baseline Standards`  
**Owner:** Platform Infrastructure Team  
**Scope:** Per device (or HA pair)  
**Validation Strategy:** Manual audit + API collection

### 1.1 Software & Licensing

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **BIG-IP Version** | See environment matrix | v11.6, v12.x, v13.x, v14.x, v15.x | bigip_ltm 15.1.5 | Use consistent versions per environment (Prod vs Non-Prod) | `tmsh show sys version` |
| **Software Build / HF Level** | Latest HF within version | e.g., 15.1.5 or later | 15.1.5-0.0.54 | Security patches mandatory within 30 days | `tmsh show sys version` |
| **Licensed Modules** | Prod: LTM, ASM; Non-Prod: LTM | Conditional per use case | LTM, ASM, AFM (optional) | Restrict unnecessary module licensing | `tmsh list sys license` |
| **Module Provisioning** | LTM: Nominal; Others: Minimum unless needed | nomical, minimum, or dedicated | LTM: nominal | Over-provisioning wastes resources | iControl REST: `/mgmt/tm/sys/provisioning` |
| **License Expiry** | Alert if < 90 days | Must not expire in production | Expires: 2026-06-15 | Track via compliance monitoring | `tmsh list sys license` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Device Version matches approved list for environment
- [ ] Build level is within approved HF release
- [ ] Required modules are licensed
- [ ] Module provisioning matches standard
- [ ] License will not expire within 90 days

---

### 1.2 System Identity & Naming

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Hostname** | `{env}-bigip-{location}-{sequence}` | env: prod/non-prod/dev; location: dc code; sequence: 01-99 | prod-bigip-dc1-01 | No spaces, lowercase, max 63 chars | `tmsh list sys global-settings hostname` |
| **Device Name (in cluster)** | Same as hostname | Unique within HA group | prod-bigip-dc1-01 | Must match when in HA device group | `tmsh list cm device` |
| **Product Module** | BIG-IP LTM | Constant | BIG-IP LTM | Should never change | `tmsh show sys version` |
| **Platform Type** | VE-XXXX or Z-Series | VE for virtual, specific hardware model for appliances | Virtual Edition or Z252 | Document for capacity planning | `tmsh show sys hardware` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Hostname follows naming convention
- [ ] Device name (in HA) matches hostname
- [ ] Platform type is documented
- [ ] No conflicting device names in cluster

---

### 1.3 System Global Settings

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Management Hostname** | Same as hostname | Single FQDN | prod-bigip-dc1-01.corp.com | For DNS reverse lookups | `tmsh list sys global-settings` |
| **mgmt-dhcp** | Disabled (static IP) | enabled / disabled | disabled | Static IPs only in production | `tmsh list sys global-settings mgmt-dhcp` |
| **LCD Display** | Enabled (if hardware appliance) | enabled / disabled | enabled | Useful for physical troubleshooting | `tmsh list sys global-settings lcd-display` |
| **GUI Setup Page** | Disabled after initial setup | enabled / disabled | disabled | Security: disable Setup wizard | `tmsh list sys global-settings gui-setup` |
| **mgmt-https** | Enabled | enabled / disabled | enabled | Always use HTTPS for management | `tmsh list sys global-settings mgmt-https` |
| **mgmt-https-allow-ipv6** | Per policy (usually disabled) | enabled / disabled | disabled | Restrict unless IPv6 required | `tmsh list sys global-settings mgmt-https-allow-ipv6` |
| **Max Privilege** | admin (default) | admin only | admin | No unprivileged users for BIG-IP | `tmsh list sys global-settings max-privilege` |

**Key Database (sys db) Parameters:**
| db Parameter | Standard | Range | Reason |
|---|---|---|---|
| `setup.run` | true | true / false | Setup completed |
| `systemauth.disablerootlogin` | true | true / false | Security: disable root SSH |
| `systemauth.disablelocaladminlockout` | false | true / false | Enable account lockout on failed logins |
| `systemauth.primaryadminuser` | Custom (not "admin") | string | Change from default "admin" |
| `system.urldb.mode` | nosync | nosync / sync-to-group / sync-from-group | Keep local, don't sync unless needed |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Global settings match template
- [ ] Critical sys.db keys set per standard
- [ ] Root login disabled
- [ ] Primary admin user is not "admin"
- [ ] Setup wizard is disabled

---

## Section 2: Management Access & Hardening (Security)

**Jira Ticket:** `STORY: Management Plane Security & Access Control Standards`  
**Owner:** Security Team + Network Automation  
**Scope:** Per device  
**Validation Strategy:** API + SSH/GUI access tests

### 2.1 Management Network Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Management Port IP** | X.X.X.X/24 (static) | RFC 1918 or public depending on network | 192.168.1.10 | Management must be separate from data plane | `tmsh list sys management-ip` / `config` utility |
| **Management Gateway** | X.X.X.0/24 gateway | Default route for mgmt VLAN | 192.168.1.1 | Ensure route to SOC, jump hosts, etc. | `tmsh list sys management-route` |
| **Management Routes** | Explicit routes only | No default route via mgmt | 10.0.0.0/8 via 192.168.1.1 | Restrict management traffic scope | `tmsh list sys management-route` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Management port has static IP
- [ ] Management network is isolated from data VLANs
- [ ] Routing is explicit (no catchall defaults)

---

### 2.2 GUI/HTTPS Access Restrictions

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **HTTP GUI Access** | Disabled | disabled / enabled | disabled | HTTPS only |  `tmsh list sys httpd port` (should not be on 80) |
| **HTTPS Port** | 443 | 443 / custom (not std ports) | 443 | Standard port for consistency | `tmsh list sys httpd port` |
| **Allowed IPs for GUI** | Restricted list | CIDR notation | 10.50.0.0/16 (SOC), 10.100.0.0/16 (ops) | Allow only from trusted networks | `tmsh list sys httpd allow` |
| **GUI Idle Timeout** | 900 seconds (15 min) | 300–1800 (5–30 min) | 900 | Lock GUI on inactivity | `tmsh list sys httpd auth-pam-idle-timeout` |
| **GUI Session Limit** | 5 simultaneous sessions per user | 1–10 | 5 | Prevent credential sharing | `tmsh list sys httpd max-auth-requests` |
| **HTTPS Cipher Suite** | TLSv1.2+ only | TLSv1.2, TLSv1.3 | See SSL Profiles section | Use strong modern ciphers | `tmsh list sys httpd ssl-protocol` |
| **TLS Version** | TLSv1.2 minimum | TLSv1.2 / TLSv1.3 | TLSv1.2 | Disable older TLS versions | `tmsh list sys httpd ssl-protocol` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] HTTP GUI disabled
- [ ] HTTPS on standard port 443
- [ ] Allowed IP list is restricted (no 0.0.0.0/0)
- [ ] Idle timeout is set (900 sec standard)
- [ ] Session limit enforced
- [ ] TLS 1.2+ only

---

### 2.3 SSH Access Restrictions

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **SSH Service** | Enabled but restricted | enabled / disabled | enabled | Required for automation, but controlled | `tmsh list sys sshd` |
| **SSH Port** | 22 | 22 / custom | 22 | Standard unless SOC policy differs | `tmsh list sys sshd port` |
| **SSH Allowed IPs** | Restricted list (not 0.0.0.0/0) | CIDR notation | 10.50.0.0/16, 10.100.0.0/16 | Allow only from SOC, automation systems | `tmsh list sys sshd allow` |
| **SSH Ciphers** | Strong only | Exclude: MD5, DES, RC4 | aes128-ctr, aes256-ctr | Align with corporate SSH hardening standard | `tmsh list sys sshd ciphers` |
| **SSH Key Exchange** | Strong algorithms | Exclude: diffie-hellman-group1-sha1 | diffie-hellman-group14-sha256 | Modern KEX only | `tmsh list sys sshd key-exchange` |
| **SSH MACs** | Strong only | Exclude: md5, sha1 | hmac-sha2-256, hmac-sha2-512 | Strong message authentication | `tmsh list sys sshd macs` |
| **SSH Inactivity Timeout** | 900 seconds (15 min) | 300–1800 (5–30 min) | 900 | Auto-disconnect idle sessions | `tmsh list sys sshd inactivity-timeout` |
| **SSH Root Login** | Disabled | disabled / enabled | disabled | Disable direct root SSH | `tmsh list sys sshd disable-root-login` |
| **SSH Password Auth** | Enabled (if key auth unavailable) | enabled / disabled | enabled | Prefer key auth, but allow password as fallback | `tmsh list sys sshd passwd-auth` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] SSH allowed IPs are restricted
- [ ] SSH ciphers/KEX/MACs are strong
- [ ] Inactivity timeout is 900 sec
- [ ] Root login disabled
- [ ] Password auth policy matches standard

---

### 2.4 User Accounts & Authentication

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Admin Account** | Changed from "admin" | Custom name, min 6 chars | automation_admin | Default "admin" account must be disabled/renamed | `tmsh list auth user admin disabled` |
| **Admin Password Policy** | See table below | Min 10 chars, complexity | Enforced by policy | Set via `auth password-policy` | `tmsh list auth password-policy` |
| **Root Account** | Disabled for login | disabled only | n/a | `systemauth.disablerootlogin = true` | `tmsh list sys db systemauth.disablerootlogin` |
| **Guest Account** | Disabled or removed | Disabled | n/a | No guest access allowed | `tmsh list auth user guest` |

**Password Policy Standard:**

| Policy Parameter | Standard Value | Description |
|---|---|---|
| `minimum-length` | 10 | Minimum password length |
| `required-uppercase` | 1 | At least 1 uppercase letter |
| `required-lowercase` | 1 | At least 1 lowercase letter |
| `required-numeric` | 1 | At least 1 digit |
| `required-special` | 1 | At least 1 special character |
| `password-memory` | 3 | Cannot reuse last 3 passwords |
| `max-duration` | 90 | Change password every 90 days |
| `max-login-failures` | 3 | Lock account after 3 failed logins |

**Command to set:**
```
tmsh modify auth password-policy minimum-length 10 required-uppercase 1 required-lowercase 1 required-numeric 1 required-special 1 password-memory 3 max-duration 90 max-login-failures 3
```

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Admin account is not "admin"
- [ ] Password policy enforced (min 10 chars, complexity, expiry 90 days)
- [ ] Root login disabled
- [ ] Guest account disabled
- [ ] Account lockout after 3 failed attempts

---

### 2.5 Remote Authentication (AAA)

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Remote Auth Method** | LDAP / RADIUS / TACACS+ | One of: LDAP, RADIUS, TACACS+ | LDAP (Active Directory) | Centralized auth mandatory for compliance | `tmsh list auth source` |
| **Auth Server 1** | Primary AD/LDAP server | FQDN or IP | ldap1.corp.com:389 | Redundancy: at least 2 servers | `tmsh list auth ldap` |
| **Auth Server 2** | Secondary AD/LDAP server | FQDN or IP | ldap2.corp.com:389 | Fallback for high availability | `tmsh list auth ldap` |
| **Auth Timeout** | 30 seconds | 10–60 seconds | 30 | Don't hang on failed auth server | `tmsh list auth ldap timeout` |
| **Fallback to Local** | Enabled (if possible) | enabled / disabled | enabled | Allow local admin if remote auth fails | `tmsh list auth ldap fallback` |
| **Remote Role Mapping** | Map to BIG-IP admin role | string | CN=BIG-IP-Admins,OU=Groups | Users in this group → admin role on BIG-IP | `tmsh list auth ldap remote-role-mapping` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Remote auth source is configured (not local only)
- [ ] At least 2 auth servers for redundancy
- [ ] Auth timeout is reasonable (30 sec)
- [ ] Fallback to local auth enabled
- [ ] Remote roles map correctly

---

## Section 3: Network Layer (Data Plane Underlay)

**Jira Ticket:** `EPIC: Network – VLANs, Self IPs & Routing Standards`  
**Owner:** Network Infrastructure Team  
**Scope:** Per device or HA pair  
**Validation Strategy:** API collection + manual verification

### 3.1 VLAN Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **VLAN Naming** | `{type}_{zone}_{vlan_id}` or descriptive | Lowercase, no spaces, max 32 chars | ext_client_100, int_backend_101 | Consistent across all devices | `tmsh list net vlan` |
| **VLAN Tagged** | True (except 1:1 internal links) | tagged / untagged | tagged on all interfaces | 802.1Q tagging standard | `tmsh list net vlan` |
| **VLAN MTU** | 1500 (standard) | 1500 / 9000 (jumbo) | 1500 | Jumbo only if backend supports and documented | `tmsh list net vlan mtu` |
| **VLAN Interface** | eth0, eth1, eth2, etc. | Physical interfaces | eth0, eth1 | Map to physical switch ports | `tmsh list net vlan interfaces` |
| **HA VLAN (if clustered)** | Dedicated VLAN for HA traffic | Separate VLAN, high speed | ha_sync_vlan_200 | Low-latency HA heartbeat | `tmsh list net vlan` |
| **Management VLAN (if separated)** | Separate VLAN if not on eth0:0 | Isolated from data | mgmt_vlan_10 | Best practice: isolated mgmt plane | `tmsh list net vlan` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] All VLANs follow naming convention
- [ ] VLANs tagged/untagged correctly
- [ ] MTU settings match network requirements
- [ ] HA VLAN is dedicated (if clustered)
- [ ] No duplicate VLAN IDs across devices

---

### 3.2 Self IP Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Self IP Per VLAN** | One floating + one non-floating (in HA) | One per VLAN minimum | int_backend_self: 10.1.1.10 | Enables BIG-IP to route on that VLAN | `tmsh list net self` |
| **Self IP Naming** | `{vlan_name}_self` or `self_{vlan_name}` | Lowercase, max 32 chars | int_backend_self | Consistent naming for automation | `tmsh list net self` |
| **Self IP Netmask** | Match network CIDR | /23, /24, /25 etc. | 255.255.255.0 (/24) | Standard subnetting per network | `tmsh list net self` |
| **Traffic Group** | traffic-group-1 (for floating IPs in HA) | traffic-group-1 or per-vlan groups | traffic-group-1 | Floating IPs float to active unit in HA | `tmsh list net self traffic-group` |
| **Floating IP (HA)** | Enabled for VLANs with traffic | enabled / disabled | enabled on client-facing VLANs | Ensures continuity on failover | `tmsh list net self floating` |
| **Port Lockdown** | "Allow None" (whitelist approach) | Allow None / Allow Default / Allow Custom | Allow None | Most secure: only allow required services | `tmsh list net self allow-service` |

**Port Lockdown Standard (when set to "Allow Custom"):**

| Service | Port | Allow | Reason |
|---|---|---|---|
| `tcp:443` | 443 | YES | HTTPS GUI access |
| `tcp:22` | 22 | YES | SSH (if needed for automation) |
| `tcp:8443` | 8443 | CONDITIONAL | iControl REST (if used) |
| `tcp:4353` | 4353 | YES (if HA) | HA heartbeat (HA VLAN) |
| `udp:520` | 520 | NO | RIP (disable) |
| `udp:161` | 161 | CONDITIONAL | SNMP (if monitoring required) |
| Everything else | - | NO | Deny by default |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] One self IP per VLAN minimum
- [ ] Port lockdown set to "Allow None" OR "Allow Custom" with whitelist
- [ ] No broad port lockdown like "Allow Default"
- [ ] Floating IPs enabled on data-plane VLANs
- [ ] HA VLAN has dedicated self IPs

---

### 3.3 Static Routes

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Default Route** | Single default route | Unique gateway per environment | 0.0.0.0/0 via 10.1.1.1 | Explicit, no ambiguity | `tmsh list net route` |
| **Backend Routes** | Explicit for each backend CIDR | /24, /25, etc. per network | 10.0.0.0/16 via 10.1.2.1 | No overlapping ranges | `tmsh list net route` |
| **Route Naming** | Optional but recommended | Descriptive string | backend_private_network | Helps with troubleshooting | `tmsh list net route` |
| **Route Ordering** | More specific first (longest prefix match) | RFC 1918 + specific subnets | 10.0.0.0/16, then 0.0.0.0/0 | Standard BGP-like behavior | `tmsh list net route` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Default route is defined
- [ ] Backend routes are explicit
- [ ] No circular or conflicting routes
- [ ] Route tables consistent across HA pair

---

### 3.4 SNAT Address Objects (if used)

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **SNAT Automap (vs. Explicit SNAT)** | Use Automap when possible; explicit pools when required | automap / ip address / snat pool | SNAT Automap | Simplest: source becomes self IP on egress VLAN | `tmsh list ltm snat` |
| **SNAT Pool Naming** | `snat_pool_{purpose}` | Lowercase, descriptive | snat_pool_backend_nat | Centralized SNAT pool for multiple VIPs | `tmsh list ltm snat-pool` |
| **SNAT Pool IPs** | Per requirements (1 or more IPs) | Routable IPs on backend VLAN | 10.1.2.100-10.1.2.110 | Static pool, not dynamic | `tmsh list ltm snat-pool addresses` |

**When to Use SNAT vs. No SNAT:**
- **Use SNAT (or Automap):** Backend servers not on same VLAN as self IP; servers have default gateway elsewhere
- **No SNAT:** Backend servers can route directly back to client (e.g., F5 is on-path or is their default gateway)

**Validation Checklist (Jira Sub-Tasks):**
- [ ] SNAT strategy defined (Automap vs. explicit)
- [ ] SNAT pools have meaningful names
- [ ] SNAT IPs are routable from backend networks
- [ ] No overlapping SNAT pool addresses

---

## Section 4: Device Service Clustering / HA

**Jira Ticket:** `EPIC: High Availability & Device Service Cluster Standards`  
**Owner:** Infrastructure + HA Team  
**Scope:** Per HA pair/cluster  
**Validation Strategy:** Cluster status queries, configuration comparison

### 4.1 Device Group & Cluster Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Device Group Name** | `{env}_dg_{location}` | Lowercase, descriptive, max 32 chars | prod_dg_dc1 | Consistent naming | `tmsh list cm device-group` |
| **Device Group Type** | sync-failover | sync-failover / sync-only | sync-failover | Standard for active-standby HA | `tmsh list cm device-group type` |
| **Members in Group** | 2 (for active-standby) or N (for active-active) | 2–8 depending on setup | prod-bigip-dc1-01, prod-bigip-dc1-02 | All members must be present and valid | `tmsh list cm device-group members` |
| **Membership Status** | All devices "In Sync" | In Sync / Awaiting Initial Sync / In Recovery | In Sync | No partial HA groups | `tmsh show cm device-group sync-status` |
| **Auto Sync** | Enabled | enabled / disabled | enabled | Changes propagate automatically | `tmsh list cm device-group auto-sync` |
| **Sync Full When | Enabled (after changes) | enabled / disabled | enabled | Verify full sync on next save | `tmsh list cm device-group sync-full-on-load` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Device group name follows standard
- [ ] Type is sync-failover (or sync-only if documented)
- [ ] All devices are members
- [ ] All devices show "In Sync"
- [ ] Auto-Sync enabled

---

### 4.2 Failover & Network Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Network Failover** | Enabled | enabled / disabled | enabled | Allows failover on network issues | `tmsh list cm failover-status` |
| **Failover Unicast Addresses** | Both units configured | IP:port per unit | unit1: 10.1.1.10:1026, unit2: 10.1.1.11:1026 | Heartbeat between devices | `tmsh list cm device` or `list cm failover-unicast-addresses` |
| **Failover HA VLAN** | Dedicated VLAN (low-latency) | Separate VLAN from data | ha_sync_200 | Fast heartbeat, low-latency link | `tmsh list net vlan` |
| **Primary Device** | Assigned explicitly | Device name | prod-bigip-dc1-01 | Designates preferred active device | `tmsh show cm failover-status` |
| **Priority / Weight** | Primary: 100, Secondary: 0 | 0–255 | Primary: 100 | Controls which device is active by default | `tmsh list cm device priority` |
| **Heartbeat Timeout** | 10 seconds | 3–30 seconds | 10 | How long to wait before declaring peer dead | `tmsh list cm device heartbeat-timeout` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Network failover enabled
- [ ] Failover unicast IPs configured (both units)
- [ ] HA VLAN exists and is low-latency
- [ ] Primary device is designated
- [ ] Heartbeat timeout is reasonable (10 sec)

---

### 4.3 Traffic Groups & Floating Objects

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Default Traffic Group** | traffic-group-1 | traffic-group-1 / custom groups | traffic-group-1 | Default group for most objects | `tmsh list cm traffic-group` |
| **Floating Self IPs** | Floating on data-plane VLANs | On client, server, HA VLANs | int_client_self, int_backend_self | Move with traffic-group on failover | `tmsh list net self floating` |
| **Floating Virtual Addresses** | All VIPs should float | Per virtual server | Virtual IP for each app | Move with traffic-group on failover | `tmsh list ltm virtual-address` |
| **Traffic Group Failover Order** | Primary → Secondary | Ordered list of units | prod-bigip-dc1-01 → prod-bigip-dc1-02 | Determines failover sequence | `tmsh list cm traffic-group failover-order` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Traffic-group-1 is default and active
- [ ] All floating IPs assigned to traffic-group-1
- [ ] Failover order is Primary → Secondary
- [ ] No objects pinned to specific device (unless documented exception)

---

### 4.4 Mirror & Synchronization (if used)

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Connection Mirroring** | Enabled (for stateful failover) | enabled / disabled | enabled | Syncs active connections on failover | `tmsh list cm device connection-mirroring` |
| **Mirror IP** | Dedicated IP on HA VLAN | IP per unit | Unit1: 10.1.200.10, Unit2: 10.1.200.11 | Low-latency mirroring traffic | `tmsh list cm device mirror-ip` |
| **Mirror VLAN** | HA VLAN | Separate low-latency VLAN | ha_sync_200 | Same as failover HA VLAN | `tmsh list cm device mirror-vlan` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Connection mirroring enabled (if stateful services)
- [ ] Mirror IPs configured on HA VLAN
- [ ] Mirror VLAN is dedicated (not shared with data)

---

## Section 5: System Services – DNS, NTP, SNMP, Logging

**Jira Ticket:** `EPIC: System Services – DNS, NTP, SNMP & Syslog Standards`  
**Owner:** Network Operations / Systems Team  
**Scope:** Per device  
**Validation Strategy:** Centralized configuration checks

### 5.1 DNS Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **DNS Server 1 (Primary)** | Corporate DNS 1 | Valid DNS IP, RFC 1918 or ISP | 8.8.8.8 or internal IP | Redundancy: at least 2 servers | `tmsh list sys dns` |
| **DNS Server 2 (Secondary)** | Corporate DNS 2 | Valid DNS IP | 8.8.4.4 or internal IP | Fallback for high availability | `tmsh list sys dns` |
| **DNS Server 3 (Tertiary)** | Optional | Valid DNS IP | 1.1.1.1 | Additional redundancy (optional) | `tmsh list sys dns` |
| **DNS Search Domain** | corp.com (primary domain) | Domain suffix list | corp.com | For short name resolution | `tmsh list sys dns search` |
| **DNS Timeout** | 3 seconds | 1–10 seconds | 3 | Don't hang on DNS failures | Built-in default |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] At least 2 DNS servers configured
- [ ] DNS servers are reachable
- [ ] Search domain matches corporate standard
- [ ] DNS resolution works (test via tmsh)

---

### 5.2 NTP Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **NTP Server 1** | Corporate NTP / stratum 2+ | FQDN or IP | ntp1.corp.com or 10.10.10.10 | At least 2 for redundancy | `tmsh list sys ntp` |
| **NTP Server 2** | Corporate NTP 2 / stratum 2+ | FQDN or IP | ntp2.corp.com | Fallback | `tmsh list sys ntp` |
| **NTP Server 3** | Public NTP (backup only) | pool.ntp.org or cloud provider | pool.ntp.org | Optional fallback | `tmsh list sys ntp` |
| **NTP Timezone** | UTC (or per region policy) | UTC, EST, PST, etc. | UTC | Consistent with enterprise policy | `tmsh list sys ntp timezone` |
| **NTP Status** | Synchronized | In sync / Out of sync | In sync | Critical for logging, HA, SSL certs | `tmsh show sys ntp` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] At least 2 NTP servers configured
- [ ] NTP servers are reachable
- [ ] Time sync status is "In sync"
- [ ] Timezone matches policy

---

### 5.3 SNMP Configuration (if monitoring enabled)

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **SNMP Community (v2c)** | Read-only string (custom) | Custom, not "public" | corp-ro-community | SNMPv2c is legacy; v3 preferred | `tmsh list sys snmp` |
| **SNMP Version** | v3 preferred, v2c acceptable | v2c / v3 | v3 with authentication | v3 has encryption, auth | `tmsh list sys snmp version` |
| **SNMP Trap Destination 1** | Monitoring server IP:port | IP:162 | 10.50.100.10:162 | Send traps to NOC monitoring system | `tmsh list sys snmp trap-destination` |
| **SNMP Trap Destination 2** | Backup monitoring server | IP:162 | 10.50.100.11:162 | Redundancy | `tmsh list sys snmp trap-destination` |
| **SNMP Contact** | Team name or email | String | NetOps Team: netops@corp.com | For documentation | `tmsh list sys snmp sys-contact` |
| **SNMP Location** | Data center and device purpose | String | DC1-Prod-LB-01 | For SNMP sysDescr | `tmsh list sys snmp sys-location` |
| **SNMP MIBs Enabled** | F5-BIGIP-SYSTEM-MIB, F5-BIGIP-COMMON-MIB | Standard F5 MIBs | Defaults | Allow monitoring of LTM health | `tmsh list sys snmp` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] SNMP v3 preferred (or v2c with custom community)
- [ ] Trap destinations are reachable
- [ ] Contact and Location fields populated
- [ ] Standard F5 MIBs enabled

---

### 5.4 Remote Syslog Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Syslog Server 1 (Primary)** | SIEM or centralized syslog server IP:port | FQDN or IP:514 | syslog1.corp.com:514 | Send all logs to central system | `tmsh list sys syslog remote-servers` |
| **Syslog Server 2 (Secondary)** | Backup syslog server | FQDN or IP:514 | syslog2.corp.com:514 | Redundancy | `tmsh list sys syslog remote-servers` |
| **Syslog Protocol** | TCP (or UDP per SIEM requirement) | TCP / UDP | TCP | TCP is more reliable | `tmsh list sys syslog remote-servers` |
| **LTM Log Level** | Notice or higher | Debug / Info / Notice / Warning / Error | Notice | Don't over-log; filter noise | `tmsh list sys event-processing` |
| **ASM/WAF Log Forwarding** | All to syslog (if ASM module enabled) | All / Selected | All | Security events to SIEM | `tmsh list security log publisher` |
| **Local Log Retention** | 7 days minimum | 3–30 days | 7 days | Keep local backups temporarily | `tmsh list sys log-config` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Remote syslog servers configured (at least 1)
- [ ] Syslog servers are reachable
- [ ] Log level appropriate (not debug in production)
- [ ] Security logs being forwarded to SIEM

---

## Section 6: LTM Profiles – Global Traffic Behavior

**Jira Ticket:** `EPIC: Standard LTM Profiles – TCP, HTTP, SSL, Persistence, Monitors`  
**Owner:** Application Delivery / LTM Engineering  
**Scope:** Cluster-wide (shared profiles)  
**Validation Strategy:** Profile inventory, attribute comparison

### 6.1 TCP Profiles (Client-Side & Server-Side)

**Jira Sub-Ticket:** `STORY: Standard TCP Profiles (Client & Server-Side)`

#### 6.1.1 TCP Client Profile

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Name** | `tcp_client_std` (or per environment) | Descriptive, no spaces | tcp_client_std, tcp_client_app_web | Inherited by virtual servers | `tmsh list ltm profile tcp tcp_client_std` |
| **Parent Profile** | tcp (default) | tcp | tcp | Inherits from F5 default | Auto |
| **Idle Timeout** | 300 seconds (5 min) | 60–600 seconds | 300 | Close idle connections; tuning per app | `tmsh list ltm profile tcp idle-timeout` |
| **Time-Wait Recycle** | Enabled | enabled / disabled | enabled | Allow rapid connection reuse | `tmsh list ltm profile tcp time-wait-recycle` |
| **Nagle Algorithm** | Disabled | disabled / enabled | disabled | Reduce latency (better for HTTP) | `tmsh list ltm profile tcp nagle` |
| **TCP Timestamps** | Enabled | enabled / disabled | enabled | Helps with retransmission, RTT calculation | `tmsh list ltm profile tcp tcp-timestamps` |
| **Selective ACK (SACK)** | Enabled | enabled / disabled | enabled | Improves recovery from packet loss | `tmsh list ltm profile tcp selective-ack` |
| **Delayed ACK** | Disabled | enabled / disabled | disabled | Reduce ACK overhead (or per app need) | `tmsh list ltm profile tcp delay-ack` |
| **Window Scaling** | Enabled | enabled / disabled | enabled | Support for large TCP windows | `tmsh list ltm profile tcp window-scale` |
| **Congestion Control** | newreno (or cubic) | newreno / cubic / bbr | newreno | Standard congestion control algorithm | `tmsh list ltm profile tcp congestion-control` |

**TCP Server Profile** (similar, but tuned for backend):
- Similar parameters as client, but optimized for server-side behavior
- Can be more aggressive (longer timeouts if backend is slow)

**Validation Checklist (Jira Sub-Tasks):**
- [ ] `tcp_client_std` and `tcp_server_std` profiles exist
- [ ] Idle timeout reasonable (300 sec default)
- [ ] Nagle disabled (for HTTP)
- [ ] No custom TCP profiles attached unless exception documented

---

#### 6.1.2 HTTP Profile

**Jira Sub-Ticket:** `STORY: Standard HTTP Profile`

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Name** | `http_std` (or per environment) | Descriptive | http_std, http_app_web | Used by most HTTPS/HTTP VIPs | `tmsh list ltm profile http http_std` |
| **Parent Profile** | http (default) | http | http | Base HTTP behavior | Auto |
| **Insert X-Forwarded-For** | Enabled | enabled / disabled | enabled | Let backend see client IP | `tmsh list ltm profile http insert-x-forwarded-for` |
| **X-Forwarded-For Header** | X-Forwarded-For | Header name | X-Forwarded-For | Standard header name | Auto |
| **OneConnect** | Enabled (if HTTP/1.1) | enabled / disabled | enabled | Connection multiplexing (HTTP/1.1) | `tmsh list ltm profile http oneconnect` |
| **OneConnect Transform** | Disabled (unless needed) | enabled / disabled | disabled | Only enable if app needs custom transform | `tmsh list ltm profile http oneconnect-transform-requests` |
| **Chunked Encoding** | Enabled | enabled / disabled | enabled | Allow chunked transfer encoding | `tmsh list ltm profile http chunked-encoding` |
| **Compression** | Disabled (or Enabled per policy) | enabled / disabled | disabled | Gzip compression; enable if needed | `tmsh list ltm profile http compression` |
| **HTTP Response Timeout** | 0 (disabled) or 300 sec | 0–600 | 0 | Don't timeout backend responses | `tmsh list ltm profile http response-timeout` |
| **HTTP Request Timeout** | 0 (disabled) or 60 sec | 0–300 | 0 | Don't timeout client requests | `tmsh list ltm profile http request-timeout` |
| **Server Hostname** | (inherit from Host header) | Rewrite / Preserve | (preserve) | Don't rewrite Server header | `tmsh list ltm profile http server-agent-name` |
| **Header Insertion** | Standard set (e.g., Via, Connection) | Selective | Via, Connection | Header management per RFC | `tmsh list ltm profile http` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] `http_std` profile exists
- [ ] X-Forwarded-For insertion enabled
- [ ] OneConnect enabled for HTTP/1.1
- [ ] Compression policy matches standard
- [ ] No unnecessary header rewriting

---

#### 6.1.3 SSL/TLS Profiles (Client-Side & Server-Side)

**Jira Sub-Ticket:** `STORY: Standard SSL Profiles (Client & Server-Side) & Cipher Suite`

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Client SSL Profile Name** | `clientssl_std` (or per app) | Descriptive | clientssl_std, clientssl_web_app | Decrypt client traffic | `tmsh list ltm profile client-ssl clientssl_std` |
| **Server SSL Profile Name** | `serverssl_std` (or per app) | Descriptive | serverssl_std, serverssl_app_backend | Encrypt to backend | `tmsh list ltm profile server-ssl serverssl_std` |
| **Min TLS Version (Client)** | TLSv1.2 | TLSv1.2 / TLSv1.3 | TLSv1.2 | Disable TLS 1.0, 1.1 | `tmsh list ltm profile client-ssl min-version` |
| **Max TLS Version (Client)** | TLSv1.3 | TLSv1.2 / TLSv1.3 | TLSv1.3 | Allow latest | `tmsh list ltm profile client-ssl max-version` |
| **Min TLS Version (Server)** | TLSv1.2 | TLSv1.0 / TLSv1.1 / TLSv1.2 / TLSv1.3 | TLSv1.2 | Per backend requirements | `tmsh list ltm profile server-ssl min-version` |
| **Cipher Suite (Client)** | ECDHE-RSA, DHE, no weak | Modern ciphers only | `DEFAULT:!EXPORT:!MD5:!RC4:!aNULL` | Strong forward secrecy | `tmsh list ltm profile client-ssl ciphers` |
| **Cipher Suite (Server)** | Match backend requirements | Dependent on backend | Same as client (or more permissive) | If backend requires weak ciphers, document | `tmsh list ltm profile server-ssl ciphers` |
| **Cipher Order** | F5 preferred order | f5-default or client-preferred | f5-default | F5 chooses strongest cipher | `tmsh list ltm profile client-ssl cipher-ordering` |
| **Secure Renegotiation (Client)** | Require | require / request / none | require | Prevent renegotiation attacks | `tmsh list ltm profile client-ssl secure-renegotiation` |
| **Secure Renegotiation (Server)** | Require Strict | require-strict / require / request | require-strict | Strict on backend for security | `tmsh list ltm profile server-ssl secure-renegotiation` |
| **SSL Session ID Timeout (Client)** | 300 seconds | 60–3600 seconds | 300 | Cache session IDs for reuse | `tmsh list ltm profile client-ssl session-timeout` |
| **SSL Session Resumption** | Enabled | enabled / disabled | enabled | Allow session ID or ticket resumption | `tmsh list ltm profile client-ssl session-resumption` |
| **OCSP Stapling (if needed)** | Disabled (unless required) | enabled / disabled | disabled | Add OCSP response to handshake | `tmsh list ltm profile client-ssl ocsp-stapling-params` |
| **Certificate Chain** | Complete chain in certificate | All intermediate certs | Single cert or chain | Prevent client certificate validation failures | `tmsh list ltm profile client-ssl cert-key-chain` |

**Standard Cipher Suite (TLS 1.2+):**
```
DEFAULT:!EXPORT:!MD5:!RC4:!aNULL:!eNULL:!PSK:!DHE
```

Or per OWASP/Mozilla recommendations:
```
ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
```

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Client SSL profile with TLS 1.2+ min
- [ ] Cipher suite uses strong algorithms (no RC4, MD5, DES)
- [ ] Forward secrecy enabled (ECDHE/DHE)
- [ ] Session resumption enabled
- [ ] Certificate chain complete

---

### 6.2 Persistence Profiles

**Jira Sub-Ticket:** `STORY: Standard Persistence Profiles (Cookie, Source Address, SSL)`

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Cookie Persistence Profile Name** | `persist_cookie_std` | Descriptive | persist_cookie_std, persist_jsessionid | HTTP cookie-based affinity | `tmsh list ltm persistence cookie` |
| **Cookie Name** | `BIGipServer` (or app-specific) | Custom cookie name | BIGipServer, f5_sessionid | Unique per app; don't expose backend | `tmsh list ltm persistence cookie cookie-name` |
| **Cookie Method** | Insert (vs. HTTP_ONLY or URL rewrite) | insert / httponly / rewrite | insert | Less intrusive than rewrite | `tmsh list ltm persistence cookie method` |
| **Cookie Path** | `/` (root) | Path | / | Apply to all URLs on domain | `tmsh list ltm persistence cookie cookie-path` |
| **Cookie Expiration** | 86400 seconds (1 day) or session | Seconds or session-only | session | Per app requirement; session-only is safer | `tmsh list ltm persistence cookie expiration` |
| **Source Address Persistence Name** | `persist_source_std` | Descriptive | persist_source_std | IP-based affinity (L3) | `tmsh list ltm persistence source-addr` |
| **Source Persistence Timeout** | 180 seconds (3 min) | 60–3600 seconds | 180 | Auto-expire affinity after inactivity | `tmsh list ltm persistence source-addr timeout` |
| **Source Persistence Mask (IPv4)** | 255.255.255.255 (/32 = per IP) | /32, /24, /16 | 255.255.255.255 | Sticky per client IP | `tmsh list ltm persistence source-addr netmask` |
| **Source Persistence Mask (IPv6)** | ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff | /128 or /96 etc. | /128 (per IPv6 address) | Sticky per IPv6 address | `tmsh list ltm persistence source-addr ipv6-netmask` |
| **SSL Session ID Persistence Name** | `persist_ssl_std` | Descriptive | persist_ssl_std | SSL session affinity (if needed) | `tmsh list ltm persistence ssl` |
| **SSL Session ID Timeout** | 300 seconds | 60–3600 seconds | 300 | Timeout for SSL affinity | `tmsh list ltm persistence ssl timeout` |

**When to Use Each:**
- **Cookie:** Best for web apps (stateful sessions, shopping carts)
- **Source Address:** For non-sticky load balancing; use when app is stateless
- **SSL Session ID:** Rarely needed; only if backend requires SSL session stickiness

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Cookie persistence profile exists with standard settings
- [ ] Source address persistence profile exists
- [ ] Cookie name is not revealing backend details
- [ ] Timeout values are reasonable

---

### 6.3 Health Monitor Profiles

**Jira Sub-Ticket:** `STORY: Standard Health Monitors (TCP, HTTP, HTTPS, ICMP, etc.)`

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **TCP Monitor Name** | `monitor_tcp_std` | Descriptive | monitor_tcp_std, monitor_tcp_app | Basic TCP connectivity | `tmsh list ltm monitor tcp monitor_tcp_std` |
| **TCP Interval** | 10 seconds | 5–60 seconds | 10 | How often to check (balance: frequency vs. load) | `tmsh list ltm monitor tcp interval` |
| **TCP Timeout** | 31 seconds | ≥ 3×Interval + 1 | 31 (= 10×3+1) | Wait time for response; F5 standard formula | `tmsh list ltm monitor tcp timeout` |
| **HTTP Monitor Name** | `monitor_http_std` | Descriptive | monitor_http_std, monitor_http_app | Application-level HTTP health | `tmsh list ltm monitor http monitor_http_std` |
| **HTTP Send String** | `GET /health HTTP/1.1\r\nHost: example.com\r\n\r\n` | Custom HTTP request | GET /health HTTP/1.1 | Application-specific health check endpoint | `tmsh list ltm monitor http send` |
| **HTTP Receive String** | `HTTP/1.1 200 OK` or `HTTP/1.1 2..` | Expected response | HTTP/1.1 200 | Match success response (200, 201, 202, etc.) | `tmsh list ltm monitor http recv` |
| **HTTP Interval** | 10 seconds | 5–60 seconds | 10 | How often to check | `tmsh list ltm monitor http interval` |
| **HTTP Timeout** | 31 seconds | ≥ 3×Interval + 1 | 31 | Wait time for HTTP response | `tmsh list ltm monitor http timeout` |
| **HTTPS Monitor Name** | `monitor_https_std` (if SSL monitored) | Descriptive | monitor_https_std | HTTPS health check (if backend is HTTPS) | `tmsh list ltm monitor https monitor_https_std` |
| **HTTPS Send String** | Same as HTTP | Custom HTTPS request | GET /health HTTP/1.1 | HTTPS version of HTTP monitor | `tmsh list ltm monitor https send` |
| **HTTPS Receive String** | Same as HTTP | Expected response | HTTP/1.1 200 | Match success response | `tmsh list ltm monitor https recv` |
| **ICMP Monitor Name** | `monitor_icmp_std` (if ICMP used) | Descriptive | monitor_icmp_std | Ping-based health (basic) | `tmsh list ltm monitor icmp monitor_icmp_std` |
| **ICMP Interval** | 10 seconds | 5–60 seconds | 10 | How often to ping | `tmsh list ltm monitor icmp interval` |
| **ICMP Timeout** | 16 seconds | ≥ 3×Interval + 1 | 16 | Wait time for ICMP reply | `tmsh list ltm monitor icmp timeout` |
| **Up Threshold** | 1 (by default) | 1–3 | 1 | Consecutive successes to mark UP | `tmsh list ltm monitor up-threshold` |
| **Down Threshold** | 3 (by default) | 1–5 | 3 | Consecutive failures to mark DOWN | `tmsh list ltm monitor down-threshold` |

**Monitor Deployment Strategy:**
1. **Web apps:** Use HTTP/HTTPS monitors with meaningful endpoints
2. **Database servers:** Use TCP port monitors (e.g., 3306 for MySQL, 5432 for Postgres)
3. **Non-web services:** Use TCP or ICMP ping monitors
4. **Do NOT use:** "none" monitor for production (will not remove failed nodes)

**Example Monitor Attachment:**
```
ltm pool my_app_pool {
    members {
        app-server-1:8080 { }
        app-server-2:8080 { }
    }
    monitor /Common/monitor_http_std
}
```

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Monitor profiles defined for TCP, HTTP, HTTPS per requirement
- [ ] Timeout ≥ 3×Interval + 1
- [ ] HTTP monitor has meaningful endpoint (e.g., /health)
- [ ] HTTP monitor matches expected response (200-level)
- [ ] No "none" monitors on production pools

---

## Section 7: Virtual Servers (Per-FQDN Application Entry Points)

**Jira Ticket:** `EPIC: Virtual Server Standards (Per Service / FQDN)`  
**Owner:** Application Delivery / LTM Engineering  
**Scope:** Per application / FQDN  
**Validation Strategy:** Per-VIP configuration audit & profile compliance

### 7.1 Virtual Server General Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **VIP Name** | `vs_{env}_{app}_{port}` | Lowercase, descriptive, max 32 chars | vs_prod_web_443, vs_prod_api_80 | Consistent naming for automation | `tmsh list ltm virtual` |
| **VIP Type** | Standard | Standard / Performance / Forwarding | Standard | Most common; uses full proxy architecture | `tmsh list ltm virtual type` |
| **Destination IP (Virtual IP)** | Public or internal IP | RFC 1918 or routable IP | 10.50.100.10 | The "front-end" IP clients connect to | `tmsh list ltm virtual destination` |
| **Destination Port** | 80 (HTTP) / 443 (HTTPS) / custom | 1–65535 | 443 | Standard ports preferred | `tmsh list ltm virtual destination` |
| **Protocol** | tcp | tcp / udp | tcp | TCP for HTTP/HTTPS; UDP for DNS, etc. | `tmsh list ltm virtual protocol` |
| **Address Translation** | Enabled (default) | enabled / disabled | enabled | Translate VIP to pool member IP (standard) | `tmsh list ltm virtual address-translation` |
| **Port Translation** | Enabled (default) | enabled / disabled | enabled | Translate port if needed | `tmsh list ltm virtual port-translation` |
| **Enabled/Disabled State** | Enabled | enabled / disabled | enabled | Policy: disable VIP only for maintenance | `tmsh list ltm virtual disabled` |
| **Connection Limit** | 0 (unlimited) or per app policy | 0 (unlimited) or specific count | 10000 (example) | Protect against resource exhaustion | `tmsh list ltm virtual connection-limit` |
| **Connection Rate Limit** | 0 (unlimited) or per policy | 0 or specific rate | 1000 (connections/sec) | Prevent connection flood attacks | `tmsh list ltm virtual connection-rate-limit` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] VIP name follows standard
- [ ] Destination IP and port are correct
- [ ] Address & port translation enabled (standard)
- [ ] Enabled state matches policy

---

### 7.2 Virtual Server Profiles (Client-Side & Server-Side)

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Client TCP Profile** | `tcp_client_std` (or custom per app) | From approved profile list | tcp_client_std | Attached to client-side connection | `tmsh list ltm virtual profiles` |
| **Server TCP Profile** | `tcp_server_std` (or custom per app) | From approved profile list | tcp_server_std | Attached to server-side connection | `tmsh list ltm virtual profiles` |
| **HTTP Profile** | `http_std` (if HTTP/HTTPS) | From approved HTTP profile list | http_std | For HTTP/HTTPS VIPs only | `tmsh list ltm virtual profiles` |
| **Client SSL Profile** | `clientssl_std` (if HTTPS) | From approved SSL profile list | clientssl_std | For HTTPS VIPs (decrypt client) | `tmsh list ltm virtual profiles` |
| **Server SSL Profile** | `serverssl_std` or none (if backend is HTTP) | From approved SSL profile list | serverssl_std or none | For HTTPS to backend; none if HTTP backend | `tmsh list ltm virtual profiles` |
| **Persistence Profile** | `persist_cookie_std` or `persist_source_std` or none | From approved persistence list | persist_cookie_std | None if app is stateless | `tmsh list ltm virtual profiles` |
| **OneConnect Profile** | (inherited from HTTP profile) | N/A | N/A | Multiplexes HTTP/1.1 connections | Auto |
| **Compression Profile** | (Conditional) | Enabled or via HTTP profile | Default (no compression) | Only if data size warrants compression | `tmsh list ltm virtual profiles` |

**Only Approved Profiles:**
- All profiles must be from a pre-approved, standardized list (e.g., `tcp_client_std`, `http_std`, `clientssl_std`)
- Custom profiles require exception documentation and security review
- Never attach random or unnamed profiles

**Validation Checklist (Jira Sub-Tasks):**
- [ ] All attached profiles are on approved list
- [ ] TCP profiles match (client & server)
- [ ] HTTP profile attached for HTTP/HTTPS VIPs
- [ ] SSL profiles correct for HTTPS (client) and backend (server)
- [ ] Persistence profile appropriate for app

---

### 7.3 Virtual Server Traffic Management

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **SNAT Setting** | Per app policy (Automap, None, or Pool) | None / Automap / SNAT Pool | Automap | Enables backend servers to respond correctly | `tmsh list ltm virtual source-nat` |
| **Pool Assignment** | Explicit pool name | `pool_{app}_{env}` format | pool_web_prod | All traffic routed to pool | `tmsh list ltm virtual pool` |
| **Default Pool Member Port** | Match pool member port | 80, 8080, 3000, etc. | 80 | VIP port may differ from backend port | `tmsh list ltm virtual` |
| **Fallback Pool** | Not recommended (or explicit if needed) | None (None) or explicit | (none) | No silent failure; alert if no pool available | `tmsh list ltm virtual fallback-pool` |
| **Default Monitor** | Inherited from pool | (none) | (inherited) | Pool's monitor takes precedence | `tmsh list ltm virtual` |

**SNAT Guidance:**
- **Use Automap:** Backend servers not on same subnet as pool self IP, or when return path is uncertain
- **Use SNAT Pool:** Multiple VIPs need specific source IPs (e.g., for logging or firewall rules)
- **Use None:** Backend servers on same subnet and can route back to client directly (rare)

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Pool is assigned
- [ ] SNAT policy defined (Automap / None / Pool)
- [ ] Fallback pool is explicit or None (no ambiguity)
- [ ] Pool members can reach the backend service

---

### 7.4 Virtual Server Access & Security

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Allowed VLANs / Traffic Enabled** | Explicit list (not all) | Specific VLANs or "all" | ext_client_vlan | Restrict to required client VLANs | `tmsh list ltm virtual vlans-enabled` |
| **VLANs Disabled** | Explicit list (not needed) | Specific VLANs | Internal VLANs (if not needed) | Prevent internal clients reaching VIP | `tmsh list ltm virtual vlans-disabled` |
| **Route Domain** | 0 (/Common) unless multi-tenant | 0 / custom route domain | 0 | Default for single-tenant | `tmsh list ltm virtual route-domain` |
| **iRules Attached** | Explicit approved iRules | From approved iRule list | ir_redirect_http_to_https (example) | Only approved, versioned iRules | `tmsh list ltm virtual rules` |
| **Policies Attached** | Explicit LTM policies | From approved policy list | policy_header_rewrite (example) | Only approved policies | `tmsh list ltm virtual policies` |
| **Traffic Filtering (Source Address)** | Allow list or none | CIDR ranges or empty (allow all) | 10.0.0.0/8, 192.168.0.0/16 | Whitelist trusted client subnets if needed | Custom via iRule or firewall |
| **TCP Settings** | Via TCP profile (not custom) | Per TCP profile settings | Inherited from `tcp_client_std` | Don't override TCP at VIP level | `tmsh list ltm virtual` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] VLANs enabled restricted to client-facing VLANs
- [ ] Only approved iRules attached
- [ ] Only approved LTM policies attached
- [ ] iRules and policies versioned in Git
- [ ] No custom traffic filtering hardcoded in VIP

---

## Section 8: Pools & Pool Members

**Jira Ticket:** `EPIC: Pool & Pool Member Standards`  
**Owner:** Application Delivery  
**Scope:** Per application pool  
**Validation Strategy:** Pool configuration inventory & member health status

### 8.1 Pool Configuration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Pool Name** | `pool_{env}_{app}_{instance}` | Lowercase, descriptive, max 32 chars | pool_prod_web_01, pool_nonprod_api | Consistent naming for automation | `tmsh list ltm pool` |
| **Load Balancing Method** | Round-robin (default) or app-specific | round-robin / least-conn / ratio / weighted-least-conn | round-robin | Most common; tune per app | `tmsh list ltm pool load-balancing-mode` |
| **Priority Group Activation** | Disabled (unless multi-tier) | enabled / disabled | disabled | Standard for single-tier pools | `tmsh list ltm pool priority-group-activation` |
| **Minimum Active Members** | 1 (minimum) or per policy | 1–N | 1 | If fewer members UP, take corrective action | `tmsh list ltm pool min-active-members` |
| **Minimum Up Members Action** | failover (to standby pool) or rebalance | failover / rebalance / none | failover | Fallback if min members breached | `tmsh list ltm pool min-up-members-action` |
| **Slow Ramp Time** | 30 seconds (new member startup) | 0–300 seconds | 30 | Ramp up traffic for new members | `tmsh list ltm pool slow-ramp-time` |
| **Monitor Assignment** | Explicit monitor (e.g., `monitor_http_std`) | Approved monitor from list | monitor_http_std | Every pool needs at least 1 monitor | `tmsh list ltm pool monitors` |
| **Description** | Brief purpose or app identifier | Free-form text | "Backend web servers for production web app" | For documentation / ticket tracking | `tmsh list ltm pool description` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Pool name follows standard
- [ ] Load balancing method appropriate for app
- [ ] Monitor is assigned (not "none")
- [ ] Minimum active members policy defined

---

### 8.2 Pool Members (Backend Servers)

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Member Name / IP:Port** | `member_{hostname}_{port}` or IP:port | FQDN:port or IP:port | app-server-01:8080 or 10.1.1.100:8080 | Use FQDN if DNS available | `tmsh list ltm pool members` |
| **Member Address** | IP address (not hostname)** | IPv4 or IPv6 | 10.1.1.100 | IP address for routing | `tmsh list ltm pool members address` |
| **Member Port** | Application port | 80, 443, 8080, 3000, custom | 8080 | Port pool member listens on | `tmsh list ltm pool members port` |
| **Member State** | up / down (admin control) | up / down / disabled | up | "up" for active servers, "down" for maintenance | `tmsh list ltm pool members state` |
| **Member Ratio** | 1 (equal weight) or per app | 1–100 | 1 (equal) or 2 (twice as much traffic) | For weighted load balancing | `tmsh list ltm pool members ratio` |
| **Member Priority** | 0 (no priority) or per tier | 0–255 | 0 (equal) | For priority group activation | `tmsh list ltm pool members priority` |
| **Member Connection Limit** | 0 (unlimited) or per policy | 0 or specific count | 0 (unlimited) | Limit connections per member | `tmsh list ltm pool members connection-limit` |
| **Member Monitor Status** | up (healthy) / down (unhealthy) | up / down / disabled | up | Reflects actual health check result | `tmsh show ltm pool members` |
| **Member Session Limit** | 0 (unlimited) or per policy | 0 or specific count | 0 (unlimited) | Limit concurrent sessions per member | `tmsh list ltm pool members session-limit` |

**Best Practices:**
- Use IP addresses (not DNS names) for pool members; manage DNS externally
- Prefer hostnames for member naming (e.g., `app-server-01:8080`) for readability
- Don't mix static and dynamic pools; decide per app
- Monitor pool member health in real-time

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Member IPs are correct
- [ ] Member ports match application
- [ ] All members show health monitor status (up/down)
- [ ] No disabled members left in long-term
- [ ] Connection/session limits defined if needed

---

## Section 9: iRules & Local Traffic Policies

**Jira Ticket:** `EPIC: Custom Logic – iRules & Local Traffic Policies`  
**Owner:** Application Delivery / Development  
**Scope:** Cluster-wide (shared iRules & policies)  
**Validation Strategy:** iRule registry, version control, compliance audit

### 9.1 iRules Standards

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **iRule Name** | `ir_{purpose}_{environment}` (or versioned) | Lowercase, descriptive, max 32 chars | ir_redirect_http_to_https_prod, ir_log_security | Standard naming for inventory | `tmsh list ltm rule` |
| **iRule Naming Convention** | `ir_{purpose}_{app}_{version}` | Structured | ir_auth_check_app_v2 | Includes version for change management | Git repo naming |
| **iRule Source Control** | All iRules in Git repository | Git / GitLab / GitHub | f5-irules-repo / f5-rules/ | Single source of truth | `git log ltm/ir_*.tcl` |
| **iRule Documentation** | Comment header with purpose, author, version | Text comments | `# Purpose: Redirect HTTP to HTTPS\n# Author: NetOps\n# Version: 1.0` | Mandatory for production | Code comments |
| **iRule Testing** | Tested in non-prod before production | Automated or manual | Tested in dev, then staging, then prod | Change control workflow | Audit trail |
| **iRule Attachment** | Only approved iRules per VIP | Approved list per app | ir_redirect_http_to_https_prod, ir_add_security_headers_prod | Whitelist-based approach | `tmsh list ltm virtual rules` |
| **iRule Disable/Enable** | Documented reason if disabled | enabled / disabled | disabled (for troubleshooting) | Track why iRules are disabled | `tmsh list ltm rule disabled` |
| **Generic iRules Library** | Maintained shared iRules (security headers, logging, redirects) | E.g., ir_security_headers_std | ir_security_headers_std, ir_logging_std | Reuse across apps | Central iRule list |

**Approved Generic iRules (Examples):**
- `ir_security_headers_std` – Adds X-Frame-Options, X-Content-Type-Options, etc.
- `ir_redirect_http_https_std` – Redirects HTTP traffic to HTTPS
- `ir_add_custom_header_std` – Appends custom headers
- `ir_log_security_events_std` – Logs security events to syslog

**Validation Checklist (Jira Sub-Tasks):**
- [ ] All iRules tracked in Git with version control
- [ ] iRule name follows standard
- [ ] Only approved iRules attached to VIPs
- [ ] iRules documented (header comments)
- [ ] iRules tested before production deployment

---

### 9.2 Local Traffic Policies

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Policy Name** | `policy_{purpose}_{environment}` | Lowercase, descriptive, max 32 chars | policy_url_rewrite_prod, policy_header_insert_prod | Consistent naming | `tmsh list ltm policy` |
| **Policy Type** | L4 (if L4) / Policies (if L7) | L4 / Policies | Policies (for L7 rules) | Standard for HTTP/HTTPS | `tmsh list ltm policy type` |
| **Policy Rules** | Explicit conditions and actions | Ordered list | Rule 1: if URI contains /old redirect to /new | Clear, testable rules | `tmsh list ltm policy rules` |
| **Policy Attachment** | Only approved policies per VIP | Approved list | policy_url_rewrite_prod | Whitelist approach | `tmsh list ltm virtual policies` |
| **Policy Source Control** | Policies versioned in Git (export JSON/XML) | Git / Git Repo | f5-policies-repo / ltm/policies/ | Change management via Git | Export from BIG-IP |
| **Policy Testing** | Tested in staging before production | Manual or automated testing | Test URL rewrites in staging environment | Change control workflow | Test logs |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Policy names follow standard
- [ ] Policies versioned in Git
- [ ] Only approved policies attached to VIPs
- [ ] Policies tested in staging before production

---

## Section 10: Application Security (ASM/WAF, AFM, DDoS Protection)

**Jira Ticket:** `EPIC: Application & Network Security Standards (if ASM/AFM/DDoS Licensed)`  
**Owner:** Security Team  
**Scope:** Per security policy/VIP  
**Validation Strategy:** Security audit, threat profile validation

### 10.1 WAF/ASM Policy (if ASM module enabled)

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **ASM Policy Name** | `waf_policy_{app}_{environment}` | Descriptive, lowercase | waf_policy_web_app_prod, waf_policy_api_prod | Consistent naming | `tmsh list security asm policies` |
| **Policy Mode** | Blocking (production) or Transparent (testing) | transparent / blocking | blocking (prod) / transparent (dev/test) | Start transparent, move to blocking after validation | `tmsh list security asm policies mode` |
| **Signature Set** | Standard set (e.g., OWASP Top 10) | Updated regularly | OWASP 3.x signature set | Keep signatures current | `tmsh list security asm policies signature-sets` |
| **Logging Profile** | Send logs to SIEM | Remote syslog or HSL | Logs to security-siem-prod:514 | Track attacks and policy violations | `tmsh list security log-publisher` |
| **DDoS Profile (if applicable)** | Enabled if exposed to internet | enabled / disabled | enabled | Detect and mitigate volumetric attacks | `tmsh list security ddos-profile` |
| **Bot Detection** | Enabled if needed | enabled / disabled | disabled (unless known bot traffic) | Detect automated bot attacks | `tmsh list security bot-defense-policy` |
| **API Schema Validation (if API)** | Enabled for APIs | enabled / disabled | enabled (for REST APIs) | Validate API requests against schema | `tmsh list security asm policies` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] WAF/ASM policies exist for production VIPs
- [ ] Policies set to "blocking" in production
- [ ] Signature sets are current
- [ ] Logging to SIEM enabled
- [ ] DDoS protection enabled if internet-facing

---

### 10.2 Network Firewall (AFM) & Route Domain Policies

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **AFM Policy Name** | `firewall_policy_{vlan}_{environment}` | Descriptive | firewall_policy_client_vlan_prod | Network-level firewall rules | `tmsh list security firewall policy` |
| **AFM Rule Order** | Explicit allow rules first, deny last | Ordered rules | Allow TCP 443, then Deny all | Process rules in order; use "stop" action | `tmsh list security firewall policy rules` |
| **Default Action** | Reject (deny-all by default) | reject / drop | reject | Fail-secure: default deny | `tmsh list security firewall policy default-action` |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] AFM policies defined for VLANs (if AFM licensed)
- [ ] Firewall rules are explicit and ordered
- [ ] Default action is "reject" (deny-all)

---

## Section 11: Telemetry, Backup & Automation Hooks

**Jira Ticket:** `EPIC: Backups, Telemetry & Automation State`  
**Owner:** Infrastructure / Automation Team  
**Scope:** Per device  
**Validation Strategy:** Backup logs, telemetry endpoint status

### 11.1 Configuration Backups

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **UCS Backup Schedule** | Daily at 2 AM UTC (off-peak) | Daily / Weekly | 02:00 UTC daily | Complete device backup | Cron job / Scheduler |
| **UCS Backup Retention** | 7 days local + 30 days archive | 3–30 days | 7 days local, 30 days in S3 | Protect against ransomware | Backup logs |
| **UCS Backup Location** | Off-box (NFS or S3) | NFS / SMB / S3 / HTTP | /backup/bigip_ucs/ or s3://backups/f5/ | Don't rely on local storage | `tmsh list sys software image` |
| **SCF Export** | Optional (for critical configs) | Yes / No | Yes (for configs with sensitive data) | Smaller than UCS; config-only | `tmsh save sys config` |
| **Backup Encryption** | Enabled (if sensitive data) | enabled / disabled | enabled (for production) | Encrypt UCS backups | UCS encryption flag |
| **Backup Testing** | Quarterly restore test | Quarterly / Bi-annual | Tested Q1, Q2, Q3, Q4 | Verify backups are restorable | Test logs |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] UCS backups scheduled (daily)
- [ ] Backups stored off-box
- [ ] Backup retention policy enforced
- [ ] Backups are encrypted (production)
- [ ] Restore testing completed

---

### 11.2 Telemetry & Monitoring Integration

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Telemetry Streaming (if used)** | Enabled to central monitoring system | enabled / disabled | enabled → Prometheus / Datadog | Centralized metrics collection | `tmsh list sys telemetry` |
| **iControl LX Extensions** | AS3 / DO / TS / CFE (if declarative) | Various | AS3 (for app services), DO (for device config) | Modern automation framework | `tmsh list com extension` |
| **Custom Monitoring** | Via syslog or SNMP to Prometheus / Grafana | syslog / SNMP / HSL | Health check metrics → Prometheus | Real-time alerting | Monitoring agent logs |
| **Alert Destinations** | To SOC / incident management system | Syslog / email / Slack / PagerDuty | pool member down → alert | Proactive incident response | Alert logs |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Telemetry streaming configured (if using TS)
- [ ] Monitoring integration active (SNMP / syslog)
- [ ] Alerts configured for critical events
- [ ] Alert destinations validated

---

### 11.3 Automation State & Infrastructure-as-Code

| Parameter | Standard Value | Acceptable Range | Example | Notes | Validation Method |
|-----------|---|---|---|---|---|
| **Automation Framework** | Manual / Ansible-managed / AS3-managed / DO-managed | One per device | Ansible (for playbook-driven changes) | Determine how device is managed | Documentation / Git repos |
| **Ansible Playbook Repository** | Centralized Git repo | GitHub / GitLab / Bitbucket | f5-automation-playbooks / f5-ansible-roles | Version-controlled configs | `git clone f5-automation-playbooks` |
| **AS3 Declarations** | Optional (if using AS3 for app services) | Yes / No | Yes (for modern automation) | Declarative app services | `curl /mgmt/shared/appsvcs/declare` |
| **DO Declarations** | Optional (if using DO for system config) | Yes / No | Yes (for declarative device config) | System settings as JSON | `curl /mgmt/shared/declarative-onboarding` |
| **Change Tracking** | All changes logged (manual or automated) | Git / Change Control System | Every change in Git + Jira ticket | Audit trail for compliance | Git log + Jira |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Automation state documented (manual vs. playbook vs. declarative)
- [ ] If playbook-driven, Ansible repo is current
- [ ] If declarative (AS3/DO), declarations are in Git
- [ ] All changes tracked (Git + ticket)

---

## Section 12: Configuration Validation & Compliance

**Jira Ticket:** `EPIC: Configuration Validation & Compliance Audit`  
**Owner:** Infrastructure / Compliance  
**Scope:** Per device  
**Validation Strategy:** Automated validation scripts, compliance scoring

### 12.1 CIS Benchmark Compliance

| Check | Standard Value | Acceptable | Notes | Validation |
|---|---|---|---|---|
| **SSH Restricted** | Restricted to known IPs | allow list exists | SSH allowed only from SOC | `tmsh list sys sshd allow` |
| **HTTP GUI Disabled** | Disabled | disabled | HTTPS only (port 443) | `tmsh list sys httpd port` |
| **GUI Idle Timeout** | 900 seconds (15 min) | 300–1800 sec | Auto-logout on inactivity | `tmsh list sys httpd auth-pam-idle-timeout` |
| **SSH Idle Timeout** | 900 seconds (15 min) | 300–1800 sec | Auto-logout SSH sessions | `tmsh list sys sshd inactivity-timeout` |
| **Password Policy Enforced** | See Section 2.4 standards | All criteria met | 10 chars, complexity, 90-day expiry | `tmsh list auth password-policy` |
| **Root Login Disabled** | true | true | `systemauth.disablerootlogin = true` | `tmsh list sys db systemauth.disablerootlogin` |
| **Admin User Changed** | Not "admin" | Custom name | Rename default admin account | `tmsh list auth user` |
| **Syslog Configured** | Remote syslog to SIEM | At least 1 server | Send logs for audit | `tmsh list sys syslog remote-servers` |
| **NTP Configured** | At least 2 NTP servers | 2+ servers | Time synchronization critical | `tmsh list sys ntp` |
| **TLS 1.2+ Only** | TLSv1.2 minimum | TLSv1.2 or 1.3 | Disable TLS 1.0, 1.1 | `tmsh list sys httpd ssl-protocol` |
| **Port Lockdown on Self IPs** | Allow None or Allow Custom | Whitelist only | Restrict management services | `tmsh list net self allow-service` |
| **Certificates Valid** | No expired certs | All current | Check cert expiration | `tmsh run sys crypto check-cert` |

**CIS Compliance Scoring:**
- Calculate percentage of checks passed (e.g., 11/15 = 73%)
- Flag any failed checks as "findings" for remediation
- Re-audit after remediation to verify compliance

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Run CIS compliance check script on all devices
- [ ] Document findings per device
- [ ] Create remediation tickets for failed checks
- [ ] Re-validate after remediation

---

### 12.2 Configuration Consistency Audit (Phase 2 Prep)

| Check | Expected | Actual | Status | Notes |
|---|---|---|---|---|
| **System Version** | v15.1.5 (prod standard) | {actual} | Pass/Fail | Must match environment standard |
| **DNS Servers** | 8.8.8.8, 8.8.4.4 | {actual} | Pass/Fail | Must be in approved list |
| **NTP Servers** | ntp1.corp.com, ntp2.corp.com | {actual} | Pass/Fail | Must sync to corporate NTP |
| **Standard TCP Profile** | `tcp_client_std` attached | {actual} | Pass/Fail | Profile must exist and be used |
| **Standard HTTP Profile** | `http_std` attached | {actual} | Pass/Fail | Profile must exist and be used |
| **Standard SSL Profiles** | `clientssl_std`, `serverssl_std` | {actual} | Pass/Fail | Profiles must exist |
| **Monitor Assignment** | All pools have monitor | {actual} | Pass/Fail | No "none" monitors |
| **SNAT Policy** | Automap or explicit pool | {actual} | Pass/Fail | SNAT must be defined |
| **iRules from Approved List** | All iRules on whitelist | {actual} | Pass/Fail | Only approved iRules |
| **Policies from Approved List** | All policies on whitelist | {actual} | Pass/Fail | Only approved policies |

**Validation Checklist (Jira Sub-Tasks):**
- [ ] Collect configuration from all 300 devices
- [ ] Compare against golden standard template
- [ ] Identify deviations per device
- [ ] Create exception list for documented deviations
- [ ] Generate compliance report per device

---

## Phase 1 Deliverables Summary

### Document Outputs:
1. **Jira Epic / Story Structure** (This Document)
   - 12 major sections = 12 parent Stories/Epics
   - Each section contains parameter table + validation checklist

2. **Standard Configuration YAML/JSON Template**
   - Machine-readable version of above
   - Used later for Phase 2 validation scripts

3. **Parameter Master List (CSV)**
   - All parameters across all sections
   - Object, Parameter, Standard Value, Acceptable Range

### Jira Ticket Creation (Manual or Automated):

**Epic:**
```
Title: F5 BIG-IP Configuration Standardization – Phase 1 & 2
Description: Establish and enforce standard configuration across all 300 F5 load balancers
Status: In Progress
```

**Stories (one per section):**
```
Story 1: Platform & System Baseline Standards
Story 2: Management Plane Security & Access Control Standards
Story 3: Network – VLANs, Self IPs & Routing Standards
Story 4: High Availability & Device Service Cluster Standards
Story 5: System Services – DNS, NTP, SNMP & Syslog Standards
Story 6: Standard LTM Profiles (TCP, HTTP, SSL, Persistence, Monitors)
Story 7: Virtual Server Standards (Per Service / FQDN)
Story 8: Pool & Pool Member Standards
Story 9: Custom Logic – iRules & Local Traffic Policies
Story 10: Application & Network Security Standards (ASM/WAF, AFM, DDoS)
Story 11: Backups, Telemetry & Automation State
Story 12: Configuration Validation & Compliance Audit
```

**Each Story should include:**
- Description: Purpose of the standard section
- Acceptance Criteria: List of parameters from the table above
- Definition of Done: All parameters validated across sample devices
- Sub-Tasks: Per-parameter validation checkboxes

---

## Next Steps (Phase 2):

Once Phase 1 is complete:

1. **Export all parameters to YAML/JSON** – Single source of truth for automation
2. **Build config collector** – Python script to pull config from all 300 devices
3. **Build validator** – Compare collected config against standard
4. **Generate reports** – Per-device deviations, compliance score, exception list
5. **Integrate with Jira** – Auto-create tickets for deviations

---

**End of Phase 1 Document**

---

## Quick Reference – Validation Commands

```bash
# Check version
tmsh show sys version

# Check system global settings
tmsh list sys global-settings

# Check management access restrictions
tmsh list sys httpd allow
tmsh list sys sshd allow

# Check password policy
tmsh list auth password-policy all-properties

# Check profiles
tmsh list ltm profile tcp
tmsh list ltm profile http
tmsh list ltm profile client-ssl

# Check all virtual servers
tmsh list ltm virtual

# Check all pools
tmsh list ltm pool

# Check HA status
tmsh show cm failover-status
tmsh show cm device-group sync-status

# Check NTP status
tmsh show sys ntp

# Check syslog configuration
tmsh list sys syslog remote-servers

# Check SNMP
tmsh list sys snmp

# Check all monitors
tmsh list ltm monitor
```

