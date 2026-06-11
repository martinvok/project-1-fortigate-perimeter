# Project 1 - FortiGate Perimeter Security Lab
**Home Lab | Martin Vo | June 2026**

Enterprise perimeter firewall lab using FortiGate 60F, Cisco Catalyst 2960G, and MikroTik. Zone-based segmentation, default deny policies, and attack simulation with log evidence.


---

## What I Built

A simulated enterprise perimeter security environment using a FortiGate 60F as the perimeter next-generation firewall. The FortiGate sits between a MikroTik router acting as the simulated ISP/internet and an internal network segmented into Corporate and DMZ zones via a Cisco Catalyst 2960G switch. A Windows PC connected to the FortiGate physical DMZ port simulates a publicly accessible server.

The lab demonstrates zone-based firewalling, VLAN segmentation, default deny policy architecture, and attack simulation - proving the security policies work by generating real FortiGate log evidence of blocked attempts.

---

## Why It Matters

Most engineers can configure a firewall. Fewer can explain *why* the design decisions were made and prove the security controls actually work under test conditions.

This lab replicates the three-zone perimeter architecture used in enterprise environments worldwide - WAN, Corporate LAN, and DMZ. The key principle is default deny: nothing crosses a zone boundary unless an explicit policy allows it. This is fundamentally different from a basic router or home firewall which typically allows all outbound traffic by default.

The attack simulation section proves the policies are enforced, not just configured - every blocked attempt generates a timestamped log entry showing source, destination, policy, and action. This is the same evidence a SOC team would review during an incident investigation.

---

## Lab Hardware

| Device | Model | Role |
|--------|-------|------|
| Firewall | FortiGate 60F (FortiOS 7.2.12) | Perimeter NGFW - HQ |
| Switch | Cisco Catalyst 2960G 24-port | Internal access switch |
| Simulated ISP | MikroTik Router | ISP/internet simulation |
| Corporate user | MacBook Pro | VLAN 10 end device |
| DMZ server | Windows PC | Simulated DMZ web server |

> **Note:** FortiGate 60F is running without a FortiGuard license. Web filtering, application control, IPS signatures, and antivirus cloud features are unavailable. All core features used in this lab - zones, VLANs, firewall policies, NAT, logging, routing - work fully without a license.

---

## Topology

```
[HFC Modem]
     |
[MikroTik Router]
Simulated ISP/internet
Port 7 (203.0.113.1/30) -------- FortiGate WAN1 (203.0.113.2/30)
                                         |
                                 [FortiGate 60F]
                                 MV-FortiGate-60F
                                 Perimeter NGFW
                                    /       \
                              LAN port    DMZ port
                                 |             |
                         [Cisco 2960G]    [Windows PC]
                          SW-HQ-LAB      172.16.20.10
                         Trunk Gi0/24    Static IP
                              |
                    Gi0/1          Gi0/2
                  VLAN 10         VLAN 20 (unused)
               172.16.10.0/24
               Corporate LAN
               MacBook DHCP
```

> **Design note:** The FortiGate physical DMZ port is used for the DMZ zone rather than a VLAN sub-interface on the 2960G switch. VLAN 20 was initially configured on the switch but was replaced with the physical DMZ port to provide true physical separation between the corporate LAN and DMZ - the more architecturally accurate enterprise design. VLAN 20 on the switch remains configured but unused.

---

## IP Addressing

| Device | Interface | IP Address | Subnet | Purpose |
|--------|-----------|------------|--------|---------|
| MikroTik | Port 7 (ether7) | 203.0.113.1 | /30 | Simulated ISP gateway |
| FortiGate | WAN1 | 203.0.113.2 | /30 | WAN - faces MikroTik |
| FortiGate | VLAN10_Internal | 172.16.10.1 | /24 | Corporate LAN gateway |
| FortiGate | Physical DMZ port | 172.16.20.1 | /24 | DMZ gateway |
| MacBook | Gi0/1 on 2960G (DHCP) | 172.16.10.100 | /24 | Corporate user |
| Windows PC | FortiGate DMZ port (static) | 172.16.20.10 | /24 | Simulated DMZ server |

---

## VLAN and Zone Design

| VLAN | Name | Subnet | Zone | Interface |
|------|------|--------|------|-----------|
| 10 | Corporate | 172.16.10.0/24 | ZONE_CORPORATE | VLAN10_Internal sub-interface |
| - | DMZ | 172.16.20.0/24 | ZONE_DMZ | Physical DMZ port |

---

## Firewall Policy Design

| Policy Name | From | To | Service | Action | NAT |
|------------|------|----|---------|--------|-----|
| CORP_To_WAN | ZONE_CORPORATE | WAN1 | ALL | Allow | Yes |
| DMZ-to-WAN | ZONE_DMZ | WAN1 | ALL | Allow | Yes |
| WAN_To_DMZ | WAN1 | ZONE_DMZ | HTTP, HTTPS | Allow | No |
| CORP_To_DMZ | ZONE_CORPORATE | ZONE_DMZ | ALL | Allow | No |
| DMZ_To_CORP_Deny | ZONE_DMZ | ZONE_CORPORATE | ALL | Deny | - |
| WAN_To_CORP_Deny | WAN1 | ZONE_CORPORATE | ALL | Deny | - |
| WAN_To_DMZ_Deny | WAN1 | ZONE_DMZ | ALL | Deny | - |

---

## What I Configured

### 1. Initial Setup

Accessed the FortiGate GUI at `https://192.168.1.99`. The unit arrived from Japan with the GUI in Japanese - changed language to English via System > Settings. Default credentials (admin / no password) did not work - unit had a custom password set by the previous owner. admin/admin worked.

**Configuration applied:**
- Hostname: `MV-FortiGate-60F`
- Timezone: GMT+10 (Melbourne)
- Admin password: updated immediately

Hostname and timezone are set first on every device before anything else. In production, hostnames appear in every log entry, SNMP trap, and monitoring alert - a device called "FortiGate-60F" tells you nothing; "MV-FortiGate-60F" identifies it immediately.

---

### 2. WAN Interface

Configured WAN1 with a static IP on the same /30 subnet as MikroTik Port 7:

- **IP:** 203.0.113.2/30
- **Alias:** WAN1-Mikrotik-Uplink
- **Admin access:** PING only

HTTPS and SSH are disabled on the WAN interface. In production you never expose management access to the untrusted WAN - an attacker who discovers your firewall WAN IP should not be able to reach the management plane.

---

### 3. Default Route

Created a static default route pointing all unknown traffic toward the MikroTik gateway:

- **Destination:** 0.0.0.0/0.0.0.0
- **Gateway:** 203.0.113.1
- **Interface:** WAN1

After adding this route, internet access was confirmed working from the FortiGate internal management interface.

---

### 4. VLAN Sub-interface - Corporate LAN

Created a VLAN 10 sub-interface on the internal VLAN switch interface:

- **Name:** VLAN10_Internal
- **VLAN ID:** 10
- **IP:** 172.16.10.1/24
- **DHCP:** Enabled, pool 172.16.10.100–172.16.10.200, DNS 8.8.8.8
- **Admin access:** PING only

VLAN sub-interfaces allow a single physical interface to carry traffic for multiple VLANs simultaneously. Each sub-interface becomes the default gateway for that VLAN. The FortiGate acts as the Layer 3 boundary - traffic destined for another zone must pass through the firewall where security policy is applied.

---

### 5. Cisco 2960G Switch Configuration

Configured via console cable (USB to RJ45) from MacBook using `screen /dev/tty.usbserial-XXXX 9600`.

**Port assignment:**

| Port | Role | VLAN |
|------|------|------|
| Gi0/1 | Access - Corporate (MacBook) | VLAN 10 |
| Gi0/2 | Access - DMZ (unused in final design) | VLAN 20 |
| Gi0/3–23 | Shutdown | - |
| Gi0/24 | Trunk uplink to FortiGate | VLAN 10, 20 |

**Running configuration:**

```
hostname SW-HQ-LAB

interface GigabitEthernet0/1
 switchport access vlan 10
 switchport mode access

interface GigabitEthernet0/2
 switchport access vlan 20
 switchport mode access

interface GigabitEthernet0/3-23
 shutdown

interface GigabitEthernet0/24
 switchport trunk allowed vlan 10,20
 switchport mode trunk
```

**Why unused ports are shutdown:** An open switch port is an entry point. Any device plugging into an active unused port gets network access. Shutting down unused ports is a CIS benchmark requirement and standard hardening practice in enterprise environments.

**Why Gi0/24 is the uplink:** Using the highest-numbered port as the uplink is a common convention - it keeps uplinks visually separate from access ports in documentation and port maps.

---

### 6. VLAN Segmentation Verification

Connected MacBook to Gi0/1 (VLAN 10 access port) and FortiGate LAN port to Gi0/24 (trunk).

**Result:**
- MacBook received DHCP address: 172.16.10.100
- Ping to FortiGate gateway 172.16.10.1: **Success**
- Ping to 8.8.8.8: **Failed** (expected - no firewall policy yet)

The ping to the gateway working confirmed Layer 1, Layer 2, and Layer 3 were all functioning correctly:
- Layer 1: Physical cable and port link up
- Layer 2: VLAN 10 trunk between switch and FortiGate passing tagged frames correctly
- Layer 3: FortiGate responding as the default gateway

The failed internet ping was expected and correct. FortiGate operates on default deny - all inter-zone traffic is blocked unless an explicit policy allows it. Nothing gets through until you explicitly permit it.

---

### 7. Security Zones

Created two zones to group interfaces by trust level:

- **ZONE_CORPORATE** - assigned VLAN10_Internal sub-interface
- **ZONE_DMZ** - assigned physical DMZ interface

Zones allow firewall policies to reference trust levels rather than individual interfaces. This scales much better in enterprise environments where dozens of interfaces may share the same trust level. A policy referencing ZONE_CORPORATE applies to every interface in that zone automatically.

---

### 8. DMZ Architecture - Design Review and Change

**Initial design:** VLAN 20 sub-interface on the 2960G switch for DMZ.

**Issue identified:** Placing the DMZ on the same physical switch as the corporate LAN - even with VLAN segmentation - is not the most architecturally accurate enterprise design. Physical separation between corporate and DMZ is the correct approach when hardware allows it.

**Change made:** Migrated DMZ from VLAN sub-interface to the FortiGate's dedicated physical DMZ port.

**Troubleshooting encountered during migration:**

Deleting the VLAN20_DMZ sub-interface proved non-trivial. The delete button was greyed out even after unbinding it from ZONE_DMZ.

Investigation steps:
1. Unbound VLAN20_DMZ from ZONE_DMZ - delete still greyed out
2. Changed VLAN20_DMZ IP to 172.16.200.1/24 to eliminate IP conflict - delete still greyed out
3. Checked the **Ref** column in the interfaces table - showed value of 1
4. Clicked the Ref value - revealed FortiGate had automatically created an address object `VLAN20_DMZ_address` when the sub-interface was originally created
5. Navigated to Policy & Objects > Addresses, deleted `VLAN20_DMZ_address`
6. Returned to interfaces - delete button now active, deleted VLAN20_DMZ successfully

**Lesson:** When FortiGate creates a VLAN sub-interface it automatically generates a corresponding address object. This must be deleted before the interface can be removed. The **Ref** column in the interfaces table is the key diagnostic tool - always check it when a delete button is greyed out.

**Physical DMZ port updated:**
- Changed from default 10.10.10.1/24 to 172.16.20.1/24
- Assigned to ZONE_DMZ
- Windows PC connected directly with static IP 172.16.20.10

---

### 9. Firewall Policies

Created seven zone-based policies implementing classic three-zone perimeter architecture - five allow policies and two explicit deny policies with logging enabled for attack simulation:

**CORP_To_WAN** - Corporate users browse the internet. Source NAT translates 172.16.10.x to FortiGate WAN IP 203.0.113.2 so the internet can respond.

**DMZ-to-WAN** - DMZ servers reach the internet for updates and external API access. Source NAT applied.

**WAN_To_DMZ** - Internet can reach the DMZ Windows PC on HTTP (80) and HTTPS (443) only. This simulates a publicly accessible web server. All other inbound ports are denied.

**CORP_To_DMZ** - Corporate users can reach the DMZ server. Internal staff need access to internal servers.

**DMZ_To_CORP_Deny** - DMZ cannot initiate connections to the corporate network. This is the critical defence-in-depth policy - even if the DMZ server is compromised, the attacker cannot pivot to the corporate LAN.

**Summary:**

| Policy | Purpose | NAT |
|--------|---------|-----|
| CORP_To_WAN | Corporate internet access | Yes |
| DMZ-to-WAN | DMZ server internet access | Yes |
| WAN_To_DMZ | Inbound to DMZ on HTTP/HTTPS only | No |
| CORP_To_DMZ | Corporate users reach DMZ | No |
| DMZ_To_CORP_Deny | DMZ blocked from corporate | - |

This architecture is vendor-agnostic - the same zone design applies whether you are running FortiGate, Palo Alto, Checkpoint, or Juniper SRX.

---

### 10. Attack Simulation and Log Verification

#### Why Explicit Deny Policies Were Created

FortiGate's implicit deny at the bottom of the policy table drops unmatched traffic but does not log it by default. To generate audit evidence of blocked attack attempts, two explicit deny policies were added with logging enabled:

- **WAN_To_CORP_Deny** - WAN1 → ZONE_CORPORATE, ALL services, DENY, log all sessions
- **WAN_To_DMZ_Deny** - WAN1 → ZONE_DMZ, ALL services, DENY, log all sessions

WAN_To_DMZ_Deny sits below WAN_To_DMZ in the policy sequence. FortiGate matches top to bottom - HTTP and HTTPS match the allow policy first. Everything else including ICMP falls through to the deny policy and is logged.

---

#### Test 1 - DMZ Cannot Reach Corporate LAN

**Action:** Ping 172.16.10.100 from Windows PC (172.16.20.10)

**Result:** Failed - DMZ_To_CORP_Deny blocked and logged

**Log evidence:** FortiGate Forward Traffic log shows source 172.16.20.10, destination 172.16.10.100, Policy ID 6 (DMZ_To_CORP_Deny), action Deny - one entry per ICMP echo request.

Even if the DMZ server were compromised by an attacker, they cannot pivot to the corporate LAN. The FortiGate enforces this boundary at the policy level regardless of what happens on the DMZ server itself.

---

#### Test 2 - WAN Side Attack Simulation

**Pre-test troubleshooting:**

Initial pings from MikroTik to 172.16.x.x destinations timed out but no log entries appeared. CLI verification showed the policies were configured correctly - `set logtraffic all` was present, and `set action deny` not appearing in CLI output is expected since deny is the FortiGate default and default values are not written to configuration.

The FortiGate packet sniffer was used to investigate:

```
diagnose sniffer packet wan1 "host 203.0.113.1" 4 10
```

While pinging from MikroTik, the sniffer showed only ARP traffic - no ICMP packets at all.

**Root cause:** MikroTik had no route to 172.16.0.0/16. It dropped the packets locally before they left ether7. The FortiGate never saw the traffic.

This is correct real-world behaviour - an ISP router has no routes to RFC1918 private address space. 172.16.x.x, 10.x.x.x, and 192.168.x.x are not routable on the public internet.

**Fix:** Added a static route on MikroTik simulating an attacker who knows the internal subnet structure:

```
/ip route add dst-address=172.16.0.0/16 gateway=203.0.113.2
```

**Lesson:** Always verify traffic is actually reaching the firewall before troubleshooting policy behaviour. The packet sniffer shows what the FortiGate sees on each interface regardless of policy - if the sniffer shows no traffic, the problem is upstream.

---

**Test 2a - WAN to Corporate LAN**

Ping 172.16.10.100 from MikroTik (203.0.113.1)

**Result:** Failed - WAN_To_CORP_Deny blocked and logged

Log: source 203.0.113.1, destination 172.16.10.100, Deny, WAN_To_CORP_Deny

---

**Test 2b - WAN to DMZ via ICMP**

Ping 172.16.20.10 from MikroTik (203.0.113.1)

**Result:** Failed - WAN_To_DMZ_Deny blocked and logged

WAN_To_DMZ only permits HTTP and HTTPS. ICMP falls through to WAN_To_DMZ_Deny.

Log: source 203.0.113.1, destination 172.16.20.10, Deny, WAN_To_DMZ_Deny

---

**Test 2c - WAN to FortiGate WAN Interface**

Ping 203.0.113.2 from MikroTik (203.0.113.1)

**Result:** Success - PING is enabled on WAN1 admin access

> **Production note:** Disable PING on WAN interfaces in production to prevent attackers from confirming the firewall exists at that IP.

---

**Attack Simulation Summary**

| Test | Source | Destination | Expected | Result | Logged |
|------|--------|-------------|----------|--------|--------|
| DMZ to Corporate | 172.16.20.10 | 172.16.10.100 | Deny | Deny ✅ | Yes ✅ |
| WAN to Corporate | 203.0.113.1 | 172.16.10.100 | Deny | Deny ✅ | Yes ✅ |
| WAN to DMZ ICMP | 203.0.113.1 | 172.16.20.10 | Deny | Deny ✅ | Yes ✅ |
| WAN to FortiGate WAN | 203.0.113.1 | 203.0.113.2 | Allow | Allow ✅ | N/A |

---

## What I Learned

**Zone-based firewalling vs ACLs:** The default deny posture of zone-based firewalls is fundamentally different from ACLs. With ACLs you explicitly deny what you do not want. With zones you explicitly allow what you do want - everything else is denied automatically. This is a much stronger security model that is harder to misconfigure.

**FortiGate interface references:** When a VLAN sub-interface is created FortiGate automatically generates an address object. This dependency must be resolved before the interface can be deleted. The Ref column in the interface table is the diagnostic tool for tracking hidden dependencies.

**Default values in CLI output:** FortiGate does not write default configuration values to CLI output. `set action deny` does not appear for deny policies because deny is the default. Always cross-reference GUI and CLI and understand that missing output may mean default, not absent.

**Packet sniffer before policy troubleshooting:** When a firewall policy appears correct but traffic is not being logged, the first step is to verify the traffic is actually arriving at the firewall - not troubleshoot the policy itself. The `diagnose sniffer packet` command is the correct first tool.

**RFC1918 routing:** Private address space is not routable on the public internet. ISP routers have no routes to 172.16.x.x, 10.x.x.x, or 192.168.x.x. This is why an attacker cannot directly reach your internal network from the internet without first gaining a foothold inside the network.

---

## Key Takeaways for Production

- Never enable HTTPS or SSH admin access on WAN interfaces
- Disable PING on WAN interfaces to avoid confirming the firewall exists to external scanners
- Explicit deny policies with logging are essential - implicit deny gives you no audit trail
- Use a dedicated out-of-band management VLAN rather than enabling management access on user-facing VLANs
- Physical DMZ separation is preferred over VLAN-based DMZ when hardware allows it
- Always verify traffic is reaching the firewall with a packet sniffer before troubleshooting policy behaviour
- Check the Ref column before attempting to delete any FortiGate object

---

## Repository Structure

```
project-1-fortigate-perimeter/
├── README.md
├── topology-diagram.png
├── lab-photo.jpg
├── 01-initial-setup/
│   └── screenshots/
├── 02-wan-and-routing/
│   └── screenshots/
├── 03-vlan-interfaces/
│   └── screenshots/
├── 04-switch-configuration/
│   └── screenshots/
├── 05-security-zones/
│   └── screenshots/
├── 06-firewall-policies/
│   └── screenshots/
├── 07-dmz-architecture/
│   └── screenshots/
└── 08-attack-simulation/
    └── screenshots/
```

> **Note:** Topology diagram created in draw.io. All screenshots are organised by section in their respective folders.

---

*Next project: [Project 2 - Site-to-Site IPSec VPN and SSL VPN](../project-2-ipsec-ssl-vpn/README.md)*
