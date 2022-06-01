# Impl-tools: beyond derive

Allow me introduce the [impl-tools](https://crates.io/crates/impl-tools) crate,
by discussing the limitations of `#[derive]`.

### Deriving `Defualt`...

#### ... over a method

Frequently, a type's public API includes a `fn new() -> Self` constructor, with
`Default` implemented over this. `impl_default` lets us skip some boilerplate:
```rust
use impl_tools::impl_default;

#[impl_default(Foo::new())]
pub struct Foo {
    // fields here
}

impl Foo {
    pub fn new() -> Foo {
        Foo { /* fields here */ }
    }
}

let foo = Foo::default();
```

#### ... for an enum

Similarly, we can derive `Defualt` for enums:
```rust
#[impl_tools::impl_default(Option::None)]
pub enum Option<T> {
    None,
    Some(T),
}

// (Yes, the impl is correct with regards to generics — see below.)
let x: Option<std::time::Instant> = Default::default();
```

#### ... with specified field values

Lets say we want to implement `Default` for a struct with non-default values for
fields:
```rust
struct CarStats {
    num_doors: u8,
    fuel_is_diesel: bool,
    fuel_capacity_liters: f32,
}
```
Wouldn't it be nice to be able to specify **our** default values in-place? We can, if we re-write using impl-tools:
```rust
use impl_tools::{impl_scope, impl_default};

impl_scope! {
    #[impl_default]
    struct CarStats {
        num_doors: u8 = 3,    // specified default value
        fuel_is_diesel: bool, // no initializer: uses type's default value
        fuel_capacity_liters: f32 = 50.0,
    }
}
```
Note that `field: Ty = val` is not (currently) Rust syntax. The
[`impl_scope`](https://docs.rs/impl-tools/latest/impl_tools/macro.impl_scope.html)
macro has special support for this, besides other functionality.

### Deriving `Debug`: ignoring hidden fields

For example, let us consider [`Lcg64Xsh32`](https://github.com/rust-random/rand/blob/master/rand_pcg/src/pcg64.rs) (also known as [PCG32](https://www.pcg-random.org/)). This is a simple random number generator, and as per [policy of the `RngCore` trait](https://docs.rs/rand/latest/rand/trait.RngCore.html), does not print out internal state in its `Debug` implementation.
```rust
#[derive(Clone, PartialEq, Eq)]
pub struct Lcg64Xsh32 {
    state: u64,
    increment: u64,
}

// We still implement `Debug` since generic code often requires it
# use std::fmt;
impl fmt::Debug for Lcg64Xsh32 {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Lcg64Xsh32 {{}}")
    }
}
```

Using `impl-tools` we could reduce this to:
```rust
#[impl_tools::autoimpl(Debug ignore self.state, self.increment)]
#[derive(Clone, PartialEq, Eq)]
pub struct Lcg64Xsh32 {
    state: u64,
    increment: u64,
}
```

(This applies equally to fields which do not themselves implement `Debug`.)

### Deriving `Deref`

How could we derive `Deref` and `DerefMut`? By *using* a specified field:
```rust
#[impl_tools::autoimpl(Deref, DerefMut using self.animal)]
struct Named<A> {
    name: String,
    animal: A,
}
```

## Generics

An experienced Rustacean should know that `#[derive]` makes some incorrect
assumptions with regards to generics. For example,
```rust,ignore
/// Implements Clone where T: Clone (correct)
#[derive(Clone)]
enum Option<T> {
    None,
    Some(T),
}

/// Implements Clone where T: Clone (unnecessary bound)
#[derive(Clone)]
struct Shared<T> {
    inner: std::rc::Rc<T>,
}

/// Attempts to implement Clone where T: Clone
/// (error: Clone for Cell<T> requires T: Copy)
#[derive(Clone)]
struct InnerMutable<T> {
    inner: std::cell::Cell<T>,
}
```

The `#[autoimpl]` macro takes a different approach: do not assume any bounds,
but allow explicit listing of bounds as required.
```rust
use impl_tools::autoimpl;

// Note: autoimpl does not currently support enums (issue #6)

// No bound on T assumed
#[autoimpl(Clone)]
struct Shared<T> {
    inner: std::rc::Rc<T>,
}

// Explicit bound on T
#[autoimpl(Clone where T: Copy)]
struct InnerMutable<T> {
    inner: std::cell::Cell<T>,
}
```

To simplify the most common usage and to cover the case where multiple traits
are implemented simultaneously, the keyword `trait` may be used as a bound:
```rust
# use impl_tools::autoimpl;
#[autoimpl(Clone, Debug, Default where T: trait)]
struct Wrapper<T>(T);
```

## Auto trait implementations

Lets say you write a trait, and wish to implement that trait for reference types:
```rust
trait Greet {
    fn greet(&self, name: &str);
}

impl<T: Greet + ?Sized> Greet for &T {
    fn greet(&self, name: &str) {
        (*self).greet(name);
    }
}

// Also impl for &mut T, Box<T>, ...
```
This can be quite tedious, enough so that macros (by example) are often used to
deduplicate the implementations. But why should we have to write even
*the first* implementation? It's all trivial code!
```rust
use impl_tools::autoimpl;

// One line to do it all:
#[autoimpl(for<T: trait + ?Sized> &T, &mut T, Box<T>)]
trait Greet {
    fn greet(&self, name: &str);
}

// A test, just to prove it works:
impl Greet for String {
    fn greet(&self, name: &str) {
        println!("Hi {name}, my name is {self}!");
    }
}
let s = "Zoe".to_string();
s.greet("Alex");
(&s).greet("Bob");
Box::new("Bob".to_string()).greet("Zoe");
```
