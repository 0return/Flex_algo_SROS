# Lab_Flex_algo

### IGP Flexible Algorithm on Nokia SR OS — one BGP next hop, four L3VPNs, four different paths

![Nokia SR OS](https://img.shields.io/badge/Nokia-SR%20OS%20classic%20CLI-124191)
![Segment Routing](https://img.shields.io/badge/SR--MPLS-IS--IS%20L2-2f9e44)
![Flex-Algo](https://img.shields.io/badge/Flex--Algo-128%20%7C%20129%20%7C%20130-e8590c)
![Containerlab](https://img.shields.io/badge/Containerlab-SR--SIM-1c7ed6)

Four VPRNs between PE1 and PE2 share the same BGP next hop (`2.2.2.2`), yet each one crosses the network over a different path. Each VPRN resolves that next hop with a different IS-IS algorithm, and the lab shows **why** each path is chosen and how each service reacts when a link fails.

| Algorithm | Optimizes / avoids | Service | Path PE1 → PE2 |
| --- | --- | --- | --- |
| 0 | IGP metric | `INTERNET-A0` | PE1 – P1 – P2 – PE2 |
| 128 | Delay | `VOZ-A128` | PE1 – P6 – P7 – PE2 |
| 129 | TE metric | `BULK-A129` | PE1 – P3 – P4 – P5 – PE2 |
| 130 | IGP metric, excluding RED links | `BANCO-A130` | PE1 – P3 – P4 – P7 – PE2 |

---

## Table of contents

- [What you will learn](#what-you-will-learn)
- [Topology](#topology)
- [Design](#design)
- [Flex-Algo definitions and SIDs](#flex-algo-definitions-and-sids)
- [Services](#services)
- [Key configuration (classic CLI)](#key-configuration-classic-cli)
- [Verification](#verification)
- [Failure test: P3–P4 down](#failure-test-p3p4-down)
- [Lessons learned while building it](#lessons-learned-while-building-it)
- [Next experiments](#next-experiments)
- [Deploying the lab](#deploying-the-lab)
- [References](#references)

---

## What you will learn

- What a Flexible Algorithm Definition (FAD) is, why one or two nodes advertising it is enough, and how the FAD election works.
- Why every algorithm needs its own prefix node-SID: the label is the only thing that tells a P router which SPF to use.
- How the path changes with the metric type (IGP, TE, delay) and with admin-group constraints.
- How SR OS binds a service to an algorithm: a `vrf-import` policy with `action flex-algo`.
- How each algorithm reconverges independently when a link fails.

---

## Topology


<img width="907" height="320" alt="image" src="https://github.com/user-attachments/assets/a48a866d-d0e8-404f-bee3-91152e846dff" />



Three parallel routes (north, middle, south) plus one cross link (P4–P7).

- IS-IS area `49.0001`, **Level 2 only** on all nodes
- SRGB `100000–100999`
- iBGP VPN-IPv4 between PE1 and PE2, AS `65000`

| Link | Subnet | Side A (interface, port, IP) | Side B (interface, port, IP) |
| --- | --- | --- | --- |
| PE1–P1 | 10.0.1.0/30 | PE1 `toP1` 1/1/c1/1 .1 | P1 `toPE1` 1/1/c1/1 .2 |
| P1–P2 | 172.16.1.0/30 | P1 `toP2` 1/1/c2/1 .1 | P2 `toP1` 1/1/c1/1 .2 |
| P2–PE2 | 10.0.2.0/30 | P2 `toPE2` 1/1/c2/1 .2 | PE2 `toP2` 1/1/c1/1 .1 |
| PE1–P3 | 10.0.3.0/30 | PE1 `toP3` 1/1/c2/1 .1 | P3 `toPE1` 1/1/c1/1 .2 |
| P3–P4 | 172.16.2.0/30 | P3 `toP4` 1/1/c2/1 .1 | P4 `toP3` 1/1/c1/1 .2 |
| P4–P5 | 172.16.3.0/30 | P4 `toP5` 1/1/c2/1 .1 | P5 `toP4` 1/1/c1/1 .2 |
| P5–PE2 | 10.0.5.0/30 | P5 `toPE2` 1/1/c2/1 .2 | PE2 `toP5` 1/1/c2/1 .1 |
| PE1–P6 | 10.0.6.0/30 | PE1 `toP6` 1/1/c3/1 .1 | P6 `toPE1` 1/1/c1/1 .2 |
| P6–P7 | 172.16.4.0/30 | P6 `toP7` 1/1/c2/1 .1 | P7 `toP6` 1/1/c1/1 .2 |
| P7–PE2 | 10.0.7.0/30 | P7 `toPE2` 1/1/c2/1 .2 | PE2 `toP7` 1/1/c3/1 .1 |
| P4–P7 | 172.16.5.0/30 | P4 `toP7` 1/1/c3/1 .2 | P7 `toP4` 1/1/c3/1 .1 |

---

## Design

Each route has a "story" that justifies its values. Every value is configured on **both ends** of the link.

| Link | IGP | TE | Delay (µs) | Admin-group | Story |
| --- | --- | --- | --- | --- | --- |
| PE1–P1 | 10 | 100 | 5000 | — | North: few hops, leased capacity (expensive TE), long distance |
| P1–P2 | 10 | 100 | 5000 | RED | Segment leased from provider X |
| P2–PE2 | 10 | 100 | 5000 | — | |
| PE1–P6 | 10 | 50 | 1000 | — | Middle: direct low-latency fiber |
| P6–P7 | 50 | 50 | 2000 | — | Low-capacity express: high IGP so best effort avoids it |
| P7–PE2 | 20 | 50 | 1000 | — | |
| PE1–P3 | 10 | 10 | 3000 | — | South: own fiber (cheap TE) |
| P3–P4 | 10 | 10 | 3000 | — | |
| P4–P5 | 10 | 10 | 3000 | RED | Segment leased from provider X |
| P5–PE2 | 10 | 10 | 3000 | — | |
| P4–P7 | 10 | 10 | 2000 | — | Cross link, south to middle |

### Expected paths (PE1 → PE2)

| Path | Hops | IGP | TE | Delay | Uses RED? | Chosen by |
| --- | --- | --- | --- | --- | --- | --- |
| N: PE1–P1–P2–PE2 | 3 | **30** | 300 | 15 ms | Yes | Algo 0 |
| M: PE1–P6–P7–PE2 | 3 | 80 | 150 | **4 ms** | No | Algo 128 |
| S: PE1–P3–P4–P5–PE2 | 4 | 40 | **40** | 12 ms | Yes | Algo 129 |
| X1: PE1–P3–P4–P7–PE2 | 4 | **50** | 80 | 9 ms | No | Algo 130 |
| X2: PE1–P6–P7–P4–P5–PE2 | 5 | 90 | 130 | 11 ms | Yes | Nobody (appears under failure) |

Algo 130 first prunes the RED links (N, S and X2 disappear), then picks the lowest IGP among what is left: X1 (50) over M (80). Algo 129 and algo 130 leave PE1 through the same neighbor (P3) and split at P4: **every node runs its own SPF per algorithm**.

---

## Flex-Algo definitions and SIDs

FADs are defined and advertised by PE1 and PE2 only. All nine nodes participate in the three algorithms.

| Algo | FAD name | metric-type | Constraint | Priority |
| --- | --- | --- | --- | --- |
| 128 | `128-delay` | delay | — | 100 |
| 129 | `129-TE` | te-metric | — | 100 |
| 130 | `130-NO-RED` | igp | exclude `RED` (value 1 → mask `0x2`) | 100 |

Both PEs advertise identical FADs with the same priority, so the tie is broken by the **highest system ID**: PE2 (`0020.0200.2002`) owns the selected FAD.

Flex-Algo prefix node-SID index = algorithm offset + node id, so the hundreds digit of the label tells you the algorithm:

| Node | Algo 128 | Algo 129 | Algo 130 |
| --- | --- | --- | --- |
| PE1 | 101 | 201 | 301 |
| PE2 | 102 | 202 | 302 |
| P1…P7 | 111…117 | 211…217 | 311…317 |

Algo 0 keeps the original SIDs of the base lab (PE1 = index 1, PE2 = index 7).

---

## Services

| VPRN | Service ID | Route target | Algo | Loopback PE1 / PE2 | vrf-import |
| --- | --- | --- | --- | --- | --- |
| INTERNET-A0 | 1000 | target:65000:1000 | 0 | 192.168.0.1 / .2 | none (control group) |
| VOZ-A128 | 1128 | target:65000:1128 | 128 | 192.168.128.1 / .2 | IMP-A128 |
| BULK-A129 | 1129 | target:65000:1129 | 129 | 192.168.129.1 / .2 | IMP-A129 |
| BANCO-A130 | 1130 | target:65000:1130 | 130 | 192.168.130.1 / .2 | IMP-A130 |

`INTERNET-A0` has no import policy on purpose: without `action flex-algo`, the next hop resolves over algo 0.

---

## Key configuration (classic CLI)

### FADs (PE1 and PE2)

```
configure router
    flexible-algorithm-definitions
        flex-algo "128-delay" create
            metric-type delay
            priority 100
            no shutdown
        exit
        flex-algo "129-TE" create
            metric-type te-metric
            priority 100
            no shutdown
        exit
        flex-algo "130-NO-RED" create
            metric-type igp
            priority 100
            exclude
                admin-group "RED"
            exit
            no shutdown
        exit
    exit
exit
```

### IS-IS: level, TE, participation and node-SIDs (all nodes; example P1)

PE1 and PE2 also add `advertise "<fad-name>"` under each `flex-algo`.

```
configure router isis
    level-capability level-2
    traffic-engineering
    flexible-algorithms
        flex-algo 128
            participate
        exit
        flex-algo 129
            participate
        exit
        flex-algo 130
            participate
        exit
        no shutdown
    exit
    interface "system"
        flex-algo 128
            ipv4-node-sid index 111
        exit
        flex-algo 129
            ipv4-node-sid index 211
        exit
        flex-algo 130
            ipv4-node-sid index 311
        exit
    exit
exit
```

### Link attributes (example P1, RED link toward P2)

```
configure router
    if-attribute
        admin-group "RED" value 1
    exit
    interface "toP2"
        if-attribute
            admin-group "RED"
            delay
                static 5000
            exit
        exit
    exit
    mpls
        interface "toP2"
            te-metric 100
            no shutdown
        exit
        no shutdown
    exit
exit
```

### IGP metric (example P6, express link)

```
configure router isis interface "toP7" level 2 metric 50
```

### Service steering (example VOZ-A128 on PE1)

```
configure router policy-options
    begin
    community "RT-1128" members "target:65000:1128"
    policy-statement "IMP-A128"
        entry 10
            from
                community "RT-1128"
            exit
            action accept
                flex-algo 128
            exit
        exit
    exit
    commit
exit

configure service vprn 1128 name "VOZ-A128" customer 1 create
    interface "lo-test" create
        address 192.168.128.1/32
        loopback
    exit
    bgp-ipvpn
        mpls
            auto-bind-tunnel
                resolution-filter
                    sr-isis
                exit
                resolution filter
            exit
            route-distinguisher 1.1.1.1:1128
            vrf-import "IMP-A128"
            vrf-target target:65000:1128
            no shutdown
        exit
    exit
    no shutdown
exit
```

---

## Verification

### 1. FADs advertised by PE1

```
A:PE1# show router isis database PE1.00-00 detail
...
  Router Cap : 1.1.1.1, D:0, S:0
    TE Node Cap : B E M  P
    SR Cap: IPv4 MPLS-IPv6
       SRGB Base:100000, Range:1000
    SR Alg: metric based SPF, 128, 129, 130
    Node MSD Cap: BMI : 12 ERLD : 15
    FAD Sub-Tlv:
        Flex-Algorithm   : 128
        Metric-Type      : delay
        Calculation-Type : 0
        Priority         : 100
        Flags: M
    FAD Sub-Tlv:
        Flex-Algorithm   : 129
        Metric-Type      : te-metric
        Calculation-Type : 0
        Priority         : 100
        Flags: M
    FAD Sub-Tlv:
        Flex-Algorithm   : 130
        Metric-Type      : igp
        Calculation-Type : 0
        Priority         : 100
        Flags: M
        Exclude Admin Group: 0x2
...
      Prefix-SID Index:1, Algo:0, Flags:NnP
      Prefix-SID Index:101, Algo:128, Flags:NnP
      Prefix-SID Index:201, Algo:129, Flags:NnP
      Prefix-SID Index:301, Algo:130, Flags:NnP
```

### 2. Link attributes (P4: RED toward P5, TE 10 toward P7)

```
A:PE1# show router isis database P4.00-00 level 2 detail | match "TE IS Nbrs" post-lines 10
  TE IS Nbrs   :
    Nbr   : P3.00
    Default Metric  : 10
    Sub TLV Len     : 39
    IF Addr   : 172.16.2.2
    Nbr IP    : 172.16.2.1
    TE APP LINK ATTR    :
      SABML-flag:Non-Legacy SABM-flags:   X
        Delay Min : 3000 Max : 3000
        TE Metric : 10
    Adj-SID: Flags:v4VL Weight:0 Label:524286
  TE IS Nbrs   :
    Nbr   : P5.00
    Default Metric  : 10
    Sub TLV Len     : 45
    IF Addr   : 172.16.3.1
    Nbr IP    : 172.16.3.2
    TE APP LINK ATTR    :
      SABML-flag:Non-Legacy SABM-flags:   X
        Admin Grp : 0x2
        Delay Min : 3000 Max : 3000
        TE Metric : 10
  TE IS Nbrs   :
    Nbr   : P7.00
    Default Metric  : 10
    Sub TLV Len     : 39
    IF Addr   : 172.16.5.2
    Nbr IP    : 172.16.5.1
    TE APP LINK ATTR    :
      SABML-flag:Non-Legacy SABM-flags:   X
        Delay Min : 2000 Max : 2000
        TE Metric : 10
    Adj-SID: Flags:v4VL Weight:0 Label:524287
```

The `X` flag means the attributes are advertised for the Flex-Algo application (ASLA).

### 3. One tunnel per algorithm toward PE2

`fp-tunnel-table` shows what is programmed in the forwarding plane, including the pushed label, which is the only thing that identifies the algorithm.

```
A:PE1# show router fp-tunnel-table 1 | match 2.2.2.2 post-lines 2
2.2.2.2/32                                   SR-ISIS-0         524341
  100007
    10.0.1.2                                                 1/1/c1/1
2.2.2.2/32                                   SR-ISIS-0         524370
  100102
    10.0.6.2                                                 1/1/c3/1
2.2.2.2/32                                   SR-ISIS-0         524371
  100202
    10.0.3.2                                                 1/1/c2/1
2.2.2.2/32                                   SR-ISIS-0         524372
  100302
    10.0.3.2                                                 1/1/c2/1
```

| Label | Algo | Tunnel ID | Next hop |
| --- | --- | --- | --- |
| 100007 | 0 | 524341 | P1 |
| 100102 | 128 | 524370 | P6 |
| 100202 | 129 | 524371 | P3 |
| 100302 | 130 | 524372 | P3 |

### 4. Each VPRN resolves the same next hop with its own tunnel

```
A:PE1# show router 1000 route-table
...
192.168.0.2/32                                Remote  BGP VPN   00h00m56s  170
       2.2.2.2 (tunneled:SR-ISIS:524341)                            30

A:PE1# show router 1128 route-table
...
192.168.128.2/32                              Remote  BGP VPN   00h00m48s  170
       2.2.2.2 (tunneled:SR-ISIS:524370)                            3063

A:PE1# show router 1129 route-table
...
192.168.129.2/32                              Remote  BGP VPN   00h00m48s  170
       2.2.2.2 (tunneled:SR-ISIS:524371)                            40

A:PE1# show router 1130 route-table
...
192.168.130.2/32                              Remote  BGP VPN   00h00m49s  170
       2.2.2.2 (tunneled:SR-ISIS:524372)                            50
```

Same next hop (`2.2.2.2`), four different tunnel IDs that match the four labels above. Metrics 30, 40 and 50 are the IGP / TE / IGP cost of paths N, S and X1.

> **Note on 3063:** the Metric column for the delay-based tunnel is not the plain sum of link delays. The three link delays on path M (1000 + 2000 + 1000 µs) were verified in the LSDB; the tunnel ID is what proves the binding.

### 5. Data plane

By default the PE does not copy the TTL into the transport label, so traceroute only shows the destination:

```
A:PE1# traceroute 192.168.130.2 router-instance "BANCO-A130"
traceroute to 192.168.130.2, 30 hops max, 40 byte packets
  1  192.168.130.2 (192.168.130.2)    3.44 ms  2.91 ms  4.59 ms
```

After enabling TTL propagation, the P hops become visible as hop count:

```
A:PE1# configure router ttl-propagate vprn-local all
A:PE1# traceroute 192.168.130.2 router-instance "BANCO-A130"
traceroute to 192.168.130.2, 30 hops max, 40 byte packets
  1  0.0.0.0  * * *
  2  0.0.0.0  * * *
  3  0.0.0.0  * * *
  4  192.168.130.2 (192.168.130.2)    4.31 ms  3.55 ms  4.60 ms
```

Four hops = three P routers (P3, P4, P7), which is path X1. The P routers show `* * *` because the traceroute source lives inside the VPRN and they have no route back to it in the global table.

> Static delay is an advertised value used for path computation, not real latency. Traceroute RTTs in the lab do not reflect it.

---

## Failure test: P3–P4 down

```
A:P3# configure port 1/1/c2/1 shutdown
```

### Tunnels on PE1 after the failure

```
A:PE1# show router fp-tunnel-table 1 | match 2.2.2.2 post-lines 2
2.2.2.2/32                                   SR-ISIS-0         524341
  100007
    10.0.1.2                                                 1/1/c1/1
2.2.2.2/32                                   SR-ISIS-0         524370
  100102
    10.0.6.2                                                 1/1/c3/1
2.2.2.2/32                                   SR-ISIS-0         524371
  100202
    10.0.6.2                                                 1/1/c3/1
2.2.2.2/32                                   SR-ISIS-0         524372
  100302
    10.0.6.2                                                 1/1/c3/1
```

### Traceroutes after the failure

```
A:PE1# traceroute 192.168.129.2 router-instance "BULK-A129"
traceroute to 192.168.129.2, 30 hops max, 40 byte packets
  1  0.0.0.0  * * *
  2  0.0.0.0  * * *
  3  0.0.0.0  * * *
  4  0.0.0.0  * * *
  5  192.168.129.2 (192.168.129.2)    6.17 ms  3.73 ms  3.54 ms

A:PE1# traceroute 192.168.130.2 router-instance "BANCO-A130"
traceroute to 192.168.130.2, 30 hops max, 40 byte packets
  1  0.0.0.0  * * *
  2  0.0.0.0  * * *
  3  192.168.130.2 (192.168.130.2)    2.84 ms  1.92 ms  3.05 ms

A:PE1# traceroute 192.168.0.2 router-instance "INTERNET-A0"
traceroute to 192.168.0.2, 30 hops max, 40 byte packets
  1  0.0.0.0  * * *
  2  0.0.0.0  * * *
  3  192.168.0.2 (192.168.0.2)    6.27 ms  4.67 ms  2.19 ms

A:PE1# traceroute 192.168.128.2 router-instance "VOZ-A128"
traceroute to 192.168.128.2, 30 hops max, 40 byte packets
  1  0.0.0.0  * * *
  2  0.0.0.0  * * *
  3  192.168.128.2 (192.168.128.2)    2.30 ms  3.23 ms  4.73 ms
```

### What happened and why

| Service | Before | After | Why |
| --- | --- | --- | --- |
| BULK-A129 | S, 4 hops | **X2, 5 hops** (P6 → P7 → P4 → P5) | Lowest TE without P3–P4: X2 (130) beats M (150) and N (300). Algo 129 does not care about hop count |
| BANCO-A130 | X1, 4 hops | **M, 3 hops** (P6 → P7) | The only path left without RED links |
| INTERNET-A0 | N, 3 hops | N, 3 hops | Its path never used P3–P4 |
| VOZ-A128 | M, 3 hops | M, 3 hops | Its path never used P3–P4 |

BULK, BANCO and VOZ now leave PE1 through the same neighbor (P6) and split at P7: BULK goes to P4, VOZ and BANCO go straight to PE2.

Restore the link:

```
A:P3# configure port 1/1/c2/1 no shutdown
```

---

## Lessons learned while building it

Real issues found while building the lab, and what each one teaches.

### 1. The admin-group name is local, the bit is global

The FAD excluded `0x2` (RED = value 1 on the PEs), but P1 had RED defined as value 10:

```
A:P1# show router if-attribute admin-group
Group Name                       Group Value
--------------------------------------------------
RED                              10
```

So P1 advertised its link to P2 as `Admin Grp : 0x400`. The link was not pruned, and algo 130 went north (over RED) from PE1, while the return direction (P2 advertises `0x2`) was pruned. Flex-Algo constraints are evaluated with the attributes advertised by the node the traffic leaves from, which is why both ends must be tagged, with the same value everywhere.

### 2. In IS-IS, L1 beats L2 regardless of metric

P1–P5 were L2-only, while PE1, PE2, P6 and P7 were L1L2. That created an L1-only chain PE1–P6–P7–PE2, and every algorithm, including algo 0, followed it:

```
A:PE1# show router isis adjacency
P1                       L2    Up    23   toP1                          0
P3                       L2    Up    23   toP3                          0
P6                       L1L2  Up    23   toP6                          0

A:PE1# show router fp-tunnel-table 1 | match 2.2.2.2 post-lines 2
2.2.2.2/32                                   SR-ISIS-0         524293
  100007
    10.0.6.2                                                 1/1/c3/1
2.2.2.2/32                                   SR-ISIS-0         524321
  100102
    10.0.6.2                                                 1/1/c3/1
2.2.2.2/32                                   SR-ISIS-0         524322
  100202
    10.0.6.2                                                 1/1/c3/1
2.2.2.2/32                                   SR-ISIS-0         524326
  100302
    10.0.6.2                                                 1/1/c3/1
```

Fix: `level-capability level-2` on all nine nodes.

### 3. Every metric must be checked in the LSDB, not in the config

P7 had IGP 50 toward PE2 (instead of 20) and TE 50 toward P4 (instead of 10). The first turned BANCO's metric into 80 instead of 50. The second would have broken the failure test: BULK would have picked M (150) instead of X2 (170 with the wrong value). Auditing every node with `show router isis database <node>.00-00 level 2 detail | match "TE IS Nbrs" post-lines 10` from a single router works because the LSDB is identical on every node of the level.

### 4. Classic CLI creates what you name

In classic, `configure router interface "<name>"` creates the interface if the name does not exist. A typo (`to-P2` instead of `toP2`) silently created an empty interface. Check names with `show router interface` before pasting.

---

## Next experiments

- **Raise the express delay:** set P6–P7 to 20000 µs on both ends. Only VOZ should move, to X1.
- **BANCO would rather fail than touch RED:** shut P6–P7 and P4–P7. Algo 130 has no path, and VPRN 1130 loses the remote route. Then enable `allow-flex-algo-fallback` and watch it come back over RED.
- **Participation:** remove P7 from algo 128. VOZ moves to S even though P7 has the best delay.
- **FAD war:** advertise a FAD for algo 129 with priority 200 and metric-type igp from P5. The whole network adopts it.
- **TI-LFA inside the algorithm:** enable TI-LFA and check that BANCO's backup also avoids RED.

---

## Deploying the lab

Requirements: [Containerlab](https://containerlab.dev/), a Nokia SR-SIM image and a valid license, and enough RAM for 9 SR-SIM nodes.

```bash
sudo containerlab deploy -t Lab_Flex_algo.clab.yml
sudo containerlab destroy -t Lab_Flex_algo.clab.yml
```

---

## References

- [RFC 9350 — IGP Flexible Algorithm](https://www.rfc-editor.org/rfc/rfc9350)
- [srl-labs/nokia-segment-routing-lab](https://github.com/srl-labs/nokia-segment-routing-lab)
- [NTT: IGP Flexible Algorithm Interoperability Updates](https://engineers.ntt.com/entry/2023/09/19/090000)
- [Nokia: Configuring the flexible algorithm definition](https://infocenter.nokia.com/public/7750SR225R1A/topic/com.nokia.Segment_Routing_and_PCE_User_Guide/configuring_the-ai9ekdb655.html)
- [Nokia: Configuring IS-IS Flex-Algorithm participation](https://infocenter.nokia.com/public/7750SR225R1A/topic/com.nokia.Segment_Routing_and_PCE_User_Guide/configuring_is--ai9ekdb656.html)
- [Nokia: Configuring IS-IS Flex-Algorithm prefix node SID](https://infocenter.nokia.com/public/7750SR225R1A/topic/com.nokia.Segment_Routing_and_PCE_User_Guide/configuring_is--ai9ekdb657.html)
- [Nokia: Configuring interface delay](https://infocenter.nokia.com/public/7750SR222R1A/topic/com.nokia.Router_Configuration_Guide/configuring_int-ai9exj5xqd.html)



