# BBR Architecture and Workflow Summary

This document summarizes the BBR architecture and workflow from the RFC drafts
in `docs/rfc`, primarily the latest
`draft-cardwell-iccrg-bbr-congestion-control-02.txt`.

## 1. What BBR is trying to do

BBR (Bottleneck Bandwidth and Round-trip propagation time) is a **model-based**
congestion control algorithm.

Instead of treating packet loss as the main signal, BBR continuously measures:

- **delivery rate** to estimate bottleneck bandwidth (`BtlBw` / `max_bw`)
- **round-trip time** to estimate propagation delay (`RTprop` / `min_rtt`)
- **loss rate** to detect excessive queue pressure

It uses these measurements to control:

- **how fast to send**: pacing rate
- **how much data to allow in flight**: congestion window / inflight cap

The target operating point is:

- **Rate balance**: send at about the bottleneck bandwidth
- **Full pipe**: keep about one BDP in flight
- **Low queue pressure**: avoid persistent queues and bufferbloat

## 2. High-level architecture

```text
                   +------------------------------+
                   |     Transport / Protocol     |
                   |  send packets, receive ACKs  |
                   +--------------+---------------+
                                  |
                                  v
                   +------------------------------+
                   |     Measurement Layer        |
                   |  - delivery rate samples     |
                   |  - RTT samples               |
                   |  - loss / ECN style signals  |
                   +--------------+---------------+
                                  |
                                  v
                   +------------------------------+
                   |      Network Path Model      |
                   |  BtlBw, RTprop, BDP, bounds  |
                   |  bw_hi/lo, inflight_hi/lo    |
                   +--------------+---------------+
                                  |
                                  v
                   +------------------------------+
                   |   State Machine + Control    |
                   | Startup / Drain / ProbeBW /  |
                   | ProbeRTT                     |
                   +--------------+---------------+
                                  |
                                  v
                   +------------------------------+
                   |         Output knobs         |
                   |  pacing_rate, cwnd, quantum  |
                   +------------------------------+
```

In short, BBR is a closed-loop controller:

1. The transport sends packets.
2. ACKs and losses provide fresh measurements.
3. BBR updates its path model.
4. The state machine chooses a probing/draining tactic.
5. BBR outputs a new pacing rate and inflight limit.

## 3. Core workflow

### 3.1 Main state machine

```text
      +---------+        +-------+        +-------------------+
      | Startup | -----> | Drain | -----> | ProbeBW (steady)  |
      +---------+        +-------+        +---------+---------+
            ^                                     |
            |                                     |
            |                                     v
            +-----------------------------+  +-----------+
                                          +--| ProbeRTT |
                                             +-----------+
```

### 3.2 State purposes

#### Startup

- Quickly discovers available bandwidth.
- Uses a high pacing gain to grow aggressively.
- Tries to fill the pipe in `O(log2(BDP))` round trips.
- Exits when bandwidth growth stalls or loss indicates too much pressure.

#### Drain

- Slows pacing to remove the queue built during Startup.
- Keeps the pipe full enough to maintain throughput while reducing excess
  inflight.
- Exits when inflight drops back near the estimated BDP.

#### ProbeBW

- Main steady-state mode.
- Spends most of its time here.
- Repeats a cycle of draining, cruising, refilling, and probing upward to
  adapt to path changes.

#### ProbeRTT

- Periodically reduces inflight to a minimum so BBR can re-measure the path's
  true minimum RTT without queueing noise.
- Prevents the RTT model from becoming stale.

## 4. ProbeBW workflow

ProbeBW is the critical steady-state workflow in BBR.

```text
  +--------------+    +----------------+    +----------------+    +------------+
  | ProbeBW_DOWN | -> | ProbeBW_CRUISE | -> | ProbeBW_REFILL | -> | ProbeBW_UP |
  +--------------+    +----------------+    +----------------+    +------------+
         ^                                                                     |
         |                                                                     |
         +---------------------------------------------------------------------+
```

Typical pacing gains:

- **DOWN**: `0.9`
- **CRUISE**: `1.0`
- **REFILL**: `1.0`
- **UP**: `1.25`

### What each sub-state does

#### ProbeBW_DOWN

- Drains queue created by previous probing.
- Moves the connection back toward a low-delay operating point.
- Uses both bandwidth and inflight bounds to stay conservative.

#### ProbeBW_CRUISE

- Holds pacing near the current estimated bottleneck bandwidth.
- Lets the connection run steadily with low queue pressure.
- Waits until the next probe interval.

#### ProbeBW_REFILL

- Refills the pipe to prepare for a clean upward probe.
- Lasts about one packet-timed round trip.
- Resets the controller from conservative behavior back into probing mode.

#### ProbeBW_UP

- Increases offered load to test whether more bandwidth is available.
- Watches for rising delivery rate, rising inflight, and loss pressure.
- Ends when the probe is complete or loss signals that the path is already
  stressed.

## 5. Critical parts of BBR

The following are the most important architectural pieces to understand.

### 5.1 Explicit path model

BBR's core idea is that congestion control should be driven by a **model of the
path**, not just by packet loss.

Key model variables:

- **BtlBw / max_bw**: estimated bottleneck delivery rate
- **RTprop / min_rtt**: estimated minimum propagation delay
- **BDP = BtlBw × RTprop**

These values define the operating point BBR is trying to hold.

### 5.2 Separate control of rate and volume

BBR controls two different things:

- **Rate** with `pacing_rate`
- **Volume** with `cwnd` or inflight limits

This separation is critical. A connection can send at the right average rate
but still build too much queue if inflight volume is too high. BBR therefore
uses both knobs together.

### 5.3 Continuous probing without living in loss

BBR does not assume the best operating point is “just after loss”.
Instead it:

- probes upward to find more available bandwidth
- drains after probing to remove extra queue
- cruises most of the time near the current model

This is why the ProbeBW cycle is the heart of steady-state BBR behavior.

### 5.4 Upper and lower safety bounds

BBR keeps separate long-term and short-term bounds:

- **Long-term upper bounds**: `bw_hi`, `inflight_hi`
- **Short-term conservative bounds**: `bw_lo`, `inflight_lo`

These allow BBR to:

- keep probing when conditions look good
- react quickly when recent loss suggests congestion
- avoid over-committing to stale optimistic estimates

### 5.5 Periodic min-RTT refresh

Without ProbeRTT, the path could stay continuously queued and the RTT estimate
would drift upward. BBR periodically cuts inflight to re-measure the true
baseline propagation delay.

This is crucial because:

- `RTprop` is needed to estimate BDP
- BDP determines the target inflight volume
- stale RTT estimates would distort the whole controller

## 6. ACK-driven control loop

The operational workflow is mostly ACK-driven:

```text
Packet sent
   |
   v
Packet is ACKed
   |
   v
Generate rate sample + RTT sample + loss observations
   |
   v
Update model:
  - max_bw
  - min_rtt
  - bw_hi / bw_lo
  - inflight_hi / inflight_lo
   |
   v
Run state-machine logic
   |
   v
Recompute:
  - pacing_rate
  - cwnd / inflight target
   |
   v
Send next packets using updated controls
```

This loop is what makes BBR adaptive to both short-term congestion and
longer-term path changes.

## 7. Practical mental model

You can think of BBR as three cooperating subsystems:

```text
Measurements  ->  Model  ->  Control State Machine
     ^                              |
     |                              v
     +-------- Transport behavior <-+
```

- **Measurements** tell BBR what recently happened.
- **Model** estimates what the path can currently sustain.
- **State machine** decides whether to grow, drain, cruise, or refresh RTT.

If these three pieces stay accurate and synchronized, BBR can maintain high
throughput while avoiding the persistent queues that are common with purely
loss-based algorithms.
