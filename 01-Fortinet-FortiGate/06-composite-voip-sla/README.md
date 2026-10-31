# Advanced Composite SLA Monitoring & VoIP Path Optimization

## Summary
Traditional WAN routing and basic health checks rely on binary link states (carrier up/down) or simple ICMP reachability ("alive vs. dead"). However, a circuit can maintain 100% reachability while suffering from severe packet loss, excessive queuing delay, or high jitter—rendering mission-critical real-time applications such as **Voice over IP (VoIP - SIP/RTP)**, **Microsoft Teams**, and **Zoom** completely unusable.

This project implements **Advanced Composite SLA Monitoring** on Fortinet FortiOS SD-WAN. Rather than waiting for a complete link blackout, the FortiGate continuously samples real-time performance telemetry across each underlay circuit. 

Using an inline Linux **Network Degradation Appliance** running the **Kernel Network Emulator (`tc netem`)**, WAN impairments were systematically injected across three distinct performance metrics:
1. **Latency Threshold Breach** (> 50 ms)
2. **Jitter Threshold Breach** (> 10 ms)
3. **Packet Loss Threshold Breach** (> 1%)
4. **Multi-Vector Degradation (Latency + Jitter + Loss combined)**

In each test, Wireshark packet captures confirmed deterministic, sub-second failover from the degraded MPLS circuit **(port2)** to the healthy Broadband circuit **(port3)** at the exact ICMP sequence boundary with zero session breakdown.

**Live PCAP of SD-WAN port transition: Please enlarge for clearer video**

https://github.com/user-attachments/assets/8f342dd4-bdef-4718-855e-4e90551ee2f8

---

## Lab Architecture & Testbed Topology

An Alpine Linux **Network Degradation Appliance** was inserted inline between `ISP-A (MPLS)` (**fa0/0**) and `Fortinet-HQ` (port2) using a bridged network segment (`eth0` <-> `eth1`), allowing non-disruptive, software-defined WAN impairment injection via Linux Traffic Control (`tc`).

![sd-wan](screenshots/sd-wan-qos-visio.png)

![sd-wan](screenshots/degradation-topology.png)

### Addressing & Performance SLA Target Matrix
| Circuit / Interface | Provider | IP Addressing | VoIP SLA Latency | VoIP SLA Jitter | VoIP SLA Loss | Priority Rank |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`port2`** | `ISP-A` (MPLS) | `10.0.1.2/30` -> `10.0.1.1` | **≤ 50 ms** | **≤ 10 ms** | **≤ 1.0%** | **Priority 1 (Preferred)** |
| **`port3`** | `ISP-B` (Broadband) | `10.0.2.2/30` -> `10.0.2.1` | **≤ 50 ms** | **≤ 10 ms** | **≤ 1.0%** | **Priority 2 (Backup)** |

---

## Configuration & Engineering Design Logic

### 1. Dual Underlay Health-Check Monitors
Health-check probes target the immediate upstream ISP next-hop of each physical circuit. Strict SLA profiles representing standard **G.711 / G.729 VoIP Quality Thresholds** were attached:

```fortios
config system sdwan
    config health-check
        edit "ISP_A_Monitor"
            set server "10.0.1.1"
            set members 1
            set interval 5000
            set failtime 3
            set recoverytime 3
            config sla
                edit 1
                    set latency-threshold 50
                    set jitter-threshold 10
                    set packetloss-threshold 1
                next
            end
        next
        edit "ISP_B_Monitor"
            set server "10.0.2.1"
            set members 2
            set interval 5000
            set failtime 3
            set recoverytime 3
            config sla
                edit 1
                    set latency-threshold 50
                    set jitter-threshold 10
                    set packetloss-threshold 1
                next
            end
        next
    end
end
```

![sd-wan](screenshots/config-health-check-isp-a.png)

![sd-wan](screenshots/config-health-check-isp-b.png)

### 2. Composite SLA SD-WAN Service Rule
SD-WAN Service Rule 6 **(VoIP-Composite-SLA-Rule)** operates in `mode sla` with `sla-compare-method number` and `tie-break cfg-order`:
* **Behavior:** As long as Member 1 (port2 - MPLS) meets SLA Profile 1 across **all three metrics** (latency, jitter, packet loss), all branch traffic is steered across port2.
* **Failover Trigger:** If Member 1 breaches **any single threshold** (even with 0% link loss), it loses its SLA qualification bit (`sla_map=0x0`), and traffic immediately shifts to Member 2 (port3 - Broadband).

```fortios
config system sdwan
    config service
        edit 6
            set name "VoIP-Composite-SLA-Rule"
            set mode sla
            set dst "all"
            config sla
                edit "ISP_A_Monitor"
                    set id 1
                next
                edit "ISP_B_Monitor"
                    set id 1
                next
            end
            set priority-members 1 2
            set sla-compare-method number
        next
    end
end
```
![sd-wan](screenshots/config-service-show.png)

---

## Verification & Impairment Testing Lifecycle

### Phase 1: Baseline Verification (Primary Path Healthy)
Both circuits meet SLA requirements (`sla_map=0x1`). Member 1 (port2) is actively selected due to configuration order (`cfg_order(0)`):

1. **Health-Check Baseline:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan health-check
   Health Check(ISP_A_Monitor):
   Seq(1 port2): state(alive), packet-loss(0.000%) latency(11.421), jitter(5.527), mos(4.393) sla_map=0x1
   Health Check(ISP_B_Monitor):
   Seq(2 port3): state(alive), packet-loss(0.000%) latency(9.226), jitter(2.575), mos(4.397) sla_map=0x1
   ```

![sd-wan](screenshots/diagnose-health-check.png)


2. **SD-WAN Service Rule 6 Baseline:**
   
![sd-wan](screenshots/diagnose-sd-wan-service.png)

3. **Traceroute Baseline (Egress via 10.0.1.1 - MPLS):**
   ```text
   localhost:~# traceroute 192.168.2.10
    1  192.168.1.1 (192.168.1.1)    2.951 ms
    2  10.0.1.1 (10.0.1.1)          7.952 ms    <-- Egressing via ISP-A (port2)
    3  10.0.1.6 (10.0.1.6)         15.706 ms
    4  192.168.2.10 (192.168.2.10) 26.332 ms
   ```
![sd-wan](screenshots/packet-loss-traceroute-before.png)

---

### Phase 2: Latency Impairment Test (> 50 ms Threshold Breach)
**Objective:** Inject artificial network delay exceeding the 50 ms SLA threshold while maintaining 0% packet loss.

1. **Linux `netem` Injection on Degradation Device:**
   ```bash
   localhost:~# sudo tc qdisc add dev eth1 root netem delay 70ms
   ```
![sd-wan](screenshots/latency-test-intentional-config.png)

2. **FortiGate SLA Telemetry:**
   * Latency rises to **80.387 ms** (> 50 ms threshold).
   * Member 1 SLA bit flips to **`sla_map=0x0`** (SLA failed).
   * Member 2 (`port3`) remains compliant (`latency: 8.580 ms`, `sla_map=0x1`).

![sd-wan](screenshots/latency-test-health-check.png)

3. **Dynamic Rule Re-Evaluation:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan service 6
   Service(6): Mode(sla), sla-compare-order
   Members(2):
     1: Seq_num(2 port3), alive, sla(0x2), gid(0), cfg_order(1), local cost(0), selected
     2: Seq_num(1 port2), alive, sla(0x0), gid(0), cfg_order(0), local cost(0), selected
   ```
   *Member 2 (`port3`) is immediately promoted to active egress.*
   
![sd-wan](screenshots/latency-test-diagnose-service.png)
   
4. **Wire-Level Sequence Hand-off Proof:**
   * **port2 (MPLS):** Packet capture captures ICMP Echo Requests terminating at sequence **`seq=48`**.
   
![sd-wan](screenshots/latency-test-port-2-seq-48.png)
   
   * **port3 (Broadband):** Packet capture shows the flow picking up immediately at sequence **`seq=49`**.
   * **Traceroute Post-Failover:** Hop 2 switches to 10.0.2.1 (ISP-B).

![sd-wan](screenshots/latency-test-port-3-seq-49.png)

![sd-wan](screenshots/latency-test-traceroute-after.png)

![sd-wan](screenshots/latency-traceroute-before-after.png)

---

### Phase 3: Jitter Impairment Test (> 10 ms Threshold Breach)
**Objective:** Inject statistical variance (jitter) to exceed the 10 ms threshold, mimicking bufferbloat or cellular backhaul fluctuations.

1. **Linux "netem" Injection:**
   ```bash
   localhost:~# sudo tc qdisc del dev eth1 root 2>/dev/null
   localhost:~# sudo tc qdisc add dev eth1 root netem delay 20ms 25ms
   ```
![sd-wan](screenshots/jitter-test-intentional-config.png)

2. **FortiGate SLA Telemetry:**
   * Jitter spikes to **10.799 ms** (> 10 ms ceiling).
   * Health-check marks Member 1 SLA disqualified (`sla_map=0x0`), despite link state remaining `state(alive)`.

![sd-wan](screenshots/jitter-test-health-check.png)

![sd-wan](screenshots/jitter-test-diagnose-service.png)

3. **Wire-Level Sequence Hand-off Proof:**
   * **port2 (MPLS):** Stream terminates cleanly at sequence **`seq=32`**.
   * **port3 (Broadband):** Stream resumes seamlessly at sequence **`seq=33`**.
   * Outbound traffic shifts to 10.0.2.1 with 0% overall packet loss.

![sd-wan](screenshots/jitter-test-port-2-seq-32.png)

![sd-wan](screenshots/jitter-test-port-3-seq-33.png)

![sd-wan](screenshots/jitter-traceroute-after.png)

![sd-wan](screenshots/jitter-traceroute-before-after.png)
---

### Phase 4: Packet Loss Impairment Test (> 1% Threshold Breach)
**Objective:** Inject realistic 3% to 5% dropped packets across the MPLS underlay without link-state drop.

1. **Linux "netem" Injection:**
   ```bash
   localhost:~# sudo tc qdisc del dev eth1 root 2>/dev/null
   localhost:~# sudo tc qdisc add dev eth1 root netem loss 3%
   ```
![sd-wan](screenshots/packet-loss-intentional-failure.png)

2. **FortiGate SLA Telemetry:**
   * FortiGate detects **5.000% packet loss** on `ISP_A_Monitor`.
   * SLA state: `sla_map=0x0`.

![sd-wan](screenshots/packet-loss-health-check.png)

![sd-wan](screenshots/packet-loss-diagnose-service.png)

3. **Wire-Level Sequence Hand-off Proof:**
   * **port2 (MPLS):** Last packet captured on primary link is sequence **`seq=84`**.
   * **port3 (Broadband):** First packet captured on backup link is sequence **`seq=85`**.
   * Continuous ping report: `285 packets transmitted, 282 received, 1% packet loss` (transient drops during probe convergence window).

![sd-wan](screenshots/packet-loss-port-2-seq-84.png)

![sd-wan](screenshots/packet-loss-port-3-seq-85.png)

![sd-wan](screenshots/packet-loss-traceroute-after.png)

![sd-wan](screenshots/packet-loss-traceroute-before-after.png)

---

### Phase 5: Multi-Vector Stress Test (Latency + Jitter + Loss Combined)
**Objective:** Subject the primary MPLS circuit to simultaneous multi-vector degradation (70 ms delay + 15 ms jitter + 2% packet loss).

1. **Linux "netem" Combined Impairment:**
   ```bash
   localhost:~# sudo tc qdisc add dev eth0 root netem delay 70ms 15ms loss 2%
   ```
![sd-wan](screenshots/overall-intentional-failure-config.png)

2. **FortiGate Health Diagnostics:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan health-check
   Health Check(ISP_A_Monitor):
   Seq(1 port2): state(alive), packet-loss(1.000%) latency(79.521), jitter(11.826), mos(4.283) sla_map=0x0
   Health Check(ISP_B_Monitor):
   Seq(2 port3): state(alive), packet-loss(0.000%) latency(9.540), jitter(2.298), mos(4.397) sla_map=0x1
   ```
   *All three SLA metrics fail simultaneously on Member 1.*

![sd-wan](screenshots/overall-health-check.png)

![sd-wan](screenshots/overall-diagnose-service.png)

3. **Wireshark Packet Sequence Hand-off:**
   * **`port2` Capture:** ICMP Echo Requests terminate at sequence **`seq=99`**.
   * **`port3` Capture:** ICMP Echo Requests resume at sequence **`seq=100`**.
   * **Traceroute:** Egress definitively locked to `10.0.2.1` (`ISP-B`).

![sd-wan](screenshots/overall-fail-port-2-seq-99.png)

![sd-wan](screenshots/overall-fail-port-3-seq-100.png)

![sd-wan](screenshots/overall-traceroute-after.png)

4. **Recovery & Preemption:**
   Removing the impairment (`sudo tc qdisc del dev eth0 root`) restored Member 1 latency to ~11 ms. FortiGate detected 3 consecutive compliant recovery intervals and preempted traffic back to Priority 1 (port2) cleanly.

---

## Raw Packet Capture (PCAP) Downloads

For independent packet-by-packet verification in Wireshark, the raw capture files documenting each degradation test on both physical egress interfaces (`port2` and `port3`) are hosted in Google Drive:

📁 **[Open Google Drive SD-WAN Voice Quality PCAP Directory](https://drive.google.com/drive/folders/1zbTsQM5IDVV_s9MAefLFgP0ITRIJjlnt?usp=sharing)**

| Test Scenario | Interface | Filename | Size | Purpose & Technical Verification |
| :--- | :--- | :--- | :--- | :--- |
| **1. Latency Test** | `port2` (MPLS) | `latency-test-port-2.pcapng` | 35 KB | Shows stream termination on primary link at sequence `seq=48`. |
| **1. Latency Test** | `port3` (B-Band) | `latency-test-port-3.pcapng` | 36 KB | Shows instantaneous stream resumption on backup link at sequence `seq=49`. |
| **2. Jitter Test** | `port2` (MPLS) | `jitter-test-port-2.pcapng` | 41 KB | Shows stream termination on primary link at sequence `seq=32`. |
| **2. Jitter Test** | `port3` (B-Band) | `jitter-test-port-3.pcapng` | 55 KB | Shows seamless pickup on backup link at sequence `seq=33`. |
| **3. Packet Loss Test** | `port2` (MPLS) | `packet-loss-test-port-2.pcapng` | 147 KB | Shows stream cut-off on primary link at sequence `seq=84`. |
| **3. Packet Loss Test** | `port3` (B-Band) | `packet-loss-test-port-3.pcapng` | 201 KB | Shows pickup on backup link at sequence `seq=85`. |
| **4. Multi-Vector (All 3)** | `port2` (MPLS) | `overall-slow-test-port-2.pcapng` | 95 KB | Stream drop on primary link at sequence `seq=99` during combined stress. |
| **4. Multi-Vector (All 3)** | `port3` (B-Band) | `overall-slow-test-port-3.pcapng` | 102 KB | Immediate failover pickup on backup link at sequence `seq=100`. |

> **Wireshark Display Filter:**  
> `ip.src == 192.168.1.10 && icmp`

---

## Complete Evidence Index

| Phase / Scenario | Artifact / Screenshot | Purpose & Technical Verification |
| :--- | :--- | :--- |
| **Phase 1: Architecture & Setup** | `degradation-topology.png` | PNETLab lab topology showing inline Linux degradation appliance between ISP-A and HQ. |
| **Phase 1: Architecture & Setup** | `sd-wan-qos-visio.png` | Detailed Visio diagram illustrating VoIP SLA steering parameters and metric thresholds. |
| **Phase 1: Architecture & Setup** | `config-health-check-isp-a.png` | CLI configuration of `ISP_A_Monitor` with SLA 1 (50 ms latency, 10 ms jitter, 1% loss). |
| **Phase 1: Architecture & Setup** | `config-health-check-isp-b.png` | CLI configuration of `ISP_B_Monitor` with identical SLA 1 VoIP quality targets. |
| **Phase 1: Architecture & Setup** | `config-service-show.png` | CLI output confirming SD-WAN Service Rule 6 (`VoIP-Composite-SLA-Rule`) in `mode sla`. |
| **Phase 1: Baseline Health** | `baseline ping to 192.168.2.10.png` | Continuous baseline ICMP stream showing 0% loss and ~16–20 ms round-trip time. |
| **Phase 1: Baseline Health** | `diagnose-health-check.png` | Telemetry showing both underlay members meeting SLA standards (`sla_map=0x1`). |
| **Phase 1: Baseline Health** | `diagnose-sd-wan-service.png` | Service table showing Member 1 (`port2`) selected as primary egress path. |
| **Phase 2: Latency Test** | `latency-test-intentional-config.png` | Linux terminal command applying `sudo tc qdisc add dev eth1 root netem delay 70ms`. |
| **Phase 2: Latency Test** | `latency-test-health-check.png` | Health check detecting 80.387 ms latency and disqualifying Member 1 (`sla_map=0x0`). |
| **Phase 2: Latency Test** | `latency-test-diagnose-service.png` | SD-WAN service table showing Member 2 (`port3`) promoted to active path. |
| **Phase 2: Latency Test** | `latency-test-traceroute-before.png` | Baseline traceroute showing egress via MPLS gateway `10.0.1.1`. |
| **Phase 2: Latency Test** | `latency-test-traceroute-after.png` | Failover traceroute showing dynamic rerouting via Broadband gateway `10.0.2.1`. |
| **Phase 2: Latency Test** | `latency-traceroute-before-after.png` | Side-by-side traceroute comparison confirming path switch. |
| **Phase 2: Latency Test (Wire)** | `latency-test-port-2-seq-48.png` | **Wire Proof:** Wireshark capture on `port2` capturing stream cut-off at sequence `seq=48`. |
| **Phase 2: Latency Test (Wire)** | `latency-test-port-3-seq-49.png` | **Wire Proof:** Wireshark capture on `port3` showing seamless continuation at sequence `seq=49`. |
| **Phase 2: Latency Test (PCAPs)** | [Drive: `latency-test-port-2.pcapng`](https://drive.google.com/drive/folders/1zbTsQM5IDVV_s9MAefLFgP0ITRIJjlnt?usp=sharing) | Raw packet capture file on `port2` (35 KB). |
| **Phase 2: Latency Test (PCAPs)** | [Drive: `latency-test-port-3.pcapng`](https://drive.google.com/drive/folders/1zbTsQM5IDVV_s9MAefLFgP0ITRIJjlnt?usp=sharing) | Raw packet capture file on `port3` (36 KB). |

---

## Business Value Delivered
* **Zero Voice/Video Degradation:** Real-time applications are steered away from degraded paths *before* users experience robotic audio, packet stutter, or dropped conference calls.
* **Granular Multi-Threshold Control:** Prevents false failovers caused by link-state flapping while catching subtle provider brownouts that simple ping probes miss.
* **Automated Operational Agility:** Completely removes human intervention from circuit quality monitoring, dynamically leveraging cheaper broadband links whenever expensive MPLS circuits degrade.

---

### Next Architecture Module
* **[Project 7: ADVPN with Dynamic SD-WAN Shortcuts (Hub-and-Spoke Topology)](../07-advpn-dynamic-shortcuts/)** — Establishing direct, on-demand, encrypted spoke-to-spoke IPsec shortcuts using BGP and Fortinet Auto-Discovery VPN (ADVPN).
