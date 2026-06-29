# Day 013 — Consistent Hashing

---

## What is Consistent Hashing?

Consistent Hashing is a technique for **distributing data across multiple servers while minimising data movement** when servers are added or removed.

It's one of the most important concepts in distributed systems — used everywhere from caches to databases to CDNs.

**Used in:** Apache Cassandra, Amazon DynamoDB, Redis Cluster, Memcached, Akamai CDN

---

## The Problem with Normal Hashing

The naive approach to distributing data across N servers:

```
server = hash(key) % N
```

This works fine — until you add or remove a server.

**Example with 4 servers:**
```
hash(Rahul) = 15
15 % 4 = 3  →  Store on Server 3
```

**Now add a 5th server:**
```
15 % 5 = 0  →  Rahul moves to Server 0
```

Almost every key maps to a different server. Nearly all data must be moved.

**The real cost at scale:**
```
100 TB of data + add 1 server
= move ~80–90 TB of data
= massive network traffic + downtime + cost
```

This is why modulo hashing doesn't work for distributed systems.

---

## The Solution — Hash Ring

Instead of using modulo, **place both servers and keys on a circle** (hash ring). The ring represents hash values from 0 to 360°.

```
              0° / 360°
            ↗           ↖
         330°              30°

      300°                    60°

      270°                    90°

         240°              120°

            ↙           ↘
              210°    150°
                  180°
```

**Step 1 — Place servers on the ring:**
```
hash(Server A) = 20°
hash(Server B) = 120°
hash(Server C) = 220°
hash(Server D) = 320°

Ring:
              A (20°)
            ↗           ↖
         D (320°)          B (120°)
            ↙           ↘
              C (220°)
```

**Step 2 — Place keys on the ring:**
```
hash(Rahul) = 40°
hash(Alex)  = 250°
hash(David) = 340°
```

**Step 3 — Assign each key to the first server clockwise:**
```
Rahul (40°)  →  clockwise → hits B (120°)  →  Server B
Alex  (250°) →  clockwise → hits D (320°)  →  Server D
David (340°) →  clockwise → wraps to A (20°) → Server A
```

**Rule:** Every key belongs to the first server encountered going clockwise on the ring.

---

## Full Assignment Example

```
Servers on ring:   A=20°,  B=120°,  C=220°,  D=320°

Key    Position    Clockwise to    Server
────   ────────    ────────────    ──────
K1     40°         120° (B)        B
K2     90°         120° (B)        B
K3     170°        220° (C)        C
K4     250°        320° (D)        D
K5     340°        20°  (A)        A  ← wraps around
```

---

## Adding a Server

**Add Server E at 160°:**

```
Before:  ... B(120°) ────────────── C(220°) ...
             K3 at 170° → C

After:   ... B(120°) ── E(160°) ── C(220°) ...
             K3 at 170° → E
```

Only the keys between B(120°) and E(160°) move to E. Everything else stays exactly where it is.

```
With 4 servers → adding 1 server moves ~1/5 of data
With 10 servers → adding 1 server moves ~1/11 of data
```

Compare to modulo hashing where adding 1 server moves ~80–90% of data.

---

## Removing a Server

**Remove Server C (220°):**

```
Before:  K3(170°) → C,  K4(180°) → C,  K5(210°) → C
After:   K3(170°) → D,  K4(180°) → D,  K5(210°) → D
```

Only C's keys move to the next server clockwise (D). All other keys are untouched.

---

## The Hotspot Problem

Even with consistent hashing, servers may end up unevenly placed:

```
Ring:   A(10°) ─ B(20°) ──────────────────────── C(300°)

Server A: tiny slice (10°–20°)     → very little data
Server C: huge slice (20°–300°)    → most of the data
```

This causes uneven storage, CPU load, and request distribution.

---

## Solution — Virtual Nodes (VNodes)

Instead of placing each server once on the ring, place it **multiple times** at different positions:

```
Instead of:   A at 1 position
Place:         A1, A2, A3, A4 at 4 different positions

Ring becomes:
  A1 → B1 → C1 → A2 → B2 → C2 → A3 → B3 → C3 → A4 → ...
```

**Without VNodes:**
```
Server A: 40% of keys
Server B: 10% of keys   ← underloaded
Server C: 50% of keys   ← overloaded
```

**With VNodes:**
```
Server A: 33% of keys
Server B: 34% of keys
Server C: 33% of keys   ← balanced
```

### Benefits of Virtual Nodes

| Benefit | Why |
|---|---|
| Better load balancing | Keys spread more uniformly |
| Better fault tolerance | Failure spreads load across many servers, not one neighbour |
| Easy scaling | New server takes a few VNodes from each existing server |
| Better storage utilisation | No single server becomes a hotspot |

---

## Normal Hashing vs Consistent Hashing

| | Normal Hashing (`hash % N`) | Consistent Hashing |
|---|---|---|
| Add a server | ~80–90% data moves | ~1/N data moves |
| Remove a server | ~80–90% data moves | Only that server's keys move |
| Hotspot risk | High | Reduced with VNodes |
| Implementation | Simple | More complex |
| Good for static clusters | ✅ Yes | ✅ Yes |
| Good for dynamic clusters | ❌ No | ✅ Yes |

---

## Time Complexity

Using a balanced binary search tree to maintain the hash ring:

| Operation | Complexity |
|---|---|
| Find server for a key | O(log N) |
| Add a server | O(log N) |
| Remove a server | O(log N) |

---

## Where Consistent Hashing is Used

| System | How it uses Consistent Hashing |
|---|---|
| Apache Cassandra | Data partitioning across nodes |
| Amazon DynamoDB | Key distribution across partitions |
| Redis Cluster | Sharding data across nodes |
| Memcached | Cache key distribution |
| CDN (Akamai) | Routing requests to nearest edge server |
| Load Balancers | Session affinity / sticky routing |

---

## Common Interview Questions

**Q: Why not use `hash(key) % N`?**
Adding or removing one server changes N, causing ~80–90% of all keys to remap. At scale this means massive data migration, downtime, and cost.

**Q: One server is getting much more traffic than others. Why?**
Servers are unevenly placed on the hash ring, creating partitions of different sizes. Fix: use virtual nodes so each physical server owns multiple small ring segments.

**Q: A server crashes. What happens?**
Only the keys owned by the crashed server are reassigned — to the next server clockwise. All other servers and keys are completely unaffected.

**Q: You add 5 servers to a 50-server cluster. How much data moves?**
Only the key ranges that the new servers take over. With consistent hashing, this is roughly 5/55 (~9%) of data. With modulo hashing it would be nearly everything.

**Q: What if hash values aren't uniformly distributed?**
Some servers become hotspots. Fix: use a high-quality hash function (MurmurHash, MD5) and introduce virtual nodes.

---

## Quick Reference Cheat Sheet

| Concept | One-Line Summary |
|---|---|
| Hash Ring | Circle where both servers and keys are placed by hash value |
| Clockwise Rule | Each key belongs to the first server clockwise on the ring |
| VNodes | One physical server = many virtual positions on the ring |
| Data movement | Add/remove server → only ~1/N of keys move |
| Modulo hashing | Add/remove server → ~80–90% of keys move |
| Hotspot | One server gets disproportionately more data or traffic |

---

## One-Line Formula

```
Consistent Hashing = Hash Ring + Clockwise Assignment + Minimal Data Movement
Virtual Nodes      = Multiple Ring Positions per Server + Uniform Distribution
```

---

## Summary

```
Normal Hashing:       hash(key) % N
                      ❌ Add/remove server → almost everything remaps

Consistent Hashing:   Both keys and servers on a hash ring
                      ✅ Add/remove server → only ~1/N keys move
                      ✅ Scales horizontally with minimal disruption

Virtual Nodes:        Each server = multiple ring positions
                      ✅ Uniform load distribution
                      ✅ No hotspots
                      ✅ Better fault tolerance
```
