# Forcepoint DLP → Wazuh SIEM Integration

**Date:** August 2026
**Goal:** Forward Forcepoint DLP incident, audit, and system logs to Wazuh over syslog, build custom decoders/rules to parse the CEF-formatted messages, generate real alerts, and surface them in a Wazuh dashboard.

This closes out two items from the lab's Future Improvements list: *"add SIEM correlation rules"* and groundwork for *"test DLP with sample exfiltration attempts"* (the alerting pipeline needed to exist before exfiltration testing would be observable).

## Architecture

```
Forcepoint DLP (Settings > General > Remediation > Syslog Settings)
        │  UDP/514, CEF-formatted syslog
        ▼
Wazuh Manager (<remote> syslog listener)
        │
        ▼
Custom decoders (etc/decoders/forcepoint_dlp_decoders.xml)
        │  parses CEF header + type-specific extension fields
        ▼
Custom rules (etc/rules/forcepoint_dlp_rules.xml)
        │  severity escalation, blocked-action highlighting
        ▼
alerts.json → Filebeat → Wazuh Indexer (OpenSearch)
        │
        ▼
Wazuh Dashboard (Discover + custom "Forcepoint DLP Overview" dashboard)
```

## Configuration Steps

1. **Forcepoint DLP Manager** → Data Security module → Settings > General > Remediation
   - Syslog Settings: IP/hostname + port (514) of the Wazuh manager
   - Enabled **Send syslog message** under both Logs > Audit log and Logs > System log
   - Added **Audit Incident > Send Syslog Message** to the DLP policy's action plan for incident-type events

2. **Wazuh Manager** (`/var/ossec/etc/ossec.conf`)
   ```xml
   <remote>
     <connection>syslog</connection>
     <port>514</port>
     <protocol>udp</protocol>
     <allowed-ips>192.168.50.0/24</allowed-ips>
     <local_ip>192.168.70.100</local_ip>
   </remote>
   ```
   Also enabled `<logall>yes</logall>` / `<logall_json>yes</logall_json>` under `<global>` during troubleshooting, to confirm raw messages were landing in `archives.log` before rules existed to alert on them.

3. **Custom decoders and rules** — see `configs/wazuh/decoders/forcepoint_dlp_decoders.xml` and `configs/wazuh/rules/forcepoint_dlp_rules.xml` in this repo.

## Log Format

Forcepoint DLP sends ArcSight CEF-formatted messages. Three distinct message shapes exist, differentiated by the `name` field in the CEF header:

| Type | CEF Name field | Notes |
|---|---|---|
| Incident | `DLP Syslog` | Has a signature ID in the header |
| Audit Log | `DLP Audit Log` | Has a signature ID in the header |
| System Log | `DLP System Log` | **No** signature ID field — shorter header than the other two |

Example (incident):
```
CEF:0|Forcepoint|Forcepoint DLP|10.3.0|9666301|DLP Syslog|0| act=Blocked duser=DLPTEST.COM fname=N/A msg=EndPoint Operation suser=Internal User cat=DLP Test Policy sourceServiceName=Endpoint HTTPS analyzedBy=Policy Engine FMSSRVR loginName=FORCELAB\user1 sourceIp=N/A severityType=HIGH sourceHost=TESTPC.forcelab.local productVersion=8.0 maxMatches=28 timeStamp=2026-08-29 22:49:29.516 destinationHosts=DLPTEST.COM eventId=5149638932895828039
```

## Problems Hit and Fixed

Getting from "packets arriving" to "alert visible on a dashboard" surfaced six distinct, non-obvious failures. Documenting these because each one looked like success right up until the next layer:

1. **TCP vs UDP mismatch.** `telnet` to port 514 timed out and looked like a connectivity failure. Forcepoint's syslog client uses plain UDP (no protocol selector in its UI at all); `telnet` only tests TCP. `tcpdump` confirmed UDP packets were arriving correctly the whole time — the "failure" was testing with the wrong tool, not a real problem.

2. **`logall`/`logall_json` disabled.** Wazuh only writes to `archives.log` for logs that either match a rule or have these flags enabled. With no decoder/rule yet, nothing matched, so the archive stayed empty even with a working listener.

3. **Reserved field name (`action`) crashed the manager.** A custom decoder field literally named `action` collides with a Wazuh analysisd static/reserved field, used internally for active response. This didn't fail at config-write time — it crashed `wazuh-manager` on restart with `Field 'action' is static`. Renamed to `dlp_action`.

4. **Multi-level decoder chaining rejected.** Structured the decoders as parent → header → type-specific (three levels), using `<use_own_name>yes</use_own_name>` so each type-specific decoder would register under its own name. This build of Wazuh (4.12.0) rejected the grandchild's `<parent>` reference with `Parent decoder name invalid`, even though the referenced decoder existed and had `use_own_name` set. Fixed by flattening to two levels — each type-specific decoder is a direct child of the single top-level parent, each parsing the full CEF header **and** its own extension fields in one combined regex.

5. **Decoder name collapsed to parent name.** Even with `<use_own_name>yes</use_own_name>`, `wazuh-logtest` reported every decoder as `forcepoint-dlp` (the top-level parent), not the more specific child name. This meant rules using `<decoded_as>forcepoint-dlp-incident</decoded_as>` never matched anything. Rewrote the rules to branch on **which fields are present** instead (`severity_type` only exists on incident events, `admin_role` only on audit events, `reporter` only on system events) — more robust than depending on decoder-name granularity that this build doesn't reliably expose.

6. **Reserved field name (`timestamp`) silently dropped documents at the indexer.** This one was the hardest to catch: decoding worked, the rule matched, `alerts.json` had a complete valid entry — and it still never appeared in Discover. `Failed to parse date field [2026-08-29 23:43:02.725] with format [strict_date_optional_time]` in the Filebeat log revealed that Wazuh's index template pre-maps `data.timestamp` as a strict ISO-8601 date field. Forcepoint's own timestamp format (`YYYY-MM-DD HH:MM:SS.mmm`, space-separated, no timezone) failed that mapping, and OpenSearch rejected the *entire document* rather than just that field. Renamed to `dlp_timestamp`. Also pre-emptively renamed two other generically-named fields (`code` → `dlp_code`, `local_time` → `dlp_local_time`) in the System Log decoder to avoid hitting the same class of bug a third time.

**Lesson for future decoder work in this lab:** custom field names need checking against two separate reserved-word lists — Wazuh analysisd's static fields (crashes the manager, caught immediately by `wazuh-analysisd -t`) and the Wazuh index template's pre-mapped field types (silently drops documents at the indexer, only visible in Filebeat's own log file, not in Wazuh's).

7. **Rule group tags didn't propagate to child rules.** Dashboard filters using `rule.groups: forcepoint_dlp_incident` returned zero results even for genuine incident alerts. The group tag had only been added to the top-level incident-detection rule (100201); the actual severity/action-specific rules that fire in practice (100210/100211/100212/100220) never inherited it, since Wazuh doesn't propagate a `<group>` tag through an `<if_sid>` chain automatically. Fixed by adding the group tag explicitly to every rule that can actually be the one reported as "fired."

8. **Terms aggregation collapsed into a single "All docs" bucket.** Dynamic string field mappings create both an analyzed `text` version and a `.keyword` sub-field; Kibana/OpenSearch Dashboards Terms aggregations need the `.keyword` variant to bucket correctly. Selecting plain `data.category` instead of `data.category.keyword` silently produced one fallback bucket instead of splitting by actual category values.

## Result

All three Forcepoint DLP log types (incident, audit, system) decode correctly, generate appropriately-leveled Wazuh alerts, and are indexed and dashboarded. Rule levels are tuned so routine system/audit noise stays below the email alert threshold, while HIGH-severity blocked incidents both alert and email.

## Dashboard

Built a **"Forcepoint DLP Overview"** dashboard in Wazuh (OpenSearch Dashboards) with four panels:
- Blocked vs Allowed incidents over time (stacked area, split by `data.dlp_action`)
- Incidents by policy category (bar chart, `data.category.keyword`)
- Severity breakdown (pie chart, `data.severity_type`)
- Top source users (data table, `data.source_user`)

## Next Steps

- Extend rule coverage: currently only `POLICY_MNG` category audit events get a dedicated escalation rule (100230) — add rules for other audit categories as they're observed
- Feeds directly into the existing "test DLP with sample exfiltration attempts" future-improvement item — the alerting pipeline this documents is the prerequisite for that test to be observable/measurable
- Consider adding a scheduled Wazuh report or alert-based notification for HIGH severity blocks, beyond the existing email
