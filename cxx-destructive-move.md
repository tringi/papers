:arrow_backward: [C++ Proposals and ideas](README.md)

# C++ Design Proposal: Destructive Move

A destructive move is the final (within a scope, compiler-detected) move-from, which is allowed to leave the object in random invalid state.
After destructive move-from, the object's lifetime ended, and it will never be accessed again.
No (other) destructor is executed for it.

There are many designs proposed or talked about for destructive move in C++, the following one is the least intrusive.

TL;DR: Use `std::move` as normal. The compiler replaces the last one with destructive move, if defined.

## Key assumptions

* destructive move-from can happen to object during its lifetime only once
* destructive move-from ends object's lifetime
* which move-from is final, and thus destructive, is detected by the compiler

## What destructive move fits C++

* simple and minimal initial design
* only classes implementing the new mechanism can be destructively moved-from (it changes object's lifetime)
* transparently works throughout existing code-bases, containers and standard library, without need for changes

## The proposed mechanism

Classes may define two new destructors, see [Syntax](#Syntax) below.  
We call them *"destructively-movable-from"*, *"partially"* if both destructors are not defined.

For destructively-movable-from instances, the compiler performs additional static lifetime analysis:
If it can **prove** the instance isn't touched after the last move-from (either implicit or `std::move`),
it is allowed to **observably** replace that last move-from by the appropriate new destructor (see below),
and end its lifetime there (early).

*If any of the conditions isn't met, a regular move, or copy, whichever is defined, is called.*

If (N)RVO cannot be applied on `return` statement, if the class is destructively-movable-from, it is destructively moved from.

## Syntax
Destructively-movable-from class defines at least one of these destructors:

```cpp
struct A {
    ~A (A & a) noexcept {
        // destructively assign content into 'a'
    }
    ~A () noexcept -> A {
        // destructively (N)RVO construct new A
        return A { ... };
    }
};
```

Design considerations for the syntax above:

* it's a destructive move, thus destructor
* destructors don't return value now? so what? it's special :)

<table>
<tr>
<th><p>Destructive assignment:</p></th>
<th><p>Destructive initialization:</p></th>
</tr>
<tr>
<td>

```cpp
{
    A a;
    A b;
    // ...
    b = std::move (a); // normal move-from assignment
    // because...
    somefunc (a); // ...'a' is re-used here and below
    // ...
    b = std::move (a); // invokes a.~A(b);
    // because...
    // ...'a' is never used within this scope again
}
```

</td>
<td>

```cpp
{
    A a;
    // ...
    A b (std::move (a)); // normal move-from constructor
    A c (std::move (a)); // the a's dtor (N)RVO-constructs 'c'
    // ... because 'a' is never used within this scope again
}
```

</td>
</tr>
</table>

## Rationale

Emphasis is on *minimalistic* here. This design certainly doesn't solve what everyone wants, it offers start of an incremental approach because:
1. nothing larger is going to get through the process within our lifetimes,
2. we're not getting anything like Rust, Circle, nor any magic bullet, in C++ (probably) ever,
3. obviously everyone is attempting to solve way too much in a single go.

## FAQ:
* **Rule of Seven?**  
  No. The behavior of the two new extra destructors is completely independent to regular move and copy. Adding them possibly changes lifetime of the class.

* **What happens if I use the variable after it's destructively moved-from?**  
  It's impossible. The compiler trivially sees the last time it's touched within a scope, and will not call destructive move-from before that point.

* **Does taking address of the variable change anything?**  
  Taking address, just like any operation on the variable, makes any preceeding `std::move` on that variable ineligible to move from it destructively.
  Any subsequent move is still eligible to shorten it's lifetime.

## Syntax: Forcefully invoking destructive move destructor
This is probably antipattern, but as an example, let's implement regular move operations in terms of destructive move operators:

```cpp
struct A {
    // ...see above

    A & operator = (A && other) {
        other.~A (*this); // destructively move 'other' into 'this'
        new (&other) A;   // construct new 'other'
        return *this;
    }
    A (A && other) : A (other.~A ()) { // destructively (for 'other') RVO-construct this A
        new (&other) A;   // construct new 'other'
    }
}
```

## Remarks
* The identical rules for inheritance apply as for regular move operations

## Possible extensions
* both destructors could be `= default`, akin to regular move, creating objects with life-times possibly shorter than their scope
* some `[[ attribute ]]` for debug methods allowed to be called on destructively moved-from objects
* in some situations, like RVO or NRVO now, it could be guaranteed that the destructive move, if defined, is called instead

## TODO:
* how are destructively-movable-from members destroyed? when parent class is or isn't destructively-movable-from?
* examples side by side: passing into functions, returning

