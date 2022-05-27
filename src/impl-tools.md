# impl-tools

Allow me introduce the [impl-tools](https://crates.io/crates/impl-tools) crate,
by discussing the limitations of `#[derive]`.

### `Default::default` for an enum

Enums often support `Default`, yet we can't use `#[derive]`. It would be nice to
have an alternative:
```rust
#[impl_tools::impl_default(Option::None)]
pub enum Option<T> {
    None,
    Some(T),
}

// (Yes, the impl is correct with regards to generics — see below.)
let x: Option<std::time::Instant> = Default::default();
```

### `Default::default` is a wrapper

Frequently, a type's public API includes a `fn new() -> Self` constructor, with
`Default` implemented over this; lets skip the boilerplate:
```rust
use impl_tools::impl_default;
use std::ffi::OsString;

#[impl_default(PathBuf::new())]
pub struct PathBuf {
    inner: OsString,
}

impl PathBuf {
    pub fn new() -> PathBuf {
        PathBuf { inner: OsString::new() }
    }
}
```

### Non-default values

Lets say we want to implement `Default` for a struct with non-default values for
fields:
```rust
struct CarStats {
    uses_diesel: bool,
    fuel_capacity_liters: f32,
    num_doors: u8,
}
```
Wouldn't it be nice to be able to specify **our** default values in-place?
```rust
use impl_tools::{impl_scope, impl_default};

impl_scope! {
    #[impl_default]
    struct CarStats {
        uses_diesel: bool,
        fuel_capacity_liters: f32 = 50.0,
        num_doors: u8 = 3,
    }
}
```
Unfortunately, `field: Ty = val` is not valid Rust syntax. Using the
[`impl_scope`](https://docs.rs/impl-tools/latest/impl_tools/macro.impl_scope.html)
macro allows us to get around this limitation.

### Skip `Debug` for hidden / unformattable fields

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

### `Deref`

We can also use `#[autoimpl]` for some traits that `#[derive]` can't support:
```rust
#[impl_tools::autoimpl(Deref, DerefMut using self.animal)]
struct Named<A> {
    name: String,
    animal: A,
}
```

## Generics

Many an experienced Rustacean will know that `#[derive]` makes some incorrect
assumptions with regards to generics. For example,
```rust,ignore
use std::fmt;
use std::marker::PhantomData;

/// Implements Debug where T: Debug (correct)
#[derive(Debug)]
enum Option<T> {
    None,
    Some(T),
}

/// Implements Debug where T: Debug (unnecessary bound)
#[derive(Debug)]
struct HiddenTy<T> {
    _pd: PhantomData<T>,
}

// Some type with its own ideas of how to implement Debug
struct Foo<T>(T);
impl<T: fmt::Display> fmt::Debug for Foo<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Foo({})", &self.0)
    }
}

/// Implements Debug where T: Debug (error: should use bound T: Display)
#[derive(Debug)]
struct S<T> {
    foo: Foo<T>,
}
```

The `#[autoimpl]` macro takes a different approach: do not assume any bounds,
but allow explicit listing of bounds as required.
```rust
use impl_tools::autoimpl;
use std::fmt;
use std::marker::PhantomData;

// autoimpl does not currently support enums: issue #6

#[autoimpl(Debug)]
struct HiddenTy<T> {
    _pd: PhantomData<T>,
}

// Some type with its own ideas of how to implement Debug
struct Foo<T>(T);
impl<T: fmt::Display> fmt::Debug for Foo<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Foo({})", &self.0)
    }
}

#[autoimpl(Debug where T: fmt::Display)]
struct S<T> {
    foo: Foo<T>,
}
```

To simplify the most common usage and to cover the case where multiple traits
are implemented simultaneously, the keyword `trait` may be used as a bound:
```rust
# use impl_tools::autoimpl;
#[autoimpl(Clone, Debug, Default where T: trait)]
struct S<T>(T);
```
