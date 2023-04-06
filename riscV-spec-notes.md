# RISC-V specification reading notes(编辑中)

这篇文档算是我阅读RISC-V的spec时写的一份outline，主要把一些从验证人员视角中比较重要的一些概念摘出来了。其实体系结构最不重要的就是指令本身，所以这里主要都是一些概念。

目前的进度是unpriv spec差浮点，priv spec刚开始，下周会补完。

Document Version 20191213

- [RISC-V specification reading notes(编辑中)](#risc-v-specification-reading-notes编辑中)
  - [1 Basic concept](#1-basic-concept)
    - [1.1 Terminology](#11-terminology)
    - [1.2 ISA](#12-isa)
    - [1.3 Memory](#13-memory)
    - [1.4 Base Instruction-Length Encoding](#14-base-instruction-length-encoding)
    - [1.5 Privilege-support implementation stacks](#15-privilege-support-implementation-stacks)
    - [1.6 Privilege Level](#16-privilege-level)
  - [2. Programmer's model](#2-programmers-model)
    - [2.1 Base integer ISA RV32I](#21-base-integer-isa-rv32i)
    - [2.2 Base integer ISA RV32E](#22-base-integer-isa-rv32e)
    - [2.3 Base integer Instruction Set RV64I](#23-base-integer-instruction-set-rv64i)
    - [2.4 Base integer Instruction Set RV128I](#24-base-integer-instruction-set-rv128i)
  - [3. RV32I: Base Integer Instruction Set](#3-rv32i-base-integer-instruction-set)
    - [3.1 Memory ordering instruction](#31-memory-ordering-instruction)
    - [3.2 HINT](#32-hint)
  - [4 Zifencei: Instruction-Fetch Fence](#4-zifencei-instruction-fetch-fence)
  - [5 RV64I](#5-rv64i)
  - [6 M Extension: Multiplication and Division](#6-m-extension-multiplication-and-division)
    - [6.1 Multiplication](#61-multiplication)
    - [6.2 Division](#62-division)
  - [7 A standard Extension for Atomic Instruction](#7-a-standard-extension-for-atomic-instruction)
    - [7.1 Load-Reserved/Store-Conditional](#71-load-reservedstore-conditional)
    - [7.2 LR/SC loop constraint](#72-lrsc-loop-constraint)
    - [7.3 AMO(Atomic Memory Operation)](#73-amoatomic-memory-operation)
  - [8 Zicsr: CSR instructions](#8-zicsr-csr-instructions)
    - [8.1 CSR: Control and Status Registers](#81-csr-control-and-status-registers)
    - [8.2 CSR Field Specification](#82-csr-field-specification)
    - [8.3 CSR Instructions](#83-csr-instructions)
    - [8.4 CSR Access Ordering](#84-csr-access-ordering)
  - [9. Counters](#9-counters)
  - [10. RISC-V Weak Memory Ordering(RVWMO)](#10-risc-v-weak-memory-orderingrvwmo)
    - [10.1 Syntactic Dependencies](#101-syntactic-dependencies)
    - [10.2 Preserved Program Order](#102-preserved-program-order)
    - [10.3 Why](#103-why)
    - [11 Machine-Level ISA](#11-machine-level-isa)
    - [11.1 CSR](#111-csr)


## 1 Basic concept

### 1.1 Terminology

<table>
  <tr>
    <td><b>Core</b></td>
    <td>A component which contains an independent instruction fetch unit</td>
    <td>A RISC-V-compatible core might support multiple RISC-V-compatible hardware threads(<b>harts</b>), through multithreading</td>
  </tr>
  <tr>
    <td><b>Coprocessor</b></td>
    <td>a unit that is attached to a RISC-V core and is mostly sequenced by a RISC-V instruction stream</td>
    <td>but it contains additional architectural state and instruction-set extensions, and possibly some limited autonomy relative to the primary RISC-V instruction stream</td>
  </tr>
    <tr>
    <td><b>Accelerator</b></td>
    <td>Refer to either a non-programmable fixed-function unit or a core that can operate autonomously but is specialized for certain tasks</td>
    <td>Example: I/O accelerator</td>
  </tr>
      <tr>
    <td><b>Hart</b></td>
    <td>From the perspective of software running in a given execution environment, a hart is a resource that autonomously fetches and executes RISC-V instructions within that execution environment.</td>
    <td>The important distinction between a hardware thread (hart) and a software thread context is that the software running inside an execution environment is not responsible for causing progress of each of its harts; that is the responsibility of the outer execution environment.</td>
  </tr>
</table> 

<table>
  <tr>
    <td><b>Exception</b></td>
    <td>An unusual condition occurring at run time associated with
an instruction in the current RISC-V hart</td>
  </tr>
  <tr>
    <td><b>Interrupt</b></td>
    <td>An external asynchronous event that may cause a RISC-V hart to experience an unexpected transfer of control.</td>
  </tr>
    <tr>
    <td><b>Trap</b></td>
    <td>The transfer of control to a trap handler caused by either an exception or an interrupt</td>
  </tr>
</table>

<table>
  <tr>
    <td><b>Contained Trap</b></td>
    <td>The trap is visible to, and handled by, software running inside the execution environment</td>
    <td>For example, in an EEI providing both supervisor and user mode on harts, an ECALL by a user-mode hart will generally result in a transfer of control to a supervisor</td>
  </tr>
  <tr>
    <td><b>Requested Trap</b></td>
    <td>The trap is a synchronous exception that is an explicit call to the execution environment requesting an action on behalf of software inside the execution environment.</td>
    <td>An example is a system call. In this case, execution may or may not resume on the hart after the requested action is taken by the execution environment. For example, a system call could remove the hart or cause an orderly termination of the entire execution environment</td>
  </tr>
    <tr>
    <td><b>Invisible Trap</b></td>
    <td>The trap is handled transparently by the execution environment and execution resumes normally after the trap is handled.</td>
    <td>Page fault</td>
  </tr>
  </tr>
    <tr>
    <td><b>Fatal Trap</b></td>
    <td>The trap represents a fatal failure and causes the execution environment to terminate execution</td>
    <td>Examples include failing a virtual-memory page-protection check or allowing a watchdog timer to expire. Each EEI should define how execution is terminated and reported to an external environment.</td>
  </tr>
</table>

**Execution Environment Interface(EEI)** defines:
- initial state of the program
- the number and type of harts in the environment including the privilege modes supported by the harts
- the accessibility and attributes of memory and I/O regions
- the behavior of all legal instructions executed on each hart (i.e., the ISA is one component of the EEI)
- the handling of any interrupts or exceptions raised during execution including environment calls

### 1.2 ISA

The four base ISAs in RISC-V are treated as distinct base ISAs:
- RV32I
- RV32E(subset variant of RV32I for small microcontrollers)
- RV64I
- RV128I(future)

Pros:
- each base ISA can be optimised for its needs without requiring to support all the operations needed for other base ISAs.

Cons:
- it complicates
the hardware needed to emulate one base ISA on another. 

The RISC-V privileged architecture provides fields in __misa__ to control the unprivileged ISA at
each level to support emulating different base ISAs on the same hardware.

we divide each RISC-V instruction-set encoding space (and related encoding spaces such as the CSRs) into three disjoint categories:

<table>
  <tr>
    <td><b>Standard</b></td>
    <td>Defined by foundation.<br/> Shall not conflict with other standard extensions for the same base ISA</td>
  </tr>
  <tr>
    <td><b>Reserved</b></td>
    <td>Currently not defined but are saved for future standard extensions</td>
  </tr>
    <tr>
    <td><b>Custom</b></td>
    <td>Shall never be used for standard extensions and are made available for vendor-specific non-standard extensiond</td>
  </tr>
</table> 

**Non-conforming**: to describe a non-standard extension that uses either a standard or a reserved encoding (i.e., custom extensions are not non-conforming).

### 1.3 Memory

A RISC-V hart has a single _byte-addressable_ address space of $2^{XLEN}$ bytes for all memory accesses.

<table>
  <tr>
    <td><b>word</b></td>
    <td>32 bits</td>
  </tr>
  <tr>
    <td><b>doubleword</b></td>
    <td>64 bits</td>
  </tr>
    <tr>
    <td><b>quadword</b></td>
    <td>128 bits</td>
  </tr>
</table> 

The execution environment determines the mapping of hardware resources into a hart’s address
space.

Different address ranges of a hart’s address space may:
- be vacant
- contain main memory
- contain one or more I/O devices.

When a RISC-V platform has multiple harts, the address spaces of any two harts may
- be entirely the same
- be entirely different
- be partly different but sharing some subset of resources,
mapped into the same or different address ranges.

Memory access:
<table>
  <tr>
    <td><b>Implicit</b> access</td>
    <td>Done to obtain the encoded instruction to execute.<br/>Implementations sometimes perform implicit reads of CSRs.</td>
  </tr>
  <tr>
    <td><b>Explicit</b> access</td>
    <td>Specific load and store instructions</td>
</table> 

Default memory consistency model: **RISC-V Weak Memory Ordering**(RVWMO).

Optional: Total Store Ordering.

### 1.4 Base Instruction-Length Encoding

The base RISC-V ISA has fixed-length 32-bit instructions that must be naturally aligned on 32-bit
boundaries.

ISA extensions with variable-length instructions: each instruction can be any number of 16-bit instruction
parcels in length and parcels are naturally aligned on 16-bit boundaries

<table>
  <tr>
    <td><b>IALIGN</b></td>
    <td>The instruction-address alignment constraint the implementation enforces</td>
  </tr>
    <tr>
    <td><b>ILEN</b></td>
    <td>The maximum instruction length supported by an implementation.<br/>It is always a multiple of IALIGN.</td>
  </tr>
</table>

IALIGN is 32 bits in the base ISA, but some ISA extensions, including
the compressed ISA extension, relax IALIGN to 16 bits.

The expanded encoding allows the instruction cache of standard implementation to only hold the most-significant 30 bits(6.25% saving). More information can be found in Unpriv Spec Section 1.5.

IALIGN may not take on any value other than 16 or 32.

**illegal instructions**:
- Encodings with bits [15:0] all zeros: These instructions are considered to be of minimal length.<br/>Software can rely on a naturally aligned 32-bit word containing zero to act as an illegal
instruction on all RISC-V implementations, to be used by software where an illegal instruction
is explicitly desired.
- The encoding with bits [ILEN-1:0] all ones is also illegal: This instruction is considered to be ILEN bits long. <br/> Software can rely on a naturally aligned 32-bit word containing zero to act as an illegal instruction on all RISC-V implementations, to be used by software where an illegal instruction is explicitly desired.

RISC-V base ISAs have either _little-endian_ or _big-endian_ memory systems, with the privileged architecture further defining bi-endian operation.
> Note: Instructions are stored in memory as a sequence
of 16-bit little-endian parcels, regardless of memory system endianness.
> We have to fix the order in which instruction parcels are stored in memory, independent of memory system endianness, to ensure that the length-encoding bits always appear first in halfword address order. This allows the length of a variable-length instruction to be quickly determined by an instruction-fetch unit by examining only the first few bits of the first 16-bit instruction parcel.

### 1.5 Privilege-support implementation stacks

<table>
  <tr>
    <td><b>AEE/SEE/HEE</b></td>
    <td>Application/Supervisor/Hypervisor execution environment</td>
  </tr>
  <tr>
    <td><b>ABI</b></td>
    <td>Application binary interface.<br/>ABI includes the supported user-level ISA plus a set of ABI calls to interact with the AEE. The ABI hides details of the AEE from the application to allow greater flexibility in implementing the AEE.</td>
  </tr>
  <tr>
    <td><b>SBI</b></td>
    <td>Supervisor binary interface.<br/>An SBI comprises the user-level and supervisor-level ISA together with a set of SBI function calls. Using a single SBI across all SEE implementations allows a single OS binary image to run on any SEE. The SEE can be a simple boot loader and BIOS-style IO system in a low-end hardware platform, or a hypervisor-provided virtual machine in a high-end server, or a thin translation layer over a host operating system in an architecture simulation environment.</td>
  </tr>
  <tr>
    <td><b>HBI</b></td>
    <td>Hypervisor binary interface.</td>
  </tr>
  <tr>
    <td><b>OS</b></td>
    <td>Operating System</td>
  </tr>
</table> 

Simplest:

<table border="1">
  <tr>
    <td>Application</td>
  </tr>
  <tr>
    <td><b>ABI</b></td>
  </tr>
  <tr>
    <td>AEE</td>
  </tr>
</table> 

Conventional OS:

<table border="1">
  <tr>
    <td>Application</td>
    <td>Application</td>
  </tr>
  <tr>
    <td><b>ABI</b></td>
    <td><b>ABI</b></td>
  </tr>
  <tr>
    <td colspan="2" align="center">OS</td>
  </tr>
  <tr>
    <td colspan="2" align="center"><b>SBI</b></td>
  </tr>
  <tr>
    <td colspan="2" align="center">SEE</td>
  </tr>
</table> 

Virtual machine monitor configuration:

<table border="1">
  <tr>
    <td>Application</td>
    <td>Application</td>
    <td>Application</td>
    <td>Application</td>
  </tr>
  <tr>
    <td><b>ABI</b></td>
    <td><b>ABI</b></td>
    <td><b>ABI</b></td>
    <td><b>ABI</b></td>
  </tr>
  <tr>
    <td colspan="2" align="center">OS</td>
    <td colspan="2" align="center">OS</td>
  </tr>
  <tr>
    <td colspan="2" align="center"><b>SBI</b></td>
    <td colspan="2" align="center"><b>SBI</b></td>
  </tr>
  <tr>
    <td colspan="4" align="center">Hypervisor</td>
  </tr>
  <tr>
    <td colspan="4" align="center"><b>HBI</b></td>
  </tr>
  <tr>
    <td colspan="4" align="center">HEE</td>
  </tr>
</table> 

### 1.6 Privilege Level

<table>
  <tr>
    <th>Level</th>
    <th>Encoding</th>
    <th>Name</th>
    <th>Abbreviation</th>
  </tr>
  <tr>
    <td>0</td>
    <td>00</td>
    <td>User/Application</td>
    <td>U</td>
  </tr>
  <tr>
    <td>1</td>
    <td>01</td>
    <td>Supervisor</td>
    <td>S</td>
  </tr>
  <tr>
    <td>2</td>
    <td>10</td>
    <td>Reserved</td>
    <td></td>
  </tr>
  <tr>
    <td>3</td>
    <td>11</td>
    <td>Machine</td>
    <td>M</td>
  </tr>
</table>

**Debug mode** (D-mode) can be considered an additional privilege mode, with even more access than M-mode.


## 2. Programmer's model

### 2.1 Base integer ISA RV32I

<table>
  <tr>
    <td><b>XLEN</b></td>
    <td>The number of X register(general purpose registers)<br/>As well as the length of X registers.</td>
  </tr>
</table>

i.e. XLEN = 32, x0-x31

> Note: There is no dedicated stack pointer or subroutine return address link register in the Base Integer ISA; the instruction encoding allows any x register to be used for these purposes.

<table>
  <tr>
    <th colspan="2" align="center">Special register</th>
  </tr>
  <tr>
    <td><b>x0</b></td>
    <td>hardwired with all bits equal to 0</td>
  </tr>
  <tr>
    <td><b>x1</b></td>
    <td>In convention, used to hold return address for a call(LR)</td>
  </tr>
    <tr>
    <td><b>x2</b></td>
    <td>In convention, used as stack pointer(SP)</td>
  </tr>
    <tr>
    <td><b>x5</b></td>
    <td>In convention, an alternate link register.</td>
  </tr>
  <tr>
    <td><b>pc</b></td>
    <td>program counter</td>
  </tr>
</table>

### 2.2 Base integer ISA RV32E

The number of general registers is reduced to 16(x0-x15).

### 2.3 Base integer Instruction Set RV64I

Extend the integer registers and user addr space to 64 bits. (XLEN=64)

### 2.4 Base integer Instruction Set RV128I

Extend the integer registers and user addr space to 128 bits. (XLEN=128).

Suffix:

- W: operate on 32-bit values in the low bits of a register are retained but now sign extend their results from
bit 31 to bit 127
- D: operate on 64-bit values in the low bits of a register are retained but now sign extend their results from
bit 63 to bit 127

## 3. RV32I: Base Integer Instruction Set

Exception:

<table>
  <tr>
    <td><b>instruction-address-misaligned</b></td>
    <td>Generated on a taken branch or unconditional jump if the target address is not four-byte aligned.<br/>No instruction-address-misaligned exception is generated for a conditional branch that is not taken.</td>
    <td>The alignment constraint for base ISA instructions is relaxed to a two-byte boundary when instruction extensions with 16-bit lengths or other odd multiples of 16-bit lengths are added (i.e., IALIGN=16).</td>
  </tr>
</table>

> No integer computational instructions cause arithmetic exceptions


Instructions:

- I
- S
- B
- U
- J:Provide a Return Address predication Stack(RAS) implicit hinting optimisation by the registers that are used. A JAL instruction should push the return address onto a RAS only when rd=x1 or x5. JALR instructions should push/pop a RAS as shown in the Table:

<table>
  <tr>
    <th>rd</th>
    <th>rs1</th>
    <th>rs1==rd</th>
    <th>RAS</th>
    <th>Stimulus Function</th>
  </tr>
  <tr>
    <td>not link</td>
    <td>not link</td>
    <td></td>
    <td>None</td>
    <td>Snippet branch</td>
    </tr>
  <tr>
    <td>not link</td>
    <td>link</td>
    <td></td>
    <td>POP to rd</td>
    <td>Function return</td>
    </tr>
      <tr>
    <td>link</td>
    <td>not link</td>
    <td></td>
    <td>Push return addr to stack</td>
    <td>Function call</td>
    </tr>
      <tr>
    <td>link</td>
    <td>link</td>
    <td>False</td>
    <td>Pop to rd, and Push return addr</td>
    <td>Yield(not entirely sure)</td>
    </tr>
      <tr>
    <td>link</td>
    <td>link</td>
    <td>True</td>
    <td>Push return addr to stack</td>
    <td>Function call, use the LR to store target addr</td>
    </tr>
</table>

### 3.1 Memory ordering instruction

**FENCE**: used to order device I/O and memory accesses as viewed by other RISC-V harts and external devices or coprocessors. Informally, no other RISC-V hart or external device can observe any operation in the successor set following a FENCE before any operation in the predecessor set preceding the FENCE.

### 3.2 HINT

like the NOP instruction, HINTs do not change any architecturally
visible state, except for advancing the pc and any applicable performance counters. Implementations are always allowed to ignore the encoded hints


## 4 Zifencei: Instruction-Fetch Fence

**FENCE.I** instruction that provides explicit synchronization between writes to instruction memory and instruction fetches on the samev hart.

RISC-V does not guarantee that stores to instruction memory will be made visible to instruction fetches on a RISC-V hart until that hart executes a FENCE.I instruction.

FENCE.I does not ensure that other RISC-V harts’ instruction fetches will observe the local hart’s stores in a multiprocessor system. To make a store to instruction memory visible to all RISC-V harts, the writing hart has to execute a data FENCE before requesting that all remote RISC-V harts execute a FENCE.I.

## 5 RV64I

Instructions without any suffix operate on XLEN-bit values. Here XLEN=64.

Instructions with 'W' suffix ignore the upper 32 bits of their inputs and always produce 32-bit signed values, i.e. bits XLEN-1 through 31 are equal.

## 6 M Extension: Multiplication and Division

### 6.1 Multiplication

MUL: performs an XLEN-bit×XLEN-bit multiplication, places the _lower_ XLEN bits in the destination register.
MULH: performs an XLEN-bit×XLEN-bit multiplication, places the _upper_ XLEN bits in the destination register.

Microarchitectures can fuse these into a single multiply operation instead of performing two separate multiplies.

### 6.2 Division

DIV[U]: perform an XLEN bits by XLEN bits signed and unsigned integer division of rs1 by rs2, rounding towards zero. 

REM[U]: provide the remainder of the corresponding division
operation. For REM, the sign of the result equals the sign of the dividend.

For both signed and unsigned division, it holds that dividend = divisor × quotient + remainder.

Microarchitecture also can support fusion.

> Divide-by-zero exception does not exist in RISC-V now because this would be the only arithmetic trap in the standard ISA.
> 
> Current solution is a single branch instruction needs to be added to
each divide operation if you want to examine, and this branch instruction can be inserted after the divide and should normally be very predictably not taken, adding little runtime overhead.

## 7 A standard Extension for Atomic Instruction

### 7.1 Load-Reserved/Store-Conditional
Load-Reserved/Store-Conditional: load a value and register the addr set as _reserved_ . Store the value only if the reservation is still valid. _Used for creating lock-free data structure._

Each atomic instruction has two bits specifying additional memory ordering constraints:

<table>
  <tr>
    <td><b>aq</b></td>
    <td>acquire access<br/>no following memory operations on this RISC-V hart can be observed to take place before the acquire memory operation</td>
  </tr>
    <tr>
    <td><b>rl</b></td>
    <td>release access.<br/>the release memory operation cannot be observed to take place before any earlier memory operations on this RISC-V hart</td>
  </tr>
</table>

Exception:

<table>
  <tr>
    <td><b>instruction-address-misaligned</b></td>
    <td>LR-SC requires naturally-aligned.</td>
  </tr>
</table>

### 7.2 LR/SC loop constraint

1. The loop comprises only an LR/SC sequence and code to retry the sequence in the case of failure, and must comprise at most 16 instructions placed sequentially in memory(within 64 contiguous instruction bytes).
2. An LR/SC sequence begins with an LR instruction and ends with an SC instruction. The dynamic code executed between the LR and SC instructions can only contain instructions from the base “I” instruction set, excluding loads, stores, backward jumps, taken backward branches, JALR, FENCE, FENCE.I, and SYSTEM instructions. If the “C” extension is supported, then compressed forms of the aforementioned “I” instructions are also permitted.
3. The code to retry a failing LR/SC sequence can contain backwards jumps and/or branches to repeat the LR/SC sequence, but otherwise has the same constraint as the code between the LR and SC.
4. The LR and SC addresses must lie within a memory region with the LR/SC eventuality property. The execution environment is responsible for communicating which regions have this property.
5. The SC must be to the same effective address and of the same data size as the latest LR executed by the same hart.

LR/SC sequences that do not lie within constrained LR/SC loops are **unconstrained**.

### 7.3 AMO(Atomic Memory Operation)

AMO instructions perform read-modify-write operations for Multiprocessor synchronization.

Used to implement parallel reduction operations. i.e. semaphore

Amo instrs also have aq and rl bits.

<table>
  <tr>
    <th>Example AMO instruction</th>
    <th>Behaviour</th>
  </tr>
  <tr>
    <td>amoswap.w rd,rs2,(rs1)</td>
    <td>Atomically load a 32-bit signed data value from the address in rs1, place the value into register rd, swap the loaded value and the original 32-bit signed value in rs2, then store the result back to the address in rs1.</td>
    </tr>
</table>

Exception:

<table>
  <tr>
    <td><b>instruction-address-misaligned</b></td>
    <td>Amo instructions require that the address held in rs1 be naturally aligned to the size of the operand</td>
  </tr>
</table>

## 8 Zicsr: CSR instructions

### 8.1 CSR: Control and Status Registers

Encoding: 12-bit encoding space for up to 4096(=1<<12) CSRs.

By convention, the upper 4 bits of the CSR address (csr[11:8]) are used to encode the read and write accessibility of the CSRs according to privilege level.

<table>
  <tr>
    <th>Value in top 2 bits(csr[11:10])</th>
    <th>accessibility</th>
  </tr>
  <tr>
    <td>00</td>
    <td>Depend on Spec</td>
    </tr>
      <tr>
    <td>01</td>
    <td>Depend on Spec</td>
    </tr>
      <tr>
    <td>10</td>
    <td>Depend on Spec</td>
    </tr>
      <tr>
    <td>11</td>
    <td>Read-only</td>
    </tr>
</table>

<table>
  <tr>
    <th>Value in next 2 bits(csr[9:8])</th>
    <th>Lowest Privilege level</th>
  </tr>
  <tr>
    <td>00</td>
    <td>Unprivileged and User Level</td>
    </tr>
      <tr>
    <td>01</td>
    <td>Supervisor Level</td>
    </tr>
      <tr>
    <td>10</td>
    <td>Hypervisor and VS</td>
    </tr>
      <tr>
    <td>11</td>
    <td>Machine Level</td>
    </tr>
</table>

Machine-mode standard read-write CSRs 0x7A0–0x7BF are reserved for use by the debug system. Of these CSRs, 0x7A0–0x7AF are accessible to machine mode, whereas 0x7B0–0x7BF are only visible to debug mode.

### 8.2 CSR Field Specification

<table>
  <tr>
    <td><b>WPRI</b></td>
    <td>Reserved Writes Preserve Values, Reads Ignore Values</td>
    <td>Reserved field. Software should ignore the values read from these fields, and should preserve the values held in these fields when writing values to other fields of the same register.</td>
  </tr>
    <tr>
    <td><b>WLRL</b></td>
    <td>Write/Read Only Legal Values</td>
    <td>Software should not write anything other than legal values to such a field, and should not assume a read will return a legal value unless the last write was of a legal value, or the register has not been written since another operation (e.g., reset) set the register to a legal value<br/><br/>Implementations are permitted but not required to raise an <b>illegal instruction exception</b> if an instruction attempts to write a non-supported value to a WLRL field</td>
  </tr>
    <tr>
    <td><b>WARL</b></td>
    <td>Write Any Values, Reads Legal Values</td>
    <td>Similar to WLRL, but allow any value to be written while guaranteeing to return a legal value whenever read.<br/><br/>Implementations can return any legal value on the read of a WARL field when the last write was of an illegal value, but the legal value returned should deterministically depend on the illegal written value and the architectural state of the hart.</td>
  </tr>
</table>

> Modulation: If a write to one CSR changes the set of legal values allowed for a field of a second CSR, then unless specified otherwise, the second CSR’s field immediately gets an <b>unspecified</b> value from among its new legal values. This is true even if the field’s value before the write remains legal after the write; the value of the field may be changed in consequence of the write to the controlling CSR.

> Width Modulation: Zero extend if becoming wider. More info can be found in Spec.

> <b>Implicit read</b>: Unless otherwise specified, the value returned by an implicit read of a CSR is the same value that would have been returned by an explicit read of the CSR, using a CSR-access instruction in a sufficient privilege mode.

### 8.3 CSR Instructions

All CSR instructions **atomically read-modify-write** a single CSR, whose CSR specifier is encoded in the 12-bit csr field of the instruction held in bits 31–20. (4096 = 0x1000)

**Exception**:

<table>
  <tr>
    <td><b>illegal instruction exception</b></td>
    <td>1. Attempts to access a non-existent CSR<br/>2. Attempts to access a CSR without appropriate privilege level or to write a read-only register<br/>3. Implementations should raise illegal instruction exceptions on machine-mode access to Machine-mode standard read-write CSRs 0x7B0–0x7BF.(Used by debug system. Invisible to Machine mode)</td>
  </tr>
</table>

Some CSRs, such as the instructions-retired counter, instret, may be modified as side effects of instruction execution. In these cases, if a CSR access instruction reads a CSR, it reads the value prior to the execution of the instruction. If a CSR access instruction writes such a CSR, the write is done instead of the increment.

### 8.4 CSR Access Ordering

On a given hart, explicit and implicit CSR access are performed in program order with respect to those instructions whose execution behavior is affected by the state of the accessed CSR.

- a CSR access is performed after the execution of any prior instructions in program order whose behavior modifies or is modified by the CSR state and before the execution of any subsequent instructions in program order whose behavior modifies or is modified by the CSR state.
- a CSR read access instruction returns the accessed CSR state before the execution of the instruction
- a CSR write access instruction updates the accessed CSR state after the execution of the instruction.

## 9. Counters

RISC-V ISAs provide a set of up to 32×64-bit performance counters and timers that are accessible via unprivileged XLEN read-only CSR registers 0xC00–0xC1FF (with the upper 32 bits accessed via
CSR registers 0xC80–0xC9F on RV32).

First 3 counters:

<table>
  <tr>
    <td><b>CYCLE</b></td>
    <td>cycle count</td>
  </tr>
    <tr>
    <td><b>TIME</b></td>
    <td>real-time clock</td>
  </tr>
    <tr>
    <td><b>INSTRET</b></td>
    <td>instructions-retired</td>
  </tr>
</table>

> Note:
> 
> 1.Some execution environments might prohibit access to counters to impede timing side-channel attacks. (Annotation: In cryptography, a timing attack is a side-channel attack in which the attacker attempts to compromise a cryptosystem by analyzing the time taken to execute cryptographic algorithms.)

## 10. RISC-V Weak Memory Ordering(RVWMO)

> Definition: A memory consistency model is a set of rules specifying the values that can be returned by loads of memory. The RVWMO memory model is defined in terms of the global memory order, a total ordering of the memory operations produced by all harts.

Code written for weaker memory model is automatically and inherently compatible with stronger model(RVTSO), but code written assuming stronger model is not guarantted to run correctly on weaker implementation.

RVWMO ensures that code running on a single hart appears to execute in order from the perspective of other memory instructions in the same hart, but memory instructions from another hart may observe the memory instructions from the first hart being executed in a different order

### 10.1 Syntactic Dependencies

The meaning of syntactic dependencies is the same as it in compiler.

Syntactic dependencies are defined in terms of _instructions’ source registers_, _instructions’ destination registers_, and the way instructions _carry a dependency_ from their source registers to their destination registers.

### 10.2 Preserved Program Order

The subset of program order that must be respected by the global memory order is known as preserved program order.

- Overlapping-Address Ordering
- Explicit Synchronisation: FENCE, LR/SC
- Syntactic Dependencies
- Pipeline Dependencies

### 10.3 Why

RVWMO is between the two extremes of the memory model spectrum. The RVWMO memory model enables architects to build simple implementations, aggressive implementations, implementations embedded deeply inside a much larger system and subject to complex memory system interactions, or any number of other possibilities, all while simultaneously being strong enough to support programming language memory models at high performance

The Appendix A in unpriv spec is a very good article to understand the memory system in architecture level.


### 11 Machine-Level ISA

### 11.1 CSR

<table>
  <tr>
    <th>CSR</th>
    <th>Function</th>
    <th>Spec</th>
    <th>Explanation</th>
  </tr>
  <tr>
    <td><b>misa</b></td>
    <td>cycle count</td>
  </tr>
</table>


