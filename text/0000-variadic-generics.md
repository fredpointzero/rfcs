- Feature Name: variadic_generics
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow types and functions to be generic over a variable number of types.
This will allow users to write impls which are abstract over one or more
given types when the number of types are of an arbitrary length e.g.
`Tuple`.

# Motivation
[motivation]: #motivation

Currently implementing variadic functions is accomplished through the use
of a variadic macro as described in the [Variadic Interfaces] section of
the book. This may be an adequate workaround for functions, but there is
no way to extend this to work with impls or traits. As a result, the
following still hold:

 - Implementing traits for tuples could use some cleanup. For example, it
   is currently impossible to compare two Tuples with `T: Eq` where the
   length of the tuple is above 12. The implementations of the standard
   traits `PartialEq`, `Eq`, and `Default` are implemented in a macro
   which is repeated for the lengths 1 - 12. See [src/libcore/tuple.rs]
   for details.
 - Working with function which have a variable number of arguments is
   currently impossible. For example, a generic `bind` method,
   `f.bind(a, b)(c) == f(a, b, c)`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Declaring a variadic
[declaring-a-variadic]: #declaring-a-variadic

Variadic type parameters always take the form `...$ty`.

### Defining types
[defining-types]: #defining-types

A `...T` in generic formal type parameters, where `...T` is the trailing
type will define the type as a variadic type.

```rust
struct S<...T>;

S();
S(42);
S(12, 13, 17)
```

### Defining functions
[defining-functions]: #defining-functions

Similar to defining types the same syntax may be used in a functions generic
parameters to define a variable length list of types.

```rust
fn bar<...T>(/* arguments */);
```

### Capturing multiple arguments
[capturing-multiple-arguments]: #capturing-multiple-arguments

In the functions arguments the pattern `...args: T` (where `T` is the name of
the variadic list of types and `args` is the argument name) will be allowed
to define a function which takes a variable number of arguments at the position
of `args`.

```rust
fn bar<...T>(...args: T) -> R {...}

bar();
bar(42);
bar(12, 13, 17);
```

### Unpacking variadic arguments
[unpacking-variadic-arguments]: unpacking-variadic-arguments

In order to unpack the variable length list of arguments in the function
body the pattern `...args` may be used.

```rust
fn f<...T>(...args: T) -> R {
    // Unpack args and pass them to the function g
    g(...args)
}
```

### Defining bounds for `T`
[defining-bounds-for-t]: #defining-bounds-for-t

A type bound for each type in the variadic list of types `...T` may be
defines with the syntax `...T: Trait`. This indicates that each type type
in `T` must satisfy the bound `Trait`.

```rust
fn example<...T: Clone>(...args: T);
```

### Iterating over the list
[iterating-over-the-list]: #iterating-over-the-list

Now that the types in `T` can be bound and the list of arguments may be unpacked
it becomes possible to iterate over the list of arguments and call trait methods
on each argument in the list.

```rust
fn example<...T: Clone>(...args: T) {
    // All types in T implement clone, so we can use
    // the clone method on the unpacked args
    f(...args.clone());
}
```

### The terminating condition
[the-terminating-condition]: #the-terminating-condition

**TODO(dlrobertson)**: What do we do here? Rely on `Default`?

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

```rust
// in another tuple type:
type AppendTuple<T, U> = (...T, U);
type PrependTuple<T, U> = (U, ...T);
AppendTuple<PrependTuple<Tuple<B>, A>, C> => (A, B, C)
// in a fn type's parameter list:
type VoidFn<...Args> = fn(...Args);
VoidFn<A, B, C> => fn(A, B, C);
// in a variadic generic parameter list:
Tuple<...(A, B, C), ...(D, E, F)> => (A, B, C, D, E, F)
```

The simple examples shown above only use `...T`, but not `T` itself. The
question which arises is this: what is `T`? `T` is merely a tuple of types
which stores the actual variadic generic type parameters:

```rust
// Completely equivalent definition:
type Tuple<...T> = T;
// Detailed expansion of (...T) from above:
Tuple<> => (...()) => ()
Tuple<int> => (...(int,)) => (int,)
Tuple<A, B, C> => (...(A, B, C)) => (A, B, C)
```

Naturally this could be extended to tuples of values allowing for better
ergonomics in some variants of tuple destruction.

```rust
// Destructure a tuple.
let (head, ..tail) = foo;

// in another tuple:
(...(true, false), ...(1, "foo"), "bar") => (true, false, 1, "foo", "bar")
// in a function call:
f(...(a, b), ...(c, d, e), f) => f(a, b, c, d, e, f)
```

Due to the layout of `Tupple` types and the possible introduction of
padding to propperly align types, the type of the r-value returned from
unpacking a `Tupple` should be carefully considered. When capturing the
remaining elements of a `Tupple` with `...` syntax, the capture will
capture each value in the tail by reference. The capture is a tuple of
references to the remaining values in `foo`.

```rust
// types
(head, ...tail) = (A, B, C) // head: A, tail: (B, C)
// values
(head, ...tail) = (a, b, c) // head: a, tail: (&b, &c)
```

## Position of list
[position-of-list]: #position-of-list

The list `...T` must be the trailing type in the function call or type definition.

```rust
type Tuple<...T, U> // invalid
type Tuple<U, ...T> // valid
```

The list `...T` may not be the trailing type in the generic parameters of
a function, but must be the trailing type in the function signature.

```rust
fn invalid_fn<...T, U>(args: ...T, x: U); // invalid
fn valid_fn<U, ...T>(x: U, args: ...T); // valid
fn valid_fn<...T, U>(x: U, args: ...T); // valid
```

Using two variadic typs in a generic is always invalid.

```
type Tuple<...T, (), ...U> // invalid
type Tuple<...T, ...U> // invalid
```

## Implementing `Tuple` `zip`
[implementing-tuple-zip]: #implementing-tuple-zip

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for
  not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

C++11 sets a decent precedent with its variadic templates, which can be
used to define type-safe variadic functions, among other things. C++11
has a special case for variadic parameter packs.

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this
  feature exist in other programming languages and what experience have
  their community had?
- For community proposals: Is this done by some other community and what
  were their experiences with it?
- For other teams: What lessons can we learn from what other communities
  have done here?
- Papers: Are there any published papers or great posts that discuss this?
  If you have some relevant papers to refer to, this can serve as a more
  detailed theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other languages, provide readers of your RFC with a fuller
picture.
If there is no prior art, that is fine - your ideas are interesting to us
whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it
does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally
diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Duplicating syntax
[duplicating-syntax]: #duplicating-syntax

We are using the same syntax (`...` as a prefix) for both declaration and
expansion. C++ uses `...` as a prefix for the declaration and `...` as a
postfix for expansion. This does make it much easier to differentiate the
two. For example, given the following:

```rust
type Example<...T>
```

Is `T` a new variadic list of types or did the users intended to unpack
the `Tuple` of types named `T`? It would be preferable to use `...` as a
postfix (like C++), but it is currently reserved for ranges.

## Trait bound syntax
[trait-bound-syntax]: trait-bound-syntax

Is the use of `...T: Trait` a wise use of the syntax. Would it be better
to define `T` as a non-variadic generic first, followed by the variadic
definition. For example:

```rust
fn example<T: Clone, ...T>(...args: T) {
    f(...args.clone());
}
```

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the
  implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could
  be addressed in the future independently of the solution that comes out of
  this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

[Eq for Tuple]: https://doc.rust-lang.org/std/cmp/trait.Eq.html#impl-Eq-100
[src/libcore/tuple.rs]: https://github.com/rust-lang/rust/blob/master/src/libcore/tuple.rs
[Variadic Interfaces]: https://doc.rust-lang.org/rust-by-example/macros/variadics.html
[C++ fold expressions]: https://en.cppreference.com/w/cpp/language/fold
