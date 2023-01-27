*Jan Ringoš*
# C++ Design Proposal: Destructive Move

A destructive move is final move-from, that is allowed to leave the object in random/invalid state.
After that point, after destructive move-from, the object will (should) never be accessed again.
No (other) destructor is executed for it.

## Key assumptions

* destructive move-from can happen to object during its lifetime only once
* destructive move-from ends object's lifetime
* touching the object after being destructively moved-from is an error

## What destructive move fits C++

* simple and minimal initial design
* only classes implementing the new mechanism, as it's a change in life-time, can be destructively moved-from
* transparently works throughout existing code-bases, containers and standard library, without need for changes

## Proposed mechanism

Classes may define two new destructors, see Syntax below.
For instances of such classes the compiler performs aditional life-time analysis:
If it can **prove** the instance isn't touched after the last move-from (either implicit or `std::move`),
it is allowed to (observably) replace that last move-from by the appropriate destructor,
and end its lifetime there.

If any of the conditions isn't met a regular move, or copy, whichever is defined, is called.

In some situations, like RVO or NRVO now, it would be guaranteed that the destructive move, if defined, is called instead.

## Syntax

    struct A {
        ~A (A & a) noexcept {
            // destructively assign content into 'a'
        }
        ~A () noexcept -> A {
            // destructively (N)RVA construct new A
            return A{ … };
        }
    };

Design considerations for the syntax above:

* it's a destructive move, thus destructor
* destructors don't return now? so what? it's special :)

Destructive assignment:

    A a;
    A b;
    // …
    b = std::move (a); // invokes a.~A(b);
    // …
    // never use 'a' within this scope

Destructive initialization:

    A a;
    // …
    A b (std::move (a)); // the second destructor NRVO-constructs 'b'
    // …
    // never use 'a' within this scope

## Rationale

Emphasis is on *minimalistic* here. This design certainly doesn't solve what everyone wants.
But let's start with incremental approach because
**(1)** Nothing larger is going to get through the process within our lifetimes,
**(2)** We're not getting anything like Rust, nor any magic bullet, in C++ ever,
**(3)** obviously everyone is attempting to solve way too much in a single go.

## Possible extensions
* both destructors could be `= default`, akin to regular move, creating objects with life-times possibly shorter than their scope
* [[attribute]] for debug methods allowed to be called on destructively moved-from objects?

