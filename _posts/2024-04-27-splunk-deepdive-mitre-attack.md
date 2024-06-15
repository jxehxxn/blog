---
layout: post
title: "MITRE ATT&CK Deep Dive and Splunk Detection Engineering: A Comprehensive Guide"
date: 2024-04-27 09:00:00 +0900
categories: security splunk
---

Modern security operations centers (SOC) face a constant barrage of threats. To defend effectively, we must move beyond simple signature-based detection. We need a structured way to understand how attackers operate. This is where the MITRE ATT&CK framework comes in. This chapter explores how to integrate MITRE ATT&CK with Splunk to build a resilient detection engineering pipeline.

### The Foundation: MITRE ATT&CK Framework

MITRE ATT&CK is a globally accessible knowledge base of adversary tactics and techniques based on real-world observations. It provides a common language for security professionals to describe and analyze attacks.

Tactics represent the "why" of an attack. They are the adversary's tactical goals, such as gaining initial access or exfiltrating data. Techniques represent the "how." They are the specific methods used to achieve a tactical goal. Procedures are the specific implementation of a technique by a particular threat actor.

### Data Models and Normalization

Before we can detect anything, we need data. In Splunk, this data must be normalized using the Common Information Model (CIM). This ensures that fields like "src_ip" or "user" are consistent across different data sources.

For endpoint detection, we rely on the Endpoint data model. This includes datasets for processes, file systems, and registry changes. For network detection, we use the Network Traffic and Network Resolution (DNS) models. Without proper normalization, our correlation searches will fail to catch threats across diverse environments.

### Deep Dive: 5 Critical Techniques and Splunk Mappings

Let's examine five common techniques and how to map them to Splunk detection logic.

#### 1. T1059: Command and Scripting Interpreter

Attackers use interpreters like PowerShell, Python, or the Windows Command Shell to execute malicious code. This is a versatile technique used for everything from initial execution to persistence.

**Splunk Mapping:**
We look for suspicious process executions. We focus on interpreters launched with encoded commands or from unusual parent processes.

**Correlation Search (Splunk ES):**
```splunk
| tstats `summariesonly` count min(_time) as firstTime max(_time) as lastTime from datamodel=Endpoint.Processes where Processes.process_name IN ("powershell.exe", "pwsh.exe", "cmd.exe") Processes.process IN ("*-enc*", "*-encodedcommand*", "*bypass*", "*-nop*") by Processes.dest Processes.user Processes.parent_process_name Processes.process_name Processes.process Processes.process_id Processes.parent_process_id
| `drop_dm_object_name(Processes)`
| `security_content_ctime(firstTime)`
| `security_content_ctime(lastTime)`
```
This search identifies PowerShell or CMD instances running with common obfuscation flags. It uses the Endpoint data model for performance.

#### 2. T1078: Valid Accounts

Adversaries often use stolen credentials to gain access or move laterally. This is difficult to detect because the activity looks like legitimate user behavior.

**Splunk Mapping:**
We look for anomalies in login patterns. This includes logins from unusual locations, at odd hours, or to systems the user doesn't typically access.

**Correlation Search (Splunk ES):**
```splunk
| tstats `summariesonly` count from datamodel=Authentication where Authentication.action="success" by Authentication.user, Authentication.src, Authentication.dest
| iplocation Authentication.src
| eventstats avg(count) as avg_logins, stdev(count) as dev_logins by Authentication.user
| where count > (avg_logins + (3 * dev_logins))
| `drop_dm_object_name(Authentication)`
```
This search uses statistical analysis to find users with a sudden spike in successful logins, which might indicate credential abuse.

#### 3. T1003: OS Credential Dumping

Attackers dump credentials from the operating system to escalate privileges. On Windows, this often involves accessing the Local Security Authority Subsystem Service (LSASS) memory.

**Splunk Mapping:**
We monitor for processes accessing LSASS.exe with suspicious access rights. Tools like Mimikatz are common culprits.

**Correlation Search (Splunk ES):**
```splunk
index=windows source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=10 TargetImage="C:\\Windows\\system32\\lsass.exe"
| search GrantedAccess="0x1010" OR GrantedAccess="0x1410"
| stats count by _time, Computer, SourceImage, SourceProcessId, TargetImage, GrantedAccess
```
This search uses Sysmon Event ID 10 (Process Access) to find non-standard processes requesting handle access to LSASS.

#### 4. T1021: Remote Services

Lateral movement often involves using legitimate remote services like RDP or SMB. Attackers use these to move from one compromised host to another.

**Splunk Mapping:**
We look for RDP connections between internal hosts that don't usually communicate. We also watch for "Pass-the-Hash" attempts over SMB.

**Correlation Search (Splunk ES):**
```splunk
| tstats `summariesonly` count from datamodel=Network_Traffic where All_Traffic.dest_port=3389 by All_Traffic.src, All_Traffic.dest
| lookup internal_assets.csv device_id as All_Traffic.src OUTPUT type as src_type
| lookup internal_assets.csv device_id as All_Traffic.dest OUTPUT type as dest_type
| where src_type="workstation" AND dest_type="workstation"
| `drop_dm_object_name(All_Traffic)`
```
This search flags RDP traffic between two workstations. In most environments, workstations should only RDP to servers, not to each other.

#### 5. T1055: Process Injection

Process injection is a method of executing arbitrary code in the address space of a separate live process. This helps attackers evade detection and gain higher privileges.

**Splunk Mapping:**
We monitor for cross-process memory operations. Sysmon Event ID 8 (CreateRemoteThread) is a key indicator.

**Correlation Search (Splunk ES):**
```splunk
index=windows source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=8
| stats count by _time, Computer, SourceImage, TargetImage, StartAddress
| where NOT match(SourceImage, "(?i)C:\\\\Windows\\\\System32\\\\.*")
```
This search identifies instances where a process creates a thread in another process, excluding standard Windows system processes.

### Building the Detection Pipeline

Detection engineering is a continuous cycle. It starts with identifying a threat and ends with a validated, tuned alert.

1.  **Threat Modeling**: Identify which MITRE techniques are most relevant to your organization.
2.  **Data Gap Analysis**: Determine if you have the logs needed to detect those techniques.
3.  **Logic Development**: Write the SPL or correlation search.
4.  **Testing and Validation**: Use tools like Atomic Red Team to simulate the attack and verify the alert.
5.  **Tuning**: Reduce false positives by adding exceptions for legitimate administrative activity.

### Measuring Success with Coverage Heatmaps

A coverage heatmap is a visual representation of your detection capabilities. It maps your active correlation searches to the MITRE ATT&CK matrix.

Green cells indicate techniques with robust detection. Yellow cells show techniques where you have data but no active alerts. Red cells represent gaps where you lack both data and detection.

This heatmap is not just for the SOC. It is a powerful tool for communicating security posture to leadership. It shows exactly where investments in new data sources or engineering time will have the most impact.

### Conclusion

Integrating MITRE ATT&CK with Splunk transforms your SOC from reactive to proactive. By focusing on attacker techniques rather than just indicators of compromise, you build a defense that is much harder to bypass. Start small, pick five techniques, and build your first correlation searches today. The goal is not to cover the entire matrix at once, but to continuously improve your visibility and response capabilities.
