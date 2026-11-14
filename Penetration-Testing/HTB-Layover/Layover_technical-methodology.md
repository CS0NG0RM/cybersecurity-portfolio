# Layover | Technical Methodology

**Assessment:** Multi-Stage Infrastructure Penetration Test  
**Platform:** Hack The Box Labs  
**Date:** 14 November 2026  
**Scope:** Authorised laboratory environment

> Selected commands and sanitised evidence from the assessment. Credentials, flags, raw captures and machine-specific exploit payloads are excluded from this public record.

## 1. Network Reconnaissance

I began with a connectivity check and a full TCP scan to establish the exposed attack surface.

```bash
ping -c 3 <TARGET_IP>
nmap -p- --open -Pn -T4 <TARGET_IP>
```

**Results:** ICMP replies received (0% packet loss); two open TCP ports identified.

```text
PORT     STATE SERVICE
22/tcp   open  ssh
3389/tcp open  ms-wbt-server
```

SSH rejected the supplied account credentials. I repeated the test without manual password entry to rule out a keyboard-layout issue.

```bash
ssh contractor@<TARGET_IP>
sshpass -p '<REDACTED>' ssh -o PubkeyAuthentication=no contractor@<TARGET_IP>
```

```text
Permission denied (publickey,password).
```

**Decision:** With connectivity confirmed and SSH authentication unsuccessful, I investigated RDP.

## 2. Initial Access & Host Enumeration

The exposed RDP service accepted the supplied account credentials and opened a Linux desktop on `airside-ws01`.

```bash
xfreerdp /v:<TARGET_IP> /u:contractor /p:'<REDACTED>' /cert:ignore +clipboard
```

I checked account identity, group membership and administrative permissions from the remote terminal.

```bash
whoami
id
sudo -l
sudo id
```

**Selected output:**

```text
contractor
User contractor may run the following commands on airside-ws01:
    (ALL : ALL) ALL
uid=0(root) gid=0(root) groups=0(root)
```

The `contractor` account could invoke commands as root **on this workstation**. This did not imply root access to the separate internal application server.

```bash
sudo ls -la /root
ls -la /home/contractor
ls -la /home/contractor/Desktop
```

The Desktop was empty and the targeted directory checks did not reveal the expected assessment artefacts. A broader filesystem search also encountered access restrictions on the RDP-mounted `thinclient_drives` path, so I switched to targeted inspection rather than altering mount permissions.

**Decision:** The workstation appeared to be an entry point rather than the system hosting the main application. I moved on to its network interfaces.

## 3. Wireless Investigation & Traffic Analysis

Interface enumeration showed two wireless adapters. Scanning identified an internal wireless network operating on channel 6 (2437 MHz).

```bash
ip -br addr
sudo iw dev wlan2 scan | grep -E 'SSID:|freq:|signal:'
```

I connected using `wlan2` and checked the assigned network and routes.

```bash
sudo nmcli device wifi connect '<REDACTED_SSID>' ifname wlan2
ip -4 addr show wlan2
ip route
```

**Observed:** A DHCP-assigned address on `10.13.37.0/24` and a wireless gateway at `10.13.37.1`.

I configured the second adapter for passive traffic monitoring, leaving the first adapter connected.

```bash
sudo ip link set wlan3 down
sudo iw dev wlan3 set type monitor
sudo ip link set wlan3 up
sudo iw dev wlan3 set channel 6
sudo tcpdump -i wlan3 -nn -s 0 -w /tmp/layover-wifi.pcap
```

The capture collected **1,156 packets with zero reported drops**. I inspected HTTP traffic for requests and form fields.

```bash
sudo tcpdump -A -nn -r /tmp/layover-wifi.pcap 'tcp port 80' \
  | grep -E 'GET |POST |Host:|username=|password='
```

**Finding:** An HTTP authentication request exposed credentials in plaintext. This was an application transport-security failure, not evidence that wireless encryption itself had been broken. The raw capture and credential values remain private.

## 4. Web Application Assessment & RCE Validation

The traffic investigation led to an internal web application running **Craft CMS 5.9.8 (Solo edition)**. I inspected the login page and CSRF-related fields before testing authenticated functionality.

```bash
python3 -c "import requests; r=requests.get('http://<INTERNAL_HOST>/admin/login'); print('Status:',r.status_code); print('Cookies:',list(r.cookies.keys())); print('CSRF fields:',[x for x in r.text.splitlines() if 'csrf' in x.lower()][:5])"
```

Using an authenticated CMS session, I tested the application through Chrome DevTools. I compared the response time of a normal request with a controlled five-second delay test.

**Baseline request:**

```javascript
(async () => {
  const start = performance.now();
  const response = await fetch('/admin/actions/element-search/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      'X-CSRF-Token': window.Craft.csrfTokenValue
    },
    body: JSON.stringify({
      elementType: 'craft\\elements\\Category',
      siteId: 1,
      search: ''
    })
  });
  console.log('HTTP status:', response.status);
  console.log('Elapsed seconds:', ((performance.now() - start) / 1000).toFixed(2));
})();
```

| Test | HTTP status | Time |
|---|---:|---:|
| Baseline | 200 | 0.08 s |
| Controlled delay | 500 | 5.28 s |

The delay supported the command-execution hypothesis but was not sufficient proof by itself. I then used a temporary HTTP listener to verify an outbound callback from the application server.

```bash
python3 -m http.server 8000 --bind <LISTENER_IP> --directory /tmp
```

```text
<APPLICATION_SERVER> - - [...] "GET /<TEST_MARKER> HTTP/1.1" 404 -
```

The missing resource explains the `404`. The relevant evidence is the request originating from the application server following the controlled test. A separate attempt returned literal shell-substitution text, so I did not treat that attempt as successful command expansion.

**Finding:** Authenticated server-side command execution was verified through timing comparison, an observable callback and subsequent shell access.

## 5. Application Server Access & Configuration Review

I staged a short Bash script from the assessment workstation and prepared a listener for an interactive connection.

```bash
# Script content and execution request omitted from the public version
nc -lvnp 4444
```

The application server retrieved the staged file from the temporary HTTP server.

```text
<APPLICATION_SERVER> - - [...] "GET /rs.sh HTTP/1.1" 200 -
```

**Selected listener output:**

```text
Connection received from <APPLICATION_SERVER>
www-data@portal:~/portal/web$
```

This confirmed shell access as the web-service account. I inspected the application configuration:

```bash
cat /var/www/portal/.env
```

```dotenv
CRAFT_SECURITY_KEY=<REDACTED>
CRAFT_DB_USER=<REDACTED>
CRAFT_DB_PASSWORD=<REDACTED>
CRAFT_DB_DATABASE=<REDACTED>
```

**Finding:** The compromised application account could read database connection details and an application security key. During subsequent database investigation, an unresponsive interactive command and a `Ctrl+C` interruption terminated the basic reverse shell. This was a shell-stability issue, not a verified database extraction.

## 6. Internal SSH Access

I investigated authentication material and the available access paths beyond the web-service account. Database-based recovery of the subsequent SSH credential was **not completed** during this session; it should not be presented as a verified extraction step.

An identified internal account was successfully authenticated over SSH:

```bash
ssh aporter@<INTERNAL_HOST>
```

```text
aporter@portal:~$
```

I verified access to the account's assessment artefact:

```bash
cat ~/user.txt
```

**Result:** The User Flag was retrieved and accepted by Hack The Box. The flag and account password are not included in the public record.

## 7. Linux Privilege Escalation Assessment

With authenticated Linux access, I examined locally available privileged services, beginning with CUPS.

```bash
systemctl status cups --no-pager
cups-config --version
ls -l /run/cups/cups.sock
```

**Selected observations:**

```text
cups.service: active (running)
CUPS version: 2.4.16
/run/cups/cups.sock: srwxrwxrwx
```

The socket's permissive mode warranted further checks of CUPS authentication and administrative controls. Neither its permission bits nor the installed version alone established an exploitable vulnerability.

**Result:** Privileged-service enumeration completed; root-level compromise was **not demonstrated** in this session.

---

### Assessment Summary

| Area | Verified outcome |
|---|---|
| External access | RDP authentication to a Linux workstation |
| Wireless/network | Internal connectivity and plaintext credential exposure |
| Application security | Authenticated remote code execution, independently validated |
| Post-exploitation | Interactive `www-data` shell and accessible application secrets |
| Internal access | Authenticated SSH session and accepted User Flag |
| Local escalation | CUPS service and socket investigated; no root compromise |

*This methodology contains selected evidence, not a complete command history. Sensitive data and detailed active-machine exploitation steps are retained separately.*
