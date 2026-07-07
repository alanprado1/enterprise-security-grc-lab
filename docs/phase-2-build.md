# Phase 2 - Build & Configuration

## Objective

Provision all VMs in dependency order, configure network interfaces and static IPs, and verify connectivity between each zone before moving to detection configuration. Every VM was built manually, check [ADR-05](adrs/adr-05-manual-provisioning.md) for the reasoning behind choosing manual provisioning over IaC for this lab.

## Build order and rationale

VMs were provisioned in strict dependency order:

1. **VMware virtual networks**:LAN Segments must exist before any VM is created
2. **pfSense**:all other VMs need a gateway and firewall rules to exist first
3. **Cowrie (Ubuntu)**:honeypot configured once its DMZ gateway exists
4. **Windows Server**:internal endpoint configured once its gateway exists
5. **Kali Linux**:attack VM, no dependencies other than the SIMINTERNET gateway
6. **Shuffle SOAR (Ubuntu)**:management zone, configured last

## VMware network configuration

Rather than VMware's Host-only networks, the lab uses **LAN Segments**, a VMware feature that creates direct VM-to-VM connections bypassing the VMware virtual switch infrastructure entirely. This was necessary due to a known issue with Host-only network traffic passing between VMs on some Windows 11 configurations.

Four LAN Segments were created:

| LAN Segment | Zone | Subnet |
|---|---|---|
| LAB-SIMINTERNET | Simulated Internet | 10.0.0.0/24 |
| LAB-DMZ | DMZ / Honeypot | 192.168.10.0/24 |
| LAB-INTERNAL | Internal | 192.168.20.0/24 |
| LAB-MGMT | Management | 192.168.30.0/24 |

VMnet8 (NAT) was temporarily added to each VM during setup to allow internet access for package installation. It was removed from Windows Server and Kali once configuration was complete. It remains on Cowrie and Shuffle as the log forwarding path to Azure, see the design deviation note below.

> **Design deviation Cowrie internet path:** The original design called for Cowrie to forward logs through pfSense's INTERNAL zone, keeping VMnet8 fully removed. In practice, the zero-egress firewall rule on the DMZ blocks all outbound connections from Cowrie including Azure log forwarding calls. Rather than create a specific egress exception that weakens honeypot isolation, VMnet8 (ens34) was retained on Cowrie as a dedicated, isolated log-forwarding path. This is an accepted trade-off: the honeypot remains isolated from all lab zones via pfSense, while retaining a separate out-of-band internet path for telemetry only.

## pfSense firewall

**Version:** pfSense CE 2.8.1

**Interface assignments:**

| Interface | Name | Adapter | IP |
|---|---|---|---|
| WAN | WAN_TEMP | VMnet8 (NAT) | DHCP 192.168.81.x |
| LAN | SIMINTERNET | LAB-SIMINTERNET | 10.0.0.1/24 |
| OPT1 | DMZ | LAB-DMZ | 192.168.10.1/24 |
| OPT2 | INTERNAL | LAB-INTERNAL | 192.168.20.1/24 |
| OPT3 | MGMT | LAB-MGMT | 192.168.30.1/24 |

**Firewall rules implemented:**

DMZ zone rules (controlling traffic originating from Cowrie):
- Block all outbound from DMZ to any destination - zero-egress policy (ISO A.8.20)
- Block DMZ to INTERNAL - prevents lateral movement from honeypot to internal zone
- Allow SSH (port 22) inbound from SIMINTERNET to 192.168.10.10
- Allow Telnet (port 23) inbound from SIMINTERNET to 192.168.10.10

INTERNAL zone rules (controlling traffic from Windows Server):
- Allow outbound port 443 to any - Sentinel log forwarding
- Allow outbound port 53 UDP to any - DNS resolution for Azure endpoints
- Block INTERNAL to DMZ - internal machines cannot probe the honeypot

MGMT zone rules (controlling traffic from Shuffle SOAR):
- Allow outbound port 443 to any - Sentinel REST API calls
- Allow port 443 to pfSense - SOAR platform can call pfSense API
- Block MGMT to DMZ - management systems cannot directly reach the honeypot

SIMINTERNET zone rules (controlling what Kali can reach):
- Allow SSH (port 22) to 192.168.10.10 only
- Allow Telnet (port 23) to 192.168.10.10 only
- Block SIMINTERNET to INTERNAL
- Block SIMINTERNET to MGMT

> **Evidence:** pfSense configuration XML export in [`configs/pfsense-config-export.xml`](../configs/pfsense-config-export.xml). Screenshots of each rule set in [`evidence/screenshots/`](../evidence/screenshots/).

## Cowrie honeypot (Ubuntu 26.04)

**IP:** 192.168.10.10 - DMZ zone

**Key configuration:**

Cowrie listens on port 2222. An iptables rule redirects port 22 traffic to 2222 transparently:

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
```

The `iptables-persistent` package saves this rule across reboots.

Cowrie's `userdb.txt` is configured to reject most login attempts and accept only one specific credential pair. This generates realistic volumes of failed login events (T1110) while allowing attackers to eventually gain a fake shell session for TTY recording.

JSON logging is enabled (`output_jsonlog = true`) - all session data is written to `var/log/cowrie/cowrie.json` in structured format for Sentinel ingestion.

Cowrie runs as a dedicated low-privilege `cowrie` user and starts automatically via a systemd service on boot.

**Network configuration (`/etc/netplan/00-installer-config.yaml`):**

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.10.10/24]
      routes:
        - to: 10.0.0.0/24
          via: 192.168.10.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    ens34:
      dhcp4: yes
```

The specific static route for `10.0.0.0/24` via pfSense's DMZ interface is required for Cowrie to send reply packets back to the attacker, without it, SYN packets arrive but no SYN-ACK is returned and connections time out silently.

## Windows Server 2025

**IP:** 192.168.20.10 — Internal zone

**Configuration:**

- Renamed to `INTERNAL-SRV01`
- Windows Defender active — ISO A.8.7 evidence
- WinRM enabled for remote management
- Audit policies configured via `auditpol`:
  - Logon/Logoff — Success and Failure
  - Account Logon — Success and Failure
  - Privilege Use — Success and Failure
  - Detailed Tracking — Success and Failure

Security Events are forwarded to Microsoft Sentinel every minute via a PowerShell script (`scripts/winserver-sentinel.ps1`) running as a Windows Scheduled Task with elevated privileges. The script collects Event IDs 4624, 4625, 4648, 4720, 4732, and 4672.

## Kali Linux

**IP:** 10.0.0.10 - Simulated Internet zone

Deployed using the pre-built VMware image from kali.org. Network interface `eth0` configured with a static IP and default gateway pointing to pfSense's SIMINTERNET interface (10.0.0.1). All attack tools (Hydra, Nmap) are pre-installed in the Kali image.

## Shuffle SOAR (Ubuntu 26.04)

**IP:** 192.168.30.10 — Management zone

Shuffle deployed via Docker Compose. Docker Swarm initialised to support Shuffle's Orborus worker architecture:

```bash
docker swarm init --advertise-addr 192.168.30.10
docker network create --driver overlay --attachable shuffle_swarm_executions
```

`ENVIRONMENT_NAME` set to `onprem` in `docker-compose.yml`. OpenSearch Java heap reduced from 3202MB to 512MB to prevent OOM kills on the 4.9GB VM.

See [ADR-04](adrs/adr-04-shuffle-soar-choice.md) for the Shuffle + Python integration architecture reasoning.

## Connectivity verification

All inter-zone connectivity verified before Phase 3:

| Source | Destination | Port | Expected | Result |
|---|---|---|---|---|
| Kali (10.0.0.10) | Cowrie (192.168.10.10) | 22 | Allow | ✅ |
| Kali (10.0.0.10) | Windows Server (192.168.20.10) | any | Block | ✅ |
| Cowrie via pfSense | 10.0.0.10 (Kali) | ICMP | Block (zero-egress) | ✅ |
| Cowrie via ens34 | 8.8.8.8 | ICMP | Allow (VMnet8 path) | ✅ |
| Windows Server | portal.azure.com | 443 | Allow | ✅ |
| Windows Server | Cowrie (192.168.10.10) | any | Block | ✅ |
| pfSense | Windows Server (192.168.20.10) | ICMP | Allow | ✅ |

Continue to [Phase 3 — Detection & SOAR](phase-3-detection.md)
