# CrashProof Control: Zero-Trust Hardware-Interposed Safety Architecture
### Microchip PolarFire® SoC (MPFS250T) — Track 2 Design Contest

---

## System Architecture

### Conceptual Fallback Hierarchy
<img src="CrashProof Control 3-Tier Overview.png" alt="CrashProof Control 3-Tier Overview" width="100%"/>

*Figure 1: Hierarchical runtime assurance ladder showing staged demotion from Tier 1 (Primary Computer) to Tier 2 (Secondary Linux) down to Tier 3 (FPGA Safety FSM).*

### Silicon Implementation & Interconnect Pipeline
<img src="CrashProof Control Detailed Architecture.png" alt="CrashProof Control Detailed Architecture" width="100%"/>

*Figure 2: Microchip PolarFire® SoC interconnect topology showing timestamp ingestion, Ethernet routing, dual liveness monitors, and inline hardware PWM multiplexing.*

## Problem Statement
Robots operating in remote or extreme environments where manual reboots are impossible require general-purpose Linux and ROS 2 compute for perception and planning. These systems routinely suffer two distinct failure modes:

1. **Liveness Failures:** Silent lockups, kernel panics, CPU starvation, and communication blackouts where reboots consume tens of seconds.
2. **Semantic / Byzantine Failures:** The processor remains alive and services standard watchdogs, but due to memory corruption, driver stalls, or dropped network packets, it continuously streams invalid, stale, or dangerous actuator commands.

Conventional watchdogs only detect binary liveness and never validate command content. Dedicated safety-certified PLCs are cost-prohibitive for agile research platforms, leaving robots without an autonomous, in-line command validation mechanism.

---

## Proposed Solution
CrashProof Control implements an open, parameterized, three-tier fallback architecture on the PolarFire SoC Icicle Kit, establishing the FPGA fabric as the ultimate physical authority over actuation:

* **Tier 1 (Primary Navigation Domain — External Host or Network Link):** Executes full navigation, SLAM mapping, and perception under normal operations via Linux and ROS 2 (monitored by 20 ms liveness watchdog and 50 ms command freshness checks).
* **Tier 2 (Fallback Navigation Domain — U54 Cores & E51 Supervisor):** Operates on the PolarFire SoC processor complex. The U54 cores run a secondary Linux instance driving safe fallback navigation (slow crawl, hold, or reverse) with a reduced sensor set. The bare-metal E51 monitor acts as an independent supervisor (2 ms liveness tick) to resynchronize or reset U54 cores during faults.
* **Tier 3 (Trusted Fabric Safety Layer — PolarFire FPGA Logic):** Sits in-line between processors and physical motor drivers. It validates incoming commands across sequence counters, rolling tokens, CRCs, and per-cycle kinematic slew-rate/boundary limits, enforcing safe states via an autonomous FSM.

---

## Autonomous Fallback Flow
Control strictly transitions down the ladder: **Tier 1 $\rightarrow$ Tier 2 $\rightarrow$ Tier 3**.

1. **Independent Timestamp Ingestion:** Physical sensor streams (IMU, wheel encoders, 2D LiDAR, ToF range) enter the FPGA fabric first. The fabric tags samples with 64-bit hardware timestamps and duplicates data for Tier 1 and Tier 2 independently, ensuring fallback compute is never sensor-starved.
2. **Command Validation:** Actuator commands from software must satisfy physical kinematic boundaries. Invalid or stale commands are clipped; repeated breaches trip the state machine to demote control.
3. **Staged Degradation Sequence:**
   * *Tier 1 Stall:* If the primary computer drops its Ethernet link, hangs, or breaches kinematic limits, control automatically shifts to Tier 2 (Secondary Linux with reduced sensor set).
   * *Tier 2 / Supervisor Stall:* If the secondary domain locks up or breaches bounds, the Tier 3 FPGA Safety FSM clamps PWM lines to safe presets and engages brakes in $<10\text{ }\mu\text{s}$.

---

## Distinctive Features
* **Zero-Trust Fabric Authority:** Tier 3 has absolute physical arbitration over motor PWM lines; outputs are guaranteed fail-safe from initial power-on.
* **Content-Aware Gating:** Evaluates command freshness windows (<50 ms), monotonic sequence IDs, CRCs, and physical rate-of-change ($d(\text{PWM})/dt$) entirely in FPGA RTL.
* **Sensor Preservation:** Hardware sensor streams remain fully functional during an OS crash, eliminating sensor starvation during background Linux warm reboots (`kexec`).
* **Hardware Challenge-Response:** Dual independent digital watchdogs monitor both the Linux network thread and the real-time control tick.

---

## Benchmark & Validation Plan
* **Physical Demo:** Dual-axis dynamic ball-and-plate balancing rig subjected to live software fault injection (kernel panics, CPU fork bombs, and stale-packet spam) without losing physical equilibrium.
* **Quantitative Campaign:** An automated 500+ iteration fault-injection testbench benchmarked against PREEMPT_RT and standard AMP baselines measuring:
  * Clamping and safe-state transition latency.
  * Measured control-loop jitter (<50 µs target under 100% Linux load).
  * Peak plant error and false-trip rates across fault distributions.

---

## Deliverables
* Complete Libero SoC RTL IP blocks (parameterized command gate, dual watchdogs, sensor timestamp FIFO, safe-state FSM).
* Bare-metal C runtime for the E51 real-time domain.
* Yocto/Linux BSP configuration and ROS 2 fault-injection test scripts.
* Full technical report and 3–5 minute validation video.

---

## Comparison with Existing Approaches

| Capability / Metric | Standard Linux/ROS 2 | External MCU (e.g., micro-ROS) | Embedded Hypervisor | **CrashProof Control (PolarFire SoC)** |
| :--- | :--- | :--- | :--- | :--- |
| **Silicon Architecture** | Single-domain CPU | Dual-board (SBC + external MCU) | Single SoC with virtualized OS partitions | **Heterogeneous single-chip SoC (Hard RISC-V AMP + FPGA Fabric Interposer)**[cite: 1] |
| **Actuator Authority** | Software drivers | MCU timer registers | Hypervisor virtual drivers | **Pure FPGA logic gates (Zero-trust hardware MUX)** |
| **Command Validation** | None (Executes raw memory) | Basic software checksums | Software boundary checks | **Hardware checks: sequence counters, rolling tokens, CRC, slew-rate clamping** |
| **Reaction to Stale / Zombie Telemetry** | Repeats frozen command | Blocks on agent timeouts | Software timeout handling | **Hardware down-counters drop authority to degraded reflex in $\le 20\text{ ms}$** |
| **Failsafe Transition Latency** | Unbounded (Manual E-stop) | $1\text{ to }10\text{ ms}$ (Context switch) | Hundreds of $\mu\text{s}$ (CPU scheduling) | **$<10\text{ }\mu\text{s}$ (Autonomous RTL FSM switches PWM mux directly)** |
| **Behavior on Total Software Hang** | PWM latches last state | MCU reboots (50–500 ms floating-pin blackout) | System-wide kernel panic | **Hardware safe-state: dynamic braking or spring-applied holding brakes**[cite: 1] |

### Key Differentiators

* **Hardware-Gated Content Validation:** Traditional watchdogs check binary CPU liveness; CrashProof Control validates command content and clamps out-of-bounds kinematic acceleration in hardware logic before signals reach the motor pins.
* **Zero Host-Agent Dependency:** Avoids micro-ROS communication stalls by having the real-time E51 core pull directly from hardware sensor FIFOs, fully isolated from Linux driver crashes.
* **Active Degraded Reflexes:** Rather than cutting power like a killswitch (which causes rolling rovers to skid or robot arms to drop), it preserves inertial sensors to execute controlled S-curve deceleration or balance holds[cite: 1].
