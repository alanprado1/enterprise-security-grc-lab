# Phase 1 — Architecture & Design

## Objective

Design a network topology that safely isolates a deliberately vulnerable honeypot from the rest of the lab, while still allowing legitimate telemetry to flow out to a cloud SIEM. Every design decision here was made before a single VM was built — the goal was to avoid discovering segmentation gaps after the fact.

## Network topology

The lab runs four isolated network zones inside VMware, connected through a single pfSense firewall. Only Microsoft Sentinel sits outside the VMware host, in Azure.

```
                        ┌─────────────────────────┐
                        │   Simulated Internet     │
                        │   10.0.0.0/24            │
                        │   (Kali Linux)           │
                        └────────────┬────────────┘
                                     │
                        ┌────────────▼────────────┐
                        │      pfSense Firewall    │
                        │   4 interfaces, least-   │
                        │   privilege rule set     │
                        └──┬──────────┬─────────┬──┘
                           │          │         │
            ┌──────────────▼─┐  ┌─────▼──────┐ ┌▼──────────────┐
            │      DMZ        │  │  Internal  │ │  Management   │
            │ 192.168.10.0/24 │  │192.168.20.0│ │192.168.30.0/24│
            │  (Cowrie)       │  │(Win Server)│ │  (Shuffle)    │
            └─────────────────┘  └────────────┘ └───────────────┘
                                                         │
                                                  ┌──────▼──────┐
                                                  │    Azure     │
                                                  │   Sentinel   │
                                                  │ Log Analytics│
                                                  └──────────────┘
```

## Why four zones instead of two
