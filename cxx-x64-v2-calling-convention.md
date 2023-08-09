:arrow_backward: [C++ Proposals and ideas](README.md)

# C++ Implementation Proposal: x86-64 calling convention v2

TL;DR: Spill everything, pack what you can, don't skip any registers, and be done with it.

This is obviously NOT a suggestion to change ABI for interfacing with the OS.
That would not be feasible in any realistic form
(albeit AArch64 Windows does have [2 native ABIs](https://learn.microsoft.com/en-us/windows/arm/arm64ec)).

This is a suggestion for ABI used within the process and accompanied DLLs.  
Just like C++ process on x86-32 is [__cdecl](https://learn.microsoft.com/en-us/cpp/cpp/cdecl),
but is interfacing the OS via [__stdcall](https://learn.microsoft.com/en-us/cpp/cpp/stdcall).

## Rationale

Large production C++ codebases still resist modernization efforts,
such as replacing pointer & size parameters with `std::span` or `std::string_view`,
or nullable pointers with `std::optional` or `std::unique_ptr`,
for the reason that doing so is significant measurable pessimisation.

The culprit is Windows' X64 calling convention, which mandates those utilities are passed via pointer,
instead of spilled into registers.
This introduces complex constructions at the call site and, primarily, unnecessary indirection inside the callee.

### The simplicity argument

After a mess of myriad x86-32 calling conventions, it is often presented as an important advantage and a fact,
that there is only one single calling convention on *"X64"*. **That is not true.**

There are at least
[System V AMD64](https://en.wikipedia.org/wiki/X86_calling_conventions?useskin=vector#x86-64_calling_conventions),
[Microsoft's x64](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention?view=msvc-170)
and
[__vectorcall](https://learn.microsoft.com/en-us/cpp/cpp/vectorcall?view=msvc-170) conventions.
And then [[[trivial_abi]]](https://quuxplusone.github.io/blog/2018/05/02/trivial-abi-101/) exists on Clang.

## Proposed calling convention semantics

Non-skipping
* Parameter registers are progressively allocated, not assigned by argument index.

Spilling
* Structures, larger than a register, passed as arguments are spilled into registers

Packing
* All structures smaller or equal to a register size are passed in single register.  
  This can be achieved by a single MOV instruction.
  * Potential issues with structures of non-power of 2 size:
    * Loading 6B at the end of page, causing possible access violation fault. Restrict to power of 2?
    * Mov 6B to a register moves 8B. Insecure. Possible data leaking. Require extra masking?
* While 8B heterogeneous `struct { int i; float f; }` is passed in a single register,
  larger ones are spilled per-element. Someone will eventually need to unpack those before use,
  so why not do that at call site.
* Structures containing multiple float/double types are packed into as many XMM registers as possible.
  * YMM/ZMM?

Lifetime
* Function arguments' destructors are invoked by the callee.
  * This allows for better elision and improves code size.
  * Reference: [CppCon 2019: Chandler Carruth “There Are No Zero-cost Abstractions”](https://www.youtube.com/watch?v=rHIkrotSwcc)

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

* Working within current boundaries of Microsoft Visual C++ we need to fit within:
   * RCX, RDX, R8, R9, R10, R11, and XMM0 to XMM5 are considered volatile (caller-saved)
* If the arguments don't fit the available registers, the calling convention reverts to the original one.

## Return values

* The exactly same mechanism is used to pack and spill return value into RAX, R10, R11, XMM4 and XMM5 registers.
  * Microsoft's ABI considers those volatile on return

### Additional reading

* Discusion on StackOverflow: [Why does Windows64 use a different calling convention from all other OSes on x86-64?](https://stackoverflow.com/questions/4429398/why-does-windows64-use-a-different-calling-convention-from-all-other-oses-on-x86)
* [Calling convention reference on Agner.org](https://www.agner.org/optimize/calling_conventions.pdf)

