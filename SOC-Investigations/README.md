# Security Operations Center (SOC) Investigations

This section of my cybersecurity portfolio documents hands-on security investigations conducted in authorized TryHackMe SOC Simulator environments.

The projects demonstrate practical experience with SIEM analysis, security alert triage, incident reconstruction, false-positive identification, escalation decisions, and technical reporting.

## SOC Investigation Projects

### 1. Introduction to Phishing

**Platform:** TryHackMe SOC Simulator  
**Focus:** Phishing Detection and Alert Triage

An introductory SOC investigation focused on evaluating suspicious activity, interpreting security alerts, and distinguishing malicious behavior from legitimate activity.

**Skills:** Alert triage, phishing analysis, security monitoring, and incident classification.

*Detailed technical documentation to be added.*

### 2. Phishing Unfolding

**Platform:** TryHackMe SOC Simulator  
**SIEM:** Microsoft Sentinel  
**Focus:** Phishing Investigation and Incident Response

A practical investigation involving correlated phishing alerts, suspicious activity analysis, evidence-based incident classification, and escalation decisions.

**Verified results:**
- 19 alerts closed.
- 100% True Positive identification rate.
- 75% False Positive identification rate.

**Skills:** Microsoft Sentinel, log correlation, phishing investigation, incident response, and security reporting.

*Detailed technical documentation to be added.*

### 3. Repossessed

**Platform:** TryHackMe SOC Simulator  
**SIEM:** Splunk  
**Focus:** Multi-Stage Intrusion Investigation

Investigation of a simulated security compromise involving malicious PowerShell execution, credential reconnaissance, command-and-control communications, lateral movement, unauthorized Windows service creation, and attempted collection of proprietary development assets.

**Verified results:**
- 9 alerts correctly classified and closed.
- 7 True Positives correctly identified.
- 2 False Positives correctly identified.
- 100% classification accuracy.
- Mean time to resolve: 7 minutes.

**Skills:** Splunk SPL, Sysmon, Windows Security logs, PowerShell analysis, process correlation, lateral movement investigation, and incident escalation.

[View verified Repossessed results](https://tryhackme.com/soc-sim/public-summary/da7a82ef0e5c1e7180c12eb9a27916bcaa1162f684f607340b3cef04f7a306fc01a263d4ad1101ba97af471149a44bec)

*Detailed technical documentation to be added.*

## Investigation Methodology

My investigations follow an evidence-based workflow:

1. Review and prioritize security alerts.
2. Identify affected users, endpoints, and processes.
3. Investigate relevant SIEM and endpoint telemetry.
4. Correlate events and reconstruct incident timelines.
5. Distinguish confirmed findings from assumptions.
6. Classify alerts and determine escalation requirements.
7. Document remediation recommendations and lessons learned.

## Technical Skills

- SOC alert triage and incident investigation
- Microsoft Sentinel and Splunk
- Security log analysis and event correlation
- Windows Security, Sysmon, and PowerShell telemetry
- Phishing and malicious execution analysis
- Network and lateral movement investigation
- True Positive and False Positive classification
- Incident escalation and technical reporting

## Disclaimer

All investigations were conducted in authorized simulated training environments. Findings reflect hands-on laboratory exercises, not real-world security incidents.
