---
title: "Troubleshooting PMTU Black Holes on Cisco FTD: Syslog 602101, MSS Clamping, and VPN MTU Design"
date: 2026-06-14
draft: false
categories: ["incident"]
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
---

## Introduction

If you've ever seen TCP sessions establish cleanly but then stall or time out during actual data transfer — while pings pass and the VPN tunnel shows as up — you're likely dealing with a PMTU black hole. This is one of the most frustrating failure patterns in VPN environments: the infrastructure looks healthy, but applications silently break.

This article walks through a real-world investigation on a Cisco Secure Firewall Threat Defense (FTD) deployment where repeated `%FTD-6-602101` log events were the first clue that something was wrong with the VPN path. The root cause turned out to be ICMP Type 3 being blocked — silently preventing Path MTU Discovery from working and trapping the network in a black hole. It covers the full diagnostic process and the fix that actually resolved it.

Intended for: network engineers and firewall administrators managing Cisco FTD/ASA in VPN or MPLS environments.

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
| Destination Interface | Port-channel2.351                          |
| Interface Name        | <vpn_interface>                            |

Interface configuration:

```bash
show running-config interface Port-channel2.351
```

```text
interface Port-channel2.351
 description VL351-REMOTE-SITE
 vlan 351
 nameif <vpn_interface>
 security-level 0
```

---

## Problem Description

Users reported intermittent application connectivity across VPN paths while alternative routes functioned normally. The symptoms were classic PMTU failure:

- TCP sessions established successfully but degraded or stalled during data transfer
- Large file transfers timing out while small transactions worked fine
- Ping and basic reachability tests passing — giving a false sense that the path was healthy
- Application-layer failures that looked like server or application issues, not network

The primary log message observed on the FTD:

```text
%FTD-6-602101:
PMTU-D packet 1400 bytes greater than effective mtu 1348,
dest_addr=<destination_ip>,
src_addr=<source_ip>,
prot=TCP
```

This single log entry contains everything needed to understand the problem: the packet size, the effective MTU on the VPN path, and the affected flow.

Note: the identical message and root cause apply on classic Cisco ASA appliances and on FTD code prior to release 6.3, where it appears with the legacy %ASA-6-602101 prefix. The troubleshooting steps below are the same on either platform.

---

## Troubleshooting Steps

### Step 1 — Parse the FTD 602101 Log

The 602101 message tells a precise story. A TCP packet of 1400 bytes arrived at an interface whose effective MTU was only 1348 bytes.

```text
1400 - 1348 = 52 bytes over the limit
```

Modern operating systems typically set the Don't Fragment (DF) bit on TCP traffic by default to support Path MTU Discovery. Because the DF bit is set, the FTD cannot fragment the oversized packet. It must drop it. The firewall is behaving correctly — the problem is upstream in the architecture.

---

### Step 2 — Determine Why the Effective MTU Is 1348

Standard Ethernet MTU is 1500 bytes. The 1348-byte effective MTU reflects two combined factors: the underlying MPLS/VPN transport path, which was already running below the standard 1500-byte MTU, plus the additional overhead added by IPsec encapsulation on top of that. IPsec/NAT-T overhead alone typically only accounts for 80–90 bytes (1500 → ~1410–1420); a 152-byte reduction points to a sub-1500 MTU already present on the transport before encryption overhead is applied. Each encrypted packet carries additional headers that consume payload space:

| Header                           | Approximate Overhead |
| -------------------------------- | -------------------- |
| Outer IP Header                  | 20 bytes             |
| UDP NAT-T Header                 | 8 bytes              |
| ESP Header + Trailer + Auth Data | Variable             |

The combined overhead on this tunnel reduces the usable MTU to 1348 bytes — meaning any payload larger than that cannot traverse the tunnel unmodified.

---

### Step 3 — Understand PMTU Discovery and Why It Can Fail

Path MTU Discovery (PMTU) is the mechanism that allows TCP hosts to self-correct their packet sizes:

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

This works — unless ICMP Type 3 Code 4 is filtered or blocked anywhere in the path. When that happens, the host never learns the correct MTU, keeps sending 1400-byte packets, the FTD keeps dropping them, and applications appear broken. This is the **PMTU black hole**.

---

### Step 4 — Audit the ICMP Policy — Key Finding

Reviewing the ICMP policy on the relevant interface revealed the problem. The policy only explicitly permitted echo and echo-reply:

```text
service-object icmp echo
service-object icmp echo-reply
```

ICMP Unreachable (Type 3) was not included. This is the message type that carries the **Fragmentation Needed** signal (Code 4) — the exact feedback the FTD sends to source hosts when it drops an oversized packet.

Because Type 3 was blocked, the PMTU Discovery loop could never complete:

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

This confirmed the black hole. The VPN tunnel was healthy. The ICMP suppression was the failure.

This was corroborated with direct checks rather than relying on the ACL reading alone:

- `show interface <vpn_interface>` / `show running-config interface <vpn_interface>` — confirm interface MTU and config
- `show crypto ipsec sa` — confirm the tunnel's negotiated path MTU and check for fragmentation counters
- `show asp drop` — check for PMTU-exceeded / DF-set drop counters at the data-plane level
- `packet-tracer input <inside_zone> tcp <src_ip> <src_port> <dst_ip> 443 detailed` — simulate the flow and confirm where in the policy pipeline the packet is evaluated

---

### Step 5 — Apply Temporary Mitigation

As an immediate workaround, the MTU on the source VLAN was reduced to 1200 bytes. This prevented hosts from generating packets larger than the tunnel MTU, which stopped the 602101 events.

**Effect:** Immediate symptom resolution.

**Problem:** This lowered the MTU for all traffic on that VLAN — including flows that don't use the VPN — reducing efficiency globally. This is a valid emergency workaround, not a production solution.

---

## Root Cause

The root cause was ICMP Type 3 (Destination Unreachable — Fragmentation Needed) being blocked by the firewall's ICMP service policy. The IPsec VPN tunnel on the FTD had an effective MTU of 1348 bytes due to encapsulation overhead, while source hosts were generating standard 1400-byte TCP segments. When the FTD dropped those oversized packets, it attempted to return ICMP Type 3 Code 4 messages to notify the senders of the correct path MTU. Because the policy only permitted ICMP echo and echo-reply, those feedback messages were silently discarded. Source hosts never received the MTU signal, continued transmitting 1400-byte segments, and the firewall continued dropping them. The VPN tunnel itself was fully operational — the failure was entirely caused by suppressed ICMP feedback preventing Path MTU Discovery from completing.

---

## Resolution

**Temporary workaround applied:** Source VLAN MTU reduced to 1200 bytes. This stopped the 602101 events by preventing hosts from generating packets larger than the tunnel MTU — but it degraded performance for all VLAN traffic and was not a sustainable fix.

**Permanent fix — Permitting ICMP Type 3:**

The ICMP service policy was updated to explicitly allow ICMP Unreachable messages. The change was straightforward:

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

Once ICMP Type 3 Code 4 (Fragmentation Needed) messages could reach the source hosts, Path MTU Discovery operated as designed. Hosts received the MTU signal, self-adjusted their TCP segment sizes to fit within the 1348-byte effective tunnel MTU, and application traffic normalized. FTD 602101 events dropped to expected, self-correcting transient levels. The source VLAN MTU was then restored to 1500 bytes.

**No MSS clamping was required to resolve this incident.**

---

> **Best Practice Note:** While fixing the ICMP policy resolved this case, TCP MSS clamping provides an additional layer of defense that makes your VPN path resilient even when ICMP is filtered by upstream or downstream devices outside your control:
>
> ```bash
> sysopt connection tcpmss 1308
> ```
>
> Consider deploying both — they are complementary, not exclusive.

---

## Key Takeaways

- **FTD 602101 is a symptom, not the root cause** — the FTD is dropping packets correctly; the investigation should focus on why PMTU Discovery isn't fixing the problem automatically.
- **PMTU Discovery depends entirely on ICMP Type 3** — if Fragmentation Needed messages are blocked anywhere in the path, hosts never self-correct and a black hole develops silently.
- **Always explicitly permit ICMP unreachable on VPN-facing interfaces** — echo and echo-reply alone are not enough. ICMP Type 3 is operationally critical.
- **A healthy VPN tunnel does not mean healthy application traffic** — the tunnel can be fully up while TCP sessions stall, making this failure pattern easy to misdiagnose as an application or server issue.
- **MSS clamping is a resilient complement** — it protects against MTU issues even when ICMP is filtered by devices outside your control, and should be standard practice on any VPN-facing interface.
