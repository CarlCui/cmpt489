# ShuffleVectorPass update 3

## Previously

* Adopted IDISA library to write SIMD algorithm inside the pass
* Used steady clock inside chrono library to measure the running time, which was inaccurate

## Refined way of measuring speedup

Intel provides an instruction called [rdtsc](https://en.wikipedia.org/wiki/Time_Stamp_Counter). It counts CPU cycles since last reset. So we just need to make a call at the beginning of the algorithm, and make another one at the end, then subtract the values to get the result. One thing to notice is that `rdtsc` itself also costs some CPU cycles, so we call it several times ahead of measurement to calculate the cost of it. Below is a quite straightforward way of doing that:

```cpp
void countRdtscUsedCycles() {
    uint64_t a1 = rdtsc();
    uint64_t a2 = rdtsc();
    uint64_t a3 = rdtsc();
    uint64_t a4 = rdtsc();
    uint64_t a5 = rdtsc();
    uint64_t a6 = rdtsc();

    errs() << (a2 - a1) << "\n";
    errs() << (a3 - a2) << "\n";
    errs() << (a4 - a3) << "\n";
    errs() << (a5 - a4) << "\n";
    errs() << (a6 - a5) << "\n";
}
```

In the detection function, we do the measurement like this:

```cpp
auto begin = rdtsc();

BitBlock diff = simd<fw>::sub(indexVector, maskVector);
BitBlock modded = simd<fw>::add(diff, lengthVector);
BitBlock cmpMask = simd<fw>::gt(diff, zeroVector);
BitBlock result = simd<fw>::ifh(cmpMask, diff, modded);

int firstElement = mvmd<fw>::extract<0>(result);

BitBlock constantFirst = mvmd<fw>::fill(firstElement);
BitBlock constantFirstTillLength = simd<fw>::ifh(lengthMaskVector, constantFirst, zeroVector);
BitBlock allEqual = simd<fw>::sub(result, constantFirstTillLength);

bool allEqualResult  = bitblock::any(allEqual);

auto end = rdtsc();
errs() << "cycles = " << (end - begin) << "\n";
```

Finally, when we get the result, we subtract that from the cost of `rdtsc` to get the final measurement.

Example output:

```
22
27
22
76
23
cycles = 300
```

So the actual cost of this algorithm is ~278 cycles.

## Metadata infrastructure

Remember the goal is to let other passes or backend optimization benefit from the result of this pattern recognition process.

In the original function declaration for `matches()`, it returns a boolean value so that it is easy to use in the pass. Now, we change the return type to our `PatternMetadata` object pointer. If the function does not recognize this particular pattern, it just returns `NULL`, so it is still easy to use in the pass.

Original function declaration:

```cpp
virtual bool matches(ShuffleVectorInst *inst, CommonVectors commonVectors) = 0;
```

Current function declaration:

```cpp
virtual PatternMetadata *matches(ShuffleVectorInst *inst, CommonVectors commonVectors) = 0;
```

Then in the pass, if the function returns a `PatternMetadata` object, we firstly try to do optimization. If no optimization can be done for that pattern, we simply attach the metadata to the instruction.

```cpp
if (pm != NULL) {
    if (!optimizer.tryOptimize(inst, pm)) {
        inst->setMetadata(PATTERN_METADATA_KIND_ID, pm->asMDNode(inst->getContext()));
    }
}
```

Output with attached metadata:

```llvm
; ModuleID = 'prog2.s'
source_filename = "prog2.s"

define i32 @main() {
  %1 = alloca <4 x i32>
  %2 = load <4 x i32>, <4 x i32>* %1
  %a161 = alloca <16 x i32>
  %a162 = load <16 x i32>, <16 x i32>* %a161
  %3 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 0, i32 1, i32 2, i32 3>, !shuffleVector.pattern !0
  %4 = shufflevector <16 x i32> %a162, <16 x i32> undef, <16 x i32> <i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7, i32 8, i32 9, i32 10, i32 11, i32 12, i32 13, i32 14, i32 15, i32 0>, !shuffleVector.pattern !1
  %5 = shufflevector <16 x i32> %a162, <16 x i32> undef, <16 x i32> <i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1, i32 1>, !shuffleVector.pattern !2
  %aaa = load <4 x i32>, <4 x i32>* %1
  %6 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 2, i32 3, i32 0, i32 1>, !shuffleVector.pattern !3
  %7 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 2, i32 3, i32 0, i32 1>, !shuffleVector.pattern !3
  %8 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 2, i32 3, i32 0, i32 1>, !shuffleVector.pattern !3
  %9 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 2, i32 3, i32 0, i32 1>, !shuffleVector.pattern !3
  %10 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 2, i32 3, i32 0, i32 1>, !shuffleVector.pattern !3
  %11 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 2, i32 3, i32 0, i32 1>, !shuffleVector.pattern !3
  %12 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 1, i32 2, i32 3, i32 0>, !shuffleVector.pattern !4
  %13 = shufflevector <4 x i32> %2, <4 x i32> undef, <4 x i32> <i32 3, i32 0, i32 1, i32 2>, !shuffleVector.pattern !5
  %14 = shufflevector <4 x i32> %2, <4 x i32> %2, <4 x i32> <i32 0, i32 5, i32 2, i32 3>, !shuffleVector.pattern !6
  %15 = shufflevector <4 x i32> %2, <4 x i32> %2, <4 x i32> <i32 0, i32 5, i32 6, i32 7>, !shuffleVector.pattern !6
  %16 = shufflevector <4 x i32> %2, <4 x i32> %2, <4 x i32> <i32 0, i32 2, i32 6, i32 7>
  ret i32 0
}

!0 = !{i32 2}
!1 = !{i32 0, i32 15}
!2 = !{i32 1, i32 1}
!3 = !{i32 0, i32 2}
!4 = !{i32 0, i32 3}
!5 = !{i32 0, i32 1}
!6 = !{i32 4}

```

## Case Study: Optimize Merge Pattern

Now let's look at an example about optimizing LLVM IR code. We takes merge pattern as an example because it's a little complexed and there is no previous works on it.

```llvm
define <8 x i32> @merge(<4 x i32> %a, <4 x i32> %b) {
entry:
  %t = shufflevector <4 x i32> %a, <4 x i32> %b, <8 x i32> <i32 0, i32 4, i32 1, i32 5, i32 2, i32 6, i32 3, i32 7>
  ret <8 x i32> %t
}
```

We try to replace original `shufflevector` instruction with other operations like shift and bit operations, as the below shows.  

```llvm
define <8 x i32> @merge(<4 x i32> %a, <4 x i32> %b) {
entry:
    %0 = zext <4 x i32> %a to <4 x i64>
    %1 = zext <4 x i32> %b to <4 x i64>
    %2 = shl <4 x i64> %0, <i64 32, i64 32, i64 32, i64 32> 
    %3 = or <4 x i64> %1, %2
    %4 = bitcast <4 x i64> %3 to <8 x i32>
    ret <8 x i32> %4
}
```

However, it's difficult to find difference on various aspects like time overhead, CPU cycles, number of instructions etc. As for the reason, on the one hand, changes in performance is insignificant for only one function. On the other hand, our optimization may not be that effective. One work around could be, directly look at assembly code. 

```asm
merge:                                  # @merge
	.cfi_startproc
# BB#0:                                 # %entry
	movdqa	%xmm0, %xmm2
	punpckldq	%xmm1, %xmm0    # xmm0 = xmm0[0],xmm1[0],xmm0[1],xmm1[1]
	punpckhdq	%xmm1, %xmm2    # xmm2 = xmm2[2],xmm1[2],xmm2[3],xmm1[3]
	movdqa	%xmm2, %xmm1
	retq
.Lfunc_end0:
	.size	merge, .Lfunc_end0-merge
	.cfi_endproc
```

Actually, the "optimized" version has more instructions. The reason is there is already some pack/unpack instructions that doing this quite well. 

```asm
merge:                                  # @merge
	.cfi_startproc
# BB#0:                                 # %entry
	xorps	%xmm2, %xmm2
	movaps	%xmm0, %xmm3
	punpckldq	%xmm2, %xmm3    # xmm3 = xmm3[0],xmm2[0],xmm3[1],xmm2[1]
	punpckhdq	%xmm2, %xmm0    # xmm0 = xmm0[2],xmm2[2],xmm0[3],xmm2[3]
	movaps	%xmm1, %xmm4
	punpckhdq	%xmm2, %xmm4    # xmm4 = xmm4[2],xmm2[2],xmm4[3],xmm2[3]
	punpckldq	%xmm2, %xmm1    # xmm1 = xmm1[0],xmm2[0],xmm1[1],xmm2[1]
	psllq	$32, %xmm0
	psllq	$32, %xmm3
	por	%xmm3, %xmm1
	por	%xmm0, %xmm4
	movaps	%xmm1, %xmm0
	movaps	%xmm4, %xmm1
	retq
.Lfunc_end0:
	.size	merge, .Lfunc_end0-merge
	.cfi_endproc
```

But then we tried to change vector type to `i5` which is usually not directly supported by most architecture and instructions. Hopefully, the results may be different. 

```llvm
define <8 x i5> @merge(<4 x i5> %a, <4 x i5> %b) {
entry:
  %t = shufflevector <4 x i5> %a, <4 x i5> %b, <8 x i32> <i32 0, i32 4, i32 1, i32 5, i32 2, i32 6, i32 3, i32 7>
  ret <8 x i5> %t
}
```

The amount of instructions increase rapidly, as we expected. 

```asm
merge:                                  # @merge
	.cfi_startproc
# BB#0:                                 # %entry
	pshuflw	$232, %xmm1, %xmm1      # xmm1 = xmm1[0,2,2,3,4,5,6,7]
	pshufhw	$232, %xmm1, %xmm1      # xmm1 = xmm1[0,1,2,3,4,6,6,7]
	pshufd	$232, %xmm1, %xmm1      # xmm1 = xmm1[0,2,2,3]
	pshuflw	$232, %xmm0, %xmm0      # xmm0 = xmm0[0,2,2,3,4,5,6,7]
	pshufhw	$232, %xmm0, %xmm0      # xmm0 = xmm0[0,1,2,3,4,6,6,7]
	pshufd	$232, %xmm0, %xmm0      # xmm0 = xmm0[0,2,2,3]
	punpcklwd	%xmm1, %xmm0    # xmm0 = xmm0[0],xmm1[0],xmm0[1],xmm1[1],xmm0[2],xmm1[2],xmm0[3],xmm1[3]
	retq
.Lfunc_end0:
	.size	merge, .Lfunc_end0-merge
	.cfi_endproc
```


```asm
merge:                                  # @merge
	.cfi_startproc
# BB#0:                                 # %entry
	movdqa	.LCPI0_0(%rip), %xmm2   # xmm2 = [31,31,31,31]
	pand	%xmm2, %xmm0
	pand	%xmm2, %xmm1
	pslld	$5, %xmm0
	por	%xmm1, %xmm0
	movd	%xmm0, %eax
	andl	$1023, %eax             # imm = 0x3FF
	movw	%ax, -8(%rsp)
	pshufd	$231, %xmm0, %xmm1      # xmm1 = xmm0[3,1,2,3]
	movd	%xmm1, %eax
	andl	$1023, %eax             # imm = 0x3FF
	movw	%ax, -2(%rsp)
	pshufd	$78, %xmm0, %xmm1       # xmm1 = xmm0[2,3,0,1]
	movd	%xmm1, %eax
	andl	$1023, %eax             # imm = 0x3FF
	movw	%ax, -4(%rsp)
	pshufd	$229, %xmm0, %xmm0      # xmm0 = xmm0[1,1,2,3]
	movd	%xmm0, %eax
	andl	$1023, %eax             # imm = 0x3FF
	movw	%ax, -6(%rsp)
	movl	-8(%rsp), %eax
	movq	%rax, %rcx
	shrq	$35, %rcx
	andl	$31, %ecx
	movd	%ecx, %xmm0
	movl	%eax, %ecx
	shrl	$15, %ecx
	andl	$31, %ecx
	movd	%ecx, %xmm1
	punpcklwd	%xmm0, %xmm1    # xmm1 = xmm1[0],xmm0[0],xmm1[1],xmm0[1],xmm1[2],xmm0[2],xmm1[3],xmm0[3]
	movl	%eax, %ecx
	shrl	$25, %ecx
	andl	$31, %ecx
	movd	%ecx, %xmm0
	movl	%eax, %ecx
	shrl	$5, %ecx
	andl	$31, %ecx
	movd	%ecx, %xmm2
	punpcklwd	%xmm0, %xmm2    # xmm2 = xmm2[0],xmm0[0],xmm2[1],xmm0[1],xmm2[2],xmm0[2],xmm2[3],xmm0[3]
	punpcklwd	%xmm1, %xmm2    # xmm2 = xmm2[0],xmm1[0],xmm2[1],xmm1[1],xmm2[2],xmm1[2],xmm2[3],xmm1[3]
	movq	%rax, %rcx
	shrq	$30, %rcx
	andl	$31, %ecx
	movd	%ecx, %xmm0
	movl	%eax, %ecx
	shrl	$10, %ecx
	andl	$31, %ecx
	movd	%ecx, %xmm1
	punpcklwd	%xmm0, %xmm1    # xmm1 = xmm1[0],xmm0[0],xmm1[1],xmm0[1],xmm1[2],xmm0[2],xmm1[3],xmm0[3]
	movl	%eax, %ecx
	andl	$31, %ecx
	movd	%ecx, %xmm0
	shrl	$20, %eax
	andl	$31, %eax
	movd	%eax, %xmm3
	punpcklwd	%xmm3, %xmm0    # xmm0 = xmm0[0],xmm3[0],xmm0[1],xmm3[1],xmm0[2],xmm3[2],xmm0[3],xmm3[3]
	punpcklwd	%xmm1, %xmm0    # xmm0 = xmm0[0],xmm1[0],xmm0[1],xmm1[1],xmm0[2],xmm1[2],xmm0[3],xmm1[3]
	punpcklwd	%xmm2, %xmm0    # xmm0 = xmm0[0],xmm2[0],xmm0[1],xmm2[1],xmm0[2],xmm2[2],xmm0[3],xmm2[3]
	retq
.Lfunc_end0:
	.size	merge, .Lfunc_end0-merge
	.cfi_endproc
```

