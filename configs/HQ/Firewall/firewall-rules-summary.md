# Forcepoint NGFW — Firewall Rules Summary (HQ Cluster)

## Zones

| Zone | Subnet | Purpose |
|---|---|---|
| WAN | 192.168.122.0/24 | External connectivity (BhartiAirtel), cluster CVI at .180 |
| Security Services | 192.168.50.0/24 | Web/Email Security Gateway, DLP (FSM, DLP2, Analytics, Protector) |
| Corp-LAN | 192.168.60.0/24 | AD/DNS, Windows 10/11 clients, non-domain test PC, TrueNAS |
| SOC | 192.168.70.0/24 | Wazuh SIEM/XDR |
| Site-to-Site VPN | 192.168.150.0/24 | Tunnel to USA branch (FortiGate) |

## Key Rules

| # | Source | Destination | Service | Action | Notes |
|---|---|---|---|---|---|
| 1 | Corp-LAN | Security Services | HTTP/HTTPS | Allow | Web traffic routed through Web Security Gateway |
| 2 | Security Services | Corp-LAN | SMTP | Allow | Email flow through Email Security Gateway |
| 3 | SOC | Any | Syslog / Agent ports | Allow | Wazuh log collection |
| 4 | HQ Cluster CVI | USA (192.168.150.0/24) | IPSec | Allow | Site-to-site VPN |
| 5 | External (WAN) | Any internal zone | Any | Deny (default) | Default-deny inbound, confirmed via Nmap (see `/screenshots/nmap-scans/`) |

*(Replace with your actual rule numbers, ports, and any rules specific to the cluster CVI vs. NDI addressing.)*

## Notes

- Default-deny policy applied at the WAN edge; validated via external Nmap SYN scans returning uniform filtered/no-response results across scanned subnets.
- Rules are configured once via SMC and pushed to both cluster nodes — not configured per-node.
