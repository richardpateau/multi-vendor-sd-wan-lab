# Policy 1: Priority-Based Performance Failover (Active / Standby)

## Executive Summary
This module validates Fortinet FortiOS SD-WAN **intent-based, performance-driven link steering**. Rather than relying on layer-1/layer-2 physical link state (carrier detect), FortiGate monitors real-time path quality and next-hop reachability over the underlay WAN links.

When the primary MPLS link (`ISP-A` / `port2`) experiences a routing blackhole or upstream failure (simulated via an ingress ACL blocking health probes on the ISP router), the FortiGate detects the failed health-check and automatically steers branch-bound traffic to the secondary Broadband link (`ISP-B` / `port3`) with zero manual intervention and sub-second convergence.

---

## Lab Architecture & Addressing

```
[ HQ Server ]
  192.168.1.10/24
        │
    (port1: 192.168.1.1)
  [ FortiGate HQ ]
    ├── port2 (10.0.1.2/30) ──> [ ISP-A: MPLS ] (10.0.1.1) ──> (10.0.1.6) port2 ──┐
    │                                                                               [ FortiGate Branch ] ── (port1: 192.168.2.1) ── [ Branch Client ]
    └── port3 (10.0.2.2/30) ──> [ ISP-B: B-Band ] (10.0.2.1) ──> (10.0.2.6) port3 ──┘                                               192.168.2.10/24
```

### Underlay & Subnet Matrix
| Device / Role | Interface | IP Address | Subnet Mask | Next-Hop / Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **HQ FortiGate** | `port1` (LAN) | `192.168.1.1` | `255.255.255.0` | — |
| **HQ FortiGate** | `port2` (WAN1 - ISP-A MPLS) | `10.0.1.2` | `255.255.255.252` | `10.0.1.1` |
| **HQ FortiGate** | `port3` (WAN2 - ISP-B Broadband) | `10.0.2.2` | `255.255.255.252` | `10.0.2.1` |
| **Branch FortiGate** | `port1` (LAN) | `192.168.2.1` | `255.255.255.0` | — |
| **Branch FortiGate** | `port2` (WAN1) | `10.0.1.6` | `255.255.255.252` | `10.0.1.5` |
| **Branch FortiGate** | `port3` (WAN2) | `10.0.2.6` | `255.255.255.252` | `10.0.2.5` |
| **HQ Server** | `eth0` | `192.168.1.10` | `255.255.255.0` | `192.168.1.1` |
| **Branch Client** | `eth0` | `192.168.2.10` | `255.255.255.0` | `192.168.2.1` |

---

## Configuration & Engineering Logic

### 1. Dedicated Underlay Health Monitors
To prevent false-positive link fails caused by transient Internet-wide outages, dedicated link-level monitors probe the **immediate provider next-hop** of each physical interface:
* **`ISP_A_Monitor`**: Sends ICMP probes out `port2` targeting `10.0.1.1`.
* **`ISP_B_Monitor`**: Sends ICMP probes out `port3` targeting `10.0.2.1`.

```fortios
config system sdwan
    config health-check
        edit "ISP_A_Monitor"
            set server "10.0.1.1"
            set protocol ping
            set members 1
            set interval 500
            set failtime 3
            set recoverytime 3
        next
        edit "ISP_B_Monitor"
            set server "10.0.2.1"
            set protocol ping
            set members 2
            set interval 500
            set failtime 3
            set recoverytime 3
        next
    end
end
```

### 2. SD-WAN Service Rule (Priority Steering)
* **Steering Strategy:** `mode priority` (Active/Standby).
* **Preference Hierarchy:** Member 1 (`port2` - MPLS) > Member 2 (`port3` - Broadband).
* **Failover Logic:** As long as `ISP_A_Monitor` passes its health-check, all traffic to `Branch-Subnet` (`192.168.2.0/24`) is pinned to `port2`. If `ISP_A_Monitor` fails 3 consecutive intervals, the service rule immediately selects Member 2 (`port3`).

```fortios
config system sdwan
    config service
        edit 1
            set name "HQ-to-Branch"
            set mode priority
            set dst "Branch-Subnet"
            set src "all"
            set health-check "ISP_A_Monitor" "ISP_B_Monitor"
            set priority-members 1 2
            set status enable
            set default enable
        next
    end
end
```

### 3. Static Routing Integration
```fortios
config router static
    edit 1
        set distance 1
        set sdwan-zone "virtual-wan-link"
    next
    edit 2
        set dst 192.168.2.0 255.255.255.0
        set distance 1
        set sdwan-zone "virtual-wan-link"
    next
end
```

---

## Test Execution & Verification Lifecycle

### Phase 1: Baseline Verification (Primary Path Healthy)
**Objective:** Confirm both underlay links are passing health checks and outbound traffic exits through `port2` (ISP-A MPLS).

1. **Verify SD-WAN Health-Check Quality:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan health-check
   Health Check(ISP_A_Monitor):
   Seq(1 port2): state(alive), packet-loss(0.000%) latency(8.663), jitter(2.876)
   Health Check(ISP_B_Monitor):
   Seq(2 port3): state(alive), packet-loss(0.000%) latency(10.048), jitter(1.689)
   ```

2. **Verify Active Member Steering:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan member
   Member(1): interface: port2, flags=0x0 , gateway: 10.0.1.1, priority: 1 1024, weight: 0
   Member(2): interface: port3, flags=0x0 , gateway: 10.0.2.1, priority: 1 1024, weight: 0
   ```

3. **Verify Baseline Traceroute (HQ Server -> Branch Client):**
   ```bash
   localhost:~# traceroute 192.168.2.10
   traceroute to 192.168.2.10 (192.168.2.10), 30 hops max, 46 byte packets
    1  192.168.1.1 (192.168.1.1)    2.863 ms
    2  10.0.1.1    (10.0.1.1)      12.805 ms   <-- Egressing via ISP-A (MPLS)
    3  10.0.1.6    (10.0.1.6)      15.393 ms   <-- Branch WAN1 Interface
    4  192.168.2.10 (192.168.2.10) 14.918 ms
   ```

---

### Phase 2: Simulating Failure (Upstream ACL Blackhole)
**Objective:** Simulate an underlay ISP failure without physical interface down-state by denying FortiGate probe packets on `ISP-1-MPLS`.

```cisco
ISP-1-MPLS(config)# ip access-list extended BLOCK-SDWAN-PROBE
ISP-1-MPLS(config-ext-nacl)# 10 deny icmp host 10.0.1.2 host 10.0.1.1 echo
ISP-1-MPLS(config-ext-nacl)# 20 permit ip any any
ISP-1-MPLS(config-ext-nacl)# exit
ISP-1-MPLS(config)# interface FastEthernet0/0
ISP-1-MPLS(config-if)# ip access-group BLOCK-SDWAN-PROBE in
```

* **Physical State Check:** `FastEthernet0/0` stays **UP / UP**, proving traditional static routing with interface-tracking would fail to detect this failure.
* **SD-WAN Service Re-Evaluation:**
  ```text
  FortiGate-VM64-KVM # diagnose sys sdwan service
  Service(1): Address Mode(IPV4) flags=0x240 use-shortcut-sla
  Tie break: cfg
  Gen(62), Mode(priority), health-check(ISP_A_Monitor ISP_B_Monitor)
  Members(2):
    1: Seq_num(2 port3), alive, latency: 8.960, selected  <-- PROMOTED TO ACTIVE
    2: Seq_num(1 port2), dead                             <-- FLAGGED DEAD
  ```

---

### Phase 3: Traffic Failover & Data Plane Continuity
**Objective:** Confirm active data plane re-route and packet loss during failover.

1. **Traceroute via Backup Path (`ISP-B` Broadband):**
   ```bash
   localhost:~# traceroute 192.168.2.10
   traceroute to 192.168.2.10 (192.168.2.10), 30 hops max, 46 byte packets
    1  192.168.1.1 (192.168.1.1)    2.628 ms
    2  10.0.2.1    (10.0.2.1)      17.510 ms   <-- Shifted to ISP-B (Broadband)
    3  10.0.2.6    (10.0.2.6)      24.376 ms   <-- Branch WAN2 Interface
    4  192.168.2.10 (192.168.2.10) 24.905 ms
   ```

2. **Continuous ICMP Stream Analysis:**
   Continuous ICMP pings remained uninterrupted across the transition with sub-second failover (0% packet drop observed once next-hop reconverged).

---

### Phase 4: Recovery (Restoration of Preferred Path)
**Objective:** Remove the ACL blackhole on `ISP-1-MPLS` and observe automatic failback.

1. **Remove ACL on ISP Router:**
   ```cisco
   ISP-1-MPLS(config)# interface FastEthernet0/0
   ISP-1-MPLS(config-if)# no ip access-group BLOCK-SDWAN-PROBE in
   ```

2. **SD-WAN Service Re-selection:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan service
   Service(1): Address Mode(IPV4) flags=0x240 use-shortcut-sla
   Tie break: cfg
   Gen(63), Mode(priority), health-check(ISP_A_Monitor ISP_B_Monitor)
   Members(2):
     1: Seq_num(1 port2), alive, latency: 10.055, selected  <-- RESTORED PRIMARY
     2: Seq_num(2 port3), alive, latency: 9.521, selected
   ```
   *Traffic immediately preempts back to Priority 1 (`port2`) cleanly.*

---

## Evidence Index

| Phase | Artifact / Screenshot | Purpose & Verification |
| :--- | :--- | :--- |
| **Phase 1: Baseline** | `show system sdwan baseline.png` | Confirms SD-WAN zone configuration, member mappings, and probe timers. |
| **Phase 1: Baseline** | `show router static baseline.png` | Validates default route bound to the `virtual-wan-link` interface. |
| **Phase 1: Baseline** | `diagnose sys sdwan member baseline.png` | Confirms initial member priority assignments (`port2` vs `port3`). |
| **Phase 1: Baseline** | `diagnose sys sdwan health-check baseline.png` | Baseline latency, jitter, and 0% loss to both provider next-hops. |
| **Phase 1: Baseline** | `traceroute baseline.png` | Baseline validation that egress traffic exits `10.0.1.1` (MPLS). |
| **Phase 2: Link Failure** | `ISP-A ACL block.png` | Cisco ACL configuration dropping ICMP echo requests from `10.0.1.2`. |
| **Phase 2: Link Failure** | `diagnose sys sdwan health-check during failure.png` | Detects `ISP_A_Monitor` dead state while physical interface remains up. |
| **Phase 2: Link Failure** | `diagnose sys sdwan service during failure.png` | Verifies dynamic promotion of `port3` as the only active selected member. |
| **Phase 3: Impact** | `traceroute during failover.png` | Proves live data plane reroute through `10.0.2.1` (`ISP-B`). |
| **Phase 3: Impact** | `continuous ping failover.png` | Evidence of business continuity and zero session breakdown. |
| **Phase 4: Recovery** | `ISP-A ACL removal.png` | Administrative removal of the drop ACL on `ISP-1-MPLS`. |
| **Phase 4: Recovery** | `diagnose sys sdwan service recovered.png` | Automatic preemption back to `port2` (MPLS Priority 1). |

---

## Operational Runbook & SOPs
* **[Operational Runbook](./fortinet-runbook.txt)** — Contains step-by-step troubleshooting workflows, packet sniffer commands (`diagnose sniffer packet any 'icmp' 4 0 l`), session table verification, and clean rollback scripts.

---

## Business Value Delivered
* **Autonomous High Availability:** Eliminates SLA penalties by automatically rerouting branch traffic around silent upstream ISP brownouts without operational intervention.
* **Carrier Cost Optimization:** Enables secure, cost-effective broadband links to serve as hot-standbys for expensive MPLS circuits.
* **Preserved User Experience:** Convergence occurs fast enough that real-time TCP applications (RDP, ERP, CRM) survive transient WAN failures without reconnecting.

---
*➡️ [Return to Fortinet Module Overview](../README.md)*
```
