---
title: "OSPF Adjacency Stuck in EXSTART: How an MTU Mismatch Silently Breaks the DBD Exchange"
date: 2026-06-27
draft: false
tags:
  - ospf
  - exstart
  - mtu
  - mtu-mismatch
  - dbd
  - routing
  - troubleshooting
  - cisco-ios
  - rfc-2328
categories: ["incident", "architecture"]
summary: "An OSPF neighbor sits in EXSTART while the other end shows EXCHANGE — a walkthrough of two documented MTU-mismatch cases that explain the asymmetric signature and how to fix it."
description: "Why OSPF adjacencies stick in EXSTART: an interface MTU mismatch silently rejects DBD packets per RFC 2328. How to spot it, debug it, and fix it."
---

> **Note on sourcing:** This article walks through two *publicly documented* cases rather than a first-person incident — Cisco's own TAC recreation (Document ID 13684) and a field account from Ivan Pepelnjak's ipSpace.net blog. Output and values are quoted from those published sources and cited inline.

## State

The adjacency that won't come up but never hard-fails. One side shows `EXSTART`, the other shows `EXCHANGE`, and neither ever reaches `FULL`. Hellos are flowing, the neighbor is seen, the interface is up — and yet the database never synchronizes, so no LSAs are exchanged and no routes are learned across that link.

The asymmetry is the fingerprint. The router with the **lower** MTU is stuck in `EXSTART`; the router with the **higher** MTU sits in `EXCHANGE`. That split is not random — it falls directly out of how OSPF negotiates the Database Description (DBD) exchange, and it points straight at an interface MTU mismatch.

In Cisco's published recreation of this failure (TAC Doc 13684), the symptom is exactly that: two routers across a Frame Relay link, one holding `EXSTART`, the other `EXCHANGE`, indefinitely. Nothing is down. The adjacency simply never completes.

## Impact

When an OSPF adjacency never leaves EXSTART, the two routers never synchronize their link-state databases. No LSAs cross that link, so neither router installs the routes the other is advertising. The interface is up and pingable, which is what makes this deceptive: connectivity tests pass while routing over that link is effectively dead. On a new circuit turn-up it looks like the link "isn't working"; on a link added for redundancy, the backup path silently fails to form and nobody notices until the primary drops.

## Environment

Cisco's documented recreation (TAC Doc 13684) uses this topology:

| Component | Details |
| --- | --- |
| Platform A — "Router 6" | Cisco IOS, serial interface, **MTU 1500** (default) |
| Platform B — "Router 7" | Cisco IOS, serial interface, **MTU 1450** |
| Link type | Frame Relay point-to-point sub-interfaces |
| OSPF area | Area 0 |
| Trigger | One side's serial MTU set to 1450 while the other kept the 1500 default |

The mismatch origin here is the simplest possible: one interface was explicitly configured `mtu 1450` while the neighbor ran the IOS default of 1500. That single line is enough.

The harder real-world variant comes from Ivan Pepelnjak's ipSpace.net account: both customer routers had **MTU 1500** configured and matching — yet OSPF still stuck in EXSTART. The mismatch wasn't on the routers at all. The Frame Relay PVC traversed three providers, and the provider *in the middle* had the PVC MTU set to **1100 bytes**. The endpoints agreed; the path didn't. (Source: ipSpace.net, "OSPF Neighbors Stuck in EXSTART," reader field report.)

## Diagnostic path

### Step 1 — Read the neighbor table on both ends

The single most diagnostic move is to look at `show ip ospf neighbor` on **both** routers at once. The MTU-mismatch signature is the asymmetric state. From Cisco's documented case:

```text
router-6# show ip ospf neighbor
Neighbor ID     Pri   State          Dead Time   Address       Interface
172.16.7.11      1   EXCHANGE/ -    00:00:36    172.16.7.11   Serial2.7

router-7# show ip ospf neighbor
Neighbor ID     Pri   State          Dead Time   Address       Interface
10.170.10.6      1   EXSTART/  -    00:00:33    10.170.10.6   Serial0.6
```

Router 6 (MTU 1500) holds `EXCHANGE`; Router 7 (MTU 1450) holds `EXSTART`. Seeing both at once is what separates this from the dozen other reasons an adjacency won't form.

### Step 2 — Compare the MTUs (and the right kind of MTU)

OSPF compares the **IP MTU** it advertises in the DBD, which is not always the same as the L2 interface MTU — they can diverge on jumbo-frame configs, sub-interfaces, and tunnels. So check both:

```text
show interface <intf>        ! L2 MTU
show ip interface <intf>     ! IP MTU — the value OSPF actually advertises
show ip ospf interface <intf>
```

This is exactly the trap in the ipSpace.net case: both endpoint interfaces read MTU 1500, so a quick `show interface` comparison "passes" and sends you looking in the wrong direction. The lesson there — only learned by asking how the PVC was actually built — is that a matching MTU on both routers does not guarantee a matching MTU along the *path*.

### Step 3 — Confirm it in the adjacency debug

`debug ip ospf adj` on the EXSTART router prints the smoking gun. Scope it to the neighbor and log to buffer — `debug ip packet` (which the Cisco doc pairs with it) is far chattier and ACL-scoping it is mandatory on anything busy.

From Cisco's documented trace, the lower-MTU router (Router 7) receives Router 6's DBD and immediately flags the mismatch:

```text
OSPF: Rcv DBD from 10.170.10.6 on Serial0.6
   seq 0xE44 opt 0x2 flag 0x7 Len 32 mtu 1500 state EXSTART
OSPF: Nbr 10.170.10.6 has larger interface MTU
```

That second line — `has larger interface MTU` — is the diagnosis in plain text. The `mtu 1500` in the receive line is the neighbor's advertised value; the local interface is 1450, and the 50-byte gap is the whole problem. Router 7 then never ACKs and retransmits its own initial DBD instead:

```text
OSPF: Send DBD to 10.170.10.6 on Serial0.6 seq 0x13FD opt 0x2 flag 0x7 Len 32
OSPF: Retransmitting DBD to 10.170.10.6 on Serial0.6 [1]
```

The `Retransmitting DBD ... [1] ... [2] ... [3]` counter climbing with no progress is the loop made visible.

## Root cause

The mechanism is defined by RFC 2328. During adjacency formation, each router advertises its interface MTU in the **Interface MTU field** of its DBD packets. Per **RFC 2328 §10.6**:

> "If the Interface MTU field in the Database Description packet indicates an IP datagram size that is larger than the router can accept on the receiving interface without fragmentation, the Database Description packet is rejected."

Cisco IOS has enforced this since 12.0(3). The failure then proceeds asymmetrically, which is why the two ends show different states:

1. In `EXSTART`, the router with the **higher Router-ID becomes Primary (master)** and sets the DBD sequence number. In Cisco's case that's Router 7 (172.16.7.11 > 10.170.10.6).
2. The router with the **larger** MTU (Router 6, 1500) receives the smaller neighbor's DBD, accepts it, and moves to `EXCHANGE`.
3. The router with the **smaller** MTU (Router 7, 1450) sees a DBD advertising MTU 1500 — larger than it can accept — **rejects it silently**, and stays in `EXSTART`, retransmitting its own initial DBD.
4. As Subordinate, Router 6 adopts the Primary's sequence number and sends a fuller DBD carrying LSA headers (the Cisco trace shows this grow to `Len 1472`, a ~1492-byte IP packet). That exceeds Router 7's 1450 MTU and is dropped — so it's never ACKed.

```text
Lower-MTU router (EXSTART, 1450)       Higher-MTU router (EXCHANGE, 1500)
        │  initial DBD (small) ─────────────►│  accepts → EXCHANGE
        │◄───────── DBD advertising mtu 1500 │
   rejects (RFC 2328 §10.6):                 │
   "Nbr has larger interface MTU"            │
        │  retransmit initial DBD ──────────►│  ACKs with larger DBD (LSA hdrs, ~1492B)
        │   ✗ exceeds local MTU, dropped     │
        └────────────── loop forever ────────┘
```

The loop is indefinite by design. Neither side is malfunctioning — both are following the spec. The link is simply asked to carry a DBD larger than one end will accept.

In the ipSpace.net variant the same rejection happens for a subtler reason: the routers advertised matching 1500-byte MTUs to each other, but the mid-path provider's 1100-byte PVC silently dropped the oversized frames at Layer 2 before they ever arrived. The OSPF MTU values matched; the *transport* couldn't carry them.

## Action / Fix

There are two ways out, and they are not equal.

**Preferred — align the MTU on both ends.** Make the interfaces agree. Depending on platform and what's driving the value, that's the L2 MTU, the IP MTU, or both:

```text
interface <intf>
 mtu <bytes>        ! L2
 ip mtu <bytes>     ! L3 — the value OSPF advertises and compares
```

In the Cisco case the fix is to make the two serial MTUs match — set Router 7 back to 1500, or Router 6 down to 1450. Note an MTU change can bounce the interface and affect other neighbors and L2 adjacencies on it — plan it.

**Last resort — `ip ospf mtu-ignore`.** This disables the DBD MTU check so the adjacency reaches `FULL` despite the mismatch. Apply it on the relevant interface on **both** ends:

```text
interface <intf>
 ip ospf mtu-ignore
```

Cisco positions this as needed only in rare cases — their canonical example is an FDDI-to-Ethernet conversion (a Catalyst 5000 RSM, virtual Ethernet at 1500 facing FDDI at 4500) where the platform fragments internally and the mismatch is benign.

The ipSpace.net case is the cautionary tale for *why* `mtu-ignore` is not a real fix: there, `ip ospf mtu-ignore` did **not** resolve the problem, because the middle Frame Relay provider was dropping the oversized frames at Layer 2. Ignoring the check let OSPF form the adjacency, but the data plane still couldn't carry the frames. The working fix was to lower the customers' IP MTU to fit the real path (the engineer used `ip mtu 1024`). That captures the rule precisely: `mtu-ignore` silences the protocol's complaint but does nothing about a path that genuinely can't carry the frame.

## Verification

After aligning the MTU, both ends should reach `FULL` and stay there, the LSDBs should synchronize, and the previously missing routes should install:

```text
router-7# show ip ospf neighbor
Neighbor ID     Pri   State        Dead Time   Address       Interface
10.170.10.6      1   FULL/  -     00:00:38    10.170.10.6   Serial0.6
```

Confirm the database is actually synced (`show ip ospf database` matching on both ends) and that the routes learned over the link now appear in `show ip route ospf` — reaching FULL is necessary but the real proof is routes installed and traffic taking the path.

## Lessons learned

The two cases together draw a clean line. Matching the MTU you can see on each router is the *first* check, not the last one — Cisco's recreation is the textbook version where the two interface values plainly disagree. But the ipSpace.net case is the one worth remembering: endpoints agreed at 1500 and OSPF still stuck, because the constraint lived in a mid-path provider PVC at 1100 bytes. So the durable takeaways are:

- **MTU belongs on the OSPF link turn-up checklist**, verified on *both* ends before expecting an adjacency — it's the most common cause of stuck EXSTART/EXCHANGE, especially in vendor-interop and carrier-transport links.
- **Matching router MTUs is not the same as a matching path MTU.** When both ends agree and OSPF still won't come up, ask how the transport in between is actually built.
- **`ip ospf mtu-ignore` treats the symptom, not the path.** It can let an adjacency form over a link that still can't carry the frames — which is worse than a stuck adjacency, because it fails silently in the data plane.

---

*Sources: [Cisco — Troubleshoot OSPF Neighbors Stuck in Exstart/Exchange State (Doc 13684)](https://www.cisco.com/c/en/us/support/docs/ip/open-shortest-path-first-ospf/13684-12.html); [RFC 2328 §10.6](https://datatracker.ietf.org/doc/html/rfc2328); [ipSpace.net — OSPF Neighbors Stuck in EXSTART](https://blog.ipspace.net/2007/10/ospf-neighbors-stuck-in-exstart/).*
