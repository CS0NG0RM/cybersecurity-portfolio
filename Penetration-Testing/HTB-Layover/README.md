# Layover
<img width="1006" height="582" alt="image" src="https://github.com/user-attachments/assets/a5d3ecf7-6770-417d-9c04-ab1414d73a4d" />

### Multi-Stage Infrastructure Penetration Test

**Hack The Box Labs** · `Network Security` · `Web Exploitation` · `Linux Security`

---

## Overview

Layover is a multi-stage penetration-testing assessment examining how vulnerabilities across interconnected infrastructure can be chained to compromise internal systems.

The project spans **wireless networking, packet analysis, credential security, web application exploitation, and Linux post-exploitation**, providing an end-to-end perspective on how individual weaknesses contribute to broader organisational risk.

Rather than treating vulnerabilities in isolation, the assessment focuses on **attack-path development**, tracing how each discovery creates opportunities for further investigation and access.

## Assessment Workflow

```text
Reconnaissance
      |
      v
Network & Wireless Investigation
      |
      v
Traffic Analysis & Credential Exposure
      |
      v
Web Application Security Assessment
      |
      v
Remote Code Execution
      |
      v
Internal Infrastructure Access
      |
      v
Linux Privilege-Escalation Assessment
```

## Technical Coverage

| Domain | Tools & Techniques |
|:---|:---|
| **Network Security** | Nmap, service enumeration, network mapping |
| **Wireless Analysis** | Wi-Fi enumeration, monitor mode, tcpdump |
| **Traffic Investigation** | Packet capture, protocol analysis, credential exposure |
| **Web Application Security** | Craft CMS, authenticated application testing |
| **Exploitation** | Server-side remote code execution, reverse shells |
| **Internal Access** | SSH, Linux configuration and secret analysis |
| **Privilege Escalation** | CUPS service enumeration, vulnerability research |

## Security Findings & Implications

**01 / Sensitive Information Exposure**

Unprotected communications can expose authentication material, creating opportunities for attackers to progress beyond their initial access.

**02 / Web Application Attack Surface**

Authenticated application functionality may introduce exploitable server-side behaviour, making secure configuration and vulnerability management essential.

**03 / Internal Trust Boundaries**

Compromised credentials and application secrets can facilitate access to internal infrastructure, illustrating the risks of insufficiently enforced trust boundaries.

**04 / Privileged Service Security**

Local service permissions and administrative interfaces warrant careful assessment because weaknesses in these components can expose opportunities for privilege escalation.

## Defensive Priorities

- **Encrypted communications:** Protect authentication traffic and sensitive information in transit.
- **Credential management:** Restrict access to secrets and strengthen account protections.
- **Application hardening:** Maintain security updates and review authenticated functionality.
- **Network segmentation:** Limit unnecessary communication between infrastructure components.
- **Least privilege:** Restrict administrative capabilities and secure privileged services.
- **Monitoring:** Detect suspicious authentication, unusual outbound connections and unexpected process execution.

## Key Takeaways

This assessment demonstrates how weaknesses across wireless networks, authentication systems, web applications, and internal infrastructure can be chained together to enable progressively deeper system compromise.

The assessment strengthened practical experience in attack-surface mapping, traffic analysis, web exploitation, internal access assessment, technical troubleshooting and evidence-driven reporting.

It also reinforced the value of translating exploitation findings into **actionable defensive recommendations**.

---

<sub>**Environment:** Authorised Hack The Box laboratory · **Documentation:** Public, non-spoiler assessment summary</sub>
