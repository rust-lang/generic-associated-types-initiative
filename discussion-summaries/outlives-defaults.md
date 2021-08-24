# Outlives defaults

## Links

* Discussion issue: [#87479](https://github.com/rust-lang/rust/issues/87479)

## Summary

We are moving towards stabilizing GATs (tracking issue: [#44265]) but there is one major ergonomic hurdle that we should decide how to manage before we go forward. In particular, a great many GAT use cases require a surprising where clause to be well-typed; this typically has the form `where Self: 'a`. In "English", this states that the GAT can only be used with some lifetime `'a` that could've been used to borrow the `Self` type. This is because GATs are frequently used to return data owned by the `Self` type. It might be useful if we were to create some rules to add this rule by default. Once we stabilize, changing defaults will be more difficult, and could require an edition, therefore it's better to evaluate the rules now.

[#44265]: https://github.com/rust-lang/rust/issues/44265

## I have an opinion! What should I do?

To make this decision in an informed way, **what we need most are real-world examples and experience reports**. If you are experimenting with GATs, for example, how often do you use `where Self: 'a` and how did you find out that it is necessary? Would the default proposals described below work for you? If not, can you describe the trait so we can understand why they would not work?

Of great use would be example usages that do NOT require `where Self: 'a`. It'd be good to be able to evaluate the various defaulting schemes and see whether they would interfere with the trait. Knowing the trait and a rough sketch of the impls would be helpful.

## Background: what where clause now?

Consider the typical "lending iterator" example. The idea here is to have an iterator that produces values that may have references into the **iterator itself** (as opposed to references into the collection being iterated over). In other words, given a `next` method like `fn next<'a>(&'a mut self)`, the returned items have to be able to reference `'a`. The typical `Iterator` trait cannot express that, but GATs can:

```rust
trait LendingIterator {
    type Item<'a>;

    fn next<'b>(&'b mut self) -> Self::Item<'b>;
}
```

Unfortunately, this trait definition turns out to be not quite right in practice. Consider an example like this, an iterator that yields a reference to the same item over and over again (note that it owns the item it is referencing):

```rust
struct RefOnce<T> {
    my_data: T    
}

impl<T> LendingIterator for RefOnce<T> {
    type Item<'a> where Self: 'a = &'a T;

    fn next<'b>(&'b mut self) -> Self::Item<'b> {
        &self.my_data
    }
}
```

Here, the type `type Item<'a> = &'a T` declaration is actually illegal. Why is that? The assumption when authoring the trait was that `'a` would always be the lifetime of the `self` reference in the `next` function, of course, but that is not in fact *required*. People can reference `Item` with any lifetime they want. For example, what if somebody wrote the type `<SomeType<T> as LendingIterator>::Item<'static>`? In this case, `T: 'static` would have to be true, but `T` may in fact contain borrowed references. This is why the compiler gives you a "T may not outlive `'a`" error ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=821e30ee635326a22fc19cd940bbaf62)). 

We can encode the constraint that "`'a` is meant to be the lifetime of the `self` reference" by adding a `where Self: 'a` clause to the `type Item` declaration. This is saying "you can only use a `'a` that could be a reference to `Self`". If you make this change, you'll find that the code compiles ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=87cb2430ee76ece77499d3c6605874df)): 

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;

    fn next<'b>(&'b mut self) -> Self::Item<'b>;
}
```

## When would you NOT want the where clause `Self: 'a`?

If the associated type cannot refer to data that comes from the `Self` type, then the `where Self: 'a` is unnecessary, and is in fact somewhat constraining.

### Example: Output doesn't borrow from Self

In the `Parser` trait, the `Output` does not ultimately contain data borrowed from `self`:

```rust
trait Parser {
    type Output<'a>;
    fn parse<'a>(&mut self, data: &'a [u8]) -> Self::Output<'a>;
}
```

If you were to implement `Parser` for some reference type (in this case, `&'b Dummy`) you can now set `Output` to something that has no relation to `'b`:

```rust
impl<'b> Parser for &'b Dummy {
    type Output<'a> = &'a [u8];

    fn parse<'a>(&mut self, data: &'a [u8]) -> Self::Output<'a> {
        data 
    }
}
```

Note that you would need a similar `where` clause if you were going to have a setup like:

```rust
trait Transform<Input> {
    type Output<'a>
    where
        Input: 'i;

    fn transform<'i>(&mut self: &'i Input) -> Self::Output<'i>;
}
```

### Example: Ruma

```rust
// Every endpoint in the Matrix REST API has two request and response types in Ruma, one Incoming
// (used for deserialization) and out Outgoing (used for serialization). To avoid annoying clones when
// sending a request, most non-copy fields in the outgoing structs are references.
//
// The request traits have an associated type for the corresponding response type so things can be
// matched up properly.
pub trait IncomingRequest: Sized {
    // This is the current definition of the associated type I'd like to make generic.
    type OutgoingResponse: OutgoingResponse;
    // AFAICT adding a lifetime parameter is all I need.
    type OutgoingResponse<'a>: OutgoingResponse;

    // Other trait members... (not using Self::OutgoingResponse)
}
```

[full definition](https://docs.rs/ruma-api/0.17.1/ruma_api/trait.IncomingRequest.html)

### Example: something :) 

[link](https://github.com/rust-lang/rust/issues/87479#issuecomment-890760913)

### Example: Sanakirja


## Alternatives

### Status quo

We ship with no default. This kind of locks in a box, because adding a default later would be a breaking change to existing impls that are affected by the default. since some of them may be using the associated types with a lifetime unrelated to `Self`. Note though that a sufficiently tailored default would only break code that was going to -- or perhaps *very likely to* -- not compile anyhow.

### Force people to write `where Self: 'a`

To buy time, we could force people to write `where Self: 'a`, so that we can later allow it to be elided. This unfortunately would eliminate a number of valid use cases for GATs (though they would later be supported).

### Special syntax

We could use the `'self` "keyword", permitted only in GATs, to indicate "a lifetime with the where clause `where Self: 'self`". The `LendingIterator` trait would therefore be written

```rust
trait LendingIterator {
    type Item<'self>;

    fn next(&mut self) -> Self::Item<'_>;
}
```

*Forwards compatibility note:* This option could be added later; note also that `'self` is not currently valid.

### Smart default: add `where Self: 'a` if the GAT is used with the lifetime from `&self` 

Consider the `LendingIterator` trait:

```rust
trait LendingIterator {
    type Item<'a>;

    fn next<'b>(&'b mut self) -> Self::Item<'b>;
}
```

Analyzing the closure body, we see that it contains `Self::Item<'b>` where `'b` is the lifetime of the `self` reference (e.g., `self: &'b Self` or `self: &'b mut Self`). Because we find at least one such method, we can deduce that this associated type refers to contents potentially borrowed from `self`, and therefore `where Self: 'a` is appropriate and added as a default.

This check is a fairly simple syntactic check, though not necessarily easy to explain. There are no known "false defaults" (defaults where you wouldn't actually want it). It would accept all examples given in the "not desired" list. 

If it is truly not desired, there is a workaround: one can eschew method form and write `fn next(this: &mut self)`, for example. 

## Ruled out alternatives

### Dumb default: Always default to `where Self: 'a`

The most obvious default is to add `where Self: 'a` to the where clause list for any GAT with a lifetime parameter `'a`, but that seems too crude. It will rule out all existing cases unless we add some form of "opt-out" syntax, for which we have no real precedent.


## Considerations

* [How 'obvious' are the rules?](https://github.com/rust-lang/rust/issues/87479#issuecomment-890111937)
