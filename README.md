# Single-Cycle RISC-V Processor (RV32I)  

A modular implementation of a single-cycle **RISC**-V processor based on the **RV32I** instruction set architecture, written in Verilog **HDL** and simulated using Vivado.  

The processor implements instruction fetch, instruction decode, execution, memory access, and register write-back within a single clock cycle. The design is organized into independent datapath and control modules to make the processor architecture easier to understand, simulate, and extend.  

## Architecture  

The processor follows a conventional single-cycle datapath:  

```text  
    ┌────────────────────┐  
    │  Program Counter   │  
    └─────────┬──────────┘  
    │  
    ▼  
    ┌────────────────────┐  
    │ Instruction Memory │  
    └─────────┬──────────┘  
    │  
    ▼  
    ┌────────────────────┐  
    │   Control Unit     │  
    └─────────┬──────────┘  
    │  
    ┌─────────▼──────────┐  
    │   Register File    │  
    └─────────┬──────────┘  
    │  
    ▼  
    ┌────────────────────┐  
    │        **ALU**         │  
    └─────────┬──────────┘  
    │  
    ▼  
    ┌────────────────────┐  
    │    Data Memory     │  
    └─────────┬──────────┘  
    │  
    ▼  
    ┌────────────────────┐  
    │    Write Back      │  
    └────────────────────┘  
```  

Branch target generation and multiplexers control the next PC and **ALU** operands based on the decoded instruction and control signals.  

## Implemented Modules  

### Program Counter  

Maintains the current 32-bit instruction address and updates it on every rising clock edge.  

- Synchronous PC update  
- Asynchronous reset  
- Reset value: `0`  

### Instruction Memory  

Implements a 64-entry instruction memory with 32-bit instructions.  

Instructions are selected using the word-aligned PC address:  

```verilog  
I_Mem[read_address[7:2]]  
```  

The memory is preloaded with instructions used to exercise different **RISC**-V instruction formats.  

### Register File  

Implements:  

- 32 general-purpose registers  
- 32-bit register width  
- Two combinational read ports  
- One synchronous write port  

The source registers are selected using `rs1` and `rs2`, while `rd` receives the write-back result when `RegWrite` is asserted.  

### Control Unit  

Decodes the instruction opcode and generates the control signals required by the datapath.  

Implemented control signals include:  

- `RegWrite`  
- `MemRead`  
- `MemWrite`  
- `MemToReg`  
- `ALUSrc`  
- `Branch`  
- `ALUOp`  

The control unit distinguishes between R-type, I-type, load, store, branch, jump, and U-type instructions.  

### Immediate Generator  

Extracts and sign-extends immediate values according to the instruction format.  

Supported formats:  

- I-type  
- S-type  
- B-type  
- U-type  
- J-type  

The implementation handles the non-contiguous immediate fields used by branch, store, and jump instructions.  

### ALU  

The **ALU** supports the following operations:  

| Operation | Description                 |  
| --------- | --------------------------- |  
| ADD       | Addition                    |  
| SUB       | Subtraction                 |  
| AND       | Bitwise AND                 |  
| OR        | Bitwise OR                  |  
| XOR       | Bitwise XOR                 |  
| SLL       | Logical left shift          |  
| SRL       | Logical right shift         |  
| SRA       | Arithmetic right shift      |  
| SLT       | Signed less-than comparison |  

The **ALU** also generates a `Zero` flag used by the branch datapath.  

### ALU Control  

Combines `ALUOp`, `funct3`, and `funct7` fields to determine the operation performed by the **ALU**.  

### Data Memory  

Implements a 64-entry, 32-bit data memory with:  

- Read enable  
- Write enable  
- Combinational read output  
- Clocked writes  

Load and store instructions use the **ALU** to calculate the memory address.  

### Branch Adder  

Computes the branch target using:  

```text  
Branch Target = PC + Immediate  
```  

The resulting target is selected by the PC multiplexer when the branch control condition is satisfied.  

### Multiplexers  

Two 2-to-1 multiplexers are used in the datapath:  

1. ****ALU** input multiplexer**  

   * Selects between register data and an immediate value.  

2. **Write-back multiplexer**  

   * Selects between the **ALU** result and data memory output.  

## Instruction Support  

The instruction memory contains test instructions covering multiple **RV32I** instruction formats.  

### R-Type  

Register-to-register arithmetic and logical operations:  

```text  
**ADD**  
**SUB**  
**AND**  
OR  
**XOR**  
**SLL**  
**SRL**  
**SRA**  
**SLT**  
```  

### I-Type  

Immediate arithmetic and logical operations:  

```text  
**ADDI**  
**ORI**  
**XORI**  
**ANDI**  
**SLLI**  
**SRLI**  
**SRAI**  
**SLTI**  
```  

### Load Instructions  

The instruction memory includes examples of:  

```text  
LB  
LH  
LW  
```  

### Store Instructions  

Examples include:  

```text  
SB  
SH  
SW  
```  

### Branch Instructions  

The implementation includes:  

```text  
**BEQ**  
**BNE**  
```  

with branch targets generated from the B-type immediate.  

### U-Type  

The design includes:  

```text  
**LUI**  
**AUIPC**  
```  

### J-Type  

The instruction memory includes:  

```text  
**JAL**  
```  

## Datapath Operation  

For each instruction, the processor performs the following operations within a single clock cycle:  

```text  
PC  
 ↓  
### Instruction Fetch  
 ↓  
### Instruction Decode  
 ↓  
Register Read / Immediate Generation  
 ↓  
**ALU** Execution  
 ↓  
### Memory Access  
 ↓  
### Write Back  
```  

For branch instructions, the immediate generator and branch adder calculate the target address, while the **ALU**'s zero flag contributes to branch selection.  

## Verification  

A dedicated Verilog testbench (`RISCV_ToP_Tb`) is used to simulate the processor.  

The testbench:  

- Generates the processor clock.  
- Applies an initial reset.  
- Runs the processor for multiple instruction cycles.  
- Generates a **VCD** waveform dump for signal-level debugging.  

Clock generation is implemented using:  

```verilog  
always #50 clk = ~clk;  
```  

The reset is initially asserted and then released after 50 simulation time units. The simulation continues for **5200** additional time units before `$finish` is called.  

Waveforms are captured using:  

```verilog  
$dumpfile(*waveform.vcd*);  
$dumpvars(0, **UUT**);  
```  

These waveforms can be used to inspect signals such as:  

- Program counter  
- Instruction  
- Register operands  
- Immediate values  
- **ALU** control  
- **ALU** result  
- Control signals  
- Branch signals  
- Memory accesses  
- Write-back data  

## Tools  

- ****HDL**:** Verilog  
- **Simulation / Verification:** Vivado  
- **Architecture:** **RISC**-V **RV32I**  
- **Target:** 32-bit processor  
- **Verification:** Verilog testbench + waveform analysis  

## Design Characteristics  

- Single-cycle datapath  
- Modular Verilog implementation  
- 32-bit data path  
- 32 general-purpose registers  
- Separate instruction and data memories  
- Combinational **ALU** and control logic  
- Branch target computation  
- Immediate generation for multiple **RISC**-V formats  
- Simulation-based verification  

## Future Extensions  

The current single-cycle architecture provides a foundation for more advanced processor implementations. Potential extensions include:  

- Pipeline the datapath into IF/ID/EX/**MEM**/WB stages  
- Add pipeline registers  
- Implement data forwarding  
- Add hazard detection  
- Expand instruction coverage  
- Add multiply/divide instructions  
- Implement caches  
- Add memory-mapped I/O  
- Improve verification with automated instruction-level test cases  
- Add assertions and broader waveform-based verification  

## Project Structure  

```text  
.  
├── RISCV_Top.v  
├── RISCV_ToP_Tb.v  
├── waveform.vcd  
└── **README**.md  
```  

## Learning Outcomes  

This project provided hands-on experience with:  

- **RISC**-V **ISA** and instruction encoding  
- **CPU** datapath design  
- Control-unit design  
- Verilog **RTL** development  
- Combinational and sequential logic  
- Register files and memories  
- **ALU** design  
- Branch and jump address generation  
- Hardware debugging through simulation waveforms  
- Modular digital system design  
# RISC-V-RV32I-Core-Verilog-RTL-Design
