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

### Example: Iter static

In the previous example, the lifetime parameter for `Output` was not related to the `self` parameter. Are there (realistic) examples where the associated type is applied to the lifetime parameter from `self` *but* the `where Self: 'a` is not desired?

There are some, but they rely on having "special knowledge" of the types that will be used in the impl, and they don't seem especially realistic. The reason is that, if you have a GAT with a lifetime parameter, it is likely that the GAT contains some data borrowed for that lifetime! But if you use the lifetime of `self`, that implies we are borrowing some data from `self` -- however, it doesn't *necessarily* imply that we are borrowing data of any particular type. Consider this example:

```rust
trait Message {
    type Data<'a>: Display;

    fn data<'b>(&'b mut self) -> Self::Data<'b>;

    fn default() -> Self::Data<'static>;
}

struct MyMessage<T> {
    text: String,
    payload: T,
}

impl<T> Message for MyMessage<T> {
    type Data<'a>: Display = &'a str;
    // No requirement that `T: 'a`!

    fn data<'b>(&'b mut self) -> Self::Data<'b> {
        // In here, we know that `T: 'b`
    }

    fn default() -> Self::Data<'static> {
        "Hello, world"        
    }
}
```

Here the `where T: 'a` requirement is not necessary, and may in fact be annoying when invoking `<MyMessage<T> as Message>::default()` (as it would then require that `T: 'static`).

Another possibility is that the usage of `<MyMessage<T> as Message>::Data<'static>` doesn't appear inside the trait definition, although it is hard to imagine exactly how one writes a useful function like that in practice.

## Alternatives

### Status quo

We ship with no default. This kind of locks in a box, because adding a default later would be a breaking change to existing impls that are affected by the default. since some of them may be using the associated types with a lifetime unrelated to `Self`. Note though that a sufficiently tailored default would only break code that was going to -- or perhaps *very likely to* -- not compile anyhow.

### Smart default: add `where Self: 'a` if the GAT is used with the lifetime from `&self` (and extend to other type parameters)

Analyze the types of methods within the trait definition. It a GAT is applied to a lifetime `'x`, examine the implied bounds of the method for bounds of the form `T: 'x`, where `T` is an input parameter to the trait. If we find such bounds on all methods for every use of the GAT, then add the corresponding default.

Consider the `LendingIterator` trait:

```rust
trait LendingIterator {
    type Item<'a>;

    fn next<'b>(&'b mut self) -> Self::Item<'b>;
}
```

Analyzing the closure body, we see that it contains `Self::Item<'b>` where `'b` is the lifetime of the `self` reference (e.g., `self: &'b Self` or `self: &'b mut Self`). The implied bounds of this method contain `Self: 'b`. Since there is only one use of `Self::Item<'b>`, and the implied bound `Self: 'b` applies in that case, then we add the default `where Self: 'a` to the GAT. 

This check is a fairly simple syntactic check, though not necessarily easy to explain. It would accept all the examples that appear in this document, including the example with `fn default() -> Self::Data<'static>` (in that case, the default is not triggered, because we found a use of `Data` that is applied to a lifetime for which no implied bound applies). The only case where this default behaves *incorrectly* is the case where all uses of `Self::Data` that appear within the trait need the default, but there are uses outside the trait that do not (I couldn't come up with a realistic example of how to do this usefully).

#### Extending to other type parameters

The inference can be extended naturally beyond `self` to other type parameters. Therefore this example:

```rust
trait Parser<Input> {
    type Output<'i>;

    fn get<'input>(&mut self, i: &'input Input) -> Self::Output<'input>;
}
```

would infer a `where Input: 'i` bound on `type Output<'i>`.

Similarly:

```rust
trait Parser<Input> {
    type Output<'i>;

    fn get(&mut self, i: &'input Input) -> Self::Output<'input>;
}
```

would infer a `where Input: 'i` bound on `type Output<'i>`.

#### Avoiding the default

If this default is truly not desired, there is a workaround: one can declare a supertrait that contains just the associated type. For example:

```rust
trait IterType {
    type Iter<'b>;
}

trait LendingIterator: IterType {
    fn next(&mut self) -> Self::Iter<'_>;
}
```

This workaround is not especially obvious, however.

#### Related precedent

We used to require `T: 'a` bounds in structs:

```rust
struct Foo<'a, T> {
    x: &'a T
}
```

but as of [RFC 2093] we infer such bounds from the fields in the struct body. In this case, if we do come up with a default rule, we are essentially inferring the presence of such bounds by usages of the associated type within the trait definition.

[RFC 2093]: https://rust-lang.github.io/rfcs/2093-infer-outlives.html

## Recommendation

Niko's recommendation is to use the "smart defaults". Why? They basically always do the right thing, thus contributing to [supportive](https://rustacean-principles.netlify.app/how_rust_empowers/supportive.html), at the cost of (theoretical) [versatility](https://rustacean-principles.netlify.app/how_rust_empowers/versatile.html). This seems like the right trade-off to me.

The counterargument would be: the rules are sufficiently complex, we can potentially add this later, and people are going to be surprised by this default when it "goes wrong" for them. It would be hard, but not impossible, to add a tailored error message for cases where the `where T: 'b` check fails.

Not sure about Jack's opinion. =)

## Appendix A: Ruled out alternatives

### Special syntax

We could use the `'self` "keyword", permitted only in GATs, to indicate "a lifetime with the where clause `where Self: 'self`". The `LendingIterator` trait would therefore be written

```rust
trait LendingIterator {
    type Item<'self>;

    fn next(&mut self) -> Self::Item<'_>;
}
```

*Forwards compatibility note:* This option could be added later; note also that `'self` is not currently valid.

**Why not?** `'self` is an awfully suggestive syntax. It may be useful for things like self-referential structs. This just doesn't important enough.

### Force people to write `where Self: 'a`

To buy time, we could force people to write `where Self: 'a`, so that we can later allow it to be elided. This unfortunately would eliminate a number of valid use cases for GATs (though they would later be supported).

**Why not?** Rules out a number of useful cases.

### Dumb default: Always default to `where Self: 'a`

The most obvious default is to add `where Self: 'a` to the where clause list for any GAT with a lifetime parameter `'a`, but that seems too crude. It will rule out all existing cases unless we add some form of "opt-out" syntax, for which we have no real precedent.


**Why not?** Rules out a number of useful cases.

## Appendix B: Considerations

* [How 'obvious' are the rules?](https://github.com/rust-lang/rust/issues/87479#issuecomment-890111937)

## Appendix C: Other examples

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

## Appendix D: Examples

We go through several examples and document whether and why bounds are required.

### Default bounds in return position

```rust
#![feature(generic_associated_types)]

trait Foo {
    type Item<'me>
    where
        Self: 'me; // <-- Required

    fn push_back<'a>(&'a mut self) -> Self::Item<'a>;
}

fn main() { }
```

The bound here is required because:

* `push_back` returns a value of type `Self::Item<'a>`:
    * the `&'a mut self` parameter implies that `Self: 'a`
    * therefore, we require this bound be written on the trait

### No default bounds in argument position

```rust
#![feature(generic_associated_types)]

trait Foo {
    type Item<'me>; 
    fn push_back1<'a>(&'a mut self, arg: Self::Item<'a>);
    fn push_back2<'a>(&'a mut self, arg: &mut Self::Item<'a>);
}

fn main() { }
```

No bounds here are required because `Self::Item` only appears in argument position.

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

You may have noticed that `Item` didn't need to be a GAT at all here -- in fact, the same logic would apply to a trait with no lifetime parameters. However, adding the rules that users write the bounds explicitly after the fact is not backwards compatible. We found crates that would stop compiling, such as gimli:

https://github.com/gimli-rs/object/blob/0a38064531fef4ddbaf93770a3551d333338980e/src/read/traits.rs#L24

```rust
/// An object file.
pub trait Object<'data: 'file, 'file>: read::private::Sealed {
    ...
    type SectionIterator: Iterator<Item = Self::Section>;
    // needs `Self: 'file`
    ...
    fn sections(&'file self) -> Self::SectionIterator;
}
```

Interestingly, if you look closely at the trait header, you can see that it is `'data: 'file`. This `'data` lifetime turns out to be the lifetime of data that appears in `Self`. So e.g. an [example impl](https://github.com/gimli-rs/object/blob/0b76070f1281ebfad5b6b79c74f0c2508e5ee85c/src/read/coff/file.rs#L54) looks like this:

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