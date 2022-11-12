# Required bounds

{{#include ../badges/nightly.md}} {{#include ../badges/stabilization-96709.md}} {{#include ../badges/seeking-feedback.md}}

> *We are actively soliciting feedback on the design of this aspect of GATs. This section explains the current nightly behavior, but at the end there is note about the behavior we expect to adopt in the future.*

A common use for GATs is to represent the lifetime of data that is borrowed from a value of type `Self`, or some other parameter. Consider the `iter` method of the `Iterable` trait:

```rust
trait Iterable {
    ...

    fn iter<'a>(&'a self) -> Self::Iterator<'a>;
}
```

Here, the argument `'a` that is given to `Self::Iterator` represents the lifetime of the `self` reference. However, by default, there is nothing in the definition of the `Iterator` associated type that links its lifetime argument and the `Self` type:

```rust
// Warning: For reasons we are in the midst of explaining,
// this version of the trait will not compile.
trait Iterable {
    type Item<'me>;

    type Iterator<'me>: Iterator<Item = Self::Item<'me>>;

    fn iter<'a>(&'a self) -> Self::Iterator<'a>;
}
```

If you try to compile this trait, you will find that you get an error. To make it compile, you have to indicate the link between `'me` and `Self` by adding `where Self: 'me`. This [outlives bound](https://doc.rust-lang.org/nightly/reference/trait-bounds.html?highlight=outlives#lifetime-bounds) indicates the GATs can only be used with a lifetime `'me` that could legally be used to borrow `Self`. This version compiles:

```rust
trait Iterable {
    type Item<'me>
    where
        Self: 'me;

    type Iterator<'me>: Iterator<Item = Self::Item<'me>>
    where
        Self: 'me;

    fn iter<'a>(&'a self) -> Self::Iterator<'a>;
}
```

## Why are these bounds required?

Without these bounds, users of the trait would almost certainly not be able to write the impls that they need to write. Consider an implementation of `Iterable` for `Vec<T>`, assuming that there are no `where Self: 'me` bounds on the GATs:

```rust
impl Iterable for Vec<T> {
    type Item<'c> = &'c T;
    type Iterator<'c> = std::vec::Iter<'c, T>;
    fn iter(&self) -> Self::Iterator<'_> { self.iter() }
}
```

The problem comes from the associated types. Consider the `type Item<'c> = &'c T` declaration, for example: for the type `&'c T` to be legal, we must know that `T: 'c`. Otherwise, nothing stops us from using `Self::Item<'static>` to construct a reference with a lifetime `'static` that may outlive its referent `T`, and that can lead to unsoundness in the type system. In the case of the `iter` method, the fact that it takes a parameter `self` of type `&'c Vec<T>` already implies that `T: 'me` (otherwise that parameter would have an invalid type). However, that doesn't apply to the GAT `Item`. This is why the associated types need a where clause:

```rust
impl Iterable for Vec<T> {
    type Item<'c> = &'c T where Self: 'c;
    type Iterator<'c> = std::vec::Iter<'c, T> where Self: 'c;
    fn iter(&self) -> Self::Iterator<'_> { self.iter() }
}
```

However, this impl is not legal unless the trait *also* has a `where Self: 'c` requirement. Otherwise, the impl has more where clauses than the trait, and that causes a problem for generic users that don't know which impl they are using.

## Where can I learn more?

You can learn more about the precise rules of this decision, as well as the motivations, by visiting the [detailed design page](../design-discussions/outlives-defaults.md).

## Feedback requested!

The current compiler adds the future defaults as a **hard error** precisely so that we can get the attention of early users and find out if these where clauses pose any kind of problem. We are not sure yet what long term path is best:

* Remove the required bounds altogether.
* Remove the required bounds and replace them with a warn-by-default lint, allowing users to more easily opt out.
* Add the required bounds by default so you don't have to write them explicitly.

If you are finding that you have a trait and impls that you believe would compile fine, but doesn't because of these where clauses, then we would like to hear from you! Please [file an issue] on this repository, and use the "Feedback on required bounds" template. In the meantime, there is a workaround described in the next section that should allow any trait to work.

[file an issue]: https://github.com/rust-lang/generic-associated-types-initiative/issues/new/choose

## Workaround

If you find that this requirement is causing you a problem, there is a workaround. You can refactor your trait into two traits. For example, to write a version of `Iterable` that doesn't require `where Self: 'me`, you might do the following:

```rust
trait IterableTypes {
    type Item<'me>;
    type Iterator<'me>: Iterator<Item = Self::Item<'me>>;
}

trait Iterable: IterableTypes {
    fn iter<'a>(&'a self) -> Self::Iterator<'a>;
}
```

This is a bit heavy-handed, but there's a logic to it: the rules are geared around ensuring that the associated types and methods that appear together in a single trait will work well together. By separating the associated types from the function into distinct traits, you are effectively asserting that the associated types are meant to be used independently from the function and hence it doesn't necessarily make sense to have the where clauses derived from the method signature.
