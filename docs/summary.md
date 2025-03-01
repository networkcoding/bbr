- [1. What's BBR?](#1-whats-bbr)
- [2.  **Network Path Model**](#2--network-path-model)
  - [Key Parameters](#key-parameters)
  - [Mathematical Formulas](#mathematical-formulas)
  - [Control Mechanisms](#control-mechanisms)
  - [State Machine](#state-machine)
  - [Updating Estimates](#updating-estimates)
  - [Handling Loss and Recovery](#handling-loss-and-recovery)
  - [Restarting from Idle](#restarting-from-idle)
- [2. Pacing Gain cycling](#2-pacing-gain-cycling)
  - [Gain Cycling Details](#gain-cycling-details)
  - [Gain Cycling Workflow](#gain-cycling-workflow)
  - [Summary](#summary)
- [3. Measurement](#3-measurement)
    - [Measuring RTT](#measuring-rtt)
    - [Measuring Delivery Rate](#measuring-delivery-rate)
  - [Summary](#summary-1)
- [4. RTT Sample](#4-rtt-sample)
  - [Filtering Spurious RTT Samples](#filtering-spurious-rtt-samples)
  - [Example Implementation](#example-implementation)
  - [Summary](#summary-2)


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

### Mathematical Formulas

1. **Bandwidth-Delay Product (BDP)**:
   - The BDP is the product of the bottleneck bandwidth and the round-trip propagation time.
   - It represents the amount of data that can be in flight in the network path without causing congestion.
   - Formula: `BDP = BBR.BtlBw * BBR.RTprop`

2. **Target Congestion Window (BBR.target_cwnd)**:
   - The target congestion window is the maximum volume of data BBR allows in-flight in the network at any time.
   - It is calculated based on the estimated BDP and a gain factor.
   - Formula: `BBR.target_cwnd = gain * BDP + quanta`
   - Here, `gain` is a scaling factor, and `quanta` is a small constant to ensure a minimum amount of data in flight.

3. **Pacing Rate**:
   - The pacing rate is the rate at which BBR sends data.
   - It is set to match the estimated bottleneck bandwidth.
   - Formula: `Pacing Rate = BBR.BtlBw`

### Control Mechanisms

The optimal Target Operating Point in BBR is the point where the connection achieves high throughput and low delay. This optimal point is based on two key conditions:

1. **Rate Balance**:
   - The bottleneck packet arrival rate equals the bottleneck bandwidth available to the transport flow.
   - This ensures that the sending rate matches the capacity of the network path, avoiding both underutilization and excessive queuing.

2. **Full Pipe**:
   - The total data in flight along the path is equal to the Bandwidth-Delay Product (BDP).
   - This ensures that the network path is fully utilized without causing congestion.

BBR uses its model to maintain an estimate of this optimal operating point by pacing packets at or near the estimated bottleneck bandwidth (`BBR.BtlBw`) and modulating the congestion window to keep the data in flight close to the BDP (`BBR.BtlBw * BBR.RTprop`).

### State Machine

BBR uses a state machine to vary its control parameters and achieve high throughput, low latency, and fair bandwidth sharing. The states include:

1. **Startup**:
   - BBR ramps up its sending rate quickly to estimate the maximum available bandwidth.
   - Formula: `BBR.pacing_gain = BBRStartupPacingGain`
   - Formula: `BBR.cwnd_gain = BBRStartupCwndGain`

2. **Drain**:
   - BBR drains any queue created in Startup by reducing the pacing rate.
   - Formula: `BBR.pacing_gain = BBRDrainPacingGain`

3. **ProbeBW**:
   - BBR probes for bandwidth using gain cycling, alternating between higher and lower pacing gains to find the optimal bandwidth.
   - Formula: `BBR.pacing_gain = [5/4, 3/4, 1, 1, 1, 1, 1, 1]` (eight-phase cycle)

4. **ProbeRTT**:
   - BBR periodically reduces the congestion window to measure the minimum round-trip time.
   - Formula: `BBR.cwnd = min_cwnd`

### Updating Estimates

BBR continually updates its estimates of `BBR.BtlBw` and `BBR.RTprop` using recent delivery rate and RTT samples. This helps in adapting to changing network conditions and maintaining optimal performance.

### Handling Loss and Recovery

BBR interprets packet loss as a hint of potential changes in path behavior and adjusts its congestion window conservatively during loss recovery. This helps in avoiding congestion and maintaining stability.

### Restarting from Idle

When restarting from idle, BBR paces packets at the estimated bottleneck bandwidth to quickly return to the target operating point. This helps in maintaining high throughput and low latency.

## 2. Pacing Gain cycling

Gain cycling is a mechanism used by BBR in the ProbeBW state to probe for available bandwidth and maintain a small, well-bounded queue. Here are more details about gain cycling and its workflow:

### Gain Cycling Details

1. **Purpose**:
   - Gain cycling helps BBR flows reach high throughput, low queuing delay, and convergence to a fair share of bandwidth.

2. **Phases**:
   - BBR cycles through a sequence of pacing gain values in an eight-phase cycle:
     - `[5/4, 3/4, 1, 1, 1, 1, 1, 1]`
   - Each phase typically lasts for roughly one round-trip propagation time (`BBR.RTprop`).

3. **Phases Explained**:
   - **Phase 0 (5/4)**: Probes for more bandwidth by using a pacing gain of 5/4, which gradually raises the amount of data in flight.
   - **Phase 1 (3/4)**: Drains any queue created in Phase 0 by using a pacing gain of 3/4, chosen to be the same distance below 1 as 5/4 is above 1.
   - **Phases 2-7 (1)**: Cruises with a short queue and full utilization by using a pacing gain of 1.0.

4. **Average Gain**:
   - The average gain across all phases is 1.0, aiming for the average pacing rate to equal the estimated available bandwidth (`BBR.BtlBw`).

### Gain Cycling Workflow

1. **Initialization**:
   - Upon entering the ProbeBW state, BBR (re)starts gain cycling:
     ```plaintext
     BBREnterProbeBW():
       BBR.state = ProbeBW
       BBR.pacing_gain = 1
       BBR.cwnd_gain = 2
       BBR.cycle_index = BBRGainCycleLen - 1 - random_int_in_range(0..6)
       BBRAdvanceCyclePhase()
     ```

2. **Cycle Phase Check**:
   - On each ACK, BBR checks if it's time to advance to the next gain cycle phase:
     ```plaintext
     BBRCheckCyclePhase():
       if (BBR.state == ProbeBW and BBRIsNextCyclePhase())
         BBRAdvanceCyclePhase()
     ```

3. **Advance Cycle Phase**:
   - BBR advances to the next gain cycle phase:
     ```plaintext
     BBRAdvanceCyclePhase():
       BBR.cycle_stamp = Now()
       BBR.cycle_index = (BBR.cycle_index + 1) % BBRGainCycleLen
       pacing_gain_cycle = [5/4, 3/4, 1, 1, 1, 1, 1, 1]
       BBR.pacing_gain = pacing_gain_cycle[BBR.cycle_index]
     ```

4. **Cycle Phase Conditions**:
   - BBR determines if it's time to advance to the next cycle phase:
     ```plaintext
     BBRIsNextCyclePhase():
       is_full_length = (Now() - BBR.cycle_stamp) > BBR.RTprop
       if (BBR.pacing_gain == 1)
         return is_full_length
       if (BBR.pacing_gain > 1)
         return is_full_length and
                   (packets_lost > 0 or
                    prior_inflight >= BBRInflight(BBR.pacing_gain))
       else  //  (BBR.pacing_gain < 1)
         return is_full_length or
                    prior_inflight <= BBRInflight(1)
     ```

### Summary

Gain cycling is a crucial part of BBR's ProbeBW state, allowing the algorithm to dynamically adjust its pacing rate to probe for available bandwidth and maintain optimal network performance. By cycling through different pacing gains, BBR can effectively balance high throughput and low queuing delay.

## 3. Measurement

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

## 4. RTT Sample

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