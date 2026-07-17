# Network Segmentation & Application-Layer DoS Lab

A hands-on offensive-and-defensive lab built inside a single VirtualBox sandbox. It runs in two parts that share the same environment:

- **Part 1 — Network Segmentation & Access Control.** Stand up a multi-subnet network separated by a pfSense firewall, design an access-control matrix, enforce it with router- and host-level rules, and validate enforcement with active scanning and packet captures.
- **Part 2 — Slowloris Application-Layer DoS.** Use that same segmented sandbox to run a controlled low-bandwidth DoS attack against an Apache web server, then detect and analyze it with Splunk, packet captures, and host telemetry — and document the mitigations.

Together they tell one story: first stand up and lock down a realistic network, then attack a service inside it and prove the attack is visible in your telemetry.

- **Project type:** Targeted research (authorized, isolated sandbox only)
- **Role:** Lab builder & test executor
- **Date:** Fall 2025

> **Note on addressing.** The two exercises were run as separate sessions on the same sandbox design, so the exact subnet numbering differs between them — Part 1 uses `192.168.10.0/24` / `192.168.20.0/24`; Part 2 uses `192.168.1.0/24` / `192.168.2.0/24`. Each part documents the addressing it actually used.

---

## Table of Contents

**Part 1 — Network Segmentation & Access Control**

1. [Overview](#part-1-overview)
2. [Sandbox Components](#sandbox-components)
3. [Tools & Technologies](#tools--technologies-used)
4. [Network Topology](#network-topology)
5. [Setup](#setup-high-level-steps)
6. [Discovery & Baseline Validation](#discovery--baseline-validation)
7. [Security Policy Implementation](#security-policy-implementation)
8. [Post-Implementation Verification](#post-implementation-verification)
9. [Host-Level Firewall](#host-level-firewall-server-a1)
10. [Part 1 Outcomes](#part-1-outcomes)

**Part 2 — Slowloris Application-Layer DoS**

11. [Overview](#part-2-overview)
12. [Skills Demonstrated](#skills-demonstrated)
13. [Why This Matters](#why-this-matters)
14. [Lab Environment & Topology](#lab-environment--topology)
15. [Splunk Configuration & Ingestion Validation](#splunk-configuration--ingestion-validation)
16. [Pre-Attack Assessment](#pre-attack-assessment)
17. [Pre-Attack Tuning (Kali)](#pre-attack-tuning-kali)
18. [Executing the Slowloris Attack](#executing-the-slowloris-attack)
19. [Evidence & Attack Verification](#evidence--attack-verification)
20. [Technical Analysis — Why the Attack Worked](#technical-analysis--why-the-attack-worked)
21. [Detection Engineering Insights](#detection-engineering-insights)
22. [Security Impact](#security-impact)
23. [Mitigation Recommendations](#mitigation-recommendations)
24. [Part 2 Outcomes](#part-2-outcomes)

[Ethical Notice](#ethical-notice)

---

# Part 1 — Network Segmentation & Access Control

## Part 1 Overview

A repeatable, VirtualBox-based sandbox for implementing, enforcing, and verifying network security policy. The environment models an internal company network and an external attacker network separated by a pfSense virtual router. The goal is to show how router-level and host-level controls combine into defense-in-depth, and how to validate that enforcement with active scanning and packet captures rather than assuming the rules work.

## Sandbox Components

- **Network A (Internal / Company)** — Ubuntu Desktop (server) and Windows XP (workstation)
- **Network B (External / Attacker)** — Kali Linux (attacker / scanner) and Windows 95 (legacy)
- **Router / Firewall** — pfSense virtual appliance connecting Network A and Network B

**Notes & lessons learned**

- Resolved legacy VM patching and NIC misconfiguration issues to keep the environment reproducible.
- Identified policy items that required host-level enforcement when gateway rules alone were insufficient.
- Captured configuration and verification artifacts so the sandbox can be re-created for future testing.

---

## Tools & Technologies Used

| Category | Tools |
|---|---|
| **Virtualization** | VirtualBox (VM creation, NIC binding, snapshots) |
| **Router & firewall** | pfSense (interface configuration, rule authoring, logging) |
| **Operating systems** | Ubuntu, Kali Linux, Windows XP, Windows 95 (legacy) |
| **Network analysis & scanning** | Wireshark, tcpdump, Nmap / Zenmap |
| **Services** | Apache (HTTP), OpenSSH (SSH) |
| **Host hardening** | iptables (Ubuntu) |
| **Testing & validation** | curl, ping, ssh |

---

## Network Topology

<img src="https://i.imgur.com/0PbmIyM.png" alt="Network segmentation topology: two subnets separated by a pfSense router" width="60%">

*A.1 = Ubuntu (server), A.2 = Windows XP (workstation), pfSense = router, B.1 = Kali, B.2 = Windows 95 (legacy).*

---

## Setup (High-Level Steps)

1. Installed Oracle VirtualBox and provisioned VMs for Ubuntu, Kali, Windows XP, and Windows 95.
   <img src="https://i.imgur.com/iG1MSqE.png" alt="VirtualBox VM inventory for the lab" width="60%">

2. Configured pfSense with two interfaces — `LAN` for Network A and `OPT1` for Network B.
   <img src="https://i.imgur.com/ccb1EGK.png" alt="pfSense interface assignment for LAN and OPT1" width="60%">

3. Assigned static IPs and verified NIC-to-subnet mappings.
4. Installed Apache and OpenSSH on the internal server (A.1) and confirmed the services were reachable.
5. Installed Wireshark and Nmap on the attacker and server VMs for traffic capture and scanning.
6. Validated baseline connectivity with `ping`, `curl`, and `ssh`.
7. Documented environment quirks (snapshots + notes) to keep the build reproducible.

---

## Discovery & Baseline Validation

Ran discovery scans and packet captures to establish a clear baseline *before* any policy changes, so post-rule behavior could be compared against a known-good reference.

- Nmap scans from the attacker VM (Kali) to enumerate services and open ports.
- Wireshark / tcpdump captures during `ping`, `curl`, and `ssh` to record normal traffic patterns.
- Baseline evidence saved for before/after comparison.

### Selected Baseline Captures

**Ping and curl from attacker → server**
<img src="https://i.imgur.com/Cfs9b6y.png" alt="Baseline ping and curl from attacker to server" width="45%">

**SSH from attacker → server**
<img src="https://i.imgur.com/Bu5DlBx.png" alt="Baseline SSH from attacker to server" width="45%">

**pfSense packet capture (attacker → server)**
<img src="https://i.imgur.com/P3JaWdy.png" alt="pfSense packet capture of attacker-to-server traffic" width="45%">

**Ping from attacker → workstation**
<img src="https://i.imgur.com/amkWHpx.png" alt="Baseline ping from attacker to internal workstation" width="45%">

**Failed curl/ssh from attacker → workstation (expected after rules)**
<img src="https://i.imgur.com/borAgQu.png" alt="Blocked curl and SSH from attacker to workstation after rules applied" width="45%">

**Ping & curl within external network**
<img src="https://i.imgur.com/9Ku8xDv.png" alt="Ping and curl within the external attacker network" width="45%">

**Internal workstation → internal server traffic**
<img src="https://i.imgur.com/aPPdxJM.png" alt="Traffic from internal workstation to internal server" width="45%">

### Nmap Baseline Examples

<img src="https://i.imgur.com/pMhYda2.png" alt="Nmap baseline scan results, part 1" width="60%">
<img src="https://i.imgur.com/NrFsGvb.png" alt="Nmap baseline scan results, part 2" width="60%">

---

## Security Policy Implementation

Policy enforcement was implemented primarily in pfSense, with host-level controls (iptables) filling gaps the router could not express.

**Corporate policy summary**

- Server: HTTP and SSH allowed internally; HTTP allowed externally (read-only).
- Workstations: may initiate internal access but may not host external services.
- Server must not initiate outbound external connections.
- External hosts must not be able to ping internal hosts.

### Access Control Matrix

<img src="https://i.imgur.com/p4HTJUe.png" alt="Access-control matrix mapping sources, destinations, and allowed protocols" width="60%">

### pfSense Rules (Visual Snapshots)

**WAN rules**
<img src="https://i.imgur.com/DOAkzbD.png" alt="pfSense WAN firewall rules" width="60%">

**LAN rules**
<img src="https://i.imgur.com/TYuD7lH.png" alt="pfSense LAN firewall rules" width="60%">

**OPT1 rules**
<img src="https://i.imgur.com/7YP6lje.png" alt="pfSense OPT1 firewall rules" width="60%">

---

## Post-Implementation Verification

- Nmap confirmed that only authorized services were exposed after the rules were applied.
- Wireshark captures verified that blocked traffic never traversed the internal interfaces.
- System and firewall logs corroborated rule enforcement.

<img src="https://i.imgur.com/bZeB44P.png" alt="Post-rule verification evidence, part 1" width="60%">
<img src="https://i.imgur.com/nYRv0ry.png" alt="Post-rule verification evidence, part 2" width="60%">
<img src="https://i.imgur.com/DthG8qW.png" alt="Post-rule verification evidence, part 3" width="60%">

---

## Host-Level Firewall (Server A.1)

Applied host-level iptables rules on the internal server to enforce policy items the gateway could not:

```bash
sudo iptables -A OUTPUT -d 192.168.20.0/24 -j DROP                       # block outbound to the external network
sudo iptables -A INPUT  -p tcp --dport 22 -s 192.168.10.0/24 -j ACCEPT   # allow SSH only from internal
sudo iptables -A INPUT  -p tcp --dport 80 -j ACCEPT                      # allow HTTP
sudo iptables -A INPUT  -p icmp -s 192.168.10.0/24 -j ACCEPT             # allow ICMP only from internal
```

<img src="https://i.imgur.com/f1mLB6a.png" alt="iptables ruleset on the internal server, part 1" width="60%"> <img src="https://i.imgur.com/W7tmMXL.png" alt="iptables ruleset on the internal server, part 2" width="60%">

## Part 1 Outcomes

- Built a reproducible lab environment for network-policy testing and security validation.
- Implemented defense-in-depth by combining router-level and host-level controls.
- Validated enforcement and produced reproducible evidence (scans, captures, and logs) to support detection and remediation workflows.

---

# Part 2 — Slowloris Application-Layer DoS

## Part 2 Overview

A controlled study of an application-layer Denial-of-Service attack (Slowloris) against an Apache2 web server, run inside the isolated sandbox from Part 1. The exercise centers on four things: attack mechanics, SIEM ingestion and detection, packet- and host-level telemetry analysis, and operational mitigations.

## Skills Demonstrated

- Application-layer DoS emulation (Slowloris)
- VirtualBox sandbox and pfSense topology management
- SIEM ingestion and detection engineering (Splunk)
- Network discovery and reconnaissance (Nmap)
- Packet- and host-level forensics (Wireshark, tcpdump, Apache logs)
- Reproducible documentation and defensive recommendations

## Why This Matters

Slowloris-style attacks can take a web service offline while using almost no bandwidth, which makes them easy to miss with network-volume monitoring alone. Executing *and* detecting the attack demonstrates the ability to reason about attacker technique, validate the telemetry that would catch it, and turn that into detection and response.

---

## Lab Environment & Topology

- **Network A (internal)** — `192.168.1.0/24`
  - Ubuntu (web server / Apache2): `192.168.1.10`
  - Windows XP (workstation): `192.168.1.15`
- **Network B (attacker / external)** — `192.168.2.0/24`
  - Kali Linux (attacker): `192.168.2.16`
- **Router / firewall:** pfSense (LAN ↔ Network A, OPT1/WAN ↔ Network B)
- **Monitoring:** Splunk, ingesting `/var/log/apache2/access.log` and `/var/log/apache2/error.log`

<img src="assets/Topology.png" alt="Slowloris lab network topology" width="600">

---

## Splunk Configuration & Ingestion Validation

Configured Splunk to ingest Apache logs from the victim VM:

- `/var/log/apache2/access.log` — sourcetype `apache:access`
- `/var/log/apache2/error.log` — sourcetype `apache:error`

**Validation steps**

1. Confirmed both log files appeared in Splunk with the expected sourcetypes.
2. Built an initial search to surface `MaxRequestWorkers` and related Apache errors.
3. Correlated Splunk event timestamps against live availability tests.

---

## Pre-Attack Assessment

From the attacker VM (Kali):

- Ran quick port/service enumeration with Nmap.
  <img src="assets/nmap-before.png" alt="Nmap scan of the target before the attack" width="600">

- Verified the HTTP service with curl:

  ```bash
  curl http://192.168.1.10/testsplunk
  ```

  <img src="assets/curl-before.png" alt="Successful curl response before the attack" width="425">

---

## Pre-Attack Tuning (Kali)

Slowloris holds open a large number of sockets, so the attacker's file-descriptor limits were raised first:

```bash
ulimit -S -n 8192    # soft limit
ulimit -H -n 16384   # hard limit
```

## Executing the Slowloris Attack

```bash
slowloris -v -p 80 -s 1500 --sleeptime 10 192.168.1.10
```

| Flag | Meaning |
|---|---|
| `-v` | Verbose output |
| `-p 80` | Target HTTP port |
| `-s 1500` | Number of sockets (tuned to the VM's limits) |
| `--sleeptime 10` | Seconds between header-refresh attempts |

<img src="assets/slowloris-attack.png" alt="Slowloris running against the target, opening and holding sockets" width="600">

## Evidence & Attack Verification

### Apache Error Log Evidence (Splunk)

Splunk surfaced worker-exhaustion indicators throughout the attack window.

<img src="assets/splunk-maxrequestworkers.png" alt="Splunk showing Apache MaxRequestWorkers errors during the attack" width="600">

### Application Availability Impact

Application-layer failure was confirmed with a cross-terminal test:

- HTTP requests stalled during the attack.
- ICMP (ping) traffic kept flowing.

This proves **application-layer resource exhaustion** rather than a full network outage — the host was still reachable, but Apache could no longer serve requests.

<img src="assets/ping-after.png" alt="Ping succeeding while HTTP stalls during the attack" width="500">

---

## Technical Analysis — Why the Attack Worked

Slowloris wins by exhausting application resources, not bandwidth.

**Attack mechanics**

- Opens many concurrent HTTP connections.
- Sends HTTP headers extremely slowly.
- Keeps connections alive with periodic header refreshes.
- Never completes a request, so Apache never releases the worker.

**Server-side behavior**

- Apache assigns a worker thread per connection.
- Workers stay locked waiting for headers that never finish arriving.
- Permissive keep-alive and timeout settings let those connections persist.
- Eventually `MaxRequestWorkers` is reached and legitimate clients are refused.

**Environmental factors that amplified it**

- Worker limits were low relative to the attacker's socket volume.
- Keep-alive and timeout settings were permissive.
- The attacker's socket capacity was raised via `ulimit` tuning.

---

## Detection Engineering Insights

Detection was validated across multiple independent signals:

**SIEM correlation**

- Apache error logs (`MaxRequestWorkers`).
- Availability metrics.
- Time-synchronized attack activity.

**Packet-level indicators**

- Partial or incomplete HTTP headers.
- Long-lived TCP sessions with no completed request.

---

## Security Impact

- Showed that a low-bandwidth application-layer attack can cause full service degradation.
- Reinforced the need to monitor application telemetry, not just network volume.
- Demonstrated why layered detection matters — the attack is invisible to bandwidth-only monitoring.

---

## Mitigation Recommendations

**Application hardening**

- Tune Apache worker and timeout thresholds.
- Enforce per-IP rate limiting.
- Restrict persistent connections where possible.

**Architecture defenses**

- Front the server with a reverse proxy or load balancer.
- Add upstream connection throttling.

**Detection improvements**

- Create SIEM alerts for worker-exhaustion patterns.
- Monitor for abnormal connection-persistence behavior.

---

## Part 2 Outcomes

- Executed and documented Slowloris DoS testing in an isolated environment.
- Built detection-validation workflows using SIEM and host/network telemetry.
- Produced defensive guidance backed by reproducible test evidence.

---

## Ethical Notice

All testing was performed in an authorized, isolated lab environment for defensive research only. Do not reproduce any of this against systems you do not own or have explicit permission to test.
