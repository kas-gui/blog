# Easy Cast

Introducing: [easy-cast].

## Problem: correct casts aren't easy

Let me ask you, have you ever done this?
```rust
let v: Vec<_> = make_collection();

// We need a fixed-size type (not usize), or just want to save space.
// We can assume length <= u32::MAX.
let len = v.len() as u32;
```
Of course, you could write this instead:
```rust
let v: Vec<_> = make_collection();

// We need a fixed-size type (not usize), or just want to save space.
use std::convert::TryFrom;
let len = u32::try_from(v.len()).unwrap();
```

What about this?
```rust
let x: f32 = some_position();

// Convert to the nearest integer, assuming it fits
let x = (x.round()) as i32;
```

Or this?
```rust
use rand_distr::{Poisson, Distribution};
let distr = Poisson::new(1e6).unwrap();
let num = poi.sample(&mut rand::thread_rng());

// Note: u64::MAX as f64 rounds to 2^64, so we need to use < not <= here!
assert!(0.0 <= num && num < u64::MAX as f64);
let num = num as u64;
```

Or even this?
```rust
let x: i32 = some_value();
let y = x as f32;
if y as i32 == x {
    println!("No loss of precision!");
    // WARNING: this test is WRONG since e.g. i32::MAX rounds to 2^31 on cast
    // to f32, then rounds back to i32::MAX on cast to i32.
}
```

### Summary

It should be *easy* and *safe* to convert one numeric type to
another, but frequently it's not:

-   Using `x as T` is easy but not *safe*: it may truncate, sign-convert, round,
    or saturate. (Before [Rust 1.45.0] behaviour might even be undefined.)
-   Using `T::try_from(x).unwrap()` is *clunky* when you expect the conversion
    to succeed.
-   `TryFrom` doesn't even *support* float conversions.
-   Conceptually, "convert a float to the nearest integer" is a *single*
    operation, yet most solutions require separate rounding and conversion steps.

It turns out this is not an easy problem to solve *in general*. I wrote
[an RFC](https://github.com/rust-lang/rfcs/pull/2484) on this. Nearly three
years and a long discussion later, it's still unsolved.

### Existing solutions

The standard library provides:

-   `From`, but it only covers portably infallible conversions
-   `TryFrom`, but it misses float conversions and expects you to do your own
    error handling

There are existing libraries on `crates.io`, yet all of them fall short in some way:

-   [num_traits::NumCast](https://docs.rs/num-traits/0.2.14/num_traits/cast/trait.NumCast.html):
    API is not extensible to user types; error handling is the user's responsibility;
    it's a relatively big library if you just want casts
-   [cast](https://docs.rs/cast/): different API for fallible and infallible
    conversions; API around `isize` and `usize` is platform-dependent;
    error handling is the user's responsibility

[Rust 1.45.0]: https://github.com/rust-lang/rust/blob/stable/RELEASES.md#version-1450-2020-07-16


## Introducing easy-cast

Eventually I gave up trying to find a standard solution and wrote my own,
starting with one particular problem (converting between `usize` and `u32`
indices in KAS-text, which stores *many* text and list indices). This tiny
beginning grew into the [easy-cast] library:

-   a `From`-like trait, [`Conv`], for all integer conversions
-   direct support for float-to-int-with-rounding via [`ConvFloat`]
-   easier usage via the [`Cast`] trait (like `Into`)
-   `try_cast` and `try_conv`, allowing user error handling on all conversions
-   and of course a few bug fixes and diagnostic improvements

### Examples

Now, we can revisit the above problems using `easy-cast`:

```rust
use easy_cast::Conv;

let v: Vec<_> = make_collection();
// Expect success, panic on failure
let len = u32::conv(v.len());
```

```rust
use easy_cast::CastFloat;

let x: f32 = some_position();
// Convert to the nearest integer, panic if out-of-range
let x: i32 = x.cast_nearest();
```

```rust
use easy_cast::CastFloat;

use rand_distr::{Poisson, Distribution};
let distr = Poisson::new(1e6).unwrap();
let num = poi.sample(&mut rand::thread_rng());

let num: u64 = num.cast_trunc();
```

```rust
use easy_cast::Conv;

let x: i32 = some_value();
if let Ok(y) = f32::try_conv(x) {
    println!("No loss of precision!");
}
```

For any integer `x` and any integer or float type `T`, we get:

-   `T::conv(x)`
-   `x.cast()` with type inference
-   `T::try_conv(x)` and `x.try_cast()` returning `Result<T, Error>`

And for any float value `x` and integer type `T` we have:

-   `x.cast_nearest()`, `x.cast_trunc()`, `x.cast_floor()`, `x.cast_ceil()`
-   `T::conv_nearest(x)`, etc.
-   `T::try_conv_trunc(x)`, `x.try_cast_floor()`, etc.

### Don't need the fuses?

One of the design goals for easy-cast was *check everything in Debug builds,
maybe not in Release builds*. In a Debug build, or when the `always_assert`
crate feature is used, `cast` and `conv` will panic on inexact conversions.
In Release builds without this flag, the cast reduces to just `x as T`.

The same is not true for `try_cast` and `try_from` which always fail on
inexact conversions.

We may make some adjustments here in future versions, e.g. to use
`always_assert` by default (but if so, it will be considered a breaking change).


[easy-cast]: https://crates.io/crates/easy-cast
[`Conv`]: https://docs.rs/easy-cast/latest/easy_cast/trait.Conv.html
[`ConvFloat`]: https://docs.rs/easy-cast/latest/easy_cast/trait.ConvFloat.html
[`Cast`]: https://docs.rs/easy-cast/latest/easy_cast/trait.Cast.html
