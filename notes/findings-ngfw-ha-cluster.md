# Forcepoint NGFW Firewall Cluster (HA) — Design, Troubleshooting & Validation

**Lab:** CyberSecLAB (GNS3)

**Component:** INDIAFW-HA (NGFW-1 / NGFW-2) — India HQ perimeter cluster
.
**Objective:** Eliminate the single point of failure at the India HQ internet edge by clustering two Forcepoint NGFW engines in Active-Standby mode, validated with a real forced-failover test rather than a config screenshot alone.

---

## 1. Design

Following Forcepoint's clustering model (inherited from StoneSoft), each shared zone interface — WAN, Security Services, Corp-LAN, SOC — carries:

- **CVI (Cluster Virtual IP)** — one shared IP per interface that both nodes answer to. Switches, gateways, and clients route to this, not to either node's own IP.
- **NDI (Node Dedicated IP)** — each node keeps its own IP on the same interface, used for node-level management/monitoring in SMC.
- **Heartbeat interface** — a dedicated NGFW-1 ↔ NGFW-2 link, carrying only cluster state sync and "is my peer alive" checks. No CVI needed on this interface.

Cluster management is centralized through SMC (`192.168.122.10`), rather than configuring each node independently.

## 2. Issues Found & Root Causes

During initial clustering, NGFW-2 (`NGFWHA-2`) showed intermittent **Offline → Timeout → Online** cycling. Root causes investigated, in order:

1. **GNS3 host resource pressure (ruled out)** — each NGFW node is a fixed-memory VM independent of overall host capacity. Checked `free -h` inside each node directly (not host-level Task Manager) to confirm swap/RAM inside the VM itself wasn't tripping Forcepoint's internal Engine Test thresholds.
2. **Heartbeat link instability (investigated)** — confirmed the link between NGFW-1 and NGFW-2's heartbeat interface (`eth7`) was live and passing traffic; a multicast reachability test on the sync channel was inconclusive on its own and not treated as decisive evidence.
3. **Unicast MAC / switch relearning lag (considered)** — with Unicast MAC mode, CVI MAC ownership has to be relearned by the GNS3 OpenvSwitch nodes each time a node flaps, which can look like instability even when the node process itself is healthy.
4. **SMC logs (decisive check)** — filtering SMC logs by Sender = NGFW-2 gave the actual failing Engine Test rather than relying on inference from symptoms.
5. **The Interface 1,2,3 were not plugged-in to any of the switches and once they have plugged into the appropriate swithes the Interface status brought up, That made the nodes to think none of the nodes are down.

Root cause Identified was the 5 one .
## 3. Recovery Behavior — Documented, Not a Bug

After a node recovers from an outage, it does **not** automatically rejoin as Active or Standby — it sits in a non-participating state until an administrator manually issues **Go Standby**. This is Forcepoint's intended safety behavior: a node that just crashed shouldn't silently rejoin production traffic handling without a human confirming it's healthy.

Applying **Go Standby** manually resolved the recurring flap — the node then stayed persistently in Standby, confirming the underlying instability was genuinely fixed rather than papered over.

## 4. Failover Test — Methodology & Results

**Test:** Forced a failure on the active node (NGFW-1) while running continuous ping through the cluster CVI.

**Result:**
- Automatic failover to NGFW-2 confirmed — no manual intervention required to restore traffic.
- Brief, bounded interruption during cutover (3 packet lost / 3 second).
- Heartbeat, state sync, and all four zone interfaces (WAN, Security Services, Corp-LAN, SOC) confirmed stable post-failover.
- Manual recovery via **Go Standby** confirmed correct per Forcepoint's documented design (Section 3).

**Outcome:** a working, validated Active-Standby Forcepoint NGFW cluster — real automatic failover, a bounded outage window, and a correct recovery procedure for the returning node.

## 5. Follow-Up / Future Work

- [ ] Add a **Backup Heartbeat** interface (currently only Primary exists) — recommended by Forcepoint KB 000007930 for full redundancy of the heartbeat/sync path itself.
- [ ] Re-point the site-to-site IPSec VPN to India HQ to terminate on the cluster CVI (not a node's physical IP), so the VPN itself doesn't become the remaining single point of failure.
- [ ] Feed cluster failover/health events into Wazuh SIEM so failover events are logged and alertable, not just visible on the SMC dashboard.
- [ ] Document a dual-WAN uplink design as a next step, since firewall HA alone doesn't protect against an ISP-side outage.

---

*Note: true HA timing (sync-link latency, failover speed) in a GNS3 lab won't perfectly mirror physical or cloud-hosted appliances — but the architecture and validation methodology documented here mirror what a real enterprise deployment would use.*
