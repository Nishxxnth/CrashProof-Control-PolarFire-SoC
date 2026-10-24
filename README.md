# CrashProof Control: Zero-Trust Hardware-Interposed Safety Architecture
### Microchip PolarFire® SoC (MPFS250T) — Track 2 Design Contest

## Problem Statement
Robots operating in remote or extreme environments where manual reboots are impossible require general-purpose Linux and ROS 2 compute for perception and planning. These systems routinely suffer two distinct failure modes:
1. **Liveness Failures:** Silent lockups, kernel panics, CPU starvation, and communication blackouts where reboots consume tens of seconds.
2. **Semantic / Byzantine Failures:** The processor remains alive and services standard watchdogs, but due to memory corruption, driver stalls, or dropped network packets, it continuously streams invalid, stale, or dangerous actuator commands.

Conventional watchdogs only detect binary liveness and never validate command content. Dedicated safety-certified PLCs are cost-prohibitive for agile research platforms, leaving robots without an autonomous, in-line command validation mechanism.

## Proposed Solution
CrashProof Control implements an open, parameterized, three-tier fallback architecture on the PolarFire SoC Icicle Kit, establishing the FPGA fabric as the ultimate physical authority over actuation:

* **Tier 1 (Untrusted Planning Domain — 4x U54 RISC-V Cores):** Runs Linux and ROS 2 for SLAM, high-level perception, and path planning.
* **Tier 2 (Monitored Real-Time Domain — 1x E51 RISC-V Core):** Runs a deterministic, bare-metal 1 kHz control loop isolated from Linux via RISC-V Physical Memory Protection (PMP). It executes closed-loop stabilization and local degraded reflexes (smooth S-curve deceleration or dynamic balance hold).
* **Tier 3 (Trusted Fabric Safety Layer — 254K FPGA Logic):** Sits in-line between the processors and motor drivers. It validates every command in pure hardware via sequence counters, rolling tokens, CRCs, and per-cycle kinematic slew-rate/boundary checks.

### Autonomous Fallback Flow
1. **Independent Ingestion:** Feedback sensors (IMU, encoders) enter the FPGA fabric first, receiving 64-bit hardware timestamps before distribution to the E51 core (direct low-latency FIFO) and Linux (AXI DMA).
2. **Command Validation:** Actuator commands from software must satisfy physical kinematic boundaries. Invalid or stale commands are clipped; repeated breaches trip the state machine.
3. **Staged Degradation:** 
   * *Tier 1 Stall:* If Linux kernel-panics or drops network heartbeat, authority drops to Tier 2 (E51 reflex braking/balance).
   * *Tier 2 / Bus Failure:* If the real-time core locks or breaches bounds, Tier 3 (FPGA FSM) autonomously clamps PWM lines to hardwired safe presets in <10 µs.

## Distinctive Features
* **Zero-Trust Hardware Authority:** Tier 3 has absolute physical arbitration over motor PWM lines; outputs are guaranteed fail-safe from initial power-on.
* **Content-Aware Gating:** Evaluates command freshness windows (<50 ms), monotonic sequence IDs, CRCs, and physical rate-of-change ($d(\text{PWM})/dt$) entirely in FPGA RTL.
* **Sensor Preservation:** Hardware sensor streams remain fully functional during an OS crash, eliminating sensor starvation during background Linux warm reboots (`kexec`).
* **Hardware Challenge-Response:** Dual independent digital watchdogs monitor both the Linux network thread and the real-time control tick.

## Benchmark & Validation Plan
* **Physical Demo:** Dual-axis dynamic ball-and-plate balancing rig subjected to live software fault injection (kernel panics, CPU fork bombs, and stale-packet spam) without losing physical equilibrium.
* **Quantitative Campaign:** An automated 500+ iteration fault-injection testbench benchmarked against PREEMPT_RT and standard AMP baselines measuring:
  * Clamping and safe-state transition latency.
  * Measured control-loop jitter (<50 µs target under 100% Linux load).
  * Peak plant error and false-trip rates across fault distributions.

## Deliverables
* Complete Libero SoC RTL IP blocks (parameterized command gate, dual watchdogs, sensor timestamp FIFO, safe-state FSM).
* Bare-metal C runtime for the E51 real-time domain.
* Yocto/Linux BSP configuration and ROS 2 fault-injection test scripts.
* Full technical report and 3–5 minute validation video.

## Known Limitations & Design Scope
* **Reference Architecture:** This project demonstrates a functional safety methodology; it does not claim formal IEC 61508 / ISO 13849 certification.
* **Shared Silicon Boundaries:** Tiers 1, 2, and 3 reside on the same MPFS250T die, sharing common silicon power rails and master clock sources.
* **In-Boundary Semantic Errors:** The hardware gate traps kinematic boundary breaches and stale commands; it cannot determine whether an in-boundary trajectory command is globally optimal.
* **Power Blackout:** Complete loss of main battery power requires downstream fail-safe spring brakes or capacitive energy storage for orderly shutdown.
