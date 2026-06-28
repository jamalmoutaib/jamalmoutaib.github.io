---
title: "Troubleshooting PMTU Black Holes on Cisco FTD: Syslog 602101, MSS Clamping, and VPN MTU Design"
date: 2026-05-10
draft: false
categories: ["incident", "security"]
tags:
  - cisco-ftd
  - cisco-asa
  - pmtu
  - mtu
  - icmp
  - ipsec-vpn
  - troubleshooting
  - firewall
  - mss-clamping
summary: "TCP sessions establish but stall during data transfer on a Cisco FTD VPN deployment — tracing FTD 602101 events to a blocked ICMP Type 3 policy and a silent PMTU black hole."
description: "Diagnosing a PMTU black hole on Cisco FTD: how blocked ICMP Type 3 messages silently break Path MTU Discovery, traced through syslog 602101, and the ICMP policy fix that resolved it."
---

## Introduction

TCP sessions up. Pings passing. VPN tunnel showing green. Applications broken.

That's the PMTU black hole. There's no hard-down state to trace — the infrastructure looks healthy right up until you try to actually use it. Large transfers time out. Small ones succeed. The natural assumption is that something's wrong with the server or the application. The network team clears themselves early. And the problem keeps happening.

This article documents a real investigation on a Cisco Secure Firewall Threat Defense (FTD) deployment where repeated `%FTD-6-602101` events were the first signal that something was structurally wrong with the VPN path. The root cause: ICMP Type 3 blocked at the policy level, silently breaking the feedback loop that Path MTU Discovery depends on to self-correct.

Intended for network engineers and firewall administrators managing Cisco FTD or ASA in VPN or MPLS environments.

---

## Environment

| Component             | Details                                    |
| --------------------- | ------------------------------------------ |
| Security Platform     | Cisco Secure Firewall Threat Defense (FTD) |
| Firewall Hardware     | Cisco Firepower Appliance                  |
| Management Platform   | Cisco FMC                                  |
| Tunnel Type           | IPsec VPN                                  |
| Transport             | MPLS / VPN Infrastructure                  |
| Source Network        | Internal LAN                               |
| Destination Network   | Remote Site LAN                            |
| Destination Interface | `<vpn_subinterface>`                       |
| Interface Name        | `<vpn_interface>`                          |

Interface configuration:

```bash
show running-config interface <vpn_subinterface>
```

```text
interface <vpn_subinterface>
 description <vpn_site_description>
 vlan <vpn_vlan_id>
 nameif <vpn_interface>
 security-level 0
```

---

## Problem Description

Users reported intermittent application failures over VPN paths while alternative routes worked normally. The pattern was consistent:

- TCP sessions established but stalled or degraded during data transfer
- Large file transfers timed out; small transactions went through
- Pings passed — giving a misleading read on path health
- Failures surfaced at the application layer, pointing away from the network

The log message appearing on the FTD:

```text
%FTD-6-602101:
PMTU-D packet 1400 bytes greater than effective mtu 1348,
dest_addr=<destination_ip>,
src_addr=<source_ip>,
prot=TCP
```

That single entry contains the whole picture: packet size, effective MTU on the VPN path, and the affected flow. The gap is 52 bytes — invisible to basic connectivity tests, large enough to silently break any application relying on standard TCP segment sizes.

Note: the same message and root cause appear on classic Cisco ASA appliances and on FTD code prior to 6.3, where the prefix reads `%ASA-6-602101`. The troubleshooting process below applies to either platform.

---

## Troubleshooting Steps

### Step 1 — Parse the FTD 602101 Log

The 602101 message is precise: a 1400-byte TCP packet arrived at an interface with an effective MTU of 1348 bytes.

```text
1400 - 1348 = 52 bytes over the limit
```

Modern operating systems set the Don't Fragment (DF) bit on TCP traffic by default — that's how Path MTU Discovery is supposed to work. With DF set, the FTD can't fragment the packet. It drops it. The firewall is doing exactly what it should. The problem is architectural.

---

### Step 2 — Determine Why the Effective MTU Is 1348

Standard Ethernet MTU is 1500 bytes. Getting from 1500 to 1348 takes two compounding factors: a transport path already running below standard MTU on the MPLS/VPN layer, then IPsec encapsulation overhead on top of that.

IPsec/NAT-T alone typically costs 80–90 bytes (1500 → ~1410–1420). A 152-byte total reduction points to a sub-1500 transport MTU before any encryption overhead is applied.

| Header                           | Approximate Overhead |
| -------------------------------- | -------------------- |
| Outer IP Header                  | 20 bytes             |
| UDP NAT-T Header                 | 8 bytes              |
| ESP Header + Trailer + Auth Data | Variable             |

The combined overhead on this tunnel sets the payload ceiling at 1348 bytes.

---

### Step 3 — Understand PMTU Discovery and Where It Breaks

Path MTU Discovery is the self-correction mechanism that lets TCP hosts adapt to constrained paths:

```text
Host sends 1400-byte packet
       ↓
FTD detects MTU violation → drops packet
       ↓
FTD returns ICMP Type 3 Code 4 (Fragmentation Needed, MTU = 1348)
       ↓
Host receives ICMP → reduces segment size
       ↓
Traffic flows correctly
```

It works — unless ICMP Type 3 Code 4 is filtered anywhere in the path. When it is, the host never gets the signal. It retransmits the same 1400-byte packet. The FTD drops it again. The loop repeats. This is the PMTU black hole: not a broken tunnel, but a severed feedback loop.

---

### Step 4 — Audit the ICMP Policy — Key Finding

Reviewing the ICMP service policy on the VPN-facing interface showed the problem immediately. The policy permitted only:

```text
service-object icmp echo
service-object icmp echo-reply
```

ICMP Unreachable (Type 3) wasn't there. That's the message type carrying the Fragmentation Needed signal (Code 4) — exactly what the FTD sends when it drops an oversized packet.

Without it, the discovery loop had nowhere to go:

```text
FTD drops oversized packet
       ↓
FTD generates ICMP Type 3 Code 4
       ↓
ICMP Type 3 hits the policy → DROPPED
       ↓
Source host receives no feedback
       ↓
Source host retransmits the same 1400-byte packet
       ↓
Loop repeats → PMTU Black Hole
```

The VPN tunnel was healthy the entire time. The ICMP policy was the failure.

This was verified through direct platform inspection, not just an ACL read:

- `show interface <vpn_interface>` / `show running-config interface <vpn_interface>` — confirm interface MTU and config
- `show crypto ipsec sa` — confirm the tunnel's negotiated path MTU and check fragmentation counters
- `show asp drop` — verify PMTU-exceeded / DF-set drops at the data-plane level
- `packet-tracer input <inside_zone> tcp <src_ip> <src_port> <dst_ip> 443 detailed` — simulate the flow and trace it through the policy pipeline

---

### Step 5 — Apply Temporary Mitigation

As an immediate workaround, the MTU on the source VLAN was reduced to 1200 bytes. This kept hosts from generating packets larger than the tunnel MTU, which stopped the 602101 events.

**Effect:** Immediate symptom resolution.

**Problem:** It lowered MTU for all traffic on that VLAN — including flows that never touch the VPN — degrading throughput globally. A valid emergency measure, not a production fix.

---

## Root Cause

The ICMP service policy on the VPN-facing interface permitted echo and echo-reply but blocked Type 3 (Destination Unreachable — Fragmentation Needed). The IPsec VPN tunnel had an effective MTU of 1348 bytes from MPLS transport constraints plus encapsulation overhead. Source hosts were sending standard 1400-byte TCP segments with the DF bit set.

When the FTD dropped those packets, it generated ICMP Type 3 Code 4 messages to signal the correct MTU back to the senders. Those messages hit the policy and were silently discarded. The hosts never adjusted. The firewall kept dropping. The tunnel was fully operational throughout — the failure was a blocked feedback loop that prevented Path MTU Discovery from ever completing.

---

## Resolution

**Temporary workaround:** Source VLAN MTU reduced to 1200 bytes. Stopped the 602101 events by keeping packets under the tunnel ceiling. Not sustainable — affected all VLAN traffic.

**Permanent fix — Permit ICMP Type 3:**

The ICMP service policy was updated to explicitly allow ICMP Unreachable:

**Before:**

```text
service-object icmp echo
service-object icmp echo-reply
```

**After:**

```text
service-object icmp echo
service-object icmp echo-reply
service-object icmp unreachable
```

Once ICMP Type 3 Code 4 messages could reach source hosts, Path MTU Discovery worked as designed. Hosts received the signal, reduced their TCP segment sizes to fit within 1348 bytes, and application traffic normalized. FTD 602101 events dropped to expected, transient levels. The source VLAN MTU was then restored to 1500 bytes.

**No MSS clamping was required to resolve this incident.**

---

> **Best Practice Note:** Fixing the ICMP policy resolved this case. But TCP MSS clamping adds a layer of protection that holds even when ICMP is filtered by devices outside your control — upstream providers, transit networks, remote-site firewalls:
>
> ```bash
> sysopt connection tcpmss 1308
> ```
>
> Deploy both. They address the same problem from different angles.

---

## Key Takeaways

- **FTD 602101 is a symptom.** The firewall is dropping packets correctly. The question is why Path MTU Discovery isn't self-correcting.
- **PMTU Discovery depends on ICMP Type 3.** Block Fragmentation Needed anywhere in the path and the feedback loop breaks silently.
- **Echo and echo-reply are not enough.** ICMP Type 3 is operationally critical on any VPN-facing interface. Permit it explicitly.
- **A healthy tunnel doesn't mean healthy application traffic.** TCP sessions can stall on a fully operational VPN — which is why this pattern gets misattributed to servers and applications.
- **MSS clamping is resilient where ICMP isn't.** Standard complement on any VPN-facing interface.
