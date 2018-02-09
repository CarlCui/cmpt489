# Shufflevector Pattern Recognition

## Introduction

Shufflevector instruction of LLVM IR provides powerful expressiveness. However, with such expressiveness, it is hard to write a generalized and efficient algorithm for the backend. Meanwhile, in many cases, a shufflevector instruction can be replaced with another instrution that is efficiently implemented by the backend. In this project, we are aiming to tackle this problem of recognizing these cases and translate shufflevector to more efficient instructions by creating an optimized engine with a systematic approach. We also want to adopt lane detection when doing pattern recognition.

## Patterns

In the [project page](http://parabix.costar.sfu.ca/wiki/ShufflePatternLibrary#StandardPatternExamples), there are some patterns identified already. We firstly want to start off by implementing most of them. For example, a rotate operation can be expressed as

```llvm
shufflevector <4 x i32> %x, <4 x i32> undef, <4 x i32> <i32 3, i32 0, i32 1, i32 2>
```

## Systematic Approach

Taking the idea of kernels, we want to build a system where a case can be easily added or removed. Also, since these cases are completely dependent from each other, we can take advantage of multi-threading if efficiency becomes an issue.

```C++
class PatternRecognition {
private:
    Pattern *patterns;

public:
    bool tryMatch(Instruction *instr);

};

class Pattern {
public:
    bool matchAndOptimize(Instruction *instr);
}

...

// in the pass:

for (auto& instruction : basicBlock) {
    if (instruction is shuffleVector) {
        PatternRecognition patternRecognition;

        patternRecognition.tryMatch(instruction);
    }
}
```

## Lane Detection

Sometimes, patterns might appear in lanes of a vector:

```llvm
shufflevector <8 x i32> %x, <8 x i32> undef, <8 x i32> <i32 3, i32 0, i32 1, i32 2, i32 7, i32 4, i32 5, i32 6>
```

In this example, the instruction is essentially performing two rotate operations on two lanes.

We want to incorporate this idea into all pattern recognitions.

## Example 

Examples are taken from The ShuffleVector Project wiki page on Parabix website.

Sometimes, the shuffle mask pattern for a shuffle vector could be just a byte swap:

```llvm
%v3 = shufflevector <8 x i8> %v1, <8 x i8> undef,
                    <8 x i32> <i32 1, i32 0, i32 3, i32 2, i32 5, i32 4, i32 7, i32 6>  ; yields <8 x i8>
```

This could be transformed to 

```llvm
%t0 = bitcast %v1 to i64 @llvm.bswap.i64(i64 %t0)
```

Here, `llvm.bswap.i64` is an intrinsic that returns an i64 value that has the high and low byte of the input swapped. 

```llvm
%v3 = shufflevector <8 x i4> %v1, <8 x i4> undef,
              <8 x i32> <i32 1, i32 2, i32 3, i32 0, i32 5, i32 6, i32 7, i32 4>  ; yields <8 x i8>
```

Shuffles on 4-bit fields are generally not supported by SIMD instruction sets, but this one can be implemented by transforming to 16-bit vector shift operations. 

```llvm
%t0 = bitcast %v1 to <2 x i16>
%t1 = shl %t0, <2 x i16> <i16 12, i16 12> 
%t2 = lshr %t0, <2 x i16> <i16 4, i16 4> 
%v3 = xor %t1, %t2
```

In this case, we use shift instructions to optimize the performance. 


## Optimize Pass with SIMD Technology

As we all know, compilation efficiency matters, and C++ is known for its slow compilation. Thus, we want to create this pass that could potentially use SIMD operations to achieve high efficiency by using llvm `SmallVector`s when performing pattern recognitions.


# Resources

1. Parabix shufflevector project introduction: http://parabix.costar.sfu.ca/wiki/ShuffleVector
2. Parabix shufflevector pattern recognition library introduction:  http://parabix.costar.sfu.ca/wiki/ShufflePatternLibrary#StandardPatternExamples
3. LLVM shufflevector document: http://llvm.org/docs/LangRef.html#shufflevector-instruction
4. LLVM ShuffleVectorInst class reference: http://llvm.org/doxygen/classllvm_1_1ShuffleVectorInst.html
5. LLVM ShuffleVectorInst class source code: http://llvm.org/doxygen/Instructions_8cpp_source.html
6. Project from previous students: https://github.com/laishzh/LLVM_ShuffleVector_Optimizer

