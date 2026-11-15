
# Splunk Queries: Quiet as the Grave

The investigation used Splunk to correlate process execution, network connections, persistence, credential access, lateral movement, discovery, and attempted exfiltration. Searches are organized by investigative progression. All timestamps are UTC.

## 1. Initial Execution

Identify the suspicious review helper and its process ancestry.

```spl
ComputerName="THM-SREYES" EventCode=1 "NeuralNexusReview"
| table _time ComputerName Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 12:29:48, `explorer.exe` launched PowerShell to execute `review-helper.ps1` from the user's Downloads directory.

## 2. Command-and-Control Connections

Having identified the suspicious PowerShell process, examine outbound connections to its external destination.

```spl
ComputerName="THM-SREYES" EventCode=3 DestinationIp="45.147.230.131"
| table _time ComputerName Image SourceIp DestinationIp DestinationPort ProcessGuid
| sort _time
| head 10
```

**Finding:** PowerShell connected to `45.147.230.131:8443`. Process GUID correlation linked the connection to the malicious helper.

## 3. Persistence

Search for Registry modifications associated with the helper.

```spl
ComputerName="THM-SREYES" "NeuralNexusReview"
| search EventCode=13
| table _time ComputerName EventCode Image TargetObject Details Message
| sort _time
```

**Finding:** At 12:37:20, the process configured the `NeuralNexusReview` Run value to execute a copy of the script from AppData.

Verify that the persisted copy subsequently executed.

```spl
ComputerName="THM-SREYES" EventCode=1 "review-helper.ps1"
| table _time ComputerName User Image CommandLine ParentImage ParentCommandLine Message
| sort _time
```

**Finding:** At 12:44:04, `explorer.exe` launched the AppData copy under a new logon session.

## 4. Credential Access

Continue examining PowerShell activity for credential-bearing project configuration files.

```spl
"deploy-settings.json"
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** At 13:01:41, PowerShell Event ID 4104 recorded `Get-Content`, `ConvertFrom-Json`, and `[PSCredential]` construction using the deployment configuration. Later authentication as `svc-studioops` supported the credential-access assessment.

## 5. Lateral Movement and IT Inventory

Trace the service account through WinRM authentication and remote execution.

**Correlated evidence:** Event ID 4624 and Sysmon Event ID 1 established access to `THM-LSULLIVAN` at 13:16:42. The remote `wsmprovhost.exe` process launched `cmd.exe /c dir C:\ProgramData\StudioOps`.

Examine subsequent PowerShell activity involving the IT support files.

```spl
"StudioOps" ("csv" OR "Get-Content" OR "Import-Csv" OR "Copy-Item")
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** At 13:36:43, an `Invoke-Command` script block targeted `endpoint-list.csv` and `maintenance-runbook.txt` through a remote session.

## 6. Domain Controller and Marketing Workstation

Investigate explicit credential use toward the domain controller.

```spl
ComputerName="THM-LSULLIVAN" EventCode=4648 "DC-01"
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** At 13:48:25, credentials were supplied toward `HTTP/DC-01`. Network authentication succeeded, but remote command execution was not established.

Follow the activity to the Marketing workstation.

```spl
ComputerName="THM-MSMITH" EventCode=1 "approved-release"
| table _time ComputerName Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 15:03:37, WinRM-associated execution targeted `C:\StudioMarketing\approved-release` under `svc-studioops`.

## 7. Post-Compromise Discovery

Examine administrative enumeration on the Marketing workstation.

```spl
ComputerName="THM-MSMITH" EventCode=1 "Administrators"
| table _time ComputerName Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 16:03:38, `wsmprovhost.exe` launched `net.exe localgroup Administrators`, which subsequently spawned `net1.exe`. Related evidence also identified `net user` and `net share` commands.

## 8. Collection and Archive Preparation

Search PowerShell script blocks for the staging directory.

```spl
ComputerName="THM-SREYES" EventCode=4104 "review-staging"
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** At 16:27:01, commands targeted Marketing release materials and NeuralNexus project files for collection. At 16:27:28, `Compress-Archive` targeted `$env:TEMP\review-cache.zip`.

## 9. Attempted Exfiltration

Trace the archive name to process execution and its destination.

```spl
ComputerName="THM-SREYES" EventCode=1 "review-cache.zip"
| table _time ComputerName User Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 16:27:54, `curl.exe` executed beneath the persisted PowerShell process with `--upload-file`, targeting `https://45.147.230.131:8443/ingest/review-cache.zip`.

**Evidence limitation:** The upload command is confirmed; successful remote receipt was not independently verified.

## Conclusion

Correlating Sysmon, Windows Security events, and PowerShell script blocks reconstructed the intrusion from initial execution through persistence, credential access, WinRM lateral movement, discovery, collection, and attempted exfiltration.
