# Forcepoint NGFW — Firewall Rules Summary (HQ)

## Zones
| Zone | Subnet | Purpose |
|---|---|---|
| WAN | 192.168.122.0/24 | External connectivity |
| Security Services | 192.168.50.0/24 | Web/Email Security, DLP,Analytics Engine, MSSQL |
| Corp-LAN | 192.168.60.0/24 | AD, DNS, endpoints, NAS |
| SOC | 192.168.70.0/24 | Wazuh XDR |

## Key Rules
| # | Source | Destination | Service | Action | Notes |
|---|---|---|---|---|---|
| 1 | Corp-LAN | Security Services | HTTP/HTTPS | Allow | Web traffic routed through WCG |
| 2 | Security Services | Corp-LAN | SMTP | Allow | Email flow through ESG |
| 3 | SOC | Any | Syslog/Agent ports | Allow | Wazuh log collection |
| 4 | External | Any | Any | Deny (default) | Default deny inbound |


## Notes
- Default-deny policy applied at WAN edge
- Nmap scan against WAN interface confirmed only expected ports exposed (see `/screenshots/nmap-scans/`)
