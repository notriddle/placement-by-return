- Feature Name: `repr_transparent`
- Start Date: 2016-09-26
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Extend the existing `#[repr]` attribute on newtypes with a `transparent` option
specifying that the type representation is the representation of its only field.
This matters in FFI context where `struct Foo(T)` might not behave the same
as `T`.


# Motivation
[motivation]: #motivation

On some ABIs, structures with one field aren't handled the same way as values of
the same type as the single field. For example on ARM64, functions returning
a structure with a single `f64` field return nothing and take a pointer to be
filled with the return value, whereas functions returning a `f64` return the
floating-point number directly.

This means that if someone wants to wrap a `f64` value in a struct tuple
wrapper and use that wrapper as the return type of a FFI function that actually
returns a bare `f64`, the calls to this function will be compiled incorrectly
by Rust and the execution of the program will segfault.

This also means that `UnsafeCell<T>` cannot be soundly used in place of a
bare `T` in FFI context, which might be necessary to signal to the Rust side
of things that this `T` value may unexpectedly be mutated.

```c
// The value is returned directly in a floating-point register on ARM64.
double do_something_and_return_a_double(void);
```

```rust
mod bogus {
    #[repr(C)]
    struct FancyWrapper(f64);

    extern {
        // Incorrect: the wrapped value on ARM64 is indirectly returned and the
        // function takes a pointer to where the return value must be stored.
        fn do_something_and_return_a_double() -> FancyWrapper;
    }
}

mod correct {
    #[repr(transparent)]
    struct FancyWrapper(f64);

    extern {
        // Correct: FancyWrapper is handled exactly the same as f64 on all
        // platforms.
        fn do_something_and_return_a_double() -> FancyWrapper;
    }
}
```

Given this attribute delegates all representation concerns, no other `repr`
attribute should be present on the type. This means the following definitions
are illegal:

```rust
#[repr(transparent, align = "128")]
struct BogusAlign(f64);

#[repr(transparent, packed)]
struct BogusPacked(f64);
```

# Detailed design
[design]: #detailed-design

The `#[repr]` attribute on newtypes will be extended to include a form such as:

```rust
#[repr(transparent)]
struct TransparentNewtype(f64);
```

This structure will still have the same representation as a raw `f64` value.

Syntactically, the `repr` meta list will be extended to accept a meta item
with the name "transparent". This attribute can be placed only on newtypes,
which means structures (and structure tuples) with a single field.

Some examples of `#[repr(transparent)]` are:

```rust
// Transparent struct tuple.
#[repr(transparent)]
struct TransparentStructTuple(i32);

// Transparent structure.
#[repr(transparent)]
struct TransparentStructure { only_field: f64 }
```

This new representation is mostly useful when the structure it is put on must be
used in FFI context as a wrapper to the underlying type without actually being
affected by any ABI semantics.

It is also useful for `AtomicUsize`-like types, which [RFC 1649] states should
have the same representation as their underlying types.

[RFC 1649]: https://github.com/rust-lang/rfcs/pull/1649

# Drawbacks
[drawbacks]: #drawbacks

None.

# Alternatives
[alternatives]: #alternatives

None.

# Unresolved questions
[unresolved]: #unresolved-questions

* Should this representation be usable on structures that contain a single
  non-zero-sized field and `PhantomData` markers?
