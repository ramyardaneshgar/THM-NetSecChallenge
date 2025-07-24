# THM-NetSecChallenge
Network security recon and enumeration using `nmap`, `hydra`, `ftp`, and `telnet` to uncover open ports, banner data, and weak credentials across standard and non-standard services.

By Ramyar Daneshgar 


## **Objective**

I approached this challenge as a focused assessment of my ability to conduct early-stage network reconnaissance, service enumeration, and credential-based access testing in a simulated environment. The scenario reflects a typical black-box penetration testing engagement, where the adversary is provided no credentials or system context and must leverage foundational tools to enumerate exposed services, identify misconfigurations, and escalate access through authentication weaknesses. My methodology prioritized stealth scanning, complete port coverage, and logical attack chaining to simulate real-world adversarial behavior.

---

## **\[2.1] What is the highest port number open under 10,000?**

**Command Used:**

```bash
nmap -sS -sV -p1-10000 -vv <target-ip>
```

### Why this command:

* `-sS`: TCP SYN scan is used for stealth. It sends SYN packets and monitors responses without completing the handshake, reducing detection by intrusion detection systems (IDS).
* `-sV`: Enables version detection, helping identify the application running on each port.
* `-p1-10000`: Focuses the scan on the first 10,000 TCP ports instead of just the top 1000 (default).
* `-vv`: Provides verbose output to analyze scan progress and results in detail.

### Context:

In real-world environments, attackers often begin by scanning a broader port range. Many admins configure services on non-standard but still low-numbered ports (e.g., 8080, 8888) in hopes of "security through obscurity." Identifying such ports early enables focused enumeration.

### Result:

Port `8080` was identified as open. This port is commonly used for web applications, test environments, or administrative consoles that are separated from production traffic.

**Answer:** `8080`

---

## **\[2.2] What open port is above 10,000?**

**Command Used:**

```bash
nmap -sS -sV -p- -vv <target-ip>
```

### Why:

* `-p-`: Tells Nmap to scan all 65,535 TCP ports.
* This helps identify services that may be intentionally running on high-numbered ports to evade detection by automated scanners.

### Context:

Administrators may run services like FTP, SSH, or even custom web apps on high ports to bypass basic firewall rules or script kiddie tools. From an adversarial standpoint, these ports must be included in any thorough assessment.

### Result:

Port `10021` was identified as open. The service running was later determined to be FTP (via banner grabbing), indicating intentional misconfiguration or obfuscation.

**Answer:** `10021`

---

## **\[2.3] How many TCP ports are open?**

**Approach:**
From the previous full-range scan, I counted each line in the Nmap output that contained the word “open.” These lines reflect successfully established TCP connections.

**Result:**
There were a total of 6 open TCP ports, which forms the attack surface for this host.

**Answer:** `6`

---

## **\[2.4] What is the flag in the HTTP server header?**

**Command Used:**

```bash
nmap -sV --script=http-headers -p80 <target-ip>
```

### Why:

* `--script=http-headers`: Part of the Nmap Scripting Engine (NSE). It retrieves HTTP headers which may include hidden metadata or flags.
* `-p80`: Targeted port where HTTP was previously detected.

### Context:

Misconfigured servers can inadvertently expose sensitive metadata in headers, including internal version numbers, custom fields, or debug values. During CTFs, this location is often used to hide flags.

### Result:

The `Server` response header contained a flag: `THM{web_server_25352}`

**Answer:** `THM{web_server_25352}`

---

## **\[2.5] What is the flag in the SSH server header?**

**Command Used:**

```bash
nmap -sV --script=ssh2-enum-algos -p22 <target-ip>
```

### Why:

* This NSE script enumerates supported SSH algorithms. Some banners or algorithm responses contain identifiers, messages, or flags.
* SSH, by default, presents its version and sometimes additional metadata before login is attempted.

### Context:

Although SSH is more secure by default compared to protocols like FTP, its banners can still leak sensitive information, especially in non-production environments.

### Result:

Flag found in response: `THM{946219583339}`

**Answer:** `THM{946219583339}`

---

## **\[2.6] What is the version of the FTP server running on a non-standard port?**

**Command Reference:**
This information was already collected during the all-ports scan (`nmap -sS -sV -p-`). The service running on port `10021` was FTP, and version fingerprinting identified it as:

**Result:** `vsftpd 3.0.3`

### Context:

vsftpd (Very Secure FTP Daemon) is generally hardened, but older or misconfigured versions may be vulnerable to backdoor access or command injection. Identifying the version is critical for vulnerability matching during later exploitation phases.

**Answer:** `vsftpd 3.0.3`

---

## **\[2.7] What is the flag found in one of the two user accounts (via FTP)?**

### Initial Information:

Two usernames (`eddie` and `quinn`) were discovered via social engineering, suggesting credential-based attack vector.

### Step 1 – Brute Force `eddie`

```bash
hydra -l eddie -P rockyou.txt ftp://<target-ip>:10021
```

* Password found: `jordan`
* Logged in via `ftp <target-ip> 10021`, but no sensitive files were present in eddie’s directory.

### Step 2 – Brute Force `quinn`

```bash
hydra -l quinn -P rockyou.txt ftp://<target-ip>:10021 -v
```

* Password found: `andrea`
* On successful login, I listed the directory contents and found a file called `ftp_flag.txt`.

### File Retrieval:

```ftp
get ftp_flag.txt
```

* Opening the file revealed the flag: `THM{321452667098}`

### Context:

Brute-forcing exposed accounts is a common post-recon tactic. Here, use of weak passwords and lack of account lockout enabled rapid compromise. The presence of the flag in user space demonstrates how exposed FTP services can become initial access vectors.

**Answer:** `THM{321452667098}`

---

## **\[2.8] What is the flag from the web challenge on port 8080?**

**Approach:**
Navigated to `http://<target-ip>:8080` in the browser.

### Observation:

A custom JavaScript-based challenge was embedded in the site, designed to require interaction. Solving the mini-puzzle revealed a final flag.

**Answer:** `THM{f7443f99}`

### Context:

This represents a real-world equivalent of client-side logic flaws or hidden content accessed via DOM manipulation. Attackers often analyze scripts for hidden form fields, obfuscated flags, or bypass conditions.

---

## **Lessons Learned**

### 1. **Port Obfuscation ≠ Security**

While placing FTP on port 10021 was likely intended to reduce exposure, it was immediately uncovered through comprehensive scanning. Obscurity alone offers no security and can lead to complacency in access controls.

### 2. **Thorough Enumeration Pays Off**

Many vulnerabilities are found not through exploitation, but by simply being thorough. Running all-port scans and inspecting headers revealed sensitive details that would have been missed using default scan settings.

### 3. **Default Credentials and Weak Passwords Remain Common**

Hydra succeeded due to reliance on `rockyou.txt`, a widely known password list. This underscores how dangerous weak or reused passwords can be, especially on externally accessible services.

### 4. **Service Metadata Can Leak Flags or Versions**

Both HTTP and SSH headers contained embedded flags. In enterprise environments, this translates to leaked version info that can be mapped to known CVEs.

### 5. **Structured Recon is Foundational**

This assessment emphasized the importance of structured reconnaissance before attempting any exploits. By identifying open ports, fingerprinting services, and analyzing banner data, I gained a complete picture of the target’s network posture without sending a single exploit.
