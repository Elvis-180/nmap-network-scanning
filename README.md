#  Nmap Network Scanning 
A hands-on SOC home lab detecting and investigating Nmap scans using Snort IDS, pfSense, Sysmon, and Splunk.

---

## Overview
This project replicates a real Security Operation Center (SOC) environment in a controlled home lab. The lab is designed to practice the full detect -> investigate -> respond cycle against live Nmap scanning activity originating from an external Kali Linux machine (Attacker)

---

## What this lab covers
- Active network reconnaissance using Nmap from an external attacker (Kali Linux)
- Perimeter traffic looging via pfsense firewall
- Intrusion detection using snort IDS on ubuntu server
- Log correlation and threat hunting in splunk
- Firewall rule creation and enforcement

  ---

  ## Lab Architecture
        [ Kali Linux 192.168.2.5 (OPT1) ] 
                   |
                   | Nmap Scan
                   v
      ------------------------------------
       |            pfSense              |
       | Firewall + Traffic Logging      |
       -----------------------------------
               |                    |
               |                    |
          Internal LAN          Logging Forward
               |                    |
       ------------------           |
       |      |         |           |
       | [Windows DC]   |           |
       |   [+ Sysmon]   |           |
       |      [+ UF]    |           |
       ------------------           |
               |                    |
               |                    |
      [Ubuntu Server + Snort IDS]   |
              |          ---- -------       
              v          |
       ---------------------
      | Splunk Enterprise  |
      | SIEM / Correlation |
      ---------------------

  ---

  ## Tools Used
![Kali](https://img.shields.io/badge/Kali_Linux-Attacker-red?style=flat-square&logo=kalilinux&logoColor=white)
![Splunk](https://img.shields.io/badge/Splunk-SIEM-orange?style=flat-square&logo=splunk&logoColor=white)
![Snort](https://img.shields.io/badge/Snort-IDS-blue?style=flat-square)
![pfSense](https://img.shields.io/badge/pfSense-Firewall-blueviolet?style=flat-square)

---

  ## Pfsense Firewall Rule 
   | Action  |    Pass|
  |----------|-------|
   | Interface |  LAN|
  |  Protocol |   Any|
  |  Source |     Kali Linux IP (192.168.2.5)|
  |  Destination| Any|
   | Description | Allow Kali Attacker|

---

  ## Scan Command Used
  - nmap -sS -sV -sC -A -T4 -p- --open-- -oA full_recon 192.168.1.1 (This is done on Kali Linux terminal)
      |    Flag      |          Name           |                        What It Does                                        |
    |--------------|-------------------------|----------------------------------------------------------------------------|
    |    -sS       |  SYN Stealth Scan       | Sends SYN packets without completing the TCP handshake  stealthy and fast |
    |    -sV       |  Version Detection      | Identifies the service and version running on each open port               |
    |    -sC       |  Default Scripts        | Runs Nmap built-in NSE scripts for extra service enumeration               |
    |    -A        |  Aggressive Scan        | Enables OS detection, version detection, scripts and traceroute in one flag|
    |    -T4       |  Timing Template        | Fast scan speed  suitable for a single host                               |
    |    -p-       |  All Ports              | Scans all 65,535 ports instead of the default 1,000                        |
    |    -oA       |  Output All Formats     | Saves results in normal, XML and grepable formats simultaneously           |
    |  full_recon  |  Output Filename        | Name given to the three saved scan result files                            | 
    | 192.168.1.1  |  Target IP              | The host being scanned  pfSense firewall in this lab                      |
        

    ---


##  Investigation Findings

### Scan Overview
| Field | Details |
|-------|---------|
| Scan Command | `nmap -sS -sV -sC -A -T4 -p- --open -oA full_recon 192.168.1.1` |
| Target | 192.168.1.1 |
| Scan Date | 2026-05-12 00:13 |
| Scan Duration | 212.23 seconds |
| Host Status | Up (0.0094s latency) |
| Filtered Ports | 65,509 filtered TCP ports |
| Device Type | General Purpose |
| Domain | cis.net |
| DNS Name | Server.cis.net |

---

###  OS Detection Results
| OS | Confidence |
|----|------------|
| Microsoft Windows Server 2022 | 96% |
| Microsoft Windows 11 24H2 | 91% |
| Microsoft Windows Server 2016 | 91% |
| Microsoft Windows 11 21H2 | 90% |

>  OSScan results may be unreliable — could not find at least 1 open and 1 closed port

---

###  Open Ports Discovered
| Port | State | Service | Version | Risk |
|------|-------|---------|---------|------|
| 53/tcp | Open | DNS | Simple DNS Plus |  Medium |
| 80/tcp | Open | HTTP | Microsoft IIS httpd 10.0 |  Medium |
| 88/tcp | Open | Kerberos | Microsoft Windows Kerberos |  Critical |
| 135/tcp | Open | MSRPC | Microsoft Windows RPC |  Medium |
| 139/tcp | Open | NetBIOS | Microsoft Windows NetBIOS-ssn |  Medium |
| 389/tcp | Open | LDAP | Microsoft AD LDAP (Domain: cis.net) | Critical |
| 445/tcp | Open | SMB | Microsoft-ds |  Critical |
| 464/tcp | Open | Kpasswd | Kpasswd5 |  Medium |
| 593/tcp | Open | HTTP-RPC | Microsoft RPC over HTTP 1.0 |  Medium |
| 636/tcp | Open | TCPWrapped | — |  Medium |
| 3268/tcp | Open | LDAP | Microsoft AD LDAP (Domain: cis.net) |  Critical |
| 3269/tcp | Open | TCPWrapped | — |  Medium |
| 3389/tcp | Open | RDP | Microsoft RDP ms-wbt-server |  Critical |
| 5985/tcp | Open | WinRM | Microsoft HTTPAPI httpd 2.0 |  Critical |
| 9389/tcp | Open | .NET | Message Framing |  Medium |
| 49664-49712/tcp | Open | MSRPC | Microsoft Windows RPC |  Medium |

---

###  Target Identification
| Field | Result |
|-------|--------|
| Target Name | CIS |
| NetBIOS Domain | CIS |
| NetBIOS Computer | SERVER |
| DNS Domain | cis.net |
| DNS Computer | Server.cis.net |
| DNS Tree | cis.net |
| OS Build | 10.0.20100 |
| System Time | 2026-05-11 23:45:57 |
| Service Info | Host: SERVER — OS: Windows |

---

###  SMB Analysis
| Field | Result |
|-------|--------|
| SMB2 Date | 2026-05-11 23:45:57 |
| Clock Skew | Mean 13m01s — Deviation 0s |
| SMB2 Security Mode | 3.1.1 |
| Message Signing | Enabled and Required |

---

###  Traceroute
| Hop | RTT | Address |
|-----|-----|---------|
| 1 | 8.89 ms | 192.168.2.1 |
| 2 | 12.54 ms | 192.168.1.1 |


---

###  Critical Findings
| # | Finding | Severity |
|---|---------|----------|
| 1 | SMB port 445 open — lateral movement and ransomware risk |  Critical |
| 2 | RDP port 3389 exposed — brute force and remote access risk |  Critical |
| 3 | LDAP ports 389 and 3268 open — Active Directory enumeration possible |  Critical |
| 4 | Kerberos port 88 open — Kerberoasting attack possible | Critical |
| 5 | WinRM port 5985 open — remote command execution risk |  Critical |
| 6 | IIS 10.0 on port 80 with TRACE method enabled |  Medium |
| 7 | Clock skew of 13 minutes detected on target |  Medium |
| 8 | OS detection unreliable — non-ideal scan conditions |  Low |


###  Alerts Triggered During Scan

| Alert | Source | Cause | Severity |
|-------|--------|-------|----------|
|  DDoS Detected | pfSense + Snort + Splunk SIEM | `-T4 -p-` scanned all 65,535 ports aggressively |  Critical |
| SNMP Enumeration | Snort IDS | Nmap NSE scripts probing SNMP service |  Medium |

---

###  Alerts Triggered During Scan

| Alert | Source | Cause | Severity |
|-------|--------|-------|----------|
|  DDoS Detected | pfSense + Snort + Splunk SIEM| `-T4 -p-` flooded single host with SYN packets | Critical |
| SNMP Enumeration | Snort IDS + Splunk SIEM| Nmap `-sC` scripts probed SNMP port 161 |  Medium |

---

###  Why It Triggered

**DoS Alert:**
- Flag `-T4` with `-p-` sent SYN packets across all 65,535 ports
  rapidly to one device
- pfSense was overwhelmed by the volume of packets hitting it
- Snort flagged it as a Denial of Service attack
- This is why the scan took **212.23 seconds** to complete

**SNMP Alert:**
- Nmap `-sC` automatically ran default scripts including SNMP probing
- Snort detected the SNMP community string enumeration on port 161
- SNMP was exposing device configuration information

---

###  Recommendations
| # | Action | Priority |
|---|--------|----------|
| 1 | Block RDP (3389) on pfSense — allow trusted IPs only |  Immediate |
| 2 | Block SMB (445) from external network on pfSense | Immediate |
| 3 | Restrict LDAP (389, 3268) to internal network only |  Immediate |
| 4 | Enable Kerberos armoring to prevent Kerberoasting |  Immediate |
| 5 | Disable or restrict WinRM (5985) to admin IPs only |  Immediate |
| 6 | Disable TRACE method on IIS web server |  High |
| 7 | Sync server clock to fix 13 minute skew |  High |
| 8 | Enable Snort IPS mode to auto-block future scans |  High |

---

##  Lessons Learned

### 1. Scan Timing Matters
| Lesson | Detail |
|--------|--------|
| What happened | `-T4 -p-` triggered a DoS alert on a single host |
| What I learned | Aggressive timing causes real stress even on one device |
| What to do next | Use `-T3` for single hosts and `-T2` for whole networks |

---

### 2. Know Your Flags Before You Scan
| Lesson | Detail |
|--------|--------|
| What happened | `-sC` silently triggered SNMP enumeration |
| What I learned | Default scripts run automatically without warning |
| What to do next | Always research what NSE scripts `-sC` runs before using it |

---

### 3. Not Every Alert is an Attack
| Lesson | Detail |
|--------|--------|
| What happened | DoS and SNMP alerts fired during an authorized scan |
| What I learned | Context determines whether an alert is real or false positive |
| What to do next | Always investigate before escalating an alert |

---

### 4. Exposed Services are Real Risk
| Lesson | Detail |
|--------|--------|
| What happened | SMB, RDP, LDAP, Kerberos, WinRM all found open |
| What I learned | A real attacker would use these to gain access |
| What to do next | Restrict all unnecessary services via pfSense firewall rules |

---

### 5. Attacker Perspective Improves Defense
| Lesson | Detail |
|--------|--------|
| What happened | Scanning from Kali revealed what an attacker would see |
| What I learned | You cannot defend what you cannot see |
| What to do next | Run regular authorized scans to audit your own network |

---

### 6. Log Correlation is Everything
| Lesson | Detail |
|--------|--------|
| What happened | pfSense, Snort all logged the same event differently |
| What I learned | One source alone gives incomplete picture |
| What to do next | Always correlate multiple log sources in Splunk |

---

### 7. False Positives Need Investigation
| Lesson | Detail |
|--------|--------|
| What happened | Two alerts triggered — both were false positives |
| What I learned | Blindly trusting alerts wastes time and causes alert fatigue |
| What to do next | Tune Snort rules to whitelist authorized scan sources |

---

### 8. Documentation 
| Lesson | Detail |
|--------|--------|
| What happened | Every finding was recorded and analyzed |
| What I learned | Without documentation findings have no value |
| What to do next | Always record scan date, command, findings and verdict |

---

### 9. Kerberos and LDAP Exposure is Dangerous
| Lesson | Detail |
|--------|--------|
| What happened | Ports 88 and 389 were open on the domain controller |
| What I learned | Open Kerberos enables Kerberoasting attacks against AD |
| What to do next | Restrict Kerberos and LDAP to internal trusted hosts only |

---

### 10. Scanning is Just the Beginning
| Lesson | Detail |
|--------|--------|
| What happened | Scan revealed the attack surface |
| What I learned | Scanning shows what is exposed — not how to fix it |
| What to do next | Follow every scan with remediation and re-verification |

---

### Key Takeaway

> A SOC Analyst does not just run tools, they understand
> what the tools do, why alerts fire, whether alerts are
> real, and what to do next. This lab proved that the most
> important skill in cybersecurity is not scanning 
> it is thinking critically about what the results mean.



