---
rskip: 120
title: Shifting opcodes  
description: 
status: Draft
purpose: Sca
author: SMS (@sebastians)
layer: Core
complexity: 1
created: 2019-04-23
---
# Shifting opcodes

|RSKIP          |120           |
| :------------ |:-------------|
|**Title**      |Shifting opcodes |
|**Created**    |23-APR-19 |
|**Author**     |SMS |
|**Purpose**    |Sca |
|**Layer**      |Core|
|**Complexity** | 1 |
|**Status**     |Draft |

## Abstract

The purpose of this RSKIP is to add the opcodes of shift left, right and arithmetic shift right, as added to the Ethereum VM in the [EIP 145](https://eips.ethereum.org/EIPS/eip-145).

## Motivation

The RSK VM is lacking bitwise shifting operators, but this can be simulated by dividing by a power of two. The cost of this division is significantly higher than the cost of shifting, which takes 3 gas. The following describes the requirements made by the Ethereum team which are implemented in their VM.

## Specification


The following instructions are introduced:

### `0x1b`: `SHL` (shift left)

The `SHL` instruction (shift left) pops 2 values from the stack, first `arg1` and then `arg2`, and pushes on the stack `arg2` shifted to the left by `arg1` number of bits. The result is equal to

```
(arg2 * 2^arg1) mod 2^256
```

Notes:
- The value (`arg2`) is interpreted as an unsigned number.
- The shift amount (`arg1`) is interpreted as an unsigned number.
- If the shift amount (`arg1`) is greater or equal 256 the result is 0.
- This is equivalent to `PUSH1 2 EXP MUL`.

### `0x1c`: `SHR` (logical shift right)

The `SHR` instruction (logical shift right) pops 2 values from the stack, first `arg1` and then `arg2`, and pushes on the stack `arg2` shifted to the right by `arg1` number of bits with zero fill. The result is equal to

```
floor(arg2 / 2^arg1)
```

Notes:
- The value (`arg2`) is interpreted as an unsigned number.
- The shift amount (`arg1`) is interpreted as an unsigned number.
- If the shift amount (`arg1`) is greater or equal 256 the result is 0.
- This is equivalent to `PUSH1 2 EXP DIV`.

### `0x1d`: `SAR` (arithmetic shift right)

The `SAR` instruction (arithmetic shift right) pops 2 values from the stack, first `arg1` and then `arg2`, and pushes on the stack `arg2` shifted to the right by `arg1` number of bits with sign extension. The result is equal to

```
floor(arg2 / 2^arg1)
```

Notes:
- The value (`arg2`) is interpreted as a signed number.
- The shift amount (`arg1`) is interpreted as an unsigned number.
- If the shift amount (`arg1`) is greater or equal 256 the result is 0 if `arg2` is non-negative or -1 if `arg2` is negative.
- This is **not** equivalent to `PUSH1 2 EXP SDIV`, since it rounds differently. See `SDIV(-1, 2) == 0`, while `SAR(-1, 1) == -1`.

The cost of the shift instructions is set at `verylow` tier (3 gas).

## Rationale

Instruction operands were chosen to fit the more natural use case of shifting a value already on the stack. This means the operand order is swapped compared to most arithmetic insturctions.

## Backwards Compatibility

The newly introduced instructions have no effect on bytecode created in the past.

## Test Cases

### `SHL` (shift left)

1. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x00
   SHL
   ---
   0x0000000000000000000000000000000000000000000000000000000000000001
   ```
2. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x01
   SHL
   ---
   0x0000000000000000000000000000000000000000000000000000000000000002
   ```
3. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0xff
   SHL
   ---
   0x8000000000000000000000000000000000000000000000000000000000000000
   ```
4. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x0100
   SHL
   ---
   0x0000000000000000000000000000000000000000000000000000000000000000
   ```
5. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x0101
   SHL
   ---
   0x0000000000000000000000000000000000000000000000000000000000000000
   ```
6. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0x00
   SHL
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
7. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0x01
   SHL
   ---
   0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe
   ```
8. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0xff
   SHL
   ---
   0x8000000000000000000000000000000000000000000000000000000000000000
   ```
9. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0x0100
   SHL
   ---
   0x0000000000000000000000000000000000000000000000000000000000000000
   ```
10. ```
    PUSH 0x0000000000000000000000000000000000000000000000000000000000000000
    PUSH 0x01
    SHL
    ---
    0x0000000000000000000000000000000000000000000000000000000000000000
    ```
11. ```
    PUSH 0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    PUSH 0x01
    SHL
    ---
    0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe
    ```


### `SHR` (logical shift right)

1. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x00
   SHR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000001
   ```
2. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x01
   SHR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000000
   ```
3. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0x01
   SHR
   ---
   0x4000000000000000000000000000000000000000000000000000000000000000
   ```
4. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0xff
   SHR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000001
   ```
5. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0x0100
   SHR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000000
   ```
6. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0x0101
   SHR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000000
   ```
7. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0x00
   SHR
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
8. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0x01
   SHR
   ---
   0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
9. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0xff
   SHR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000001
   ```
10. ```
    PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    PUSH 0x0100
    SHR
    ---
    0x0000000000000000000000000000000000000000000000000000000000000000
    ```
11. ```
    PUSH 0x0000000000000000000000000000000000000000000000000000000000000000
    PUSH 0x01
    SHR
    ---
    0x0000000000000000000000000000000000000000000000000000000000000000
    ```

### `SAR` (arithmetic shift right)

1. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x00
   SAR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000001
   ```
2. ```
   PUSH 0x0000000000000000000000000000000000000000000000000000000000000001
   PUSH 0x01
   SAR
   ---
   0x0000000000000000000000000000000000000000000000000000000000000000
   ```
3. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0x01
   SAR
   ---
   0xc000000000000000000000000000000000000000000000000000000000000000
   ```
4. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0xff
   SAR
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
5. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0x0100
   SAR
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
6. ```
   PUSH 0x8000000000000000000000000000000000000000000000000000000000000000
   PUSH 0x0101
   SAR
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
7. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0x00
   SAR
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
8. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0x01
   SAR
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
9. ```
   PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   PUSH 0xff
   SAR
   ---
   0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
   ```
10. ```
    PUSH 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    PUSH 0x0100
    SAR
    ---
    0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    ```
11. ```
    PUSH 0x0000000000000000000000000000000000000000000000000000000000000000
    PUSH 0x01
    SAR
    ---
    0x0000000000000000000000000000000000000000000000000000000000000000
    ```
12. ```
    PUSH 0x4000000000000000000000000000000000000000000000000000000000000000
    PUSH 0xfe
    SAR
    ---
    0x0000000000000000000000000000000000000000000000000000000000000001
    ```
13. ```
    PUSH 0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    PUSH 0xf8
    SAR
    ---
    0x000000000000000000000000000000000000000000000000000000000000007f
    ```
14. ```
    PUSH 0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    PUSH 0xfe
    SAR
    ---
    0x0000000000000000000000000000000000000000000000000000000000000001
    ```
15. ```
    PUSH 0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    PUSH 0xff
    SAR
    ---
    0x0000000000000000000000000000000000000000000000000000000000000000
    ```
16. ```
    PUSH 0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    PUSH 0x0100
    SAR
    ---
    0x0000000000000000000000000000000000000000000000000000000000000000
    ```


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
