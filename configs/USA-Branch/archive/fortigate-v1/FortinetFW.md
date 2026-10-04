# FortiGate Branch Firewall — USA Office Configuration

**Lab:** CyberSecLAB (GNS3)

**Component:** USAFW (FortiGate) — USA branch perimeter firewall

**Role:** Site-to-site IPSec VPN termination and branch-office network defense

---

## Overview

The USA branch operates as a remote office satellite to the India HQ. The FortiGate firewall at the WAN edge:
- Terminates an **IPSec tunnel** to the India HQ (NGFW-HA cluster)
- Enforces **local network segmentation** within the USA subnet
- Routes **branch-to-HQ traffic** through the encrypted tunnel
- Falls back to **internet breakout** for external services (e.g., SaaS applications, updates)

This design eliminates the need for every USA branch user to route through India HQ for internet access (a major latency and bandwidth bottleneck in real enterprises) while maintaining secure, encrypted access to corporate resources.

---

## Network Configuration

### WAN Interface

| Parameter | Value |
|---|---|
| **Interface** | eth0 (external) |
| **IP Address** | 192.168.123.231 (AT&T Fiber ISP subnet) |
| **ISP Simulation** | AT&T Fiber (separate virtual ISP router in GNS3) |
| **Role** | Termination point for site-to-site VPN from India HQ |

### LAN Interface

| Parameter | Value |
|---|---|
| **Interface** | eth1 (internal) |
| **IP Subnet** | 192.168.150.0/24 |
| **Gateway** | 192.168.150.1 (FortiGate) |
| **Connected Systems** | SalesPC-USA (192.168.150.100), route back to India via tunnel |

### VPN Tunnel Configuration

| Parameter | Value |
|---|---|
| **Tunnel Name** | INDIA-HQ-TO-USA-VPN |
| **Remote Endpoint** | 192.168.122.180 (India HQ NGFW cluster CVI) |
| **Local Endpoint** | 192.168.123.231 (USA USAFW public WAN IP) |
| **Encryption** | IPSec (IKEv2, DES/3DES or AES preferred) |
| **Authentication** | Pre-shared key (configured in both endpoints) |
| **Remote Subnet** | 192.168.50.0/24, 192.168.60.0/24, 192.168.70.0/24 (India HQ zones) |
| **Local Subnet** | 192.168.150.0/24 (USA branch LAN) |
| **Status** | Active; manual route-based forwarding configured |

---

## Firewall Policy

The FortiGate policy is organized into three functional groups:

### 1. Site-to-Site VPN Inbound (India → USA)

Any traffic arriving from the India HQ tunnel destined for branch systems is allowed:

| # | Source | Destination | Protocol | Action | Notes |
|---|---|---|---|---|---|
| 1 | India-HQ (192.168.50.0/24, 192.168.60.0/24, 192.168.70.0/24) | SalesPC-USA (192.168.150.100) | ANY | Allow | Tunnel traffic not re-encrypted; IPSec already provides confidentiality |

### 2. Site-to-Site VPN Outbound (USA → India)

Branch users accessing HQ resources (AD login, file shares, DLP policy fetch, Wazuh agent) must route through the VPN:

| # | Source | Destination | Protocol | Action | Notes |
|---|---|---|---|---|---|
| 2 | SalesPC-USA (192.168.150.100) | India-HQ DC1 (192.168.60.10) | SMB/CIFS, Kerberos | Allow | Active Directory domain join & GPO |
| 3 | SalesPC-USA (192.168.150.100) | India-HQ DLP (192.168.50.23) | HTTP/HTTPS | Allow | DLP policy enforcement, compliance checks |
| 4 | SalesPC-USA (192.168.150.100) | India-HQ Wazuh (192.168.70.100) | Syslog, Wazuh Agent | Allow | Centralized logging, threat detection |

### 3. Internet Breakout (USA LAN → External WAN)

Branch devices are allowed to reach public internet for SaaS, updates, and external services:

| # | Source | Destination | Protocol | Action | Notes |
|---|---|---|---|---|---|
| 5 | SalesPC-USA (192.168.150.100) | External (0.0.0.0/0) | HTTP/HTTPS | Allow | Public internet access; separate from VPN path |
| 6 | SalesPC-USA (192.168.150.100) | External (0.0.0.0/0) | DNS (UDP/53) | Allow | Recursive DNS queries to public resolvers |

### 4. Default Deny

All other traffic is implicitly denied. This includes:
- Attempts to reach internal India HQ subnets without going through the VPN tunnel
- Inbound connections from the public internet (WAN) to branch LAN
- East-West lateral movement between branch and external networks

---

## Routing Configuration

The FortiGate uses a combination of **static routes** and **policy-based routing** (PBR) to direct traffic appropriately:

### Static Routes

| Destination | Gateway | Interface | Metric | Purpose |
|---|---|---|---|---|
| 192.168.150.0/24 | Direct (connected) | eth1 (LAN) | 0 | Local branch subnet |
| 192.168.50.0/24 | 192.168.150.1 (VPN tunnel) | VPN Tunnel | 10 | Security Services zone via HQ |
| 192.168.60.0/24 | 192.168.150.1 (VPN tunnel) | VPN Tunnel | 10 | Corp-LAN zone via HQ |
| 192.168.70.0/24 | 192.168.150.1 (VPN tunnel) | VPN Tunnel | 10 | SOC zone via HQ |
| 0.0.0.0/0 | AT&T ISP gateway (192.168.123.1) | eth0 (WAN) | 20 | Default route to internet |

### Policy-Based Routing (PBR)

Each firewall policy above implicitly selects the appropriate outbound interface:
- Policies matching the India HQ subnets are bound to the VPN tunnel (encapsulated and encrypted)
- Policies matching 0.0.0.0/0 (external) use the WAN interface (direct internet, no tunnel)
- No cross-mixing: traffic cannot accidentally bypass the tunnel

---

## VPN Tunnel Status & Monitoring

The tunnel is monitored through several mechanisms:

1. **Tunnel Status Dashboard** — FortiGate web UI shows:
   - IPSec tunnel up/down state
   - Bytes encrypted/decrypted
   - Phase 1 (IKE) and Phase 2 (ESP) status
   - Last handshake timestamp

2. **Syslog Forwarding** — IKE negotiation, tunnel establish/teardown, and DPD (Dead Peer Detection) events are sent to Wazuh SIEM for centralized alerting.

3. **Continuous Connectivity Test** — Lab validation used ping from SalesPC-USA to India HQ systems (e.g., AD server at 192.168.60.10) to confirm:
   - Tunnel auto-negotiates on first traffic
   - Tunnel remains stable under sustained load
   - Latency is acceptable for interactive workloads

---

## Security Considerations

### Strengths

✅ **Encrypted tunnel** isolates branch-to-HQ traffic from ISP inspection  
✅ **Split tunnel** design allows internet breakout without forcing all traffic through HQ (no bandwidth bottleneck)  
✅ **Default-deny policy** at WAN edge; no unexpected inbound exposure  
✅ **Centralized DLP & SIEM** — branch endpoints still report to India HQ security services  

### Known Limitations (Documented for Future Hardening)

⚠️ **VPN termination on physical IP** — Currently terminates on FortiGate's own WAN IP (192.168.123.231), not a cluster VIP. If the FortiGate fails, the tunnel must be re-established to a new IP. *Mitigation*: FortiGate HA (active-active) can be added in future iterations.

⚠️ **Dual-WAN redundancy** — Only one ISP (AT&T Fiber) is simulated. Real deployments would add a secondary WAN link (cable, LTE) for failover. *Mitigation*: add a second WAN interface and FortiGate SD-WAN rules.

⚠️ **No branch-side SIEM agent** — SalesPC-USA sends logs to India HQ via Wazuh agent but has no local SIEM cache if tunnel goes down. *Mitigation*: deploy a lightweight syslog cache or local log aggregator at the branch.

---

## Testing & Validation

### Scenario 1: Tunnel Establishment

**Test:** Power on FortiGate and SalesPC-USA with tunnel configured.

**Expected:** IPSec Phase 1/2 negotiation completes automatically on first data traffic attempt.

**Result:** ✅ Tunnel established within 2-3 seconds of first ping attempt to India HQ DC1.

### Scenario 2: Sustained Traffic Through Tunnel

**Test:** Run large file copy from SalesPC-USA to India HQ file share (TrueNAS) for 5 minutes.

**Expected:** Tunnel remains stable, no drops, no re-keying interruptions.

**Result:** ✅ 500 MB copied successfully; FortiGate dashboard shows steady bytes encrypted/decrypted.

### Scenario 3: Internet Breakout (Split Tunnel)

**Test:** From SalesPC-USA, attempt both (a) HQ access (192.168.60.10), (b) external web (8.8.8.8).

**Expected:** (a) routes through VPN tunnel (shown in routing table), (b) routes direct to WAN (no tunnel).

**Result:** ✅ Dual-path confirmed; `tracert` from SalesPC-USA shows HQ traffic via tunnel, internet traffic direct to ISP gateway.

### Scenario 4: Tunnel Failure & Recovery

**Test:** Bring down the VPN tunnel, wait 30 seconds, bring it back up.

**Expected:** Tunnel re-establishes automatically; buffered traffic is retransmitted.

**Result:** ✅ DPD timeout triggered teardown; reconnect took ~3 seconds; minimal packet loss.

---

## Integration with Lab Architecture

The USA branch is part of the larger enterprise security lab and integrates as follows:

- **Endpoint Security** — SalesPC-USA (192.168.150.100) is a Wazuh agent managed from India HQ SOC
- **DLP Compliance** — Branch PC fetches DLP policies from India HQ manager, enforces same rules as HQ
- **Authentication** — Branch PC domain-joined to India HQ AD; GPOs applied centrally
- **Centralized Logging** — All branch activity (logins, file access, DLP events) forwarded to Wazuh dashboard

See [README.md](../../README.md) for full topology and [notes/forcepoint-dlp-wazuh-integration.md](../../notes/forcepoint-dlp-wazuh-integration.md) for details on centralized SIEM and DLP integration.

---

## Future Improvements

- [ ] **FortiGate HA** — Deploy active-standby or active-active pair for branch edge redundancy
- [ ] **Dual-WAN SD-WAN** — Add secondary ISP link and configure Fortigate's SD-WAN rules for automatic failover
- [ ] **Local Branch SIEM Agent** — Cache logs locally if tunnel is down to avoid loss of visibility
- [ ] **IPSec Encryption Hardening** — Audit and enforce latest IKEv2 + AES-GCM + SHA-256+ standards (replacing older DES/3DES if currently in use)
- [ ] **VPN Tunnel Alerts in Wazuh** — Feed FortiGate IPSec events into Wazuh for alerting on unexpected tunnel state changes
- [ ] **Bandwidth QoS for Branch** — Implement per-application QoS on the FortiGate to prioritize business-critical apps (AD, DLP) over casual internet use
