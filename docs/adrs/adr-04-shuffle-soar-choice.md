
# ADR-04 - Shuffle SOAR + Python integration layer

**Date:** Phase 2 - Build & Configuration (revised during Phase 3)  
**Status:** Accepted with documented deviation

## Context

A SOAR platform was required to automate incident response - specifically to acknowledge new Sentinel incidents, enrich attacker IPs with threat intelligence, and post automated comments. The original design called for Shuffle SOAR to execute this workflow natively.

## Options considered

**Shuffle SOAR** is an open-source SOAR platform deployable via Docker Compose. It has a visual workflow editor, pre-built app integrations including Microsoft Sentinel, and a webhook trigger system.

**Custom Python scripts** calling the Sentinel REST API directly - no dedicated SOAR platform.

**Commercial SOAR platforms** (Splunk SOAR, Palo Alto XSOAR) - eliminated immediately due to cost (tens of thousands of dollars annually).

## Decision

**Shuffle SOAR deployed as the orchestration platform, with Python scripts as the execution layer.**

## Reasoning

Shuffle was installed and configured successfully. The Orborus worker - Shuffle's execution engine - requires Docker Swarm to distribute workflow execution containers across nodes. On a single-VM deployment, Docker Swarm's overlay network configuration created persistent connectivity issues between the Orborus container and the backend API, preventing reliable workflow execution.

Rather than remove Shuffle entirely (which would require updating all diagrams, documentation, and the mind map), the architecture was revised to a hybrid model:

- **Shuffle** remains deployed and documents the intended production workflow design - the visual workflow canvas shows the logical sequence (Get Token → Get Incidents → Post Comment) that a hiring manager or architect would expect to see
- **Python scripts** implement the same logic directly against the Sentinel REST API, providing reliable execution without Swarm dependency

This approach is honest about the lab constraint while demonstrating knowledge of both SOAR platform design and direct API integration - a combination that reflects how production SOAR environments actually work (platforms augmented by custom integrations for complex or non-standard workflows).

## Consequences

The Python scripts are tightly coupled to the Sentinel REST API version (`2023-02-01`). An API version change would require script updates. In production, the Shuffle workflow would be the maintainable long-term path once Orborus worker connectivity is resolved by adding a second node or switching to a managed Shuffle Cloud deployment.
