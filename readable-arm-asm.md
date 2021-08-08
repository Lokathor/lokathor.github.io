
# Alternate ARM Assembly

This is a draft of a potential alternate way to read/write ARM assembly.

The goal is to make an assembly language variant that looks more like "normal" programming.

This is purely a textual change, all instructions behave exactly the same as before.

The document focuses on just ARMv4T, because that's what the GBA uses.

## Thumb / t32

### Addition

| asm | pseudo |
|:-|:-|
| `ADC Rd, Rm` | `rd +=C rm` (1) |
| `ADD Rd, Rn, #<imm3>` | `rd = rn + imm3` |
| `ADD Rd, #<imm8>` | `rd += imm8` |
| `ADD Rd, Rn, Rm` | `rd = rn + rm` (2) |
| `ADD Rd, Rm` | `rd +=H rm` (3) |
| `ADD Rd, PC, #<imm8>` | `rd = pc + imm8` |
| `ADD Rd, SP, #<imm8>` | `rd = sp + imm8` |
| `ADD SP, SP, #<imm7>` | `sp += imm7` |

1. The `+=C` operation is like `+=`, but with an extra +1 if the Carry flag is set
2. This can be written as `rd += rn` if one of the source registers is also the destination.
3. The `+=H` operation can work with high registers.
   You can add low and high, high and low, or high and high.
   The condition flags are *not* set when there's addition using high registers.

### Bitwise AND

| asm | pseudo |
|:-|:-|
| `AND Rd, Rm` | `rd &= rm` |

### Bit Shifting

| asm | pseudo |
|:-|:-|
| `ASR Rd, Rm, #<right_shift_imm>` | `rd = rm a>> right_shift_imm` |
| `ASR Rd, Rs` | `rd a>>= rs` |
| `LSL Rd, Rm, #<imm5>` | `rd = rm << imm5` |
| `LSL Rd, Rs` | `rd <<= rs` |
| `LSR Rd, Rm, #<right_shift_imm>` | `rd = rm >> right_shift_imm` |
| `LSR Rd, Rs` | `rd >>= rs` |

`right_shift_imm`: is written as 1 through 32, then encoded as 5 bits with 32 mapping to 0.

`a>>` and `a>>=` arithmetic shifting, it shifts in sign bits.
All other shifts are logical shifts, they shift in zero bits.

### Branching

| asm | pseudo |
|:-|:-|
| `B<cond> <target_address>` | `b<cond>(target_address)` |
| `B <target_address>` | `b(target_address)` |
| `BL <target_address>` | `bl(target_address)` |
| `BX Rm` | `bx(rm)` |

`cond`: Can be any of the various two letter condition mnemonics normally used.
Examples: `bgt`, `beq`.

### Bit Clear

| asm | pseudo |
|:-|:-|
| `BIC Rd, Rm` | `rd &!= rm` |

If a bit is set in `rm`, it will be cleared from `rd`.

```
// equivalent
rd = rd & not(rm)
```

### Comparison

| asm | pseudo |
|:-|:-|
| `CMN Rn, Rm` | `cmp(rn, !rm)` |
| `CMP Rn, #<imm8>` | `cmp(rn, imm8)` |
| `CMP Rn, Rm` | `cmp(rn, rm)` (1) |
| `TST Rn, Rm` | `test(rn, rm)` |

1. This form allows a high register to be selected.

There's no destination register for a comparison, it just updates the status flags as follows:

* Comparing a register and a negated register works like `rn + rm`.
* Comparing a register and an immediate is like `rn - imm8`.
* Comparing two registers is like `rn - rm`.
* Testing two registers is like `rn & rm`.

Note: precisely how you encode the instruction for comparison of two registers depends on if a high register is used or not in the comparison.
However, the user doesn't need to care about that since (unlike with addition) nothing else is affected by the difference.

### Exclusive Bitwise OR

| asm | pseudo |
|:-|:-|
| `EOR Rd, Rm` | ` rd ^= rm` |

### Load

| asm | pseudo |
|:-|:-|
| `LDMIA Rn!, <reg_list>` | `<reg_list> = ldmia(rn!)` |
| `LDR Rd, [Rn, #imm5_offset]` | `rd = load(rn + imm5_offset)` |
| `LDR Rd, [Rn, Rm]` | `rd = load(rn + rm)` |
| `LDR Rd, [PC, #imm8_offset]` | `rd = load(pc + imm8_offset)` |
| `LDR Rd, [SP, #imm8_offset]` | `rd = load(sp + imm8_offset)` |
| `LDRB Rd, [Rn, #imm5_offset]` | `rd = load_u8(rn + imm5_offset)` (1) |
| `LDRB Rd, [Rn, Rm]` | `rd = load_u8(rn + rm)` (1) |
| `LDRH Rd, [Rn, #imm5_offset]` | `rd = load_u16(rn + imm5_offset)` (2) |
| `LDRH Rd, [Rn, Rm]` | `rd = load_u16(rn + rm)` (2) |
| `LDRSB Rd, [Rn, Rm]` | `rd = load_i8(rn + rm)` (3) |
| `LDRSH Rd, [Rn, Rm]` | `rd = load_i16(rn + rm)` (4) |

1. loads bits 0 through 7, other bits are zeroed.
2. loads bits 0 through 15, other bits are zeroed.
3. loads bits 0 through 7, other bits are sign extended.
3. loads bits 0 through 15, other bits are sign extended.

`reg_list`: a list of registers to fill with data, written in curly braces.
Registers are comma separated, and `rA-rB` is allowed for register ranges.
All registers and register ranges must be listed in order.

### Move

| asm | pseudo |
|:-|:-|
| `MOV Rd, #<imm8>` | `rd = imm8` |
| `MOV Rd, Rm` | `rd = rm` (1) |
| `MVN Rd, Rm` | `rd = !rm` |

1. This form allows a high register to be selected.

Note: precisely how you encode the instruction for moving between two registers depends on if a high register is used or not in the comparison.
However, the user doesn't need to care about that since (unlike with addition) nothing else is affected by the difference.

### Multiplication

| asm | pseudo |
|:-|:-|
| `MUL Rd, Rm` | `rd *= rm` |

### Negation

| asm | pseudo |
|:-|:-|
| `NEG Rd, Rm` | `rd = -rm` |

### Bitwise OR

| asm | pseudo |
|:-|:-|
| `ORR Rd, Rm` | `rd \|= rm` |

### Pop / Push

| asm | pseudo |
|:-|:-|
| `POP {<register_list>, <PC>}` | `pop(register_list, pc)` |
| `PUSH {<register_list>, <LR>}` | `push(register_list, lr)` |

### Right Rotate

| asm | pseudo |
|:-|:-|
| `ROR Rd, Rs` | `rd ror= rs` |

### Subtraction

| asm | pseudo |
|:-|:-|
| `SBC Rd, Rm` | `rd -=C rm` (1) |
| `SUB Rd, Rn, #<imm3>` | `rd = rn - imm3` |
| `SUB Rd, #<imm8>` | `rd -= imm8` |
| `SUB Rd, Rn, Rm` | `rd = rn - rm` (2) |
| `SUB, SP, SP, #<imm7>` | `sp -= imm7` |

1. The `-=C` operation is like `-=`, but with an extra -1 if the Carry flag is *not* set.
2. This can be written as `rd -= rn` if one of the source registers is also the destination.

### Store

| asm | pseudo |
|:-|:-|
| `STMIA Rn!, <register_list>` | `stmia(rn!, {register_list})` |
| `STR Rd, [Rn, #imm5_offset]` | `[rn + #imm5_offset] = rd` |
| `STR Rd, [Rn, Rm]` | `[rn + rm] = rd` |
| `STR Rd, [SP, #imm8_offset]` | `[sp + imm8_offset] = rd` |
| `STRB Rd, [Rn, #imm5_offset]` | `[rn + imm5_offset].u8 = rn` |
| `STRB Rd, [Rn, Rm]` | `[rn + rm].u8 = rd` |
| `STRH Rd, [Rn, #imm5_offset]` | `[rn + imm5_offset].u16 = rd` |
| `STRH Rd, [Rn, Rm]` | `[rn + rm].u16 = rd` |

This is the portion of the draft I'm least happy with, but it works for now.

### Software Interrupt

| asm | pseudo |
|:-|:-|
| `SWI <imm8>` | `swi(imm8)` |

### Example

Here's a sample from the [Tonc assembly tutorial](https://www.coranac.com/tonc/text/asm.htm).

Before
```
PlotPixel3:
    lsl     r3, r1, #4
    sub     r3, r3, r1
    lsl     r3, r3, #4
    add     r3, r3, r0
    mov     r1, #192
    lsl     r1, r1, #19
    lsl     r3, r3, #1
    add     r3, r3, r1
    strh    r2, [r3]
    bx      lr
```

After
```
PlotPixel3:
    r3 = r1 << 4
    r3 -= r1
    r3 <<= 4
    r3 += r0
    r1 = 192
    r1 <<= 19
    r3 <<= 1
    r3 += r1
    [r3] = r2
    bx(lr)
```

<!--
main:
    push    {r4, lr}
    mov     r4, #0
.L4:
    mov     r2, #248
    lsl     r2, r2, #2
    mov     r0, r4
    mov     r1, #16
    add     r4, r4, #1
    bl      PlotPixel3
    cmp     r4, #240
    bne .L4
    ldr     r3, .L14
    ldr     r2, .L14+4
    mov     r1, #31
.L6:
    strh    r1, [r3]
    add     r3, r3, #2
    cmp     r3, r2
    bne .L6
.L12:
    b .L12
-->
