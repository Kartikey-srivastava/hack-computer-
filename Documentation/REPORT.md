# Hack Computer Capstone Report

**Name:** Kartikey Srivastava  
**Roll number:** 2501010148  
**Personal K (last two digits):** 48  
**K mod 3:** 0  
**K mod 4:** 0  
**Box dimensions:** 16 × 1 = **16 pixels wide** × **20 rows high**  
**Top screen row:** 48  
**First screen word address:** 16384 + 32 × 48 = **17920**

---

## 1. Objective

The objective of this capstone is to build and test the **Hack computer** (from the Nand2Tetris course), implement a two-pass **Hack assembler** in Python, and write a personalised Hack assembly program (`Box.asm`) that displays a black rectangle while a keyboard key is held.

The three parts are:

| Part | Task | Key Files |
|------|------|-----------|
| A | HDL hardware implementation | `CPU.hdl`, `Memory.hdl`, `Computer.hdl` |
| B | Python two-pass assembler | `assembler.py`, `config.json` |
| C | Personalised box program | `generate_box.py`, `Box.asm`, `Box.hack` |

---

## 2. Part A — Hardware Implementation (HDL)

### 2.1 CPU.hdl

The CPU chip executes A-instructions and C-instructions on the 16-bit Hack architecture.

**Key design decisions:**

| Component | Purpose |
|-----------|---------|
| A Register | Holds a 16-bit address or immediate value. Loaded from the instruction word (A-instruction) or from the ALU output (C-instruction with `d1=1`). |
| D Register | Holds the ALU's data operand. Loaded only on a C-instruction when `d2=1`. |
| ALU | Performs 18 possible computations determined by control bits `instruction[6..11]`. |
| Program Counter (PC) | Increments each cycle; loaded with A's value when a jump condition is satisfied. |

**Instruction decoding logic:**

- **A-instruction** (`instruction[15] = 0`): the entire 16-bit word is loaded into the A register.
- **C-instruction** (`instruction[15] = 1`): bits `[12..6]` → ALU control; bits `[5..3]` → destination; bits `[2..0]` → jump condition.

The seven jump conditions (JGT, JEQ, JGE, JLT, JNE, JLE, JMP) are decoded from the ALU's `zr` (zero) and `ng` (negative) output flags. The PC is loaded with the A-register value only when the instruction is a C-instruction **and** the computed condition evaluates to true.

```hdl
// A register loads either an A-instruction value or the ALU result.
Not(in=instruction[15], out=isA);
And(a=instruction[15], b=instruction[5], out=cLoadA);
Or(a=isA, b=cLoadA, out=loadA);
Mux16(a=instruction, b=aluOut, sel=instruction[15], out=aInput);
ARegister(in=aInput, load=loadA, out=aOut, out[0..14]=addressM);
```

### 2.2 Memory.hdl

The Memory chip maps the 32 KB address space into three regions:

| Address Range | Region | Size |
|--------------|--------|------|
| 0 – 16383 | RAM (general purpose) | 16 K words |
| 16384 – 24575 | Screen (memory-mapped) | 8 K words |
| 24576 | Keyboard (memory-mapped) | 1 word (read-only) |

A `DMux4Way` selects which sub-chip receives the write signal based on `address[13..14]`, and a `Mux4Way16` routes the correct chip's output to the caller.

```hdl
DMux4Way(in=load, sel=address[13..14], a=loadRAM, b=loadScreen, ...);
RAM16K(in=in, load=loadRAM, address=address[0..13], out=ramOut);
Screen(in=in, load=loadScreen, address=address[0..12], out=screenOut);
Keyboard(out=keyboardOut);
Mux4Way16(a=ramOut, b=screenOut, c=false, d=keyboardOut, sel=address[13..14], out=out);
```

### 2.3 Computer.hdl

The top-level chip wires the CPU, Memory and ROM together:

```hdl
ROM32K(address=pc, out=instruction);
CPU(inM=memoryOut, instruction=instruction, reset=reset,
    outM=outM, writeM=writeM, addressM=addressM, pc=pc);
Memory(in=outM, load=writeM, address=addressM, out=memoryOut);
```

The program counter (`pc`) drives the ROM address, the ROM's output is the current instruction, and the CPU's data output goes to Memory.

**Test results (run in the official Nand2Tetris Hardware Simulator):**

| Test Script | Status |
|-------------|--------|
| CPU.tst | **Pass** |
| Memory.tst | **Pass** |
| ComputerAdd.tst | **Pass** |
| ComputerMax.tst | **Pass** |
| ComputerRect.tst | **Pass** |

---

## 3. Part B — Python Assembler

### 3.1 Architecture overview

The assembler (`Part-B-Assembler/assembler.py`) is a standard two-pass Hack assembler written in pure Python (stdlib only). It reads a `.asm` file and writes a `.hack` file of 16-bit binary strings.

```
assembler.py
├── clean_lines()   – strip comments and blank lines, track line numbers
├── assemble()      – two-pass core; writes symbols.txt
│   ├── Pass 1: scan for label declarations → populate symbol table with ROM addresses
│   └── Pass 2: translate each instruction → 16-bit binary; allocate variables from RAM 16
└── main()          – CLI wrapper; reads config.json for K
```

### 3.2 Predefined symbols

The symbol table is pre-seeded with 23 standard Hack symbols:

| Symbol | Address | Symbol | Address |
|--------|---------|--------|---------|
| SP | 0 | R0 – R15 | 0 – 15 |
| LCL | 1 | SCREEN | 16384 |
| ARG | 2 | KBD | 24576 |
| THIS | 3 | | |
| THAT | 4 | | |

### 3.3 Two-pass algorithm

**Pass 1 — Label scan:**
```
ROM counter = 0
for each line:
    if line is "(LABEL)":
        symbols[LABEL] = ROM counter   # label points to NEXT instruction
    else:
        ROM counter += 1
```

**Pass 2 — Instruction encoding:**
```
next_var = 16
for each instruction:
    if A-instruction (@symbol or @number):
        if numeric:
            encode as 0 + 15-bit binary of number
        else:
            look up / allocate symbol in RAM (from address 16 upward)
            encode as 0 + 15-bit binary of address
    if C-instruction:
        split on '=' and ';' to extract dest, comp, jump
        look up each field in DEST / COMP / JUMP tables
        encode as 111 + comp(7) + dest(3) + jump(3)
```

### 3.4 C-instruction encoding tables

```python
DEST = {"":"000","M":"001","D":"010","MD":"011",
        "A":"100","AM":"101","AD":"110","AMD":"111"}

JUMP = {"":"000","JGT":"001","JEQ":"010","JGE":"011",
        "JLT":"100","JNE":"101","JLE":"110","JMP":"111"}
```

The COMP table maps 28 mnemonic strings (e.g. `"D+M"`, `"!A"`, `"-1"`) to 7-bit patterns.

### 3.5 K-dependent extra feature

Since **K = 48**, we have **K mod 3 = 0**, which selects the **symbol table feature**: after every assembly run, the assembler writes a sorted `symbols.txt` listing every symbol and its resolved address.

```
Feature index = K mod 3 = 0  →  "save symbol table"
```

### 3.6 Symbol table output (from Box.asm assembly)

```
CHECK_KEY     0        DRAW          6
DRAW_ROW     14        ROW          16
COUNT        17        PTR          18
WIDTH        19        DRAW_WORD    24
ERASE        45        ERASE_ROW    53
ERASE_WORD   61        SCREEN    16384
KBD       24576
```

### 3.7 Sample assembly outputs

**Add.asm (`@2 \ D=A \ @3 \ D=D+A \ @0 \ M=D`):**

| Assembly | Binary |
|----------|--------|
| `@2` | `0000000000000010` |
| `D=A` | `1110110000010000` |
| `@3` | `0000000000000011` |
| `D=D+A` | `1110000010010000` |
| `@0` | `0000000000000000` |
| `M=D` | `1110001100001000` |

### 3.8 Unit tests

All 4 unit tests pass:

```
test_add_output                        ... ok
test_duplicate_label_warning_for_k2   ... ok
test_labels_and_variables              ... ok
test_line_number_error_for_k1          ... ok

Ran 4 tests in 0.052s — OK
```

| Test | What it verifies |
|------|-----------------|
| `test_add_output` | Exact binary output for a 3-instruction add program |
| `test_labels_and_variables` | Label resolves to correct ROM address; variable allocated from RAM 16 |
| `test_duplicate_label_warning_for_k2` | K mod 3 = 2 mode emits a duplicate-label warning |
| `test_line_number_error_for_k1` | K mod 3 = 1 mode includes line numbers in error messages |

---

## 4. Part C — Personalised Box Program

### 4.1 Geometry calculation

| Parameter | Formula | Value (K = 48) |
|-----------|---------|----------------|
| Top screen row | K | **48** |
| Height | 20 rows (fixed) | **20 rows** |
| Words per row | (K mod 4) + 1 | (48 mod 4) + 1 = **1 word** |
| Width | 16 × words per row | **16 pixels** |
| First screen word | SCREEN + 32 × K | 16384 + 1536 = **17920** |

### 4.2 Program flow

```
START:
  CHECK_KEY:
      D = Memory[KBD]      // read keyboard
      if D > 0: goto DRAW
      else: goto ERASE

  DRAW:
      ROW   = 17920        // first screen word for K = 48
      COUNT = 20           // 20 rows to fill
      DRAW_ROW:
          PTR   = ROW      // pointer to start of current row
          WIDTH = 1        // 1 word per row (K mod 4 = 0)
          DRAW_WORD:
              Memory[PTR] = -1   // paint 16 pixels black
              PTR++
              WIDTH--
              if WIDTH > 0: goto DRAW_WORD
          ROW += 32        // advance to next screen row
          COUNT--
          if COUNT > 0: goto DRAW_ROW
      goto CHECK_KEY

  ERASE:                   // same loop, writes 0 instead of -1
      ...
      goto CHECK_KEY
```

The value `-1` is `1111111111111111` in 16-bit two's complement, turning all 16 pixels of a screen word black. Writing `0` clears them.

### 4.3 Generated Box.asm (K = 48)

```asm
// Personal K = 48
// Top row = 48; height = 20 rows; width = 16 pixels (1 words).
(CHECK_KEY)
@KBD
D=M
@DRAW
D;JGT
@ERASE
0;JMP
(DRAW)
@17920
D=A
@ROW
M=D
@20
D=A
@COUNT
M=D
(DRAW_ROW)
@ROW
A=M
@ROW
D=M
@PTR
M=D
@1
D=A
@WIDTH
M=D
(DRAW_WORD)
@PTR
A=M
M=-1
@PTR
M=M+1
@WIDTH
MD=M-1
@DRAW_WORD
D;JGT
@ROW
D=M
@32
D=D+A
@ROW
M=D
@COUNT
MD=M-1
@DRAW_ROW
D;JGT
@CHECK_KEY
0;JMP
(ERASE)
@17920
D=A
@ROW
M=D
@20
D=A
@COUNT
M=D
(ERASE_ROW)
@ROW
D=M
@PTR
M=D
@1
D=A
@WIDTH
M=D
(ERASE_WORD)
@PTR
A=M
M=0
@PTR
M=M+1
@WIDTH
MD=M-1
@ERASE_WORD
D;JGT
@ROW
D=M
@32
D=D+A
@ROW
M=D
@COUNT
MD=M-1
@ERASE_ROW
D;JGT
@CHECK_KEY
0;JMP
```

**Assembly result:** 82 machine instructions written to `Box.hack`.

### 4.4 How to run

```powershell
# From project root
cd Part-C-Box-Program
python generate_box.py
cd ..\Part-B-Assembler
python assembler.py ..\Part-C-Box-Program\Box.asm -o ..\Part-C-Box-Program\Box.hack
cd ..\Part-C-Box-Program
python emulator.py Box.hack
```

Click the emulator window and **hold any key** → a 16-pixel-wide black rectangle appears at row 48. **Release** → rectangle erases.

---

## 5. One Mistake I Fixed

**Problem:** When first generating `Box.asm`, the config.json still held `K = 64` (the placeholder default). The assembler was called with this value, so the generated box appeared at row 64 instead of the correct row 48.

**Cause:** The `generate_box.py` script reads `K` from `Part-B-Assembler/config.json` at runtime; the config file had not been personalised before the first generation run.

**Fix:** Updated `config.json` to `"K": 48` and re-ran `generate_box.py`. The screen base address changed from:

> 16384 + 32 × 64 = **18432** → 16384 + 32 × 48 = **17920**

This is visible in the git diff:

```diff
-// Personal K = 64
-// Top row = 64; height = 20 rows; width = 16 pixels (1 words).
+// Personal K = 48
+// Top row = 48; height = 20 rows; width = 16 pixels (1 words).
 ...
-@18432
+@17920
```

**Verification:** Re-ran `python assembler.py Box.asm` and confirmed the output reported `K=48; selected extra feature: save symbol table`. All 4 unit tests still passed (`Ran 4 tests in 0.052s — OK`).

---

## 6. Conclusion

This capstone demonstrated the complete stack from gates to running programs:

| Layer | Lesson learned |
|-------|---------------|
| **Hardware (HDL)** | How 16-bit instruction fields directly map to mux selectors, register load signals and PC jump conditions. A single wrong gate output cascades silently through the entire instruction pipeline. |
| **Binary instructions** | C-instructions pack three independently decoded fields (comp, dest, jump) into 13 bits. The `a-bit` (instruction[12]) alone switches the ALU between using the A register and reading from RAM. |
| **Symbol resolution** | Two-pass assembly is necessary because a forward jump references a label before its ROM address is known. The first pass fixes all labels; the second pass uses those resolved addresses. |
| **Memory-mapped I/O** | The screen and keyboard are not special hardware registers — they are regions of the same address bus. Writing `-1` to a screen word turns 16 pixels on; reading address 24576 returns the keyboard scancode. |
| **Screen drawing** | Each row of the 512-pixel screen occupies exactly 32 consecutive memory words. The box top row K = 48 starts at word 16384 + 32 × 48 = 17920; iterating 20 rows means stepping the pointer by 32 each time. |

The project is fully committed and pushed to [https://github.com/Kartikey-srivastava/hack-computer-](https://github.com/Kartikey-srivastava/hack-computer-).
