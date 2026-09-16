8-Bit LFSR Data Scrambler and Descrambler
This project implements an 8-bit data scrambler and descrambler in Verilog using a 16-bit Linear Feedback Shift Register (LFSR). The design randomizes serial data for digital communication systems and restores it at the receiver.

Features

8-bit input and output data paths

16-bit LFSR with an initial seed of 16'hFFFF

Synchronized scrambling and descrambling behavior

sync_in control for LFSR re-synchronization

bypass control for transparent data transfer

Reset support with deterministic initial conditions

Suitable for simulation and RTL verification

Modules

data_scrambler

Scrambles the input data stream using the current LFSR state.

Signal

Direction

Description

clk

Input

Clock

rst

Input

Reset; initializes the LFSR and output

data_in[7:0]

Input

Parallel input data

sync_in

Input

Reloads the LFSR seed

bypass

Input

Passes input data directly to the output

data_out[7:0]

Output

Scrambled or bypassed data

data_descrambler

Uses the same LFSR sequence to recover the original data from scrambled_data_in.

Signal

Direction

Description

clk

Input

Clock

rst

Input

Reset; initializes the LFSR and output

scrambled_data_in[7:0]

Input

Scrambled input data

sync_in

Input

Reloads the LFSR seed

bypass

Input

Passes input data directly to the output

data_out[7:0]

Output

Descrambled or bypassed data

Operation Priority

At each active clock edge, controls are evaluated in this order:

rst: reset the LFSR and output.

bypass: forward input data without scrambling.

sync_in: reload the LFSR seed.

Otherwise: update the LFSR and process one input bit.

LFSR Feedback

The design uses a 16-bit LFSR register named x. Its initial state is:

x = 16'hFFFF

On every active clock edge during normal operation, the current state is used to generate one scrambling bit. The feedback process works as follows:

Save the current most-significant bit: old_x_msb = x[15].

Perform a circular left shift by moving x[15] to bit 0:

next_x = {x[14:0], x[15]}

Apply XOR feedback using the old LFSR state:

next_x[3] = x[2] ^ x[15]
next_x[4] = x[3] ^ x[15]
next_x[5] = x[4] ^ x[15]

Store the calculated state back into the LFSR: x <= next_x.

The current LFSR MSB acts as the pseudo-random sequence bit. In the scrambler, it is XORed with the incoming data bit:

scrambled_bit = data_in[0] ^ x[15]

In the descrambler, the same LFSR sequence is regenerated and removed using XOR:

recovered_bit = scrambled_data_in[0] ^ x[15]

Because XOR is self-inverting, applying the same sequence twice restores the original bit:

(data_bit ^ lfsr_bit) ^ lfsr_bit = data_bit

The generated bit is inserted into the MSB of data_out, while the previous output bits shift right:

data_out <= {generated_bit, data_out[7:1]}

Therefore, the implementation processes one input bit per clock, specifically bit 0 of the input bus. The external logic or testbench must provide the next serial input bit on data_in[0] (or scrambled_data_in[0]) for each clock cycle.

When sync_in is asserted, the LFSR is reloaded with 16'hFFFF and no data scrambling/descrambling is performed in that clock cycle.

Simulation

The included testbench instantiates both modules in a loopback path:

original_data_in -> data_scrambler -> scrambled_data_out
                                      -> data_descrambler -> descrambled_data_out

The clock has a 10 ns period. After reset is released, the testbench samples the outputs every eight positive clock edges, allowing one complete 8-bit serial word to be processed. It also generates testbench.vcd and prints the input, output, and internal LFSR values during simulation.

Example using Icarus Verilog:

iverilog -g2012 -o simulation.vvp data_scrambler.v testbench.v
vvp simulation.vvp

To inspect the waveform with GTKWave:

gtkwave testbench.vcd

The current testbench exercises reset and normal loopback operation for nine 8-cycle observation windows. sync_in and bypass are initialized to 0 and remain unchanged; add dedicated stimulus to verify synchronization and bypass behavior.

Notes

The source and testbench use SystemVerilog-compatible constructs, including block-local declarations and an integer declared in the for loop; compile with SystemVerilog support, such as Icarus Verilog's -g2012 option.

Although the comments describe rst as synchronous, the sensitivity list includes posedge rst; therefore, the implemented reset behavior is asynchronous assertion with clocked operation otherwise. Update the sensitivity list if a strictly synchronous reset is required.
