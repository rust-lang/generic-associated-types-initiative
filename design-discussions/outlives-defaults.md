# Outlives defaults

## Summary

GATs as naively implemented have a major footgun. Given a trait like this...

```rust
trait Iterable {
    type Iter<'c>;

    fn iter<'s>(&'s self) -> Self::Iter<'s>;
}
```

...users would not be able to write a typical impl, e.g....

```rust
impl<T> Iterable for Vec<T> {
    type Iter<'c> = std::slice::Iter<'c, T>;

    fn iter(&self) -> Self::Iter<'_> {
        self.iter()
    }
}
```

This would not work because the type `Iter<'c, T>` is only well-formed if `T: 'c`, and that is not known on the associated type. How should we manage this?

## Conclusion: Require "probably needed" where-clauses on GATs to be written explicitly

The original write-up and details of the discussion can be found [here](./outlives-defaults-1.md). The conclusion was to adopt the most conservative route and **require** users to explicitly write a set of where clauses on associated types. These where-clauses are deduced by examining the method signatures of methods that appear in the same trait, looking for relationships that hold within the methods and requiring those same relationships to be reproduced on the associated type.

In our example trait `Iterable`...

```rust
trait Iterable {
    type Iter<'c>;

    fn iter<'s>(&'s self) -> Self::Iter<'s>;
}
```

...the method `iter` returns `<Self as Iterable>::Iter<'s>` (written in fully qualified form), and we have that `self: &'s Self`. The parameter type implies that `Self: 's`, and therefore we require the bound `where Self: 'c` to be placed on the associated type:

```rust
trait Iterable {
    type Iter<'c>
    where
        Self: 'c; // required for trait to compile

    fn iter<'s>(&'s self) -> Self::Iter<'s>;
}
```

## Rationale

The rationale for this decision is that it is the most forwards compatible one: we can opt to remove the required bounds later, and all code still works. We can also opt to add the required bounds by default later, and all existing code still works, it is merely more explicit than required.

## Further reading

You can read more about this decision here:

* The explainer has a [page on this decision](../explainer/required_bounds.md) that gives a more thorough explanation and covers how you can give feedback if you are finding this infereres with a trait you are trying to write.
* The [original write-up and associated discussion](./outlives-defaults-1.md) is available. The issue was also discussed on [#87479](https://github.com/rust-lang/rust/issues/87479) and there was also a [lang team design meeting](https://github.com/rust-lang/lang-team/blob/master/design-meeting-minutes/2021-09-22-GAT-defaults.md).
* This page supplied a reference version of the semantics, works through some examples, and motivates the rules.

## Reference rules

The precise rules are as follows:

* For every GAT `G` in a trait definition with generic parameters `X0...Xn` from the trait and `Xn..Xm` on the GAT... (e.g., `Item` or `Iterable`, in the case of `Iterable`, with generic parameters `[Self]` from the trait and `['me]` from the GAT)
    * If, for every method in the trait... (e.g., `iter`, in the case of `Iterable`)
        * When the method signature (argument types, return type, where clauses) references `G` like `<P0 as Trait<P1..Pn>>::G<Pn..Pm>` (e.g., `<Self as Iterable>::Iterator<'a>`, in the `iter` method, where `P0 = Self` and `P1` = `'a`)...
            * we can show that `Pi: Pj` for two parameters on the reference to `G`, and `Pi` is not `'static` (e.g., `Self: 'a`, in our example)
                * then the GAT must have `Xi: Xj` in its where clause list in the trait (e.g., `Self: 'me`).

## Frequently asked questions

### Can you work through the `Iterable` example in more detail?

You mean the reference example from this page? Sure! This trait requires a `where Self: 'c` clause on the associated type `Iter`...

```rust
trait Iterable {
    type Iter<'c>;

    fn iter<'s>(&'s self) -> Self::Iter<'s>;
}
```

...this occurs because:

* the `iter` function references `<Self as Iterable>::Iter<'s>` in its return type
* we can show that `Self: 's` in the method environment
* and `Self` is not `'static` (in fact, it's a type, not a lifetime)
* when we translate `Self: 's` into the namespace of `Iter`, we wind up with `Self: 'c`, which is the required bound

### Why do the rules ignore parameters equal to `'static`?

Consider [this example](https://github.com/rust-lang/rust/issues/87479#issuecomment-1010484170):

```rust
#![feature(generic_associated_types)]

trait X<'a> {
    type Y<'b>;
    fn foo(&self) -> Self::Y<'static>;
})):
```

Without the special case for `'static`, we would see that the return type includes

```rust
<Self as X<'a>>::Y<'static>
```

and then check that `'static: 'a` (it does, unsurprisingly), and hence conclude that we need to preserve that relationship by adding a `where 'b: 'a` clause to the associated type. But that where clause isn't likely to help any impls type check. In fact, the fact that `Self::Y<'static>` can be hard-coded into the trait signature suggests that, for all impls, the value of `Y` must either (a) not reference `'b` or else (b) only use `'b` as part of some ref `&'b T` where `T: 'static`. So really there isn't much point to adding where-clauses relating `'b`. You *could* imagine that an impl might want to have `&'a &'b u32`, and to rely on the fact that `'b: 'a` in every case where it appears in the interface -- but right now, the only usage in the interface is `'static`, and so that same type could just be `&'a &'static u32`, which would work fine.

### How do you *know* you've gotten the exact rules for required bounds correct (for backcompat)?

Well, we are pretty sure, because our algorithm is quite general. It essentially looks for any patterns or relationships between parameters found in the method signatures of the trait, modulo the carve-out for `'static` described in answer to the previous question. It's possible that we could find a source of relationships we haven't considered, or we could find that the carve-out masks something more common, but those seem unlikely, and regardless they would likely be quite obscure cases (and hence it may be possible to tweak the rules without affecting existing code, or tweak the rules in an edition).

### Bounds against other parameters

The required bounds sometimes relate to parameters on the trait itself, and not the GAT parameters:

```rust
pub trait Get<'a> {
    type Item<'c>
    where
        Self: 'a; // <-- Required

    fn get(&'a self) -> Self::Item<'static>;
}
```

The reason for this is that the value of the `Item` type likely incorporates `'a` and `Self` and relies on the relationships of those types:

```rust
pub trait Get<'a> {
    type Item<'c>
    where
        Self: 'a; // <-- Required

    fn get(&'a self) -> Self::Item<'static>;
}

impl<'a, 'b> Get<'a> for &'b [String] {
    type Item<'c> = &'a str;

    fn get(&'a self) -> Self::Item<'static> {
        &self[0]
    }
}
```

### Why not issue default bounds against other associated types?

Hmm, good question! It turns out that the idea of default bounds *is* applicable beyond GATs. For example, you might have a trait

```rust
trait Iterator<'i> {
    type Item;

    fn foo(&'i self) -> Self::Item;
}
```

and the code could suggest that `type Item` wants a where-clause like `where Self: 'i`. After all, it will only be used in cases where `&'i self` is valid type.

We actually tried to enable default bounds but found that it caused the compiler to fail to bootstrap. Interestingly the trait in question was found in gimli, and it turned out to be a case where the default bounds *weren't wrong*. They were expressing a lifetime relationship that the trait did require, but that relationship was being encoded on the trait in a different, arguably more roundabout way. The trait in question is the [`Object` trait](https://github.com/gimli-rs/object/blob/0a38064531fef4ddbaf93770a3551d333338980e/src/read/traits.rs#L24):

```rust
/// An object file.
pub trait Object<'data: 'file, 'file>: read::private::Sealed {
    ...
    type SectionIterator: Iterator<Item = Self::Section>;
    ...
    fn sections(&'file self) -> Self::SectionIterator;
}
```

The error here suggested adding `where Self: 'file` to the `type SectionIterator`. Interestingly, if you look closely at the trait header, you can see that it is `'data: 'file`. This `'data` lifetime turns out to be the lifetime of data that appears in `Self`. So e.g. an [example impl](https://github.com/gimli-rs/object/blob/0b76070f1281ebfad5b6b79c74f0c2508e5ee85c/src/read/coff/file.rs#L54) looks like this:

```rust
impl<'data, 'file, R> Object<'data, 'file> for CoffFile<'data, R>
where
    'data: 'file,
    R: 'file + ReadRef<'data>,
{
    type Segment = CoffSegment<'data, 'file, R>;
    type SegmentIterator = CoffSegmentIterator<'data, 'file, R>;
    ...
}
```

In other words, the default bound of `where Self: 'file` was correct, but was being managed in a more complex way by the trait -- i.e., by adding a special lifetime (`'data`) into the trait signature that reflects "the lifetime of borrowed data in `Self`", and then relating that lifetime to `'file`. In fact, the entire `Object` trait in gimil looks like it probably wanted to be a GAT, roughly like so:

```rust
/// An object file.
pub trait Object: read::private::Sealed {
    ...
    type SectionIterator<'file>: Iterator<Item = Self::Section>
    where
        Self: 'file;
    ...
    fn sections(&self) -> Self::SectionIterator<'_>;
}
```

To my eyes, this is unquestionably a simpler trait (and it fits what will likely become a fairly standard pattern).