# ADR-01 - Cowrie over T-Pot

**Date:** Phase 1 - Architecture & Design  
**Status:** Accepted

## Context

A honeypot was required for the DMZ zone to emulate vulnerable services, capture attacker sessions, and generate structured logs for Sentinel ingestion. Two options were evaluated.

## Options considered

**T-Pot** is a full multi-honeypot platform that bundles approximately 20 honeypot daemons (including Cowrie) plus a complete ELK stack for local log visualisation. It is highly capable and visually impressive.

**Cowrie** is a focused SSH and Telnet honeypot daemon. It captures full session TTY recordings, logs every login attempt and command in structured JSON, and has a well-documented API for log output.

## Decision

**Cowrie** was selected.

## Reasoning

T-Pot requires a dedicated VM with 8–16GB RAM to run its bundled ELK stack alongside the honeypot daemons. On a 16GB host running five other VMs, this would exhaust the available memory budget entirely and make stable operation of the full lab impossible.

Cowrie on Ubuntu runs comfortably in 512MB RAM - leaving the remaining VM budget available for pfSense, Windows Server, Kali, and Shuffle SOAR.

Beyond the resource constraint, Cowrie alone satisfies every detection requirement for this lab: SSH and Telnet emulation, session recording, structured JSON logs, and configurable credential acceptance. T-Pot's additional honeypot daemons (HTTP, SMB, RDP emulators) would add attack surface without adding proportional value to the detection scenarios being tested.

The ELK stack bundled in T-Pot is also redundant given that Microsoft Sentinel serves as the SIEM - running two log aggregation platforms in the same lab would create complexity without benefit.

## Consequences

Cowrie covers SSH (T1110, T1595) and Telnet attack scenarios. HTTP, SMB, and RDP honeypot coverage is out of scope for this lab. In a production deception deployment, a platform like T-Pot or a commercial alternative would be appropriate where resource constraints don't apply.
