# Slowloris Application-Layer DoS — Targeted Research (Lab)  
**Repo:** https://github.com/VanshBhardwaj1945/slowloris-dos-attack-lab-  
**Project type:** Targeted research (authorized sandbox only)  
**Role:** Test executor (authorized lab)  
**Date:** December 2, 2025

## Table of Contents

1. [Overview](#overview)  
2. [Skills Demonstrated](#skills-demonstrated)  
3. [Why This Matters](#why-this-matters)  
4. [Lab Environment & Topology](#lab-environment--topology)  
5. [Splunk Configuration & Ingestion Validation](#splunk-configuration--ingestion-validation)  
6. [Pre-Attack Assessment](#pre-attack-assessment)  
7. [Pre-Attack Tuning (Kali)](#pre-attack-tuning-kali)  
8. [Executing the Slowloris Attack](#executing-the-slowloris-attack)  
9. [Evidence & Attack Verification](#evidence--attack-verification)  
   - [Apache Error Log Evidence](#apache-error-log-evidence-splunk)  
   - [Application Availability Impact](#application-availability-impact)  
10. [Technical Analysis — Why the Attack Worked](#technical-analysis--why-the-attack-worked)  
11. [Detection Engineering Insights](#detection-engineering-insights)  
12. [Security Impact](#security-impact)  
13. [Mitigation Recommendations](#mitigation-recommendations)  
14. [Summary Outcomes](#summary-outcomes)  
15. [Ethical Notice](#ethical-notice)


## Overview
A controlled study of application-layer Denial-of-Service (Slowloris) against an Apache2 web server in an isolated VirtualBox sandbox. The exercise focused on attack mechanics, SIEM ingestion and detection, packet- and host-level telemetry analysis, and operational mitigations.

## Skills demonstrated
- Application-layer DoS emulation (Slowloris)  
- VirtualBox sandbox and pfSense topology management  
- SIEM ingestion and detection engineering (Splunk)  
- Network discovery and reconnaissance (Nmap)  
- Packet- and host-level forensics (Wireshark, tcpdump, Apache logs)  
- Reproducible documentation and defensive recommendations

## Why this matters
Slowloris-style attacks can render web services unavailable while using very little bandwidth. Demonstrating both attack execution and detection shows the ability to reason about attacker techniques, validate telemetry, and operationalize detection and response.

---

## Lab environment & topology
- **Network A (internal)** — `192.168.1.0/24`  
  - Ubuntu (Web server / Apache2): `192.168.1.10`  
  - Windows XP (Workstation): `192.168.1.15`  
- **Network B (attacker / external)** — `192.168.2.0/24`  
  - Kali Linux (Attacker): `192.168.2.16`  
- **Router / Firewall:** pfSense (LAN <-> Network A, OPT1/WAN <-> Network B)  
- **Monitoring:** Splunk (ingesting `/var/log/apache2/access.log` and `/var/log/apache2/error.log`)

<img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/9eeca3498fec829bbb37bbe6dd78d8206148d887/assets/Topology.png" alt="Lab Network Topology" width="600">

---

## Splunk: configuration & ingestion validation
Configured Splunk to ingest Apache logs from the victim VM:

- `/var/log/apache2/access.log` (sourcetype: `apache:access`)  
- `/var/log/apache2/error.log` (sourcetype: `apache:error`)  

Validation steps:
1. Confirmed log files appear in Splunk with expected sourcetypes.  
2. Built initial search to surface `MaxRequestWorkers` and related Apache errors.  
3. Correlated timestamps between Splunk events and live availability tests.

---

## Pre-attack assessment
From the attacker VM (Kali):

- Performed quick port/service enumeration with Nmap.  
  <img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/503a6d2df94b20090fcf65b02b8dff522f44c16c/assets/nmap-before.png" alt="Nmap Scan Before Attack" width="600">

- Verified HTTP service with curl:
```bash
curl http://192.168.1.10/testsplunk
```
<img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/e1d86277e7dc1562869b93d4e2f61481ca64b43f/assets/curl-before.png" alt="Curl Output Before Attack" width="425">

---

## Pre-attack tuning (Kali)
Slowloris uses many sockets; increased file-descriptor limits before running:

```bash
# Soft limit
ulimit -S -n 8192

# Hard limit
ulimit -H -n 16384
```

## Executing the Slowloris attack

Command used:
```bash
slowloris -v -p 80 -s 1500 --sleeptime 10 192.168.1.10
```

Command explanation:
- ``` -v ``` = verbose output
- ``` -p 80 ```= HTTP port
- ``` -s 1500 ```= number of sockets (tuned to VM limits)
- ``` --sleeptime 10 ``` = seconds between header refresh attempts

Attack visualization:

<img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/f32ab6abddd5fb991a9310c2a7fe3ab9de1c7d00/assets/slowloris-attack.png" width="600">

## Evidence & Attack Verification

### Apache Error Log Evidence (Splunk)
Splunk detected worker exhaustion indicators during the attack window.

<img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/f32ab6abddd5fb991a9310c2a7fe3ab9de1c7d00/assets/splunk-maxrequestworkers.png" width="600">

---

### Application Availability Impact
Application-layer failure was confirmed using cross-terminal testing.

- HTTP requests stalled during the attack  
- ICMP traffic remained functional  

This confirmed application-layer resource exhaustion rather than a full network outage.

<img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/f32ab6abddd5fb991a9310c2a7de1c7d00/assets/ping-after.png" width="500">

---

## Technical Analysis — Why the Attack Worked

Slowloris succeeds by exhausting application resources rather than bandwidth.

### Attack Mechanics
- Opens many concurrent HTTP connections  
- Sends HTTP headers extremely slowly  
- Keeps connections alive using periodic header refreshes  
- Prevents Apache from completing request processing  

### Server-Side Behavior
- Apache assigns worker threads per connection  
- Workers remain locked waiting for full HTTP headers  
- Keep-alive and timeout configurations allowed connections to persist  
- Eventually MaxRequestWorkers was reached  

### Environmental Vulnerability Factors
- Worker limits were low relative to socket volume  
- Permissive keep-alive and timeout settings  
- Increased attacker socket capacity via ulimit tuning  

---

## Detection Engineering Insights

Attack detection was validated using multiple independent signals.

### SIEM Correlation
- Apache error logs  
- Availability metrics  
- Time-synchronized attack activity  

### Packet-Level Indicators
- Partial or incomplete HTTP headers  
- Long-lived TCP sessions without completed requests  

---

## Security Impact

- Demonstrated that low-bandwidth application-layer attacks can cause full service degradation  
- Reinforced the importance of application telemetry monitoring  
- Showed need for layered security beyond network monitoring  

---

## Mitigation Recommendations

### Application Hardening
- Tune Apache worker and timeout thresholds  
- Implement per-IP rate limiting  
- Restrict persistent connections where possible  

### Architecture Defenses
- Deploy reverse proxies or load balancers  
- Add upstream connection throttling  

### Detection Improvements
- Create SIEM alerts for worker exhaustion patterns  
- Monitor abnormal connection persistence behavior  

---

## Summary Outcomes
- Successfully executed and documented Slowloris DoS testing in an isolated environment  
- Built detection validation workflows using SIEM and telemetry  
- Produced defensive guidance and reproducible test evidence  

## Ethical Notice
All testing was performed in an authorized, isolated lab environment for defensive research purposes only. Do not reproduce on systems you do not own or have permission to test.
