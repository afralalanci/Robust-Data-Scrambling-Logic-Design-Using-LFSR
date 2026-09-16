# 8-Bit LFSR Data Scrambler and Descrambler

This project implements an **8-bit data scrambler and descrambler in Verilog** using a 16-bit Linear Feedback Shift Register (LFSR). The design randomizes serial data for digital communication systems and reconstructs the original data at the receiver.

## Features

* 8-bit input and output data paths
* 16-bit LFSR with an initial seed of `16'hFFFF`
* Synchronized scrambling and descrambling behavior
* `sync_in` control for LFSR re-synchronization
* `bypass` control for transparent data transfer
* Reset support with deterministic initial conditions
* Suitable for simulation and RTL verification
* Serial bit processing at one bit per clock cycle

## System Architecture

The design consists of two main RTL modules:

```text
Original Data
     |
     v
+------------------+
|  Data Scrambler  |
+------------------+
     |
     | Scrambled Data
     v
+--------------------+
| Data Descrambler   |
+--------------------+
     |
     v
Recovered Data
```

The scrambler and descrambler use identical LFSR sequences. Since XOR is self-inverting, applying the same pseudo-random sequence during descrambling recovers the original data.

---

## Modules

### `data_scrambler`

Scrambles the input data stream using the current LFSR state.

| Signal          | Direction | Description                              |
| --------------- | --------- | ---------------------------------------- |
| `clk`           | Input     | Clock                                    |
| `rst`           | Input     | Reset; initializes the LFSR and output   |
| `data_in[7:0]`  | Input     | Parallel input data                      |
| `sync_in`       | Input     | Reloads the LFSR seed                    |
| `bypass`        | Input     | Passes input data directly to the output |
| `data_out[7:0]` | Output    | Scrambled or bypassed data               |

### `data_descrambler`

Uses the same LFSR sequence to recover the original data from `scrambled_data_in`.

| Signal                   | Direction | Description                              |
| ------------------------ | --------- | ---------------------------------------- |
| `clk`                    | Input     | Clock                                    |
| `rst`                    | Input     | Reset; initializes the LFSR and output   |
| `scrambled_data_in[7:0]` | Input     | Scrambled input data                     |
| `sync_in`                | Input     | Reloads the LFSR seed                    |
| `bypass`                 | Input     | Passes input data directly to the output |
| `data_out[7:0]`          | Output    | Descrambled or bypassed data             |

---

## Operation Priority

At each active clock edge, the control signals are evaluated in the following order:

1. **`rst`** – Resets the LFSR and output.
2. **`bypass`** – Forwards the input data without scrambling or descrambling.
3. **`sync_in`** – Reloads the LFSR with the initial seed.
4. **Normal operation** – Updates the LFSR and processes one input bit.

---

## LFSR Feedback

The design uses a 16-bit LFSR register named `x`.

The initial LFSR state is:

```text
x = 16'hFFFF
```

During normal operation, the current LFSR state is used to generate the scrambling sequence.

The feedback process is:

### 1. Save the current MSB

```text
old_x_msb = x[15]
```

### 2. Perform a circular left shift

The current MSB is moved to bit 0:

```text
next_x = {x[14:0], x[15]}
```

### 3. Apply XOR feedback

The following feedback operations are performed using the previous LFSR state:

```text
next_x[3] = x[2] ^ x[15]
next_x[4] = x[3] ^ x[15]
next_x[5] = x[4] ^ x[15]
```

### 4. Update the LFSR

The calculated state is stored back into the LFSR:

```text
x <= next_x
```

The current LFSR MSB is used as the pseudo-random sequence bit.

---

## Scrambling Operation

For the scrambler, the generated LFSR bit is XORed with the incoming data bit:

```text
scrambled_bit = data_in[0] ^ x[15]
```

The generated bit is then inserted into the MSB of the output while the previous output bits shift right:

```text
data_out <= {generated_bit, data_out[7:1]}
```

---

## Descrambling Operation

The descrambler regenerates the same LFSR sequence and removes it using XOR:

```text
recovered_bit = scrambled_data_in[0] ^ x[15]
```

Because XOR is self-inverting:

```text
(data_bit ^ lfsr_bit) ^ lfsr_bit = data_bit
```

the original data is recovered at the output.

---

## Serial Data Processing

Although the interface uses an 8-bit data bus, the implementation processes **one bit per clock cycle**.

Specifically, bit `0` of the input bus is processed on each clock cycle:

```text
data_in[0]
```

or, for the descrambler:

```text
scrambled_data_in[0]
```

The generated bit is inserted into `data_out[7]`, while the previous output bits shift toward the LSB.

Therefore, external logic or the testbench must provide the next serial input bit on bit `0` for each clock cycle.

After eight clock cycles, one complete 8-bit serial word has been processed.

---

## Synchronization

When `sync_in` is asserted, the LFSR is reloaded with the initial seed:

```text
16'hFFFF
```

No scrambling or descrambling operation is performed during that clock cycle.

This allows the scrambler and descrambler to be synchronized to the same LFSR state before processing data.

---

## Bypass Mode

When `bypass` is asserted, the input data is transferred directly to the output without scrambling or descrambling.

This provides a transparent data path for testing or system configurations where scrambling is not required.

---

## Simulation

The included testbench instantiates both modules in a loopback configuration:

```text
original_data_in
        |
        v
+------------------+
|  data_scrambler  |
+------------------+
        |
        | scrambled_data_out
        v
+--------------------+
| data_descrambler   |
+--------------------+
        |
        v
descrambled_data_out
```

The simulation uses a **10 ns clock period**.

After reset is released, the testbench samples the outputs every eight positive clock edges, allowing one complete 8-bit serial word to be processed.

The testbench also generates:

```text
testbench.vcd
```

for waveform analysis and prints the input, output, and internal LFSR values during simulation.

### Compile and Run

Using Icarus Verilog:

```bash
iverilog -g2012 -o simulation.vvp data_scrambler.v testbench.v
vvp simulation.vvp
```

### View Waveforms

The generated VCD file can be opened using GTKWave:

```bash
gtkwave testbench.vcd
```

---

## Verification

The current testbench exercises:

* Reset behavior
* Normal scrambling operation
* Descrambling operation
* Loopback data recovery
* Multiple 8-cycle observation windows

The testbench currently runs **nine 8-cycle observation windows**.

The `sync_in` and `bypass` signals are initialized to `0` and remain unchanged during the current test. Dedicated test cases can be added to verify synchronization and bypass functionality.

---

## Reset Behavior

The source comments describe `rst` as a synchronous reset; however, the implemented sensitivity list includes `posedge rst`.

Therefore, the current implementation provides:

* Asynchronous reset assertion
* Clocked operation during normal operation

If a strictly synchronous reset is required, the reset should be removed from the sensitivity list and evaluated inside the clocked block.

---

## Tools

* **Verilog / SystemVerilog**
* **Icarus Verilog**
* **GTKWave**
* RTL Simulation
* Digital Design & Verification

## Key Concepts

This project demonstrates:

* Linear Feedback Shift Registers (LFSR)
* Data scrambling and descrambling
* XOR-based data processing
* Sequential RTL design
* Serial data processing
* Control signal prioritization
* RTL simulation
* Waveform analysis
* Testbench-based verification
