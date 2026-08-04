# Network-Service-Scanning-Investigation-T1046-
This project demonstrates the development and validation of a custom Wazuh detection rule for identifying network service scanning activity.
# Wazuh Detection Engineering: Network Service Scanning (MITRE T1046)

## Overview

This project demonstrates the development and validation of a custom Wazuh detection rule for identifying network service scanning activity.

A Kali Linux VM was used to simulate adversary reconnaissance against an Ubuntu endpoint using Nmap. Default Wazuh detection coverage was analyzed, a visibility gap was identified, and custom detection logic was developed using Ubuntu UFW firewall telemetry.

The objective was to improve visibility into MITRE ATT&CK **T1046 — Network Service Scanning**.

---

# Environment

| Component | Role |
|---|---|
| Kali Linux | Adversary simulation node |
| Ubuntu 22.04 | Target endpoint with Wazuh agent |
| Wazuh OVA | SIEM for log collection and detection |

---

# Phase 1 — Adversary Simulation

## Technique

**MITRE ATT&CK: T1046 — Network Service Scanning**

## Attack Simulation

A network reconnaissance scan was executed from Kali Linux:

```bash
nmap -sS -sV 192.168.56.106
```

## Result

The scan successfully identified exposed services on the Ubuntu endpoint.

| Port | Service | Version |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.9p1 |

![Nmap Scan](screenshots/nmap_scan.png)

---

# Phase 2 — Default Wazuh Detection Analysis

## Objective

Evaluate whether the default Wazuh deployment detects network reconnaissance activity.

## Findings

Following the Nmap scan, Wazuh Threat Hunting was reviewed.

Searches performed:

- Attacker IP
- Victim IP
- `nmap`
- `scan`

No alerts were generated for the reconnaissance activity.

Default Wazuh telemetry captured normal authentication activity but failed to identify the scanning behavior.

![Default Wazuh Results](screenshots/wazuh_default_detection_gap.png)

---

# Detection Gap Identified

## Gap

The default Wazuh ruleset collected endpoint telemetry but did not detect network service scanning activity.

## Impact

An analyst relying only on default alerts would miss early-stage reconnaissance activity against the endpoint.

## Detection Requirement

A detection capability was required to identify:

- Multiple TCP SYN attempts
- Multiple destination ports
- Blocked firewall connections
- Source IP attribution

---

# Phase 3 — Detection Engineering

## Telemetry Source

Ubuntu UFW firewall logs were identified as the detection source.

Example event:

```
[UFW BLOCK] SRC=192.168.56.102 DST=192.168.56.106 PROTO=TCP DPT=443 SYN
```

The event showed:

- Source: Kali Linux (`192.168.56.102`)
- Destination: Ubuntu endpoint (`192.168.56.106`)
- Protocol: TCP
- Action: Firewall blocked connection

![UFW Telemetry](screenshots/ufw_logs.png)

---

# Custom Wazuh Detection Rule

Rule location:

```
/var/ossec/etc/rules/local_rules.xml
```

Detection logic:

```xml
<rule id="100100" level="10">
  <match>UFW BLOCK</match>
  <description>
    Possible Network Service Scanning Detected - Multiple UFW blocked connection attempts
  </description>
  <mitre>
    <id>T1046</id>
  </mitre>
</rule>
```

The complete rule file is available:

```
rules/local_rules.xml
```

---

# Rule Validation

The rule was tested using Wazuh log testing:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

The UFW event successfully matched:

```
Rule ID: 100100
Severity: Level 10
MITRE: T1046
```

![Wazuh Logtest Validation](screenshots/wazuh_logtest.png)

---

# Phase 4 — Operational Validation

The Nmap scan was executed again:

```bash
nmap -sS -sV 192.168.56.106
```

The Wazuh dashboard generated alerts using the custom rule.

![Wazuh Alert](screenshots/wazuh_alert.png)

The validation confirmed that the custom detection closed the visibility gap identified during baseline testing.

---

# Phase 5 — Analyst Investigation

## Alert Summary

| Field | Value |
|---|---|
| Detection | Possible Network Service Scanning Detected |
| Rule ID | 100100 |
| Severity | Level 10 |
| MITRE ATT&CK | T1046 |
| Source | Kali Linux |
| Target | Ubuntu Endpoint |

---

## Investigation Process

Reviewed:

- Source IP address
- Target endpoint
- Firewall telemetry
- Scanned ports
- Timeline correlation

The source IP was confirmed as the authorized Kali Linux testing VM.

---

# Analyst Assessment

Classification:

```
True Positive — Authorized Activity
```

The detection correctly identified behavior matching MITRE ATT&CK T1046. The activity was expected because it was generated during an authorized security testing exercise.

---

# Recommended Actions

For a production environment:

- Validate whether scanning activity is authorized
- Investigate unknown scanning sources
- Restrict unnecessary exposed services
- Monitor repeated scanning attempts
- Add correlation rules for high-volume reconnaissance

---

# Skills Demonstrated

- Wazuh SIEM administration
- Detection engineering
- MITRE ATT&CK mapping
- Linux firewall telemetry analysis
- SOC alert triage
- Network reconnaissance analysis
- Custom rule development
