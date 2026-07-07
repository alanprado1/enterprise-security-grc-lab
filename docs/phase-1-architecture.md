# Phase 1: Architecture & Design

## Objective

Design a network topology that safely isolates a deliberately vulnerable honeypot from the rest of the lab, while still allowing legitimate telemetry to flow out to a cloud SIEM. Every design decision here was made before a single VM was built, the goal was to avoid discovering segmentation gaps after the fact.

## Network topology

The lab runs four isolated network zones inside VMware, connected through a single pfSense firewall. Only Microsoft Sentinel sits outside the VMware host, in Azure.

```
                          ┌──────────────────────────┐
                          │   Simulated Internet     │
                          │   10.0.0.0/24            │
                          │   (Kali Linux)           │
                          └────────────┬─────────────┘
                                       │
                          ┌────────────▼─────────────┐
                          │      pfSense Firewall    │
                          │   4 interfaces, least-   │
                          │   privilege rule set     │
                          └──┬──────────┬─────────┬──┘
                             │          │         │
            ┌────────────────▼┐  ┌──────▼─────┐ ┌─▼─────────────┐
            │      DMZ        │  │  Internal  │ │  Management   │
            │ 192.168.10.0/24 │  │192.168.20.0│ │192.168.30.0/24│
            │  (Cowrie)       │  │(Win Server)│ │  (Shuffle)    │
            └─────────────────┘  └────────────┘ └───────────────┘
                                                         │
                                                  ┌──────▼───────┐
                                                  │    Azure     │
                                                  │   Sentinel   │
                                                  │ Log Analytics│
                                                  └──────────────┘
```

## Why four zones instead of two

A simpler design might use just a DMZ and an internal zone. Four zones were chosen deliberately:

- **Simulated Internet zone** exists so the attacking machine (Kali) is architecturally identical to a real external attacker, without ever touching a real public IP or risking an ISP abuse complaint for generating brute-force traffic on a residential connection.
- **DMZ** hosts only the honeypot, nothing else. If it were shared with other services, a compromise of the honeypot would put those services at risk too.
- **Internal zone** simulates a real corporate endpoint, generating legitimate background telemetry (Windows Security Events) that a SOC analyst would need to distinguish from attacker activity.
- **Management zone** is isolated because in a real enterprise, security tooling (SOAR platforms, SIEM collectors) sits on a hardened network that ordinary user traffic, and certainly internet traffic should never be able to reach.

## IP address scheme

| Zone | Subnet | Gateway (pfSense) | Hosts |
|---|---|---|---|
| Simulated Internet | 10.0.0.0/24 | 10.0.0.1 | Kali - 10.0.0.10 |
| DMZ | 192.168.10.0/24 | 192.168.10.1 | Cowrie - 192.168.10.10 |
| Internal | 192.168.20.0/24 | 192.168.20.1 | Windows Server - 192.168.20.10 |
| Management | 192.168.30.0/24 | 192.168.30.1 | Shuffle SOAR - 192.168.30.10 |

The Simulated Internet zone deliberately uses a different address class (10.x rather than 192.168.x) so that, when reviewing logs or diagrams later, it's immediately visually obvious which traffic originates from the "outside".

## Firewall design principles

Two rules govern every firewall decision made throughout the build:

**Zero-egress from the DMZ.** The honeypot can receive connections, but cannot initiate any outbound connection of its own, anywhere. This is the single most important control in the lab, it guarantees that even a fully compromised honeypot cannot be used to pivot, exfiltrate data, or contact a real command-and-control server.

**Least privilege between every other zone.** No zone is permitted to reach another zone by default. Specific allow rules were added only where a real operational need existed, for example, the Internal zone is allowed outbound on port 443 specifically so Windows Security Events can reach Sentinel, and nothing else.

## VM resource budget

Planned against a 16GB RAM host before any VM was built:

| VM | Role | RAM allocated |
|---|---|---|
| pfSense | Firewall / router | 512 MB |
| Cowrie (Ubuntu) | Honeypot - DMZ | 512 MB |
| Windows Server 2025 | Internal endpoint telemetry | 2 GB |
| Kali Linux | Simulated attacker | 2 GB |
| Shuffle SOAR (Ubuntu) | SOAR orchestration | 4.9 GB |
| **Total** | | **~9.9 GB**, leaving headroom for the VMware host process and Windows 11 itself |

## MITRE ATT&CK techniques in scope

Mapped at the design stage so that the attack simulation in Phase 3 would have specific, named techniques to demonstrate rather than vague "attack testing":

- **T1110** - Brute Force (Kali performs, Cowrie detects)
- **T1595** - Active Scanning (Kali performs, Cowrie/pfSense detects)
- **T1046** - Network Service Discovery (Kali performs, Cowrie/pfSense detects)
- **T1562** - Impair Defenses (simulated firewall rule failure scenario)

## ISO 27001 controls mapped at design stage

- **A.8.20 Network Security** → satisfied by the pfSense segmentation and zero-egress policy
- **A.8.16 Monitoring Activities** → satisfied by the Cowrie → Sentinel log pipeline
- **A.8.7 Protection Against Malware** → satisfied by Windows Defender running on the Windows Server VM

Full evidence for each control is documented in [Phase 4 - GRC Mapping](phase-4-grc.md).

## Outcome of this phase

A finalized network diagram, IP scheme, firewall policy, and resource budget, all decided before build work began. This avoided the most common failure mode in lab projects: discovering a segmentation gap or resource shortfall halfway through the build and having to retrofit the design.

Continue to [Phase 2 - Build & Configuration](phase-2-build.md)
