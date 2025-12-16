# Slowloris Application-Layer DoS — Targeted Research (Lab)

**Project type:** Targeted Research / Exploit (sandbox only)  
**Role:** Attacker (I executed and documented the attack)  
**Author:** I did this (Vansh Bhardwaj)  
**Date:** December 2, 2025

---

## High-level summary
I executed an application-layer Denial-of-Service (DoS) against an Apache2 web server inside an isolated VirtualBox sandbox using the classic Slowloris technique. I configured Splunk to ingest the Apache `access.log` and `error.log`, tuned the attacking VM to allow many sockets, launched Slowloris from Kali, and validated the attack using Splunk, host-level tooling, and direct HTTP tests. This repo contains the evidence, exact commands, detection queries, and recommended mitigations.

---

## Skills demonstrated
- Application-layer DoS emulation (Slowloris)  
- VirtualBox sandbox and pfSense topology management  
- SIEM ingestion and monitoring (Splunk)  
- Reconnaissance with `nmap`  
- Network forensics (Splunk)  
- Host forensics (Apache logs)  
- Reproducible documentation and defensive recommendations

---

## Why this matters
Slowloris-style attacks can render web services unavailable while using very little bandwidth. Demonstrating both attack execution and detection in Splunk shows I can reason about attack techniques and operationalize detection/response — a key skill for SOC analysts, security engineers, and GRC practitioners.

---

## Lab environment & brief topology
I built an isolated sandbox in VirtualBox and used pfSense as the firewall/router between two /24 subnets.

**Network A (internal)** — `192.168.1.1/24`  
- Ubuntu (Web Server / Victimm / Apache2): **`192.168.1.10`**  
- Windows XP (Workstation): `192.168.1.15`

**Network B (attacker / external)** — `192.168.2.1/24`  
- Kali Linux (Attacker): **`192.168.2.16`**

**Router / Firewall**: pfSense (LAN <-> Network A, OPT1/WAN <-> Network B)  
**Monitoring**: Splunk (configured to ingest `/var/log/apache2/access.log` and `/var/log/apache2/error.log`)

<img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/9eeca3498fec829bbb37bbe6dd78d8206148d887/assets/Topology.png" alt="Lab Network Topology" width="500">


---

## Splunk: what I configured & how I validated ingestion
I configured Splunk to ingest the two Apache log files from the victim VM:

- `/var/log/apache2/access.log` (sourcetype: `apache:access`)  
- `/var/log/apache2/error.log` (sourcetype: `apache:error`)


# Pre-Attack Assessment 

**From the attacker (Kali):**
- **Quick port/service scan**
  
  <img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/503a6d2df94b20090fcf65b02b8dff522f44c16c/assets/nmap-before.png" alt="Nmap Scan Before Attack" width="500">

- ****Confirm HTTP service****
  ```bash
  $ curl http://192.168.1.10/testsplunk
  ```

  <div style="display: flex; gap: 20px;">
    <img 
    src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/e1d86277e7dc1562869b93d4e2f61481ca64b43f/assets/curl-before.png"
    alt="Curl Output Before Attack"
    width="425"
    >
    <img 
    src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/main/assets/curlLog-before.png"
    alt="Curl Log Before Attack"
    width="460"
    >
  </div>


# Pre-attack tuning (what I changed on Kali)(((
- ***Slowloris uses lots of sockets***
- ***So I increased file-descriptor limits on Kali before running***

  - Soft Limit
    ```bash
    $   ulimit -S -n 8192
    ```
    
  - Hard Limit 
    ```bash
    $   ulimit -S -n 16384
    ```
    
# Executing Slowloris Attack 
***Command Used***

    
    $   slowloris -v -0 80 -s 1500 --sleeptime 192.168.1.10
    
    
***Command Explanation***
- -v = verbose
- -p 80 = HTTP port
- -s 1500 = socket count (tuned to VM limits)
- --sleeptime 10 = seconds between header refreshes

***Slowloris Running***

<img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/f32ab6abddd5fb991a9310c2a7fe3ab9de1c7d00/assets/slowloris-attack.png" alt="Slowloris Attack Visualization" width="500">


***How I verified this was Slowloris (evidence & indicators)***

- I confirmed the attack type by correlating two independent indicators (these are exactly the things I used):

**1. Apache error.log evidence**

- Splunk showed MaxRequestWorkers reached and similar exhaustion errors while the attack was running. 

  <img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/f32ab6abddd5fb991a9310c2a7fe3ab9de1c7d00/assets/splunk-maxrequestworkers.png" alt="Splunk Max Request Workers Chart" width="600">


**2. Application-level availability effect**

- curl from another terminal stalled
- while ICMP (ping) still worked
- indicating application-layer (HTTP) exhaustion rather than network down 

  <img src="https://raw.githubusercontent.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/f32ab6abddd5fb991a9310c2a7fe3ab9de1c7d00/assets/ping-after.png" alt="Ping After Attack" width="500">


Because all both indicators aligned (error logs and application-level failure), I concluded the observed behavior was Slowloris.


# Analysis — why the attack worked

***Quick executive takeaway***

- This Slowloris attack succeeded by exhausting Apache’s request‑handling capacity, not by overwhelming network bandwidth. I demonstrated the attack, observed the failure mode in real time, and mapped it directly to defensive controls.

***What the attack does*** 
- Opens many HTTP connections to Apache
- Sends incomplete HTTP headers very slowly
- keeps connections alive indefinitely by periodically refreshing headers
- Prevents requests from ever completing
  
***What Apache did internally***
- Assigned a worker thread/process to each incoming connection
- Waited for headers to fully arrive before releasing the worker
- Allowed connections to remain open due to permissive timeout settings
- Eventually reached the MaxRequestWorkers limit

***Why hypothetical users were impacted***
- Once all workers were consumed, Apache could not process new HTTP requests
- clients experienced stalled or timed‑out connections
- ICMP traffic continued to function, confirming the host and network were still up
- This confirmed an application‑layer denial of service, not a network outage

***Why the environment was vulnerable***
- Apache worker limits were low relative to the attack’s socket volume
- Header and keep‑alive timeouts allowed connections to persist too long
- Kali’s increased file‑descriptor limit (ulimit -n) enabled thousands of concurrent sockets with minimal bandwidth usage

***How I confirmed it was Slowloris***
- Apache error logs showed worker exhaustion (MaxRequestWorkers reached)
- Packet captures showed partial and incomplete HTTP headers
- Splunk correlated Apache errors with service unavailability during the attack window
- Live testing showed HTTP failure while other protocols remained responsive

***Security impact***
- A small number of attacker resources caused full service degradation
- Traditional network‑volume monitoring would not detect this attack
- Proper application‑layer logging and alerting were required to identify the root cause

***Why this matters to security roles***
- Demonstrates understanding of application‑layer DoS mechanics
- Shows ability to validate attacks using SIEM, packet analysis, and live service testing
- Connects technical behavior to concrete mitigation strategies
- Reflects real SOC, security engineering, and GRC decision‑making skills


# Ethical notice & scope

All testing was performed inside an authorized, isolated lab environment. This documentation is for defensive education only. Do not run these steps against systems you do not own or are not authorized to test.

