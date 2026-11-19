<div align="center">

# 🚫 Standard ACL: Blocking a Single Host with Correct Placement

### ACL Series · Lab 2 · Cisco Packet Tracer

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Packet Tracer](https://img.shields.io/badge/Packet%20Tracer-Lab-049fd9?style=for-the-badge)
![ACL](https://img.shields.io/badge/Standard-ACL-red?style=for-the-badge)
![Direction](https://img.shields.io/badge/Direction-Outbound-orange?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Verified-success?style=for-the-badge)

*Block one host. Leave everyone else alone. Placement is the whole trick.*

</div>

---

## 📑 Table of Contents
1. [Overview](#-overview)
2. [Topology](#-topology)
3. [Addressing Table](#-addressing-table)
4. [Lab 1 vs Lab 2](#-lab-1-vs-lab-2)
5. [Configuration](#-configuration)
6. [Why R2 Gi0/0 Outbound? (Placement Logic)](#-why-r2-gi00-outbound-placement-logic)
7. [Verification: Stage by Stage](#-verification-stage-by-stage)
8. [Test Matrix](#-test-matrix)
9. [Rule Order Matters](#-rule-order-matters)
10. [Key Takeaways](#-key-takeaways)
11. [Troubleshooting Notes](#-troubleshooting-notes)
12. [Next Labs](#-next-labs-in-the-acl-series)
13. [Author](#-author)

---

## 🎯 Overview

A **Standard ACL** makes its decision on the **source IP address only**. Because it cannot see the destination, **where you apply it** decides how much traffic it affects.

**Goal:** Stop host `192.168.1.3` from reaching the `200.100.100.0/24` LAN while every other host keeps full access.

| Item | Detail |
|---|---|
| ACL type | Standard, numbered (ACL 1) |
| Match method | `host` keyword (exact single source) |
| Applied on | **R2**, GigabitEthernet0/0 |
| Direction | **Outbound** (towards LAN 2) |
| Blocked host | 192.168.1.3 |
| Everyone else | Permitted (`permit any`) |

---

## 🗺 Topology

```
                         1.0.0.0 (Serial WAN)
        1.1.1.1  ●━━━━━━━━━━━━━━⚡━━━━━━━━━━━━━━●  1.1.1.2
              ┌──────┐                          ┌──────┐
              │  R1  │                          │  R2  │
              │ 1941 │                          │ 1941 │
              └──┬───┘                          └──┬───┘
        192.168.1.1/24                       200.100.100.1/24
                 │                                 │  ◄── Gi0/0: ip access-group 1 out
            ┌────┴────┐                       ┌────┴────┐
            │ Switch0 │                       │ Switch1 │
            │ 2950-24 │                       │ 2950-24 │
            └─┬──┬──┬─┘                       └─┬──┬──┬─┘
              │  │  │                           │  │  │
        ┌─────┘  │  └─────┐             ┌───────┘  │  └───────┐
     PC-PT     PC-PT    PC-PT        PC-PT      PC-PT     Server-PT
  192.168.1.4  .1.2   192.168.1.3   200.100.100.2 .100.3   200.100.100.4
   (✅ allowed)       (🚫 BLOCKED)
```

---

## 📋 Addressing Table

| Device | Interface | IP Address | Role |
|---|---|---|---|
| R1 (1941) | LAN interface | 192.168.1.1/24 | Gateway, LAN 1 |
| R1 (1941) | Serial | 1.1.1.1 | WAN link |
| R2 (1941) | Serial | 1.1.1.2 | WAN link |
| R2 (1941) | **GigabitEthernet0/0** | **200.100.100.1/24** | Gateway, LAN 2 (**ACL applied here**) |
| PC-PT | NIC | 192.168.1.4/24 | LAN 1, allowed |
| PC-PT | NIC | 192.168.1.2/24 | LAN 1, allowed |
| PC-PT | NIC | **192.168.1.3/24** | LAN 1, 🚫 **blocked host** |
| PC-PT | NIC | 200.100.100.2/24 | LAN 2 |
| PC-PT | NIC | 200.100.100.3/24 | LAN 2 |
| Server-PT | NIC | 200.100.100.4/24 | LAN 2 |

---

## 🔄 Lab 1 vs Lab 2

| | Lab 1 | Lab 2 (this lab) |
|---|---|---|
| Approach | **Whitelist** (permit one, deny rest) | **Blacklist** (deny one, permit rest) |
| Router | R1 | R2 |
| Interface | Serial0/0/0 | GigabitEthernet0/0 |
| Direction | Inbound | **Outbound** |
| Rules | `permit host 200.100.100.4` + `deny any` | `deny host 192.168.1.3` + `permit any` |
| Effect | Only the server gets through | Only one host is blocked |

---

## ⚙️ Configuration

### R1: remove the Lab 1 ACL
> Lab 1 left an ACL on R1's serial interface. Remove it first, or the replies from LAN 2 would still be denied.

```
R1(config)# interface serial 0/0/0
R1(config-if)# no ip access-group 1 in
R1(config-if)# exit
R1(config)# no access-list 1
```

### R2: the ACL

```
R2(config)# access-list 1 deny host 192.168.1.3
R2(config)# access-list 1 permit any

R2(config)# interface gigabitEthernet 0/0
R2(config-if)# ip access-group 1 out
R2(config-if)# end
```

### Routing between the two LANs
> Replace this block with the routing you actually used.

```
R1(config)# ip route 200.100.100.0 255.255.255.0 1.1.1.2
R2(config)# ip route 192.168.1.0   255.255.255.0 1.1.1.1
```

### Config breakdown

| Command | Purpose |
|---|---|
| `access-list 1 deny host 192.168.1.3` | Drop packets whose **source** is 192.168.1.3 |
| `access-list 1 permit any` | Allow every other source (without it, the implicit `deny any` blocks everyone) |
| `interface gigabitEthernet 0/0` | LAN 2 facing interface on R2 |
| `ip access-group 1 out` | Check packets **leaving** R2 through Gi0/0 |

---

## 🧠 Why R2 Gi0/0 Outbound? (Placement Logic)

**Rule:** A Standard ACL goes **as close to the destination as possible**.

A Standard ACL only knows the source, so it cannot say "block 192.168.1.3 only when it goes to LAN 2". It blocks that source **for every destination** that crosses the interface where it is applied.

```
                                   ┌─────────── R2 Gi0/0 OUT ◄── ✅ ACL HERE
                                   │             blocks .3 only towards LAN 2
 192.168.1.3 ──► R1 ──► Serial ──► R2 ──► LAN 2
        ▲
        └── 🚫 ACL near R1 would block .3 towards EVERY destination beyond R1
```

| Option | Placement | Effect on 192.168.1.3 | Verdict |
|---|---|---|---|
| A | R1 LAN-facing interface (inbound) | Blocked towards **every** destination beyond R1 | ❌ Over-blocks |
| B | R1 Serial (outbound) | Blocked towards **every** destination via the WAN | ❌ Over-blocks |
| C | **R2 Gi0/0 (outbound)** | Blocked **only** towards LAN 2 | ✅ **Chosen** |

> 💬 My reasoning: if the ACL were on R1, the router would do the routing lookup first, then the ACL, and the host could not leave R1 for anything else. I placed it next to the network I wanted to protect, where the deny is actually needed.

### Order of operations on a router

| Direction | When the ACL is checked |
|---|---|
| **Inbound** | **Before** the routing-table lookup |
| **Outbound** | **After** the routing-table lookup, as the packet leaves |

Outbound ACLs also do **not** filter traffic generated by the router itself.

---

## ✅ Verification: Stage by Stage

### 🔎 Stage 1: Confirm the ACL (run on R2)

```
R2# show access-lists
R2# show ip interface gigabitEthernet 0/0 | include access list
```

Expected:
```
Standard IP access list 1
    10 deny host 192.168.1.3 (x match(es))
    20 permit any (x match(es))
  Outbound access list is 1
```
> 📸 Paste your own counters here after testing.

---

### 🔴 Stage 2: PC2 (192.168.1.3) → 200.100.100.2 → BLOCKED

```
C:\> ping 200.100.100.2

Reply from 1.1.1.2: Destination host unreachable.
Reply from 1.1.1.2: Destination host unreachable.
Reply from 1.1.1.2: Destination host unreachable.
Reply from 1.1.1.2: Destination host unreachable.

Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

**Annotations**
- The packet matched `deny host 192.168.1.3` as it tried to leave R2 Gi0/0.
- The unreachable came from **1.1.1.2**, which is R2's serial interface. This proves the packet **crossed R1 and the WAN** and was dropped at **R2**, not at R1.

---

### 🔴 Stage 3: PC2 (192.168.1.3) → 200.100.100.3 → BLOCKED

```
C:\> ping 200.100.100.3

Reply from 1.1.1.2: Destination host unreachable.  (x4)

Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

**Annotation:** The same result for the second LAN 2 host. The ACL protects the **whole** outbound side of Gi0/0, not one destination.

---

### 🟢 Stage 4: PC0 (192.168.1.4) → 200.100.100.2 → ALLOWED

```
C:\> ping 200.100.100.2

Reply from 200.100.100.2: bytes=32 time=16ms TTL=126
Reply from 200.100.100.2: bytes=32 time=1ms  TTL=126
Reply from 200.100.100.2: bytes=32 time=1ms  TTL=126
Reply from 200.100.100.2: bytes=32 time=1ms  TTL=126

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
Minimum = 1ms, Maximum = 16ms, Average = 4ms
```

**Annotations**
- ✅ Source 192.168.1.4 does not match the deny line, so it falls through to `permit any`.
- **TTL = 126**: 128 minus 2 routers (R1 and R2) shows the packet crossed both.
- The first reply (16 ms) is slower because of ARP and initial path setup.

---

### 🟢 Stage 5: PC0 (192.168.1.4) → 200.100.100.3 → ALLOWED

```
C:\> ping 200.100.100.3

Reply from 200.100.100.3: bytes=32 time=1ms TTL=126  (x4)

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
Minimum = 1ms, Maximum = 1ms, Average = 1ms
```

**Annotation:** Every host except 192.168.1.3 is unaffected, which was the goal.

---

## 🧪 Test Matrix

| # | Source | Destination | Result | Why |
|---|---|---|---|---|
| 1 | 192.168.1.3 | 200.100.100.2 | ❌ Unreachable (from 1.1.1.2) | Matches `deny host`, dropped at R2 Gi0/0 out |
| 2 | 192.168.1.3 | 200.100.100.3 | ❌ Unreachable (from 1.1.1.2) | Same rule |
| 3 | 192.168.1.4 | 200.100.100.2 | ✅ 4/4, TTL 126 | Falls to `permit any` |
| 4 | 192.168.1.4 | 200.100.100.3 | ✅ 4/4, TTL 126 | Falls to `permit any` |
| 5 | 192.168.1.3 | 200.100.100.4 (server) | ❌ Expected | Same outbound ACL |
| 6 | 192.168.1.2 | any LAN 2 host | ✅ Expected | Not matched by the deny |

> Rows 5 and 6 follow from the ACL logic. Add your own screenshots to confirm them.

---

## 🔃 Rule Order Matters

ACLs are processed **top-down, first match wins**.

| Config | Result |
|---|---|
| `deny host 192.168.1.3` then `permit any` | ✅ Correct: only .3 blocked |
| `permit any` then `deny host 192.168.1.3` | ❌ Everyone permitted, the deny is never reached |
| `deny host 192.168.1.3` only | ❌ Everyone blocked by the implicit `deny any` |

---

## 💡 Key Takeaways

| # | Takeaway |
|---|---|
| 1 | Standard ACLs match the **source IP only** |
| 2 | Place Standard ACLs **close to the destination** to avoid over-blocking |
| 3 | Outbound ACLs are checked **after** the routing lookup, inbound **before** it |
| 4 | `deny host` blocks one device, and `permit any` keeps everyone else working |
| 5 | Put the specific deny **first**, then the permit |
| 6 | The source of the ICMP unreachable shows **which router** dropped the packet |
| 7 | Whitelist (Lab 1) and blacklist (Lab 2) solve opposite problems |
| 8 | ACL numbers are local to each router, so ACL 1 on R1 and ACL 1 on R2 are unrelated |

---

## 🛠 Troubleshooting Notes

| Symptom | Check |
|---|---|
| Nobody can reach LAN 2 | Is `permit any` missing? |
| 192.168.1.3 still gets through | Is the direction `out`? Is the ACL on Gi0/0? Is the deny line first? |
| Everything is blocked, even .4 | Is the old Lab 1 ACL still on R1 Se0/0/0? |
| No hit counters | Is the traffic actually leaving R2 through Gi0/0? |

---

## 🚀 Next Labs in the ACL Series

- [x] Lab 1: Standard ACL, whitelist a host ✅
- [x] Lab 2: Standard ACL, blacklist a host with placement logic ✅
- [ ] Standard ACL with network / wildcard mask ranges
- [ ] Named Standard ACL (edit with sequence numbers)
- [ ] Extended ACL
- [ ] VTY (Telnet/SSH) access restriction with ACL

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-certified Network & NOC Engineer · Lower Dir, KPK, Pakistan

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github)](https://github.com/maazkhanms)

⭐ If this lab helped you, give the repo a star!
