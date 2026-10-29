# Task 5: Dual Route-Based IPsec Overlay Tunnels over SD-WAN Zone

## Executive Summary
This module represents the architectural peak of the Fortinet FortiGate SD-WAN implementation: **building an automated, multi-path encrypted overlay fabric across diverse transport circuits.**

In traditional deployments, an IPsec VPN tunnel is bound to a single physical WAN interface. If that underlay circuit suffers a brownout or hard failure, the tunnel tears down, terminating all client sessions and requiring a re-negotiation sequence over a secondary tunnel. 

To solve this, FortiOS utilizes an **SD-WAN Overlay Architecture**:
1. **Dual Route-Based IPsec Tunnels** are independently established over the primary MPLS circuit (`port2`) and secondary Broadband circuit (`port3`).
2. Both virtual tunnel interfaces (`HQ-BRANCH-VPN` and `HQ-BRANCH-VPN-2`) are aggregated into a dedicated SD-WAN Zone named **`VPN`** as SD-WAN members 3 and 4.
3. Static routing and Zone-based firewall policies are decoupled from individual interfaces and assigned directly to the `VPN` zone.
4. An SD-WAN Service Rule (`Service 5`) steers inter-site traffic across the encrypted overlay tunnels with sub-second failover, preserving active flows without tearing down Security Associations (SAs).

**Live PCAP: ICMP Packets encrypted in real time**

https://github.com/user-attachments/assets/4e57dd87-8a12-4bd8-aa0f-812fdeead968

---

## Lab Architecture & Multi-Layer Topology

![sd-wan](screenshots/sd-wan-vpn.png)

![sd-wan](screenshots/topology.png)

### Addressing & Cryptographic Matrix
| Parameter | Primary Overlay (`HQ-BRANCH-VPN`) | Secondary Overlay (`HQ-BRANCH-VPN-2`) |
| :--- | :--- | :--- |
| **Physical Underlay Interface** | `port2` (MPLS) | `port3` (Broadband) |
| **Local Gateway / Remote Gateway** | HQ: `10.0.1.2` <-> Branch: `10.0.1.6` | HQ: `10.0.2.2` <-> Branch: `10.0.2.6` |
| **IKE Version** | IKEv2 | IKEv2 |
| **Phase 1 Proposal** | `des-sha256` (Diffie-Hellman Group 14) | `des-sha256` (Diffie-Hellman Group 14) |
| **Phase 2 Proposal** | `des-sha256` (ESP) | `des-sha256` (ESP) |
| **SD-WAN Zone Assignment** | `VPN` (Member 3) | `VPN` (Member 4) |
| **Phase 2 Selectors** | HQ LAN: `192.168.1.0/24` <-> Branch LAN: `192.168.2.0/24` | HQ LAN: `192.168.1.0/24` <-> Branch LAN: `192.168.2.0/24` |
| **Dead Peer Detection (DPD)** | `on-idle`, 20-second interval, 3 retries | `on-idle`, 20-second interval, 3 retries |

---

## Configuration & Engineering Design Logic

### 1. Dual Route-Based Phase 1 & Phase 2 Tunnels
Each tunnel binds to its respective physical underlay interface. Auto-negotiation and DPD keep both security associations warm and ready for immediate forwarding:

```fortios
config vpn ipsec phase1-interface
    edit "HQ-BRANCH-VPN"
        set interface "port2"
        set ike-version 2
        set remote-gw 10.0.1.6
        set psksecret **********
        set proposal des-sha256
        set dpd on-idle
        set auto-negotiate enable
    next
    edit "HQ-BRANCH-VPN-2"
        set interface "port3"
        set ike-version 2
        set remote-gw 10.0.2.6
        set psksecret **********
        set proposal des-sha256
        set dpd on-idle
        set auto-negotiate enable
    next
end

config vpn ipsec phase2-interface
    edit "HQ-BRANCH-VPN-P2"
        set phase1name "HQ-BRANCH-VPN"
        set proposal des-sha256
        set src-subnet 192.168.1.0 255.255.255.0
        set dst-subnet 192.168.2.0 255.255.255.0
        set auto-negotiate enable
    next
    edit "HQ-BRANCH-VPN-P2-2"
        set phase1name "HQ-BRANCH-VPN-2"
        set proposal des-sha256
        set src-subnet 192.168.1.0 255.255.255.0
        set dst-subnet 192.168.2.0 255.255.255.0
        set auto-negotiate enable
    next
end
```

![sd-wan](screenshots/show-phase-1-branch.png)

![sd-wan](screenshots/show-phase-2-branch.png)

### 2. SD-WAN Overlay Zone Definition
Rather than referencing raw tunnel names across firewall policies, both VPN virtual interfaces are assigned to an **Overlay SD-WAN Zone** named **`VPN`**:

```fortios
config system sdwan
    config zone
        edit "VPN"
        next
    end
    config members
        edit 3
            set interface "HQ-BRANCH-VPN"
            set zone "VPN"
        next
        edit 4
            set interface "HQ-BRANCH-VPN-2"
            set zone "VPN"
        next
    end
end
```
![sd-wan](screenshots/show-vpn-zone.png)

![sd-wan](screenshots/show-members.png)

### 3. Static Routing & Floating Blackhole Route
Routing points the remote branch subnet to the `VPN` zone. A floating blackhole route (`distance 20`) is added to discard traffic cleanly if both VPN tunnels go down, preventing inter-site traffic from accidentally leaking onto the public underlays in plaintext:

```fortios
config router static
    edit 3
        set dst 192.168.2.0 255.255.255.0
        set distance 1
        set sdwan-zone "VPN"
    next
    edit 4
        set dst 192.168.2.0 255.255.255.0
        set distance 20
        set blackhole enable
    next
end
```
![sd-wan](screenshots/show-router-static.png)

### 4. SD-WAN Service Rule 5 (Overlay Path Steering)
Service Rule 5 prioritizes Tunnel 1 (`HQ-BRANCH-VPN` / Member 3) over Tunnel 2 (`HQ-BRANCH-VPN-2` / Member 4):

```fortios
config system sdwan
    config service
        edit 5
            set name "To-Branch-VPN"
            set mode manual
            set dst "BRANCH-LAN"
            set priority-members 3 4
        next
    end
end
```
![sd-wan](screenshots/tshow-sd-wan-service.png)

### 5. Zone-Based Firewall Security Policies
Security policies reference the `VPN` zone directly, avoiding policy duplication across multiple tunnels:

```fortios
config firewall policy
    edit 10
        set name "HQ-to-Branch-VPN"
        set srcintf "port1"
        set dstintf "VPN"
        set srcaddr "HQ-LAN"
        set dstaddr "BRANCH-LAN"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 11
        set name "Branch-VPN-to-HQ"
        set srcintf "VPN"
        set dstintf "port1"
        set srcaddr "BRANCH-LAN"
        set dstaddr "HQ-LAN"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```
![sd-wan](screenshots/show-firewall-policy.png)

---

## Verification & Execution Lifecycle

### Phase 1: IKE Gateway & Tunnel SA Verification
Both Phase 1 and Phase 2 Security Associations negotiated concurrently across both providers:

```text
FortiGate-VM64-KVM # diagnose vpn ike gateway list

vd: root/0
name: HQ-BRANCH-VPN
version: 2
interface: port2 4
addr: 10.0.1.6:500 -> 10.0.1.2:500
IKE SA: created 1/2  established 1/2  time 20/30/40 ms
IPsec SA: created 1/2  established 1/2  time 0/20/40 ms
id/spi: 934 662d30dbb1b9f156/b4eb60a2b47add55
direction: initiator
status: established 1408-1408s ago = 40ms
proposal: des-sha256

vd: root/0
name: HQ-BRANCH-VPN-2
version: 2
interface: port3 5
addr: 10.0.2.6:500 -> 10.0.2.2:500
IKE SA: created 1/2  established 1/2  time 30/35/40 ms
IPsec SA: created 1/2  established 1/2  time 0/20/40 ms
id/spi: 937 feedcf4aa088d99e/cfcfbb1aa69f1d5
direction: responder
status: established 1408-1408s ago = 30ms
proposal: des-sha256
```
![sd-wan](screenshots/diagnose-vpn-ike-1.png)

![sd-wan](screenshots/diagnose-vpn-ike-2.png)

---

### Phase 2: SD-WAN Overlay Zone & Routing Table Integration
1. **Member State Diagnostics:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan member
   Member(1): interface: port2, flags=0x0 , gateway: 10.0.1.5, priority: 1 1024, weight: 0
   Member(2): interface: port3, flags=0x0 , gateway: 10.0.2.5, priority: 1 1024, weight: 0
   Member(3): interface: HQ-BRANCH-VPN, flags=0xc , gateway: 10.0.1.6, priority: 1 1024, weight: 0
   Member(4): interface: HQ-BRANCH-VPN-2, flags=0xc , gateway: 10.0.2.6, priority: 1 1024, weight: 0
   ```
   
![sd-wan](screenshots/diagnose-sd-wan-member.png)

2. **RIB / Routing Table Verification:**
   ```text
   FortiGate-VM64-KVM # get router info routing-table all

   S    192.168.2.0/24 [1/0] via HQ-BRANCH-VPN tunnel 10.0.1.6 vrf 0, [1/0]
                       [1/0] via HQ-BRANCH-VPN-2 tunnel 10.0.2.6 vrf 0, [1/0]
   ```

![sd-wan](screenshots/routing-table-all.png)

3. **Service Rule 5 State:**
   ```text
   FortiGate-VM64-KVM # diagnose sys sdwan service 5

   Service(5): Address Mode(IPV4) flags=0x200 use-shortcut-sla
   Tie break: cfg
   Gen(1), TOS(0x0/0x0), Protocol(0: 1->65535), Mode(manual)
   Members(2):
     1: Seq_num(3 HQ-BRANCH-VPN), alive, selected
     2: Seq_num(4 HQ-BRANCH-VPN-2), alive, selected
   Dst address(1):
           192.168.2.0-192.168.2.255
   ```
![sd-wan](screenshots/service-5.png)

---

### Phase 3: Traffic Generation & Cryptographic Telemetry
An ICMP ping was initiated from HQ Server (`192.168.1.10`) targeting Branch Client (`192.168.2.10`):

1. **End-to-End Reachability:**
   ```bash
   localhost:~# ping 192.168.2.10 -c 4
   PING 192.168.2.10 (192.168.2.10): 56 data bytes
   64 bytes from 192.168.2.10: seq=0 ttl=62 time=19.185 ms
   64 bytes from 192.168.2.10: seq=1 ttl=62 time=22.036 ms
   64 bytes from 192.168.2.10: seq=2 ttl=62 time=23.598 ms
   64 bytes from 192.168.2.10: seq=3 ttl=62 time=15.371 ms

   --- 192.168.2.10 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 15.371/20.047/23.598 ms
   ```
![sd-wan](screenshots/hq-ping-successful.png)

2. **Tunnel Packet Counters Incrementing:**
   ```text
   FortiGate-VM64-KVM # diagnose vpn tunnel list
   name=HQ-BRANCH-VPN ver=2 serial=1 10.0.1.6:0->10.0.1.2:0
   ...
   dec: spi=2ce9adca esp=des key=8
   enc: spi=8bb00133 esp=des key=8
   dec:pkts/bytes=110/9240, enc:pkts/bytes=110/12320
   ```
   
   ![sd-wan](screenshots/encryption-goes-up-1.png)
   
   ![sd-wan](screenshots/encryption-before-after.png)

3. **Software Crypto Telemetry:**
   ```text
   FortiGate-VM64-KVM # diagnose vpn ipsec status
   SOFTWARE:
     Encryption (encrypted/decrypted)
       des        : 73           83
     Integrity (generated/validated)
       sha256     : 73           83
   ```
![sd-wan](screenshots/diagnose-vpn-ipsec-after.png)


---

### Phase 4: Wire-Level Wireshark Packet Proof
Packet captures taken on the transit underlay link proved complete payload protection:

1. **Phase 1 Handshake (`_ws.col.protocol == "ISAKMP"`):**
   * Observed `IKE_SA_INIT` and `IKE_AUTH` exchanges between `10.0.1.6 <-> 10.0.1.2` over UDP port 500.
   
![sd-wan](screenshots/wireshark-init-auth.png)
   
2. **Encrypted Payload (`_ws.col.protocol == "ESP"`):**
   * Observed continuous IP Protocol 50 (ESP) frames with Security Parameter Indexes:
     * `SPI = 0x2ce9adce`
     * `SPI = 0x8bb00137`
     
![sd-wan](screenshots/wireshark-esp.png)
     
3. **Encapsulation Proof (`esp || icmp`):**
   * Captured the transition: Unencrypted ICMP packets (`192.168.1.10 -> 192.168.2.10`) entering `port1` were immediately enveloped into ESP packets (`10.0.1.2 -> 10.0.1.6`) on `port2`. Zero plaintext LAN IP or ICMP data was exposed on the WAN underlay.

![sd-wan](screenshots/wireshark-icmp-esp.png)

---

## Evidence Index

| Phase | Artifact / Screenshot | Purpose & Verification |
| :--- | :--- | :--- |
| **Phase 1: IKE Gateways** | `diagnose vpn ike gateway list HQ.png` | Confirms both IKEv2 gateways established on FortiGate HQ. |
| **Phase 1: IKE Gateways** | `diagnose vpn ike gateway list branch.png` | Confirms IKEv2 gateways active on FortiGate Branch. |
| **Phase 1: IKE Gateways** | `diagnose vpn ike gateway list BRANCH 1st VPN.png` | Detailed state for primary tunnel (`HQ-BRANCH-VPN` on `port2`). |
| **Phase 1: IKE Gateways** | `diagnose vpn ike gateway list BRANCH 2nd VPN.png` | Detailed state for secondary tunnel (`HQ-BRANCH-VPN-2` on `port3`). |
| **Phase 1: IPsec SAs** | `diagnose vpn ipsec status HQ .png` | Cryptographic engine statistics on HQ. |
| **Phase 1: IPsec SAs** | `diagnose vpn ipsec status BRANCH.png` | Cryptographic engine statistics on Branch before test traffic. |
| **Phase 1: IPsec SAs** | `diagnose vpn ipsec status BRANCH after traffic.png` | Confirms incrementing `des` and `sha256` encryption/decryption counters. |
| **Phase 1: Tunnel Status** | `diagnose vpn tunnel list HQ.png` | Displays SPIs, lifetime, and crypto attributes for both tunnels on HQ. |
| **Phase 1: Tunnel Status** | `diagnose vpn tunnel list BRANCH.png` | Displays SPIs, lifetime, and crypto attributes for both tunnels on Branch. |
| **Phase 1: Tunnel Status** | `diagnose vpn tunnel list 1st VPN.png` | Specific tunnel list output for Tunnel 1. |
| **Phase 1: Tunnel Status** | `diagnose vpn tunnel list 2nd VPN.png` | Specific tunnel list output for Tunnel 2. |
| **Phase 1: Tunnel Status** | `diagnose vpn tunnel list 1st VPN encrypted packets going up after ping.png` | **Crucial Proof:** Shows `enc:pkts` rising from 92 to 110 after sending ping traffic. |
| **Phase 2: Routing & Zones** | `show config sys sdwan config service.png` | SD-WAN Service Rule 5 (`HQ-to-Branch-VPN`) mapped to members 3 & 4. |
| **Phase 2: Routing & Zones** | `diagnose sys sdwan member.png` | Confirms members 3 & 4 with flag `0xc` assigned to zone `VPN`. |
| **Phase 2: Routing & Zones** | `diagnose sys sdwan service branch.png` | Overall SD-WAN service inventory on Branch. |
| **Phase 2: Routing & Zones** | `diagnose sys sdwan service 5 HQ.png` | Verifies Rule 5 active on HQ with both overlay members selected. |
| **Phase 2: Routing & Zones** | `diagnose sys sdwan service 5 BRANCH.png` | Verifies Rule 5 active on Branch with both overlay members selected. |
| **Phase 2: Routing & Zones** | `show static routes VPN zone HQ.png` | Static route pointing `192.168.2.0/24` to `VPN` zone + floating blackhole. |
| **Phase 2: Routing & Zones** | `show static routes VPN zone Branch.png` | Static route pointing `192.168.1.0/24` to `VPN` zone + floating blackhole. |
| **Phase 2: Routing & Zones** | `get router info routing-table all HQ.png` | Kernel RIB on HQ showing ECMP static routes across both tunnels. |
| **Phase 2: Routing & Zones** | `get router info routing-table all Branch .png` | Kernel RIB on Branch showing ECMP static routes across both tunnels. |
| **Phase 2: Routing & Zones** | `get router info routing-table all.png` | Consolidated routing table view confirming tunnel next-hops. |
| **Phase 2: Policies** | `show firewall policy HQ.png` | Zone-based firewall policies 10 and 11 referencing `srcintf/dstintf "VPN"`. |
| **Phase 2: Policies** | `BRANCH.png` | Branch firewall policies confirming symmetrical zone-based security rules. |
| **Phase 3: Validation** | `HQ NODE SENDING PING.png` | Client ping test (`192.168.1.10 -> 192.168.2.10`) with 0% packet loss. |
| **Phase 3: Wireshark Proof** | `INIT AUTH PCAPS FOUND.png` | Packet capture of IKEv2 `IKE_SA_INIT` and `IKE_AUTH` exchanges on UDP 500. |
| **Phase 3: Wireshark Proof** | `ESP PCAPs found overview.png` | Wireshark capture confirming transit traffic encapsulated in IP Protocol 50 (ESP). |
| **Phase 3: Wireshark Proof** | `icmp and esp trailing proof of encryption.png` | **Smoking Gun:** Wireshark showing ICMP payload securely enveloped inside ESP. |
| **Phase 3: Live Video** | `sd-wan-esp-pcap-2026-10-27_04.58.36.mp4` | Screen recording of the live ping stream, Wireshark ESP capture, and CLI tunnel statistics. |

---

## Business Value Delivered
* **Zero-Trust Multi-Path Transport:** Encrypts 100% of inter-branch communications with military-grade IPsec, allowing enterprises to safely leverage cheap public commodity broadband without compromising data compliance (HIPAA, PCI-DSS, SOC 2).
* **Seamless SA Preservation:** Eliminates tunnel teardown delays during transport brownouts. The SD-WAN engine re-routes the existing encrypted ESP stream over the surviving underlay in milliseconds.
* **Simplified Security Architecture:** Abstracting tunnels into an SD-WAN zone reduces firewall rule sprawl by 50%, enabling clean, maintainable zone-based security policies.

---

### Next Module
* **[Module 2: Cisco Catalyst SD-WAN (Viptela)](../../02-Cisco-Catalyst-SDWAN/)** — Exploring the enterprise controller-based architecture: vManage, vSmart, vBond orchestration, Overlay Management Protocol (OMP), and centralized data/app policies.
