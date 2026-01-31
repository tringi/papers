# Windows OS improvement recipes

* [DWM: ClearType on composited/translucent surfaces](win32-composited-cleartype.md)  
  *Defer text rendering to be done by DWM at composition time.  
   DwmSetDeferredText (HWND, LONG_PTR, RECT, HFONT, ..., LPCWSTR text, int cch);*

* [Win32 API: Support (w)string_view in place of LPCWSTR](win32-wstring_view_api.md)  
  *Tagged pointer to `wstring_view` in place of LPCWSTR parameter,
   unwrapped internally directly into UNICODE_STRING that NT APIs consume.*

* [Start Menu: Uninstall command for Win32](win32-uninstall-from-start.md)  
  *Introduce new IShellLinkDataList block or extended IShellLink2 to embed uninstaller path or uninstall registry
   key into .lnk file so that Start Menu can directly run it, and bring consistency with Store/Moden apps*

* Add current thread's [NUMA node number](https://learn.microsoft.com/en-us/windows/win32/procthread/numa-support)
  to [TEB](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block) to improve performance of fast allocators
  and caches.  
  Updated on context switch.
  It will reduce to a single `mov` the following routine, which contains extra syscall and an expensive loop inside
  [GetNumaProcessorNodeEx](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getnumaprocessornodeex):

  ```cpp
  USHORT GetCurrentProcessorNumaNode () {
      USHORT numa;
      PROCESSOR_NUMBER processor;
      GetCurrentProcessorNumberEx (&processor);
      GetNumaProcessorNodeEx (&processor, &numa);
      return numa;
  }
  ```

* `SetThreadExecutionState (..., ES_DISPLAY_REQUIRED)` should apply only to thread's
  [Desktop](https://learn.microsoft.com/en-us/windows/win32/winstation/window-stations-and-desktops).  
  *This will prevent browser playing a video on a **locked** computer (e.g. user is listening to music)
   from keeping the screens on, when there's nothing of value being displayed.*

* `DeleteMultipleFiles` API to request deletion of more than single file with one syscall
  to improve deletion performance of large directories.

## The state of Windows

* [The abysmal state of Taskbar icons of Windows 11](windows-state-of-icons.md)  
  *Making all the wrong choices when rendering taskbar icons.*

* ClearType subpixel antialiasing disappearing all over the OS
  *The reason being it leaves artifacts on composited surfaces.
  Fix suggested [here]([DWM: ClearType on composited/translucent surfaces](win32-composited-cleartype.md)).*

## Random fixes/additions that would be nice:

* 10-bit support in GDI
* Incorrect window menu item highlighted when there's no menubar
* `CreateFiber (..., FIBER_FLAG_AVX_SWITCH)`
* SetTimerEx (..., LPVOID context);
* TlsAllocEx (lpCallback);

# C++ papers &amp; proposals

* [C++ Destructive Move](cxx-destructive-move.md) - quick and dirty proposal draft

  ```cpp
  struct A {
      ~A (A & a) noexcept { ... }
      ~A () noexcept -> A { return A{ ... }; }
  };
  ```

* [Fast x86-64 calling convention for C++](cxx-x64-v2-calling-convention.md) **Work In Progress**  
  *Fully utilize registers, pack smaller structures, spill larger structures.
   Keep current ABI when interfacing the OS.*  
  Calling convention for modern era.

* Lambdas have access to static objects in their encompassing scope  
  *Because they are, well, global.*

  ```cpp
  if (lParam) {
      static WPARAM wParam = 0;
      static LRESULT result = 0;
      
      wParam = GetWParam (...);
      
      EnumThreadWindows (GetCurrentThreadId (),
                         [] (HWND hWnd, LPARAM lParam) {
                             
                             // 'result' and 'wParam' is static, thus accessible!
                             result = PostMessage (hWnd, m, wParam, lParam);
                             
                             return TRUE;
                         }, lParam);
  }
  ```

* Fix `operator[]` by adding assignment-only version  
  *Has priority in overload resolution for assignment when present.*

  ```cpp
  template <typename Key, typename Value>
  struct Map {
      Value && operator[] (Key && k, Value && v) {
          return this->insert_or_assign (k, v).first->second;
      }
  };
  
  Map <int, float> m;
  m[1] = 2.3f; // calls the operator[] above
  float c = m[1]; // calls normal operator[], if any
  ```

* Automatic type deduction in constexpr scope  
  *Common code after the deduction is instantiated once for each distinct deduced type.*

<table>
<tr>
<th><p>Usage:</p></th>
<th><p>Rewritten as:</p></th>
</tr>
<tr>
<td>

```cpp
template <typename T>
std::string function (T p) {
    auto r;
    if constexpr (std::is_integral_v <T>) {
        r = sub1 (p); // returns 'int'
        r <<= 4;
    } else {
        auto x = sub3 ();
        r = sub2 (p); // returns 'float'
        r /= 100.0f + x;
    }
    r += 123;
    return std::to_string (r);
}
```

</td>
<td>

```cpp
template <typename T>
std::string function (T p) {
    if constexpr (std::is_integral_v <T>) {
        int r = sub1 (p);
        r <<= 4;
        r += 123;
        return std::to_string (r);
    } else {
        auto x = sub3 ();
        float r = sub2 (p);
        r /= 100.0f * x;
        // 'x' is destroyed here
        r += 123;
        return std::to_string (r);
    }
}
```

</td>
</tr>
</table>

* Fix regular comparison operators for intrinsic types of distinct signedness  
  *...now that [chained comparisons are getting fixed](https://wg21.link/p3439).*

  ```cpp
  bool operator < (signed int a, unsigned int b) noexcept {
      if (a < 0)
          return true;
      if (b > INT_MAX)
          return true;
  
      return unsigned (a) < b;
  }
  ```

# C++ Syntactic sugar

* **Function-return-statement (akin to function-try-block)**  
  *In function definition, where function body would normally begin, or where `try` statement
  could appear, an immediate `return` can now appear too.*
  
  ```cpp
  int fma (int a, int b, int c)
      return a * b + c;
  ```
  
* `const` and `volatile` implies `auto`  
  *While it might appear ambiguous, as `a` and `b` can be existing types,
  those being immediatelly followed by `=` disambiguates the parse.*
  
  ```cpp
  const a = 1; // same as: const auto a = 1;
  volatile b = 2.0; // same as: volatile auto b = 2.0
  ```

* **Named break/continue**  
  *New control transfer statements are introduced, `break for;` and `break switch;` that can be used to
  transfer control out of nearest `for` or `switch` statements from within nested constructions.*

  ```cpp
  for (...) {
      switch (...) {
          case X:
              break for;
          case Y:
              for (...) {
                  if (...)
                      break switch;
              }
      }
  }
  ```

* **goto case/default**  
  *Within a `switch` statement, a `goto case X;` or `goto default;` can be used to transfer control to
  appropriate `case` or `default` label belonging to that switch.*
  
  ```cpp
  switch (x) {
      case 0:
          func0 ();
          goto case 2;
      case 1:
          func1 ();
          goto default;
      case 2:
          func2 ();
          break;
      default:
          func_default ();
  }
  ```
  
  `goto case/default` to a missing case label is allowed and simply jumps past the `switch` statement.
  Evaluation of `goto case a * 2 + 1;` should also be allowed.

* **Rotate operators**  
  *All ISAs have ROTL and ROTR instructions, but emitting them requires calling instrinsics or relying on
  compiler detecting and optimizing patterns. The idea is:*
  
  ```cpp
  a <<> 3; // rotate left by 3 bits
  b <>> 4; // rotate right by 4 bits
  ```
  
## Superseded ideas

* **Argument dependent lookup for scoped enumerations**  
  Superseded by [Using-enum-declaration](https://en.cppreference.com/w/cpp/language/enum#using_enum_declaration)  
  *If an argument cannot be found, relevant enum scope is searched too, enabling one to write:*
  
  ```cpp
  enum class color {
      red,
      green,
      blue,
  };
  int function (color c);
  
  // ...
  
  function (red); // color::red
  
  color green = blue; // color::blue
  function (green); // local variables still have priority
  ```

# x86 ISA Extensions proposals

* **LOCK BSFR** [BSF (Bit Scan Forward)](https://www.felixcloutier.com/x86/bsf) and Reset.  
  The same operation as BSF with addition of clearing the found set bit.
  Viola, lock-free bitmap allocator with a single instruction.
  [BSR](https://www.felixcloutier.com/x86/bsr)R optional.  
  It seems like a sensible addition to complete the [BTS](https://www.felixcloutier.com/x86/bts),
  [BTR](https://www.felixcloutier.com/x86/btr), [BTC](https://www.felixcloutier.com/x86/btc) set.

* **vpccintersectps rax, ymm0, ymm1**  
  Would set RAX to distance (squared?) of envelopes of two circles. Each YMM register would contain X,Y,Z and Radius.
  If RAX <= 0 then circles intersect, otherwise RAX is their distance (squared?).

