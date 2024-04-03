---
layout: post
title: Learning Haskell
published: false
---

My friend vernonlim was learning about Lean, Haskell and functional programming for university and self-interest. Later on we decided to write two chip8 emulator implementations at the same time, he wrote it in Java and I wrote it in Haskell. So on Febuary 17th I started writing and learning about Haskell.

## Basic Implementation

I'd heard and read _a bit_ about Monads before but I decided to not try to use them this time before I fully understood them. I started the chip8 implementation by writing some functions to handle basic opcodes, taking in a "machine" and each nibble of the opcode in a 4-tuple.

```haskell
data Machine = Machine
  { regv :: [Word8]
  }

runOpcode :: (Word8, Word8, Word8, Word8) -> Machine -> Machine
```

This runOpcode signatures means it's a function that takes the split up opcode tuple, and then it applies the corresponding operating to the machine and returns the new machine. The machine data structure is a "struct" containing the state of the chip8 (virtual-)machine.

This is the code for the 0x6VNN opcode where V refers to the register index and NN refers to a 8-bit number.

```haskell
runOpcode (6, vIndex, valueA, valueB) machine = machine { regv = newRegv }
    where value = mergeNibble8 valueA valueB
          newRegv = replaceNth vIndex value (regv machine)
```

The where clause at the end specifies a list of values that can be used in the function expression. The `replaceNth` function replaces a value at an index, so in this case it returns a new array where the `vIndex`th value of `machine.regv` is replaced with `value`. Then the function expression says to replace the machine's regv field with the newRegv value. So in the end calling this function will return a new machine whose register has been updated based on the opcode parameters.

# Generalized functions

In order to generalize register manipulation I created two functions called `applyToReg` and `applyTo2Regs` which operate on a single register and two registers respectively.
