:arrow_backward: [C++ Proposals and ideas](README.md)

# C++ Design Proposal: Destructive Move

A destructive move is final move-from, that is allowed to leave the object in random invalid state.
After destructive move-from, the object's lifetime ended, and it will (should) never be accessed again.
No (other) destructor is executed for it.

There are many designs proposed or talked about for destructive move in C++, the following one is the least intrusive.

## Key assumptions

* destructive move-from can happen to object during its lifetime only once
* destructive move-from ends object's lifetime
* touching the object after being destructively moved-from would be an error

## What destructive move fits C++

* simple and minimal initial design
* only classes implementing the new mechanism can be destructively moved-from (it changes object's lifetime)
* transparently works throughout existing code-bases, containers and standard library, without need for changes

## The proposed mechanism

Classes may define two new destructors, see [Syntax](#Syntax) below.

For instances of such classes, the compiler performs additional lifetime analysis:
If it can **prove** the instance isn't touched after the last move-from (either implicit or `std::move`),
it is allowed to **observably** replace that last move-from by the appropriate new destructor (see below),
and end its lifetime there (early).

*If any of the conditions isn't met, a regular move, or copy, whichever is defined, is called.*

## Syntax

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

Destructive assignment:

```cpp
A a;
A b;
// ...
b = std::move (a); // invokes a.~A(b);
// ...
// never use 'a' within this scope
```

Destructive initialization:

```cpp
A a;
// ...
A b (std::move (a)); // the second destructor NRVO-constructs 'b'
// ...
// never use 'a' within this scope
```

## Rationale

Emphasis is on *minimalistic* here. This design certainly doesn't solve what everyone wants, it offers start of an incremental approach because:
1. nothing larger is going to get through the process within our lifetimes,
2. we're not getting anything like Rust, nor any magic bullet, in C++ (probably) ever,
3. obviously everyone is attempting to solve way too much in a single go.

## Possible extensions
* both destructors could be `= default`, akin to regular move, creating objects with life-times possibly shorter than their scope
* some `[[ attribute ]]` for debug methods allowed to be called on destructively moved-from objects
* in some situations, like RVO or NRVO exist now, it could be guaranteed that the destructive move, if defined, is called instead
* passing the address (or a reference) of `a` (above) to a function cancels the eligibility for destructive move
   * unless, perhaps, the compiler can proove the function doesn't store or forward the pointer/reference

## TODO:
* inheritance
* how members are destroyed, with/without destructive move defined
