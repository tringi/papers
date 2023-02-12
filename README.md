*C++ and other*
# Papers &amp; proposals

* [C++ Destructive Move](cxx-destructive-move.md) - quick and dirty proposal draft

      struct A {
          ~A (A & a) noexcept { ... }
          ~A () noexcept -> A { return A{ ... }; }
      };

# Syntactic sugar ideas

* Argument dependent lookup for scoped enumerations
  If an argument cannot be found, relevant enum scope is searched too, enabling one to write:
  
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
      

* Function-return-statement (akin to function-try-block)
  In function definition, where function body would normally begin, or where `try` statement
  could appear, an immediate `return` can now appear too.

      int fma (int a, int b, int c)
          return a * b + c;

* `const` and `volatile` implies `auto`

      const a = 1; // same as: const auto a = 1;
      volatile b = 2.0; // same as: volatile auto b = 2.0

