+++
date = '2026-04-01T12:54:27+05:30'
draft = true
title = 'Wide Div Instruction Lowering With Chunk Summation in LLVM'
+++

More recently I have contributed a patch (https://github.com/llvm/llvm-project/commit/796b218edd358ab112e876e9c22490987dae7f06) related to div instruction optimization for wider 128 integer types with some constant divisors. 

Before the fix llvm was generating libcalls to compiler-rt for those instructions which has a huge cost of function calls to external libraries, also they can not be optimized by the compiler passes.

## Background -
128 bit type integers have mainly been used in cryptography, HPC and other financial applications which require high precision math. 

Since we do not have 128 bit registers for most hardwares, we can not have native division instruction. Also division have a high cost so even on the target that supports div, division is usually performed by mul, add and shift instructions. 

### LLVM previous Implementation -
So If we talk about 128 support in LLVM. It was generating a libcall of __umodti3 for rem and __udivdi3 for div which performs a serialized shift-subtract loop over the bit width.
```
--- compiler-rt/lib/builtins/udivmodti4.c
for (; shift >= 0; --shift) {
    quotient <<= 1;
    if (dividend >= divisor) {
        dividend -= divisor;
        quotient |= 1;
    }
    divisor >>= 1;
  }
  ```
 
### Earlier codegen for this look like this -
```
div_by_7(unsigned __int128):
       push    rax
       mov     edx, 7
       xor     ecx, ecx
       call    __udivti3@PLT
       pop     rcx
       ret
```

### GCC x86 Codegen -
However GCC was not using libcalls, It has a trick of Hacker delight’s division by constant with the variable chunk size. Following is the output for that routine -
```
div_by_7(unsigned __int128):
       movabs  rcx, 1152921504606846975
       mov     r8, rdi
       mov     r9, rsi
       mov     rax, r8
       and     rdi, rcx
       shrd    rax, rsi, 60
       shr     rsi, 56
       and     rax, rcx
       add     rdi, rax
       movabs  rax, 2635249153387078803
       add     rsi, rdi
       mul     rsi
       mov     rax, rsi
       sub     rax, rdx
       shr     rax
       add     rdx, rax
       shr     rdx, 2
       lea     rax, [0+rdx*8]
       sub     rax, rdx
       movabs  rdx, -5270498306774157605
       sub     rsi, rax
       sub     r8, rsi
       movabs  rsi, 7905747460161236407
       sbb     r9, 0
       imul    rdx, r8
       mov     rax, r8
       mov     rcx, r9
       imul    rcx, rsi
       add     rcx, rdx
       mul     rsi
       add     rcx, rdx
       mov     rdx, rcx
       ret
```


### Hacker Delight remainder trick -  
The original idea was mentioned in the Hacker delights book, 10-19 Remainder by Summing Digits -
```
static char table[62] = {0,1,2,3,4, 0,1,2,3,4,
0,1,2,3,4, 0,1,2,3,4, 0,1,2,3,4, 0,1,2,3,4,
0,1,2,3,4, 0,1,2,3,4, 0,1,2,3,4, 0,1,2,3,4,
0,1,2,3,4, 0,1,2,3,4, 0,1};

int remu7(unsigned n) {
  static char table[75] = {0, 1, 2, 3, 4, 5, 6, 0, 1, 2, 3, 4, 5, 6, 0,
                           1, 2, 3, 4, 5, 6, 0, 1, 2, 3, 4, 5, 6, 0, 1,
                           2, 3, 4, 5, 6, 0, 1, 2, 3, 4, 5, 6, 0, 1, 2,
                           3, 4, 5, 6, 0, 1, 2, 3, 4, 5, 6, 0, 1, 2, 3,
                           4, 5, 6, 0, 1, 2, 3, 4, 5, 6, 0, 1, 2, 3, 4};
  n = (n >> 15) + (n & 0x7FFF); // Max 0x27FFE.
  n = (n >> 9) + (n & 0x001FF); // Max 0x33D.
  n = (n >> 6) + (n & 0x0003F); // Max 0x4A.
  return table[n];
}
```

To better understand this with example, let's remember the trick we used to study in schools -  
A number is divisible by 9 if the sum of its digits is divisible by 9, in binary so divisor 9 have some special property,
Similarly a number is divisible by d if the sum of its W-bit "digits" is divisible by d, provided (2^W (mod) D) = 1.

So what does this constraint mean? Why is it there? -  
Let's write it as   
```
2^W ≡ 1 (modD)
```
Which means multiplying by `2^W` is the same as multiplying by `1(mod D)`.  

Now think about a number split into chunks of size W bits (for example 60):  
```
N = a0 + a1*2^W + a2*2^2W + ...
```
Now apply modulo D:  
Since:  
```
2^W≡1
```
Then:  
```
2^2W ≡ 1, 2^3W ≡ 1 
```
So the number becomes:  
```
N ≡ a0 + a1 + a2 + ... (mod D)
```
All powers are collapsed to 1

Every chunk contributes equally modulo D, so instead of doing full division, we can just add the chunks together before division. And this is what we will be doing in our implementation shown below in the next section. We will partition the dividend into smaller chunks.

### Real World example -
Lets takes this number as dividend - 
```
N = 101011001
1*2^8 + 0*2^7 + 1*2^6 + 0*2^5 + 1*2^4 + 1*2^3 + 0*2^2 + 0*2^1 + 1*2^0
256+64+16+8+1 = 345
```
Now lets div by 7 
We get 345/7 = 49 as quotient and 2 is remainder. 

For this 7 divisor we have 3 which satisfies the property (2^W (mod) D) = 1.
```
(2^3) % 7 = 1
```
So let's divide this into the chunk size of three. We will have three chunks and we sum them.
```
N1 = 101 -> 5
N2 = 011 -> 3
N3 = 001 -> 1
5 + 3 + 1 = 9
```

And take the modulo 
```
9 mod 7 = 2
```

So by this method we also get the same remainder but we have reduced the input N into smaller chunks.  

To get quotient when we already have remainder, we usually do -  
```
345 = Q⋅7 + 2
Q = 343 / 7 = 49
```
Which means this is the formula -  
```
Q = (N − R)/ D
```
Now you are wondering if we need to divide again. For this case we use multiplicative inverse formula -  
Which is  
```
Q = (N−R) * D ^ −1(mod 2^n)
```
Here R is the remainder we calculated earlier.  
 
### Performance Number -
We studied the theory, before we get into LLVM implementation lets read the numbers on how much performance it improves -  

"Before vs. After" comparison on google micro-benchmark of division on Macbook Air m4 which is arm64 chip.  
```
Before
Running ./div
Run on (10 X 24 MHz CPU s)
CPU Caches:
  L1 Data 64 KiB
  L1 Instruction 128 KiB
  L2 Unified 4096 KiB (x10)
Load Average: 1.68, 1.86, 2.80
-----------------------------------------------------------
Benchmark                 Time             CPU   Iterations
-----------------------------------------------------------
BM_Uint128DivBy7       3.21 ns         3.21 ns    217350129
---------------------------------------------------------------

After 
Running ./div
Run on (10 X 24 MHz CPU s)
CPU Caches:
  L1 Data 64 KiB
  L1 Instruction 128 KiB
  L2 Unified 4096 KiB (x10)
Load Average: 2.33, 2.02, 2.77
-----------------------------------------------------------
Benchmark                 Time             CPU   Iterations
-----------------------------------------------------------
BM_Uint128DivBy7       2.10 ns         2.10 ns    332630058
```
We saw **35%** performance gain with Google benchmark. 


### LLVM implementation -
#### GlobalISel -
Before discussing the summing digits trick with variable chunks, let's see what the current approach for GlobalISel following. Instead of generating libcalls it is using the popular magic multiplication algorithm (https://github.com/llvm/llvm-project/blob/96db944036a09e773d1e3fff4fa3805a4f75e59e/llvm/lib/CodeGen/GlobalISel/CombinerHelper.cpp#L5578).
It generates code like this on aarch64 - 
```
ui128_7:
      %bb.0: // %entry
      mov x8, #18725
      mov x10, #9362 
      movk x8, #9362, lsl #16
      movk x10, #37449, lsl #16
      movk x8, #37449, lsl #32
      movk x10, #18724, lsl #32
      movk x8, #18724, lsl #48
      movk x10, #9362, lsl #48
      mul x9, x1, x8
      mul x11, x0, x10
      umulh x12, x0, x8
      mul x13, x1, x10
      adds x9, x9, x11
      umulh x14, x1, x8
      cset w11, hs
      cmn x9, x12
      and x11, x11, #0x1
      and x12, xzr, #0x1
      umulh x15, x0, x10
      cset w9, hs
      and x9, x9, #0x1
      umulh x8, xzr, x8
      add x9, x11, x9
      and x11, xzr, #0x1
      adds x13, x13, x14
      add x11, x11, x12
      umulh x10, x1, x10
      cset w12, hs
      adds x13, x13, x15
      and x12, x12, #0x1
      umulh x14, x0, xzr
      cset w15, hs
      adds x9, x13, x9
      add x11, x11, x12
      and x12, x15, #0x1
      cset w13, hs
      add x11, x11, x12
      and x12, x13, #0x1
      add x8, x8, x10
      add x10, x11, x12
      add x8, x8, x14
      add x8, x8, x10
      subs x10, x0, x9
      sbc x11, x1, x8
      lsr x11, x11, #1
      adds x9, x10, x9
      mov w10, #7 // =0x7
      adc x8, x11, x8
      extr x9, x8, x9, #2
      lsr x8, x8, #2
      umulh x10, x9, x10
      lsl x11, x9, #3
      lsl x12, x8, #3
      sub x9, x11, x9
      sub x8, x12, x8
      subs x0, x0, x9
      add x8, x8, x10
      sbc x1, x1, x8
      ret
```
Compared to that chunk summation algorithms produce 
```
ui128_7:
       %bb.0:
       extr x8, x1, x0, #60
       and x9, x0, #0xfffffffffffffff
       and x8, x8, #0xfffffffffffffff
       add x8, x9, x8
       mov x9, #18725
       movk x9, #9362, lsl #16
       add x8, x8, x1, lsr #56
       mov x1, xzr
       movk x9, #37449, lsl #32
       movk x9, #18724, lsl #48
       umulh x9, x8, x9
       lsr x9, x9, #1
       sub x9, x9, x9, lsl #3
       add x0, x8, x9
       ret
```

This is much more efficient and simpler code than the magic multiplicative inverse algorithm.  

Chunk summation is more optimized but also relies on the special property of the divisor to satisfy - `2^I % Divisor == 1`. So it can only apply to divisors like 7, 34, 100 etc.

### SelectionDAG implementation -
Let's see how the Summing Digits algorithm is implemented in LLVM (https://github.com/xgupta/llvm-project/blob/16e06589e963aadced8a5afeeaa2ec2a95d0a483/llvm/lib/CodeGen/SelectionDAG/TargetLowering.cpp#L8167).

Before the my commit the Codegen had summing digits technique but only for 
Divisor like 12 where `(1 << (Bitwidth / 2)) % Divisor == 1` for which chunk size is fixed at 2, here we can add the high and low halves together and use a urem. This lefts many common divisors.

To generalise it to support more integers we need to use variable chunks not just two. As you saw gcc was using three chunks like `60 + 60 + 8` to carry out div by 7.

The current implementation is implemented in an targeting independent manner in TargetLowering which uses DAG structure. This works on only constant divisors since for variable divisors we can not do the summing of their digits. Since this trick only works for odd numbers, we need to right shift zeros from the even number to make it odd. We need to keep the zero digits count so that we can shift them back for even divisors.

Next we look for chunk size that can be used to satisfy the Summing Digits property `(1 << W) % Divisor == 1`. We search backward to find the largest chunk size possible so we do have fewer chunks and fewer additions. 

Next we have a lambda function that checks whether the target supports funnel shift(FSHR) instruction. If it supports it, like in X86/Aarch64 we use that instruction otherwise we will need to explicitly use OR(SHL, SRL) instruction sequence.

We need to shift the input also if we have shifted the divisor. 

Next there are two cases. 1. When best chunk width is half of the size of chunkwidth, we add the low and high half, if any carry left add it to the final sum. Then we check if the target supports the uaddo_carry instruction if it then uses it otherwise needs special handling to check the overflow and add the carry to sum. 

In second case or in the else part when chunk size is variable but smaller than half of VT, we iterate through the dividend, masking and shifting to extract
chunks of BestChunkWidth, summing them into a single register. 
```
for (unsigned I = 0; I < BitWidth - TrailingZeros; I += BestChunkWidth) {
      // If there were trailing zeros in the divisor, increase the shift amount.
      unsigned Shift = I + TrailingZeros;
      SDValue Chunk;
      if (Shift == 0)
        Chunk = LL;
      else if (Shift >= HBitWidth)
        Chunk = DAG.getNode(
            ISD::SRL, dl, HiLoVT, LH,
            DAG.getShiftAmountConstant(Shift - HBitWidth, HiLoVT, dl));
      else
        Chunk = GetFSHR(LL, LH, Shift);
      // If we're on the last chunk, we don't need an AND.
      if (I + BestChunkWidth < BitWidth - TrailingZeros)
        Chunk = DAG.getNode(ISD::AND, dl, HiLoVT, Chunk, Mask);
      if (!Sum)
        Sum = Chunk;
      else
        Sum = DAG.getNode(ISD::ADD, dl, HiLoVT, Sum, Chunk);
    }
```

For udiv instruction we need the Quotient. And for urem instruction we need a remainder.

We calculate the remainder using truncated divisor, rem = sum % divisor.

Calculation for urem is as easy as we can already calculate the remainder but since we shifted the divisor for even, we need to shift again. 

For division, we first shift the input dividend (both halves) for training zero if the even divisor also shifted earlier. 

Next we subtract the remainder from the shifted dividend, making it exactly divisible by the divisor.

To get the Quotient we use the mul instruction by using inverse multiplication.
Lets see what happens here in this call 
```
    APInt MulFactor = Divisor.multiplicativeInverse();
```
Multiplicative Inverse is to get the final quotient(Quotient = Dividend * MulFactor). We have the remainder already and this is a trick to
get it without the div instruction. 

Then we split quotient into two halves to store in the res vector and return that.

### X86 SelectionDAG codegen - 
With that the new code which is now emitted for x86 for div by 7 is -
```
div_by_7(unsigned __int128):            
                                        ; Mask creation
       movabs  rax, 1152921504606846975 ; 0x0FFFFFFFFFFFFFFF mask

                                        ; Extract low 60 bits
       mov     rdx, rdi                 ; rdx = low64(n)
       and     rdx, rax                 ; rdx = a = n & ((1<<60)-1)

                                        ; Extract middle 60 bits
       mov     r8, rdi                  ; r8 = low64(n)
       shrd    r8, rsi, 60              ; r8 = ( (rsi << 64) | rdi ) >> 60
       and     r8, rax                  ; rax = b = (n >> 60) & ((1<<60)-1)

                                        ; Extract top bits
       mov     rcx, rsi                 ; rcx = high64(n)
       shr     rcx, 56                  ; rcx = c = n >> 120 (64+56)  // top 8 bits

                                        ; Fold modulo 7
       add     rcx, rdx                 ; rcx = c + a
       add     rcx, r8                  ; rcx = x = a + b + c

                                        ; Approximate division
       movabs  rdx, 5270498306774157605 ; rdx = M = floor(2^64 / 7)
       mov     rax, rcx                 ; rax = x
       mul     rdx                      ; rdx = high64(x * M)
       shr     rdx                      ; rdx = q = approx(x / 7)

                                        ; Compute remainder
       lea     rax, [8*rdx]             ; rax = 8*q
       sub     rdx, rax                 ; rdx = q - 8*q = -7*q
       add     rdx, rcx                 ; rdx = r = x - 7*q

                                        ; Subtract remainder from original
       sub     rdi, rdx                 ; low64(n) -= r
       sbb     rsi, 0                   ; high64(n) -= borrow

                                        ; Final exact division
       movabs  rcx, -5270498306774157605 ; rcx = -M
       movabs  r8, 7905747460161236407  ; r8 = magic constant for exact division
       mov     rax, rdi                 ; rax = low64(n)
       mul     r8                       ; (rax * r8) → rdx:rax
       imul    rcx, rdi                 ; rcx = (-M) * low64(n)
       add     rdx, rcx                 ; rdx += rcx
       imul    r8, rsi                  ; r8 = magic * high64(n)
       add     rdx, r8                  ; rdx += r8
       ret                              ; return rdx
```
```
rdi = low 64 bits of n
rsi = high 64 bits of n
```
If the question you might have why Quotient calculation is happening two time? This is because x(a+b+c) is not exactly equal to n, it is approximate, the first stage is
only to recover the remainder (via a reduced value x), and the final division operates on the corrected original number n − r, where multiplicative division becomes exact. 

### Future work - 
GCC still has better codegen than LLVM for this routine with more refinements. 
This is only a small part of what could be possible, The Hacker Delight book is filled with so many similar tricks. 
I could see this week Craig Topper just put up a patch (https://github.com/llvm/llvm-project/pull/189286) to extend this algorithm to cover more constant divisors like 67 that satisfies `2^k mod divisor == -1`. This is similar to the trick we studies in school that a number is dvisible by 11 if we add the even digits and subtract the odd digits and if the resulting sum is divisible by 11 the original number is also divisible by 11. Though covering this type of integer requires extra subtraction of chunks, but it is still better than magic multiplication or generating libcalls and GCC still generates libcall for this. 

I have also ported the algorithm to GlobalIsel instruction selector (https://github.com/llvm/llvm-project/pull/189231/) which was lacking behind from SelectionDAG.

Note: I would like to thanks Craig Topper who reviewed the pull request and provide a lot of suggestion to improve. 


