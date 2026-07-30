# Lab 06 — Connecting Four Routers and Simplifying Their Directions

**Date:** July 24–25, 2026
**Tools:** Cisco Packet Tracer (a free program that lets you build pretend networks on your computer), four routers
**Status:** Finished and working

---

## The quick version

I connected four routers in a row and got a message to travel from one end all the way to the other. Then I cleaned up the instructions each router was using so that a long list of directions became a single, simpler one — without breaking anything.

---

## A few words you'll need first

**Router** — a device that passes traffic between different groups of computers. Picture a post office that receives a package and decides which road to send it down next.

**Network** — a group of devices that live "on the same street." Every device on a street has an address that starts the same way. In this lab the streets are named 192.168.10, 192.168.20, and 192.168.30. So 192.168.10.1 and 192.168.10.2 are two houses on the same street.

**IP address** — a device's mailing address, written as four numbers, like 192.168.10.1.

**Route** — a direction card inside a router. It says "to reach *that* street, hand the package to *this* neighbor."

**Ping** — a way to test if you can reach another device. It's like knocking on a door and listening for someone to answer. If they answer, you're connected.

---

## What the setup looked like

Four routers in a line. I'll call them R4, R5, R6, and R7. Between each pair of routers is one street:

```
   Street 10           Street 20           Street 30
[ R4 ]----------[ R5 ]----------[ R6 ]----------[ R7 ]
```

- **R4** sits at the far left. It touches only Street 10.
- **R5** is in the middle. It touches Street 10 and Street 20.
- **R6** is in the middle. It touches Street 20 and Street 30.
- **R7** sits at the far right. It touches only Street 30.

The goal: let a message start at R4, travel through R5 and R6, and arrive at R7 — and back again.

---

## The addresses I gave each router

Each router needs an address on every street it touches.

| Router | Street it's on | Address I gave it |
|--------|----------------|-------------------|
| R4 | Street 10 | 192.168.10.1 |
| R5 | Street 10 | 192.168.10.2 |
| R5 | Street 20 | 192.168.20.1 |
| R6 | Street 20 | 192.168.20.2 |
| R6 | Street 30 | 192.168.30.1 |
| R7 | Street 30 | 192.168.30.2 |

---

## The core idea of the whole lab

A router automatically knows about the streets it's directly sitting on. It does **not** know how to reach streets further away. You have to tell it.

Here's who needs directions to where:

| Router | Streets it already knows | Streets I had to give it directions for |
|--------|--------------------------|------------------------------------------|
| R4 | Street 10 | Street 20 and Street 30 |
| R5 | Street 10, Street 20 | Street 30 |
| R6 | Street 20, Street 30 | Street 10 |
| R7 | Street 30 | Street 10 and Street 20 |

**The one rule that matters most:** when you give a router directions, you tell it to hand the package to its *next-door neighbor* — the router directly across the street. You can only hand a package to someone you can actually reach. You can't skip ahead to a router two streets away.

---

## Step 1 — Turn on the last road

Two of the routers (R6 and R7) hadn't been switched on for Street 30 yet, so that road didn't exist. I gave each of them an address there and turned the connection on. In router language:

```
interface GigabitEthernet0/0/1
 ip address 192.168.30.1 255.255.255.0
 no shutdown
```

`no shutdown` is the command that turns a connection on. (Yes, "no shutdown" means "turn on." Router language is like that sometimes.)

---

## Step 2 — Give each router its directions (the long way)

At first I wrote out a separate direction card for every faraway street. Each card follows the same pattern:

> "To reach `<that street>`, hand the package to `<my neighbor's address>`."

**R4** needs to reach Streets 20 and 30. Both are off to its right, through R5:

```
ip route 192.168.20.0 255.255.255.0 192.168.10.2
ip route 192.168.30.0 255.255.255.0 192.168.10.2
```

(192.168.10.2 is R5 — R4's neighbor.)

**R5** only needs Street 30, which is through R6:

```
ip route 192.168.30.0 255.255.255.0 192.168.20.2
```

**R6** only needs Street 10, which is through R5:

```
ip route 192.168.10.0 255.255.255.0 192.168.20.1
```

**R7** needs Streets 10 and 20, both off to its left through R6:

```
ip route 192.168.10.0 255.255.255.0 192.168.30.1
ip route 192.168.20.0 255.255.255.0 192.168.30.1
```

After this, a message could travel all the way from R4 to R7 and back. It worked.

---

## Step 3 — Replace the long list with one simple direction

Some routers now had two or three direction cards. But notice something: all three streets (10, 20, 30) start with the same two numbers, **192.168**. That means I can write one card that says "anything starting with 192.168, send it this way" instead of listing each street separately. This shortcut is called **summarization** — combining several directions into one.

So on each router I erased the specific cards and wrote a single one:

```
R4:  ip route 192.168.0.0 255.255.0.0 192.168.10.2
R5:  ip route 192.168.0.0 255.255.0.0 192.168.20.2
R6:  ip route 192.168.0.0 255.255.0.0 192.168.20.1
R7:  ip route 192.168.0.0 255.255.0.0 192.168.30.1
```

The `255.255.0.0` part is what turns "192.168.10 only" into "192.168.anything." Each router still points its one card toward the rest of the network.

**Won't this confuse the router about its own streets?** No — and this is the neat part. A router always prefers the *most specific* instruction it has. It still knows Street 20 is right outside its door, so for Street 20 it uses that exact knowledge. The broad "192.168.anything" card only kicks in for streets it has no closer information about. Specific beats general, every time.

The result: same connectivity as before, but each router now carries one tidy direction instead of a stack.

---

## How I checked it was working

I used ping — knocking on doors and listening. Ping gives back one of three signals for each knock, and learning to read them is half the skill:

| Signal | What it means |
|--------|---------------|
| `!` | Someone answered. Success. |
| `.` | Silence. No answer came back. |
| `U` | Someone along the way shouted "I don't know how to get there." |

A good result looks like five exclamation marks:

```
!!!!!
Success rate is 100 percent (5/5)
```

Early on, before I'd finished the directions, one test came back like this:

```
U.U.U
Success rate is 0 percent (0/5)
```

Those `U`s were the important clue. They meant my message *was* reaching the far routers — but those routers didn't know how to send the reply *back*. The problem wasn't the trip out; it was the trip home. That told me exactly what to fix.

One more thing worth knowing: the very first knock often gets a `.` even when everything's fine, like this:

```
.!!!!
Success rate is 80 percent (4/5)
```

That first miss is normal. The router just needs a moment to look up where the door is, and every knock after the first one succeeds. It's not a real problem — knowing the difference between this and an actual failure saves you from chasing ghosts.

---

## Things that went wrong (and what fixed them)

Keeping these on purpose — the mistakes are where the learning is.

| What happened | Why | Fix |
|---------------|-----|-----|
| Street 30 didn't work at all | Two routers weren't switched on for that road | Gave them addresses and turned the connection on |
| Gave R7 a direction, but it did nothing | I told R7 to hand packages to a router that *isn't* its neighbor — two streets away, unreachable | Pointed it at its actual next-door router instead |
| Couldn't erase a direction card | I described the card differently than how it was written | Erased it by describing it exactly |
| "Inconsistent address and mask" error | I typed a street name with a house number stuck on the end | Used the plain street name (ending in .0) |
| Warning about a repeated address | Two devices briefly claimed the same address while I was setting up | Sorted itself out once the addresses settled |
| Message reached the far side but got no reply | The far routers were missing their return directions | Added the missing directions on those routers |
| A command got rejected | I tried to run a "knock on the door" test from the wrong menu inside the router | Ran it from the right menu |
| Typos like "not shutdown" and "piong" | Fast typing | Fixed the spelling |

---

## What I'd tell someone else after doing this

- **Always hand a package to your actual next-door neighbor.** You can't skip ahead to someone further down the line — the message has no way to reach them directly. This tripped me up and it's the single most important idea here.
- **If your message goes out but nothing comes back, the problem is usually the trip home, not the trip out.** Check whether the other side knows how to reply.
- **One broad instruction can replace a whole stack of specific ones** — and the router is smart enough to still use a specific instruction when it has one.
- **Read the answer carefully.** A `U` (someone's lost) and a `.` (total silence) mean different things and point you to different problems.
- **Save your work** before closing, so the router remembers everything after a restart.

---

## What I want to try next

- Let the routers figure out the directions *themselves* automatically (a smarter system called OSPF) instead of me writing every card by hand, and see how it reacts when a road goes down.
- Make the broad "send everything this way" direction even tighter, and show the one situation where being too broad can accidentally trap a message in a loop.
- Move on to splitting one physical network into several separate virtual ones (called VLANs).

---

## Files in this folder

- `topology.png` — a picture of the four routers and three streets
- `configs/` — the full settings copied from each router
- `verification/` — screenshots of the ping tests
- `lab.pkt` — the Packet Tracer file, so anyone can open and explore the setup