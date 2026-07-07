# ADR-03 - pfSense over OPNsense

**Date:** Phase 1 - Architecture & Design  
**Status:** Accepted

## Options considered

**pfSense CE** and **OPNsense** are both open-source FreeBSD-based firewall/router distributions. They are functionally very similar - both support VLAN tagging, multi-interface routing, stateful packet inspection, and a web-based management UI.

## Decision

**pfSense CE** was selected.

## Reasoning

The deciding factor was community documentation volume. pfSense has a significantly larger body of lab-specific tutorials, VMware configuration guides, and troubleshooting resources. When configuring a multi-interface firewall in a VMware environment for the first time, the availability of precise, tested documentation reduces build time and the risk of misconfiguration.

OPNsense has been gaining ground and has a strong community, but pfSense's documentation advantage is meaningful in a time-constrained lab build.

Both platforms would satisfy the technical requirements of this lab equally well.

## Consequences

Netgate (pfSense's commercial backer) has been moving features to the paid pfSense Plus version. The CE version used here remains fully functional for all lab requirements but may diverge further from the commercial offering over time. OPNsense may be the better long-term choice for a lab that needs to stay current with open-source features.
