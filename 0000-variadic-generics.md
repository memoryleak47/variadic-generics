- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC adds variadic generics to Rust.

# Motivation
[motivation]: #motivation

* [fn types need a reform](http://smallcultfollowing.com/babysteps/blog/2013/10/10/fn-types-in-rust/), and being able to define a trait with a variable number of type parameters would help
* working with functions which have a variable number of arguments is impossible right now (e.g. a generic `bind` method, `f.bind(a, b)(c) == f(a, b, c)`) and defining such functions may only be done (in a limited fashion) with macros
* tuples also have similar restrictions right now, there is no way to define a function which takes any tuple and returns the first element, for example

# Detailed design
[detailed-design]: #detailed-design

## The unfold syntax for tuple values

This RFC adds the tuple unfold syntax `..`.
This syntax is allowed only in specific contexts, where a comma-separated list of expressions is expected.
The elements of the tuple are inserted into this list.
The entire set of permitted contexts is as follows:
- Function Call
```rust
    fn addition(x: u32, y: u32) { x + y }
    let result = addition(..(1, 2));
    assert_eq!(result, 3);
```
- Macro Invocation
```rust
    let t = (1u32, 1u32);
    assert_eq!(..t);
```
- Array
```rust
    let a = [1, ..(2, 3, 4)];
    assert_eq!(a, [1, 2, 3, 4]);
```
- Tuple

```rust
    let a = (1, ..(2, 3, 4));
    assert_eq!(a, (1, 2, 3, 4));

In addition to this, you can use the `..`-syntax on function parameters, if they have a tuple type.
```rust
    fn addition(..arg: (u32, u32)) -> u32 {
        arg.0 + arg.1
    }
```
or using `..`-syntax again:
```rust

    fn addition(..arg: (u32, u32)) -> u32 {
        [..arg].iter().sum()
    }
```
An argument prefixed by `..` has to be the last function argument.
These addition functions are equivalent to the definition of `addition` above.

## The unfold syntax for tuple types

Analogous to the unfold syntax for tuple values, there is also such a syntax for tuple types.
The entire set of permitted contexts is as follows:
- Type Parameters in definitions
```rust
    fn foo<..T>() { ... }
```
In this context, if `foo<A, B, C, D>()` is called, the type T would be equal to the tuple type `(A, B, C, D)`.
foo can also be called with one ore zero type arguments, this would cause T to be a one-element-tuple or the unit-type respectively.
The `..T` syntax is also allowed in combination with other function parameters:
```rust
    fn bar<T, ..U>() { ... }
```
But it is important, that every tuple type parameter is the last type parameter.
Calling `bar<A, B, C, D>()` would cause T to be equal to A and U to be equal to `(B, C, D)`.
- Type Parameters in applications
```
    foo<..(u32, u32)>();
    type A = HashMap<..(String, bool)>;
```
- Where Clauses
```rust
    fn foo<..T>() where ..T: Into<u32> { ... }
```

This requires every type within the tuple type T to implement `Into<u32>`.

### Examples
This requires https://github.com/rust-lang/rust/issues/20041
```rust
    fn u32_addition<..T>(..arg: T) -> u32
            where ..T = u32 {
        [..arg].iter().sum()
    }

    fn any_addition<T: std::ops::Add<T, Output=T>, ..U>(..arg: U) -> T
            where ..U = T {
        [..arg].iter().sum()
    }

    fn tuple<..T>(..arg: T) -> T { arg }

    assert_eq!((true, false), tuple(true, false));
```
# Drawbacks
[drawbacks]: #drawbacks

This RFC adds a new feature, namely the `..` syntax on tuples and therefore adds complexity to the language.

# Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- don't add this or anything similar

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
