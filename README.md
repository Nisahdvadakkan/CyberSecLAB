# Enterprise Security Lab — Multi-Site Network with Layered Defense (GNS3).

## Overview

This lab simulates a two-site enterprise network (India HQ and USA branch) built in GNS3, designed to test and validate layered security controls across network, web, email, and data-loss-prevention layers. The goal was to model a realistic corporate environment — complete with a segmented internal network, dedicated security services zone, and a SOC — then actively test the effectiveness of those controls using reconnaissance and attack simulation techniques.

**Why this lab:** Most home labs stop at "I set up a firewall." This one goes further — combining perimeter defense with true **HA clustering**, endpoint infrastructure, threat detection (XDR), and site-to-site connectivity, then validating the setup with real scans, forced-failover tests, and rule testing rather than just assuming it works.

## Lab Evolution

This lab is built iteratively, with each phase documented rather than overwritten:

- **v1 — Initial build:** India HQ Forcepoint NGFW + USA branch FortiGate, connected via a cross-vendor IPsec site-to-site VPN.
- **v2 — HA clustering:** India HQ NGFW upgraded to an Active-Standby cluster (NGFW-1/NGFW-2), validated with a forced-failover test. See [`notes/findings-ngfw-ha-cluster.md`](notes/findings-ngfw-ha-cluster.md).
- **v3 — DLP-to-SIEM pipeline:** Forcepoint DLP incident/audit/system logs forwarded to Wazuh via custom CEF decoders and rules, with a live dashboard. See [`notes/forcepoint-dlp-wazuh-integration.md`](notes/forcepoint-dlp-wazuh-integration.md).
- **v4 — Branch firewall migration (current):** The USA branch FortiGate was replaced with a second, standalone Forcepoint NGFW after the FortiGate evaluation license expired — used as an opportunity to move to a single-vendor, native NGFW-to-NGFW route-based VPN managed by the same SMC as the India cluster, rather than a cross-vendor IPsec link. Includes a full VPN hardening pass (PFS, certificate-based auth, scoped least-privilege access rules, defense-in-depth policy on both firewalls). See [`notes/findings-usa-ngfw-migration.md`](notes/findings-usa-ngfw-migration.md). The original FortiGate configuration is preserved, not deleted, at [`configs/USA-Branch/archive/fortigate-v1/`](configs/USA-Branch/archive/fortigate-v1/).

## Topology

**India (Corporate HQ)**

- **Forcepoint NGFW Cluster (Active-Standby HA)** — NGFW-1 and NGFW-2, perimeter firewall / network segmentation, managed centrally via SMC (`192.168.122.10`). WAN edge at `192.168.122.180`. Validated with a real forced-failover test (see [`notes/findings-ngfw-ha-cluster.md`](notes/findings-ngfw-ha-cluster.md)).
- **Security Services Subnet** (`192.168.50.0/24`)
  - Forcepoint Web Security Gateway (Web-Proxy1, Web-Proxy2)
  - Forcepoint Email Security Gateway
  - Forcepoint DLP — FSM + DLP Manager, DLP2, Analytics, Protector, FSM-SQL
- **Corp-LAN Subnet** (`192.168.60.0/24`)
  - Windows Server (AD, DNS, GPO) — also hosts HMailServer
  - Windows 10 / Windows 11 client PCs, plus a non-domain test PC (negative-control testing)
  - TrueNAS (shared storage)
- **SOC Subnet** (`192.168.70.0/24`)
  - Wazuh (SIEM + XDR) for centralized logging, alerting, and threat detection
- **Kali-Attacker** — positioned externally (via BhartiAirtel ISP), used for reconnaissance and attack-simulation testing against the WAN edge

**USA (Branch Office)**

- **Forcepoint NGFW (standalone)** — `USA_NGFW`, managed by the same SMC instance as the India HA cluster. WAN `192.168.123.180/24` (AT&T Fiber ISP), LAN `192.168.150.1/24`. Not clustered — a single branch sales endpoint doesn't justify HA complexity, unlike HQ.
- **Site-to-Site VPN** — native Forcepoint NGFW-to-NGFW, route-based (SD-WAN), profile `Suite-B-GCM-128`: IKEv2, AES-GCM-128, DH Group 19, PFS enabled, RSA certificate authentication, always-on tunnel. Firewall policy on both ends is scoped and direction-aware rather than a shared ANY/ANY rule — see [`notes/findings-usa-ngfw-migration.md`](notes/findings-usa-ngfw-migration.md) for the full design and hardening writeup.
- **SalesPC-USA** endpoint

![Network Topology](diagrams/topology.png)

## Objectives

- Simulate a realistic enterprise network with segmented zones (security services, corporate LAN, SOC)
- Implement layered security: network firewalling, web filtering, email security, and DLP
- Eliminate the firewall as a single point of failure via an Active-Standby NGFW cluster
- Establish secure site-to-site connectivity between two geographically separated offices
- Centralize visibility and detection using an XDR platform (Wazuh)
- Validate the security posture through active testing rather than assuming configs are correct

## Tools & Technologies

| Category | Tool |
|---|---|
| Network Emulation | GNS3 |
| Firewall / NGFW | Forcepoint NGFW (India: Active-Standby cluster; USA: standalone) |
| Firewall Management | Forcepoint SMC (single instance, both sites) |
| Web Security | Forcepoint Web Security Gateway |
| Email Security | Forcepoint Email Security Gateway |
| DLP | Forcepoint DLP (FSM, DLP Manager, Analytics, Protector) |
| Connectivity | Site-to-Site VPN (native NGFW-to-NGFW, route-based, IKEv2/AES-GCM-128) |
| Directory Services | Windows Server (AD, DNS, GPO) |
| Endpoints | Windows 10 / Windows 11 |
| Storage | TrueNAS |
| SOC / XDR | Wazuh |
| Testing | Nmap, Kali Linux |

## Testing & Validation

### Firewall Cluster (NGFW HA) — India HQ

- **Cluster Configuration** — Deployed Active-Standby architecture with CVI (Cluster Virtual IP) and NDI (Node Dedicated IP) per interface. Screenshots show SMC dashboard, interface hierarchy, routing configuration, and live policy rules in [`screenshots/ha-cluster/`](screenshots/ha-cluster/).
- **External Reconnaissance** — Ran Nmap scans from a Kali Linux attacker position (via BhartiAirtel ISP) targeting the cluster WAN edge (192.168.122.180). Results: all ports return `filtered`, no service banners, no information leakage — confirming default-deny policy enforcement.
- **Zone Segmentation Testing** — Blind scans from internal non-domain test PC (192.168.60.100) against Security Services (192.168.50.0/24), SOC (192.168.70.0/24), and other zones returned uniformly `filtered` responses — confirming bidirectional zone isolation and zero topology leakage.
- **Failover Validation** — Forced failure of active node (NGFW-1) during continuous ping to cluster CVI. Measured results: automatic failover to NGFW-2 confirmed, ~3 packets lost / ~3 second outage, manual recovery via "Go Standby" verified correct per Forcepoint design. Full writeup in [`notes/findings-ngfw-ha-cluster.md`](notes/findings-ngfw-ha-cluster.md).

### Firewall Rules & Policy — India HQ

- **Policy Review** — Live screenshot of INDIAFW-HA firewall policy table showing zone-to-zone rules, VPN rules, heartbeat sync, and outbound policies. See [`configs/HQ/Firewall/firewall-rules-summary.md`](configs/HQ/Firewall/firewall-rules-summary.md) for detailed rule-by-rule breakdown.

### Site-to-Site VPN & USA Branch NGFW

- **VPN Migration** — USA branch firewall was migrated from FortiGate (cross-vendor) to Forcepoint NGFW (single-vendor) to simplify management and improve interoperability. Original FortiGate config preserved in [`configs/USA-Branch/archive/fortigate-v1/FortinetFW.md`](configs/USA-Branch/archive/fortigate-v1/FortinetFW.md).
- **VPN Design & Hardening** — Native Forcepoint NGFW-to-NGFW route-based (SD-WAN) tunnel using `Suite-B-GCM-128` profile:
  - IKEv2 with AES-128 (IKE) + AES-GCM-128 (IPsec)
  - PFS enabled (DH Group 19)
  - RSA certificate-based authentication (not PSK)
  - Always-on tunnel persistence
  - NAT exemption for VPN traffic, ordered *before* dynamic-PAT rule (prevents a common misconfiguration)
  - Defense-in-depth: both firewalls independently enforce least-privilege access rules; no shared ANY/ANY rule
  - Screenshots show VPN profile (General/IKE/IPsec), gateways, routing table, and policy rules from both ends. Full design rationale, hardening comparison, and known limitations in [`notes/findings-usa-ngfw-migration.md`](notes/findings-usa-ngfw-migration.md).

### DLP & SIEM Integration

- **End-to-End Pipeline** — Built Forcepoint DLP → Wazuh integration: configured syslog forwarding from DLP Manager, wrote custom Wazuh decoders/rules to parse CEF-formatted logs, validated with wazuh-logtest, confirmed real incidents generate appropriately-leveled alerts.
- **Dashboard** — Live Wazuh dashboard ("Forcepoint DLP Overview") shows blocked vs. allowed incidents over time, policy categories, severity breakdown, and top source users. Screenshot in [`screenshots/wazuh-siem/`](screenshots/wazuh-siem/).
- **Endpoint Visibility** — Wazuh manages six active agents across infrastructure and endpoints (FMSSRVR, DC1, DLP2, SQL, testpc3, TestPC-USA-01), providing centralized logging and threat detection. Agent status visible in SIEM dashboard.
- **Known Issues & Fixes** — Documented eight non-obvious integration gotchas: TCP/UDP mismatch, reserved field name collisions, decoder chaining limits, field-name collapsing, timestamp mapping failures, rule group inheritance, and aggregation field selection. Full writeup in [`notes/forcepoint-dlp-wazuh-integration.md`](notes/forcepoint-dlp-wazuh-integration.md).
- **Firewall-level Wazuh integration (India HA cluster and USA NGFW) is planned but not yet configured** — tracked as a future roadmap item; see [`notes/findings-usa-ngfw-migration.md`](notes/findings-usa-ngfw-migration.md).

## Repository Structure

```
├── README.md
├── LICENSE
├── diagrams/
│   ├── topology.png
│   └── New_Topology.png
├── configs/
│   ├── HQ/
│   │   ├── Firewall/
│   │   │   └── firewall-rules-summary.md
│   │   └── Wazuh/
│   │       ├── decoders/
│   │       │   └── forcepoint_dlp_decoders.xml
│   │       └── rules/
│   │           └── forcepoint_dlp_rules.xml
│   └── USA-Branch/
│       ├── forcepoint-ngfw/
│       │   └── (USA NGFW config notes — see findings-usa-ngfw-migration.md)
│       └── archive/
│           └── fortigate-v1/
│               └── FortinetFW.md (legacy FortiGate config, preserved for reference)
├── screenshots/
│   ├── ha-cluster/
│   │   ├── INDIAFW-HA_cluster_dashboard.png
│   │   ├── INDIAFW-HA_cluster_interface_Config.png
│   │   ├── INDIAFW-HA_cluster_interface_routing.png
│   │   └── INDIAFW-HA_cluster_Policy_rule.png
│   ├── usa-ngfw-vpn/
│   │   ├── INDIAHA-VPN_Policy.png
│   │   ├── Route-Based-VPN.png
│   │   ├── USA_NGFW_Interfaces.png
│   │   ├── USA_NGFW_NAT_Policy_rules.png
│   │   ├── USA_NGFW_routing.png
│   │   ├── USA_NGFW_VPN_Policy_rules.png
│   │   ├── VPN_gateways.png
│   │   ├── VPN_profile_General.png
│   │   ├── VPN_profile_IKE_SA.png
│   │   └── VPN_profile_IPsec_SA.png
│   └── wazuh-siem/
│       ├── forcepoint_DLP_Wazuh_dashboard.png
│       └── WazuhDashboard.PNG
└── notes/
    ├── findings-ngfw-ha-cluster.md
    ├── findings-usa-ngfw-migration.md
    └── forcepoint-dlp-wazuh-integration.md
```

## Future Improvements

- [ ] Add a Backup Heartbeat interface for the NGFW cluster (currently only Primary exists) for full redundancy of the sync path itself
- [ ] Re-point the site-to-site IPSec VPN to terminate on the cluster CVI rather than a node's physical IP, so the VPN doesn't become the remaining single point of failure
- [ ] Feed cluster failover/health events into Wazuh SIEM so failover events are logged and alertable
- [ ] Document a dual-WAN uplink design for the India HQ edge, since firewall HA alone doesn't protect against an ISP-side outage
- [x] DLP-to-SIEM pipeline complete (decoders, rules, dashboard) — see `notes/forcepoint-dlp-wazuh-integration.md`
- [x] USA branch migrated from FortiGate to a native Forcepoint NGFW-to-NGFW site-to-site VPN, with a full hardening pass (PFS, certificate auth, scoped least-privilege policy) — see `notes/findings-usa-ngfw-migration.md`
- [ ] Extend Wazuh integration to firewall-level logs for both the India HA cluster and the USA NGFW
- [ ] Add SIEM correlation rules for lateral movement
- [ ] Test DLP with sample exfiltration attempts (alerting pipeline now in place to make this measurable)
- [ ] SOC investigation / incident response exercise
