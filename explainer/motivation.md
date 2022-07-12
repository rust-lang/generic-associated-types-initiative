# Why GATs?

{{#include ../badges/nightly.md}} {{#include ../badges/stabilization-96709.md}}

As the name suggests, *generic* associated types are an extension that permit associated types to have generic parameters. This allows associated types to capture types that may include generic parameters that came from a trait method. These can be lifetime or type parameters.

To see why they are useful, we'll start by looking at the `Iterator` trait, which defines a single iterator over some items. This trait works great with plain associated types. We'll then consider how we could create an `Iterable` trait, that defines a collection that can be iterated many times -- this trait requires generic associated types.

## Example: Iterator

Associated types in a trait are used to capture types that appear in methods but are determined based on the impl that is chosen (i.e., by the type implementing the trait) rather than being specified from the outside. The simplest example is the `Iterator` trait:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Because the `Iterator` trait defines the `Item` type as an associated type, that implies that every type which implements `Iterator` will also specify what sort of `Item` it generates.

As an example, imagine I have an iterator for items in a slice `&'c [T]`...

```rust
struct Iter<'c, T> {
    data: &'c [T],
}

impl<'c, T> Iterator for Iter<'c, T> {
    type Item = &'c T;

    fn next(&mut self) -> Option<Self::Item> {
        if let Some((prefix_elem, suffix)) = self.data.split_first() {
            self.data = suffix;
            Some(prefix_elem)
        } else {
            None
        }
    }
}
```

...the impl specifies that the type of this iterator is `&'c T`.

## Extending to Iterable

Associated types are useful, but on their own they are often insufficient to capture the patterns we would like to write as a trait. Often, this occurs when the associated type wants to include data borrowed from `self` or some other parameter.

The `Iterator` trait is great, but if you write a generic function that takes an `Iterator`, that iterator can only be iterated over a single time. Imagine a function that wanted to iterate once to find a total count and then *again* to process each item, this time knowing the full count:

```rust
fn count_twice<I: Iterator>(iterator: I) {
    let mut count = 0;
    for _ in iterator {
        count += 1;
    }

    // Error: iterator already used
    for elem in iterator {
        process(elem, count);
    }
}
```

Of course, most Rust types in the standard library have a method called `iter` that is exactly what we want (e.g., [`[T]::iter`](https://doc.rust-lang.org/std/primitive.slice.html#method.iter)). Given an `&'i self`, these functions return a fresh iterator that yields up `&'i T` references into the collection. Because they take a `&self`, they can be called as many times as we want. So, if we knew which kind of collection we had, we could easily write `count_twice` to take a `[T]` or a `HashMap<T>` or whatever. But what we if want to write it generically, so it works across *any* collection? That turns out to be impossible to do nicely with associated types.

To see why, let's try to write the code and see where we get a stuck. We'll start by defining an `Iterable` trait:

```rust
trait Iterable {
    // Type of item yielded up; will be a reference into `Self`.
    type Item;

    // Type of iterator we return. Will return `Self::Item` elements.
    type Iter: Iterator<Item = Self::Item>;


    fn iter<'c>(&'c self) -> Self::Iter;
}
```

But when we try to write the impl, we run into a problem:

```rust
impl<T> Iterable for [T] {
    type Item = &'hmm T;
    //           ^^^^ what lifetime to use here?

    type Iter = Iter<'hmm, T>;
    //               ^^^^ what lifetime to use here?

    fn iter<'c>(&'c self) -> Self::Iter {
        //       ^^ THIS is the lifetime we want, but it's not in scope!
        Iter { data: self }
    }
}
```

You see the problem? In the case of `Iterator`, the `Self` type was `&'c [T]` and the `Item` type was `&'c T`. Since both `'c` and `T` appeared at the impl header level, both of those generic parameters were in scope and usable in the associated type. But for `Iterable`, we still want to yield references like `&'c T`, but `'c` is not declared on the impl -- it's specific to the call to `iter`. 

The fact that the lifetime parameter `'c` is declared on the method is not just a minor detail. It is exactly what allows something to be iterated many times; that is, the thing we are trying to capture with the `Iterable` trait. It means that the borrow you get when you call `iter()` only has to last as long as this specific call to `iter`. If you call `iter` multiple times, they can be instantiated with distinct borrows. (In contrast, if `'c` were declared at the impl scope, the borrow would last across *all* calls to any method in `Iterable`.)
