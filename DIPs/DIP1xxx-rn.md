Implementing Copy Constructor

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | (your name and contact data)                                    |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

This document proposes the implementation of copy constructors as an alternative
to the design flaws and inherent limitations of the postblit.

### Reference

* [1] https://github.com/dlang/dmd/pull/8032

* [2] https://forum.dlang.org/post/p9p64v$ue6$1@digitalmars.com

* [3] http://en.cppreference.com/w/cpp/language/copy_constructor

* [4] https://dlang.org/spec/struct.html#struct-postblit

* [5] https://news.ycombinator.com/item?id=15008636

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale and Motivation

This section highlights the existing problems with the postblit and motivates
why the implementation of a copy constructor is more desirable than an attempt
to fix all the postblit issues.

### Overview of this(this)

The postblit function was designed as a non-overloadable, non-qualifiable
function. However, up until recently [1], the compiler did not reject the
following code:

```D
struct A { this(this) const {} }
struct B { this(this) immutable {} }
struct C { this(this) shared {} }
```

Since the semantics of the postblit in the presence of qualifiers was
not defined and most likely not intended, this led to a series of problems:

* `const` postblits were not able to modify any fields in the destination
* `immutable` postblits would never get called (resulting in compilation errors)
* `shared` postblits cannot guarantee atomicity while blitting the fields

#### `const`/`immutable` postblits

The solution for `const` and `immutable` postblits is to type check them as normal
constructors where the first assignment of a member is treated as an initialization
and subsequent assignments are treated as modifications. This is problematic because
after the blitting phase, the destination object is no longer in its initial state
and subsequent assignments to its fields will be regarded as modifications making
it impossible to construct `immutable`/`const` objects in the postblit. In addition,
it is possible for multiple postblits to modify the same field. Consider:

```D
struct A
{
    immutable int a;
    this(this)
    {
        this.a = a + 2;  // modifying immutable or ok?
    }
}

struct B
{
    A a;
    this(this)
    {
        this.a.a = a.a + 2;  // modifying immutable error or ok?
    }
}

void main()
{
    B b = B(A(7));
    B c = b;
}
```

When `B c = b;` is encountered, the compiler does the following steps:

1. blits the fields of `b` to `c`
2. calls A's postblit
3. calls B's postblit

After `step 1`, the object `c` has the exact contents as `b` but it is not
initialized (the postblits still need to fix it) nor uninitialized (the field
`B.a` does not have its initial value). From a type checking perspective
this is a problem because the compiler will decide that the assignment
inside A's postblit is breaking immutability. This makes it impossible to
postblit objects that have `immutable`/`const` fields. To alleviate this problem
we can consider that after the blitting phase the object is in a raw state,
therefore uninitialized; this will allow the compiler to consider the first
assignment of `B.a.a` as an initialization. However, after this step the field
`B.a.a` is considered initialized, therefore how is the compiler suppose to type
check the assignment inside B's postblit? Is it breaking immutability or should
it be legal? Indeed it is breaking immutability since it is changing an immutable
value, however being part of initialization (remember that `c` is initialized only
after all the postblits are ran) it should be legal, thus weakening the immutability
concept and creating a different strategy from the one that normal constructors
implement.

#### Shared postblits

Shared postblits cannot guarantee atomicity while blitting the fields because
that part is done by the compiler automatically and it does not involve any
synchronization techniques. The following example demonstrates the problem:

```D
shared struct A
{
    long a, b, c;
    this(this) { ... }
}

A a = A(1, 2, 3);

void fun()
{
    A b = a;
    /* do other work */
}
```

Let's consider the above code is ran in a multithreaded environment
When `A b = a;` is encountered the compiler does:

* `mempcy` fields from a to b
* call user code defined in this(this)

The compiler does not do any locking when doing `memcpy` (blitting), which means
that while the copying is done, another thread may modify `a`'s data resulting
in the corruption of `b`. In order to fix this issue there are 4 possibilities:

1. Make shared objects larger than 2 words uncopyable. This solution cannot be
taken into account since it imposes a major arbitrary limitation: almost all
structs will become uncopyable.

2. Allow the compiler to memcpy them incorrectly and expect that the user will
do the necessary synchronization. Example:

```D
shared struct A
{
    Mutex m;
    long a, b, c;
    this(this) { ... }
}

A a = A(1, 2, 3);

void fun()
{
    A b;
    a.m.aquire();
    b = a;
    a.m.release();
    /* do other work */
}
```

Although this solution solves our synchronization problem it does it in a manner
that requires unencapsulated attention at each copy site. Another problem is
represented by the fact that the mutex release is done after the postblit was
ran which imposes some overhead. The release may be done right after the blitting
phase (first line of the postblit) because the copy is thread-local, but then
we end up with non-scoped locking: the mutex is released in a different scope
than the scope in which it was acquired. Also, the mutex is automatically
(wrongfully) copied.

3. Introduce a preblit function that will be called before blitting the fields.
The purpose of the preblit is to offer the possibility of preparing the data
before the blitting phase; acquiring the mutex on the `struct` that will be
copied is one of the operations that the preblit will be responsible for. Later
on, the mutex may be released in the postblit. This approach has the benefit of
minimizing the mutex protected area in a manner that offers encapsulation, but
suffers the disadvantage of adding even more complexity on top of existing one by
introducing a new concept which requires typecheking of disparate sections of
code (need to typecheck across preblit, mempcy and postblit).

4. Use an implicit global mutex to synchronize the blitting of fields. This approach
has the advantage that the compiler will do all the work and synchronize all blitting
phases (even if the threads don't actually touch each other's data) at the cost
of performance. Python implements a global interpreter lock and it was proven
to cause unscalable high contention; there are ongoing discussions of removing it
from the Python implementation [5].

### Introducing the copy constructor

As stated above, the fundamental problem with the postblit is the automatic blitting
of fields which makes it impossible to type check and cannot be synchronized without
additional costs.

As an alternative, this DIP proposes the implementation of a copy constructor. The benefits
of this approach are the following:

* it is a known concept. C++ has it [3];
* it can be typechecked as a normal constructor (since no blitting is done, the data is
initialized as if it were in a normal constructor); this means that `const`/`immutable`/`shared`
copy constructors will be type checked exactly as their analogous constructors
* offers encapsulation

The downside of this solution is that the user must do all the field copying by hand
and every time a field is added to a struct, the copy constructor must be modified.
However, this can be easily bypassed by D's introspection mechanisms. For example,
this simple code may be used as a language idiom or library function:

```D
static foreach (i, ref field; src.tupleof)
    this.tupleof[i] = field;
```

As shown above, the single benefit of the postblit can be easily substituted with a
few lines of code. On the other hand, to fix the limitations of the postblit it is
required that more complexity is added for little to no benefit. In these circumstances,
for the sake of uniformity and consistency, replacing the postblit constructor with a
copy constructor is the reasonable thing to do.

## Description

Required.

Detailed technical description of the new semantics. Language grammar changes
(per https://dlang.org/spec/grammar.html) needed to support the new syntax
(or change) must be mentioned. Examples demonstrating the new semantics will
strengthen the proposal and should be considered mandatory.

## Breaking Changes and Deprecations

This section is not required if no breaking changes or deprecations are anticipated.

Provide a detailed analysis on how the proposed changes may affect existing
user code and a step-by-step explanation of the deprecation process which is
supposed to handle breakage in a non-intrusive manner. Changes that may break
user code and have no well-defined deprecation process have a minimal chance of
being approved.


## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
