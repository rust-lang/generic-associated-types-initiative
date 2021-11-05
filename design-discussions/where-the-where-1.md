# 💬 Where does the where clause go?

This is write-up of the conclusion to the [where does the where clause go?] question. To read more background, see the [links](#links-to-older-discussions) section below.

## Conclusion

### Where clauses in generic associated types comes after value/binding

Example:

```rust
trait MyTrait {
    type MyType<T>: Iterator
    where
        T: Ord;
}
    
impl MyTrait for MyOtherType {
    type MyType<T> = MyIterator
    where
        T: Ord;
}
```

Effectively the `= type` in the impl replaces the `: Bound` from the declaration with the value that has to meet those bounds.

### Later phase: type aliases

Type aliases will eventually be aligned to this syntax as well:

```rust
type MyType<T> = Vec<T> where T: Ord;
```

Currently, however, where clauses on type aliases are ignored, so we will not stabilize this new syntax until they have the meaning we want.

### Suggestions for users who put the where clause in the wrong place

We will parse where clauses in both positions and suggest to users that they be moved:


```rust
impl MyTrait for MyOtherType {
    type MyType<T>
    where
        T: Ord
    = MyIterator;
}
```

Gets an error with a suggested rewrite. The compiler proceeds "as if" the where clauses had been written after the `= Type`.

### Where clause syntax for trait aliases will have to be revisited

As described in the FAQ below, we currently support trait alias syntax like

```rust
trait ReverseEq<T> = where T: PartialEq<Self>;
```

This syntax will be removed. Although its capabilities could be useful, it is also quite confusing (the placement of the `where` is a subtle distinction), and not clearly needed. If we find that we do want it, we can add in a similar syntax later, but hopefully in a way that is more broadly consistent with the language.

## Discussion and FAQ

### But isn't it inconsistent with other trait items to put the where clauses *before* the `=`?

From one perspective, yes. One can view the value of an associated type as its "body", and the where clauses typically come before the "body" of an item. Put another way, typically you can "copy and paste" the impl and then add some text to the end of each item to specify its value: but with this syntax, you have to edit the "middle" of an associated type to specify its value.

The analogy of an associated type value to a function body, however, is somewhat flawed. The value of an associated type needs to be considered part of the "signature", or public facing, part of the impl. Consider: you can change the body of a function and be certain that your callees will still compile, but you cannot do the same for the value of an associated type.

Given this perspective, when you copy the associated type from the trait to the impl, you are "completing" the signature that was left incomplete by the trait. Moreover, to do so, you replace the `: Bound1 + Bound2` list (which constraints what kinds of types the impl might use) with a specific type, thus making it more specific.

### What about a more purely syntactic point-of-view? What is more consistent?

There is precedent that the placement of the where clause has less to do with the logical role that it plays and more to do with other factors, like whether it is followed by a braced list of items:

* With `struct Foo<T> where T: Ord { t: T }`, the "body" of the struct is its fields, and the where clause comes first.
* But we write `struct Foo<T>(T) where T: Ord`, thus placing the "body" (the fields `(T)`) first and the where clause second. Moreover, we initially implemented the grammar `struct Foo<T> where T: Ord (T)` but this was deemed so obviously confusing that it was [changed with little discussion](https://github.com/rust-lang/rust/issues/17904#issuecomment-58603749).

As further evidence that this syntax is inconsistent with Rust's traditions, placing the where clauses before the `= ty` makes it objectively hard to determine how to run rustfmt in a way that feels natural. rustfmt handles `where` by putting the `where` onto its own line, with one line per where clause. This structure works for existing Rust items because where clauses are always either following by nothing (tuple structs) or by a braced (`{}`) list of items (e.g., struct fields, fn body, etc). That opening `{` can therefore go on its own line. This `where` clause formatting does not work well with `=`.

The idea of having where clauses come at the "end" of the signature is also supported by the [original RFC](https://rust-lang.github.io/rfcs/0135-where.html#readability), which motivated where clauses in part by describing how they allow you to treat the precise bounds as "secondary" to the "important stuff":

> If I may step aside from the "impersonal voice" of the RFC for a moment, I personally find that when writing generic code it is helpful to focus on the types and signatures, and come to the bounds later. Where clauses help to separate these distinctions. Naturally, your mileage may vary. - nmatsakis

In the case of an impl specifying the value for an associated type, the "important stuff" the value of the associated type.

### What about trait aliases, don't they distinguish where clause placement?

As currently implemented, trait aliases have two distinct possible placements for where clauses, which effectively distinguishes between a *where clause* (which must be proven true in order to use the alias) and an *implied bound* (which is part of what the alias expands to). One can write:

```rust
trait Foo<T: Debug> = Bar<T> + Baz<T>
```

in which case `where X: Foo<Y>` is only legal if `Y: Debug` is known from some other place. This is roughly equivalent to a trait like so:

```rust
trait Foo1<T: Debug>: Bar<T> + Baz<T> { }
```

The clause `where X: Foo1<Y>` is also only valid when `Y: Debug` is known. This is in contrast to the "supertraits" `Bar<Y>` and `Baz<Y>`, which are implied by `X: Foo1<Y>` ("supertraits" are also sometimes called "implied bounds").

Alternatively, one can include the where clause in the "value" of the trait alias like so:

```rust
trait ReverseEq<T> = where T: PartialEq<Self>;
```

In this case, `where X: ReverseEq<Y>` is equivalent to `Y: PartialEq<X>`. There is no "equivalent trait" for usage like this; the `T: PartialEq<Self>` effectively acts like a supertrait or implied bound.

Our decision was that this is a subtle distinction and that using the placement of the where clause was not a great way to make it.

### Is that trait alias syntax consistent with the rest of the language?

Not really. There are other places in the language that could benefit from a similar flexibility around implied bounds. For example, one could imagine wanting to have an associated type `T::MyType<Y>` where it is known that `Y: PartialEq<T::MyType<Y>>`, but this cannot be readily written with today's syntax:

```rust
trait MyTrait {
    type MyType<T>: PartialEq<T>;
    //              ^^^^^^^^^ not what we wanted
}
```

We decided that if we were going to offer that capability, we should find a way to offer it more generally, and hopefully with more clarity than putting the where clause before or after the `=`. As we have seen, where clauses for different kinds of items can be rather variable in their placement, so it is not clear that all users will recognize that distinction and understand it (many present in the meeting were surprised by the distinction as well).

Alternatively, the implied bounds proposal goes another way, turning most where clauses into implied bounds by default!

### Why do you even need where clauses in the impl anyway?

Given that the where clauses appear in the trait, you might wonder why they are needed in the impl anyway. After all, the impl could just assume that the trait bounds are met when checking the value of the associated type for validity, making the whole issue moot.

This would however be inconsistent with other sorts of items, which do require users to copy over the bounds from the trait. Furthermore, we have discussed the idea of allowing impls to relax the bounds from those described in the trait if they are not needed in the impl -- this came up most recently in the context of allowing impls to forego the `unsafe` keyword for `unsafe fn` declared in the trait if the fn in the impl body is completely safe. This could then even be relied upon by people invoking the method who know the precise impl they will be using.

In short, this might be a reasonable choice to make, but we should make it uniformly, and it shuts down the direction of using the lack of bounds in the impl as a kind of signal.

### Why not change type alias notation too?

Top-level type aliases currently parse with the where clause before the `=`:

```rust
type Foo<T> where T: Ord = T;
```

If you try that above example, however, you will find that you get a warning: this is because the `where T: Ord` is completely ignored! This is an implementation limitation in the way the current compiler eagerly expands type aliases. Moving the placement of where clauses actually gives us an opportunity to change this behavior without breaking any existing code, which is nice. It will however require some kind of opt-in (such as a `cargo fix` run) to migrate existing code that uses where clauses in the "ignored place" to the new format.

## Links to older discussions

* [Implementation PR](https://github.com/rust-lang/rust/pull/90076)
* [Design document](https://rust-lang.github.io/generic-associated-types-initiative/design-discussions/where-the-where.html)
* [Design meeting minutes](https://github.com/rust-lang/lang-team/blob/master/design-meeting-minutes/2021-10-13-where-the-where.md)
* [Rust issue](https://github.com/rust-lang/rust/issues/89122)