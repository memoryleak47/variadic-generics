- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC adds variadic generics using a new tuple destructuring syntax.

# Motivation
[motivation]: #motivation

* [fn types need a reform](http://smallcultfollowing.com/babysteps/blog/2013/10/10/fn-types-in-rust/), and being able to define a trait with a variable number of type parameters would help
* working with functions which have a variable number of arguments is impossible right now (e.g. a generic `bind` method, `f.bind(a, b)(c) == f(a, b, c)`) and defining such functions may only be done (in a limited fashion) with macros
* tuples also have similar restrictions right now, there is no way to define a function which takes any tuple and returns the first element, for example

# Detailed design
[detailed-design]: #detailed-design

## The Tuple Type

This RFC adds "tuple-types";
These are types of tuples, which impose conditions to all tuple elements.
#### Examples
- Tuples which only contain `u32`: `(u32;)`
- Tuples where all members are `Clone`: `(T;T: Clone)`
- Any Tuples: `(T;T)`
- Old School Tuple Types: `(i32, u32)`

## The unfold syntax

This RFC adds the `..`-syntax for destructuring tuples.
Intuitively the `..(a, b, ..., z)` syntax removes the parens of the tuple,
and therefore returns a comma-separated list `a, b, ..., z`.
This is just allowed in very specific contexts as defined below.

### The unfold syntax for tuple values

#### in Function Calls
```rust
    fn addition(x: u32, y: u32) { x + y }
    let t = (1, 2);
    let result = addition(..t);
    assert_eq!(result, 3);
```
#### in Arrays
```rust
    let a = [1, ..(2, 3, 4)];
    assert_eq!(a, [1, 2, 3, 4]);
```
#### in Tuples
```rust
    let a = (1, ..(2, 3, 4));
    assert_eq!(a, (1, 2, 3, 4));
```
#### Destructuring
```rust
	let x = (1u32, 2u32, 3u32);
	let (a, ..b) = x;
	let (ref c, ref ..d) = x;
	assert_eq!(a, 1u32);
	assert_eq!(b, (2u32, 3u32));
	assert_eq!(c, &1u32);
	assert_eq!(d, (&2u32, &3u32));
```

### The unfold syntax for tuple types

Analogous to the unfold syntax for tuple values, there is also such a syntax for tuple types.
The `..` syntax is only applicable for those types, which are Tuple Types by definition above.
The entire set of permitted contexts is as follows:
#### in Tuple Types
```rust
    type Family32 = (u32, ..(f32, i32));
```
#### in Type Parameters
```rust
    foo<..(u32, u32)>();
    type A = HashMap<..(String, bool)>;
```
## The Asterisk Syntax

The Asterisk Syntax combines tuples and variadic functions and templates.
A function- or template- parameter prefixed by an asterisk `*` will automatically fold all following arguments into a tuple.

#### in Type Parameters in definitions
```rust
    fn foo<*T>() { ... }
```
In this context, if `foo<A, B, C, D>()` is called, the type T would be equal to the tuple type `(A, B, C, D)`.
foo can also be called with one or zero type arguments, this would cause T to be a one-element-tuple or the unit-type respectively.
The `*T` syntax is also allowed in combination with other function parameters:
```rust
    fn bar<T, *U>() { ... }
```
But it is important, that every asterisk type parameter is the last type parameter.
Calling `bar<A, B, C, D>()` would mean `T = A` and `U = (B, C, D)`.
Asterisk type parameters can not only be used for functions but anywhere, where normal type parameters are allowed.

#### on Function Parameters
In addition to this, you can use the `*`-syntax on function parameters.
Consider a function with an argument `*arg: (A, B, C, D)`: this causes the function to accept 4 distinct arguments of type A, B, C and D.
These are internally put together into the quadruple `arg`, when the function is called.
```rust
    fn addition(*arg: (u32, u32)) -> u32 {
        arg.0 + arg.1
    }
```
or using `..`-syntax again:
```rust

    fn addition(*arg: (u32, u32)) -> u32 {
        [..arg].iter().sum()
    }
```
An argument prefixed by `*` has to be the last function argument.
These addition functions are equivalent to the definition of `addition` above.

## Examples
```rust
    fn u32_addition<T: (u32;)>(*arg: T) -> u32 {
        [..arg].iter().sum()
    }

    fn any_addition<T: std::ops::Add<T, Output=T>, U: (T;)>(*arg: U) -> T
        [..arg].iter().sum()
    }

    fn tuple<*T>(*arg: T) -> T { arg }

    fn main() {
        assert_eq!((true, false), tuple(true, false));
    }
```

## Edge cases
`*`-parameters can also be mutable.
```rust
    fn foo(mut *args: (u32, u32)) { ... }
```

# Drawbacks
[drawbacks]: #drawbacks

This RFC adds a new feature, namely the `..` and `*` syntax on tuples and therefore adds complexity to the language.

# Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- don't add this or anything similar

# Prior art
[prior-art]: #prior-art

C++11 has powerful variadic templates, yet these have some drawbacks:
- Tedious syntax: `template <typename ...Ts> void foo(Ts ...args) { ... }`
- You can't easily impose conditions onto types like `T: Clone`

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
