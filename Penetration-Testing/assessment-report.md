# Layover | Security Assessment Report

**Project:** Multi-Stage Infrastructure Penetration Test  
**Platform:** Hack The Box Labs  
**Assessment Date:** 14 November 2026  
**Focus Areas:** Network Security, Wireless Analysis, Web Exploitation, Linux Security

---

## 1. Executive Summary

The Layover assessment examined how weaknesses across different parts of an IT environment can be combined to gain access to internal infrastructure.

Starting with access to a remote workstation, I investigated the surrounding network, analysed wireless traffic, identified exposed credentials, and assessed an internal web application. This ultimately led to successful remote code execution and authenticated access to an internal Linux server.

What made this exercise particularly interesting was the relationship between its attack surfaces. Each discovery provided new information that shaped the next stage of the investigation. The resulting attack path crossed several security boundaries, from network communications to application functionality and operating-system access.

The assessment demonstrates why security teams need to consider how vulnerabilities interact across an environment, rather than evaluating individual systems in isolation.

## 2. Scope and Approach

The assessment was conducted entirely within an authorised Hack The Box laboratory.

I followed a structured process, beginning with reconnaissance and progressing through network analysis, vulnerability validation, controlled exploitation, post-exploitation investigation, and Linux security enumeration.

| Phase | Activities |
|---|---|
| Reconnaissance | Identifying exposed services and potential entry points |
| Wireless analysis | Discovering internal network connectivity and examining captured traffic |
| Credential assessment | Investigating authentication information exposed through network communications |
| Application testing | Assessing authenticated web application functionality and validating command execution |
| Internal access | Investigating application configuration and establishing access to an internal Linux server |
| Privilege-escalation assessment | Enumerating privileged services and researching potential escalation paths |

Throughout the exercise, I recorded successful techniques, unsuccessful attempts, and observations that influenced subsequent decisions.

## 3. Security Findings

### F-01 | Authentication Credentials Exposed in Network Traffic

**Severity: High**

During wireless traffic analysis, I identified authentication information transmitted through an unencrypted web connection.

The captured traffic demonstrated how credentials can be exposed when applications use plaintext communications, particularly where an attacker has access to the same network segment.

**Security implications:** Exposed credentials can allow an attacker to access applications and potentially reach additional internal resources.

**Recommendations:** Enforce HTTPS for all authentication traffic, disable insecure login endpoints, use secure session-cookie settings, and monitor for sensitive information transmitted over unencrypted protocols.

### F-02 | Authenticated Remote Code Execution in a Web Application

**Severity: Critical**

The internal web application presented an opportunity to investigate potentially unsafe server-side behaviour.

I validated command execution through a combination of response-time analysis and an observable network callback. Further testing established an interactive shell on the application server.

This was one of the most significant findings because it demonstrated that an authenticated application vulnerability could be used to move beyond the web interface and interact directly with the underlying operating system.

**Security implications:** Remote code execution can expose application data, enable unauthorised system activity, and create opportunities for further compromise.

**Recommendations:** Apply relevant security updates, review authenticated application functionality, restrict service-account permissions, and monitor unexpected process execution and outbound network connections.

### F-03 | Exposure of Application Configuration and Secrets

**Severity: High**

After obtaining access to the application server, I examined its configuration and identified database connection information and other sensitive application secrets.

This demonstrated how compromising a web service can expose information that supports further investigation of connected systems.

**Security implications:** Access to application secrets can increase the impact of an initial compromise, especially where credentials provide access to other services.

**Recommendations:** Restrict access to configuration files, use appropriate secrets management, minimise database-account privileges, and rotate exposed credentials following an incident.

### F-04 | Privileged Linux Service Security

**Assessment area: Privilege escalation**

The Linux security investigation identified an active CUPS service and a locally accessible communication socket.

This provided a basis for researching potential privilege-escalation weaknesses in privileged system services.

**Security implications:** Weak authentication controls or insecure privileged-service behaviour can increase the impact of an existing system compromise.

**Recommendations:** Maintain supported service versions, review local socket permissions, restrict unnecessary administrative functionality, and monitor changes to privileged system configurations.

## 4. Technical Outcomes

The assessment produced several meaningful results:

- Identified the initial external attack surface and established remote workstation access.
- Discovered and investigated an internal wireless environment.
- Captured and analysed network traffic containing exposed authentication information.
- Authenticated to an internal web application and demonstrated server-side remote code execution.
- Established an interactive shell and investigated sensitive application configuration.
- Reached an internal Linux system through authenticated SSH access.
- Identified a potential privilege-escalation path involving a privileged system service.

These results illustrate how a series of technical discoveries can be connected into a broader infrastructure attack path.

## 5. Defensive Recommendations

Several recurring security themes emerged from the investigation.

**Protect authentication data.** Sensitive credentials should never be transmitted through unencrypted application connections.

**Maintain application security.** Authenticated functionality must be tested and maintained as carefully as publicly accessible endpoints.

**Limit the impact of compromised accounts.** Least-privilege access and secure service configurations can help prevent an initial compromise from spreading.

**Strengthen network boundaries.** Internal systems should not automatically trust traffic or credentials originating from less-trusted environments.

**Improve monitoring across layers.** Unexpected web-server processes, unusual outbound connections, and suspicious authentication activity can provide valuable indicators of compromise.

## 6. Lessons Learned

The most valuable aspect of Layover was learning to connect findings across different technologies.

Network analysis was not an isolated exercise; it informed the web application investigation. Similarly, application exploitation was not the final objective, but a way to understand what additional systems and information could become accessible.

The exercise also reinforced the importance of troubleshooting systematically. Several approaches required adjustment, and keeping track of what had already been tested proved essential.

Finally, I gained a clearer appreciation of the difference between discovering a potential vulnerability and demonstrating its actual security impact. Reproducible evidence is what turns an observation into a defensible technical finding.

## 7. Conclusion

Layover provided a practical opportunity to investigate an interconnected infrastructure environment through multiple stages of penetration testing.

The project brought together wireless traffic analysis, credential security, web application exploitation, internal Linux access, and privileged-service investigation.

The central lesson was that **the wider attack path often matters as much as the individual vulnerabilities that make it possible**. Understanding those relationships is essential for both effective security testing and stronger defensive architecture.

---

*All testing was conducted within an authorised Hack The Box environment. Detailed exploitation procedures and sensitive lab information are maintained separately from this public assessment report.*
