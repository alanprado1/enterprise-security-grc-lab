# Phase 4 - GRC Mapping & Documentation

## Objective

Map every technical control implemented in this lab to the corresponding ISO 27001:2022 Annex A control, document the risk register with likelihood/impact assessments, and produce evidence packages suitable for an internal audit. The goal was to demonstrate that security engineering and governance aren't separate disciplines, every firewall rule, every log pipeline, and every detection rule has a corresponding compliance rationale.

## ISO 27001 control mapping

Three controls were selected as the primary focus because they map directly and demonstrably to the lab's technical implementations.

---

### A.8.20 - Network Security

**Control statement:** Networks and network services shall be secured, managed and controlled to protect information and systems.

**Implementation in this lab:**

The pfSense firewall enforces strict network segmentation across four zones, Simulated Internet, DMZ, Internal, and Management. Each zone has explicitly defined ingress and egress rules based on the principle of least privilege. No zone can communicate with another unless a specific allow rule exists.

The most critical implementation is the **zero-egress policy** on the DMZ zone. The honeypot VM (Cowrie) is permitted to receive inbound connections on ports 22 and 23, but is blocked from initiating any outbound connection to any destination. This ensures that even a fully compromised honeypot cannot be used as a pivot point, cannot exfiltrate data, and cannot contact a command-and-control server.

**Evidence artifacts:**
- pfSense configuration XML export `configs/pfsense-config-export.xml`
- Screenshots of all four zone rule sets `evidence/screenshots/03` through `06`
- Network topology diagram, Phase 1 documentation

**Gap analysis:** The lab uses a single pfSense instance with no redundancy. In a production environment, a high-availability pair would be required. This is documented as a known gap acceptable for a lab context.

---

### A.8.16 - Monitoring Activities

**Control statement:** Networks, systems and applications shall be monitored for anomalous behaviour and appropriate actions taken to evaluate potential information security incidents.

**Implementation in this lab:**

Microsoft Sentinel serves as the centralised SIEM, ingesting logs from two sources:

1. **Cowrie honeypot** (`CowrieLogs_CL`) - every SSH and Telnet session, every login attempt (successful and failed), every command executed inside the fake shell, and every connection event is captured in structured JSON and forwarded to Sentinel within 60 seconds.

2. **Windows Server** (`WindowsSecurityEvents_CL`) - security events covering logon activity, account management, privilege use, and process tracking are forwarded every minute.

Five scheduled analytics rules correlate raw log data into high-fidelity incidents, covering brute force detection, successful honeypot logins, port scanning, and firewall egress violations.

An automated SOAR response script acknowledges every new incident within 5 minutes, posts a comment with threat intelligence enrichment, and tracks all processed incidents.

**Evidence artifacts:**
- Sentinel workspace screenshot `evidence/screenshots/18`
- Cowrie logs ingested in Sentinel `evidence/screenshots/19`, `20`
- Analytics rules list `evidence/screenshots/23`
- Incident creation `evidence/screenshots/25`
- SOAR automated comments verified `evidence/screenshots/26`
- Threat intel enrichment `evidence/screenshots/38`

**Gap analysis:** Log ingestion uses the legacy HTTP Data Collector API rather than the current Azure Monitor Agent with Data Collection Rules. The API remains fully supported and the data is identical in Sentinel. Migration to AMA/DCR would be required before a formal ISO 27001 audit in a production environment.

---

### A.8.7 - Protection Against Malware

**Control statement:** Protection against malware shall be implemented and supported by appropriate user awareness.

**Implementation in this lab:**

Windows Defender is active and fully configured on the Windows Server 2025 VM (`INTERNAL-SRV01`). Virus and threat protection shows no action required. Windows Defender provides real-time protection, cloud-delivered protection, and automatic sample submission.

Audit policy is configured to capture all security-relevant events including process creation (Detailed Tracking), which would detect malware execution attempts on the endpoint.

**Evidence artifacts:**
- Windows Defender active status screenshot `evidence/screenshots/10`
- Audit policy configuration screenshot `evidence/screenshots/12`

**Gap analysis:** Only one endpoint is present in the lab. A production environment would require endpoint protection deployed across all systems with centralised management and alerting.

---

## STRIDE threat model

Applied to the two highest-value assets in the lab:

### Cowrie honeypot (intentionally exposed)

| Threat | Category | Mitigation |
|---|---|---|
| Attacker escapes honeypot and pivots to internal zone | Elevation of Privilege | Zero-egress rule + Block DMZ→INTERNAL firewall rule |
| Attacker poisons honeypot logs | Tampering | Logs forwarded immediately, attacker cannot modify already-forwarded data |
| Attacker performs DoS against honeypot | Denial of Service | Cowrie handles high connection volume by design; pfSense rate limiting available |
| Honeypot used as C2 relay | Spoofing | Zero-egress blocks all outbound from DMZ |

### Microsoft Sentinel workspace

| Threat | Category | Mitigation |
|---|---|---|
| Unauthorised access to log data | Information Disclosure | Azure RBAC, Service Principal has minimum required permissions only |
| Log data tampered with post-ingestion | Tampering | Log Analytics workspace data is immutable once ingested |
| Service Principal credentials compromised | Elevation of Privilege | Credentials stored only on Cowrie VM; rotate periodically |

---

## MITRE ATT&CK technique mapping

| Technique | Name | Simulated by | Detected by |
|---|---|---|---|
| T1110 | Brute Force | Hydra (Kali) | Cowrie login.failed events → Sentinel Rule 1 |
| T1110.001 | Password Guessing | Hydra with rockyou.txt | Same as T1110 |
| T1595 | Active Scanning | Nmap (Kali) | Cowrie session.connect events → Sentinel Rule 4 |
| T1595.001 | Scanning IP Blocks | Nmap full port scan | Same as T1595 |
| T1046 | Network Service Discovery | Nmap service detection | Same as T1595 |
| T1562 | Impair Defenses | pfSense rule disabled (simulated) | Firewall logs + Sentinel Rule 5 |
| T1562.004 | Disable or Modify System Firewall | Zero-egress rule temporarily disabled | pfSense firewall log |

---

## Risk register

| Risk ID | Description | Likelihood | Impact | Rating | Mitigation | Residual risk |
|---|---|---|---|---|---|---|
| R-01 | Honeypot lateral movement to internal zone | Low | Critical | High | Zero-egress + DMZ→INTERNAL block rule | Low |
| R-02 | Sentinel log ingestion cost overrun | Medium | Low | Low | Forward alerts only, not raw events; budget alert at $10/month | Low |
| R-03 | VM resource exhaustion on host | Low | Medium | Low | RAM budget planned before build; Shuffle capped at 4.9GB | Low |
| R-04 | Service Principal credentials exposed | Medium | High | High | Credentials stored on isolated VM; rotate periodically | Medium |
| R-05 | pfSense single point of failure | Low | High | Medium | Acceptable for lab; HA pair required in production | Medium |
| R-06 | Log forwarding script failure (silent) | Medium | Medium | Medium | Cron logs to file; position file detects rotation | Low |

---

## Asset register

| Asset | Classification | Zone | Owner | Notes |
|---|---|---|---|---|
| Cowrie VM | Deception asset, intentionally exposed | DMZ | Lab | Deliberately vulnerable; isolated by design |
| Windows Server VM | Internal endpoint | Internal | Lab | Production-equivalent telemetry source |
| Microsoft Sentinel workspace | Critical security infrastructure | Azure | Lab | Primary detection and alerting platform |
| pfSense firewall | Critical infrastructure | Perimeter | Lab | Single point of failure, check R-05 |
| Shuffle SOAR VM | Security tooling | Management | Lab | Automated response platform |
| Kali Linux VM | Attack simulation tool | Simulated Internet | Lab | Used only for controlled attack scenarios |

---

## Architecture Decision Records

Five ADRs document the reasoning behind every major tool selection:

- [ADR-01 - Cowrie over T-Pot](adrs/adr-01-cowrie-over-tpot.md)
- [ADR-02 - Sentinel over Wazuh](adrs/adr-02-sentinel-over-wazuh.md)
- [ADR-03 - pfSense over OPNsense](adrs/adr-03-pfsense-over-opnsense.md)
- [ADR-04 - Shuffle SOAR + Python integration layer](adrs/adr-04-shuffle-soar-choice.md)
- [ADR-05 - Manual provisioning over IaC](adrs/adr-05-manual-provisioning.md)

---

## Incident response playbooks

Two playbooks document the response procedures for the lab's primary attack scenarios:

- [Playbook 01 - SSH Brute Force Response](playbooks/playbook-01-brute-force.md)
- [Playbook 02 - Firewall Egress Failure Response](playbooks/playbook-02-firewall-failure.md)

---

## Summary

This lab demonstrates that ISO 27001 compliance is not a paper exercise separate from engineering. it is the engineering, documented. Every control in this mapping has a working technical implementation behind it, every gap is acknowledged honestly, and every risk has a documented mitigation and residual risk rating.

The combination of network segmentation (A.8.20), continuous monitoring (A.8.16), and endpoint protection (A.8.7) forms a defence-in-depth posture where a single control failure does not result in a total compromise, the honeypot can be fully owned by an attacker and the rest of the lab remains protected.
