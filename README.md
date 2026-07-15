# SPI Slave with Single-Port RAM Controller

This project was completed under the supervision of **Eng. Kareem Waseem** in **April 2026**. It is a synthesizable Verilog implementation of an SPI (Serial Peripheral Interface) slave interface that drives a single-port RAM. The design decodes read/write commands received over SPI, shifts addresses and data in/out through an internal shift register, and returns read data back to the SPI master over MISO.

## Overview

The core of the project is the `SPI_slave` module, a Moore-style finite-state machine (FSM) with sequential output logic that acts as a bridge between an SPI Master and a single-port RAM. It recognizes and executes three primary commands coming from an SPI master:
- **Write Address/Data to RAM:** Latch an address or data word and forward it to the RAM.
- **Send Read Address to RAM:** Latch the address to be read and forward it to the RAM.
- **Receive Read Data from RAM:** Capture the RAM's output and shift it back out to the master via MISO.

A 10-bit shift register accumulates each incoming SPI word. The top 2 bits act as a mode-select field routed to the RAM, while the remaining 8 bits carry the address or data payload. A paired 4-bit counter tracks how many bits have been shifted in and flags the FSM once the register is full (10 bits).

## Command / Word Format

Each SPI transaction shifts in a 10-bit word, MSB-first, whose top 2 bits select the operation:

| Mode bits | Operation | Remaining 8 bits |
| :--- | :--- | :--- |
| `00` | Write Address | Target RAM address |
| `01` | Write Data | Data byte to store |
| `10` | Read Address | Address to read from |
| `11` | Read Data (clock-out) | Don't-care — clocks the response out on MISO |

A write consists of two back-to-back transactions (address, then data); a read consists of an address transaction followed by a data-clocking transaction during which the captured RAM byte is shifted out on MISO.

## FSM States

The controller utilizes a Gray-encoded FSM (`fsm_encoding = "gray"`) clocked on the rising edge of `clk` and reset on `rst` low.

| State | Encoding | Purpose |
| :--- | :--- | :--- |
| `IDLE` | `3'b000` | Standby; waits for `SS_n` to go low |
| `CHK_CMD` | `3'b001` | Reads the first bit on MOSI to determine which command is being issued |
| `WRITE` | `3'b010` | Shifts in a write address or write data word |
| `READ_ADD` | `3'b011` | Shifts in the address to read from RAM |
| `READ_DATA` | `3'b100` | Waits for `tx_valid` from RAM, then shifts the returned byte out over MISO |

## Port Descriptions

### Master-Slave Interface
| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `clk` | Input | 1 | System clock, posedge |
| `rst` | Input | 1 | Asynchronous reset, active low (negedge) |
| `SS_n` | Input | 1 | Slave Select control signal (Active low) |
| `MOSI` | Input | 1 | Master-Out, Slave-In serial data line |
| `MISO` | Output reg | 1 | Master-In, Slave-Out serial data line |

### Slave-RAM Interface
| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `rx_data` | Output | 10 | Data/Address word forwarded to RAM |
| `rx_valid` | Output reg | 1 | Pulses high when `rx_data` is valid for the RAM |
| `tx_data` | Input | 8 | Data byte returned from RAM to SPI Slave |
| `tx_valid` | Input | 1 | RAM asserts when `tx_data` is valid |

## Module Structure & Directory

| Module | File (suggested) | Description |
| :--- | :--- | :--- |
| `SPI_slave` | `Design/SPI_Slave.v` | FSM + shift register + bit counter; core SPI-to-RAM command decoder |
| `counter` | `Design/counter.v` | 4-bit counter that increments while `cnt_on` is asserted and flags bit 9 |
| `single_port_RAM` | `Design/single_port_RAM.v` | Single-port RAM backing store addressed/written by the SPI slave |
| `SPI_slave_RAM` | `Design/top_module.v` | Top-level wrapper connecting `SPI_slave` to the RAM module |
| `SPI_slave_RAM_tb` | `Testbench/top_module_tb.v` | Self-checking testbench exercising write-then-read operations |


## Tools & Verification

- **Verification:** Rudimentary functional verification was performed using **QuestaSim** (Simulation) and **QuestaLint** (Linting).
- **FPGA Implementation:** Synthesized and implemented for FPGA targets using Xilinx Vivado.
- **Testbench:** The included self-checking testbench drives a **write operation** task (shifting in a 10-bit address word followed by a 10-bit data word) and a **read operation** task (shifting in a read address, then clocking out the RAM's response and capturing it bit-by-bit from MISO). It compares the captured byte against an expected value and reports `Error!` via `$display`/`$stop` on mismatch.

To run the simulation in QuestaSim, a sample `.do` file flow is provided within the repository scripts:
```bash
vlib work
vlog +acc Design/counter.v Design/SPI_Slave.v Design/single_port_RAM.v Design/top_module.v Testbench/top_module_tb.v
vsim -voptargs=+acc work.SPI_slave_RAM_tb
run -all
```



