# Enterprise Security Architecture, Deception & GRC Lab

A hybrid security lab combining hands-on network engineering with a governance framework, built to demonstrate the skill set required for Cybersecurity Solutions Architect and Senior Security Engineer roles. The lab bridges deception technology, SIEM engineering, SOAR automation, and ISO 27001 compliance mapping into a single, cohesive build.

## What this lab demonstrates

A simulated attacker probes a deliberately exposed honeypot, the honeypot's activity is captured and forwarded to a cloud SIEM, the SIEM correlates and alerts, and an automated response layer acknowledges and acts on the incident. Every technical decision is mapped back to a governance control, and every governance control has a working technical implementation behind it.

| Domain | What's built |
|---|---|
| Network architecture | Segmented VMware lab — DMZ, internal zone, management zone, and a simulated internet zone — enforced by a pfSense firewall with a zero-egress policy isolating the honeypot |
| Deception technology | Cowrie honeypot emulating SSH and Telnet, with full session capture and structured JSON logging |
| Detection & SIEM | Microsoft Sentinel ingesting honeypot and Windows Security Event logs, with custom KQL analytics rules and entity-mapped incidents |
| SOAR automation | Automated incident response via direct Sentinel REST API integration, with Shuffle SOAR deployed as the orchestration platform |
| GRC | ISO 27001 control mapping (A.8.20, A.8.16, A.8.7), STRIDE threat modelling, MITRE ATT&CK technique mapping, and a maintained risk register |

## Architecture

The lab runs entirely on a single VMware Workstation Pro host (16GB RAM), with Microsoft Sentinel as the only cloud component. No compute runs in Azure, only logs leave the lab environment, and only over encrypted HTTPS.

```
Simulated Internet (Kali)
        │
        ▼
   pfSense Firewall ── 4 network zones, least-privilege rules
        │
   ┌────┴────┬─────────────┬──────────────┐
   ▼          ▼             ▼              ▼
  DMZ      Internal      Management    (Azure: Sentinel,
(Cowrie)  (Win Server)   (Shuffle)      Log Analytics)
```

Full network diagram and IP scheme: [`docs/phase-1-architecture.md`](docs/phase-1-architecture.md)

## Documentation

The build is documented phase by phase, matching how it was actually executed:

- [Phase 1 — Architecture & Design](docs/phase-1-architecture.md)
- [Phase 2 — Build & Configuration](docs/phase-2-build.md)
- [Phase 3 — Detection & SOAR](docs/phase-3-detection.md)
- [Phase 4 — GRC Mapping & Documentation](docs/phase-4-grc.md)

Architecture decisions are recorded individually:

- [ADR-01 — Cowrie over T-Pot](docs/adrs/adr-01-cowrie-over-tpot.md)
- [ADR-02 — Sentinel over Wazuh](docs/adrs/adr-02-sentinel-over-wazuh.md)
- [ADR-03 — pfSense over OPNsense](docs/adrs/adr-03-pfsense-over-opnsense.md)
- [ADR-04 — Shuffle SOAR + Python integration layer](docs/adrs/adr-04-shuffle-soar-choice.md)
- [ADR-05 — Manual provisioning over IaC](docs/adrs/adr-05-manual-provisioning.md)

Incident response playbooks:

- [Playbook 01 — SSH Brute Force Response](docs/playbooks/playbook-01-brute-force.md)
- [Playbook 02 — Firewall Egress Failure Response](docs/playbooks/playbook-02-firewall-failure.md)

## Skills demonstrated

| Skill area | Evidence |
|---|---|
| Network security & segmentation | pfSense rule exports, zero-egress policy, topology diagram |
| Deception technology | Cowrie session logs, TTY recordings, dwell time metrics |
| SIEM engineering | Custom KQL analytics rules, log pipeline, entity-mapped incidents |
| SOAR automation | Python-based Sentinel REST API integration, automated incident response |
| Threat intelligence | OTX and AbuseIPDB IOC enrichment |
| GRC & compliance | ISO 27001 control mapping, risk register, STRIDE threat model |
| Threat modelling | MITRE ATT&CK technique mapping (T1110, T1595, T1046, T1562) |
| Incident response | Documented playbooks, tabletop exercise, after-action report |
| Architectural reasoning | 5 ADRs covering every major tool decision |

## Tech stack

VMware Workstation Pro · pfSense CE · Cowrie · Kali Linux · Windows Server 2025 · Microsoft Sentinel · Shuffle SOAR · Python · AlienVault OTX · AbuseIPDB

## About this project

Built as a portfolio project to demonstrate practical, end-to-end security engineering — from network design through to governance reporting — on a near-zero budget using free-tier cloud services and open-source tooling.

