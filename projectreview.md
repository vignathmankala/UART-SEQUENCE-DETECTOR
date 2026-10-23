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

### RTL Code (`seq_detector.v`)
```verilog
module seq_detector (
    input  wire clk,
    input  wire rst_n,
    input  wire din,
    output reg  dout
);

    localparam S0 = 3'b000,
               S1 = 3'b001,
               S2 = 3'b010,
               S3 = 3'b011,
               S4 = 3'b100;

    reg [2:0] state, next_state;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            state <= S0;
        else
            state <= next_state;
    end

    always @(*) begin
        next_state = S0;
        dout       = 1'b0;

        case (state)
            S0: next_state = din ? S1 : S0;
            S1: next_state = din ? S1 : S2;
            S2: next_state = din ? S3 : S0;
            S3: begin
                if (din) begin
                    next_state = S4;
                    dout       = 1'b1;
                end else
                    next_state = S2;
            end
            S4: next_state = din ? S1 : S2;
            default: next_state = S0;
        endcase
    end
endmodule
```

Testbench Code (seq_detector_tb.v)

```verilog
`timescale 1ns/1ps

module seq_detector_tb;
    reg  clk, rst_n, din;
    wire dout;

    seq_detector dut (
        .clk(clk), .rst_n(rst_n), .din(din), .dout(dout)
    );

    initial clk = 0;
    always #5 clk = ~clk;

    reg [10:0] data_stream = 11'b10110110111;
    integer i;

    initial begin
        $dumpfile("seq_detector.vcd");
        $dumpvars(0, seq_detector_tb);

        rst_n = 0; din = 0;
        #20 rst_n = 1;

        for (i = 10; i >= 0; i = i - 1) begin
            @(posedge clk);
            din = data_stream[i];
        end

        @(posedge clk); din = 0;
        #50 $finish;
    end

    initial begin
        $monitor("T=%0t  din=%b  state=%b  dout=%b",
                 $time, din, dut.state, dout);
    end
endmodule
```

5. RTL Simulation and Verification

A Verilog testbench generates a 10 ns clock, applies reset, and feeds an 11-bit serial stream into the detector.

<img width="1920" height="983" alt="seq_detector_gtk" src="https://github.com/user-attachments/assets/6470732c-35ed-4fc1-b6df-6bd10ee01d7d" />

6. Synthesis

The RTL was synthesized using Yosys, targeting the sky130_fd_sc_hd typical library.

Item Value
Total cells 28
Flip-flops 3 (state register)
Combinational gates 25
Memories 0

<img width="1920" height="983" alt="seq_detector_gates" src="https://github.com/user-attachments/assets/d5fd6380-f66a-43f7-9619-63d633660b33" />

7. Gate-Level Simulation (GLS)

The synthesized netlist was simulated using the Sky130 cell models. A text-trace comparison showed that the RTL and GLS cycle-by-cycle outputs were identical.

8. Pre-Layout Timing Analysis

Timing was checked on the synthesized netlist using OpenSTA with an ideal clock.

Metric Value
WNS 0.00 ns
TNS 0.00 ns
din → dout path slack +4.908 ns (MET)

<img width="1920" height="983" alt="seq_detector_timingsummary" src="https://github.com/user-attachments/assets/1b42447d-d980-427b-948d-40442fd0927e" />

9. Physical Design

The physical design was performed using OpenLane v0.21 in its Docker environment with the Sky130A PDK.

Item Result
Flow status Completed without fatal errors
Die area 0.01 mm² (100 × 100 µm)
TritonRoute violations 0
Magic DRC violations 0
LVS errors 0 (Netgen log confirms 0 mismatches)

9.1 Placement

<img width="1920" height="983" alt="seq_detector_placement" src="https://github.com/user-attachments/assets/1e8a0f33-d7d2-45d2-8a43-b812a059b386" />

9.2 Routing

<img width="1920" height="983" alt="seq_detector_routing" src="https://github.com/user-attachments/assets/3639ed80-2899-466d-a841-7ac7e7cf7c5d" />

9.3 Final GDS Layout

<img width="1920" height="983" alt="seq_detector_finallayout" src="https://github.com/user-attachments/assets/ad35e727-5621-4ff4-a02d-68806560a78d" />

10. Post-Route Timing Analysis

After routing, OpenSTA was run across 21 PVT corners. WNS and TNS are 0.00 ns at all corners.

<img width="1920" height="983" alt="seq_detector_prelayout" src="https://github.com/user-attachments/assets/4d224c1b-3710-4c54-b345-9a83d0312d30" />

<img width="1920" height="983" alt="seq_detector_wns" src="https://github.com/user-attachments/assets/5b6f3b1d-e86e-4fad-9ab6-97f6d51847d3" />

<img width="1920" height="983" alt="seq_detector_tns" src="https://github.com/user-attachments/assets/d0ae9fbe-3540-4f4f-9dd7-c626bfa0cfec" />

11. Summary of Results

Stage Result
RTL simulation PASSED
Synthesis 28 cells, 3 flip-flops
Gate-level simulation Traces identical
Pre-layout timing WNS 0.00 ns, TNS 0.00 ns
Floorplan 100 µm × 100 µm die
Placement, CTS, routing Completed, zero routing violations
DRC / antenna 0 / 0
Post-route multi-corner timing No reported violations
Output GDSII file generated

Note on LVS: The OpenLane summary CSV reports "2 errors" due to a known bug in the Sky130 PDK where the KAPWR (keep-alive power) pin on the CTS clock buffers is not automatically connected by the flow script. The actual Netgen log confirms device count difference = 0 and unmatched pins = 0, proving the physical layout perfectly matches the schematic.

12. Limitations and Future Work

· The detector currently looks for a fixed sequence only.
· A 10 ns clock and 2 ns input/output delays were assumed. A faster clock was not tested.
· Future work could make the sequence programmable or integrate the detector directly into a full UART receiver.

13. Conclusion

A UART data framing sequence detector was designed in synthesizable Verilog and verified in RTL and gate-level simulation. The design was synthesized to 28 cells and implemented on a 100 µm × 100 µm die using OpenLane and the Sky130 standard-cell library. The physical-design flow completed with clean reported DRC and antenna checks. The flow generated the final GDSII layout.
