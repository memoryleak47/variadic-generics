- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC adds variadic generics using tuples.

# Motivation
[motivation]: #motivation

* [fn types need a reform](http://smallcultfollowing.com/babysteps/blog/2013/10/10/fn-types-in-rust/), and being able to define a trait with a variable number of type parameters would help
* working with functions which have a variable number of arguments is impossible right now (e.g. a generic `bind` method, `f.bind(a, b)(c) == f(a, b, c)`) and defining such functions may only be done (in a limited fashion) with macros
* tuples also have similar restrictions right now, there is no way to define a function which takes any tuple and returns the first element, for example

# Detailed design
[detailed-design]: #detailed-design

## Tuple Types

This RFC adds abstract-tuple-types;<br />
These types are `trait`s or at least behave like them, as you can use them as trait bounds and they are unsized.<br />
abstract-tuple-types are types of tuples, which impose conditions to all tuple elements.<br />
A "tuple-type" is a either an abstract-tuple-type or any type matching `(T1, ..., Tn)`.
It is forbidden to implement an abstract-tuple-type on any type manually.
### Syntax
Note, that the following informal syntax is not part of abstract-tuple-types, but will be used to define the syntax!<br />
- `[]`-brackets represent optional elements.<br />
- `...` represent continuing patterns.<br />
- `<>`-brackets represents placeholders.<br />

Syntax of an abstract-tuple-type: `(<type-expression>;T1[: <type_1>], ..., Tn[: <type_n>], <condition_1>, ..., <condition_m>)`<br />
`type-expression`, `condition_i`s, and `type_i`s may contain the types `T1, ..., Tn`.

### Semantics
A non-abstract tuple-type `(S1, ..., Sm)` is subtype of the abstract-tuple-type `(<type-expression>;T1[: <type_1>], ..., Tn[: <type_n>], <condition_1>, ..., <condition_m>)`,<br />
iff for every tuple member-type `Si`, there exist types `T1`, to `Tn`,<br />
which satify all conditions (`condition_1` to `condition_m`),<br />
so that `Si` matches the `type-expression`, where `T1` to `Tn` are inserted into `type-expression` accordingly.
### Examples
- tuples which only contain `u32`: `(u32;)`
- tuples where all members are `Clone`: `(T;T: Clone)`
- any tuples: `(T;T)`
- any tuples, which only contain elements, which are pairs of addable types: `((T,U); T: std::ops::Add<U>, U)`

## The unfold syntax

This RFC adds the `..`-syntax for destructuring tuples.<br />
Intuitively the `..(a, b, ..., z)` syntax removes the parens of the tuple,<br />
and therefore returns a comma-separated list `a, b, ..., z`.<br />
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

Analogous to the unfold syntax for tuple values, there is also such a syntax for tuple types.<br />
The `..` syntax is only applicable for those types, which are Tuple Types by definition above.<br />
This is also just allowed in very specific contexts as defined below.

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

The Asterisk Syntax allows to create<br />
- functions with a variable amount of parameters, and
- generics with a variable amount of generic parameters<br />
by folding this variable amount of parameters into one tuple paramter of variable size, which in turn may be processed using `..`.

#### in Type Parameters in definitions
```rust
    fn foo<*T>() { ... }
```
In this context, if `foo<A, B, C, D>()` is called, the type T would be equal to the tuple type `(A, B, C, D)`.<br />
foo can also be called with one (named `U`) or zero type arguments, this would cause T to be a `(U,)` or `()` respectively.<br />
Every asterisk type parameter `*T` implicitly gets the trait bound `(X;X)`, and therefore you can call `..T` onto it.<br />
The `*T` syntax is also allowed in combination with other generic parameters:
```rust
    fn bar<T, *U>() { ... }
```
But it is important, that every asterisk type parameter is the last type parameter.<br />
Calling `bar<A, B, C, D>()` would mean `T = A` and `U = (B, C, D)`.<br />
Asterisk type parameters can not only be used for functions but anywhere, where normal type parameters are allowed.

#### on Function Parameters
In addition to this, you can use the `*`-syntax on function parameters.<br />
Consider a function with an argument `*arg: (A, B, C, D)`: this causes the function to accept 4 distinct arguments of type A, B, C and D.<br />
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
An argument prefixed by `*` has to be the last function argument.<br />
These addition functions are equivalent to the definition of `addition` above.<br />
The asterisk can also be used in lambda-expressions, and can also be combined with `mut`: `|mut *arg| { ... }`.

## Examples
```rust
    use std::fmt::Display;

    // addition
    fn u32_addition<T: (u32;)>(*arg: T) -> u32 {
        [..arg].iter().sum()
    }

    fn any_addition<T: std::ops::Add<T, Output=T>, U: (T;)>(*arg: U) -> T {
        [..arg].iter().sum()
    }

    // tuple
    fn tuple<T>(*arg: T) -> T { arg }

    // display tuple
    trait DisplayTuple {
        fn printall(&self);
    }

    impl DisplayTuple for () {
        fn printall(&self) {}
    }

    impl<T, U> DisplayTuple for (T, ..U)
                where T: Display, U: DisplayTuple {
        fn printall(&self) {
            let (a, ..b) = self;
            println!("{}", a);
            b.printall();
        }
    }

	// bind
    fn bind<T, F: Fn(T, ..U), *U>(f: F, t: T) -> impl Fn(..U) {
        |*u: U| f(t, ..u)
    }

    fn main() {
        assert_eq!((true, false), tuple(true, false));
        (2i32, 3u32).printall();
    }
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
The asterisk syntax `*T` is inspired by pythons `*args`-syntax.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### What parts of the design do you expect to resolve through the RFC process before this gets merged?
- Syntax: whether `..` and `*` are the best choices
