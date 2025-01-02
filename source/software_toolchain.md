<!---
   Copyright (c) 2025 OpenHW Group

   Licensed under the Solderpad Hardware Licence, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

   https://solderpad.org/licenses/

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.

   SPDX-License-Identifier: Apache-2.0 WITH SHL-2.1
--->

# Software Toolchain
Having made the decision to utilize real-world programming tools to generate the machine code needed to verify our cores,
the next questions are which ones to use and where to get them.
Both `gcc` and `llvm` are popular choices for OpenHW projects (and probably for most other RISC-V projects as well).
Although it is a design goal of CORE-V-VERIF to support either of these compilers, in this document we will work with gcc.

```{admonition} Attribution
Parts of this chapter were adopted from:
- **_The RISC-V Reader_** by Patterson and Waterman.
- **_RISC-V Architecture and Design_** by Harris, Stine & Thompson
```

## Obtaining a Compiler for CORE-V Cores
Embecosm maintains a set of prebuilt RISC-V compilers including [riscv32-corev-elf-gcc](https://embecosm.com/downloads/tool-chain-downloads/#core-v-top-of-tree-compilers),
and [riscv64-unknown-linux-gnu-gcc](https://embecosm.com/downloads/tool-chain-downloads/#risc-v-linux-top-of-tree-compilers).
When working with CORE-V-VERIF you are strongly encouraged (but not required) to use one of these prebuilt compilers.
You may also choose to build your toolchain from source or use an in-house toolchain, in which case you will probably run `riscv64-unknown-elf-gcc`.
We use the term GCC as shorthand to mean any of these compilers.

## Notes about the Compiler
GCC is part of a "toolchain" of GNU software tools.
The ones we will be using the most are:
- GCC: the compiler that translates C and/or RISC-V assembler into machine code.
- OBJDUMP: generates a human readable view of the final object file.
- READELF: generates a human readable view of the ELF file.
- OBJCOPY: used to generate a "memory file" which can be loaded into a SystemVerilog memory during simulation.

GCC is both a C compiler and assembler.
It is invoked as a single command, but under the hood it performs four steps:
- **Preprocess**: insert #include files and substitute #define macros with their values.
- **Compile**: turn each C program into assembly language.
- **Assemble**: turn each assembly language program into an object file.
- **Link**: join all the object files together, relocate sections to desired addresses, resolve cross-references, produce an executable file.

GCC follows a GNU Target Triplet standard of `cpu-vendor-abi`.
For our purposes this triplet will be:
- **cpu** is riscv64 or riscv32, although GCC can supports any of the RV32 and/or RV64 flavors.
- **vendor** is normally _unknown_ because it is not platform specific.
The OpenHW prebuilt toolchains from Embecosm use _corev_ as the vendor.
- **abi** or application binary interface defines calling conventions plus the sizes and alignements of basic data types.
In this document we assume the ABI used is the [Extendable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format).

### Writing C test-programs
In order to support C programs, we will need a library that provides macros, type defintions
and functions for tasks such as string manipulation, mathematical computation, memory management, and input/output.
GCC uses a lightweight standard C library called [newlib](https://en.wikipedia.org/wiki/Newlib) for this purpose.
Later we will see how this is used to support C test-program development in CORE-V-VERIF, but for now we restrict ourselves to assembly.

## GCC Command Line
Recall the GCC command-line from the Makefile:
```
$ riscv64-unknown-elf-gcc -o example32.elf -march=rv32i -mabi=ilp32 -nostartfiles -mcmodel=medany -Tlink.ld eg.S
```
Based on the .S suffix, GCC determines that eg.S is an assembly language source file.
The command-line options used in this example are:
- `-o example32.elf` means the executable output file is named `example32.elf`.
- `-march` specifies the machine architecture as `rv32i`.
- `-mabi` specifies the ABI.
- `-mcmodel` specifies the code model.
- `-nostartfiles` prevents the assembler from inserting startup code, so only the actual code in `eg.S` is assembled.
- `-T` specifies the linker control script, named `link.ld` in our example.

The following expands on the `-march`, `-mabi`, and `-mcmodel` options.
Be aware that the definitive source material for all GCC options is [here](https://gcc.gnu.org/onlinedocs/gcc/RISC-V-Options.html).

### -march
The machine architecture passed to GCC on the command line defines the instruction set used in the compiled program.
RISC-V has many ISA flavors, so GCC needs to be told the machine architecture (ISA) to target.
For a machine architecture of rv32imac, the `mul` instruction is legal in assembly language,
while in rv32i, using `mul` produces an “Unrecognized opcode” compiler error.
Similarly, the C statements:
```
int a, b, c;
a = b * c;
```
will compile into `mul s0, s1, s2` in rv32imac,
while in rv32i, it will compile into a library call to a multiplication routine using a sequence of shifts and adds.

GCC can support longer architecture names containing other extensions separated by underscores, such as `-march rv64gc_zba_zbb_svpbmt`.
As detailed in [gcc RISC-V options](https://gcc.gnu.org/onlinedocs/gcc/RISC-V-Options.html), there are more than 100 supported extensions,
and as each CORE-V core supports a core-specific set of ISA extensions, sometimes the architecture names can get rather long indeed.
For example, the `-march` for the [CV32E40S](https://docs.openhwgroup.org/projects/cv32e40s-user-manual/en/latest/intro.html#standards-compliance) is `rv32im_zicsr_zifencei_zbb_zcb_zba_zbs_zcmp_zbc_zcmt`.

```{warning}
If `-march` is not specified, the machine architecture is system-dependent.
```

### -mabi
The ABI was briefly introduced in `Spike Programming Conventions`.
Recall that the RISC-V ABI supports ELF for executable and object files.
What follow is additional information that may be useful for developers of CORE-V-VERIF test-program writters.

An ABI defines conventions used by the compiler when building a program.
It is the standard that defines how programs communicate during function calls and system calls, including:
- The binary format of executables and object and library files.
- Which instruction set (ISA) is supported.
- The size, layout, and alignment of data types.
- The calling convention for how arguments are passed to functions in registers or on the stack.
- Operating system calling conventions.
- Code models of how instructions access a label in the program.

GCC defines a variety of 32- and 64-bit RISC-V ABI data type conventions summarized in the Table below.
The name of these data type ABIs use the following convention: i = integer, l = long, p = pointer.
So, `ilp32` means that integers, pointers, and longs are each 32 bits;
`lp64` means that longs and pointers are 64 bits (but ints are still 32 bits).
Data types are aligned to address boundaries that are a multiple of their size;
for example, address 0x3002 is legal for char and short but not for int or long.

| Data Type | ilp32 | ilp32f | ilp32d | lp64 | lp64d |
| --------- | ----- | ------ | ------ | ---- | ----- |
| int       | 32    | 32     | 32     | 32   | 32    |
| long      | 32    | 32     | 32     | 64   | 64    |
| pointers  | 32    | 32     | 32     | 64   | 64    |
| char      | 8     | 8      | 8      | 8    | 8     |
| short     | 16    | 16     | 16     | 16   | 16    |
| float     | n/a   | 32     | 32     | n/a  | 32    |
| double    | n/a   | n/a    | 64     | n/a  | 64    |

The Table below provides the recommended ABI for the embedded-class CORE-V cores curated by the OpenHW Foundation.

| CORE-V Core | Version | ABI    |
| ----------- | ------- | ------ |
| CV32E40P    | v1.0.0  | ilp32  |
| CV32E40P    | v1.8.3  | ilp32f |
| CV32E40S    | any     | ilp32  |
| CV32E40X    | any     | ilp32  |
| CV32E20     | any     | ilp32  |
| CV32A65X    | any     | ilp32  |

<br>
The ABI also defines conventions for GPR usage and calling conventions.
For example GPR x1 is given the ABI name `ra` and is used to store the return address from a function call.
Similarly, x2 is given the ABI name `sp` and is used as a stack pointer.

### Machine Architecture and ABI
There is a strong relationship between `march` and `mabi`.
Some ABIs are impossible to implement on some machine architectures.
For example, `-march=rv32if -mabi=ilp32d` is invalid because the ABI requires 64-bit values be passed in F registers,
but the machine architecture's F registers are only 32 bits wide.
The table below lists a variety of common machine architectures and their associated ABIs.

| Architecture | Synonym | ABI    |
| ------------ | ------- | ------ |
| rv32e        |         | ilp32e |
| rv32i        |         | ilp32  |
| rv32im       |         | ilp32  |
| rv32iac      |         | ilp32  |
| rv32imac     |         | ilp32  |
| rv32imafc    |         | ilp32f |
| rv32imafdc   | rv32gc  | ilp32d |
| rv64i        |         | lp64   |
| rv64im       |         | lp64   |
| rv64ic       |         | lp64   |
| rv64iac      |         | lp64   |
| rv64imac     |         | lp64   |
| rv64imafdc   | rv64gc  | lp64   |

The G (general) extension is shorthand for supporting the I (base integer),
M (multiply/divide), A (atomic), F (floating-point), and D (double) extensions.

### -mcmodel : Code Models
For RISC-V GCC defines three code models: medium-low (`medlow`), medium-any (`medany`) and large (`large`).
For the Embedded core environments, the Makefiles in CORE-V-VERIF generally do not specify the code model and thus the default `medlow` model is implicitly used.
Thus, the program and its statically defined symbols must lie within a single 2 GiB address range and must lie between absolute addresses −2 GiB and +2 GiB.
Programs can be statically or dynamically linked.
These restrictions are fine for cores with a 32-bit memory address space and no memory management such as the CV32E4 and CV32E2 cores.
Having said that, each project should determine its own code model requirements.

### Supervisor Binary Interface (SBI)
Not considered for the Embedded core environments in CORE-V-VERIF.

### Assembler Directives
ToDo

### RISC-V Assembler Directives
ToDo
