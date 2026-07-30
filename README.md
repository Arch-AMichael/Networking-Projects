# Network Labs

A working record of hands-on networking labs — configuration, verification, and the troubleshooting that happened along the way. Each lab is documented the way I comment code: the notes explain *why* a decision was made and *what broke*, not just the final commands. The dead ends are kept on purpose; they're where the learning is.

Building toward **CCNA**. Platform is Cisco Packet Tracer plus a physical home lab (pfSense, Windows Server AD/DNS/DHCP/GPO, Ubuntu, Kali, VirtualBox/VMware).

---

## Labs

| # | Lab | Focus | Key concepts | Status |
|---|-----|-------|--------------|--------|
| 05 | [Lab 05 - Static Routing Between Two Routers](./05-static-routing-two-router/) | Point-to-point static routing | Next-hop = far end of link, return-path symmetry, `/30` WAN sizing, TTL as path evidence | ✅ Complete |
| 06 | [Static Route Summarization (Four-Router Chain)](./06-static-route-summarization/) | Multi-hop static routing + summarization | Route aggregation, longest-prefix match, `ping` result chars (`!` `.` `U`), summary trade-offs | ✅ Complete |

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

## Recurring themes so far

- **A static route's next hop is always the *adjacent* interface** — the other end of a directly connected link, never an address on the local router or one several hops away. This has bitten me in two different disguises (an explicit `%Invalid next hop` error, and a silently useless route that threw no error at all).
- **One-way ping failure usually means a missing *return* route**, not a broken forward path. Reading the ping result characters — `!` success, `.` timeout, `U` unreachable — points straight at which half is missing.
- **Save the config.** `copy running-config startup-config` at session end, every time. Still building the reflex.

---

## Next up

- Static → OSPF conversion on the Lab 06 chain, to compare convergence and link-failure behavior against static summaries.
- Tighter summarization (`/19` vs. the `/16` used in Lab 06) and demonstrating the loose-summary loop.
- VLAN segmentation and inter-VLAN routing.
