# Module 1: Fortinet FortiGate SD-WAN

An end-to-end SD-WAN fabric on FortiGate-VMs, going beyond basic connectivity to **intent-based routing, traffic shaping, and encrypted overlays**. Every change was verified with CLI diagnostics and packet captures, and tested by deliberately breaking the underlay.

**Headline result:** when ISP-A was failed on purpose, critical traffic moved to ISP-B with only 1-2 dropped pings before recovery.

---

## Lab Topology

![Topology](topology.png)

| Component | Role |
|-----------|------|
| HQ FortiGate-VM | Port1 LAN, Port2 MPLS/ISP-A, Port3 Broadband/ISP-B |
| Branch FortiGate-VM | Port1 LAN, Port2 MPLS/ISP-A, Port3 Broadband/ISP-B |
| Cisco vIOS routers | Two independent ISP backbones (underlay) |
| Alpine Linux hosts | iPerf3 server at HQ; TCPDump and client at Branch |

---

## Results Summary

| # | Scenario | Outcome | Folder |
|---|----------|---------|--------|
| 1 | SLA-based failover (active/standby) | 1-2 packets lost on failure, then immediate recovery | [Policy 1](Policy-1-SLA-Failover/) |
| 2 | Application-aware routing (FQDN) | SaaS traffic steered by domain name; path change confirmed after policy edit | [Policy 2](Policy-2-Application-Aware/) |
| 3 | Active/active load balancing | Sessions egress both links simultaneously (PCAP on each port) | [Policy 3](Policy-3-Load-Balancing/) |
| 4 | Traffic shaping and QoS | Bulk transfer held to a hard 2 Mbps cap | [Policy 4](Policy-4-Traffic-Shaping-QoS/) |
| 5 | IPsec overlay on SD-WAN zone | ESP-encrypted traffic; tunnel stays up through WAN failover | [Policy 5](Policy-5-IPsec-Overlay/) |

---

## Task Details

### Task 1: SLA-Based Performance Failover

**Objective:** Detect a degraded link and move critical traffic automatically.

**Approach:** Each WAN link has its own ICMP health check pointed at that link's **next-hop gateway** (`10.0.1.1` for ISP-A, `10.0.2.1` for ISP-B), not a public IP. This gives accurate per-link state. Failure was injected with an ACL on ISP-A while the interface stayed up, so only the SLA check could detect it.

**Evidence**
- Baseline: `TESTING POLICY 1 get diagnose sys ssdwan member baseline.png`
- Failure detection: `diagnose sys sdwan health-check during failure.png`
- Continuous ping showing 1-2 lost packets: `POLICY 1 continuours ping.png`
- Traceroute before and during failure, plus recovery screenshots
- Packet-level and video proof: `sd-wan-test-1.pcap`, `sd-wan-test-2026-10-26_02.06.16.mp4`

### Task 2: Application-Aware Routing (FQDN)

**Objective:** Steer critical SaaS traffic by domain name instead of static IPs.

**Approach:** An FQDN address object feeds an SD-WAN rule. A strict `cache-ttl` makes the FortiGate pick up DNS changes without manual intervention. An intentional policy change moved traffic from Port2 to Port3 to confirm the rule is what controls the path.

**Evidence**
- FQDN object: `fortinate show firewall address SAAS object.png`
- DNS resolution: `Application aware test DNS server nslookup server itself.png`, `NS LOOKUP HQ NODE.png`
- SD-WAN rule: `Fortinet show system sdwan policy 2.png`
- Path change: `POLICY 2 switching to port 3.png`, `POLICy CHANGE SUCCESSFUL PC is prefering port 3 after changing .png`

### Task 3: Active/Active Load Balancing

**Objective:** Use both WAN links at once to raise aggregate bandwidth.

**Approach:** Three configurations were tested: source/destination-IP hashing with a single stream, the same with two streams, and round-robin. With hashing, a single flow always lands on one link, so a second client IP was needed to see both links in use. Round-robin spreads sessions regardless.

> Note: in `mode auto`, listing `priority-members` produced round-robin distribution rather than strict failover.

**Evidence**
- Configuration: `ROUND ROBIN TEST SINGLE STREAM CONFIGURATION ON FORTINET.png`
- Hash-based tests: `single-stream-test-port-2/3.pcapng`, `double-stream-test-port-2/3.pcapng`
- Round-robin egress on both ports: `round-robin-test-port-2.pcapng`, `round-robin-test-port-3.pcapng`
- Video: `sd-wan-test-round-robin-single-stream-2026-10-26_09.40.45.mp4`

### Task 4: Traffic Shaping and QoS

**Objective:** Stop bulk transfers from starving critical applications.

**Approach:** Path selection (`config system sdwan`) and bandwidth enforcement (`config firewall shaping-policy`) are kept as separate objects, so each can be changed independently.

**Evidence**
- Shaper and policy config: `show firewall shaper traffic-shaper .png`, `show firewall shaping-policy.png`
- Wireshark IO graph showing the cap: `wirehsark io shows shaping being successful.png`
- Video: `qos-shaping-test-2026-10-27_00.41.33.mp4`

### Task 5: IPsec Overlay over the SD-WAN Zone

**Objective:** Encrypt branch-to-branch traffic while keeping SD-WAN path selection and failover.

**Approach:** The IPsec Phase 1 interface is bound to the `virtual-wan-link` zone rather than a single physical port. ESP packets in the capture show the traffic is encrypted, and the tunnel stayed `UP` when a physical WAN link failed.

**Evidence**
- Tunnel state and encrypted counters: `diagnose vpn tunnel list 1st VPN encrypted packets going up after ping.png`
- IKE/IPsec status on both ends: `diagnose vpn ike gateway list HQ.png` / `BRANCH.png`, `diagnose vpn ipsec status HQ .png` / `BRANCH.png`
- ESP proof: `icmp and esp trailing proof of encryption.png`, `ESP PCAPs found overview.png`
- Routing: `get router info routing-table all HQ.png`, `get router info routing-table all Branch .png`
- Video: `sd-wan-esp-pcap-2026-10-27_04.58.36.mp4`

---

## Troubleshooting and Lessons Learned

| Problem | Cause | Fix |
|---------|-------|-----|
| Traffic stayed on one link after switching from active/standby to active/active | Existing sessions kept their old path | `diagnose sys session clear` so flows are re-evaluated against the new hash |
| iPerf3 dropped its control socket on high-bandwidth tests | Virtual switch MTU/CPU limits in the lab | Used continuous `ping` and rate-limited iPerf3 (`-b 1M`) for clean captures |
| Health states were inaccurate when both links probed 8.8.8.8 | A shared public target does not reflect each link's own state | Probe each ISP's next-hop gateway |

---

## Runbooks

Each policy folder includes a runbook. The main one is the [Policy 1 runbook](Policy-1-SLA-Failover/fortinet-runbook), an ITIL-aligned SOP covering SD-WAN troubleshooting, health-check validation, and maintenance windows.

---

[Back to repo overview](../README.md) · [Next: Module 2, Cisco Catalyst SD-WAN](../02-Cisco-Catalyst-SDWAN/)
