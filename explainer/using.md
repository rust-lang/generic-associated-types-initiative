# Using GATs

Suppose I want to write a function that accepts any iterable and creates a vector by cloning its elements. I can now write such a thing like this:

```rust
fn into_vec<T>(
    iterable: &impl for<'a> Iterable<Item<'a> = &'a T>,
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
iterable: &impl for<'a> Iterable<Item<'a> = &'a T>,
//              ^^^^^^^          ^^^^^^^^
```

This admittedly verbose syntax is a way of saying:

* `iterable` is some kind of `Iterable` that, when iterated with some lifetime `'a`, yields up values of type `&'a T`.

The `for<'a>` binder is a way of making this bound apply for any lifetime, rather than talking about some specific lifetime.

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