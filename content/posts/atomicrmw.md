---
title: "LLVM's `atomicrmw` instruction and x86_64"
date: 2022-08-31T01:50:33-07:00
draft: false
summary: "A grimoire of how LLVM translates atomicrmw to x86_64 assembly."
---

I've been working on an [LLVM-IR-to-x86_64-assembly compiler](https://github.com/heimskr/ll2x) recently.
One of the instructions my compiler translates is the [`atomicrmw`](https://llvm.org/docs/LangRef.html#atomicrmw-instruction)
instruction, which does math/memory modification atomically. I tested the outputs for all the different operations
except for the floating point ones because I've yet to implement floating point support. The results are listed below.
Note that the generated assembly can vary depending on whether the result of the operation is used.

### Full IR example

```llvm
define void @atomic(i64 %0, i64* %1) align 2 {
    %3 = alloca i64, align 8
    %4 = atomicrmw volatile umin i64* %1, i64 %0 acq_rel, align 8
    store i64 %4, i64* %1, align 8
    ret void
}
```

### The `xchg` operation

```llvm
%3 = atomicrmw volatile xchg i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used or unused result
xchgq  %rdi, (%rsi)
```

### The `add` operation

```llvm
%3 = atomicrmw volatile add i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used result
lock  xaddq  %rdi, -8(%rsp)

# Unused result
lock  addq  %rdi, -8(%rsp)
```

### The `sub` operation

```llvm
%3 = atomicrmw volatile sub i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used result
negq  %rax
lock  xaddq  %rax, (%rsi)

# Unused result
negq  %rax
lock  addq  %rax, (%rsi)
```

### The `and` operation

```llvm
%3 = atomicrmw volatile and i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used result
.LBB0_1:
movq  %rax, %rcx
andq  %rdi, %rcx
lock  cmpxchgq  %rcx, (%rsi)
jne   .LBB0_1

# Unused result
lock  andq  %rdi, -8(%rsp)
```

### The `nand` operation

```llvm
%3 = atomicrmw volatile nand i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used or unused result
.LBB0_1:
movq  %rax, %rcx
andq  %rdi, %rcx
notq  %rcx
lock  cmpxchgq  %rcx, (%rsi)
jne   .LBB0_1
```

### The `or` operation

```llvm
%3 = atomicrmw volatile or i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used result
.LBB0_1:
movq  %rax, %rcx
orq   %rdi, %rcx
lock  cmpxchgq  %rcx, (%rsi)
jne   .LBB0_1

# Unused result
lock  orq  %rdi, (%rsi)
```

### The `xor` operation

```llvm
%3 = atomicrmw volatile xor i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used result
.LBB0_1:
movq  %rax, %rcx
xorq  %rdi, %rcx
lock  cmpxchgq  %rcx, (%rsi)
jne   .LBB0_1

# Unused result
lock  xorq  %rdi, (%rsi)
```

### The `max` operation

```llvm
%3 = atomicrmw volatile max i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used or unused result
.LBB0_1:
cmpq    %rdi, %rax
movq    %rdi, %rcx
cmovgq  %rax, %rcx
lock    cmpxchgq  %rcx, (%rsi)
jne     .LBB0_1
```

### The `min` operation

```llvm
%3 = atomicrmw volatile min i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used or unused result
.LBB0_1:
cmpq     %rdi, %rax
movq     %rdi, %rcx
cmovleq  %rax, %rcx
lock     cmpxchgq  %rcx, (%rsi)
jne      .LBB0_1
```

### The `umax` operation

```llvm
%3 = atomicrmw volatile umax i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used or unused result
.LBB0_1:
cmpq    %rdi, %rax
movq    %rdi, %rcx
cmovaq  %rax, %rcx
lock    cmpxchgq  %rcx, (%rsi)
jne     .LBB0_1
```

### The `umin` operation

```llvm
%3 = atomicrmw volatile umin i64* %2, i64 %0 acq_rel, align 8
```

```gas
# Used or unused result
.LBB0_1:
cmpq     %rdi, %rax
movq     %rdi, %rcx
cmovbeq  %rax, %rcx
lock     cmpxchgq  %rcx, (%rsi)
jne      .LBB0_1
```