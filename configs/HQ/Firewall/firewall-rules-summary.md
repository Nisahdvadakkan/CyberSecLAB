# Forcepoint NGFW — Firewall Rules Summary (HQ Cluster)

## Architecture Overview

The INDIAFW-HA cluster enforces network segmentation across five distinct security zones. All rules are managed centrally through the Forcepoint Security Management Center (SMC) and automatically synchronized to both active and standby nodes. The cluster uses **Cluster Virtual IPs (CVI)** for traffic forwarding and **Node Dedicated IPs (NDI)** for per-node management, ensuring no single node is a point of failure for either traffic or control plane connectivity.

See [`notes/findings-ngfw-ha-cluster.md`](../../notes/findings-ngfw-ha-cluster.md) for detailed cluster design and failover testing results.

## Security Zones

| Zone | Subnet | Purpose | Connected Systems |
|---|---|---|---|
| **WAN** | 192.168.122.0/24 | External internet edge (BhartiAirtel ISP) | NGFW cluster CVI at 192.168.122.180 |
| **Security Services** | 192.168.50.0/24 | Forcepoint security appliances | Web-Proxy1/2, Email-Gateway, DLP (FSM, DLP Manager, DLP2, Analytics, Protector) |
| **Corp-LAN** | 192.168.60.0/24 | Corporate endpoints & infrastructure | Windows AD, DNS, client PCs, TrueNAS, test machines |
| **SOC** | 192.168.70.0/24 | Centralized security monitoring | Wazuh SIEM/XDR manager and agents |
| **Site-to-Site VPN** | 192.168.150.0/24 | Encrypted tunnel to USA branch | FortiGate (USA) ↔ NGFW Cluster CVI (India) |

## Firewall Policy Structure

The INDIAFW-HA policy is built on a **default-deny** foundation with explicit allow rules for known traffic flows. Rules are organized into functional groups, each with logging and QoS assignments enabled for observability and traffic shaping.

### Rule Groups

**1. VPN Site-to-Site (Rules 5.6.2 – 5.6.5)**

Establishes bidirectional IPSec tunnel between India HQ and USA branch office. All traffic destined for the USA subnet (192.168.150.0/24) is encrypted and routed through the VPN gateway.

| Rule # | Source | Destination | Protocol | Auth | Action | Notes |
|---|---|---|---|---|---|---|
| 5.6.2 | internal-Zones | USA-site-internal | IPSec | — | Allow | VPN tunnel initiation |
| 5.6.3 | USA-site-internal | DC1 | ANY | AD-Auth group | Allow | Remote authentication to HQ DC |
| 5.6.4 | USA-site-internal | DLP-EndpointServers | HTTP/HTTPS | — | Allow | Remote DLP policy fetches |
| 5.6.5 | USA-site-internal | Wazuh (cluster) | Syslog, Wazuh Agent | — | Allow | Remote endpoint monitoring |

**2. Zone-to-Zone Traffic (Rules 5.6.7 – 5.6.16)**

Internal traffic between security zones is explicitly allowed after authentication or service verification. This implements **network segmentation** — each zone can only reach services it needs, not arbitrary internal traffic.

| Rule # | Source | Destination | Protocol | Auth | Action | Notes |
|---|---|---|---|---|---|---|
| 5.6.7 | ANY | ANY | ANY | — | Continue | Logging & inspection (no block) |
| 5.6.8 | USA firewall | NGFW1 | SSH | — | Allow | Remote firewall management |
| 5.6.11 | Zone-Security-Services | DC1 | AD-Auth group | — | Allow | Security appliances authenticate to AD |
| 5.6.12 | Zone-SOC | DC1 | AD-Auth group | — | Allow | SOC nodes authenticate to AD |
| 5.6.13 | Management-Server | DC1 | AD-Auth group | — | Allow | SMC management server auth |
| 5.6.14 | Zone-Corp-AD | DLP-EndpointServers | HTTP/HTTPS | — | Allow | Clients fetch DLP policies |
| 5.6.15 | Zone-Corp-AD & Zone-Security-Services | Wazuh | Syslog/Agent | — | Allow | All zones forward logs to SIEM |
| 5.6.16 | Heartbeat | Heartbeat | ANY | — | Allow | **CRITICAL**: cluster sync path |

**3. Heartbeat Interface (Rule 5.6.16)**

The heartbeat interface carries cluster state sync and liveness checks between NGFW-1 and NGFW-2. This rule is **mandatory** for cluster operation and is never subject to security policy filtering — it uses node-only IPs (192.168.10.253 and .254) and is segregated from user traffic.

**4. Outbound Policy (Rules 5.6.18 – 5.6.20)**

External-facing traffic (Corp-LAN and Security Services → WAN) is constrained to known destinations only:

| Rule # | Source | Destination | Protocol | Action | Notes |
|---|---|---|---|---|---|
| 5.6.18 | Zone-Security-Services | External | ANY | Allow | Web/Email gateways egress for service updates |
| 5.6.19 | Zone-Corp-AD | External | ANY | Allow | Controlled egress; individual client requests logged |
| 5.6.20 | Zone-SOC | External | ANY | Allow | Wazuh integration with external threat feeds (if enabled) |

**5. Default Deny**

All other traffic is implicitly denied. No inbound connections from the WAN are allowed unless explicitly listed above.

## Policy Validation

The policy is validated through multiple mechanisms:

1. **Live Screenshots** — See [ha-cluster/INDIAFW-HA_cluster_Policy_rule.png](../../screenshots/ha-cluster/INDIAFW-HA_cluster_Policy_rule.png) for the full rule table rendered in the SMC UI.

2. **External Nmap Scans** — Ran reconnaissance from an external-facing Kali Linux instance (BhartiAirtel ISP side) targeting the cluster CVI (192.168.122.180). Results confirmed:
   - All scanned ports return `filtered` or `no-response` (not open, not closed)
   - No service banners leaked
   - No information disclosure about internal subnet topology
   - Behavior consistent with a stateful default-deny policy

3. **Zone Isolation Testing** — Internal scans from non-domain test PC (192.168.60.100) targeting other subnets:
   - Blind scans of 192.168.50.0/24 (Security Services) → filtered (zone rule enforced)
   - Blind scans of 192.168.70.0/24 (SOC) → filtered (zone rule enforced)
   - Attempts to bypass via DNS/mDNS → no leakage of internal hostnames
   - Confirms segmentation is working in both directions

4. **Cluster Failover Under Load** — See [findings-ngfw-ha-cluster.md](../../notes/findings-ngfw-ha-cluster.md) for results of forced failover testing while running continuous traffic through the cluster — rule enforcement and state sync held under active failure conditions.

## Rule Maintenance

- **Changes**: pushed to both cluster nodes via SMC in a single commit; no per-node configuration
- **Logging**: every rule has packet/connection logging enabled for forensics
- **QoS**: prioritizes management traffic (AD auth, cluster sync) over user traffic during congestion
- **Revision control**: this summary is maintained in git alongside the lab documentation; config diffs are tracked
