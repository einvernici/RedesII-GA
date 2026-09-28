# OSPF Experiment

## Configuration

The first routing protocol evaluated was **OSPF (Open Shortest Path First)** using **BIRD 2.13.1**.

The topology consists of five routers (R1–R5) with a redundant path between R2 and R4:

```text
R1 ───── R2 ───── R4 ───── R5
         \       /
          \     /
            R3
```

The redundant paths between R2 and R4 are:

```text
Direct:    R2 → R4
Alternate: R2 → R3 → R4
```

This redundancy allows the behavior of OSPF to be evaluated when a link fails.

## Normal Operation

Under normal conditions, the route from R1 to R5 was:

```text
R1 → R2 → R4 → R5
```

The route was verified using:

```bash
docker exec clab-bird-uni-lab-r1 birdc show route 10.4.5.0/24 all
```

The resulting route was:

```text
10.4.5.0/24
    via 10.2.4.4 on eth2
    OSPF.metric1: 20
```

Therefore, R1 used R2 as its first hop and then reached R4 directly.

## Topology Change

To evaluate OSPF's response to a topology change, the R2–R4 connection was disabled:

```bash
docker exec clab-bird-uni-lab-r2 ip link set eth2 down
```

This removed the direct path:

```text
R2 → R4
```

and forced OSPF to use the alternate path through R3:

```text
R1 → R2 → R3 → R4 → R5
```

After OSPF reconverged, the route on R1 was:

```text
10.4.5.0/24
    via 10.1.2.2 on eth1
    OSPF.metric1: 80
```

The increase in the OSPF metric from **20 to 80** reflects the selection of the longer alternate path.

## Packet Loss and Convergence

To observe the effect of the topology change on active traffic, ICMP packets were continuously sent from R1 to R5:

```bash
docker exec -it clab-bird-uni-lab-r1 ping 10.4.5.5
```

During the experiment, the R2–R4 link was disabled while the ping was running.

The following result was obtained:

```text
19 packets transmitted, 18 packets received, 5% packet loss
round-trip min/avg/max = 0.077/0.096/0.160 ms
```

The packet sequence also showed the transition between the two paths:

```text
seq=9   ttl=62
seq=10  [lost]
seq=11  ttl=61
```

The TTL changed from **62 to 61** after the topology change, which is consistent with traffic being redirected through the additional R3 hop.

Only one packet was lost during this test, indicating that OSPF reconverged very rapidly in this experimental topology.

## OSPF Results

| Metric              |            Normal |          R2–R4 Failure |
| ------------------- | ----------------: | ---------------------: |
| Path                | R1 → R2 → R4 → R5 | R1 → R2 → R3 → R4 → R5 |
| OSPF metric         |                20 |                     80 |
| Packets transmitted |                 — |                     19 |
| Packets received    |                 — |                     18 |
| Packet loss         |                 — |                     5% |
| Average RTT         |                 — |               0.096 ms |
| Minimum RTT         |                 — |               0.077 ms |
| Maximum RTT         |                 — |               0.160 ms |
| Convergence         |                 — |             Very rapid |

## Observations

The OSPF experiment demonstrated that the protocol was able to detect the R2–R4 link failure and select the available alternate path through R3.

The route changed from:

```text
R1 → R2 → R4 → R5
```

to:

```text
R1 → R2 → R3 → R4 → R5
```

Despite the topology change, only one packet was lost in the observed test. This indicates rapid reconvergence in the five-router Containerlab topology.

The experiment also demonstrates the advantage of having redundant paths: connectivity between R1 and R5 was maintained even after the direct R2–R4 link failed.
