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

### 🚀 Next Steps

* [ ] Exploit CVE-2019-16278 on Port 2222 for shell access.
* [ ] Verify Ghostcat vulnerability on Port 8009.
* [ ] Search for sensitive configuration files in the file system.
