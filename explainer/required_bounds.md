# Required bounds

![nightly][] ![seeking-feedback][] 

{{#include ../badges.md}}

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
    type Item<'me> = &'me T;
    type Iterator<'me> = std::vec::Iter<'me, T>;
    fn iter(&self) -> Self::Iterator<'_> { self.iter() }
}
```

The problem comes from the associated types. Consider the `type Item<'me> = &'me T` declaration, for example: for the type `&'me T` to be legal, we must know that `T: 'me`. Otherwise, nothing stops us from using `Self::Item<'static>` to construct a reference with a lietime `'static` that may outlive its referent `T`, and that can lead to unsoundness in the type system. In the case of the `iter` method, the fact that it takes a parameter `self` of type `&'me Vec<T>` already implies that `T: 'me` (otherwise that parameter would have an invalid type). However, that doesn't apply to the GAT `Item`. This is why the associated types need a where clause:

```rust
impl Iterable for Vec<T> {
    type Item<'me> = &'me T where Self: 'me;
    type Iterator<'me> = std::vec::Iter<'me, T> where Self: 'me;
    fn iter(&self) -> Self::Iterator<'_> { self.iter() }
}
```

However, this impl is not legal unless the trait *also* has a `where Self: 'me` requirement. Otherwise, the impl has more where clauses than the trait, and that causes a problem for generic users that don't know which impl they are using.

## Precise rules

The precise rules that the compiler uses to decide when to issue an error are as follows:

* For every GAT `G` in a trait definition with generic parameters `X0...Xn` from the trait and `Xn..Xm` on the GAT... (e.g., `Item` or `Iterable`, in the case of `Iterable`, with generic parameters `[Self]` from the trait and `['me]` from the GAT)
    * If for every method in the trait... (e.g., `iter`, in the case of `Iterable`)
        * When the method signature (argument types, return type, where clauses) references `G` like `<P0 as Trait<P1..Pn>>::G<Pn..Pm>` (e.g., `<Self as Iterable>::Iterator<'a>`, in the `iter` method, where `P0 = Self` and `P1` = `'a`)...
            * we can show that `Pi: Pj` for two parameters on the reference to `G` (e.g., `Self: 'a`, in our example)
                * then the GAT must have `Xi: Xj` in its where clause list in the trait (e.g., `Self: 'me`).

## Plan going forward: Add rules by default

In the future, rather than reporting an error in cases like this, we expect to add these where clauses to the trait and impl by default. However, there is some possibility that this will rule out legal impls for which the where clause might not be necessary. This can happen if either (a) the lifetime, despite being linked to a type like `Self` in the methods, is not used to borrow content from `Self`; or (b) in all impls of this trait that could exist, the types being borrowed don't have references and are known to be `'static`.

## Feedback requested!

The current compiler adds the future defaults as a **hard error** precisely so that we can get the attention of early users and find out if these where clauses pose any kind of problem. If you are finding that you have a trait and impls that you believe would compile fine, but doesn't because of these where clauses, then we would like to hear from you! Please [file an issue] on this repository, and use the "Feedback on required bounds" template.

[file an issue]: https://github.com/rust-lang/generic-associated-types-initiative/issues/new/choose