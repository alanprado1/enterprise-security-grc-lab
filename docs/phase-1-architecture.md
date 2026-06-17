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
- **DMZ** hosts only the honeypot — nothing else. If it were shared with other services, a compromise of the honeypot would put those services at risk too.
- **Internal zone** simulates a real corporate endpoint, generating legitimate background telemetry (Windows Security Events) that a SOC analyst would need to distinguish from attacker activity.
- **Management zone** is isolated because in a real enterprise, security tooling (SOAR platforms, SIEM collectors) sits on a hardened network that ordinary user traffic — and certainly internet traffic — should never be able to reach.
