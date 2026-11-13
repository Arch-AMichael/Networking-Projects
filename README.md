# Network Labs

A working record of hands-on networking labs — configuration, verification, and the troubleshooting that happened along the way. Each lab is documented the way I comment code: the notes explain *why* a decision was made and *what broke*, not just the final commands. The dead ends are kept on purpose; they're where the learning is.

Building toward **CCNA**. Platform is Cisco Packet Tracer plus a physical home lab (pfSense, Windows Server AD/DNS/DHCP/GPO, Ubuntu, Kali, VirtualBox/VMware).

---

## Labs

| # | Lab | Focus | Key concepts | Status |
|---|-----|-------|--------------|--------|
| 05 | [Static Routing Between Two Routers](https://github.com/Arch-AMichael/Networking-Projects/tree/main/Lab%2005%20-%20Static%20Routing%20Between%20Two%20Routers) | Point-to-point static routing | Next-hop = far end of link, return-path symmetry, `/30` WAN sizing, TTL as path evidence | ✅ Complete |
| 06 | [Static Route Summarization (Four-Router Chain)](https://github.com/Arch-AMichael/Networking-Projects/tree/main/Lab%2006%20%E2%80%94%20Static%20Route%20Summarization%20Across%20a%20Four-Router%20Chain) | Multi-hop static routing + summarization | Route aggregation, longest-prefix match, `ping` result chars (`!` `.` `U`), summary trade-offs | ✅ Complete |
| 07 | [Dynamic Routing with OSPF (Five Routers)](https://github.com/Arch-AMichael/Networking-Projects/tree/main/Lab%2007%20%E2%80%94%20%20Dynamic%20Routing%20%28OSPF%29) | OSPF single-area, redundant paths | Adjacency states and `FULL`, wildcard masks, equal-cost load balancing, automatic failover | ✅ Complete |
| 08 | [VLANs, Trunking, and Router-on-a-Stick](https://github.com/Arch-AMichael/Networking-Projects/tree/main/Lab%2008%20%E2%80%94%20Vlan%20Trunking%20Inter%20Vlan%20Routing) | L2 segmentation + inter-VLAN routing | Access vs. trunk ports, 802.1Q, subinterfaces, `/26` subnetting, native VLAN, `allowed vlan` vs `add` | ✅ Complete |
| 09 | [OSPF Across Three Layer 3 Switches](https://github.com/Arch-AMichael/Networking-Projects/tree/main/Lab%2009%20%E2%80%94%20OSPF%20Across%20Three%20Layer%203%20Switches) | Distributed inter-VLAN routing | SVIs, `ip routing`, `no switchport` routed ports, manual router-IDs, `passive-interface`, ECMP on a triangle | ✅ Complete |
| 10 | [Multi-Area OSPF, Inter-VLAN Routing and DHCP Relay](https://github.com/Arch-AMichael/Networking-Projects/tree/main/Lab10%20%E2%80%94%20%20Multi-Area%20OSPF%20%26%20Inter-VLAN%20Routing%20Project) | Three OSPF areas + central DHCP | ABRs, `O` vs `O IA`, area mismatch errors, SVIs + router-on-a-stick together, `ip helper-address` and `giaddr`, APIPA as a diagnosis | ✅ Complete · [video series](https://www.youtube.com/@TechwithObinna) |

---

## How each lab is organized

```
NN-lab-name/
├── README.md          ← objective, addressing, config (with "why"), verification, troubleshooting log
├── topology.png       ← logical topology
├── configs/           ← annotated show running-config per device
├── verification/      ← ping / tracert / show ip route captures
└── lab.pkt            ← Packet Tracer source
```

Every README carries a **troubleshooting log** — a table of symptom → actual cause → fix. That section is the point of the whole exercise: it's where a config that "just worked" turns into transferable diagnostic experience.

---

## The arc so far

The labs build on each other deliberately — each one is the previous one's "next steps" section, executed.

**05 → 06:** two routers become four. Static routing works, then starts to not scale, and summarization is the response.

**06 → 07:** the same physical chain, closed into a loop and handed to OSPF. Static summaries can't use a redundant path or survive a link cut; OSPF does both with no manual intervention. Lab 07's failover test is the argument for dynamic routing in one screenshot.

**07 → 08:** down a layer. One physical switch becomes four broadcast domains, carried over 802.1Q trunks, routed by a single router on subinterfaces.

**08 → 09:** the routing function moves onto the switches themselves. Six VLANs across three multilayer switches, each routing its own VLANs locally and running OSPF over routed `/30` links between them — Lab 08's segmentation and Lab 07's protocol, combined, with Lab 08's single-router bottleneck designed out.

**09 → 10:** two of Lab 09's "next steps," built together. The single OSPF area splits into three, joined by two ABRs, and `O IA` routes appear for the first time. Both styles of inter-VLAN routing run side by side (Layer 3 switches at the edges, router-on-a-stick in the core), and static host addressing is replaced by one DHCP server relayed to all six VLANs. Recorded as a 3-part video series.

---

## Recurring themes so far

- **A static route's next hop is always the *adjacent* interface** — the other end of a directly connected link, never an address on the local router or one several hops away. This has bitten me in two different disguises (an explicit `%Invalid next hop` error, and a silently useless route that threw no error at all).
- **One-way ping failure usually means a missing *return* route**, not a broken forward path.
- **Read what the output is actually saying, not just whether it worked.** This keeps recurring in new forms: `U` (a router reporting no route) vs `.` (silence) in Lab 06; `Destination host unreachable` *from your own gateway* (a local, control-plane problem, provably one hop away) vs `Request timed out` in Lab 09. Each distinction cuts the search space immediately.
- **TTL counts hops, and hop count reveals architecture.** 128/127/126 in Lab 09 maps directly onto switched / locally-routed / crossed-the-backbone. It's the fastest way to confirm traffic is taking the path the design intended.
- **The first packet usually fails, and it means nothing.** ARP resolution on packet 1 produces a `.` or a timeout in nearly every capture. Telling this apart from real loss is a skill in itself.
- **Aggregation is one idea in many syntaxes.** A static `/16` summary (06), an OSPF `network` wildcard (07), and a summarized backbone statement (09) are all the same concept wearing different masks.
- **The quietest failures are the wrong device doing (or not doing) a job.** Lab 10 had three: a Layer 2 switch answering ARP for the gateway, a DHCP server with no route to send replies, and a relay command on interfaces that never see the broadcast. None logged an error. The loud failure — an OSPF area mismatch — logged every 10 seconds.
- **Save the config.** `copy running-config startup-config` at session end, every time. Reflex now established as of Lab 07.

---

## Next up

- **Failover testing on the Lab 09 triangle** — cut a backbone link during a continuous ping and count the packets lost to reconvergence. The redundancy is built but not yet proven.
- **`auto-cost reference-bandwidth`** — Lab 09 exposed that default OSPF cost can't distinguish Gigabit from FastEthernet. Fix it and watch path selection change.
- **Python automation (Netmiko)** — Lab 09's build is six VLANs × three switches of near-identical config, which is exactly the shape worth generating from a template instead of typing.
- **OSPF area summarization and stub areas** on Lab 10's ABRs — one route per area across the backbone instead of one per VLAN.
- **Access-layer hardening** — port security, unused-port shutdown, native VLAN verification.
