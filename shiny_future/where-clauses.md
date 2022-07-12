# GATs in where-clauses

{{#include ../badges/speculative.md}}

> 🚨 **Warning: Speculation ahead.** This is the "shiny future" page that integrates various speculative features. To see how things work today, see [the corresponding page on the explainer](../explainer/where-clauses.md).

Now that we have defined an [`Iterable`](./iterable.md) trait, we can explore different ways to reference it.

## Specifying the value of a GAT

Given some type `T: Clone`, this function takes any `Iterable` that yields `&T` references, clones them, and returns a vector of the resulting `T` values:

```rust
fn into_vec<T>(
    iterable: &impl Iterable<Item<'_> = &T>,
) -> Vec<T>
where
    T: Clone
{
    let mut out = vec![];
    for elem in iterable.iter() {
        out.push(elem.clone());
    }
    out
}
```

Let's look at this function more closely. The most interesting part is the type of the `iterable` parameter:

```rust
fn into_vec<T>(
    iterable: &impl Iterable<Item<'_> = &T>,
//                                ^^    ^
//                                |     |
//                                |    Lifetime elided in output position
//                               Lifetime elided in input position          
) -> Vec<T>
```

Both `'_` and `&T`-with-no-explicit-lifetime are examples of Rust's "lifetime elision" syntax. You're probably familiar with elision from functions like this one:

```rust
fn pick(c: &[T]) -> &T
//         ^        ^
//         |        |
//         |       Lifetime elided in output position
//        Lifetime elided in input position          
```

Whenever lifetimes are elided in *input* position, it means "pick any lifetime, I don't care". When they are elided in output position, it means "pick a lifetime from the inputs, or error if that's ambiguous". For functions, you can use a named lifetime to make the connection more explicit:

```rust
fn pick<'c>(c: &'c [T]) -> &'c T
```

In the same way, with GATs, we can use a named lifetime, bound with `for`, to make things more explicit:

```rust
fn into_vec<T>(
    iterable: &impl for<'c> Iterable<Item<'c> = &'c T>,
    //              -------               --     --
) -> Vec<T>
```

The `for` notation here is meant to read like "for any lifetime `'c`, `Item<'c>` will be `&'c T`".

## Applying GATs to a specific lifetime

The previous example showed an iterable applied to any lifetime. It is also possible to give bounds for some specific lifetime. This function, for example, takes an `iterable` with lifetime `'i` and yields up the first element:

```rust
fn first<'i, T>(
    iterable: &'i impl Iterable<Item<'i> = &'i T>,
) -> Option<&'i T>
{
    iterable.iter().next()
}
``` 

The bound `impl Iterable<Item<'i> = &'i T>` says "when iterated with lifetime `'i`, the resulting reference is `&'i T`".


## Bounding a GAT

Sometimes we want to specify that the value of a GAT meets some additional trait bound. For example, maybe wish to accept any `Iterable`, so long as its `Item` values implement `Send`. The `'_` notation we saw earlier can be used to do that quite easily:

```rust
fn sendable_items<I>(iterable: &I)
where
    I: Iterable,
    I::Item<'_>: Send, // 👈
{
}
```

Using another nightly feature (`associated_type_bounds`, tracked in [#52662]), you can also write the above more compactly:

[#52662]: https://github.com/rust-lang/rust/issues/52662

```rust
fn sendable_items<I>(iterable: &I)
where
    I: Iterable<Item<'_>: Send>, // 👈
```