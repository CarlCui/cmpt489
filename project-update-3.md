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
