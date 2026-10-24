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

## Comparison with Existing Approaches: How We Differ

Standard robotics platforms rely on software watchdogs, external safety microcontrollers, or heavy software hypervisors. The table below outlines how CrashProof Control differs fundamentally in silicon architecture, failure tolerance, and reaction latency[cite: 1]:

| Metric / Capability | Conventional ROS 2 / Linux Stack | External Safety MCU (e.g., micro-ROS / STM32) | Type-1 Embedded Hypervisor (e.g., Xen / QNX) | **CrashProof Control (PolarFire SoC)** |
| :--- | :--- | :--- | :--- | :--- |
| **Silicon Architecture** | Single-domain CPU (All tasks share OS & memory) | Dual-board topology (Host Linux SBC + external MCU via UART/SPI) | Single multicore SoC with virtualized guest OS partitions | **Heterogeneous single-chip SoC: Hard RISC-V AMP + Hardware FPGA Interposer**[cite: 1] |
| **Actuator Authority** | Software registers written directly by OS drivers | MCU firmware timer peripherals (vulnerable to MCU code lockups) | Hypervisor software scheduling and virtual device drivers | **Pure FPGA logic gates (Zero-Trust Hardware MUX has final arbitration)** |
| **Command Validation Depth** | None (Executes whatever memory contains) | Basic checksums in software; no cycle-by-cycle kinematic boundary enforcement | Software boundary checks subject to CPU preemption and scheduling delays | **Inline hardware verification: monotonic sequence counters, rolling tokens, CRC, and physical slew-rate clamping ($d(\text{PWM})/dt$)** |
| **Response to "Zombie" / Stale Telemetry** | Continues repeating frozen setpoint until crash or physical collision | Stalls while waiting on host-side XRCE-DDS agent timeout loops | Requires software timeout handlers in guest VM | **Dual hardware down-counters drop control authority to degraded reflex mode within $\le 20\text{ ms}$** |
| **Sensor Preservation During OS Crash** | Total sensor starvation until full OS and driver reboot (tens of seconds) | Sensor data buffered on MCU; cannot easily share high-bandwidth telemetry with Linux | Shared I/O virtualized through hypervisor; subject to virtualization overhead | **Hardware-timestamped dual ingest: E51 receives low-latency raw samples continuously; Linux re-links via DMA on reboot** |
| **Failsafe Transition Latency** | Unbounded / Non-existent (Requires manual E-stop slap) | $1\text{ ms} - 10\text{ ms}$ (software interrupt $\rightarrow$ context switch) | Hundreds of microseconds (CPU context-switch dependency) | **$<10\text{ }\mu\text{s}$ (Autonomous RTL FSM switches PWM mux directly)** |
| **Behavior Under Total Software Hang** | Undefined; outputs latch last commanded duty cycle | Internal watchdog triggers full chip reboot (50–500 ms floating-pin blackout) | System-wide kernel panic / hard lockup | **Hardware Safe-State: Clamps to pre-wired fail-passive dynamic braking or engages spring-applied mechanical brakes**[cite: 1] |

### Key Architectural Differentiators

1. **Hardware-Enforced Actuator Interposition (Not Software Watchdogs):**
   Conventional watchdogs only check binary *liveness* ("is the CPU executing instructions?"), not *sanity*. CrashProof Control uses FPGA fabric logic to validate the *contents* of every command packet. Even if Linux is fully functional, an out-of-bounds velocity spike or runaway acceleration curve is clamped by hardware gates before reaching physical motor drivers.

2. **No Dependency on Host Agents (Eliminating the micro-ROS Vulnerability):**
   Auxiliary microcontroller designs rely on a host-side communication daemon (e.g., XRCE-DDS agent). When the host OS kernel panics, the client on the microcontroller spends critical milliseconds handling communication drops and re-negotiations. In CrashProof Control, the real-time core (E51) runs bare-metal against physical hardware FIFOs, completely insulated from the Linux networking stack.

3. **Fail-Operational Degraded Reflexes (Beyond the "Dumb Killswitch"):**
   Cutting power via an E-stop causes moving industrial rovers to skid and robotic arms to drop under gravity. CrashProof Control provides staged degradation: it preserves inertial feedback and motor control to execute an active, controlled S-curve deceleration or dynamic balance hold during high-level computing outages[cite: 1].
