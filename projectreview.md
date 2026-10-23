# UART Data Framing Sequence Detector: RTL-to-GDSII Project Report

## 1. Project Overview
This project implements a digital sequence detector for UART data framing. The design follows the complete RTL-to-GDSII flow, including Verilog RTL design, simulation, synthesis, gate-level simulation, physical design, sign-off checks, timing analysis, and final GDSII layout generation.

The design uses standard cells only, without macros or memories. It targets a 100 µm × 100 µm die using the SkyWater Sky130 `sky130_fd_sc_hd` process.

**Tool flow:** Yosys (synthesis), OpenSTA (timing), OpenLane v0.21 (physical design), Magic (GDS layout), Netgen (LVS), GTKWave (waveforms).

## 2. Real-World Application
**Problem:** In serial communication protocols like UART, a receiver must identify the start of a data frame by detecting a specific start-of-frame sequence in the incoming bit stream.

**Why dedicated hardware:** A dedicated hardware FSM recognizes the pattern in a single clock cycle, eliminating the latency and CPU overhead of software-based bit-masking and pattern matching.

## 3. Specification
**Inputs and Outputs**
| Signal | Direction | Width | Purpose |
|---|---|---|---|
| clk | Input | 1 | System clock |
| rst_n | Input | 1 | Active-low asynchronous reset |
| din | Input | 1 | Serial data input |
| dout | Output | 1 | Detection pulse (high for 1 cycle) |

## 4. Design and RTL
The design is a 5-state Mealy FSM that detects the overlapping start-of-frame sequence.

| State | Code | Meaning | Next State (din=0) | Next State (din=1) |
|---|---|---|---|---|
| S0 | 000 | Initial / no match | S0 | S1 |
| S1 | 001 | Partial match | S2 | S1 |
| S2 | 010 | Partial match | S0 | S3 |
| S3 | 011 | Partial match | S2 | S4 (DETECT) |
| S4 | 100 | Sequence detected | S2 | S1 |

## 5. RTL Simulation and Verification
A Verilog testbench generates a 10 ns clock, applies reset, and feeds an 11-bit serial stream into the detector.

<img width="1920" height="983" alt="seq_detector_gtk" src="https://github.com/user-attachments/assets/6470732c-35ed-4fc1-b6df-6bd10ee01d7d" />


## 6. Synthesis
The RTL was synthesized using Yosys, targeting the `sky130_fd_sc_hd` typical library.

| Item | Value |
|---|---|
| Total cells | 28 |
| Flip-flops | 3 (state register) |
| Combinational gates | 25 |
| Memories | 0 |

<img width="1920" height="983" alt="seq_detector_gates" src="https://github.com/user-attachments/assets/d5fd6380-f66a-43f7-9619-63d633660b33" />


## 7. Gate-Level Simulation (GLS)
The synthesized netlist was simulated using the Sky130 cell models. A text-trace comparison showed that the RTL and GLS cycle-by-cycle outputs were identical.

## 8. Pre-Layout Timing Analysis
Timing was checked on the synthesized netlist using OpenSTA with an ideal clock.

| Metric | Value |
|---|---|
| WNS | 0.00 ns |
| TNS | 0.00 ns |
| `din` → `dout` path slack | +4.908 ns (MET) |

<img width="1920" height="983" alt="seq_detector_timingsummary" src="https://github.com/user-attachments/assets/1b42447d-d980-427b-948d-40442fd0927e" />


## 9. Physical Design
The physical design was performed using OpenLane v0.21 in its Docker environment with the Sky130A PDK.

| Item | Result |
|---|---|
| Flow status | Completed without fatal errors |
| Die area | 0.01 mm² (100 × 100 µm) |
| TritonRoute violations | 0 |
| Magic DRC violations | 0 |
| LVS errors | 0 (Netgen log confirms 0 mismatches) |

### 9.1 Placement
<img width="1920" height="983" alt="seq_detector_placement" src="https://github.com/user-attachments/assets/1e8a0f33-d7d2-45d2-8a43-b812a059b386" />


### 9.2 Routing
<img width="1920" height="983" alt="seq_detector_routing" src="https://github.com/user-attachments/assets/3639ed80-2899-466d-a841-7ac7e7cf7c5d" />


### 9.3 Final GDS Layout
<img width="1920" height="983" alt="seq_detector_finallayout" src="https://github.com/user-attachments/assets/ad35e727-5621-4ff4-a02d-68806560a78d" />


## 10. Post-Route Timing Analysis
After routing, OpenSTA was run across 21 PVT corners. WNS and TNS are 0.00 ns at all corners. 

<img width="1920" height="983" alt="seq_detector_prelayout" src="https://github.com/user-attachments/assets/4d224c1b-3710-4c54-b345-9a83d0312d30" />


<img width="1920" height="983" alt="seq_detector_wns" src="https://github.com/user-attachments/assets/5b6f3b1d-e86e-4fad-9ab6-97f6d51847d3" />


<img width="1920" height="983" alt="seq_detector_tns" src="https://github.com/user-attachments/assets/d0ae9fbe-3540-4f4f-9dd7-c626bfa0cfec" />


## 11. Conclusion
A UART data framing sequence detector was designed in synthesizable Verilog and verified. The design was synthesized to 28 cells and implemented on a 100 µm × 100 µm die using OpenLane. The flow generated the final GDSII layout.
