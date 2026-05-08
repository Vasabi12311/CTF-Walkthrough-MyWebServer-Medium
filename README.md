# CTF-Walkthrough-MyWebServer-Medium
Writeup for "My Web Server" (Medium) CTF. Detailed penetration testing process covering enumeration, web exploitation, and privilege escalation. 
![Difficulty: Medium](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category: Web/Linux](https://img.shields.io/badge/Category-Web%2FLinux-blue)
## 🔍 Phase 1: Enumeration

### Host Discovery
First, I identified the target machine's IP address on the local network using an Nmap ping scan.

```bash
nmap -sn 192.168.0.1/24

Target IP Found: 192.168.0.102 (identified by Oracle VirtualBox MAC address).
Service Scanning

A full TCP port scan was performed to identify open services and their versions.
Bash

nmap -p- -sC -sV 192.168.0.102

Open Ports & Services:
Port	Service	Version	Observations
22	SSH	OpenSSH 7.9p1	Standard SSH access.
80	HTTP	Apache 2.4.38	Contains robots.txt with /wp-admin/ (WordPress?).
2222	HTTP	Nostromo 1.9.6	Interesting: Rare web server version.
3306	MySQL	MySQL	Unauthorized access (standard behavior).
8009	AJP13	Apache Jserv	Used for communication with Tomcat.
8080	HTTP	Tomcat 8.0.33	Apache Tomcat manager/apps might be here.
8081	HTTP	Nginx 1.14.2	Another web service, likely a static site.
### Service Analysis & Potential Attack Vectors

After analyzing the scan results, several high-priority attack surfaces were identified:

1.  **Port 2222 (Nostromo 1.9.6):**
    *   **Vulnerability:** Known for **CVE-2019-16278** (Directory Traversal leading to Remote Code Execution).
    *   **Priority:** Critical. This is a likely entry point for a Reverse Shell.

2.  **Port 8009 (AJP13):**
    *   **Vulnerability:** Suspected **Ghostcat (CVE-2020-1938)** vulnerability due to the legacy Apache Jserv version.
    *   **Priority:** High. Could allow arbitrary file reading (e.g., `WEB-INF/web.xml` for credentials).

3.  **Port 8080 (Apache Tomcat 8.0.33):**
    *   **Vulnerability:** Older version of Tomcat. Potential for administrative panel access via brute-force or default credentials (`tomcat:tomcat`, `admin:admin`).
    *   **Priority:** Medium.

4.  **Port 80 (WordPress / armour.local):**
    *   **Status:** Detected `/wp-admin/` in `robots.txt`. Requires local DNS resolution (adding `armour.local` to `/etc/hosts`) for proper enumeration.
    *   **Priority:** Medium.

## 🔍 Phase 2: Web Enumeration (Port 80)

### Vulnerability Scanning
Since Port 80 revealed a WordPress installation, I performed an aggressive scan using `wpscan` to identify themes, plugins, and potential users.

```bash
wpscan --url [http://www.armour.local/](http://www.armour.local/) --enumerate vp,vt,u --plugins-detection aggressive --random-user-agent --ignore-main-redirect

```

**Key Findings:**

* **WordPress Version:** 5.3.21 (Outdated).
* **Theme:** `rife-free` version 2.4.7.
* **Directory Listing:** Enabled on `/wp-content/uploads/` (Information Disclosure).
* **User Enumeration:** Successfully identified a valid username: `ap20dsero039`.

### Analysis of WPScan Results

While the scan confirmed the site is outdated, no immediate high-criticality plugin vulnerabilities were found. However, gaining a valid username (`ap20dsero039`) provides a pivot point for credential attacks or lateral movement if this username is reused across other services like **SSH** or **MySQL**.

> **Note:** Brute-forcing the WordPress login was deemed inefficient at this stage. Moving focus to other high-priority services identified in the initial Nmap scan.

