# Windows OS improvement recipes

* TODO: [DWM: ClearType on composited/translucent surfaces](win32-composited-cleartype.md)
  *Defer text rendering to be done by DWM at composition time. DwmAddDeferredText (HWND, RECT, HFONT, ...);*

* TODO: [Start Menu: Uninstall command for Win32](win32-uninstall-from-start.md)
  *Introduce extended IShellLink2 to embed uninstaller path or uninstall registry key into .lnk file
   so that Start Menu can directly run it, and bring consistency with Store/Moden apps*

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

# C++ Syntactic sugar ideas

* **Argument dependent lookup for scoped enumerations**  
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

* **Function-return-statement (akin to function-try-block)**  
  *In function definition, where function body would normally begin, or where `try` statement
  could appear, an immediate `return` can now appear too.*
  
  ```cpp
  int fma (int a, int b, int c) return a * b + c;
  ```
  
* `const` and `volatile` implies `auto`
  
  ```cpp
  const a = 1; // same as: const auto a = 1;
  volatile b = 2.0; // same as: volatile auto b = 2.0
  ```
