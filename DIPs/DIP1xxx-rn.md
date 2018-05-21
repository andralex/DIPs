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

[1] https://github.com/dlang/dmd/pull/8032
[2] https://forum.dlang.org/post/p9p64v$ue6$1@digitalmars.com
[3] http://en.cppreference.com/w/cpp/language/copy_constructor
[4] https://dlang.org/spec/struct.html#struct-postblit

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

A solution for `const` and `immutable` postblits is to type check them as normal
constructors where the first assignment of a member is treated as an initialization
and subsequent assignments are treated as modifications. Although this solves the
problem for simple cases, it does not work for more complex ones were field and
aggregated postblits are involved:

```D
struct A
{
    int a;
    this(this) immutable
    {
        this.a = a;
    }
}

struct B
{
    immutable A a;
    this(this) immutable
    {
        this.a = a;   // error : initialization of immutable field `a` multiple times
    }
}

void main()
{
    B c = B(A(8));
    immutable B b = c;
}
```

In the above code, when `immutable B b = c;` is encountered, the compiler
generates a call to `B.__aggrPostblit` which is a function that calls
`A.__postblit` and then `B.__postblit`. Once `A.__postblit is called, according
to the control flow analysis done by the compiler, `B.a` is initialized and
the subsequent altering of `B.a` will be viewed as violating immutability.

The solution for this would be to enhance the control flow analysis to
identify such situations, but this is against the principle that the compiler
should do only primitive flow analysis. Moreover, the compiler internally
generated functions `__aggrPostblit` and `__fieldPostblit` are generated once
per struct declaration (not per instance) and are unqualified; this results
in the impossibility of changing the qualifier of the generated postblits.

As seen above, the main issue with postblits and qualifiers is the way in which
the postblit is recursevely called for struct field members which declare postblits.
The current strategy requires hacks around the type system (like casting to mutable)
in order to properly call the generated postblits and to be able to assign to immutable
fields. Although solutions to this problem might exist, all known ones are very
complicated and do not address all the theoretical requirements of the postblit.

As an alternative, the copy constructor is a concept with known semantics which offers the
opportunity of redesigning the postblit from scratch in order to take into account the behavior
in the presence of all qualifiers. As a bonus, implementing the copy constructor is an easier
task than working through the various hacks that the postblit uses; for example: the copy
constructor does not need to create additional internal functions since the user needs
to do all the copying by hand. Although, this is the major advantage that the postblit
has over the copy constructor, the fact is that the copying may be trivially done by
means of generic programming :

```D
static foreach (i, ref field; src.tupleof)
    this.tupleof[i] = field;
```

In addition, the copy constructor type check will be identical to the normal
constructor one, leading to uniformity and consistency.

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
