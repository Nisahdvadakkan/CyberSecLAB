# Enterprise Security Lab — Multi-Site Network with Layered Defense (GNS3)

## Overview
This lab simulates a two-site enterprise network (India HQ and USA branch) built in GNS3, designed to test and validate layered security controls across network, web, email, and data-loss-prevention layers. The goal was to model a realistic corporate environment — complete with a segmented internal network, dedicated security services zone, and a SOC — then actively test the effectiveness of those controls using reconnaissance and attack simulation techniques.

**Why this lab:** Most home labs stop at "I set up a firewall." This one goes further — combining perimeter defense, endpoint infrastructure, threat detection (XDR), and site-to-site connectivity, then validating the setup with real scans and rule testing rather than just assuming it works.

## Topology

**India (Corporate HQ)**
- Forcepoint NGFW — perimeter firewall / network segmentation
- **Security Services Subnet**
  - Forcepoint Web Security Gateway
  - Forcepoint Email Security Gateway
  - Forcepoint DLP
- **Corp-LAN Subnet**
  - Windows Server (AD, DNS, GPO)
  - Windows 10 and Windows 11 client PCs
  - NAS server (shared storage)
- **SOC Subnet**
  - Wazuh (XDR) for centralized logging, alerting, and threat detection

**USA (Branch Office)**
- FortiGate firewall
- Site-to-Site VPN tunnel connecting the USA branch to the India HQ network

/diagrams/topology.png

## Objectives
- Simulate a realistic enterprise network with segmented zones (security services, corporate LAN, SOC)
- Implement layered security: network firewalling, web filtering, email security, and DLP
- Establish secure site-to-site connectivity between two geographically separated offices
- Centralize visibility and detection using an XDR platform (Wazuh)
- Validate the security posture through active testing rather than assuming configs are correct

## Tools & Technologies
| Category | Tool |
|---|---|
| Network Emulation | GNS3 |
| Firewall / NGFW | Forcepoint NGFW |
| Web Security | Forcepoint Web Security Gateway |
| Email Security | Forcepoint Email Security Gateway |
| DLP | Forcepoint DLP |
| Branch Firewall | FortiGate |
| Connectivity | Site-to-Site VPN |
| Directory Services | Windows Server (AD, DNS, GPO) |
| Endpoints | Windows 10 / Windows 11 |
| Storage | NAS (shared storage) |
| SOC / XDR | Wazuh |
| Testing | Nmap |

## Testing & Validation
- Ran Nmap scans against the Forcepoint NGFW from an external-facing position to assess exposed ports/services
- Reviewed and validated firewall rule behavior (allowed vs. blocked traffic between zones)
- Checked whether segmentation correctly isolated the Security Services, Corp-LAN, and SOC subnets from each other
- Verified site-to-site VPN connectivity and traffic flow between India HQ and the USA branch

*(Fill in specifics here once you've documented them — e.g., "Nmap scan showed only ports X/Y open externally, consistent with firewall policy" or "identified that [specific rule] allowed broader access than intended and corrected it." Concrete findings like this are what make a lab stand out — even small ones.)*

## Key Learnings
*(2–4 bullet points in your own words — what surprised you, what you'd do differently, what this taught you about real enterprise security. Recruiters read this section closely because it shows you can reflect, not just follow a guide.)*

## Repository Structure
```
├── README.md
├── diagrams/
│   └── topology.png
├── configs/
│   ├── forcepoint-ngfw/
│   ├── fortigate/
│   └── wazuh/
├── screenshots/
│   ├── nmap-scans/
│   ├── firewall-rules/
│   └── wazuh-dashboard/
└── notes/
    └── findings.md
```

## Future Improvements
*(Optional but a nice touch — e.g., "add SIEM correlation rules for lateral movement," "test DLP with sample exfiltration attempts," "expand to a third site")*
