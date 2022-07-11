# Iterable, lending iterators, etc

{{#include ../badges/nightly.md}} {{#include ../badges/stabilization-96709.md}}

## Summary

Traits often contain methods that return data borrowed from `self` or some other argument. When the type of data is an associated type, it needs to include a lifetime that links with `self` from a calling method. For example, in the `Iterable` trait...

```rust
trait Iterable {
    type Item<'me>
    where
        Self: 'me;

    type Iter<'me>: Iterator<Item = Self::Item<'me>>
    where
        Self: 'me;

    fn iter(&iter self) -> Self::Item<'_>;
}
```

...the `Item` and `Iter` traits take a `'me` parameter, which is linked to the `self` variable given when `iter` is called.

## Details

There are many variants on this pattern:

* `Iterable`, as shown above;
* `LendingIterator` (and other `LendingFoo`) traits, which permit one to iterate over items but where the data may be stored within the iterator itself;
* etc.

The `where Self: 'me` shown in the summary is (hopefully) a temporary limitation imposed by the current MVP. It indicates that the `'me` lifetime can be used to borrow data from `Self`. Currently these where clauses are mandatory; they may be defaulted or made optional in the future. For a deeper explanation, see the [required bounds](../explainer/required_bounds.md) page in the explainer.

## Workarounds

Lacking this pattern, there are a number of common workarounds, each with downsides:

* Use `Box<dyn>` values, as in [graphene](https://github.com/Emoun/graphene/issues/7), though this adds dynamic dispatch overhead, inhibits inlining, and makes interactions with `Send` and `Sync` more complex;
* Return a collection, like a `Vec`, [as in metamolectular](https://github.com/metamolecular/gamma/issues/8), though this results in unnecessary memory allocation;
* Use HRTB, as [rustc does](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_data_structures/graph/trait.WithSuccessors.html), which is complex and leaks into your caller's signatures.