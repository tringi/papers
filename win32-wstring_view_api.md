:arrow_backward: [Win32 Proposals and ideas](README.md)

# Extend APIs that consume NUL-terminated strings to support (w)string_view

*After native UTF-8 support was finally added to Windows API a few years back,
it's way overdue for it to be extended to support non-NUL-terminated strings.
It's all converted to UNICODE_STRING inside anyway.*

## The problem

1. Modern C++ and other languages provide extensive facilities to efficiently work with strings,
especially with substrings, that aren't NUL-terminated.

2. The vast majority of Windows API function calls,
e.g. [CreateFile](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew),
consume their string argument(s) via NUL-terminated strings,
forcing the user to allocate a NUL-terminated copy of the string.

3. These API functions convert those parameters into
[UNICODE_STRING](https://learn.microsoft.com/en-us/windows/win32/api/subauth/ns-subauth-unicode_string)s
before passing it to appropiate NT API(s). UNICODE_STRING is a potentially-owning limited-size wstring_view.

This is unnecessary complex, performance pesimisation, and effectively redundant allocation and data copying.

## A solution

As various sets of API functions eventually call one common function to initialize UNICODE_STRING, e.g.
[RtlInitUnicodeString](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-rtlinitunicodestring)
or `RtlDosPathNameToNtPathName_U` that's used by file APIs, it makes sense to:

Propagate tagged pointer down to those routines, and unpack wstring_view directly to UNICODE_STRING.

```cpp
static inline LPCWSTR W64PassStringView (const std::wstring_view & s) {
    return (LPCWSTR) (0x4000000000000000uLL | ((DWORD_PTR) &s));
}

std::wstring_view s (...);
CreateFile (W64PassStringView (s), ...);
```

The extension of coversion routines is obvious:

```cpp
VOID RtlInitUnicodeString (PUNICODE_STRING DestinationString, PCWSTR SourceString) {
    if (0x4000000000000000uLL & (DWORD_PTR) SourceString) {

        auto view = (ABI_WSTRING_VIEW_TYPE *) ((DWORD_PTR) SourceString & 0x00FFFFFFFFFFFFFFuLL);

        /* TBD: check view->Length is in range */

        DestinationString->Buffer = view->Buffer;
        DestinationString->Length = 2 * view->Length;
        DestinationString->MaximumLength = 2 * view->Length;
        }
    else {
        /* original code */
    }
}

```

### Notes

* **32-bit** - Yes, I'm ignoring 32-bit code.
There won't be any new 32-bit Windows as Microsoft's people have
[openly acknowledged](https://twitter.com/JosephBialek/status/1581751766793981953)
on social networks that 32-bit code is being removed from Windows kernel.

**LA57** - Yes, on current era of x86-64 CPUs, it's possible to encode `USHORT Length` into upper 2 bytes of the pointer,
and doing so skipping one indirection.
But as of 2023, new server CPUs featuring [5-level paging](https://en.wikipedia.org/wiki/Intel_5-level_paging) are expected
to appear on the market. Those support 57-bit pointers, leaving only 7 bits (8 in user mode) for pointer tagging.
Which is clearly not enough.

## Alternative solutions

### Add whole set of new APIs

```cpp
LPCWSTR WINAPI CreateFile3 (SIZE_T len, LPCWSTR ptr, ...);
```

Obviously not feasible.

**Remark:** With CreateFile(2) additional approaches are actually possible, but we'll ignore tham, as they don't apply to other APIs.

## TODO

*
