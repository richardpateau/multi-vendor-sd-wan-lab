# ️ Fortinet FortiGate SD-WAN Implementation

## Summary
This module documents the end-to-end deployment, validation, and troubleshooting of a Fortinet FortiGate SD-WAN fabric. The implementation moves beyond basic connectivity to enforce **intent-based routing, strict Quality of Service (QoS), and secure encrypted overlays**. 

Every configuration change was validated using CLI packet sniffing, Wireshark PCAP analysis, and intentional underlay failure testing to prove sub-second convergence and business continuity.

---

## Lab Topology
*   **HQ Site:** FortiGate-VM (Port1: LAN, Port2: MPLS/ISP-A, Port3: Broadband/ISP-B)
*   **Branch Site:** FortiGate-VM (Port1: LAN, Port2: MPLS/ISP-A, Port3: Broadband/ISP-B)
*   **Underlay:** Cisco vIOS routers simulating independent ISP backbones.
*   **Endpoints:** Alpine Linux (iPerf3 Server at HQ, TCPDump/Client at Branch).

---

## Task Breakdown & Evidence Matrix

### Task 1: SLA-Based Performance Failover (Active/Standby)
**Objective:** Ensure business continuity by monitoring underlay health and automatically failing over critical traffic when a link degrades.
*   **Engineering Discovery:** Configured distinct ICMP health checks probing the **specific next-hop gateway** of each WAN link (`10.0.1.1` for ISP-A, `10.0.2.1` for ISP-B) rather than a public IP. This ensures accurate, per-link underlay state detection.
*   **Key Evidence:**
    *   `TESTING POLICY 1 get diagnose sys ssdwan member baseline.png` (Baseline health checks passing).
    *   `POLICY 1 continuours ping.png` (The "Money Shot": shows exactly 1-2 dropped packets during link failure, followed by immediate recovery).
    *   `diagnose sys sdwan health-check during failure.png` (Health check correctly detecting ISP-A gateway unreachable).
    *   `sd-wan-test-1.pcap` & `sd-wan-test-2026-10-26_02.06.16.mp4` (Packet-level and video proof of failover).

### Task 2: Application-Aware Routing (FQDN)
**Objective:** Dynamically steer critical SaaS traffic over the optimal WAN path based on domain names rather than static IPs.
*   **Engineering Discovery:** Implemented FQDN address objects with a strict `cache-ttl` to ensure the FortiGate dynamically resolves DNS changes without requiring policy reboots.
*   **Key Evidence:**
    *   `fortinate show firewall address SAAS object.png` (FQDN object creation).
    *   `Application aware test DNS server nslookup server itself.png` (Proving DNS resolution).
    *   `Fortinet show system sdwan policy 2.png` (SD-WAN rule matching the FQDN object).

### Task 3: Active/Active Load Balancing (Round-Robin)
**Objective:** Maximize aggregate WAN bandwidth by distributing sessions across both links simultaneously.
*   **Engineering Discovery:** Discovered that in `mode auto`, using `priority-members` triggers a **Round-Robin** distribution rather than strict Active/Passive failover. Bypassed Fortinet's default Source-IP hashing by assigning a secondary IP to the client to force dual-stream distribution.
*   **Key Evidence:**
    *   `ROUND ROBIN TEST SINGLE STREAM CONFIGURATION ON FORTINET.png` (Configuration proof).
    *   `round-robin-test-port-2.pcapng` & `round-robin-test-port-3.pcapng` (Simultaneous egress proof).
    *   `sd-wan-test-round-robin-single-stream-2026-10-26_09.40.45.mp4` (Video proof of session distribution).

### Task 4: SD-WAN Traffic Shaping & QoS
**Objective:** Enforce strict bandwidth limits to prevent bulk data transfers from choking critical applications.
*   **Engineering Discovery:** Separated the SD-WAN steering rule (`config system sdwan`) from the bandwidth enforcement rule (`config firewall shaping-policy`). This follows enterprise best practices for granular traffic control.
*   **Key Evidence:**
    *   `show firewall shaping-policy.png` (Proving the 2000 Kbps limit is configured).
    *   `wirehsark io shows shaping being successful.png` (Wireshark IO graph showing the hard cap at exactly 2 Mbps).
    *   `qos-shaping-test-2026-10-27_00.41.33.mp4` (Video proof of traffic throttling).

### Task 5: IPsec Overlay Tunnel over SD-WAN Zone
**Objective:** Secure branch-to-branch traffic with IPsec encryption while retaining dynamic SD-WAN path selection and failover.
*   **Engineering Discovery:** Bound the IPsec Phase 1 interface directly to the `virtual-wan-link` zone instead of a physical interface. Captured ESP (Encapsulating Security Payload) packets to prove encryption, and verified the tunnel remains `UP` during physical WAN failover.
*   **Key Evidence:**
    *   `diagnose vpn tunnel list 1st VPN encrypted packets going up after ping.png` (Tunnel status and encrypted packet counters).
    *   `icmp and esp trailing proof of encryption.png` (PCAP proving traffic is encrypted via ESP).
    *   `get router info routing-table all HQ.png` (Routing table showing the tunnel interface).

---

## Operational Documentation
*   **[fortinet-runbook](Policy-1-SLA-Failover/fortinet-runbook)** - ITIL-aligned Standard Operating Procedure for SD-WAN troubleshooting, health-check validation, and maintenance windows.
