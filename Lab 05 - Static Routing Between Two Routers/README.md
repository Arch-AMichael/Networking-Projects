# Lab 05 — Static Routing Between Two Routers

**Date:** 2026-07-19
**Platform:** Cisco Packet Tracer 8.x — 2x ISR 4331, 2x PC-PT
**Time spent:** ~1 hr
**Status:** Complete — end-to-end connectivity verified

---

## Objective

Connect two isolated LANs across a point-to-point WAN link using static routes only (no dynamic routing protocol), and verify end-to-end reachability from host to host.

Success criteria:

1. Each PC holds a valid address in its local subnet and can ping its own default gateway.
2. Both routers can ping each other across the WAN link.
3. PC0 can ping PC1 and vice versa.

---

## Topology

```
      192.168.0.0/24            10.10.10.0/30            192.168.100.0/24
                          Gi0/0/1        Gi0/0/1
   PC0 ──── Gi0/0/0 [ Router0 ] ──────────── [ Router1 ] Gi0/0/0 ──── PC1
   .2            .1        .1                    .2         .1          .2
```

![Topology](topology.png)

---

## Addressing table

| Device  | Interface | IP address     | Subnet mask     | Default gateway |
|---------|-----------|----------------|-----------------|-----------------|
| Router0 | Gi0/0/0   | 192.168.0.1    | 255.255.255.0   | —               |
| Router0 | Gi0/0/1   | 10.10.10.1     | 255.255.255.252 | —               |
| Router1 | Gi0/0/0   | 192.168.100.1  | 255.255.255.0   | —               |
| Router1 | Gi0/0/1   | 10.10.10.2     | 255.255.255.252 | —               |
| PC0     | Fa0       | 192.168.0.2    | 255.255.255.0   | 192.168.0.1     |
| PC1     | Fa0       | 192.168.100.2  | 255.255.255.0   | 192.168.100.1   |

**Why /30 on the WAN link:** a point-to-point link needs exactly two usable addresses. A /30 gives a block size of 4 — one network address (10.10.10.0), two hosts (.1 and .2), one broadcast (10.10.10.3). Using a /24 here would waste 252 addresses on a link that can never hold more than two devices.

**Starting state:** Router1 Gi0/0/1 had no address configured. The WAN link was therefore down before any routing work began — this was the first thing that needed fixing, not the static routes.

---

## Configuration

### Router1 — bring up the WAN interface

The missing piece from the starting state. Without this, no static route on either side can work, because there is no L3 path for a next hop to resolve against.

```
enable
configure terminal
 interface GigabitEthernet0/0/1
  ip address 10.10.10.2 255.255.255.252
  no shutdown
 exit
```

### Router0 — static route to the remote LAN

```
enable
configure terminal
 ! Reach the far LAN (192.168.100.0/24) by handing packets
 ! to the OTHER router's WAN interface. Next hop is always
 ! the far end of the link, never an address on this router.
 ip route 192.168.100.0 255.255.255.0 10.10.10.2
 hostname router0
```

### Router1 — static route back

```
enable
configure terminal
 ! Mirror image. Router1 needs the RETURN path to 192.168.0.0/24.
 ! Without this line, echo requests arrive but replies have
 ! nowhere to go — the failure looks identical to a forward-path
 ! problem from the sender's point of view.
 ip route 192.168.0.0 255.255.255.0 10.10.10.1
 hostname router1
```

### Both routers — persist the config

```
end
copy running-config startup-config
```

Not done during the original session. Everything above lived only in RAM and would have been lost on reload.

### PCs

Set statically via Desktop → IP Configuration, per the addressing table above. Default gateway is the local router interface — the PC's only exit from its own subnet.

---

## Verification

### Local reachability first

From PC0:

```
C:\>ping 10.10.10.1

Reply from 10.10.10.1: bytes=32 time<1ms TTL=255
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

TTL 255 confirms zero routers were crossed — this is the directly attached device answering. Establishing that the local hop works before chasing anything further avoids debugging the wrong layer.

### End-to-end

From PC0:

```
C:\>ping 192.168.100.2

Reply from 192.168.100.2: bytes=32 time<1ms TTL=126
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

From PC1:

```
C:\>ping 192.168.0.2

Reply from 192.168.0.2: bytes=32 time<1ms TTL=126
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

**TTL=126 is the proof, not the 0% loss.** A Windows host sends with TTL 128 and each router decrements by one. 128 − 126 = 2 routers traversed, which is exactly the expected path. If this had come back at 127, packets were reaching the destination by some path other than the one designed.

### Routing tables

Router0:

```
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.10.10.0/30 is directly connected, GigabitEthernet0/0/1
L        10.10.10.1/32 is directly connected, GigabitEthernet0/0/1
      192.168.0.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.0.0/24 is directly connected, GigabitEthernet0/0/0
L        192.168.0.1/32 is directly connected, GigabitEthernet0/0/0
S     192.168.100.0/24 [1/0] via 10.10.10.2
```

Router1:

```
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.10.10.0/30 is directly connected, GigabitEthernet0/0/1
L        10.10.10.2/32 is directly connected, GigabitEthernet0/0/1
S     192.168.0.0/24 [1/0] via 10.10.10.1
      192.168.100.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.100.0/24 is directly connected, GigabitEthernet0/0/0
L        192.168.100.1/32 is directly connected, GigabitEthernet0/0/0
```

One `S` route per router, each pointing at the opposite LAN via the opposite WAN address. `[1/0]` is administrative distance 1 (static) and metric 0.

---

## Troubleshooting log

| # | Symptom | Hypothesis | Actual cause | Fix |
|---|---------|------------|--------------|-----|
| 1 | Router1 Gi0/0/1 had no IP; WAN link non-functional | Interface unconfigured | Correct — no address had ever been assigned | `ip address 10.10.10.2 255.255.255.252` + `no shutdown` |
| 2 | PC0 pings 10.10.10.1 fine, 10.10.10.2 times out | Far router down or link broken | Missing return route on Router1 — echo requests arrived, replies had no path back to 192.168.0.0/24 | `ip route 192.168.0.0 255.255.255.0 10.10.10.1` on Router1 |
| 3 | `tracert 10.10.10.2` shows hop 1 = 192.168.0.1, then all timeouts | Something beyond the local gateway is broken | Same root cause as #2 | Same fix |
| 4 | `config t` rejected with `% Invalid input detected at '^' marker` | Typo | Was at user EXEC (`Router>`), not privileged EXEC | `enable` first |
| 5 | `ip route 192.168.0.0 255.255.255.0 gig` accepted nothing | Wrong syntax | Interface keyword left incomplete — and the route was pointless anyway, that network is directly connected | Dropped the route entirely |
| 6 | `show ip route` rejected inside config mode | Command doesn't exist | Wrong mode — EXEC commands need the `do` prefix from config mode | `do show ip route` |
| 7 | `ip route 192.168.100.0 255.255.255.0 10.10.10.2` on Router1 → `%Invalid next hop address (it's this router)` | Route direction confusion | Pointed Router1 at its own LAN via its own WAN IP. Both halves inverted | `ip route 192.168.0.0 255.255.255.0 10.10.10.1` |
| 8 | `ping 192.168.100.0` returns unreachable | Remote LAN not reachable | Pinged the network address, not a host — no device owns .0 in a /24. Non-diagnostic | Ping .2 instead |

---

## Key takeaways

**A static route's next hop is always on the other side of the link.** Entry #7 is the sharpest lesson here. The route lives on the router that needs to *send* traffic somewhere, the destination is the network you're trying to *reach*, and the next hop is the neighbor's interface. IOS catches the self-reference case explicitly — `%Invalid next hop address (it's this router)` is a gift of an error message.

**One-way failure means a missing return path.** A ping needs routes in both directions. When the sender sees timeouts, the instinct is to check the forward path, but the forward path is usually fine — the reply is what's stranded. Entry #2 was exactly this.

**`tracert` localizes the break instantly.** Success at hop 1 followed by nothing tells you the local gateway is healthy and the problem is one hop further out. That single output narrowed the search from "the whole network" to "Router0's outbound path or Router1's return path."

**Read TTL, not just the reply.** 0% loss says traffic got through. TTL says *how*. Two decrements matched the designed two-router path; a different number would have meant something unintended was happening.

**Verify the physical and L3 basics before touching routing.** The unconfigured Gi0/0/1 (entry #1) meant no amount of correct static routing would have worked. Bottom-up beats guessing.

---

## Open questions / next steps

- Replace the static routes with OSPF single-area and compare convergence behavior when a link is downed.
- Add a default route (`ip route 0.0.0.0 0.0.0.0`) toward a simulated ISR edge and observe how gateway-of-last-resort changes the table.
- Extend to three routers to see where static routing stops scaling — each new network needs a route on every non-adjacent router.
- Configure the WAN link with a loopback-based next hop to understand recursive route lookup.

---

## Files

- `topology.png` — logical topology
- `configs/router0-running-config.txt` — full annotated config
- `configs/router1-running-config.txt` — full annotated config
- `verification/` — ping, tracert, and `show ip route` captures
- `lab.pkt` — Packet Tracer source file
