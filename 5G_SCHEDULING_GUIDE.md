# Comprehensive Guide: 5G Scheduling & OMNeT++ Simulation Modeling

> **Course Reference:** Computer Aided Design (CAD), Lectures 8–11 & Confidence Intervals  
> **Target Platform:** OMNeT++ (Discrete Event Simulation in C++ and NED)  
> **Author / Student:** Dobra Darius  

---

## Table of Contents
1. [Introduction to Radio Resource Scheduling](#1-introduction-to-radio-resource-scheduling)
2. [5G NR Physical Layer & Resource Grid Fundamentals](#2-5g-nr-physical-layer--resource-grid-fundamentals)
3. [5G Traffic Profiles & Quality of Service (QoS)](#3-5g-traffic-profiles--quality-of-service-qos)
4. [Scheduling Algorithms: Theory, Formulas & Trade-offs](#4-scheduling-algorithms-theory-formulas--trade-offs)
5. [Designing the OMNeT++ Simulation Architecture](#5-designing-the-omnet-simulation-architecture)
6. [OMNeT++ Implementation Patterns & Code Skeletons](#6-omnet-implementation-patterns--code-skeletons)
7. [Validation, Stability & Statistical Confidence Intervals](#7-validation-stability--statistical-confidence-intervals)
8. [Next Steps & Project Scope Alignment](#8-next-steps--project-scope-alignment)

---

## 1. Introduction to Radio Resource Scheduling

### 1.1 What is Scheduling in Cellular Networks?
In wireless cellular communications (from GPRS/UMTS up to 5G NR), radio spectrum is a scarce, shared, and expensive medium. Multiple active user equipments (**UEs**) located within a cell need to transmit or receive data concurrently. 

Rather than dedicating a fixed frequency or time slice permanently to each user (which causes immense bandwidth waste during silence periods), cellular networks utilize **Packet Switching (PS)** and **dynamic resource allocation**:
* A central controller at the Base Station (in 5G called the **gNodeB** or **gNB**, and in older systems called the **PCU / BSC / eNB**) makes decisions every scheduling cycle.
* This scheduling cycle is known as the **Transmission Time Interval (TTI)**.
* The scheduler evaluates user requirements, buffer status, and radio link conditions, and grants radio resources to users dynamically.

```
       +---------------------------------------------------------+
       |                  5G Base Station (gNB)                  |
       |                                                         |
       |  +---------------------------------------------------+  |
       |  |               MAC Scheduler Module                |  |
       |  |  - Reads Buffer Status Reports (BSR)              |  |
       |  |  - Reads Channel Quality Indicators (CQI)         |  |
       |  |  - Runs Scheduling Policy (RR / WRR / PF / SPS)   |  |
       |  |  - Allocates Resource Blocks (PRBs)               |  |
       |  +-------------------------+-------------------------+  |
       +----------------------------|----------------------------+
                                    | Scheduling Grants (every TTI)
             +----------------------+----------------------+
             |                      |                      |
             v                      v                      v
        +---------+            +---------+            +---------+
        |  UE #1  |            |  UE #2  |            |  UE #K  |
        | (eMBB)  |            | (URLLC) |            | (mMTC)  |
        +---------+            +---------+            +---------+
```

### 1.2 Evolution Across Generations (Course Reference: CAD8 & CAD11)
* **2G / 2.5G GPRS:** Circuit-switched voice overlaid with packet data. TTI was fixed at **20 ms** (1 radio block = 4 GSM bursts). Users shared 8 Time Slots per carrier.
* **3G UMTS:** CDMA-based. TTI was typically **10 ms** (shortened to 2 ms in HSDPA). Resources were allocated by codes and power.
* **4G LTE:** Flat architecture, orthogonal frequency-division multiplexing (OFDMA). Fixed TTI of **1.0 ms**, subcarrier spacing of $15\text{ kHz}$. Resource granularity: **Physical Resource Block (PRB)** = $180\text{ kHz} \times 1\text{ ms}$.
* **5G NR (New Radio):** Designed for extreme diversity in requirements. Introduces **flexible numerologies** ($\mu \in \{0, 1, 2, 3, 4\}$), sub-millisecond TTIs (as short as $0.125\text{ ms}$ or mini-slots for URLLC), wider carrier bandwidths ($100\text{ MHz}$ to mmWave $400\text{ MHz}$), and network slicing.

---

## 2. 5G NR Physical Layer & Resource Grid Fundamentals

### 2.1 The Time-Frequency Grid
5G New Radio schedules transmissions across a two-dimensional grid of **Frequency** and **Time**:

```
Frequency ^
          |  +---------+---------+---------+---------+
          |  |  PRB 0  |  PRB 0  |  PRB 0  |  PRB 0  |
  180 kHz |  +---------+---------+---------+---------+
 (12 sc)  |  |  PRB 1  |  PRB 1  |  PRB 1  |  PRB 1  |
          |  +---------+---------+---------+---------+
          |  |  PRB 2  |  PRB 2  |  PRB 2  |  PRB 2  |
          |  +---------+---------+---------+---------+
          |  |   ...   |   ...   |   ...   |   ...   |
          +--+---------+---------+---------+---------+--->
             <--TTI 0--><--TTI 1--><--TTI 2--><--TTI 3-->  Time
```

* **Subcarrier Spacing (SCS):** In LTE, SCS is fixed at $15\text{ kHz}$. In 5G NR, SCS is scalable:
  $$\Delta f = 2^\mu \times 15\text{ kHz}$$
  Where $\mu = 0$ ($15\text{ kHz}$, TTI = $1\text{ ms}$), $\mu = 1$ ($30\text{ kHz}$, TTI = $0.5\text{ ms}$), $\mu = 2$ ($60\text{ kHz}$, TTI = $0.25\text{ ms}$), etc.
* **Physical Resource Block (PRB):** The minimum allocatable unit in frequency. Consists of 12 consecutive subcarriers. At $\Delta f = 15\text{ kHz}$, $12 \times 15 = 180\text{ kHz}$.
* **Transmission Time Interval (TTI):** The time duration over which a scheduling grant applies.

### 2.2 Wireless Channel Imperfections (Course Reference: CAD10)
Radio signals do not travel along a single clean path; they suffer from multiple physical phenomena:
1. **Path Loss:** The average signal power drops with distance ($d$), proportional to $1/d^\alpha$ (where $\alpha \ge 2$).
2. **Shadowing (Slow Fading):** Caused by large obstacles (buildings, hills) blocking the line of sight, typically modeled as a log-normal distribution.
3. **Multipath Propagation (Fast Fading):** Signals reflect off surfaces, arriving at the receiver with differing delays and phase shifts. At specific points, waves cancel destructively (deep fades) or add constructively.
4. **Frequency-Selective Fading:** Different frequency subcarriers experience different fading levels simultaneously. A channel bad on PRB #1 may be excellent on PRB #5.

### 2.3 Adaptive Modulation and Coding (AMC)
Depending on the signal-to-noise-and-interference ratio (SINR), the UE reports a **Channel Quality Indicator (CQI)** to the gNB. The gNB translates this into a **Modulation and Coding Scheme (MCS)**:

| CQI Index | Modulation | Code Rate Approx. | Bits / Symbol | Efficiency / Capacity |
| :---: | :---: | :---: | :---: | :---: |
| 1–3 | QPSK | $\sim 1/3$ | 2 | Very robust, low data rate |
| 4–6 | QPSK / 16-QAM | $\sim 1/2$ | 2–4 | Medium robustness |
| 7–9 | 16-QAM / 64-QAM | $\sim 2/3$ | 4–6 | High throughput |
| 10–15 | 64-QAM / 256-QAM | $\sim 3/4$–$5/6$ | 6–8 | Maximum data rate (requires high SINR) |

*In simulation models, AMC allows us to calculate how many data blocks or bytes 1 PRB can carry for user $i$ in TTI $t$:*
$$\text{Capacity}(i, r, t) = \text{MCS\_to\_Capacity}(\text{CQI}_i(r, t))$$

---

## 3. 5G Traffic Profiles & Quality of Service (QoS)

In 5G NR (CAD11, Slide 72), three distinct service pillars coexist on the same physical infrastructure (**Network Slicing**):

```
                       5G Service Triad
                       ----------------
                             /\
                            /  \
                           /    \
                          / eMBB \
                         /________\
                        / \      / \
                       /   \    /   \
                      /     \  /     \
                     / URLLC \/  mMTC \
                    /_________\________\
```

### 3.1 The Three 5G Use Cases
1. **eMBB (Enhanced Mobile Broadband):**
   * *Examples:* 4K/8K video streaming, virtual reality, high-speed file download.
   * *Characteristics:* High throughput (hundreds of Mbps), bursty/elastic traffic, tolerant to moderate delays (e.g., $10\text{–}50\text{ ms}$).
2. **URLLC (Ultra-Reliable Low-Latency Communications):**
   * *Examples:* Autonomous driving (V2X), remote robotic surgery, industrial control loops.
   * *Characteristics:* Extremely small delay tolerance ($<1\text{ ms}$ radio latency, $<5\text{ ms}$ E2E), near-zero packet loss ($99.999\%$ reliability), inelastic traffic.
3. **mMTC (Massive Machine-Type Communications):**
   * *Examples:* Smart utility meters, urban sensor networks, logistics tracking.
   * *Characteristics:* Massive device density, tiny payload sizes (e.g., tens of bytes), infrequent transmissions, battery-life conservation.

### 3.2 QoS Parameters & Traffic Classification (CAD9)
* **Delay / Latency:** Time elapsed between packet generation at the source and reception at the destination:
  $$D = t_{\text{rx}} - t_{\text{tx}} = D_{\text{queue}} + D_{\text{trans}} + D_{\text{prop}}$$
* **Jitter:** Delay variation between consecutive packets ($|D_k - D_{k-1}|$), crucial for streaming media.
* **Throughput:** Bits or data blocks successfully delivered per unit time.
* **Packet Loss Rate / Drop Precedence:** Packets dropped due to buffer overflow or missed deadlines.

---

## 4. Scheduling Algorithms: Theory, Formulas & Trade-offs

At each TTI, the scheduler must distribute $B$ Resource Blocks among $K$ active users with backlogged queues. 

The CAD course highlights an **auction-based scheduling framework** (CAD8 slides 49–51, CAD11 slide 68): each candidate user calculates a priority metric $p[i]$, and the user with the highest $p[i]$ is awarded the resource block.

```
       For each PRB r in (1 ... B):
           For each user i with Queue(i) > 0:
               Compute priority metric p[i]
           Winner u* = argmax( p[i] )
           Allocate PRB r to u*
           Update state (served bits, last served time, queue length)
```

### 4.1 Round Robin (RR)
* **Principle:** Equal opportunity in time. Users take turns sequentially.
* **Course Priority Metric:**
  $$p_{\text{RR}}[i] = t_{\text{now}} - t_{\text{last\_served}}[i]$$
  *(The user that has waited longest since its last grant gets highest priority).*
* **Advantages:** Perfect time fairness; starvation is impossible.
* **Drawbacks:** Completely channel-unaware and QoS-unaware. A user in a deep fade receives the resource block even though it transmits almost zero payload, destroying cell spectral efficiency.

### 4.2 Weighted Round Robin (WRR)
* **Principle:** Differentiates user subscriptions or QoS priority classes by assigning each user $i$ a weight $W[i] \ge 1$ (e.g., Economy = 1, Standard = 2, Premium = 4).
* **Course Priority Metric:**
  $$p_{\text{WRR}}[i] = W[i] \cdot \left( t_{\text{now}} - t_{\text{last\_served}}[i] \right)$$
* **Advantages:** Enforces proportional service rates according to user priority.
* **Drawbacks:** Still channel-unaware; ignores channel fading.

### 4.3 Maximum C/I (Maximum Rate / Greedy Scheduler)
* **Principle:** Always grants the PRB to the user with the best instantaneous radio condition.
* **Priority Metric:**
  $$p_{\text{MaxCI}}[i] = R_i(t) \quad \text{or} \quad \text{CQI}_i(t)$$
* **Advantages:** Maximizes total cell throughput (spectral efficiency).
* **Drawbacks:** Severe unfairness. Users located near the cell edge with lower SINR will experience complete starvation.

### 4.4 Proportional Fair (PF)
* **Principle:** The gold standard of cellular scheduling. Balances multi-user diversity (serving users on their channel peaks) with fairness.
* **Classical Formulation:**
  $$p_{\text{PF}}[i] = \frac{R_i(t)}{\bar{R}_i(t)}$$
  where $R_i(t)$ is instantaneous achievable rate and $\bar{R}_i(t)$ is the exponentially smoothed past average throughput of user $i$.
* **CAD Course Formulation (CAD8 Slide 50):**
  $$p_{\text{PF}}[i] = W[i] \cdot \left( t_{\text{now}} - t_{\text{last\_served}}[i] \right) \cdot \text{RadioQuality}[i]$$
  *Notice:* The term $(t_{\text{now}} - t_{\text{last\_served}}[i])$ increases as a user waits (preventing starvation), while $\text{RadioQuality}[i]$ rewards users experiencing channel peaks.

### 4.5 Static Priority / Preemptive Scheduling for 5G URLLC
* **Principle:** Strict priority queueing (CAD9 Slide 34).
* URLLC traffic cannot wait for a normal round-robin cycle. When a URLLC packet arrives:
  1. It is served immediately at the start of the TTI ahead of all eMBB/mMTC traffic.
  2. If all PRBs were assigned to ongoing eMBB transmissions, 5G NR supports **preemption/puncturing**, where URLLC overrides an ongoing eMBB block.

### Summary Comparison Table
| Algorithm | Metric $p[i]$ | Cell Throughput | Fairness | QoS Differentiation | Complexity |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Round Robin (RR)** | $t_{\text{now}} - t_{\text{last\_served}}[i]$ | Low | High | None | $O(K)$ |
| **Weighted Round Robin (WRR)**| $W[i] \cdot (t_{\text{now}} - t_{\text{last\_served}}[i])$ | Low | Proportional | High | $O(K)$ |
| **Max C/I** | $\text{CQI}_i(t)$ | Maximum | Terrible (starvation) | None | $O(K \log K)$ |
| **Proportional Fair (PF)** | $W[i] \cdot \Delta t \cdot \text{CQI}_i(t)$ | High | High | Balanced | $O(K \log K)$ |
| **Priority + PF (5G Slicing)** | High priority to URLLC, PF for eMBB | Optimal for delay | Strict for URLLC | Maximum | $O(K \log K)$ |

---

## 5. Designing the OMNeT++ Simulation Architecture

An OMNeT++ model maps directly to discrete real-world modules. Based on CAD8 (Slides 15, 20–25, 43–47), the network consists of the following components:

```
+-----------------------------------------------------------------------------------+
| Network: CellularCell                                                             |
|                                                                                   |
|  +-------------------------+             +-------------------------------------+  |
|  | User[0]                 |             |              Scheduler              |  |
|  |  +-------------------+  |             |                                     |  |
|  |  | TrafficGenerator  |  |             |  - Runs every TTI (1 ms)            |  |
|  |  +---------+---------+  |             |  - Evaluates queue status & CQI     |  |
|  |            | (packets)  |             |  - Generates grants                 |  |
|  |  +---------v---------+  | BSR (queue) |                                     |  |
|  |  |    UserBuffer     |--+------------>|                                     |  |
|  |  +---------+---------+  |<------------+                                     |  |
|  +------------|------------+ Grant(blocks)|                                     |  |
|               |                          |                                     |  |
|  +------------|------------+             |  +-------------------------------+  |  |
|  | User[1]    |            |             |  | ChannelModel / RadioCond      |  |  |
|  |            |            |             |  | - Distance path loss          |  |  |
|  |            |            |             |  | - Rayleigh / Markov fading    |  |  |
|  |            |            |             |  +---------------+---------------+  |  |
|  +------------|------------+                                |                  |  |
|               |                                             | Updates CQI      |  |
|               | (transmitted data packets)                  v                  |  |
|               +------------------------------------->+---------------+         |  |
|                                                      |     Sink      |         |  |
|                                                      | - Delays      |         |  |
|                                                      | - Throughput  |         |  |
|                                                      | - Statistics  |         |  |
|                                                      +---------------+         |  |
+-----------------------------------------------------------------------------------+
```

### 5.1 Component Responsibilities
1. **`TrafficGenerator` (`cSimpleModule`):**
   * Produces `DataPacket` messages according to configured distributions (e.g., exponential inter-arrival for Poisson processes, Pareto for heavy-tailed web traffic).
   * Sets packet attributes: `creationTime`, `packetId`, `packetSizeBytes`, `trafficClass` (URLLC / eMBB / mMTC).
2. **`UserBuffer` (`cSimpleModule`):**
   * Stores received packets inside an internal `omnetpp::cQueue`.
   * Notifies or exposes its queue length (in packets or blocks) to the scheduler.
   * Upon receiving a `GrantMessage` indicating $N$ blocks granted, pops packets from `cQueue` and sends them across the transmission channel to the `Sink`.
3. **`RadioCond / ChannelModel` (`cSimpleModule` or internal helper):**
   * Generates time-varying radio conditions for each user.
   * Can be modeled as a multi-state Markov chain (Good / Average / Bad) or Rayleigh/Rician fading with Doppler shift.
4. **`Scheduler / gNB` (`cSimpleModule`):**
   * Triggered periodically by a self-message every `tti` (e.g., $1\text{ ms}$).
   * Inspects all active users, computes priority metrics $p[i]$, and determines RB allocations.
   * Sends `GrantMessage` to the winning user buffers.
5. **`Sink` (`cSimpleModule`):**
   * Acts as the data receiver.
   * Calculates end-to-end delay: $\text{delay} = \text{simTime}() - \text{msg}\to\text{getCreationTime}()$.
   * Records scalars, vectors, and histograms via OMNeT++ statistics recording (`@statistic`, `cStdDev`, `emit()`).

---

## 6. OMNeT++ Implementation Patterns & Code Skeletons

### 6.1 Custom Messages (`DataMessages.msg`)
```msg
// DataMessages.msg
enum TrafficType {
    TRAFFIC_URLLC = 0;
    TRAFFIC_EMBB  = 1;
    TRAFFIC_MMTC  = 2;
};

packet DataPacket {
    int sourceUserId;
    int packetId;
    int trafficType;       // enum TrafficType
    simtime_t generationTime;
    int sizeInBlocks = 1;
}

packet GrantMessage {
    int grantedBlocks;
    int allocatedPRB;
}
```

### 6.2 Periodic TTI Scheduling in C++
In OMNeT++, periodic events are scheduled using self-messages:

```cpp
#include <omnetpp.h>
#include "DataMessages_m.h"

using namespace omnetpp;

class Scheduler : public cSimpleModule {
  private:
    cMessage *ttiTimer = nullptr;
    simtime_t ttiPeriod;
    int totalPRBs = 25;       // E.g., 25 PRBs for 5 MHz channel
    int numUsers = 0;
    std::string algorithm;

  protected:
    virtual void initialize() override {
        ttiPeriod = par("ttiPeriod");
        algorithm = par("algorithm").stdstringValue();
        numUsers = par("numUsers");

        ttiTimer = new cMessage("ttiTimer");
        scheduleAt(simTime() + ttiPeriod, ttiTimer);
    }

    virtual void handleMessage(cMessage *msg) override {
        if (msg == ttiTimer) {
            runSchedulingCycle();
            scheduleAt(simTime() + ttiPeriod, ttiTimer);
        } else {
            delete msg;
        }
    }

    void runSchedulingCycle() {
        // 1. Gather status from all UEs (queue length & CQI)
        // 2. Allocate PRBs according to chosen algorithm
        // 3. Dispatch GrantMessage to selected UserBuffers
    }

    virtual void finish() override {
        cancelAndDelete(ttiTimer);
    }
};
Define_Module(Scheduler);
```

### 6.3 Proportional Fair Auction Allocation Logic
```cpp
void Scheduler::runProportionalFairAuction() {
    for (int prb = 0; prb < totalPRBs; ++prb) {
        int bestUser = -1;
        double highestPriority = -1.0;

        for (int i = 0; i < numUsers; ++i) {
            int queueLen = getQueueLength(i);
            if (queueLen <= 0) continue; // User has no backlog

            double weight = getUserWeight(i);
            simtime_t waitTime = simTime() - getLastServedTime(i);
            double cqiFactor = getInstantaneousCQI(i, prb);

            // Proportional Fair metric (CAD8 & CAD11)
            double priority = weight * waitTime.dbl() * cqiFactor;

            if (priority > highestPriority) {
                highestPriority = priority;
                bestUser = i;
            }
        }

        if (bestUser != -1) {
            assignPRBToUser(bestUser, prb);
        }
    }
}
```

---

## 7. Validation, Stability & Statistical Confidence Intervals

### 7.1 Queue Stability (CAD8 Slide 11)
Before running extensive simulations, verify that the system is **stable**:
$$\text{Total Arrival Rate } (\Lambda = \sum \lambda_i) < \text{Maximum Total Service Rate } (\mu_{\text{cell}})$$
* If arrival rate exceeds service capacity, the queue grows infinitely.
* As the simulation runs longer, queue delay increases unbounded, making average delay metrics meaningless!

### 7.2 Functional Validation Phase (CAD8 Slide 10)
1. **Interactive Mode (`Qtenv`):** Use the graphical UI with single-step message delivery.
2. **Deterministic Scenarios:** Fix packet inter-arrival times (no random generation initially).
3. **Corner Cases:**
   * All user queues empty $\rightarrow$ scheduler idles cleanly.
   * Only one user has packets $\rightarrow$ scheduler grants all PRBs to that user.
   * Two users with identical parameters $\rightarrow$ grants alternate equally.

### 7.3 Batch Simulation & Confidence Intervals (ConfidenceInterv.pdf)
Because simulations use pseudo-random distributions, a single run represents only one realization of a random variable. To draw valid scientific conclusions, we run $n$ independent simulation runs ($n \ge 30$, each with a different RNG seed).

Let $X_1, X_2, \dots, X_n$ be the average delay measured in each run.
1. **Sample Mean ($\bar{X}$):**
   $$\bar{X} = \frac{1}{n} \sum_{i=1}^n X_i$$
2. **Sample Standard Deviation ($s$):**
   $$s = \sqrt{\frac{1}{n-1} \sum_{i=1}^n (X_i - \bar{X})^2}$$
3. **$95\%$ Confidence Interval:**
   $$\text{CI}_{95\%} = \bar{X} \pm 1.960 \cdot \frac{s}{\sqrt{n}}$$

#### Worked Example:
Suppose we run $n = 36$ simulation runs for Proportional Fair scheduling. We measure the average packet delay (in ms):
* Sample mean: $\bar{X} = 12.40\text{ ms}$
* Sample standard deviation: $s = 1.80\text{ ms}$
* Standard error: $\frac{s}{\sqrt{n}} = \frac{1.80}{\sqrt{36}} = \frac{1.80}{6} = 0.30\text{ ms}$
* Margin of error ($95\%$): $Z \times \frac{s}{\sqrt{n}} = 1.960 \times 0.30 = 0.588\text{ ms}$
* Result: With $95\%$ statistical confidence, the true average delay is:
  $$12.40\text{ ms} \pm 0.59\text{ ms} \quad [11.81\text{ ms}, 12.99\text{ ms}]$$

---

## 8. Next Steps & Project Scope Alignment

This learning document outlines all core principles from **CAD8 through CAD11** and the **Confidence Interval** guide.

To begin building the OMNeT++ model, we should decide on:
1. **The exact set of scheduling algorithms to compare** (e.g., RR vs. WRR vs. Max C/I vs. Proportional Fair vs. URLLC Preemptive).
2. **The number of active UEs and traffic mix** (e.g., 6 UEs: 2 URLLC, 2 eMBB, 2 mMTC).
3. **The communication mechanism** between modules (OMNeT++ direct C++ method calls vs. message passing).
4. **Channel conditions model** (simple random CQI generator vs. multi-state Markov model).
