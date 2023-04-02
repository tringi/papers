:arrow_backward: [C++ Proposals and ideas](README.md)

# C++ Implementation Proposal: x86-64 calling convention v2

TL;DR: Spill everything, pack what you can, don't skip any registers, and be done with it.

This is obviously NOT a suggestion to change ABI for interfacing with the OS.
That would not be feasible in any realistic form
(albeit AArch64 Windows does have [2 native ABIs](https://learn.microsoft.com/en-us/windows/arm/arm64ec)).

This is a suggestion for ABI used within the process and accompanied DLLs.
Just like C++ process on x86-32 is __cdecl, but is interfacing the OS via __stdcall.

## Rationale

After a mess of myriad x86-32 calling conventions, 
it is often presented as an important advantage and a definite fact,
that there is only one single calling convention on *"X64"*.
**It is not true.**

There are at least
[System V AMD64](https://en.wikipedia.org/wiki/X86_calling_conventions?useskin=vector#x86-64_calling_conventions),
[Microsoft's x64](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention?view=msvc-170)
and
[__vectorcall](https://learn.microsoft.com/en-us/cpp/cpp/vectorcall?view=msvc-170) conventions.
And then [[[trivial_abi]]](https://quuxplusone.github.io/blog/2018/05/02/trivial-abi-101/) exists on Clang.

### Additional reading

* Discusion on StackOverflow: [Why does Windows64 use a different calling convention from all other OSes on x86-64?](https://stackoverflow.com/questions/4429398/why-does-windows64-use-a-different-calling-convention-from-all-other-oses-on-x86)
* [Calling convention reference on Agner.org](https://www.agner.org/optimize/calling_conventions.pdf)

## Current issues

* TBD: optional, string_view, ...codebases still pass pointer and size

## Proposed calling convention semantics

Non-skipping
* Parameter registers are progressively allocated, not assigned per parameter index.

Spilling
* Structures passed as argument are spilled into registers

Packing
* All structures smaller or equal to a register size are passed in single register.  
  This can be achieved by a single MOV instruction.
* Structures containing multiple float/double types are packed into XMM registers, if they fit.
* Potential issue: Loading 6B at the end of page, causing possible access violation fault. Restrict to power of 2?
* Potential issue: Mov 6B to register movs 8B. Insecure. Possible data leaking. Require extra masking?

## Examples

```cpp
struct Pt {
    int x;
    int y;
};
struct Complex {
    float r;
    float i;
};
struct Data {
    int a;
    float b;
    Pt pt;
    __m128 c;
};
struct Version {
    std::uint8_t  major;
    std::uint8_t  minor;
    std::uint16_t patch;
    std::uint32_t build;
};

void CONVENTIONNAME function (int z, Data s, float f, Complex g, Version v);
```

Assignment

Parameter | Register
-|-
z | rcx
s.a | rdx
s.b | xmm0
s.pt | r8
s.c | xmm1
f | xmm2
g | xmm3
v | r9

## Possible names

* __spillcall, __packcall, __spcall, ...

## Limitations

* working within current boundaries of Microsoft Visual C++ we need to fit within:
   * RCX, RDX, R8, R9, R10, R11 are considered volatile (caller-saved)

## Possible extensions

??? If the arguments don't fit the available registers, the function has no right to compile.
