# Phase 3 - Detection & SOAR

## Objective

Build the detection pipeline from raw honeypot logs through to automated incident response, and validate it with real simulated attacks. The goal was a fully automated chain: attack occurs → logs forwarded → Sentinel correlates → incident created → SOAR responds → threat intel enriched all without manual intervention.

## Log ingestion architecture

### Why not the Azure Monitor Agent

The standard approach for forwarding on-premises logs to Microsoft Sentinel is the Azure Monitor Agent (AMA) with Data Collection Rules (DCR). This path was attempted but blocked by two factors:

1. Microsoft has deprecated the legacy OMS/MMA agent and the new AMA requires Azure Arc to manage non-Azure VMs, which adds significant complexity and cost for a single-host lab.
2. The DCR-based custom log table creation hit repeated validation errors in the Azure portal due to personal Microsoft account limitations with the Unified RBAC model.

**Solution adopted:** Direct HTTP Data Collector API ingestion via Python scripts running as cron jobs on each source VM. This approach:
- Uses the same underlying Log Analytics ingestion endpoint
- Is fully supported and stable (despite being listed as "legacy" - the endpoint remains active)
- Produces identical data in Sentinel - the `CowrieLogs_CL` and `WindowsSecurityEvents_CL` tables are queryable with standard KQL
- Requires no agents, no Azure Arc, no Data Collection Rules

### Cowrie log forwarder

**Script:** `/opt/cowrie-sentinel.py`  
**Schedule:** Every minute via cron  
**Source:** `/home/cowrie/cowrie/var/log/cowrie/cowrie.json`  
**Destination:** `CowrieLogs_CL` table in Log Analytics workspace

The script tracks its position in the log file via `/opt/cowrie-sentinel.pos` and only forwards new entries since the last run. Log rotation detection is built in, if the file shrinks below the saved position, the position resets to 0 automatically.

Key Cowrie fields available in Sentinel after ingestion:

| Field | Type | Description |
|---|---|---|
| `eventid_s` | string | Cowrie event type (e.g. cowrie.login.failed) |
| `src_ip_s` | string | Attacker source IP |
| `username_s` | string | Username attempted |
| `password_s` | string | Password attempted |
| `session_s` | string | Unique session identifier |
| `protocol_s` | string | ssh or telnet |
| `sensor_s` | string | Honeypot hostname |

### Windows Security Event forwarder

**Script:** `C:\Scripts\winserver-sentinel.ps1`  
**Schedule:** Every minute via Windows Scheduled Task  
**Source:** Windows Security Event Log  
**Destination:** `WindowsSecurityEvents_CL` table

Collects Event IDs:
- **4624** - Successful logon
- **4625** - Failed logon
- **4648** - Logon with explicit credentials
- **4720** - User account created
- **4732** - User added to security group
- **4672** - Special privileges assigned

## Microsoft Sentinel analytics rules

Four scheduled analytics rules were deployed. Due to personal Microsoft account limitations with the Defender portal's Unified RBAC model, three rules were created via the Azure CLI using the Sentinel REST API directly rather than through the portal UI.

### Rule 1 - Cowrie SSH Brute Force Detection

```kql
CowrieLogs_CL
| where eventid_s == "cowrie.login.failed"
| summarize FailedAttempts = count() by src_ip_s, bin(TimeGenerated, 1m)
| where FailedAttempts >= 5
| project TimeGenerated, src_ip_s, FailedAttempts
```

- Severity: **High**
- Tactics: Credential Access
- Techniques: T1110
- Runs every 5 minutes, looks back 1 hour
- Entity mapping: IP → src_ip_s

### Rule 2 - Cowrie Honeypot Successful Login

```kql
CowrieLogs_CL
| where eventid_s == "cowrie.login.success"
| project TimeGenerated, src_ip_s, username_s, password_s
```

- Severity: **High**
- Tactics: Initial Access
- Techniques: T1110
- Any successful login to the honeypot is always suspicious - no threshold required

### Rule 3 - Windows Failed Logon Attempts

```kql
WindowsSecurityEvents_CL
| where EventId_d == 4625
| summarize FailedLogons = count() by MachineName_s, bin(TimeGenerated, 5m)
| where FailedLogons >= 3
| project TimeGenerated, MachineName_s, FailedLogons
```

- Severity: **Medium**
- Tactics: Credential Access
- Techniques: T1110

### Rule 4 - Cowrie Port Scan Detection

```kql
CowrieLogs_CL
| where eventid_s == "cowrie.session.connect"
| summarize ConnectionCount = count() by src_ip_s, bin(TimeGenerated, 1m)
| where ConnectionCount >= 10
| project TimeGenerated, src_ip_s, ConnectionCount
```

- Severity: **Medium**
- Tactics: Reconnaissance, Discovery
- Techniques: T1595, T1046
- Deployed via Azure CLI, check `configs/sentinel-analytics-rules/cowrie-port-scan-detection.json`

### Rule 5 - Honeypot Egress Violation

```kql
CowrieLogs_CL
| where eventid_s == "cowrie.direct-tcpip.request" or eventid_s == "cowrie.direct-tcpip.data"
| project TimeGenerated, src_ip_s, eventid_s
```

- Severity: **High**
- Tactics: Defense Evasion
- Techniques: T1562
- Fires if an attacker inside Cowrie attempts to tunnel outbound connections
- Deployed via Azure CLI, check `configs/sentinel-analytics-rules/honeypot-egress-violation.json`

## SOAR automated response

### Architecture decision

Shuffle SOAR is deployed as the orchestration platform on the Management zone VM. Due to single-node Docker Swarm constraints in the lab environment, the automated response workflows are executed via a Python integration script that replicates the Shuffle workflow logic directly against the Sentinel REST API. In a production environment with multiple nodes, the Shuffle workflows would execute natively via Orborus workers.

See [ADR-04](adrs/adr-04-shuffle-soar-choice.md) for full reasoning.

### SOAR response script

**Script:** `/opt/soar-response.py`  
**Schedule:** Every 5 minutes via cron  
**Logic:**

1. Authenticate to Azure using Service Principal credentials
2. Query Sentinel for new (unacknowledged) incidents
3. For each new incident, query the Cowrie log table to extract the attacker's source IP
4. Enrich the IP against AlienVault OTX and AbuseIPDB
5. Post an automated comment to the Sentinel incident containing the response action and threat intel
6. Track processed incidents in `/opt/soar-processed.json` to prevent duplicate responses

### Threat intelligence enrichment

Two free-tier threat intelligence feeds are integrated:

**AlienVault OTX** - queries the OTX IPv4 indicator endpoint for the attacker IP, returning pulse count (number of threat reports) and reputation score.

**AbuseIPDB** - queries the check endpoint for the attacker IP, returning abuse confidence score (0-100%), total reports, country of origin, and ISP.

Both results are embedded in the automated Sentinel incident comment, giving the SOC analyst immediate context without needing to manually look up the attacker IP.

Example automated comment format:
```
SOAR AUTO-RESPONSE | 2026-06-24T07:40:03Z | Incident: Cowrie SSH Brute Force Detection | 
Severity: High | Action: Incident acknowledged by automated SOAR pipeline. Cowrie honeypot 
logs reviewed. Attacker IP flagged for threat intel correlation. | THREAT INTEL for 10.0.0.10: 
OTX: 0 threat pulses, reputation score 0 | AbuseIPDB: confidence score 0%, 2 reports, 
country None, ISP None
```

> **Note:** The lab attacker IP (10.0.0.10 - Kali VM) is a private RFC1918 address with no real threat intelligence data. In a production deployment receiving real internet-sourced attacks, meaningful OTX pulse counts and AbuseIPDB confidence scores would appear.

## Attack scenarios validated

### Scenario A - SSH Brute Force (T1110.001)

**Tool:** Hydra with rockyou.txt wordlist  
**Command:** `hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.10.10 -t 4`  
**Result:** Cowrie logged every attempt. Sentinel brute force rule fired within 5 minutes. SOAR posted automated response comment with threat intel.  
**Evidence:** Screenshots 24, 25, 26

### Scenario B - Network Reconnaissance (T1595, T1046)

**Tool:** Nmap full port scan + rapid TCP connection loop  
**Command:** `nmap -sS -sV -p- --disable-arp-ping -Pn 192.168.10.10`  
**Result:** Nmap identified port 22 open, presenting Cowrie's fake OpenSSH banner. Rapid connection loop (20 connections) triggered the Port Scan Detection rule. Incident created in Sentinel.  
**Evidence:** Screenshots 29, 30, 31, 32

### Scenario C - Firewall Egress Failure (T1562)

**Scenario:** Zero-egress block rule temporarily disabled to simulate a firewall misconfiguration or policy failure.  
**Result (baseline):** Cowrie cannot reach Kali (10.0.0.10), zero-egress confirmed working.  
**Result (misconfigured):** With zero-egress disabled and a temporary ICMP allow rule added, Cowrie successfully reaches Kali, demonstrating what a firewall failure would allow.  
**Result (remediated):** Zero-egress rule re-enabled, temp rule deleted, egress blocked again.  
**Evidence:** Screenshots 33, 34, 35, 36  

> **Detection note:** The Honeypot Egress Violation Sentinel rule (Rule 5) fires specifically on Cowrie's `direct-tcpip` events - these appear when an attacker inside Cowrie's fake shell attempts to tunnel connections outbound. The scenario above used ICMP ping to demonstrate the firewall failure concept; a real attacker using SSH port forwarding would generate the `direct-tcpip` events that trigger Rule 5.

## KPI baselines recorded

From the brute force simulation:

| Metric | Value |
|---|---|
| Mean Time to Detect (MTTD) | < 5 minutes (one analytics rule cycle) |
| Mean Time to Respond (MTTR) | < 5 minutes (one SOAR cron cycle) |
| Brute force attempts logged | 974 in single session |
| Failed logins per minute (peak) | 454 |
| Incidents auto-commented | 20 |

Continue to [Phase 4 - GRC Mapping & Documentation](phase-4-grc.md)
