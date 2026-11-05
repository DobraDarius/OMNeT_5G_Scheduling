# 5G Radio Resource Scheduling Model in OMNeT++

**Course:** Computer Aided Design (CAD)  
**Simulation Framework:** OMNeT++ (C++ / NED)  
**Author:** Dobra Darius  

---

## Overview

This project implements a discrete-event simulation model in **OMNeT++** for **radio resource allocation and packet scheduling in 5G New Radio (NR) cellular networks**.

The model evaluates how a Base Station scheduler (**gNB**) dynamically allocates time-frequency **Physical Resource Blocks (PRBs / RBs)** across multiple active User Equipments (**UEs**) every **Transmission Time Interval (TTI)** under varying radio channel conditions, traffic loads, and Quality of Service (QoS) requirements.

---

## System Architecture

The OMNeT++ network model is structured into hierarchical compound and simple modules (`cSimpleModule` subclasses in C++ interconnected via `.ned` topology definitions):

1. **User Equipment (`UE` / User Modules):**
   * **Traffic Generator (`Generator`):** Generates data blocks/packets at pseudo-random time intervals according to configurable probability distributions and 5G service profiles.
   * **User Buffer (`Buffer` / `cQueue`):** Stores incoming data blocks, reports current queue occupancy to the scheduler, and transmits the granted number of data blocks toward the destination upon receiving a scheduling command.
2. **Channel Quality / Radio Conditions Generator (`RadioCond`):**
   * Models time- and frequency-varying wireless channel conditions (path loss, shadowing, multipath fading) via **Channel Quality Indicators (CQI)** and Adaptive Modulation and Coding (**MCS**: BPSK, QPSK, 16-QAM, 64-QAM), determining the achievable transport block size / transfer rate per Resource Block.
3. **5G Base Station Scheduler (`Scheduler` / `gNB`):**
   * Operates periodically at each scheduling cycle (**TTI**).
   * Collects queue lengths and CQI feedback from active users.
   * Allocates the available $B$ Resource Blocks across the $K$ active users using a configurable scheduling algorithm and sends transmission grant commands to the user buffers.
4. **Destination & Statistics Collector (`Sink`):**
   * Receives transmitted data blocks/packets from all users.
   * Computes end-to-end packet/file transmission delays, jitter, and throughput per user and per QoS class/slice, recording scalar statistics and time-series traces.

---

## 5G Traffic Classes & QoS Differentiation

The model accounts for heterogeneous 5G service profiles and QoS classes:

* **URLLC (Ultra-Reliable Low Latency Communications):** Strict latency bounds ($1\text{–}5\text{ ms}$ E2E delay), inelastic real-time traffic requiring highest scheduling priority and short TTIs.
* **eMBB (enhanced Mobile Broadband):** High-throughput traffic (streaming, web browsing, file transfer) sensitive to sustained data rates.
* **mMTC (massive Machine Type Communications):** High connection density IoT/sensor traffic with small, sporadic payloads.
* **Subscription / Priority Tiers:** Economy, Standard, and Premium user weights ($W_i \in \{1, 2, 4, 8\}$).

---

## Scheduling Algorithms

At every scheduling cycle (TTI), Resource Blocks (RBs) are allocated among backlogged users using auction-based or cycle-based scheduling policies, where user $i$ is assigned priority metric $p[i]$:

| Algorithm | Priority Metric $p[i]$ | Characteristics |
| :--- | :--- | :--- |
| **Round Robin (RR)** | $t_{\text{now}} - t_{\text{last\_served}}[i]$ | Equal time-sharing fairness; ignores channel quality and user class. |
| **Weighted Round Robin (WRR)** | $W[i] \cdot (t_{\text{now}} - t_{\text{last\_served}}[i])$ | Provides QoS differentiation proportional to user weight $W[i]$. |
| **Max C/I (Channel-Aware)** | $\text{RadioQuality}[i]$ | Maximizes cell throughput by serving users with the best CQI; can starve cell-edge users. |
| **Proportional Fair (PF)** | $W[i] \cdot (t_{\text{now}} - t_{\text{last\_served}}[i]) \cdot \text{RadioQuality}[i]$ | Balances cell spectral efficiency, multi-user diversity, and fairness acrossQoS classes. |

---

## Validation & Statistical Confidence

Simulation experiments follow a two-stage validation methodology:
1. **Functional Verification (`Qtenv` / `Tkenv`):** Deterministic scenarios and boundary tests (empty queues, single active user, saturation load) to verify grant signaling and queue conservation.
2. **Statistical Evaluation (`Cmdenv` Batch Mode):** Long-duration simulations across $n$ independent runs (using distinct RNG seeds in `omnetpp.ini`) while maintaining queue stability. Results are evaluated using **Confidence Intervals**:

$$\bar{X} \pm Z \cdot \frac{s}{\sqrt{n}}$$

where $\bar{X}$ is the sample mean, $s$ is the sample standard deviation, $n$ is the number of independent simulation runs, and $Z = 1.960$ (for a $95\%$ confidence interval) or $Z = 2.576$ (for a $99\%$ confidence interval).

---

## Prerequisites & Usage

* **OMNeT++** (v5.x / v6.x)
* **C++ Compiler** (Clang / GCC / MinGW bundled with OMNeT++)

### Building and Running
```bash
# Generate makefiles
opp_makemake -f --deep

# Build the simulation binary
make

# Run in GUI mode or batch mode
./PROJECT -u Qtenv omnetpp.ini
./PROJECT -u Cmdenv omnetpp.ini
```
