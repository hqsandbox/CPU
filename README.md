# CPU

A pipelined RISC-V CPU implementation using the Tomasulo algorithm for out-of-order execution, written in Verilog. This project supports both simulation and FPGA deployment.

## Architecture Overview

This CPU implements the classic **Tomasulo architecture** with the following key components:

### Core Components

| Module | File | Description |
|--------|------|-------------|
| **Reorder Buffer (RoB)** | [`RoB.v`](src/RoB.v) | Manages instruction commit order and speculative execution recovery |
| **Reservation Station (RS)** | [`RS.v`](src/RS.v) | Holds instructions waiting for operands, enables out-of-order execution |
| **Load/Store Buffer (LSB)** | [`LSB.v`](src/LSB.v) | Handles memory operations with proper ordering |
| **ALU** | [`ALU.v`](src/ALU.v) | Executes arithmetic and logical operations |
| **Instruction Fetcher** | [`Fetcher.v`](src/Fetcher.v) | Fetches instructions from memory/cache |
| **Decoder** | [`Decoder.v`](src/Decoder.v) | Decodes standard RISC-V instructions |
| **Compressed Decoder** | [`C_Decoder.v`](src/C_Decoder.v) | Decodes RISC-V compressed (16-bit) instructions |
| **Register File** | [`Reg.v`](src/Reg.v) | 32 general-purpose registers with dependency tracking |
| **Cache** | [`cache.v`](src/cache.v) | Memory interface with instruction/data caching |
| **Branch Predictor** | [`predictor.v`](src/predictor.v) | Static branch prediction |

### Architecture Diagram

```
                    ┌─────────────────────────────────────────────────────┐
                    │                    Reorder Buffer (RoB)             │
                    │         (Maintains program order for commit)        │
                    └─────────────┬───────────────────────┬───────────────┘
                                  │                       │
         ┌────────────────────────┼───────────────────────┼────────────────┐
         │                        ▼                       ▼                │
         │  ┌─────────────────────────────┐  ┌─────────────────────────┐  │
         │  │   Reservation Station (RS)  │  │  Load/Store Buffer (LSB) │  │
         │  │   (ALU instructions queue)  │  │  (Memory ops queue)      │  │
         │  └─────────────┬───────────────┘  └────────────┬────────────┘  │
         │                │                               │               │
         │                ▼                               ▼               │
         │  ┌─────────────────────────────┐  ┌─────────────────────────┐  │
         │  │           ALU               │  │         Cache           │  │
         │  └─────────────────────────────┘  └─────────────────────────┘  │
         │                                                                │
         │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
         │  │ Fetcher  │→ │ Decoder  │→ │ Register │→ │  Issue Logic  │  │
         │  └──────────┘  └──────────┘  └──────────┘  └───────────────┘  │
         │       ↑              ↑                                        │
         │       │        ┌──────────────┐                               │
         │       └────────│ C_Decoder    │ (Compressed Instructions)    │
         │                └──────────────┘                               │
         └───────────────────────────────────────────────────────────────┘
```

## Supported ISA

- **RISC-V 32-bit Integer Base (RV32I)**
  - R-Type: `ADD`, `SUB`, `SLL`, `SLT`, `SLTU`, `XOR`, `SRL`, `SRA`, `OR`, `AND`
  - I-Type: `ADDI`, `SLTI`, `SLTIU`, `XORI`, `ORI`, `ANDI`, `SLLI`, `SRLI`, `SRAI`
  - Load: `LB`, `LH`, `LW`, `LBU`, `LHU`
  - Store: `SB`, `SH`, `SW`
  - Branch: `BEQ`, `BNE`, `BLT`, `BGE`, `BLTU`, `BGEU`
  - Jump: `JAL`, `JALR`
  - Upper Immediate: `LUI`, `AUIPC`

- **RISC-V Compressed Extension (RV32C)**
  - Supports 16-bit compressed instructions for improved code density

## Configuration

Buffer sizes can be configured in [`config.v`](src/config.v):

```verilog
`define ROB_SIZE_WIDTH 3    // RoB size = 2^3 = 8 entries
`define RS_SIZE_WIDTH 3     // RS size = 2^3 = 8 entries
`define LSB_SIZE_WIDTH 3    // LSB size = 2^3 = 8 entries
```

## Memory Map

| Address Range | Description |
|--------------|-------------|
| `0x00000` - `0x1FFFF` | RAM (128KB) |
| `0x30000` | UART I/O (read: input byte, write: output byte) |
| `0x30004` | Clock counter (read) / Program termination (write) |

## Project Structure

```
CPU/
├── src/                    # Verilog source files
│   ├── cpu.v              # Top-level CPU module
│   ├── RoB.v              # Reorder Buffer
│   ├── RS.v               # Reservation Station
│   ├── LSB.v              # Load/Store Buffer
│   ├── ALU.v              # Arithmetic Logic Unit
│   ├── Fetcher.v          # Instruction Fetch Unit
│   ├── Decoder.v          # Instruction Decoder
│   ├── C_Decoder.v        # Compressed Instruction Decoder
│   ├── Reg.v              # Register File
│   ├── cache.v            # Memory Cache
│   ├── predictor.v        # Branch Predictor
│   ├── config.v           # Configuration parameters
│   └── common/            # Common modules (UART, FIFO, Block RAM)
├── sim/                    # Simulation testbench
├── testcase/              # Test programs
│   ├── sim/               # Simulation test cases
│   └── fpga/              # FPGA test cases
├── fpga/                  # FPGA-related files
├── Bitstream/             # FPGA bitstream files
├── script/                # Utility scripts
├── Makefile               # Build automation
└── Dockerfile             # Docker environment setup
```

## Building and Running

### Prerequisites

- **Simulation**: [Icarus Verilog](http://iverilog.icarus.com/) (`iverilog`)
- **FPGA**: Xilinx Vivado (for bitstream generation)
- **Testcase Compilation**: RISC-V GCC toolchain (`riscv64-elf-gcc`)

### Using Docker (Recommended)

Build and enter the development environment:

```bash
# Build Docker image
docker build -t myarch .

# Run container
docker run -it --rm --privileged -v $(pwd):/app -w /app myarch
```

### Simulation

```bash
# Build simulation
make build_sim

# Run specific test case
make run_sim name=<testcase_name>

# Examples:
make run_sim name=gcd
make run_sim name=qsort
make run_sim name=pi
```

### FPGA Deployment

```bash
# Build test case for FPGA
make build_fpga_test name=<testcase_name>

# Run on FPGA (after loading bitstream)
make run_fpga name=<testcase_name>

# Run all FPGA tests
make run_fpga_all
```

### Available Test Cases

| Test | Description |
|------|-------------|
| `gcd` | Greatest Common Divisor |
| `qsort` | Quick Sort |
| `pi` | Pi calculation |
| `hanoi` | Tower of Hanoi |
| `queens` | N-Queens problem |
| `bulgarian` | Bulgarian Solitaire |
| `tak` | Tak function |
| `expr` | Expression evaluation |
| `magic` | Magic square |
| `uartboom` | UART stress test |
| `testsleep` | Sleep functionality |

## Key Features

1. **Out-of-Order Execution**: Instructions are executed as soon as their operands are ready, not in program order
2. **Register Renaming**: Eliminates WAR and WAW hazards through ROB-based renaming
3. **Speculative Execution**: Branch prediction with rollback support via RoB
4. **Compressed Instruction Support**: Handles both 32-bit and 16-bit RISC-V instructions
5. **Memory Ordering**: LSB ensures correct memory operation ordering

## Implementation Details

### Tomasulo Algorithm Flow

1. **Issue**: Instructions are decoded and dispatched to RS/LSB with renamed registers
2. **Execute**: Instructions execute when all operands are available (from register file or CDB)
3. **Write Result**: Results broadcast on Common Data Bus (CDB) for dependent instructions
4. **Commit**: Instructions commit in order from RoB, updating architectural state

### Hazard Resolution

- **RAW (Read After Write)**: Resolved via operand forwarding from RoB/CDB
- **WAW (Write After Write)**: Resolved via register renaming (ROB tags)
- **WAR (Write After Read)**: Resolved via register renaming (ROB tags)
- **Control Hazards**: Branch prediction with flush on misprediction

