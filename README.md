# NIST SP 800-53 Compliant SIEM Lab using Wazuh

This repository documents the design and implementation of a **Security Information and Event Management (SIEM)** solution built with **Wazuh**, mapped to **NIST SP 800-53** security controls. The lab secures a Windows endpoint through continuous monitoring, detection, and centralized logging to support information assurance and forensic investigation requirements.

## Executive Summary

The implemented system demonstrates five core security capabilities:

- Vulnerability Detection
- File Integrity Monitoring
- Secure Configuration Assessment
- Access Control Monitoring
- Centralized Logging and Audit Retention

## Technical Architecture

| Component | Details |
|---|---|
| **SIEM Platform** | Wazuh Manager v4.x (Manager + Dashboard + Indexer) on Ubuntu Linux |
| **Monitored Endpoint** | Windows 10 Enterprise/Pro (simulated corporate workstation) with Wazuh Agent |
| **Compliance Framework** | NIST SP 800-53 — Security & Privacy Controls |
| **Key Capabilities** | Real-time log analysis, File Integrity Monitoring (FIM), CVE/vulnerability detection, Security Configuration Assessment (SCA), IDS integration (Suricata) |

## Deployment

### Wazuh Manager (Ubuntu)
```bash
sudo apt update && sudo apt upgrade -y
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```
Dashboard accessed at `https://<manager-ip>` (port 5601).

### Wazuh Agent (Windows 10)
```powershell
# Silent install
msiexec /i wazuh-agent.msi /qn WAZUH_MANAGER="192.168.1.100"

# Register agent
"C:\Program Files (x86)\ossec-agent\manage_agents.exe"
# Choose: A -> Add agent -> enter name -> enter manager IP

# Start service
net start WazuhSvc
```
Verify from the manager: `sudo /var/ossec/bin/agent_control -l`

### Enable Windows Event Log Collection
Add to `ossec.conf` on the agent:
```xml
<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>
```
Restart the agent (`net stop WazuhSvc` / `net start WazuhSvc`) to apply.

## IDS Integration: Suricata + Wazuh

Suricata was integrated to add network-level intrusion detection, with alerts decoded and surfaced directly in the Wazuh dashboard.

**Custom detection rule** (`/etc/suricata/rules/local.rules`):
```
alert icmp any any -> any any (msg:"ICMP Ping Detected - Custom Test"; sid:1000001; rev:1;)
```

**Forward Suricata's JSON alerts** (`eve.json`) via the Wazuh agent:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

**Wazuh manager decoder** (`/var/ossec/etc/decoders/`):
```xml
<decoder_list>
  <decoder name="suricata">
    <prematch>suricata</prematch>
  </decoder>
</decoder_list>
```

Test by restarting Suricata and generating traffic:
```bash
sudo systemctl restart suricata
ping -c 4 <target-ip>
sudo tail -f /var/ossec/logs/alerts/alerts.json
```
**Result:** ICMP ping alerts appeared in the Wazuh Security Events dashboard, centralizing IDS alerts alongside host telemetry.

## Security Pillars (NIST Controls Implemented)

| Pillar | NIST Control | Implementation | Result |
|---|---|---|---|
| **1. Vulnerability Detection** | RA-5 | Enabled `<vulnerability-detector>` module for daily CVE feeds (Canonical/Microsoft); `syscollector` reports full software inventory hourly | Identified critical CVEs on the target endpoint |
| **2. File Integrity Monitoring** | SI-7 | Configured `syscheck` (FIM) to monitor `C:\Users\Public` and a dedicated `C:\nist_test_folder` in real time | Instant alerts on file creation, modification, and deletion |
| **3. Secure Configuration Assessment** | CM-6 | Deployed custom Security Configuration Assessment (SCA) policy `.yml` files from an external repo to `/var/ossec/etc/sca`; re-scanned after agent restart | Real-time compliance pass/fail reporting mapped to NIST references |
| **4. Access Control Monitoring** | AC-2 / AC-7 | Ingested Windows Security Event Log channel | Detected simulated brute-force logins and unauthorized privilege escalation |
| **5. Audit & Log Retention** | AU-6 | Encrypted log forwarding from Agent to Manager on port 1514 | Verified centralized archival at `/var/ossec/logs/archives` |

### File Integrity Monitoring config example
```xml
<syscheck>
  <directories check_all="yes">C:\Users\Public</directories>
  <frequency>10</frequency>
</syscheck>

<directories check_all="yes" realtime="yes" report_changes="yes">C:\nist_test_folder</directories>
```

## Threat Simulation & Validation

Adversarial techniques were simulated to validate detection coverage:

| Attack Type | Command Executed | Detection Result | NIST Mapping |
|---|---|---|---|
| File Tampering | `New-Item -Path "C:\nist_test_folder\malware.txt"` | Detected: "File added to system" | SI-7 |
| Persistence (privilege escalation) | `net localgroup administrators hacker /add` | Detected: "User added to local group" | AC-2 |
| Service Modification | `net start Spooler` | Detected: "Service state changed" | CM-3 |

## Key Takeaways

- A full Wazuh SIEM stack (Manager, Indexer, Dashboard) was stood up on Ubuntu and paired with a monitored Windows 10 endpoint.
- Five NIST SP 800-53 control families (RA-5, SI-7, CM-6, AC-2/AC-7, AU-6) were mapped to concrete, working detections rather than left as paper controls.
- Suricata IDS integration extended detection from host-based telemetry to network-level events, decoded and centralized in the same dashboard.
- Simulated attacks (file tampering, privilege escalation, unauthorized service changes) were each successfully detected and attributed to their corresponding control.

## Source

Full walkthrough, screenshots, and configuration detail: `IA-REPORT.docx`
