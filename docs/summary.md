- [1. What's BBR?](#1-whats-bbr)
- [2.  **Network Path Model**](#2--network-path-model)
  - [Key Parameters](#key-parameters)
  - [Control Mechanisms](#control-mechanisms)
  - [Handling Loss and Recovery](#handling-loss-and-recovery)
  - [Restarting from Idle](#restarting-from-idle)
- [3. ProbeBW state cycling](#3-probebw-state-cycling)
  - [3.1. ProbeBW\_DOWN](#31-probebw_down)
  - [3.2. ProbeBW\_CRUISE](#32-probebw_cruise)
  - [3.3. ProbeBW\_REFILL](#33-probebw_refill)
  - [3.4. ProbeBW\_UP](#34-probebw_up)
  - [Summary](#summary)
- [5. Understanding BBR's Rate and Inflght Flow Control Bounds](#5-understanding-bbrs-rate-and-inflght-flow-control-bounds)
  - [Parameter Definitions](#parameter-definitions)
  - [When to update](#when-to-update)
  - [How These Parameters Affect BBR Behavior](#how-these-parameters-affect-bbr-behavior)
- [5. Measurement](#5-measurement)
    - [Measuring RTT](#measuring-rtt)
    - [Measuring Delivery Rate](#measuring-delivery-rate)
  - [Summary](#summary-1)
- [6. RTT Sample](#6-rtt-sample)
  - [Filtering Spurious RTT Samples](#filtering-spurious-rtt-samples)
  - [Example Implementation](#example-implementation)
  - [Summary](#summary-2)
- [7. protocol implementation considerations](#7-protocol-implementation-considerations)
  - [Inputs Required by BBR from Transmission Protocol](#inputs-required-by-bbr-from-transmission-protocol)
  - [Outputs from BBR to Transmission Protocol](#outputs-from-bbr-to-transmission-protocol)
  - [Other Considerations](#other-considerations)
- [8. Tracking Application-Limited Periods in BBR](#8-tracking-application-limited-periods-in-bbr)
  - [Detecting Application-Limited vs. cwnd-Limited States](#detecting-application-limited-vs-cwnd-limited-states)
    - [1. At Packet Transmission Time](#1-at-packet-transmission-time)
    - [2. At ACK Reception Time](#2-at-ack-reception-time)
  - [Key Implementation Details](#key-implementation-details)
- [9. Simplified Bandwidth Probing Time Scale in BBR](#9-simplified-bandwidth-probing-time-scale-in-bbr)
  - [Recommended Implementation](#recommended-implementation)
  - [Parameter Guidelines](#parameter-guidelines)
- [10. Probe min-RTT](#10-probe-min-rtt)

## 1. What's BBR?

BBR (Bottleneck Bandwidth and Round-trip propagation time) is a congestion control algorithm that builds an explicit model of the network path using recent measurements of a transport connection's delivery rate and round-trip time. 

## 2.  **Network Path Model**

The "Network Path Model" in BBR (Bottleneck Bandwidth and Round-trip propagation time) is a crucial component that helps the algorithm estimate the network conditions and make informed decisions about data transmission. Here is a detailed explanation of the model, including the mathematical aspects:

### Key Parameters

1. **BBR.BtlBw (Bottleneck Bandwidth)**:
   - This is the estimated maximum bandwidth available to the transport flow.
   - It is calculated as the maximum delivery rate sample from a moving window of recent delivery rate measurements.

2. **BBR.RTprop (Round-Trip Propagation Time)**:
   - This is the estimated minimum round-trip time (RTT) of the path.
   - It is calculated as the minimum RTT sample from a moving window of recent RTT measurements.

### Control Mechanisms

The optimal Target Operating Point in BBR is the point where the connection achieves high throughput and low delay. This optimal point is based on two key conditions:

1. **Rate Balance**:
   - The bottleneck packet arrival rate equals the bottleneck bandwidth available to the transport flow.
   - This ensures that the sending rate matches the capacity of the network path, avoiding both underutilization and excessive queuing.

2. **Full Pipe**:
   - The total data in flight along the path is equal to the Bandwidth-Delay Product (BDP).
   - This ensures that the network path is fully utilized without causing congestion.

BBR uses its model to maintain an estimate of this optimal operating point by pacing packets at or near the estimated bottleneck bandwidth (`BBR.BtlBw`) and modulating the congestion window to keep the data in flight close to the BDP (`BBR.BtlBw * BBR.RTprop`).


### Handling Loss and Recovery

BBR interprets packet loss as a hint of potential changes in path behavior and adjusts its congestion window conservatively during loss recovery. This helps in avoiding congestion and maintaining stability.

### Restarting from Idle

When restarting from idle, BBR paces packets at the estimated bottleneck bandwidth to quickly return to the target operating point. This helps in maintaining high throughput and low latency.

## 3. ProbeBW state cycling

The ProbeBW cycle transitions through four sequential states (DOWN→CRUISE→REFILL→UP) with varying durations. Here's a breakdown of each state's purpose and duration:

### 3.1. ProbeBW_DOWN
- **Purpose**: Deceleration tactic (pacing_gain = 0.9)
- **Duration**: Variable
- **Exit Conditions**:
  - Inflight data is less than or equal to BBR.BDP (estimated bandwidth-delay product)
  - AND there is sufficient headroom (inflight ≤ BBRHeadroom*BBR.inflight_hi)
- **Transitions to**: ProbeBW_CRUISE

### 3.2. ProbeBW_CRUISE
- **Purpose**: Cruising tactic (pacing_gain = 1.0)
- **Duration**: Adaptively determined
- **Exit Conditions**: When it's time to probe for bandwidth, based on either:
  - Wall clock time: T_bbr = 2-3 seconds (randomized)
  - OR Round-trip count: T_reno = min(estimated_BDP, cwnd) round trips (capped at 62-63 round trips)
- **Transitions to**: ProbeBW_REFILL

### 3.3. ProbeBW_REFILL
- **Purpose**: Refill the pipe (pacing_gain = 1.0)
- **Duration**: Exactly one packet-timed round trip
- **Exit Condition**: One round trip has elapsed
- **Transitions to**: ProbeBW_UP

### 3.4. ProbeBW_UP
- **Purpose**: Probe for bandwidth increases (pacing_gain = 1.25)
- **Duration**: At least BBR.min_rtt, potentially longer
- **Exit Conditions**: 
  - At least BBR.min_rtt has elapsed AND inflight > 1.25 * BBR.bdp
  - OR loss rate exceeds BBRLossThresh (2%)
- **Transitions to**: ProbeBW_DOWN (completing the cycle)

The cycle then repeats, with most of the time typically spent in CRUISE state, making brief excursions to probe bandwidth (REFILL→UP) and drain any resulting queue (DOWN).

### Summary

Gain cycling is a crucial part of BBR's ProbeBW state, allowing the algorithm to dynamically adjust its pacing rate to probe for available bandwidth and maintain optimal network performance. By cycling through different pacing gains, BBR can effectively balance high throughput and low queuing delay.

## 5. Understanding BBR's Rate and Inflght Flow Control Bounds

In BBR, four critical parameters work together to manage bandwidth utilization and in-flight data: `bw_hi`, `bw_lo`, `inflight_hi`, and `inflight_lo`. These parameters act as bounds that BBR uses to adapt to network conditions.

### Parameter Definitions

- **Long-term Model Parameters**
  - **bw_hi**: Long-term maximum sending bandwidth that produces acceptable queue pressure
  - **inflight_hi**: Long-term maximum volume of in-flight data that produces acceptable queue pressure

- **Short-term Model Parameters**
   - **bw_lo**: Short-term maximum sending bandwidth considered safe based on recent loss signals
   - **inflight_lo**: Short-term maximum volume of in-flight data considered safe based on recent loss signals

### When to update

- **Upper Bounds (bw_hi, inflight_hi)**
  1. **During ProbeBW_UP state**:
     - `inflight_hi` increases gradually through `BBRProbeInflightHiUpward()`
     - Starts cautiously, but increases more aggressively over time
     - Growth doubles each round trip (1, 2, 4, 8, 16 packets, etc.)

  2. **When loss is acceptable**:
     - If `rs.tx_in_flight > BBR.inflight_hi`, then `BBR.inflight_hi = rs.tx_in_flight`
     - If `rs.delivery_rate > BBR.bw_hi`, then `BBR.bw_hi = rs.delivery_rate`

  3. **When loss is excessive**:
     - If loss rate exceeds `BBRLossThresh` (2%), `inflight_hi` is reduced via `BBRHandleInflightTooHigh()`

- **Lower Bounds (bw_lo, inflight_lo)**

   1. **During loss events**:
      - At the end of rounds with newly detected loss:
      ```
      bw_lo = max(bw_latest, BBRBeta * bw_lo)
      inflight_lo = max(inflight_latest, BBRBeta * inflight_lo)
      ```
      - `BBRBeta` is 0.7 (similar to CUBIC's multiplicative decrease factor)

   2. **During ProbeBW_REFILL**:
      - Both `bw_lo` and `inflight_lo` are reset to "Infinity"
      - This is the pivotal moment when BBR transitions from conservative short-term constraints to probing

### How These Parameters Affect BBR Behavior

Each BBR state uses these parameters differently to constrain bandwidth and in-flight data:

| State          | Rate Constraints | Volume Constraints            |
| -------------- | ---------------- | ----------------------------- |
| Startup        | None             | None                          |
| Drain          | bw_hi, bw_lo     | inflight_hi, inflight_lo      |
| ProbeBW_DOWN   | bw_hi, bw_lo     | inflight_hi, inflight_lo      |
| ProbeBW_CRUISE | bw_hi, bw_lo     | 0.85*inflight_hi, inflight_lo |
| ProbeBW_REFILL | bw_hi            | inflight_hi                   |
| ProbeBW_UP     | bw_hi            | inflight_hi                   |
| ProbeRTT       | bw_hi, bw_lo     | 0.85*inflight_hi, inflight_lo |

This dual-bound approach allows BBR to balance two goals:
1. Quickly responding to short-term congestion (via low bounds)
2. Efficiently probing for long-term available bandwidth (via high bounds)

## 5. Measurement

#### Measuring RTT

1. **When to Measure RTT**:
   - RTT is measured continuously for each acknowledged packet.
   - BBR updates its estimate of the minimum RTT (`BBR.RTprop`) using a moving window of recent RTT samples.

2. **How to Measure RTT**:
   - RTT is calculated as the time difference between when a packet is sent and when its acknowledgment (ACK) is received.
   - BBR keeps track of the smallest RTT observed over a certain period (e.g., 10 seconds) to estimate the propagation delay.

3. **Important Considerations**:
   - Ensure that the RTT measurements are not inflated by queuing delays. BBR focuses on the minimum RTT to estimate the propagation delay accurately.
   - Filter out spurious RTT samples caused by retransmissions or delayed ACKs.

#### Measuring Delivery Rate

1. **When to Measure Delivery Rate**:
   - Delivery rate is measured continuously for each acknowledged packet.
   - BBR updates its estimate of the bottleneck bandwidth (`BBR.BtlBw`) using a moving window of recent delivery rate samples.

2. **How to Measure Delivery Rate**:
   - Delivery rate is calculated as the amount of data acknowledged over the time interval between the sending of the first packet and the acknowledgment of the last packet in a burst.
   - BBR uses the maximum delivery rate observed over a certain period (e.g., 10 seconds) to estimate the bottleneck bandwidth.

3. **Important Considerations**:
   - Ensure that the delivery rate measurements are not affected by transient network conditions or short-term fluctuations.
   - Use a sufficiently large window to smooth out variations and get a stable estimate of the bottleneck bandwidth.

### Summary

Accurate measurement of RTT and delivery rate is crucial for BBR to build an effective network path model. Here are the key points to keep in mind:

- **RTT Measurement**:
  - Measure RTT continuously for each acknowledged packet.
  - Focus on the minimum RTT to estimate the propagation delay accurately.
  - Filter out spurious RTT samples caused by retransmissions or delayed ACKs.

- **Delivery Rate Measurement**:
  - Measure delivery rate continuously for each acknowledged packet.
  - Use the maximum delivery rate observed over a certain period to estimate the bottleneck bandwidth.
  - Smooth out variations using a sufficiently large window to get a stable estimate.

By following these guidelines, BBR can maintain accurate and reliable estimates of the network path characteristics, enabling it to achieve high throughput and low latency.

## 6. RTT Sample

guidelines for filter out spurious RTT samples caused by retransmissions or delayed ACKs:

### Filtering Spurious RTT Samples

1. **Ignore RTT Samples from Retransmitted Packets**:
   - Do not use RTT samples based on the transmission time of retransmitted packets, as these are ambiguous and unreliable.
   - Only use RTT samples from the original transmission of packets.

2. **Use Selective Acknowledgments (SACK)**:
   - If the transport protocol supports SACK (Selective Acknowledgment) options (e.g., TCP SACK), use these to accurately measure RTT for individual segments.
   - SACK helps in identifying which segments were acknowledged and avoids confusion caused by cumulative ACKs.

3. **Use Transport-Layer Timestamps**:
   - If the transport protocol supports timestamps (e.g., TCP timestamps), use these to measure the exact time a packet was sent and acknowledged.
   - Timestamps provide precise timing information, reducing the impact of delayed ACKs.

4. **Focus on Minimum RTT**:
   - Maintain a moving window of recent RTT samples and focus on the minimum RTT observed over a certain period (e.g., 10 seconds).
   - The minimum RTT is less likely to be inflated by queuing delays and provides a more accurate estimate of the propagation delay.

5. **Filter Out Delayed ACKs**:
   - Delayed ACKs can inflate RTT measurements. To mitigate this, consider only using RTT samples from ACKs that are not delayed.
   - If the transport protocol allows, detect and exclude delayed ACKs from RTT calculations.

### Example Implementation

Here is a pseudocode example to illustrate how to filter out spurious RTT samples:

```plaintext
BBRUpdateRTprop():
  if (packet.is_retransmitted):
    return  // Ignore RTT sample from retransmitted packet

  if (packet.ack.is_delayed):
    return  // Ignore RTT sample from delayed ACK

  rtt_sample = Now() - packet.sent_time

  if (rtt_sample < BBR.RTprop):
    BBR.RTprop = rtt_sample  // Update minimum RTT

  // Maintain a moving window of recent RTT samples
  BBR.RTprop_window.append(rtt_sample)
  if (len(BBR.RTprop_window) > window_size):
    BBR.RTprop_window.pop(0)

  // Update minimum RTT from the window
  BBR.RTprop = min(BBR.RTprop_window)
```

### Summary

By following these guidelines and implementing the filtering logic, you can ensure that BBR maintains accurate and reliable RTT measurements, which are crucial for building an effective network path model. This helps BBR achieve high throughput and low latency by accurately estimating the propagation delay and avoiding spurious RTT samples caused by retransmissions or delayed ACKs.

## 7. protocol implementation considerations

To integrate BBR congestion control with a sliding Window Transmission Protocol; here's what's needed:

### Inputs Required by BBR from Transmission Protocol

1. **Per-Packet Data**:
   - `packet.departure_time`: Wall-clock time when each packet is sent
   - `packet.size`: Size of each packet sent
   - `packet.delivered`: Amount of data delivered when this packet was sent
   - `packet.tx_in_flight`: Amount of data in flight when packet was sent
   - Flag indicating if transmission was application-limited

2. **Per-ACK Data**:
   - Wall-clock time when ACK was received
   - Which packet(s) were acknowledged
   - `rs.newly_acked`: Bytes newly acknowledged by this ACK
   - `rs.newly_lost`: Bytes newly marked as lost by this ACK
   - `rs.delivery_rate`: Calculated delivery rate sample
   - ACK arrival timestamps for RTT calculation

3. **Connection State**:
   - Current bytes in flight
   - Whether connection is cwnd-limited or application-limited
   - Initial congestion window size
   - Sender Maximum Segment Size (SMSS)
   - `C.delivered`: Total bytes delivered so far

### Outputs from BBR to Transmission Protocol

1. **Primary Control Parameters**:
   - `BBR.pacing_rate`: The rate at which to send packets
   - `cwnd`: Maximum volume of data allowed in flight
   - `BBR.send_quantum`: Maximum burst size for transmission efficiency

2. **Additional State Information**:
   - `BBR.state`: Current state in BBR state machine
   - `BBR.packet_conservation`: Whether BBR is in packet conservation mode
   - `BBR.idle_restart`: Whether connection is restarting from idle

### Other Considerations

1. Need to implement a packet pacing mechanism if you don't already have one
2. Implement the delivery rate calculation algorithm described in [draft-cheng-iccrg-delivery-rate-estimation]
3. Add packet loss detection and RTT measurement capabilities
4. Track application-limited periods properly to avoid misinterpreting application limits as network limits

When correctly implemented, BBR will adjust these parameters based on network conditions to provide high throughput with low queuing delay.

## 8. Tracking Application-Limited Periods in BBR

Properly tracking application-limited periods is crucial for BBR to distinguish between network constraints and application constraints. Here's how to implement this:

### Detecting Application-Limited vs. cwnd-Limited States

#### 1. At Packet Transmission Time

```c
// Check if application-limited when sending a packet
is_app_limited = (available_data_to_send < cwnd - bytes_in_flight)

// If application-limited, mark the packet and record when it began
if (is_app_limited) {
    packet.is_app_limited = true;
    if (!connection.app_limited) {
        connection.app_limited = true;
        connection.app_limited_start_delivered = connection.delivered;
    }
}
```

#### 2. At ACK Reception Time

```c
// When receiving ACKs for app-limited packets
if (packet.is_app_limited) {
    rs.is_app_limited = true;
}

// Check if we've exited app-limited state
if (connection.app_limited && 
    bytes_in_flight + available_data_to_send >= cwnd) {
    connection.app_limited = false;
    // Optional: track how much data was delivered during app-limited period
    connection.app_limited_sample = connection.delivered - 
                                  connection.app_limited_start_delivered;
}
```

### Key Implementation Details

1. **Accurate Measurement**: Track both:
   - The number of bytes ready to send from the application
   - The current available window (cwnd - bytes_in_flight)

2. **State Transitions**:
   - Enter app-limited state when available_data < available_window
   - Exit app-limited state when available_data + bytes_in_flight ≥ cwnd

3. **Proper Marking**: 
   - Mark every packet sent during app-limited periods
   - Mark every resulting delivery rate sample as app-limited
   - Continue marking samples as app-limited until all packets sent during the app-limited period are acknowledged

4. **Integration with BBR**: 
   ```c
   // When updating BBR.max_bw
   BBRUpdateMaxBw() {
     if (rs.delivery_rate >= BBR.max_bw || !rs.is_app_limited) {
       BBR.max_bw = update_windowed_max_filter(...)
     }
   }
   ```

5. **Edge Cases**:
   - Reset app-limited tracking after idle periods
   - Handle reordering and packet loss appropriately
   - Consider transitions between app-limited and non-app-limited periods

This approach ensures BBR correctly distinguishes between network limitations and application constraints, preventing it from underestimating available bandwidth during periods when the application isn't fully utilizing the network path.

## 9. Simplified Bandwidth Probing Time Scale in BBR

If you don't need to coexist with legacy Reno/CUBIC flows, you can adopt a more aggressive and simplified bandwidth probing strategy in BBR. Here's how to modify the approach:

1. **Remove the Dual-Time-Scale Mechanism**
   - Eliminate the `T_reno` calculation entirely
   - Use only the BBR-native time scale for probing frequency

2. **Optimize the Probing Frequency**
   ```c
   // Instead of the complex T_probe calculation:
   T_probe = T_bbr
   
   // Where T_bbr can be adjusted based on your needs:
   T_bbr = uniformly_ranzation
    next_probe_time = now + uniform_random(mdom_between(min_probe_interval, max_probe_interval)
   ```

3. **Consider These Factors When Setting Probe Intervals**:
   - **Network RTT**: More frequent probing for low-RTT environments
   - **Expected bandwidth variability**: More frequent probing for networks with frequently changing bandwidth
   - **Application sensitivity**: Less frequent probing for latency-sensitive applications

### Recommended Implementation

```c
BBRCheckTimeToProbeBW() {
  // Simple time-based approach without Reno consideration
  if (elapsed_time_since_last_probe >= T_bbr) {
    BBRStartProbeBW_REFILL()
    // Set next probe time with some randomiin_interval, max_interval)
    return true
  }
  return false
}
```

### Parameter Guidelines

Without Reno/CUBIC coexistence concerns, you can adjust parameters to be more aggressive:

- **Minimum interval**: 1-2 seconds (instead of 2-3 seconds)
- **ProbeBW_UP pacing_gain**: Could be increased beyond 1.25 for faster bandwidth discovery
- **Loss threshold**: Could be increased beyond 2% if your application tolerates higher loss

This approach allows for more aggressive bandwidth discovery and adaptation, which can be beneficial in rapidly changing network environments where you control all endpoints.

## 10. Probe min-RTT

In BBR, ProbeRTT is a critical state for maintaining an accurate estimate of the minimum RTT. Here's when and how BBR enters ProbeRTT:

- When to Enter ProbeRTT

   BBR enters ProbeRTT when **all** of these conditions are met:

   1. **Timer Expiration**: At least ProbeRTTInterval (5 seconds) has passed since the last update to BBR.probe_rtt_min_delay
   2. **Not Already in ProbeRTT**: The current state is not already ProbeRTT
   3. **Not Restarting from Idle**: The connection is not just restarting from idle (BBR.idle_restart is false)

-  Key Implementation Logic

   ```c
   BBRUpdateMinRTT():
   // Check if ProbeRTT timer has expired
   BBR.probe_rtt_expired = Now() > BBR.probe_rtt_min_stamp + ProbeRTTInterval
   
   // Update RTT measurements if we have a lower sample or timer expired
   if (rs.rtt >= 0 && (rs.rtt < BBR.probe_rtt_min_delay || BBR.probe_rtt_expired))
      BBR.probe_rtt_min_delay = rs.rtt
      BBR.probe_rtt_min_stamp = Now()  // Reset timer

   BBRCheckProbeRTT():
   // Enter ProbeRTT if needed
   if (BBR.state != ProbeRTT && 
         BBR.probe_rtt_expired && 
         !BBR.idle_restart)
      BBREnterProbeRTT()
      BBRSaveCwnd()
      BBR.probe_rtt_done_stamp = 0
      // Additional setup...
   ```

- Natural RTT Sample Optimization

   BBR is designed to avoid unnecessary ProbeRTT states through:

   - Naturally occurring low-utilization periods (application silences, pacing adjustments)
   - Application idle periods that automatically update min_rtt

   This allows BBR to refresh its RTT estimate without explicitly entering ProbeRTT in many cases, minimizing the ~2% throughput penalty associated with ProbeRTT.

- Time Parameters

  - **ProbeRTTInterval**: 5 seconds (time between potential ProbeRTT states)
  - **MinRTTFilterLen**: 10 seconds (moving window for min_rtt tracking: 
  - **ProbeRTTDuration**: 200 milliseconds (minimum time spent in ProbeRTT)

  This design ensures BBR maintains accurate RTT estimates while minimizing performance impact.

- About **MinRTTFilterLen**
  - **Definition**: MinRTTFilterLen (10 seconds by default) is the duration of the moving window used to track minimum RTT measurements
  - **How it works**: BBR maintains a windowed min-filter that keeps the minimum RTT observed over the past MinRTTFilterLen seconds
  - **Aging mechanism**: Any RTT samples older than MinRTTFilterLen seconds are discarded from consideration
