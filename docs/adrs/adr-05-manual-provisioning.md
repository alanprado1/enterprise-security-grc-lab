# ADR-05 - Manual provisioning over Infrastructure as Code

**Date:** Phase 1 - Architecture & Design  
**Status:** Accepted

## Context

The lab required provisioning five VMs, configuring network interfaces, installing software, and managing Azure resources. Infrastructure as Code (IaC) tools like Terraform and Ansible were considered as alternatives to manual provisioning.

## Options considered

**Terraform** for Azure resource provisioning (Log Analytics workspace, resource groups, analytics rules) and potentially for VMware VM creation via the vSphere provider.

**Ansible** for configuration management - automating package installation, service configuration, and network setup across VMs.

**Manual provisioning** with documented procedures.

## Decision

**Manual provisioning** was selected, with IaC noted as the production equivalent.

## Reasoning

Terraform and Ansible are separate skill sets from the security engineering work this lab is designed to demonstrate. Implementing them properly - writing HCL for Azure resources, managing Terraform state, writing idempotent Ansible playbooks - would have consumed 30–40% of the build timeline without directly advancing the lab's security objectives.

More importantly, manual provisioning with detailed documentation produces a better learning artifact. Every configuration decision is visible, explainable, and documented with the reasoning behind it. IaC abstracts those decisions into code that is correct but not always readable to someone reviewing a portfolio.

The lab documentation explicitly notes at each configuration step what the production IaC equivalent would look like - for example, the pfSense firewall rules would be managed as Terraform resources using the pfSense provider, and the Cowrie systemd service would be an Ansible task in a honeypot role.

## Consequences

The lab is not reproducible via a single command. Rebuilding it requires following the Phase 2 documentation step by step. In a production or team environment, this would be unacceptable - IaC would be mandatory for reproducibility, auditability, and change management. This gap is acknowledged and documented.
