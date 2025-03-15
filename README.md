C++ implementation of the BBR algorithm:
This document provides an overview of the BBR algorithm, including its key states, transitions, and mechanisms. It also includes a brief explanation of how pacing works in BBR. 

## BBR Overview

* **Goals**
  * Full pipe the network path
  * Low queue pressure (low delay, low packet loss)
* **Workflow**:

  Startup → Drain → ProbeBW → ProbeRTT
* **Key paramters**:
  * BBR.bw
  * BBR.min_rtt
  * BBR.cwnd

BBR.min_rtt is initialized in Startup(SRTT), and then be updated in ProbeRTT.

## BBR Startup

- **Purpose**: Exponential bandwidth discovery (pacing_gain = 2.77, cwnd_gain = 2.0)
  - Rapidly discover available bandwidth through exponential probing
  - Double sending rate each round trip
  - Explore bandwidth space in O(log₂(BDP)) round trips
- **Duration**: Variable
  - Depends on network conditions and path BDP
  - Typically takes log₂(BDP) round trips
  - Example: A 10Gbps × 100ms link (BDP of \~82,500 packets) requires \~17 RTTs
- **Exit Conditions**:
  - **Bandwidth Plateau**: BBR.max_bw growth rate falls below 1.25× in a round
  - **Full Pipe Detection**: No significant bandwidth increase for 3 consecutive rounds
  - **Excessive Loss**: Loss rate exceeds BBRLossThresh (2%)
- **Transitions to**: Drain
  - Immediately transitions to Drain to quickly eliminate queue
  - Sets pacing_gain = 1/BBRStartupPacingGain (0.5)
  - Maintains cwnd_gain = BBRStartupCwndGain (2.0)

Startup creates a queue of approximately 1.0 × BDP by the time it completes, which the subsequent Drain state is designed to quickly eliminate before entering the steady-state ProbeBW cycle.

## BBR ProbeBW

ProbeBW is BBR's steady-state operation mode, designed to periodically probe for additional available bandwidth while maintaining high throughput and low latency.

**It's also designed to spend the vast majority of its time(about 96%) to probe the bandwidth.**

### **1. Entry Conditions**

- After Drain phase completes (when inflight ≤ BDP)
- OR after ProbeRTT phase completes
- When restarting after idle if pipe was previously filled

### 2. Workflow

ProbeBW uses a four-state cycle to efficiently probe for bandwidth while maintaining optimal queue management:

**ProbeBW_DOWN** (0.9) → 2. **ProbeBW_CRUISE** (1.0) → 3. **ProbeBW_REFILL** (1.0) → 4. **ProbeBW_UP** (1.25) → repeat

### 3. Key States

#### 3.1 ProbeBW_DOWN

- **Purpose**: Deceleration tactic (pacing_gain = 0.9), drains any queue until inflight ≤ BDP
- **Duration**: Variable
- **Exit Conditions**:
  - Inflight data is less than or equal to BBR.BDP (estimated bandwidth-delay product)
  - AND there is sufficient headroom (inflight ≤ BBRHeadroom\*BBR.inflight_hi)
- **Transitions to**: ProbeBW_CRUISE

#### 3.2. ProbeBW_CRUISE

- **Purpose**: Cruising tactic (pacing_gain = 1.0), maintains steady throughput at estimated bandwidth
- **Duration**: Adaptively determined
- **Exit Conditions**: When it's time to probe for bandwidth, based on either:
  - Wall clock time: T_bbr = 2-3 seconds (randomized)
  - OR Round-trip count: T_reno = min(estimated_BDP, cwnd) round trips (capped at 62-63 round trips)
- **Transitions to**: ProbeBW_REFILL

#### 3.3. ProbeBW_REFILL

- **Purpose**: Refill the pipe (pacing_gain = 1.0), ensures pipe is full before probing
- **Duration**: Exactly one packet-timed round trip
- **Exit Condition**: One round trip has elapsed
- **Transitions to**: ProbeBW_UP

#### 3.4. ProbeBW_UP

- **Purpose**: Probe for bandwidth increases (pacing_gain = 1.25); probes for additional available bandwidth
- **Duration**: At least BBR.min_rtt, potentially longer
- **Exit Conditions**:
  - At least BBR.min_rtt has elapsed AND inflight \> 1.25 \* BBR.bdp
  - OR loss rate exceeds BBRLossThresh (2%)
- **Transitions to**: ProbeBW_DOWN (completing the cycle)

### **4. Key Mechanisms**

- **Time-scale**: Probes approximately every 2-3 seconds for bandwidth changes
- **Bounds Management**: Updates and respects bw_hi, bw_lo, inflight_hi, inflight_lo
- **Loss Response**: Reduces inflight_hi if loss rate exceeds threshold (2%)
- **Bandwidth Estimation**: Updates BBR.max_bw when higher non-app-limited rates observed


## BBR ProbeRTT

Most of the time, BBR is probing the bandwidth and rest of it (4%) is probing the min_rtt.

- **Purpose**: Periodically drain the bottleneck queue to obtain a clean RTprop measurement (pacing_gain = 1.0)
  - Reduces data in flight to minimum value
  - Creates opportunity to measure true propagation delay without queuing
  - Updates BBR's RTprop estimate
- **Duration**: Fixed minimum time
  - Lasts for at least ProbeRTTDuration (200 milliseconds)
  - Continues until at least one ACK is received during this period
  - Total throughput penalty is approximately 2%
- **Entry Conditions**:
  - At least ProbeRTTInterval (5 seconds) has elapsed since last BBR.min_rtt update
  - Current state is not already ProbeRTT
  - Connection is not restarting from idle (BBR.idle_restart is false)
- **Key Parameters**:
  - cwnd = min_cwnd (typically 4 packets)
  - pacing_gain = 1.0
  - cwnd_gain = 1.0
- **Time Parameters**
  - **ProbeRTTInterval**: 5 seconds (time between potential ProbeRTT states)
  - **MinRTTFilterLen**: 10 seconds (moving window for min_rtt tracking:
  - **ProbeRTTDuration**: 200 milliseconds (minimum time spent in ProbeRTT)
- **Exit Conditions**:
  - Connection has spent at least ProbeRTTDuration (200ms) in ProbeRTT state
  - AND at least one ACK has been received during this period
- **Transitions to**: ProbeBW (specifically ProbeBW_CRUISE)


## Pacing with Linux tc

When using `SO_TXTIME` with `sendmmsg`, it needs a 64-bit unsigned integer (`__u64`) timestamp in nanoseconds using the clock source that is specified when setting the `SO_TXTIME` socket option (typically `CLOCK_MONOTONIC`).

```c
#include <linux/net_tstamp.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <time.h>

// First, set up SO_TXTIME on socket
struct sock_txtime txtime_options;
txtime_options.clockid = CLOCK_MONOTONIC; // Specify which clock to use
txtime_options.flags = 0;                 // Or SOCK_TXTIME_DEADLINE for deadline mode

if (setsockopt(sockfd, SOL_SOCKET, SO_TXTIME, 
               &txtime_options, sizeof(txtime_options)) < 0) {
    perror("setsockopt SO_TXTIME");
    return -1;
}

// Then, when sending with sendmmsg:
struct mmsghdr msgs[NUM_MSGS];
struct iovec iovecs[NUM_MSGS];
char control[NUM_MSGS][CMSG_SPACE(sizeof(__u64))];

// For each message:
for (int i = 0; i < NUM_MSGS; i++) {
    // Get current time
    struct timespec now;
    clock_gettime(CLOCK_MONOTONIC, &now);
    
    // Set transmission time (now + delay_ns)
    __u64 txtime = now.tv_sec * 1000000000ULL + now.tv_nsec + delay_ns;
    
    // Prepare message
    memset(&msgs[i], 0, sizeof(struct mmsghdr));
    memset(control[i], 0, CMSG_SPACE(sizeof(__u64)));
    
    // Set up iovec with data
    iovecs[i].iov_base = buffer;
    iovecs[i].iov_len = buffer_len;
    
    // Set up the message header
    msgs[i].msg_hdr.msg_iov = &iovecs[i];
    msgs[i].msg_hdr.msg_iovlen = 1;
    msgs[i].msg_hdr.msg_control = control[i];
    msgs[i].msg_hdr.msg_controllen = CMSG_SPACE(sizeof(__u64));
    
    // Add control message with txtime
    struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msgs[i].msg_hdr);
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type = SCM_TXTIME;
    cmsg->cmsg_len = CMSG_LEN(sizeof(__u64));
    *(__u64 *)CMSG_DATA(cmsg) = txtime;
}

// Send all messages
int ret = sendmmsg(sockfd, msgs, NUM_MSGS, 0);
```

The key points to remember:

1. The timestamp is in nanoseconds since the epoch of the specified clock source
2. Must use the same clock ID that be specified when setting up the socket option. 
3. The timestamp is passed as ancillary data with `cmsg_type = SCM_TXTIME`
4. If BBR implementation uses `SO_TXTIME`, it will calculate and set these timestamps based on its pacing algorithm.