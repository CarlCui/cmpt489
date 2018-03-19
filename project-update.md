# ShuffleVector 

## Transform shufflevector to other LLVM instruction

One important approach is to transform shufflevector instruction to other LLVM instruction. It could benefit all different backend once you done it LLVM IR code. We found a previous project when we were doing initial searching, and we found that project has two main issues:

1. Adding new patterns is not easy
2. It doesn’t implement some basic pattern

```
Merge Pattern
Arg1:  [1,2,3,4]
Arg2:  [5,6,7,8]
Result:[1,5,2,6,3,7,4,8]

Assume each element is 4 bits
Arg1:[0001, 0010, 0011, 0100] -> [0000 0001, 0000 0010, 0000 0011, 0000 0100] ; extend
                              -> [0001 0000, 0010 0000, 0011 0000, 0100 0000] ; shift right
Arg2:[0101, 0110, 0111, 1000] -> [0000 0101, 0000 0110, 0000 0111, 0000 1000] ; extend
                              -> [0001 0101, 0010 0110, 0011 0111, 0100 1000] ; Arg1 OR Arg2
                              -> [0001, 0101, 0010, 0110, 0011, 0111, 0100, 1000] ; split

Pack Pattern
Arg1:  [1,2,3,4]
Arg2:  [5,6,7,8]
Result:[1,3,5,7]

Assume each element is 4 bits
Arg1:[0001, 0010, 0011, 0100] -> [0001, 0000, 0011, 0000] ; AND with a contant
Arg2:[0101, 0110, 0111, 1000] -> [0101, 0000, 0111, 0000] ; AND with a contant
                              -> [0000, 0101, 0000, 0111] ; shift right (rotation)
                              -> [0001, 0101, 0011, 0111] ; Arg1 OR Arg2
```

We also tried to detect more patterns. But analyze IR code manually is tedious and not effective.

## MMX Instructions

Here is a test file. We define it as a function to prevent from being optimized by LLVM. This is a two-lanes rotation pattern. 

```llvm
define <8 x i32> @mysfl(<4 x i32> %a, <4 x i32>%b)
{
  %t = shufflevector <4 x i32> %a, <4 x i32> %b, <8 x i32> <i32 0, i32 4, i32 1, i32 5, i32 2, i32 6, i32 3, i32 7>
  ret <8 x i32> %t
}
```

Actually, there is already an instruction doing this on my laptop. So, transform it may not help a lot.

```asm
	.text
	.file	"hello.ll"
	.globl	mysfl
	.align	16, 0x90
	.type	mysfl,@function
mysfl:                                  # @mysfl
	.cfi_startproc
# BB#0:
	movdqa	%xmm0, %xmm2
	punpckldq	%xmm1, %xmm0    # xmm0 = xmm0[0],xmm1[0],xmm0[1],xmm1[1]
	punpckhdq	%xmm1, %xmm2    # xmm2 = xmm2[2],xmm1[2],xmm2[3],xmm1[3]
	movdqa	%xmm2, %xmm1
	retq
.Lfunc_end0:
	.size	mysfl, .Lfunc_end0-mysfl
	.cfi_endproc


	.section	".note.GNU-stack","",@progbits

```

## Speeding Up Reconginition Process

We have thought about using SIMD instructions when writing the pattern recognition pass. Since pattern matching deals with potentially any element in the vectors, using SIMD instructions will natually speed up the process by theroy. However, we have since then encountered some difficulties in going down this path.

## Introducing SIMD Instructions into the Pass

We have considered three ways of writing SIMD instructions in the pass:

1. Use vector extensions
1. Write intrinsics
1. Use a library in parabix

### Use vector extensions

SIMD instructions are low-level instructions, but we write the pass in high-level code. Ideally, these ideas are absracted just like in LLVM IR. However, through research, we have not found any that is powerful enough to support horizontal operations, which are crucial to our project.

For example, we looked at the [GCC vector extensions](https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html), and it has done a very good job abstracting vector length and width, just like LLVM IR. Besides, you can use operators like `+`, and `*` to operate on these vectors.

LLVM also supports other [extended vectors](https://clang.llvm.org/docs/LanguageExtensions.html#id5) like OpenCL and AltiVec, but none of them is really sufficient.

### Write intrinsics

Another way of using SIMD instructions is to just write intrinsics. However, although this is the simplest and most direct solution, there are quite a few downsides to it:

1. Intrinsics are not portable, which means that we need to write different code for different SIMD instruction sets.
1. Writing code in intrinsics is not fun (See code snippet below).
1. You really need to learn those instruction sets to write efficient algorithms. For example, AVX2 and AVX512 added a lot of new instructions that many are very specific, which means that we need to investigate into all instructions in order to choose the best way of writing the algorithm.

```cpp
auto mask = inst->getShuffleMask();

    auto maskSize = mask.size();

    uint8_t maskData[16] = {}; // 16 x i8, zero initialized
    uint8_t indeces[16] = {}; // an array of indeces ([0, 1, 2, ...])
    uint8_t lArr[16] = {};
    uint8_t lengthMaskArr[16] = {};

    for (unsigned i = 0; i < maskSize; i ++) {
        maskData[i] = (uint8_t)mask[i];
        indeces[i] = i;
        lArr[i] = maskSize;
        lengthMaskArr[i] = 0xFF;
    }

    __m128i maskVector;
    __m128i indexVector;
    __m128i constantLengthVector; // array of lengths
    __m128i lengthMaskVector; //

    maskVector = _mm_load_si128((__m128i*)&maskData);
    indexVector = _mm_load_si128((__m128i*)&indeces);
    constantLengthVector = _mm_load_si128((__m128i*)&lArr);
    lengthMaskVector = _mm_load_si128((__m128i*)&lengthMaskArr);

    __m128i diff = _mm_sub_epi8(indexVector, maskVector);
    __m128i modded = _mm_add_epi8(diff, constantLengthVector);
    __m128i cmpMask = _mm_cmpgt_epi8(diff, _mm_setzero_si128());
    __m128i result = _mm_blendv_epi8(modded, diff, cmpMask);

    int firstElement = _mm_extract_epi8(result, 0);

    __m128i constantFirst = _mm_set1_epi8((char)firstElement);
    __m128i constantFirstTillLength = _mm_blendv_epi8(_mm_setzero_si128(), constantFirst, lengthMaskVector);
    __m128i allEqual = _mm_sub_epi8(result, constantFirstTillLength);

    int allEqualResult = _mm_testz_si128(allEqual, _mm_setzero_si128());

    return allEqualResult == 0;
```

### Use SIMD library in parabix

Parabix contains a mature [library](http://parabix.costar.sfu.ca/browser/trunk/lib) that abstracts away the need of using intrinsics directly. Currently this is the most ideal way of approaching this problem, and we are exploring this possibility by looking at:

1. How to include this library into our source code
1. How to use this library

