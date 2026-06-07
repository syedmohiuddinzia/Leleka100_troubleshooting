<h1 style="text-align:center; color:#525EA1; font-weight:700;">Troubleshooting the FCC and UAV Connectivity for Leleka-100</h1>
<br>
A systematic investigation and resolution of an FCC connectivity failure within the Leleka-100 UAV ground control system was documented in this report. Despite all network devices having been confirmed reachable and live telemetry having been correctly displayed by the GCS, the FCC workstation repeatedly failed to connect to the UAV. Over three investigation phases, the problem was traced by the engineering team from basic network verification through deep process analysis, and an unexpected architectural detail was ultimately discovered: MAVLink telemetry had been delivered from the Microhard radio directly into the DFA application via a Unix pipe, never having touched the network. While this design had been successfully served for the GCS operator's display, the FCC was left with no data to receive. A non-invasive resolution was applied a MAVProxy forwarder was run as a systemd service on the GCS, bridging the internal telemetry stream to the FCC's expected UDP port. No changes were made to the DFA application, serverC backend, FCC configuration, Microhard radios, or any other system component. The complete investigation methodology, command-by-command analysis, network topology, root cause determination, and the final implemented solution are presented in this report.
<br><br><br><br><br><br><br><br><br><br>

---
---

| Field | Details |
|---|---|
| **System** | Leleka-100 UAV / FCC 4.0.11 / DFA Ground Control Software |
| **GCS Platform** | Kubuntu Linux Intel NUC8i7BEH (`pilot-NUC8i7BEH`) |
| **FCC Platform** | Windows 10 `DESKTOP-V35DUDH` |
| **Investigation Period** | May 2026 |
| **Report Status** | RESOLVED |
| **Authors** | Syed Mohiuddin Zia |

---
---

<br><br>

## Table of Contents

1. [Introduction & Problem Statement](#1-introduction--problem-statement)
2. [System Architecture & Network Topology](#2-system-architecture--network-topology)
3. [Phase 1 - Initial Connectivity Verification)](#3-phase-1---initial-connectivity-verification)
4. [Phase 2 - GCS Process & Connection Analysis](#4-phase-2---gcs-process--connection-analysis)
5. [Phase 3 - Deep Process Investigation](#5-phase-3---deep-process-investigation)
6. [Phase 4 - Configuration Analysis](#6-phase-4---configuration-analysis)
7. [Root Cause Summary](#7-root-cause-summary)
8. [Resolution & Fix Applied](#8-resolution--fix-applied)
9. [Persistence & systemd Service](#9-persistence--systemd-service)
10. [Full Command Reference](#10-full-command-reference)
11. [Configuration Parameters Reference](#11-configuration-parameters-reference)
12. [Conclusion & Recommendations](#12-conclusion--recommendations)
---
---
<br><br>

## 1. Introduction & Problem Statement
The Leleka-100 drone system has two operator computers sharing the same local network. One is a Linux machine running the ground control software that displays the drone's state and manages the radio and camera connections, and the other is a Windows machine running a separate flight control application that needs to receive live telemetry from the drone. The only way the drone communicates with the ground is through a pair of Microhard radios operating at 900 MHz, one mounted on the drone and one sitting on the ground, forming a single wireless bridge between the aircraft and the ground network. The critical architectural detail that caused all the subsequent problems is that the ground radio does not simply pass the drone's MAVLink data onto the network as a raw UDP stream that any computer could listen to, instead it wraps that data behind an HTTP API, meaning only software specifically written to query that API can actually retrieve the telemetry, and everything else on the network, including the FCC, is effectively blind to the drone's existence.

### 1.1 Presenting Problem
When the operator presses CONNECT on the FCC, no drone parameters load, no telemetry appears on screen, and the connection status shows failed every time. This happens despite the fact that the drone is clearly communicating with the ground — the GCS display is showing live data, the network is healthy, and the radio link is active. After ruling out every hardware explanation, the problem is a software routing failure on the ground where telemetry arrives at the GCS and stays there, delivered only internally to the DFA application with no mechanism in place to forward it across the network to the FCC.

### 1.2 Investigation Phases
The investigation moved through four phases before reaching a resolution.

The first phase confirmed that the network and all hardware were working correctly. Every device on the network responded to pings, the camera stream loaded on the FCC, and the Raspberry Pi was confirmed as having nothing to do with telemetry forwarding. The problem was not physical.

The second phase looked at what the GCS was actually doing with the telemetry it received. No connection to the FCC existed anywhere in the GCS connection table, DFA's port was bound to localhost only and invisible to the network, and packet captures showed that not a single MAVLink byte was present on any network interface of the GCS machine.

The third phase dug into how telemetry moved between the two GCS processes internally. It turned out that serverC was writing MAVLink data directly into DFA's standard input through a Unix pipe, meaning the data never touched the network at all. The process responsible for writing into that pipe had already terminated by the time the team looked for it, but examining the bash history revealed that MAVProxy had been tested on the machine before and the correct command was already known.

The fourth phase checked whether either system was simply misconfigured. The FCC configuration file was correct in every parameter, and the DFA settings file contained the expected UDP port configuration, though DFA was ignoring it based on how it was being launched. There was no misconfiguration on either side — the forwarding path had simply never been built.

The fix was straightforward once the source port was confirmed. MAVProxy was run interactively, immediately received heartbeats from the drone on UDP port 14551, and forwarded them to the FCC on UDP port 14550, at which point the FCC connected and loaded parameters successfully. A systemd service was then created to run MAVProxy permanently and restart it automatically if it ever stopped. Nothing else in the system was touched.
```mermaid
graph LR
    A([<b>🔍 Phase 1</b>\nConnectivity\nVerification]) --> B([🔬 Phase 2</b>\nGCS Process\nAnalysis])
    B --> C([<b>🔩 Phase 3</b>\nPipe & Architecture\nDiscovery])
    C --> D([<b>⚙️ Phase 4</b>\nConfiguration\nAnalysis])
    D --> E[\<b>✅ Resolution</b>\nMAVProxy\nService\]
```
---
---
<br><br>



## 2. System Architecture & Network Topology
Before examining the failure, it is necessary to understand how the system is built, the devices on the network, how the UAV communicates with the ground, and how the two software processes on the GCS are structured and connected to each other.

### 2.1 Network Device Table
The network contains ten devices all addressed within the same 10.The network contains ten devices all addressed within the same 10.10.10.0/24 subnet. Two of them are Microhard radios forming the wireless bridge between the drone and the ground, one mounted on the drone at 10.10.10.10 and one sitting on the ground at 10.10.10.1, both running OpenWrt. The drone also carries a Dahua Z30 camera at 10.10.10.2 and a Raspberry Pi at 10.10.10.3 handling encrypted logging. On the ground, the GCS Linux machine sits at 10.10.10.11, the FCC Windows machine at 10.10.10.110, a second Dahua camera mounted on the ground antenna at 10.10.10.22, an antenna tracker controller at 10.10.10.5 that physically steers the directional antenna toward the drone, an RTK GPS base station built on an ESP32 microcontroller at 10.10.10.111 providing precision positioning corrections, and a MikroTik router at 10.10.10.253 acting as the network gateway tying everything together.
| IP Address | Device | OS / Platform | Role |
|---|---|---|---|
| `10.10.10.1` | **Microhard (ground radio)** | OpenWrt / BusyBox | Ground side 900 MHz radio modem |
| `10.10.10.2` | **Dahua Z30 (UAV camera)** | Embedded Linux | UAV mounted thermal/optical camera |
| `10.10.10.3` | **Raspberry Pi** | Raspbian Linux | Onboard companion, crypto/log manager |
| `10.10.10.5` | **Antenna Tracker** | Embedded | Ground antenna tracker controller |
| `10.10.10.10` | **Microhard (UAV radio)** | OpenWrt / BusyBox | UAV side 900 MHz radio modem |
| `10.10.10.11` | **GCS `pilot-NUC8i7BEH`** | Kubuntu Linux | Ground Control Station, DFA + serverC |
| `10.10.10.22` | **Dahua (ground camera)** | Embedded Linux | Ground antenna camera |
| `10.10.10.110` | **FCC `DESKTOP-V35DUDH`** | Windows 10 | Flight Control Computer, FCC 4.0.11 |
| `10.10.10.111` | **RTK Base** | Espressif ESP32 | GNSS RTK base station |
| `10.10.10.253` | **MikroTik Routerboard** | RouterOS | Network gateway / LAN switch |

### 2.2 Network Topology Diagram
The flight controller on the drone sends MAVLink telemetry over a serial connection at 57,600 baud into the UAV-side Microhard radio, which transmits it wirelessly at 900 MHz down to the ground-side Microhard radio. The ground radio does not expose this telemetry as a raw network stream, instead serverC on the GCS queries it through an HTTP API on port 80 to retrieve the data. ServerC then passes that data directly into DFA through a Unix pipe, where it is consumed internally and displayed to the GCS operator. DFA separately maintains a TCP connection to the antenna tracker on port 14661 to steer the ground antenna, and serverC maintains a TCP connection to the UAV camera on port 554 to pull the video stream. The FCC sits on the same network waiting for MAVLink data on UDP port 14550 but receives nothing, because the entire telemetry chain from radio to serverC to pipe to DFA never produces any outbound network traffic in the direction of the FCC.
```mermaid
graph TD
    subgraph UAV["<b style='font-size:16px'>✈ UAV (In Flight)</b>"]
        FC[<b>ArduPilot</b>\nFlight Controller]
        PI[<b>Raspberry Pi</b>\n10.10.10.3\nCrypto/Log]
        CAM_UAV[<b>Dahua Z30</b>\n10.10.10.2\nUAV Camera]
        MH_UAV[<b>Microhard Radio</b>\n10.10.10.10\nUAV Side]
        FC -->|serial 57600 baud| MH_UAV
        linkStyle 0 stroke:#008000,stroke-width:2px,color:green
    end
    subgraph GROUND["<b style='font-size:16px'>🏗 Ground</b>"]
        MH_GND[<b>Microhard Radio</b>\n10.10.10.1\nGround Side]
        RTK[<b>RTK Base</b>\n10.10.10.111\nport 222]
        ANT[<b>Antenna Tracker</b>\n10.10.10.5\nport 14661]
        CAM_GND[<b>Ground Camera</b>\n10.10.10.22\nport 554]
        RT[<b>MikroTik Router</b>\n10.10.10.253]
        subgraph GCS["<b>💻 GCS</b> 10.10.10.11 (Linux)"]
            SERV[<b>serverC/server</b>\nPID 20734]
            DFA[<b>DFA Application</b>\nPID 20538]
            PIPE[<b>Unix Pipe</b>\n924958\nstdin of DFA]
            SERV -->|pipe fd39→fd0| PIPE
            PIPE --> DFA
        end
        subgraph FCC["<b>🖥 FCC</b> 10.10.10.110 (Windows)"]
            FCC_APP[<b>FCC 4.0.11</b>\nUDP:14550]
        end
        MH_GND -->|HTTP API port 80| SERV
        SERV -->|TCP:554| CAM_UAV
        DFA -->|TCP:14661| ANT
    end

    MH_UAV <-->|900 MHz RF Link| MH_GND
    FCC_APP -.->|❌ NO DATA| SERV
    linkStyle 0 stroke:#008000,stroke-width:2px
    linkStyle 1 stroke:#008000,stroke-width:2px
    linkStyle 2 stroke:#008000,stroke-width:2px
    linkStyle 3 stroke:#008000,stroke-width:2px
    linkStyle 4 stroke:#008000,stroke-width:2px
    linkStyle 5 stroke:#008000,stroke-width:2px
    linkStyle 6 stroke:#008000,stroke-width:2px
    linkStyle 7 stroke:#ff0000,stroke-width:2px,stroke-dasharray:3
```

### 2.3 GCS Software Stack

The GCS runs two processes side by side. **ServerC** is responsible for all external data collection, it sends HTTP requests to the Microhard radio at 10.10.10.10 on port 80 to retrieve telemetry, pulls the UAV camera video stream over RTSP from 10.10.10.2 on port 554, and relays that video internally through a localhost connection on port 8554. **DFA** is responsible for everything the operator interacts with, it receives the telemetry from serverC through a Unix pipe directly into its standard input, receives the video from serverC through the localhost TCP connection on port 8554, serves its own internal user interface on localhost port 8085 which is accessible only on the local machine, and maintains a connection to the antenna tracker at 10.10.10.5 on port 14661 to control the ground antenna direction.<br>
The two processes are tightly coupled, with serverC acting as the data collection layer feeding everything into DFA, which acts as the display and control layer.
```mermaid
graph LR
    subgraph GCS_PROCS["<b style='font-size:16px'>GCS Process Architecture"]
        A[<b>serverC/server</b>\nPID 20734\n./serverC/server] 
        B[<b>DFA</b>\nPID 20538\n/home/pilot/DFA/DFA]
        A -->|pipe stdin\ninode 924958| B
        A -->|TCP ESTABLISHED\nlocalhost:8554| B
        B -->|TCP LISTEN\nlocalhost:8085| B
    end

    subgraph EXTERNAL["<b style='font-size:16px'>External Connections"]
        MH[<b>Microhard</b>\n10.10.10.10:80]
        CAM[<b>UAV Camera</b>\n10.10.10.2:554]
        ANT[<b>Antenna Tracker</b>\n10.10.10.5:14661]
        VID[<b>Video relay</b>\nlocalhost:8554]
    end
    A -->|HTTP GET| MH
    A -->|RTSP ESTABLISHED| CAM
    B -->|TCP ESTABLISHED| ANT
    A --> VID
    linkStyle 0 stroke:#008000,stroke-width:2px
    linkStyle 1 stroke:#008000,stroke-width:2px
    linkStyle 2 stroke:#008000,stroke-width:2px
    linkStyle 3 stroke:#008000,stroke-width:2px
    linkStyle 4 stroke:#008000,stroke-width:2px
    linkStyle 5 stroke:#008000,stroke-width:2px
    linkStyle 6 stroke:#008000,stroke-width:2px

```

| Process | Binary | PID | Primary Function |
|---|---|---|---|
| **DFA** | `/home/pilot/DFA/DFA` | 20538 | Main ground control application `display, antenna tracker, video relay` |
| **server** | `/home/pilot/DFA/serverC/server` | 20734 | Backend `Microhard HTTP API, RTSP camera, MAVLink pipe writer` |

> **Note:** DFA is a PyInstaller-bundled Python application. Runtime unpacks to `/tmp/_MEIhb2hL6/`. Uses `MAVLINK_DIALECT=ardupilotmega`.

---
## 3. Phase 1 - Initial Connectivity Verification
Phase 1 established the health of the network and hardware before any deeper investigation began. Every device on the subnet was tested for reachability from both the GCS and the FCC, services were scanned on each machine, the Raspberry Pi was specifically eliminated as a MAVLink relay, the camera stream was verified over the radio link, and Pi network traffic was captured and analyzed. By the end of this phase every hardware and network explanation for the FCC connection failure had been ruled out, narrowing the problem definitively to the software layer on the GCS.

### 3.1 Ping Tests
Every device on the network was pinged from both the GCS and the FCC to confirm that basic communication between all machines was working. The GCS sent four packets to each device and the FCC did the same from its side, with each machine naturally skipping itself. Every single device responded successfully with zero packet loss in both directions, including the UAV-side Microhard radio at 10.10.10.10 which confirmed that the wireless radio link was transparent at the network level and the drone's network presence was fully reachable from the ground.<br>
`ping` is the most fundamental network reachability tool, sending ICMP echo request packets to a target address and measuring whether a reply comes back. On Linux the `-c 4` flag limits the test to four packets per target rather than pinging indefinitely, giving a quick but statistically meaningful result. On Windows the default behavior is already four packets so no flag is needed. It was run from both machines independently and against every device on the network because confirming reachability from both directions was important, a device might respond to the GCS but not the FCC due to a routing asymmetry or a host-based firewall, so testing from each machine separately gave a complete and trustworthy picture. Pinging the UAV-side Microhard at 10.10.10.10 was particularly significant because a successful reply from that address confirmed the 900 MHz wireless link was up and the drone's network presence was fully accessible from the ground.

**GCS (Linux):**
```bash
ping -c 4 10.10.10.1    # Ground Microhard
ping -c 4 10.10.10.2    # UAV camera
ping -c 4 10.10.10.3    # Raspberry Pi
ping -c 4 10.10.10.5    # Antenna tracker
ping -c 4 10.10.10.10   # UAV Microhard
ping -c 4 10.10.10.22   # Ground camera
ping -c 4 10.10.10.110  # FCC
ping -c 4 10.10.10.111  # RTK base
ping -c 4 10.10.10.253  # Router
```

**FCC (Windows CMD):**
```cmd
ping 10.10.10.1    # Ground Microhard
ping 10.10.10.2    # UAV camera
ping 10.10.10.3    # Raspberry Pi
ping 10.10.10.5    # Antenna tracker
ping 10.10.10.10   # UAV Microhard
ping 10.10.10.11   # GCS
ping 10.10.10.22   # Ground camera
ping 10.10.10.111  # RTK base
ping 10.10.10.253  # Router
```

This ruled out any possibility that the FCC connectivity failure was caused by a broken network link, a misconfigured IP address, a faulty cable, or any other layer 2 or layer 3 problem between the machines.

**Ping Results Matrix:**

| Target IP | From GCS (.11) | From FCC (.110) | Notes |
|---|---|---|---|
| 10.10.10.1 | ✅ OK | ✅ OK | Ground Microhard |
| 10.10.10.2 | ✅ OK | ✅ OK | UAV camera |
| 10.10.10.3 | ✅ OK | ✅ OK | Raspberry Pi |
| 10.10.10.5 | ✅ OK | ✅ OK | Antenna tracker |
| 10.10.10.10 | ✅ OK | ✅ OK | UAV Microhard |
| 10.10.10.11 | `N/A` | ✅ OK | GCS |
| 10.10.10.110 | ✅ OK | `N/A` | FCC |
| 10.10.10.111 | ✅ OK | ✅ OK | RTK base |
| 10.10.10.253 | ✅ OK | ✅ OK | Router |

0% packet loss for all devices. **Network layer confirmed fully operational.**

### 3.2 Network Discovery Scan
A network discovery scan was run from both machines using nmap's ping scan mode, which sends ARP and ICMP probes to every address in the subnet without doing any port scanning. Both the GCS and the FCC independently confirmed that all ten devices were online and responding, which validated the ping test results and gave a complete picture of every active node on the network before deeper scanning began. Nmap is a network scanning tool used to discover hosts and services on a network.<br>
Nmap with the -sn flag performs a ping scan across the entire subnet specified in CIDR notation, meaning it sends ARP and ICMP probes to every possible address from 10.10.10.1 through 10.10.10.254 without doing any port scanning at all. The sole purpose is to discover which addresses have a live device behind them and which are empty, making it faster and less intrusive than a full service scan. It was used at this stage simply to get a complete inventory of every active device on the network before deeper scanning began, confirming that all ten expected devices were present and responding.
```bash
# Linux — GCS
nmap -sn 10.10.10.0/24
```
```cmd
REM Windows — FCC
nmap -sn 10.10.10.0/24
```
### 3.3 Service Version Scans
Service version scans were run against every device to identify exactly what was listening on each machine. `nmap` with the `-sV` flag performs a service version scan, meaning it does not simply check whether a port is open but actively probes each open port to identify the specific software and version running behind it. It was run individually against each device on the network rather than as a subnet-wide scan so that the results for each device were clean and separate, making it easier to analyze what each machine was exposing. Running these scans from the FCC rather than the GCS was deliberate — it gave a real-world view of what services were actually reachable from the FCC's perspective on the network, which was directly relevant to understanding why the FCC could not connect to anything MAVLink-related on the GCS.
```cmd
REM Run for each device from FCC (Windows)
nmap -sV 10.10.10.1
nmap -sV 10.10.10.2
nmap -sV 10.10.10.3
nmap -sV 10.10.10.10
nmap -sV 10.10.10.11
nmap -sV 10.10.10.22
nmap -sV 10.10.10.110
nmap -sV 10.10.10.111
```
Both Microhard radios presented identical profiles with FTP, SSH, telnet, DNS, and an HTTP web interface, confirming they run the same OpenWrt firmware and that the HTTP API used by serverC was present and accessible on both. The Dahua UAV camera exposed its RTSP stream on port 554 alongside HTTP, RTMP, and UPnP management ports. The Raspberry Pi only had two SSH ports open and nothing else, which immediately ruled it out as any kind of MAVLink relay or telemetry service. The two Dahua cameras showed similar streaming service profiles. The FCC Windows machine showed only standard Windows networking ports. The RTK base exposed its raw GPS data stream on port 222. The antenna tracker returned no TCP results at all, indicating it communicates purely over UDP using its own protocol. Most significantly, the GCS at 10.10.10.11 returned no open TCP ports whatsoever, meaning it was exposing nothing to the network, no MAVLink service, no telemetry endpoint, nothing the FCC could connect to even if it tried.</br>

**Service Scan Results:**
| IP | Open TCP Ports | Identified Services |
|---|---|---|
| `10.10.10.1` | 21, 22, 23, 53, 80, 443 | Microhard OpenWrt `FTP, Dropbear SSH, BusyBox telnet, dnsmasq, uHTTPd` |
| `10.10.10.2` | 80, 554, 1935, 5000 | Dahua Z30 `HTTP, RTSP/3.0, RTMP, UPnP` |
| `10.10.10.3` | 22, 444 | Raspberry Pi `OpenSSH 7.2p2 on both ports` |
| `10.10.10.10` | 21, 22, 23, 53, 80, 443 | Microhard OpenWrt `identical profile to .10.1` |
| `10.10.10.11` | **None** | GCS `no TCP services exposed to network` |
| `10.10.10.22` | 80, 554, 1935 | Dahua ground camera |
| `10.10.10.110` | 135, 139, 445, 5357 | FCC Windows 10 `RPC, NetBIOS, SMB, HTTPAPI` |
| `10.10.10.111` | 80, 222 | RTK Base `HTTP, raw UBX/NMEA GPS stream` |

> **Notable:** `10.10.10.5` (Antenna Tracker) returned no TCP results, communicates via UDP only with a proprietary protocol.

### 3.4 MAVLink UDP Port Scan, Raspberry Pi
The Raspberry Pi was specifically scanned for every UDP port commonly associated with MAVLink telemetry.  The -sU flag tells nmap to perform a UDP scan rather than the default TCP scan, which is necessary here because MAVLink telemetry travels over UDP and a TCP scan would miss it entirely. The -p flag specifies exactly which ports to check rather than scanning all 65,535, and the four ports listed cover the most common MAVLink configurations 14550 and 14551 are the standard MAVLink ground station and vehicle ports, 5760 is the default MAVProxy TCP bridge port, and 700 is an alternative port sometimes used in custom setups. The target address 10.10.10.3 is the Raspberry Pi, which was being checked specifically because it was the only device on the UAV that could plausibly be running a MAVLink relay service. Targeting these specific ports rather than doing a broad scan made the test fast and precise, and the result immediately confirmed whether the Pi was involved in telemetry forwarding or not.
```cmd
nmap -sU -p 14550,14551,5760,700 10.10.10.3
```
All four ports came back closed, leaving no ambiguity, the Pi is not running any MAVLink service, is not bridging or relaying telemetry in any form, and plays no role in the data path between the drone and the ground stations. Its only function on the aircraft is managing encrypted logs and cryptographic keys.
Result:
```
PORT       STATE   SERVICE
700/udp    closed  epp
5760/udp   closed  unknown
14550/udp  closed  unknown
14551/udp  closed  unknown
```
### 3.5 Camera Stream Verification
The UAV camera stream was opened directly from the FCC in VLC using its RTSP address, and it loaded successfully. This confirmed Microhard radio link was capable of carrying continuous data from the drone to the ground, meaning the wireless link was not the source of any connectivity problem.
```
rtsp://admin:admin1@10.10.10.2:554/cam/realmonitor?channel=1&subtype=0
```
### 3.6 Wireshark, Raspberry Pi Traffic Analysis
Wireshark was run on the FCC with a filter isolating all traffic to and from the Raspberry Pi. 
```
host 10.10.10.3
```
The only packets that appeared were mDNS broadcasts the Pi was sending out every few minutes as part of a routine network printer discovery process. There was no MAVLink, no telemetry, no control data of any kind — confirming beyond any doubt that the Pi is completely passive on the network and has no involvement in the UAV data flow.
```
10.10.10.3 → 224.0.0.251   mDNS   PTR _ipp._tcp.local  (every ~256 seconds)
```

### 3.7 Decision Tree
Phase 1 followed a systematic elimination process. The first question was whether all devices were reachable: yes they were, so hardware and network failure were ruled out immediately. The second question was whether the Raspberry Pi had any MAVLink ports open: no it did not, so the Pi was eliminated as a telemetry bridge. The third question was whether the camera stream worked from the FCC: yes it did, which confirmed the radio link was carrying data correctly and ruled out any RF or transmission failure. The fourth and final question was whether the GCS had any active connection toward the FCC: no it had none whatsoever. With every hardware and network explanation exhausted, the investigation concluded Phase 1 with a single remaining finding: the GCS was receiving telemetry but not forwarding any of it to the FCC.
```mermaid
flowchart TB
    A[Start Investigation] --> B{All devices\npingable?}
    B -->|Yes| C{MAVLink ports\nopen on Pi?}
    B -->|No| X1[Hardware/network\nfailure, not this case]
    C -->|No, all closed| D{Camera stream\nworking in VLC?}
    C -->|Yes| X2[Pi is MAVLink bridge,\n investigate Pi config]
    D -->|Yes| E{GCS has active\nconnection to FCC?}
    D -->|No| X3[Radio/camera failure,\n not this case]
    E -->|No, nothing| F[🔴 Problem: GCS does\nnot forward to FCC]
    linkStyle 0 stroke:#008000,stroke-width:2px
    linkStyle 1 stroke:#008000,stroke-width:2px
    linkStyle 2 stroke:#ff0000,stroke-width:2px
    linkStyle 3 stroke:#ff0000,stroke-width:2px
    linkStyle 4 stroke:#008000,stroke-width:2px
    linkStyle 5 stroke:#008000,stroke-width:2px
    linkStyle 6 stroke:#ff0000,stroke-width:2px
    linkStyle 7 stroke:#ff0000,stroke-width:2px

```

---

## 4. Phase 2 - GCS Process & Connection Analysis
With the network and hardware confirmed healthy, now the investigation is turned inward to the GCS machine itself, examining every active connection, every listening port, every process file descriptor, and every byte of traffic flowing to and from the GCS to understand exactly what the machine was doing with the telemetry it was receiving and why none of it was reaching the FCC.

### 4.1 Active TCP Connections on GCS
`netstat` is a network statistics utility that displays active network connections, routing tables, and interface statistics. The flags used here are `-t` which filters the output to TCP connections only, and `-n` which shows raw IP addresses and port numbers rather than attempting to resolve them to hostnames and service names, keeping the output fast and unambiguous. The `grep ESTABLISHED` at the end pipes the output through a filter that discards everything except connections that are currently active and exchanging data, removing listening sockets and connections in other states that were not relevant at this stage. It was used here specifically to get a precise picture of every live TCP session the GCS had open at that moment, which allowed the investigation to map out exactly what the GCS was communicating with and critically confirm that the FCC was not among those destinations.

```bash
netstat -tn | grep ESTABLISHED
```
```
tcp     0      0       127.0.0.1:8085      127.0.0.1:47744      ESTABLISHED
tcp     0      0       10.10.10.11:33626   10.10.10.2:554       ESTABLISHED
tcp     0      0       10.10.10.11:41964   10.10.10.5:14661     ESTABLISHED
tcp     0      0       127.0.0.1:8085      127.0.0.1:54698      ESTABLISHED
tcp     0      4575    127.0.0.1:8554      127.0.0.1:57878      ESTABLISHED
tcp     3171   0       127.0.0.1:57878     127.0.0.1:8554       ESTABLISHED
tcp     0      0       127.0.0.1:54698     127.0.0.1:8085       ESTABLISHED
```
Running netstat to list all active TCP connections on the GCS produced seven established connections, and reading through them told the complete story of what the GCS was actually doing. 
Two connections went out to the network, one from serverC to the UAV camera at 10.10.10.2 on port 554 pulling the video stream, and one from DFA to the antenna tracker at 10.10.10.5 on port 14661 controlling the ground antenna. The remaining five connections were all between localhost addresses, meaning they existed entirely within the GCS machine itself. DFA's internal interface on port 8085 talking to itself, and the video relay passing data back and forth between serverC and DFA on port 8554. The FCC's address 10.10.10.110 did not appear anywhere in the entire connection table, not as a source, not as a destination, not in any form. This was the definitive confirmation that the GCS had no awareness of the FCC whatsoever and was making no attempt to send it anything.
| Connection | Meaning |
|---|---|
| `10.10.10.11:33626 → 10.10.10.2:554` | serverC pulling RTSP from UAV camera |
| `10.10.10.11:41964 → 10.10.10.5:14661` | DFA connected to antenna tracker |
| `127.0.0.1:8085 ↔ 127.0.0.1:*` | DFA internal MAVLink server (localhost only) |
| `127.0.0.1:8554 ↔ 127.0.0.1:57878` | Internal video relay DFA ↔ serverC |
| **10.10.10.110, ABSENT** | ❌ No connection to FCC anywhere |

### 4.2 Listening Services on GCS
ss is the modern Linux utility for inspecting socket statistics, replacing the older netstat command. The flags used here each serve a specific purpose — -t limits the output to TCP sockets only, -l shows only sockets that are currently in the listening state waiting for incoming connections, -n displays raw IP addresses and port numbers instead of resolving them to hostnames and service names which would slow the output and introduce ambiguity, and -p attaches the name and process ID of the program that owns each socket. Together these flags produce a clean, precise list of exactly which processes on the GCS are waiting for connections and on which addresses and ports they are doing so. It was used at this stage of the investigation specifically to answer whether any process on the GCS was listening on a MAVLink-related port in a way that the FCC could reach, and to confirm whether DFA's internal port was genuinely restricted to localhost or simply firewalled.
```bash
sudo ss -tlnp
```
Listing all listening services on the GCS revealed three relevant entries. DFA was listening on port 8085 but bound strictly to 127.0.0.1, meaning it was physically impossible for any other machine on the network to reach it regardless of firewall rules or any other configuration, it simply does not exist from the network's perspective. The serverC video relay was listening on port 8554 bound to all interfaces, so it was technically reachable from the network, but it carries only video data and is not a MAVLink endpoint. TeamViewer was also running on the machine on port 5939, which explained how remote access to the GCS was being maintained during the investigation but was otherwise irrelevant to the problem. The key takeaway from this output was that DFA's deliberate binding to localhost was not an accident or a firewall rule — it was a hard architectural decision that made the telemetry data structurally unreachable from any external machine, including the FCC.
```
LISTEN  0  128  127.0.0.1:8085  0.0.0.0:*  users:(("DFA",pid=20538,fd=15))
LISTEN  0    5  0.0.0.0:8554    0.0.0.0:*  users:(("server",pid=20734,fd=4))
LISTEN  0  128  127.0.0.1:5939  0.0.0.0:*  users:(("teamviewerd",pid=1177,fd=11))
```

**Critical finding:**

- **DFA port 8085**: bound to `127.0.0.1` ONLY, **completely inaccessible from the network**
- **server port 8554**: bound to `0.0.0.0`, network accessible but video only
- **TeamViewer**: running on GCS port 5939

### 4.3 Process-to-Connection Mapping
Netstat with -t for TCP, -n for numeric addresses, and -p to show the owning process name and ID was piped through grep -iE which performs a case-insensitive search for any line containing either "dfa" or "server", filtering the output down to only the connections belonging to the two GCS processes. sudo was necessary because without root privileges the -p flag cannot read process ownership information for processes belonging to other users.
```bash
sudo netstat -tnp | grep -iE "dfa|server|DFA"
```
The output shows six connections split between the two processes. DFA at PID 20538 owns three of them: its internal interface on localhost port 8085 talking to itself, and its connection out to the antenna tracker at 10.10.10.5 on port 14661. ServerC at PID 20734 owns the remaining two: its connection out to the UAV camera at 10.10.10.2 on port 554 pulling the video stream, and its internal video relay connection to DFA on localhost port 8554, where the non-zero byte counts in the send and receive buffers indicate data was actively flowing between them at the time the command ran. As with the earlier netstat output, the FCC address 10.10.10.110 appears nowhere, and this time the output additionally confirms exactly which process owns each connection, leaving no ambiguity about the role each process plays in the system.
```
tcp  0      0       127.0.0.1:8085      127.0.0.1:50466  ESTABLISHED  20538/DFA
tcp  0      0       10.10.10.11:33626   10.10.10.2:554   ESTABLISHED  20734/./serverC/ser
tcp  0      0       10.10.10.11:41964   10.10.10.5:14661 ESTABLISHED  20538/DFA
tcp  0      0       127.0.0.1:8085      127.0.0.1:54698  ESTABLISHED  20538/DFA
tcp  0      6404    127.0.0.1:8554      127.0.0.1:57878  ESTABLISHED  20734/./serverC/ser
tcp  5000   0       127.0.0.1:57878     127.0.0.1:8554   ESTABLISHED  20538/DFA
```

### 4.4 DFA File Descriptors (lsof)
lsof lists every open file descriptor belonging to a process and in Linux everything is a file, including network sockets and pipes. The -p flag targets a specific process ID, and $(pgrep DFA) automatically finds DFA's PID rather than requiring it to be typed manually. The grep at the end filters the output to show only lines related to network sockets and pipes, discarding irrelevant file handles like shared libraries and log files.
```bash
sudo lsof -p $(pgrep DFA) | grep -E 'UDP|TCP|IPv|pipe'
```
The output mapped every network-related resource DFA had open. File descriptor zero, which is always standard input in Unix, was connected to a Unix pipe with inode 924958, meaning DFA was reading its MAVLink data directly from its own stdin rather than from any network socket. Two TCP connections were present and active, one going out to the antenna tracker and one going to the internal video relay on localhost. One TCP socket was in the listening state serving DFA's internal user interface on localhost port 8085. The most important finding was what was absent, there was no UDP socket of any kind in the entire list, and specifically no socket directed toward the FCC at 10.10.10.110 on port 14550. This was conclusive proof that DFA was architecturally incapable of forwarding telemetry to the FCC, not because of a configuration mistake but because no such socket had ever been opened.<br>
**DFA Socket Map:**
| FD | Protocol | Address | Peer | Status |
|---|---|---|---|---|
| 0 | pipe | `pipe:[924958]` | — | stdin `READ from pipe` |
| fd_x | TCP | `10.10.10.11:43720` | `10.10.10.5:14661` | ESTABLISHED `antenna tracker` |
| fd_y | TCP | `10.10.10.11:46132` | `127.0.0.1:8554` | ESTABLISHED `video stream` |
| fd_z | TCP (LISTEN) | `127.0.0.1:8085` | — | LISTEN `internal UI` |
| **MISSING** | UDP | — | `10.10.10.110:14550` | ❌ NOT PRESENT |

### 4.5 Traffic to Microhard UAV Radio
`tcpdump` is a packet capture tool that intercepts and displays raw network traffic in real time. The `-i any` flag tells it to listen on all network interfaces simultaneously rather than a single one, `-n` suppresses hostname and service name resolution to keep the output clean and fast, and `host 10.10.10.1 or host 10.10.10.10` is a capture filter that limits the output to only packets involving either of the two Microhard radios, discarding everything else on the network.

```bash
sudo tcpdump -i any -n host 10.10.10.1 or host 10.10.10.10
```
The captured output revealed something unexpected. Instead of seeing MAVLink UDP packets passing between the GCS and the Microhard radio, the only traffic was HTTP. serverC was making a GET request to the UAV-side Microhard on port 80 and receiving an HTTP response back, followed by an ARP probe toward the ground-side radio. This confirmed that the entire telemetry retrieval mechanism was HTTP-based rather than a raw MAVLink stream, meaning serverC was polling the Microhard's web API to collect telemetry data and there was no MAVLink UDP traffic on the network at any point in the chain. This explained why no other machine on the network, including the FCC, could independently access the telemetry, because it was locked behind an HTTP API that only serverC knew how to query.
Captured output:
```
10.10.10.11.56394 > 10.10.10.10.80: HTTP GET /favicon.ico
10.10.10.10.80 > 10.10.10.11.56394: HTTP/1.1 200 OK
[HTTP data exchange, then connection closed]
ARP: who-has 10.10.10.1 tell 10.10.10.11
```

### 4.6 MAVLink Port Confirmation, Zero Traffic
These four commands below approached the same question from different angles to build an irrefutable confirmation. The first two tcpdump captures listened on all interfaces for any MAVLink traffic on the standard UDP port 14550 and the alternative TCP port 5760, and both returned zero packets meaning no MAVLink data was flowing on the network in either direction on either protocol. The netstat filter for the Microhard UAV radio address returned nothing, confirming the GCS had no active TCP sessions with that device beyond what had already been identified as HTTP. The ss UDP listing showed no process on the GCS was listening on any MAVLink-related UDP port at all, meaning there was no service waiting to receive MAVLink even if something had been sending it.
```bash
sudo tcpdump -i any udp port 14550 -n
# 0 packets captured

sudo tcpdump -i any tcp port 5760 -n
# 0 packets captured

netstat -tn | grep 10.10.10.10
# no output

sudo ss -ulnp
# no process listening on UDP 14550 or 14551
```
 Together these four checks eliminated every remaining possibility: MAVLink was not flowing outbound, not flowing inbound, not being received, and not being transmitted. The telemetry existed only inside the GCS processes and had no presence on the network in any form.

### 4.7 Antenna Tracker Protocol Analysis
Capturing traffic on port 14661 with the `-XX` flag to display the raw hex and ASCII content of every packet revealed that the communication between DFA and the antenna tracker was extremely simple and completely unlike MAVLink. DFA was sending a single byte to the tracker every second as a heartbeat, and the tracker was responding with eight bytes containing the values 64 64 in hexadecimal, which is decimal 100 100, likely representing status or position feedback in a custom binary format. MAVLink packets have a recognizable structure starting with a specific magic byte and containing headers, system IDs, message IDs, and checksums, none of which were present here. This ruled out any possibility that the antenna tracker connection was carrying or could carry telemetry data, confirming it was purely a directional control and status channel between DFA and the physical antenna hardware.
```bash
sudo tcpdump -i any -n port 14661 -XX
```
---

## 5. Phase 3 - Deep Process Investigation
It was confirmed in previous step that the GCS was not forwarding anything to the FCC, but had not yet explained the exact mechanism by which telemetry was moving internally between the two GCS processes. Now we went deeper into the file descriptors, the binary structure of DFA, and the system's own history to understand precisely how MAVLink data was being delivered internally, why the pipe writer process could not be identified, and what information already existed on the machine that pointed toward the correct fix.

### 5.1 The Pipe Architecture Discovery
The investigation at this point uncovered the fundamental architectural reality of how the GCS software actually worked. ServerC queried the Microhard radio's HTTP API at 10.10.10.10 on port 80 to retrieve telemetry data, then instead of putting that data onto the network it wrote it directly into a Unix pipe. A Unix pipe is an in-memory channel that connects two processes running on the same machine. ServerC held the write end of this pipe on its file descriptor 39, and DFA received everything through the read end on its file descriptor zero, which is standard input. This means MAVLink data flowed from the radio into serverC and then straight into DFA's stdin as if it were being typed at a keyboard, completely bypassing the network stack entirely. DFA then consumed this data internally and displayed it on localhost port 8085. 
```mermaid
graph TB
    subgraph PIPE_ARCH["<b style='font-size:16px'>MAVLink Delivery Chain (Actual)</b>"]
        A[<b>Microhard UAV</b>\n10.10.10.10:80\nHTTP API] -->|HTTP GET telemetry| B[<b>serverC/server</b>\nPID 20734\nfd=39 WRITE]
        B -->|Unix Pipe\ninode 924958| C[<b>DFA stdin</b>\nfd=0 READ]
        C --> D[<b>DFA Display</b>\nlocalhost:8085\ninternal only]
    end
```
At no point in this entire chain did a single MAVLink byte appear on any network interface. The second diagram shows what was simply absent, there was no forwarder, no MAVProxy instance, no relay process of any kind whose job was to take that same data and send it across the network to the FCC at 10.10.10.110 on port 14550. The path to the FCC did not exist.
```mermaid
graph 
    subgraph MISSING["<b style='font-size:16px'>❌ Missing Path</b>"]
        E[<b>MAVProxy / forwarder</b>\nNot running] -.->|Nothing| F[<b>FCC</b>\n10.10.10.110:14550]
    end

```

### 5.2 The MAvlink Pipe Lifecycle
When the system was first started at around 17:15, a process was spawned that created the Unix pipe with inode 924958, serverC began querying the Microhard HTTP API, DFA was launched and bound its stdin to the read end of that pipe, and telemetry became visible on the GCS display,and the entire chain came up in sequence and worked correctly. At 18:02 DFA was restarted, but critically the pipe writer process survived that restart independently, and the new DFA instance reattached to the same pipe and continued receiving telemetry without interruption, demonstrating that the pipe writer and DFA were separate processes with independent lifecycles. By the time the investigation team looked for the pipe writer in the second session, it was gone and lsof returned nothing, the proc filesystem had no trace of it, and the pipe inode 924958 no longer existed anywhere on the system. The process had terminated at some point, likely when the terminal session or script that originally launched it was closed, and because it was never identified before it died, the exact mechanism that created it and fed MAVLink data into the pipe remained undocumented.
```mermaid
gantt
    dateFormat  HH:mm
    axisFormat  %H:%M
    
    section 17:15 Init
    Pipe writer spawned        :done, 17:15, 0.2m
    pipe inode 924958          :done, 17:16, 0.2m
    serverC queries Microhard  :done, 17:17, 0.2m
    DFA started                :done, 17:18, 0.2m
    Telemetry visible          :done, 17:19, 0.2m
    
    section 18:02 Reset
    DFA restarted              :active, 18:02, 0.2m
    Pipe writer survives       :active, 18:03, 0.2m
    New DFA binds pipe         :active, 18:04, 0.2m
    
    section Failure
    Pipe writer DEAD           :crit, 19:00, 0.2m
    lsof returns nothing       :crit, 19:01, 0.2m
    Pipe inode gone            :crit, 19:02, 0.2m
```

### 5.3 Attempting to Identify the Pipe Writer
Three independent methods were used to try to identify the process that held the write end of the pipe. The first used `lsof` to scan all open file descriptors across every running process on the system and filter for the specific pipe inode, which would have returned the process name and PID of anything still holding that pipe open. 
```bash
# Method 1: lsof
sudo lsof -n | grep 924958
# no output, process gone
```
The second went directly to the proc filesystem, which exposes every open file descriptor of every running process as a virtual directory entry, and searched for any process that had file descriptor 39 open pointing to that specific pipe inode. 
```
# Method 2: proc filesystem
sudo ls -la /proc/*/fd/39 2>/dev/null | grep 'pipe:\[924958\]'
# no output
```
The third used `fuser`, which identifies processes using a specific file or socket. All three returned nothing. 
```
# Method 3: fuser
sudo fuser /proc/pipe/924958 2>/dev/null
# no output
```
This was conclusive, the process was not running anymore and had left no trace. The most likely explanation is that it was launched manually from a terminal session or a shell script, and when that session was closed or the script exited, the process terminated with it, taking the only copy of the pipe write end with it and leaving the pipe inode itself to be garbage collected by the kernel.

### 5.4 DFA Binary Analysis
Binary analysis of DFA was necessary because the application had no documentation, no visible source code, and no standard configuration file in a readable format, understanding what it was built with and how it stored its settings was essential to knowing whether it could be reconfigured to forward telemetry to the FCC without modifying source code.

Running the `file` command against the DFA executable confirmed it was a PyInstaller bundle, which is a packaging method that takes a Python application and wraps it together with the Python interpreter and all its dependencies into a single self-contained Linux executable. This was important because it meant DFA was fundamentally a Python application despite appearing as a binary, and that its behavior could potentially be understood by examining its extracted contents. PyInstaller bundles extract themselves into a temporary directory at runtime, and finding that directory at `/tmp/_MEIhb2hL6/` and seeing hundreds of Python files and shared libraries inside confirmed this and gave a picture of the application's dependencies.
```bash
file /home/pilot/DFA/DFA
# ELF 64-bit LSB executable, x86-64 PyInstaller bundle

ls /tmp/_MEIhb2hL6/
# PyInstaller runtime extraction directory (hundreds of .so and .py files)
```
Checking the `MAVLINK_DIALECT` environment variable confirmed the system was configured for the ardupilotmega dialect, which is the extended MAVLink message set used by ArduPilot flight controllers, consistent with what MAVProxy had already detected from the vehicle.

```
echo $MAVLINK_DIALECT
# ardupilotmega

```
Finding the DFA directory structure at `/home/pilot/DFA/` showed the application was organized into three components: the main binary, the serverC backend in its own subdirectory, and a static folder containing the settings files. The static folder contained four settings profiles in Python pickle binary format, a serialization format that cannot be read or edited as plain text, which explained why the settings could not simply be opened in a text editor and modified to add a forwarding address for the FCC.
```
find / -name "DFA" -type d 2>/dev/null
# /home/pilot/DFA/

ls -la /home/pilot/DFA/
# drwxrwxr-x  DFA (binary)
# drwxrwxr-x  serverC/
# drwxrwxr-x  static/   <- settings files here

ls -la /home/pilot/DFA/static/
# .settings         <- active config (binary pickle)
# new.settings
# original.settings
# 1413.settings
```

### 5.5 Launcher Search, Negative Result
Finding the launcher was critical because the pipe writer process,  the one feeding MAVLink data into DFA's stdin was spawned by whatever script or command originally started DFA. Identifying the launcher would have revealed exactly how the pipe was created, what process was writing into it, and where that process was getting its data from, which would have pointed directly to the correct MAVLink source and potentially offered a cleaner fix than MAVProxy.<br>
The search covered every plausible location where a launch mechanism might exist. Shell scripts were checked directly in the DFA directory and the pilot home directory since it is common practice to wrap application launches in a simple bash script that sets environment variables and starts companion processes together. The autostart directory was checked because desktop environments like GNOME and KDE use `.desktop` files placed there to automatically launch applications when a user logs in. Both systemd user services and system-wide services were checked because any properly managed background process on a Linux system would typically be registered there, making it controllable and persistent across reboots.

```bash
# Look for shell scripts
cat /home/pilot/DFA/start.sh 2>/dev/null          # nothing
ls -la /home/pilot/DFA/*.sh 2>/dev/null           # no .sh files
ls -la /home/pilot/*.sh 2>/dev/null               # nothing

# Autostart entries
ls -la ~/.config/autostart/
# ls: cannot access: No such file or directory

cat ~/.config/autostart/*.desktop 2>/dev/null     # nothing

# systemd
systemctl --user list-units --type=service        # standard desktop services only
systemctl list-units --type=service | grep -i "dfa\|pilot\|mavlink\|micro"
# no matches
```
Every single check returned nothing. No shell script, no desktop autostart entry, no systemd service of any kind was associated with DFA. This meant DFA and its companion processes were being started manually from a terminal by the operator, which explained why the pipe writer process disappeared when the session ended — it was never managed by any init system or service manager that would have kept it alive or recorded how it was launched. The launch mechanism was effectively undocumented, existing only in the operator's memory or an external procedure that was not present on the machine.

### 5.6 Bash History, Key Discovery
Examining the bash history was necessary because the pipe writer process was gone and could not be identified through any live system inspection. The history represented a record of every command previously run on the machine, and searching it for keywords related to DFA, MAVProxy, socat, and Microhard was the most direct way to understand what had already been attempted and what was already known about the system.

```bash
cat ~/.bash_history | grep -i "dfa\|mavproxy\|socat\|microhard" | tail -30
```

Critical lines found:
```bash
sudo apt install socat -y
socat UDP-RECV:14552 UDP-SENDTO:10.10.10.110:14550 &
which mavproxy.py
mavproxy.py --master=udp:0.0.0.0:14551 --out=udp:10.10.10.110:14550 --daemon
mavproxy.py --master=udp:0.0.0.0:14551 --out=udp:10.10.10.110:14550 --daemon
find / -name "DFA" -type d 2>/dev/null
cat /home/pilot/DFA/static/new.settings
cat /home/pilot/DFA/static/original.settings  
cat /home/pilot/DFA/static/1413.settings
grep -r "14550\|14551\|10.10.10.110" /home/pilot/DFA/ 2>/dev/null
socat UDP-RECV:14551,reuseaddr UDP-SENDTO:10.10.10.110:14550 &
mavproxy.py --master=udp:0.0.0.0:14551 --out=udp:10.10.10.110:14550
python3 -m pip install PyYAML mavproxy --user
mavproxy.py --version
```
The command read the entire bash history file and filtered it through a case-insensitive search for any of the four keywords, returning the last thirty matching lines to keep the output manageable. What it revealed was immediately useful. Socat had been installed and tested as a raw UDP forwarder on both port 14552 and 14551. MAVProxy had been installed, its version checked, and run with specific arguments. The DFA settings files had been examined. The DFA directory had been searched for any configuration referencing the relevant port numbers and the FCC address.
Most critically, the history contained the complete and correct MAVProxy command with --master=udp:0.0.0.0:14551, which confirmed that UDP port 14551 was the correct MAVLink source port on the GCS. This was the key piece of information that the rest of the investigation had been working toward — knowing exactly which port to listen on meant the forwarding command could now be constructed with confidence and the fix could be implemented immediately.

---

## 6. Phase 4 - Configuration Analysis
With the internal GCS architecture fully understood, we examined the configuration files on both machines to determine whether any misconfiguration could explain the missing forwarding path. The FCC configuration file was inspected parameter by parameter, the DFA settings file was extracted and analyzed despite its binary format, and the complete port state of the GCS was mapped to produce a final picture of exactly what was and was not listening on the machine.

### 6.1 FCC Configuration File
The FCC configuration file was located at `D:\FCC11\logs\config.xml` and examined to determine whether any misconfiguration on the FCC side could explain the connection failure. The file stores all connection parameters for every peripheral and service the FCC communicates with.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config>
  <!-- MAVLink Transport -->
  <comport>UDP</comport>
  <UDP_port>14550</UDP_port>

  <!-- GCS Export (unused — nothing listens here on GCS) -->
  <TCP_host>10.10.10.11</TCP_host>
  <TCP_port>14651</TCP_port>

  <!-- Antenna Tracker -->
  <TCP_Ant_ip>10.10.10.5</TCP_Ant_ip>
  <TCP_Ant_port>14661</TCP_Ant_port>

  <!-- Radio Monitoring -->
  <SNMP_host>10.10.10.1</SNMP_host>

  <!-- RTK GNSS Base -->
  <TCP_RTK_ip>10.10.10.111</TCP_RTK_ip>
  <TCP_RTK_port>222</TCP_RTK_port>

  <!-- Camera -->
  <RTSP_Cam>rtsp://admin:admin1@10.10.10.2:554/cam/realmonitor?channel=1&amp;subtype=0</RTSP_Cam>

  <!-- Features -->
  <use_mikrohard>True</use_mikrohard>
  <use_cryptodisc>True</use_cryptodisc>
  <Export_IP>10.10.10.11</Export_IP>
</config>
```

The MAVLink transport was correctly set to UDP with port 14550, which is the standard port for a ground control station to receive telemetry. The GCS IP address was correctly entered as 10.10.10.11, and a TCP export port 14651 was configured pointing to the GCS, though nothing on the GCS was listening on that port. The antenna tracker address and port matched exactly what was confirmed active on the network. The RTK base station was correctly pointed to 10.10.10.111 on port 222, which the nmap scan had already confirmed was serving a raw GPS data stream. The camera RTSP address was correct and had been verified working in VLC. The Microhard radio mode and encrypted disk feature were both enabled as expected.

**FCC Configuration Status:**

| Parameter | Value | Status |
|---|---|---|
| `comport` | `UDP` | ✅ Correct |
| `UDP_port` | `14550` | ✅ Correct standard MAVLink GCS port |
| `TCP_host` | `10.10.10.11` | ✅ Correct GCS IP |
| `TCP_port` | `14651` | ⚠️ Correct format, but nothing listening here on GCS |
| `TCP_Ant_ip` | `10.10.10.5` | ✅ Correct |
| `TCP_Ant_port` | `14661` | ✅ Correct |
| `TCP_RTK_ip` | `10.10.10.111` | ✅ Confirmed by nmap |
| `TCP_RTK_port` | `222` | ✅ UBX/NMEA stream verified |
| `RTSP_Cam` | `rtsp://...10.10.10.2:554/...` | ✅ Verified in VLC |
| `use_mikrohard` | `True` | ✅ Correct radio mode |

Every single parameter in the configuration file was correct. This was a significant finding because it ruled out FCC misconfiguration as a contributing factor and placed the entire problem back on the GCS side. One minor discrepancy was noted where the FCC user interface displayed the RTK base address as 10.10.10.4 on screen, but the underlying config.xml stored the correct address 10.10.10.111, meaning the functional connection was working properly and the wrong address was simply a cosmetic display bug in the FCC software itself with no effect on actual connectivity.

### 6.2 DFA Settings File

The DFA settings file was a Python pickle binary, meaning it was serialized Python data stored in a binary format that cannot be opened or edited with a text editor. It was located at `/home/pilot/DFA/static/.settings` and four profiles existed in that directory: the active settings file loaded at runtime, an alternative windowed profile, a baseline backup, and a port-variant configuration, suggesting the system had been configured differently at various points.

```bash
# Multiple settings profiles exist
ls -la /home/pilot/DFA/static/
```

| File | Purpose |
|---|---|
| `.settings` | Active config (binary pickle, loaded at runtime) |
| `new.settings` | Alternative windowed/test profile |
| `original.settings` | Baseline/backup configuration |
| `1413.settings` | Port-variant configuration |

The readable contents of the active settings file showed the aircraft type set to leleka100, MAVLink configured for UDP on port 14550, video sourced from RTSP, and the ground antenna camera address pointing to 10.10.10.22 on port 554. The fullscreen flag was set to false indicating the application was running in windowed mode, consistent with a desktop testing configuration rather than a dedicated full-screen operator display. The file also contained a calibration lookup table and a field of view list covering thirty zoom levels ranging from 65.5 degrees at the widest end down to 3.6 degrees at maximum zoom, which are optical parameters for the camera system.

```python
{
    'plane': 'leleka100',
    'mavlink': {
        'protocol': 'udp',
        'port': 14550
    },
    'video_stream_from': 'RTSP',
    'antena_video_src': 'rtsp://admin:admin1@10.10.10.22:554/cam/realmonitor?channel=1&subtype=0',
    'fullscreen': False,       # windowed mode — test/desktop profile
    'css': [...],              # calibration lookup table (30 zoom levels)
    'fov_list': [              # FOV from 65.5° (wide) to 3.6° (narrow)
        65.5, 58.2, 51.0, ...
    ]
}
```
The most important finding was the contradiction between what the settings file said and what DFA was actually doing. The settings specified UDP port 14550 as the MAVLink source, but DFA was not listening on that port at all, it was receiving MAVLink through the Unix pipe on its standard input instead. This meant DFA was ignoring the network socket configuration in its settings file and using the pipe-based delivery method determined by how it was launched, and because the pickle format prevented direct editing, the settings file could not simply be modified to add a forwarding address for the FCC.

### 6.3 Port Architecture, Complete View
The port map summarized the complete listening state of the GCS machine at the time of investigation. DFA's internal interface on TCP port 8085 was bound to 127.0.0.1, making it reachable only from within the machine itself. The serverC video relay on TCP port 8554 was bound to all interfaces, meaning it was technically reachable from the network but served only video data. TeamViewer was listening on TCP port 5939, also bound to localhost only, providing remote access to the machine. Most significantly, UDP port 14551 is the port that the bash history had confirmed was the correct MAVLink source and had no process listening on it at all. This last point was the architectural gap in plain numbers: the port where MAVLink data was expected to be available was completely unoccupied, meaning even if the FCC had known to look there, nothing would have responded.
```mermaid
graph LR
    subgraph PORTS["<b style='font-size:16px'>GCS Port Map 10.10.10.11</b> "]
        P1["<b>TCP :8085</b>\n LISTENING\n127.0.0.1 only\nDFA internal"]
        P2["<b>TCP :8554</b>\n LISTENING\n0.0.0.0\nserver video relay"]
        P3["<b>TCP :5939</b>\n LISTENING\n127.0.0.1\nTeamViewer"]
        P4["<b>UDP :14551</b>\nNOT LISTENING\nexpected source\nport"]
    end
```

---

## 7. Root Cause Summary
Having completed all four phases of investigation, the root cause became unambiguous. This section consolidates the findings, tracing the exact path telemetry takes from the drone to the GCS, showing where that path terminates, confirming every component that was working correctly, and listing every explanation that was ruled out. Leaving a single conclusion: the system had no mechanism to forward MAVLink data from the GCS to the FCC.

### 7.1 Data Flow, Before Fix
The flight controller sends MAVLink data over a serial connection at 57,600 baud into the UAV-side Microhard radio, which transmits it wirelessly at 900 MHz to the ground-side Microhard radio. The ground radio makes that data available through its HTTP API on port 80, which serverC queries to retrieve the telemetry. ServerC then writes that data directly into DFA's standard input through the Unix pipe, and DFA consumes it internally and displays it on localhost port 8085, at which point the data's journey ends. Nothing in the entire chain produces any outbound network traffic directed at the FCC at 10.10.10.110. On the FCC side, the operator presses CONNECT, the software waits for MAVLink heartbeats that never arrive, the connection attempt times out, and the status returns failed. The telemetry traveled from the drone through the radio link and into the GCS software correctly, but stopped there with no forwarding path to the second operator station.
```mermaid
sequenceDiagram
    participant UAV as UAV FC
    participant MH_UAV as Microhard UAV<br/>10.10.10.10
    participant MH_GND as Microhard GND<br/>10.10.10.1
    participant SERV as serverC<br/>PID 20734
    participant DFA as DFA App<br/>PID 20538
    participant FCC as FCC<br/>10.10.10.110

    UAV->>MH_UAV: MAVLink serial<br/>57600 baud
    MH_UAV-->>MH_GND: 900 MHz<br/>RF link
    MH_GND->>SERV: HTTP API<br/>port 80
    SERV->>DFA: Unix pipe<br/>stdin (fd=0)
    DFA->>DFA: Display<br/>localhost:8085 ✅
    
    Note over DFA,FCC: ❌ NO DATA<br/>FORWARDED<br/>TO FCC
    FCC->>FCC: CONNECT<br/>button
    FCC->>FCC: timeout<br/>FAILED
```

### 7.2 Root Cause Chain
Telemetry arrives at the GCS from the Microhard radio's HTTP API, where serverC retrieves it and delivers it to DFA through the Unix pipe. DFA receives it on its standard input and serves it internally on localhost port 8085, but that port is bound to 127.0.0.1 making it completely invisible to the rest of the network. Nothing on the GCS forwards that MAVLink data toward the FCC at 10.10.10.110 on port 14550, and nothing listens on port 14651 where the FCC's TCP export is configured to connect. The end result is that the FCC receives zero MAVLink packets and every connection attempt fails.

```mermaid
graph LR
    A["UAV telemetry\narrives at GCS\nvia Microhard\nHTTP API at\n10.10.10.10:80"] --> B
    B["serverC consumes\ntelemetry and\ndelivers to DFA\nvia Unix pipe"] --> C
    C["DFA receives\non stdin fd=0\nand serves\ninternally on\n localhost:8085"] --> D
    D["localhost:8085\nbound to 127.0.0.1\ncompletely inaccessible\nfrom network"] --> E
    E["Nothing on GCS\nforwards MAVLink\nto 10.10.10.110:14550 \nor listens on :14651"] --> F
    F["🔴 FCC receives\nZERO MAVLink\npackets CONNECT\nfails every time"]
```
### 7.3 Full System Status Table
Every hardware and network component in the system was confirmed working. The network was fully reachable, both Microhard radios were online and responding, the camera stream was verified, the RTK base was streaming GPS data, the antenna tracker was connected, and DFA was displaying live UAV telemetry correctly on the GCS screen. The Raspberry Pi was confirmed as having no role in the data flow. The only two failures in the entire system were both on the software routing side — no MAVLink port was exposed anywhere on the GCS network interface, and as a direct consequence the FCC received no MAVLink packets and could not connect.
| Component | Status | Detail |
|---|---|---|
| Network connectivity (all devices) | ✅ OK | All ping and service connections working |
| Microhard 900 MHz radio link | ✅ OK | Both radios online, HTTP API reachable |
| UAV camera RTSP `10.10.10.2:554` | ✅ OK | Verified working in VLC from FCC |
| RTK Base `10.10.10.111:222` | ✅ OK | Connected, GPS data streaming |
| Antenna Tracker `10.10.10.5:14661` | ✅ OK | GCS connected, proprietary protocol working |
| DFA telemetry display on GCS | ✅ OK | UAV parameters visible on GCS screen |
| Raspberry Pi `10.10.10.3` | ✅ PASSIVE | Only mDNS — not a MAVLink bridge |
| MAVLink available on GCS network | ❌ MISSING | No UDP/TCP MAVLink port exposed |
| FCC CONNECT to UAV | ❌ FAILED | FCC receives no MAVLink packets |

### 7.4 What Was NOT the Problem
Every conventional explanation was systematically eliminated. The radio link was ruled out because both Microhard radios were online and exchanging data. Network addressing and routing were ruled out because every device on the subnet responded to pings with zero packet loss. FCC misconfiguration was ruled out because every parameter in config.xml was verified correct. The Raspberry Pi was ruled out not as a broken bridge but as never having been a bridge at all, it had no MAVLink ports open and generated only mDNS traffic. DFA software failure was ruled out because DFA was functioning perfectly and displaying live telemetry to the GCS operator throughout the entire investigation. Camera and video issues were ruled out because the RTSP stream loaded successfully in VLC from the FCC. What remained after eliminating all of these was a single architectural gap, the absence of any forwarding path between the GCS and the FCC.
- Radio link failure (both Microhards online)
- IP addressing or routing issues (all pings OK)
- FCC misconfiguration (config.xml was correct)
- Raspberry Pi as broken bridge (Pi never was a bridge)
- DFA software failure (DFA worked perfectly for the GCS operator)
- Camera or video issues (RTSP confirmed working)

---

## 8. Resolution & Fix Applied

### 8.1 Fix Strategy Selection

Three possible approaches were evaluated before implementing the fix. The first was reconfiguring serverC or DFA to output MAVLink directly to the FCC, but this was not viable because DFA is a closed-source PyInstaller bundle with no documented configuration option for adding a network output, and serverC's source code was equally inaccessible. The second was reconfiguring the Microhard radio's HTTP API to also expose raw MAVLink UDP alongside its existing interface, but this carried significant risk because the Microhard firmware configuration is undocumented and any mistake could have disrupted the entire radio link and taken down the UAV communication chain entirely. The third approach was running MAVProxy as a UDP forwarder on the GCS, listening on port 14551 and forwarding everything it received to the FCC at 10.10.10.110 on port 14550. This was chosen because it required no changes to any existing component, MAVProxy was already installed and confirmed working on the machine, the correct command arguments were already known from the bash history, and the forwarder would operate independently of DFA meaning a DFA restart would not affect it. It was the only option that was both safe and immediately actionable.
```mermaid
graph TD
    A[<b style='font-size:16px'>Fix Options</b>] --> B["<b>Option A</b>\nReconfigure serverC/DFA\nto output MAVLink to FCC"]
    A --> C["<b>Option B</b>\nReconfigure Microhard\nHTTP API to also expose\nraw MAVLink UDP"]
    A --> D["<b>Option C ✅ CHOSEN</b>\nRun MAVProxy as a\nUDP forwarder on GCS"]

    B --> B1["❌ Requires access to\nclosed source DFA source code\nor undocumented config"]
    C --> C1["❌ Requires Microhard\nfirmware configuration \nrisky, undocumented"]
    D --> D1["✅ Non-invasive\n✅ MAVProxy already installed\n✅ Confirmed by bash history\n✅ Survives DFA restarts"]
```

### 8.2 MAVProxy — Interactive Test
MAVProxy was run interactively first rather than immediately as a background service, so that the output could be observed directly and the link confirmed working before committing to a permanent deployment.
The command told MAVProxy to listen for incoming MAVLink data on all network interfaces on UDP port 14551, and simultaneously forward everything it received to the FCC at 10.10.10.110 on UDP port 14550.
```bash
mavproxy.py --master=udp:0.0.0.0:14551 --out=udp:10.10.10.110:14550
```

```
WARNING: You should uninstall ModemManager as it conflicts with APM and Pixhawk
Connect udp:0.0.0.0:14551 source_system=255
Log Directory: 
Telemetry log: mav.tlog
Waiting for heartbeat from 0.0.0.0:14551
Detected vehicle 1:240 on link 0
AP: V6.393.7
AP: fmuv3 00410021 33355108 38343732
AP: RCOut: PWM:1-14
Detected vehicle 1:1 on link 0
online system 1
MANUAL> Mode MANUAL
AP: V6.393.7
...
Received 1112 parameters
Saved 1111 parameters to /tmp/mav.parm
Flight battery 90 percent
Got COMMAND_ACK: DO_MOUNT_CONTROL: ACCEPTED
Got COMMAND_ACK: DO_SET_SERVO: ACCEPTED
Got COMMAND_ACK: PREFLIGHT_STORAGE: FAILED
Flight battery 90 percent
```
The output confirmed the link immediately. Two vehicles were detected, system ID 1 component 1 which is the main ArduPilot flight controller, and system ID 1 component 240 which is the camera gimbal or a secondary autopilot component. The heartbeat was confirmed, the current flight mode came through as MANUAL, and a full download of 1,112 parameters completed successfully and was saved to disk. Live telemetry followed immediately with the flight battery reporting at 90 percent, and command acknowledgements came back from the flight controller confirming two-way communication was established. The FCC connected to the UAV successfully while MAVProxy was running.
Two warnings appeared in the output but neither indicated any problem. The SRTM error was MAVProxy attempting to download terrain elevation data for its map display and failing due to a missing local cache file, which has no effect on MAVLink forwarding. The PREFLIGHT_STORAGE failure was ArduPilot reporting that its onboard dataflash logging storage check did not pass, which is a non-critical self-test unrelated to the ground link. The ModemManager warning was a standard advisory that the service can interfere with serial ports connected to flight controllers, which was irrelevant here since no serial ports were in use on the GCS.
| Line | Meaning | Status |
|---|---|---|
| `Detected vehicle 1:1 on link 0` | Main UAV flight controller found | ✅ |
| `Detected vehicle 1:240 on link 0` | Camera gimbal/secondary component | ✅ |
| `online system 1` | MAVLink heartbeat confirmed | ✅ |
| `MANUAL> Mode MANUAL` | Current flight mode received | ✅ |
| `Received 1112 parameters` | Full parameter download | ✅ |
| `Flight battery 90 percent` | Live telemetry flowing | ✅ |
| `SRTM FileNotFoundError` | Missing terrain cache **harmless** | ⚠️ |
| `PREFLIGHT_STORAGE: FAILED` | ArduPilot dataflash log check **non-critical** | ⚠️ |

### 8.3 Run in Daemon Mode
A daemon is a background process that runs independently of any terminal session, has no interactive input or output, and continues running silently as long as the system is up regardless of whether anyone is logged in or has a terminal open. Unlike a normal process that is tied to the terminal that launched it and dies when that terminal is closed, a daemon detaches from the session completely and runs as a standalone background service managed by the operating system.

Running MAVProxy interactively meant it was tied to the terminal window, closing that window or logging out would have killed the process and severed the FCC connection immediately. Switching to daemon mode with the `--daemon` flag detached MAVProxy from the terminal entirely, meaning it continued forwarding MAVLink to the FCC in the background without requiring any terminal to remain open. The `--logfile` flag redirected all output that would normally appear on screen to a log file at `/tmp/mav.log` instead, so that the forwarding activity could still be monitored and reviewed later without needing an interactive session. This made MAVProxy operational as a persistent background service rather than a temporary interactive tool, which was the necessary intermediate step before converting it into a proper systemd service for full boot persistence.
```bash
mavproxy.py --master=udp:0.0.0.0:14551 \
            --out=udp:10.10.10.110:14550 \
            --daemon \
            --logfile=/tmp/mav.log
```

### 8.4 Data Flow After Fix
After the fix, the data flow split into two parallel paths from the Microhard radio onward. The first path remained unchanged, serverC continued querying the Microhard HTTP API, delivering telemetry to DFA through the Unix pipe, and DFA continued displaying it on the GCS screen as before. The second path was the new addition, the Microhard radio also delivered MAVLink data on UDP port 14551 where the MAVProxy systemd service was now listening, and MAVProxy forwarded everything it received directly to the FCC at 10.10.10.110 on UDP port 14550. With data now arriving at the FCC, parameters loaded successfully, telemetry appeared on all display widgets, and the connection status showed connected. Both operator stations, the GCS and the FCC, were now receiving live UAV telemetry simultaneously through their respective paths, with neither path interfering with the other.
```mermaid
sequenceDiagram
    participant UAV as ✈️ UAV FC
    participant MH as 📡 Microhard\nRadio Link
    participant SERV as serverC
    participant DFA as DFA App
    participant MAV as 🔀 MAVProxy\nsystemd service
    participant FCC as 🖥️ FCC 4.0.11

    UAV->>MH: MAVLink serial 57600 baud
    MH->>SERV: HTTP API :80
    SERV->>DFA: pipe stdin ✅
    DFA->>DFA: localhost:8085\nGCS display ✅
    
    MH->>MAV: UDP :14551 ✅
    MAV->>FCC: UDP 10.10.10.110:14550 ✅
    
    Note over FCC: Parameters loaded ✅
    Note over FCC: Telemetry displayed ✅
    Note over FCC: CONNECTED ✅
```

---

## 9. Persistence & systemd Service
With the root cause confirmed and all alternative approaches evaluated, the resolution was straightforward and non-invasive. Mavproxy is selected as the forwarding solution, the interactive test that confirmed the link, the switch to daemon mode for background operation, and the resulting data flow that brought both operator stations online simultaneously.

### 9.1 Create the Service File
The systemd service file was created using `sudo tee` which writes the content to the file with root privileges, with the `<< 'EOF'` heredoc syntax allowing the entire multi-line file content to be written in a single command. The file was placed in `/etc/systemd/system/` which is the standard location for system-wide services on Linux.

The service file itself is divided into three sections. The `[Unit]` section gives the service a human-readable description and sets `After=network.target` which tells systemd to wait until the network interfaces are up before starting the service, ensuring the UDP ports are available when MAVProxy launches.

The `[Service]` section defines how the process runs. The `User=pilot` line runs the service as the pilot user rather than root, which is safer and ensures MAVProxy can access files in the pilot home directory. The `ExecStart` line contains the full MAVProxy command with all its arguments: listening on UDP port 14551, forwarding to the FCC on UDP port 14550, running in daemon mode, writing logs to `/tmp/mav.log`, and running non-interactively since there is no terminal to interact with in a service context. `Restart=always` tells systemd to restart the service automatically if it ever stops for any reason, and `RestartSec=5` sets a five second delay before each restart attempt to prevent rapid crash loops from overwhelming the system.

The `[Install]` section with `WantedBy=multi-user.target` registers the service to start automatically during normal system boot, meaning MAVProxy will be running and forwarding telemetry to the FCC from the moment the GCS machine finishes booting without any manual intervention required.
```bash
sudo tee /etc/systemd/system/mavproxy-fcc.service << 'EOF'
[Unit]
Description=MAVProxy FCC Forwarder
After=network.target

[Service]
User=pilot
ExecStart=/home/pilot/.local/bin/mavproxy.py \
  --master=udp:0.0.0.0:14551 \
  --out=udp:10.10.10.110:14550 \
  --daemon \
  --logfile=/tmp/mav.log \
  --non-interactive
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 9.2 Enable and Start
After writing the service file, three commands were run in sequence to activate it. `daemon-reload` told systemd to re-read all service files from disk, which is always required after creating or modifying a service file because systemd caches its configuration in memory and would otherwise be unaware of the new file. `enable` created a symbolic link from the system boot target directory to the service file, which is what causes the service to start automatically on every subsequent boot. `start` then launched the service immediately without requiring a reboot, bringing MAVProxy online right away rather than waiting until the next time the machine was restarted.
```bash
sudo systemctl daemon-reload
sudo systemctl enable mavproxy-fcc
sudo systemctl start mavproxy-fcc
```

### 9.3 Confirm Running
Running `systemctl status` confirmed the service was fully operational. The output showed the service loaded correctly from its file path, marked as enabled meaning it will start on every boot, and actively running with a process ID of 244012. The service had been running for 16 seconds at the time the status was checked, was consuming 55.8 megabytes of memory, and was running as a Python process with the complete MAVProxy command visible in the process tree.
```bash
sudo systemctl status mavproxy-fcc
```
The log lines beneath confirmed the service behaved identically to the earlier interactive test. Systemd started the service at 13:10:17, and within one second MAVProxy had opened the UDP socket, detected both vehicle components: the gimbal at component ID 240 and the main flight controller at component ID 1: confirmed the heartbeat, and received the current flight mode as MANUAL. The entire startup sequence from service launch to live telemetry took under one second, and the FCC connected successfully from that point onward.
```
● mavproxy-fcc.service - MAVProxy FCC Forwarder
     Loaded: loaded (/etc/systemd/system/mavproxy-fcc.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2026-05-25 13:10:17 EEST; 16s ago
   Main PID: 244012 (mavproxy.py)
      Tasks: 5 (limit: 18870)
     Memory: 55.8M
     CGroup: /system.slice/mavproxy-fcc.service
             └─244012 /usr/bin/python3 /home/pilot/.local/bin/mavproxy.py \
               --master=udp:0.0.0.0:14551 --out=udp:10.10.10.110:14550 \
               --daemon --logfile=/tmp/mav.log --non-interactive

May 25 13:10:17 pilot-NUC8i7BEH systemd[1]: Started MAVProxy FCC Forwarder.
May 25 13:10:18 pilot-NUC8i7BEH mavproxy.py[244012]: Connect udp:0.0.0.0:14551 source_system=255
May 25 13:10:18 pilot-NUC8i7BEH mavproxy.py[244012]: Waiting for heartbeat from 0.0.0.0:14551
May 25 13:10:18 pilot-NUC8i7BEH mavproxy.py[244012]: Detected vehicle 1:240 on link 0
May 25 13:10:18 pilot-NUC8i7BEH mavproxy.py[244012]: Detected vehicle 1:1 on link 0
May 25 13:10:18 pilot-NUC8i7BEH mavproxy.py[244012]: online system 1
May 25 13:10:18 pilot-NUC8i7BEH mavproxy.py[244012]: MANUAL> Mode MANUAL
```

### 9.4 Service State Diagram
The service moves through a defined set of states during its lifecycle. When the machine boots, the service begins in an inactive state and transitions to activating the moment systemd triggers it, either through a manual start command or automatically at boot because it is enabled. Once MAVProxy starts and detects a heartbeat from the UAV, the service moves into the active running state where it stays indefinitely under normal operation, continuously forwarding telemetry to the FCC. If the process crashes or is killed for any reason, the service moves to a failed state, but because `Restart=always` is configured, systemd automatically begins the activating sequence again after the five second delay defined by `RestartSec`, bringing MAVProxy back online without any manual intervention. The only way the service reaches a permanent inactive state is through an explicit stop command. During normal active operation the service runs with process ID 244012, consuming 55.8 megabytes of memory across five tasks, with the UAV flight controller confirmed online.
```mermaid
stateDiagram-v2
    [*] --> inactive: System boot
    inactive --> activating: systemctl start</br>(or boot enabled)
    activating --> active: MAVProxy starts</br>heartbeat detected
    active --> failed: Process crashes</br>or killed
    failed --> activating: RestartSec=5s</br>auto-restart
    active --> inactive: systemctl stop
    active --> active: Normal operation</br>(continuous)

    note right of active
        PID 244012
        Memory: 55.8M
        Tasks: 5
        Vehicle 1:1 online
    end note
```

### 9.5 Service Management Commands
Service management commands are the set of systemctl and journalctl commands used to control, monitor, and inspect a systemd service after it has been created and deployed.

```bash
# Check status and recent logs
sudo systemctl status mavproxy-fcc
```
`systemctl status` displays the current state of the service along with its recent log lines, showing whether it is running, how long it has been up, its process ID, and memory usage. It is the first command to run whenever there is any doubt about whether the service is functioning correctly.
```
# Stream live logs
sudo journalctl -u mavproxy-fcc -f
```
`journalctl -u -f` streams the service logs live to the terminal in real time as new lines are written, equivalent to watching the MAVProxy output as it happens. It is useful for monitoring heartbeat detection and telemetry flow during active operations.
```
# View logs from last hour
sudo journalctl -u mavproxy-fcc --since "1 hour ago"
```
`journalctl --since` retrieves historical log entries from a specified time window rather than streaming live output, allowing past service behavior to be reviewed after the fact. It is useful for diagnosing issues that occurred while no one was watching.
```
# Restart service
sudo systemctl restart mavproxy-fcc
```
`systemctl restart` stops the service and immediately starts it again in a single command, which is used when the service needs to be refreshed after a configuration change or when it is misbehaving without having fully crashed. 
```
# Stop service
sudo systemctl stop mavproxy-fcc
```
`systemctl stop` shuts the service down cleanly without disabling it, meaning it will still start automatically on the next boot. It is used when MAVProxy needs to be temporarily halted without removing it from the boot sequence.
```
# Disable auto-start on boot
sudo systemctl disable mavproxy-fcc
```
`systemctl disable` removes the symbolic link that causes the service to start at boot, so the service will no longer launch automatically after a reboot even though the service file itself remains on disk.
```
# Re-enable auto-start
sudo systemctl enable mavproxy-fcc
```
`systemctl enable` recreates that symbolic link, restoring automatic startup on boot. It is the counterpart to disable and is used to re-register the service with the boot sequence after it has been disabled.
### 9.6 Kill Duplicate Instance

When the systemd service was started, there was a possibility that the manually launched MAVProxy instance from the earlier interactive testing was still running in the background as a daemon. Two MAVProxy processes listening on the same UDP port 14551 simultaneously would compete for the incoming packets, causing unpredictable behavior where some telemetry frames would be consumed by one instance and some by the other, resulting in incomplete and corrupted data reaching the FCC. Running `ps aux` and filtering for mavproxy listed all running instances with their process IDs.

```bash
ps aux | grep mavproxy
```

And any PID other than the service PID 244012 indicated a stale manual instance that needed to be terminated with the `kill` command to leave the systemd service as the sole owner of that port.

```bash
kill <old_pid>
```

### 9.7 Disable ModemManager

ModemManager is a Linux service that automatically manages mobile broadband and serial modem devices by scanning and claiming serial ports when they appear on the system. ArduPilot and Pixhawk flight controllers communicate over serial ports, and ModemManager's habit of probing and interfering with those ports can disrupt MAVLink communication, which is why MAVProxy warns about it at every startup.
```
WARNING: You should uninstall ModemManager as it conflicts with APM and Pixhawk
```
On this GCS however no flight controller is connected via serial port. The MAVLink data arrives over UDP from the Microhard radio, so ModemManager has nothing relevant to manage and its presence serves no purpose. Stopping and disabling it removes the warning from every subsequent MAVProxy startup and eliminates any theoretical risk of interference with no downside on this particular machine.
```bash
sudo systemctl stop ModemManager
sudo systemctl disable ModemManager
```

---

## 10. Full Command Reference

### 10.1 Complete Reference of Linux GCS Commands

#### Network Discovery & Reachability
```bash
ping -c 4 10.10.10.x                             # Basic reachability (4 packets)
nmap -sn 10.10.10.0/24                           # Discover all subnet devices
nmap -sV 10.10.10.x                              # Service version scan
nmap -sU -p 14550,14551,5760,700 10.10.10.3      # UDP MAVLink check on Pi
ifconfig                                          # Interface config and IPs
ip addr show                                      # Modern equivalent of ifconfig
ip route show                                     # Routing table
```

#### Process Analysis
```bash
ps aux | grep -iE "dfa|server|mav"               # List target processes
ps aux | grep mavproxy                            # Check for mavproxy instances
top -p $(pgrep DFA)                               # Monitor DFA resource usage
```

#### Socket & Connection Analysis
```bash
netstat -tn | grep ESTABLISHED                    # Active TCP connections
sudo ss -tlnp                                     # Listening TCP (with PIDs)
sudo ss -ulnp                                     # Listening UDP (with PIDs)
sudo netstat -tnp | grep -iE "dfa|server"        # Process-to-connection map
sudo netstat -tnp | grep 14550                    # Check MAVLink port binding
```

#### File Descriptor & Pipe Analysis
```bash
sudo lsof -p $(pgrep DFA)                         # All DFA file descriptors
sudo lsof -p $(pgrep DFA) | grep -E 'UDP|TCP|IPv|pipe'   # DFA network/pipe FDs
sudo lsof -n | grep 924958                        # Find specific pipe inode
sudo ls -la /proc/*/fd/39 2>/dev/null | grep 'pipe:\[924958\]'   # Pipe by FD number
cat /proc/$(pgrep DFA)/maps | grep heap           # Memory layout
```

#### Traffic Capture
```bash
sudo tcpdump -i any -n host 10.10.10.110          # All traffic to/from FCC
sudo tcpdump -i any -n host 10.10.10.10           # Traffic to UAV Microhard
sudo tcpdump -i any udp port 14551 -n             # MAVLink UDP source port
sudo tcpdump -i any udp port 14550 -n             # MAVLink UDP FCC port
sudo tcpdump -i any tcp port 5760 -n              # MAVLink TCP alt port
sudo tcpdump -i any -n port 14661                 # Antenna tracker traffic
sudo tcpdump -i any udp port 14551 -c 1 -XX       # One packet hex dump (check magic byte 0xFD)
```

#### DFA Settings & Files
```bash
file /home/pilot/DFA/DFA                          # Confirm binary type
ls -la /home/pilot/DFA/static/                    # List settings files
grep -r "14550\|14551\|10.10.10.110" /home/pilot/DFA/ 2>/dev/null  # Search configs
echo $MAVLINK_DIALECT                             # MAVLink dialect env var
python3 -c "import pickle; print(pickle.load(open('/home/pilot/DFA/static/.settings','rb')))"
```

#### MAVProxy
```bash
# Install
python3 -m pip install mavproxy --user
python3 -m pip install PyYAML mavproxy --user

# Version check
mavproxy.py --version

# Test interactively (shows heartbeats in real time)
mavproxy.py --master=udp:0.0.0.0:14551 \
            --out=udp:10.10.10.110:14550

# Daemon mode with log
mavproxy.py --master=udp:0.0.0.0:14551 \
            --out=udp:10.10.10.110:14550 \
            --daemon \
            --logfile=/tmp/mav.log \
            --non-interactive

# Forward to multiple outputs simultaneously
mavproxy.py --master=udp:0.0.0.0:14551 \
            --out=udp:10.10.10.110:14550 \
            --out=udp:127.0.0.1:14552

# View live mavproxy log
tail -f /tmp/mav.log
```

#### socat (raw UDP bypass — alternative)
```bash
sudo apt install socat -y

# Forward raw UDP (no MAVLink parsing — pure relay)
socat UDP-RECV:14551,reuseaddr UDP-SENDTO:10.10.10.110:14550 &

# Verify it's running
jobs
ps aux | grep socat

# Kill socat background job
kill %1
```

#### systemd Service
```bash
sudo systemctl daemon-reload
sudo systemctl enable mavproxy-fcc
sudo systemctl start mavproxy-fcc
sudo systemctl stop mavproxy-fcc
sudo systemctl restart mavproxy-fcc
sudo systemctl status mavproxy-fcc
sudo journalctl -u mavproxy-fcc -f
sudo journalctl -u mavproxy-fcc --since "1 hour ago"
sudo journalctl -u mavproxy-fcc --since "2026-05-25"
```

#### Bash History Search
```bash
cat ~/.bash_history | grep -i "dfa\|mavproxy\|socat\|microhard" | tail -30
history | grep mavproxy
```

### 10.2 Windows FCC Commands

#### Network
```cmd
ipconfig
ipconfig /all
ping 10.10.10.x
tracert 10.10.10.11
```

#### Nmap (installed on FCC)
```cmd
nmap -sn 10.10.10.0/24
nmap -sV 10.10.10.1
nmap -sV 10.10.10.2
nmap -sV 10.10.10.3
nmap -sV 10.10.10.10
nmap -sV 10.10.10.11
nmap -sV 10.10.10.22
nmap -sV 10.10.10.110
nmap -sV 10.10.10.111
nmap -sU -p 14550,14551,5760,700 10.10.10.3
```

#### Port & Connection Checks
```cmd
netstat -an | findstr 14550
netstat -an | findstr 10.10.10.11
netstat -ano | findstr ESTABLISHED
```

#### Wireshark Filters
```
host 10.10.10.3                    -- Pi traffic only
host 10.10.10.110                  -- all FCC traffic
udp.port == 14550                  -- MAVLink UDP
udp.port == 14551                  -- MAVLink UDP source
mavlink_proto                      -- MAVLink protocol filter
```

#### FCC Config File
```
D:\FCC11\logs\config.xml
```

#### VLC Camera Test
```
Media → Open Network Stream:
rtsp://admin:admin1@10.10.10.2:554/cam/realmonitor?channel=1&subtype=0
rtsp://admin:admin1@10.10.10.22:554/cam/realmonitor?channel=1&subtype=0
```
---

## 11. Configuration Parameters Reference

### 11.1 MAVLink Protocol Parameters

| Parameter | Value |
|---|---|
| **MAVLink dialect** | `ardupilotmega` |
| **MAVLink version** | v2 (magic byte `0xFD`) |
| **Transport** | UDP |
| **GCS listen port** | `14551` |
| **FCC receive port** | `14550` |
| **Vehicle system ID** | `1` |
| **Vehicle component: FC** | `1` (ArduPilot) |
| **Vehicle component: gimbal** | `240` |
| **Total parameters** | `1112` (1111 saved) |
| **Firmware** | ArduPilot `V6.393.7` |
| **Hardware** | `fmuv3 00410021 33355108 38343732` |
| **PWM outputs** | `PWM:1-14` |
| **Baud rate** | `57600` baud (UAV serial → Microhard) |

### 11.2 Network Port Map

| Port | Protocol | Direction | Bound To | Service |
|---|---|---|---|---|
| `14550` | UDP | GCS → FCC | `0.0.0.0` (MAVProxy out) | MAVLink forwarding to FCC |
| `14551` | UDP | receive | `0.0.0.0` | MAVLink input from UAV chain |
| `14651` | TCP | FCC → GCS | N/A — unused | FCC telemetry export config (nothing listens) |
| `14661` | TCP | GCS → Tracker | `10.10.10.5` | Antenna tracker (proprietary protocol) |
| `8085` | TCP | internal | `127.0.0.1` only | DFA internal MAVLink server |
| `8554` | TCP | internal | `0.0.0.0` | Internal video relay DFA ↔ serverC |
| `554` | TCP | serverC → camera | `10.10.10.2` | RTSP UAV camera stream |
| `80` | TCP | serverC → Microhard | `10.10.10.10` | HTTP API telemetry retrieval |
| `222` | TCP | FCC → RTK | `10.10.10.111` | UBX/NMEA GPS data stream |
| `5939` | TCP | internal | `127.0.0.1` | TeamViewer remote access |

### 11.3 systemd Service File

**Path:** `/etc/systemd/system/mavproxy-fcc.service`

```ini
[Unit]
Description=MAVProxy FCC Forwarder
After=network.target

[Service]
User=pilot
ExecStart=/home/pilot/.local/bin/mavproxy.py \
  --master=udp:0.0.0.0:14551 \
  --out=udp:10.10.10.110:14550 \
  --daemon \
  --logfile=/tmp/mav.log \
  --non-interactive
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 11.4 RTSP Stream URLs

| Camera | URL |
|---|---|
| UAV camera (Dahua Z30) | `rtsp://admin:admin1@10.10.10.2:554/cam/realmonitor?channel=1&subtype=0` |
| Ground antenna camera | `rtsp://admin:admin1@10.10.10.22:554/cam/realmonitor?channel=1&subtype=0` |

### 11.5 Microhard Radio Credentials & Access

| Radio | IP | Management |
|---|---|---|
| Ground side | `10.10.10.1` | HTTP: `http://10.10.10.1` / SSH port 22 / Telnet port 23 |
| UAV side | `10.10.10.10` | HTTP: `http://10.10.10.10` / SSH port 22 / Telnet port 23 |

Both radios run identical OpenWrt profiles with uHTTPd, dnsmasq, Dropbear SSH, vsftpd FTP, and BusyBox telnet.

### 11.6 DFA Camera FOV Table (from settings pickle)

| Zoom Level | FOV |
|---|---|
| 1 (wide) | 65.5° |
| 5 | 51.0° |
| 10 | 36.0° |
| 15 | 22.0° |
| 20 | 14.0° |
| 25 | 7.5° |
| 30 (narrow) | 3.6° |

---

## 12. Conclusion & Recommendations
The investigation identified a single architectural gap in an otherwise fully functional system and resolved it without touching any existing component. This section presents the final system architecture with all data paths active, summarizes the problem and solution in a single reference table, and lists seven recommendations covering reboot verification, DFA launcher documentation, ModemManager cleanup, log rotation, Microhard API documentation, an FCC display bug, and future expansion of the MAVProxy forwarding configuration.

### 12.1 Final System Architecture After Fix
The final system architecture after the fix represents a fully operational ground control setup where every component is connected and every data path is active. The ArduPilot flight controller on the drone sends MAVLink telemetry over serial at 57,600 baud into the UAV-side Microhard radio, which transmits it across the 900 MHz wireless link to the ground-side Microhard radio. From that point the telemetry branches into two independent and simultaneous paths.

The first path is the original internal path that was always working; serverC queries the Microhard HTTP API on port 80, retrieves the telemetry, and delivers it to DFA through both the Unix pipe on stdin and the localhost video relay on port 8554. DFA processes everything and displays live UAV state on its internal interface for the GCS operator, while also controlling the antenna tracker at 10.10.10.5 on port 14661 to keep the directional antenna pointed at the drone.

The second path is the newly added forwarding path; the ground Microhard also delivers MAVLink data over UDP on port 14551 to the MAVProxy systemd service running as PID 244012 on the GCS, which immediately forwards every packet to the FCC at 10.10.10.110 on UDP port 14550. The FCC receives live telemetry, loads all parameters, and displays the full UAV state to the second operator.

Alongside these two telemetry paths, serverC maintains a direct RTSP connection to the UAV camera at 10.10.10.2 on port 554 for the video feed, and the FCC maintains its own direct TCP connection to the RTK base station at 10.10.10.111 on port 222 for precision GPS corrections. Every connection across the entire system is active, stable, and functioning correctly.
```mermaid
graph TD
    subgraph UAV_BLOCK["<b style='font-size:16px'>UAV</b>"]
        FC["<b>ArduPilot FC</b>\nfmuv3\nV6.393.7"]
        CAM_U["<b>Dahua Z30</b>\n10.10.10.2"]
        PI_U["<b>Raspberry Pi</b>\n10.10.10.3\nCrypto/Log"]
        MH_U["<b>Microhard UAV</b>\n10.10.10.10"]
        FC -->|serial 57600| MH_U
    end

    subgraph GND["<b style='font-size:16px'>Ground</b>"]
        MH_G["<b>Microhard GND</b>\n10.10.10.1"]
        RTK["<b>RTK Base</b>\n10.10.10.111:222"]
        ANT_T["<b>Antenna Tracker</b>\n10.10.10.5:14661"]

        subgraph GCS_B["<b>GCS 10.10.10.11</b>"]
            SERV2["<b>serverC</b>\nPID 20734"]
            DFA2["<b>DFA App</b>\nPID 20538\nlocalhost:8085 ✅"]
            MAVP["<b>MAVProxy</b>\nsystemd service\nPID 244012"]

            SERV2 -->|pipe stdin| DFA2
            SERV2 -->|localhost:8554| DFA2
        end

        subgraph FCC_B["<b>FCC 10.10.10.110</b>"]
            FCC2["<b>FCC 4.0.11</b>\nUDP:14550 ✅"]
        end

        MH_G -->|HTTP API :80| SERV2
        MH_G -->|UDP :14551| MAVP
        MAVP -->|UDP :14550| FCC2
        DFA2 -->|TCP :14661| ANT_T
        FCC2 -->|TCP :222| RTK
        SERV2 -->|RTSP :554| CAM_U
    end

    MH_U <-->|900 MHz RF| MH_G
    linkStyle 0 stroke:#008000,stroke-width:2px
    linkStyle 1 stroke:#008000,stroke-width:2px
    linkStyle 2 stroke:#008000,stroke-width:2px
    linkStyle 3 stroke:#008000,stroke-width:2px
    linkStyle 4 stroke:#008000,stroke-width:2px
    linkStyle 5 stroke:#008000,stroke-width:2px
    linkStyle 6 stroke:#008000,stroke-width:2px
    linkStyle 7 stroke:#008000,stroke-width:2px
    linkStyle 8 stroke:#008000,stroke-width:2px
    linkStyle 9 stroke:#008000,stroke-width:2px
```

### 12.2 Problem & Solution Summary
The root cause was that MAVLink telemetry was being consumed entirely within the GCS through the serverC to DFA pipe and never leaving the machine. The fix was running MAVProxy as a daemon listening on UDP port 14551 and forwarding everything to the FCC at 10.10.10.110 on port 14550, made permanent through a systemd service configured to start at boot and restart automatically if it ever crashed. The impact was immediate and complete, both the GCS and the FCC began receiving live telemetry simultaneously through their respective paths with no interference between them. The entire fix took approximately thirty seconds to execute once the correct command was confirmed, and it carried zero risk because nothing in the existing system was modified, no changes were made to DFA, serverC, the FCC configuration, or the Microhard radios. MAVProxy was simply added alongside the existing architecture as an independent forwarding layer.

| Item | Details |
|---|---|
| **Root cause** | MAVLink telemetry consumed internally by GCS (serverC -> DFA via pipe/localhost:8085) with no forwarding path to FCC |
| **Fix** | MAVProxy daemon listening on UDP:14551, forwarding to FCC at `10.10.10.110:14550` |
| **Persistence** | systemd service `mavproxy-fcc.service`; enabled, auto-restart on crash |
| **Impact** | Both DFA (GCS display) and FCC now receive live telemetry simultaneously |
| **Time to fix** | ~30 seconds once MAVProxy command confirmed correct |
| **Risk of fix** | Zero; non-invasive, no changes to DFA, serverC, FCC, or Microhard |

### 12.3 Recommendations
- The first is to verify the fix survives a reboot by restarting the GCS machine, checking that the MAVProxy service comes up automatically with an active running status, and confirming the FCC connects and loads parameters without any manual intervention.
- The second is to locate and document the DFA launcher. The script or mechanism that starts DFA, serverC, and creates the pipe between them was never found during the investigation. This is a reliability risk because if the system is rebooted or DFA crashes, whoever restarts it needs to know the correct procedure. Ideally the entire DFA startup chain should be converted into a systemd service just as MAVProxy was.
- The third is to disable ModemManager to permanently eliminate the warning it generates at every MAVProxy startup and remove any theoretical risk of serial port interference on the machine.
- The fourth is to configure log rotation for the MAVProxy log file at /tmp/mav.log. Because MAVProxy runs continuously as a permanent service it writes to this file indefinitely, and without rotation the file will grow until it fills the disk. The logrotate configuration shown rotates the log daily, keeps seven days of compressed history, and uses copytruncate to rotate the file without interrupting the running MAVProxy process.
- The fifth is to document the Microhard HTTP API that serverC uses to retrieve telemetry. The specific endpoints, authentication method, and data format of this API were not investigated and are currently undocumented. If serverC is ever replaced or the Microhard firmware is updated, the compatibility of this interface must be verified beforehand or the entire telemetry chain could break silently.
- The sixth is to report the RTK display bug to the FCC software vendor, where the user interface shows the wrong IP address for the RTK base station despite the underlying configuration being correct.
- The seventh is that if additional ground stations or a logging system are ever added to the network, MAVProxy can forward to multiple UDP destinations simultaneously by simply adding extra --out arguments to the service file, making it straightforward to expand the system without any architectural changes.<br/>

**1. Verify after reboot**
Restart the GCS machine and confirm:
```bash
sudo systemctl status mavproxy-fcc   # should show: active (running)
```
Then press CONNECT on FCC and verify parameters load.

**2. Document the DFA launcher**
The mechanism that starts DFA (which also starts serverC and creates the pipe) was not found during investigation. This should be located, documented, and ideally converted to a systemd service for reliability.

**3. Resolve ModemManager**
```bash
sudo systemctl disable ModemManager
sudo systemctl stop ModemManager
```

**4. MAVProxy log rotation**
`/tmp/mav.log` grows indefinitely. Add a logrotate entry:
```bash
sudo tee /etc/logrotate.d/mavproxy << 'EOF'
/tmp/mav.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

**5. Microhard HTTP API documentation**
The `serverC` binary queries the Microhard at `10.10.10.10:80` using its HTTP API to retrieve MAVLink telemetry. This API's endpoints, authentication, and data format are undocumented in this investigation. If serverC is ever replaced or the Microhard firmware updated, this API compatibility must be verified.

**6. RTK UI display bug**
FCC UI shows `10.10.10.4:222` for RTK Base, but config.xml and nmap both confirm the correct address is `10.10.10.111:222`. This is a cosmetic display bug in FCC — no functional impact, but should be reported to the FCC software vendor.

**7. Consider additional MAVProxy output**
If a third ground station or logging system is ever needed, MAVProxy can forward to multiple targets simultaneously:
```bash
ExecStart=... --out=udp:10.10.10.110:14550 --out=udp:10.10.10.xxx:14550
```

---
---

*Report compiled from three investigation sessions across GCS and FCC platforms. All commands, outputs, and findings documented verbatim from live system sessions. No configuration was changed on DFA, serverC, FCC, Microhard radios, Raspberry Pi, or any other system component. The only change made was the addition of the `mavproxy-fcc.service` systemd unit on the GCS.*

---

**End of Report**
