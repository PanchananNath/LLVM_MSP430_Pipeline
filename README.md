# LLVM_MSP430_Pipeline || LLVM to MSP430FR5969 Bare-Metal Compilation Guide

End-to-End Pipeline for optimization and checkpointing using LLVM_MSP430_Pipeline


## Overview

This document describes a complete, working pipeline to take a **C program**, convert it into **LLVM IR**, compile it for the **MSP430FR5969 (16‑bit MCU)**, and run it on real hardware using **Code Composer Studio (CCS)**.

The goal is to understand **LLVM as an intermediate compiler**, not as a full embedded toolchain replacement.

---

## System Setup

### Host Environment
- **OS**: WSL (Linux)
- **Host Architecture**: x86_64
- **LLVM / Clang Version**: 6.0.1

### Target Environment
- **Target MCU**: MSP430FR5969
- **Architecture**: 16‑bit
- **Endianness**: Little-endian
- **ABI**: MSP430 EABI
- **Execution**: Bare‑metal (no OS)

---

## Key Concepts

### What LLVM IR Is
- Intermediate representation
- Platform‑independent
- Not executable on hardware

### Why LLVM IR Cannot Run on MSP430
The MSP430 only understands **machine code produced by an MSP430 backend**.  
LLVM IR must be lowered using the MSP430 backend before it becomes usable.

---

## Correct Compilation Pipeline

```
C Source Code
   ↓
LLVM IR (.ll)
   ↓
LLVM MSP430 Backend
   ↓
MSP430 Object File (.o)
   ↓
Linking with MSP430 Runtime
   ↓
ELF Executable
   ↓
Flashed to MSP430FR5969
```

---

## Required Toolchain

| Component | Purpose |
|--------|--------|
| LLVM (clang, llc, opt) | IR generation and codegen |
| MSP430 LLVM backend | Target-specific codegen |
| MSP430 binutils | Assembler & linker |
| MSP430 libc | Startup + basic C runtime |
| Linker scripts | Memory layout |
| Programmer (MSP‑FET / LaunchPad) | Flashing |
| Debugger (optional) | Verification |

---

## Phase 1 — Prepare Bare‑Metal C Code

### Design Rules
- No `stdio`
- No `printf`
- No OS assumptions
- Use global `volatile` variables
- Infinite loop at end of `main()`

### Example Code (`add_sub_msp430.c`)

```c
volatile int add_result;
volatile int sub_result;

int add(int a, int b) {
    return a + b;
}

int subtract(int a, int b) {
    return a - b;
}

int main(void) {
    int x = 20;
    int y = 10;

    add_result = add(x, y);
    sub_result = subtract(x, y);

    while (1) {}
}
```

### Verify with GCC

```
msp430-gcc -mmcu=msp430fr5969 -Os add_sub_msp430.c -o test_gcc.elf
```

---

## Phase 2 — Generate LLVM IR

```
clang -S -emit-llvm add_sub_msp430.c -o add_sub.ll
```

Optional cleanup:

```
opt -S -mem2reg add_sub.ll -o add_sub_opt.ll
```

---

## Phase 3 — Retarget LLVM IR to MSP430

```
clang -S -emit-llvm   -target msp430   -mmcu=msp430fr5969   -ffreestanding   -nostdlib   add_sub_msp430.c   -o add_sub_msp430.ll
```

Expected IR header:

```
target datalayout = "e-p:16:16-i16:16-i32:16-n8:16"
target triple = "msp430"
```

---

## Phase 4 — LLVM IR → MSP430 Assembly

```
llc -march=msp430 add_sub_msp430.ll -o add_sub_msp430.s
```

Assemble:

```
msp430-as add_sub_msp430.s -o add_sub_msp430.o
```

---

## Phase 5 — Link Object to ELF

```
msp430-gcc -mmcu=msp430fr5969 add_sub_msp430.o -o add_sub_llvm.elf
```

Optional symbol inspection:

```
msp430-nm add_sub_llvm.elf | grep result
```

---

## Phase 6 — Flash & Debug in CCS

### Create Target‑Only Project
- Empty project
- Device: MSP430FR5969
- Compiler: None (or TI MSP430 Compiler)

### Load ELF Manually
```
Run → Load → Load Program…
Select add_sub_llvm.elf
```

---

## How to Confirm Execution

### What Execution Means
- Code runs once
- Results stored in RAM
- No LEDs, UART, or prints

### Verification Steps
1. Resume execution
2. Open **Expressions View**
3. Add:
   - `add_result`
   - `sub_result`

Expected values:
```
add_result = 30
sub_result = 10
```

This is **definitive proof of execution**.

---

## Final Result

You successfully built:

```
C Code
 ↓
LLVM IR
 ↓
MSP430‑targeted IR
 ↓
MSP430 Assembly
 ↓
Object File
 ↓
ELF Executable
 ↓
Running on MSP430FR5969
```

🎉 **Congratulations — you built a real LLVM‑to‑MCU pipeline.**

