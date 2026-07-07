# ADR-02 - Microsoft Sentinel over Wazuh

**Date:** Phase 1 - Architecture & Design  
**Status:** Accepted

## Context

A SIEM was required to centralise logs from Cowrie and Windows Server, run correlation rules, generate incidents, and provide a dashboard for SOC analyst and executive views. Two options were evaluated.

## Options considered

**Wazuh** is an open-source SIEM and XDR platform. It is self-hosted, free, and well-regarded for on-premises deployments.

**Microsoft Sentinel** is a cloud-native SIEM built on Azure Log Analytics. It has a 90-day free trial and a free tier of 5GB/month log ingestion.

## Decision

**Microsoft Sentinel** was selected.

## Reasoning

The lab already had an existing Azure environment with a prior honeypot project. Reusing that Azure tenant avoided provisioning a new platform from scratch.

More importantly, Microsoft Sentinel is the enterprise standard SIEM in a significant portion of the market, particularly in organisations running Microsoft 365 and Azure workloads. Demonstrating hands-on Sentinel experience (KQL queries, analytics rules, entity mapping, incident management) has direct portfolio value for roles in those environments. Wazuh experience, while valid, is more niche.

Sentinel also eliminates the need for a dedicated SIEM VM. Wazuh requires 4GB RAM minimum for a functional installation, running it locally would have consumed a quarter of the available VM memory budget. Sentinel runs entirely in Azure at no compute cost to the lab host.

The 5GB/month free ingestion tier is sufficient for the lab's log volume when the forwarder scripts are configured to send only alerts and session events rather than raw syslog.

## Consequences

All log data leaves the VMware host and traverses the internet to Azure. This is an accepted trade-off given the lab context. In a high-security production environment, a local SIEM would be preferable to avoid log data leaving the network perimeter. A hybrid approach (local Wazuh forwarding to Sentinel) would be the production-grade solution.
