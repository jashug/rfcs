- Feature Name: `fn_static`
- Start Date: 2022-10-17
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)
- **Status:** Pre-RFC

# Summary
[summary]: #summary

Introduce a trait `FnStatic` that represents functions that can be called without any `self` parameter at all.

Automatically implement this trait for closures with no upvars.

(Large sections of this proposal cribbed from [Extending the capabilities of compiler-generated function types](https://lang-team.rust-lang.org/design_notes/fn_type_trait_impls.html), written by nikomatsakis and Diggsey. This proposal advocates for one of the mentioned possible solutions.)

Some aspects of this proposal depend on the [`fn_traits`](https://github.com/rust-lang/rust/issues/29625) feature.

# Motivation
[motivation]: #motivation

There are several cases where it is necessary to write a [trampoline]. A trampoline
is a (usually short) generic function that is used to adapt another function
in some way.

Trampolines have the caveat that they must be standalone functions. They cannot
capture any environment, as it is often necessary to convert them into a
function pointer.

Trampolines are most commonly used by compilers themselves. For example, when a
`dyn Trait` method is called, the corresponding vtable pointer might refer
to a trampoline rather than the original method in order to first down-cast
the `self` type to a concrete type.

However, trampolines can also be useful in low-level code that needs to interface
with C libraries, or even in higher level libraries that can use trampolines in
order to simplify their public-facing API without incurring a performance
penalty.

By expanding the capabilities of compiler-generated function types it would
be possible to write trampolines using only safe code.

[trampoline]: https://en.wikipedia.org/wiki/Trampoline_(computing)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Background

Both standalone functions and closures have unique compiler-generated types.
The rest of this document will refer to both categories as simply "function
types", and will use the phrase "function types without upvars" to refer to
standalone functions _and_ closures without upvars.

Today, these function types have a small set of capabilities, which are
exposed via trait implementations and implicit conversions.

- The `Fn`, `FnMut` and `FnOnce` traits are implemented based on the way
  in which upvars are used.

- `Copy` and `Clone` traits are implemented when all upvars implement the
  same trait (trivially true for function types without upvars).

- `auto` traits are implemented when all upvars implement the same trait.

- Function types without upvars have an implicit conversion to the
  corresponding _function pointer_ type.

## Wrapping a C callback

In this example, we are building a crate which provies a safe wrapper around
an underlying C API. The C API contains at least one function which accepts
a function pointer to be used as a callback:

```rust
mod c_api {
    extern {
        pub fn call_me_back(f: extern "C" fn());
    }
}
```

We would like to allow users of our crate to safely use their own callbacks.
The problem is that if the callback panics, we would unwind into C code and this
would be undefined behaviour.

To avoid this, we want to interpose between the user-provided callback and
the C API, by wrapping it in a call to `catch_unwind`. Unfortunately, the C API
offers no way to pass an additional "custom data" field that we could use to
store the original function pointer.

Instead, we can write a generic function like this:

```rust
use std::{panic, process};

pub fn call_me_back_safely<F: FnStatic()>(_f: F) {
    extern "C" fn catch_unwind_wrapper<F: FnStatic()>() {
        if panic::catch_unwind(|| {
            F::call_static()
        }).is_err() {
            process::abort();
        }
    }
    unsafe {
        c_api::call_me_back(catch_unwind_wrapper::<F>);
    }
}
```
We can then use our new API like so:
```rust
fn my_callback() {
    println!("I was called!")
}

fn main() {
    call_me_back_safely(my_callback);
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO: This section is not yet written

## Corner cases

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives

Several mechanisms have been proposed to allow trampolines to be written in safe
code. These have been discussed at length in the following places.

PR adding `Default` implementation to function types:

- https://github.com/rust-lang/rust/pull/77688

Lang team triage meeting discussions:

- https://youtu.be/NDeAH3woda8?t=2224
- https://youtu.be/64_cy5BayLo?t=2028
- https://youtu.be/t3-tF6cRZWw?t=1186

See also [Extending the capabilities of compiler-generated function types](https://lang-team.rust-lang.org/design_notes/fn_type_trait_impls.html).

## What are the consequences of not doing this?

# Prior art
[prior-art]: #prior-art

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities
