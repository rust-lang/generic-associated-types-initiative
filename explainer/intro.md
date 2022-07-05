# Introduction

## Role of associated types

Associated types in a trait are used to capture types that appear in methods but are determined based on the impl that is chosen (i.e., by the type implementing the trait) rather than being specified from the outside. 

### Example: Iterator

The simplest example is the `Iterator` trait:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Because the `Iterator` trait defines the `Item` type as an associated type, that implies that every type which implements `Iterator` will also specify what sort of `Item` it generates.

As an example, imagine I have an iterator for items in a slice `&'me [T]`...

```rust
struct Iter<'me, T> {
    data: &'me [T],
}

impl<'me> Iterator for Iter<'me, T> {
    type Item = &'me T;

    fn next(&mut self) -> Option<Self::Item> {
        if let Some(prefix_elem, suffix) = self.data.split_first() {
            self.data = suffix
            Some(prefix_elem)
        } else {
            None
        }
    }
}
```

...the impl specifies that the type of this iterator is `&'me T`.

## Limits of "plain" associated types

Associated types are useful, but on their own they are often insufficient to capture the patterns we would like to write as a trait. Often, this occurs when the associated type wants to include data borrowed from `self` or some other parameter.

### Example: Iterable

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


    fn iter<'me>(&'me self) -> Self::Iter;
}
```

But when we try to write the impl, we run into a problem:

```rust
impl<T> Iterable for [T] {
    type Item = &'hmm T;
    //           ^^^^ what lifetime to use here?

    type Iter = Iter<'hmm, T>;
    //               ^^^^ what lifetime to use here?

    fn iter<'me>(&'me self) -> Self::Iter {
        //        ^^^ THIS is the lifetime we want, but it's not in scope!
        Iter { data: self }
    }
}
```

You see the problem? In the case of `Iterator`, the `Self` type was `&'me [T]` and the `Item` type was `&'me T`. Since both `'me` and `T` appeared at the impl header level, both of those generic parameters were in scope and usable in the associated type. But for `Iterable`, we still want to yield references like `&'me T`, but `'me` is not declared on the impl -- it's specific to the call to `impl`. 

The fact that the lifetime parameter `'me` is declared on the method is not just a minor detail. It is exactly what allows something to be iterated many times; that is, the thing we are trying to capture with the `Iterable` trait. It means that the borrow you get when you call `iter()` only has to last as long as this specific call to `iter`. If you call `iter` multiple times, they can be instantiated with distinct borrows. (In contrast, if `'me` were declared at the impl scope, the borrow would last across *all* calls to any method in `Iterable`.)

## Enter generic associated types

To express traits like `Iterable`, we can make use generic associated types -- that is, associated types with generic parameters. In this case, we add the lifetime parameter `'me`, which refers to the lifetime of the `self` argument given in the call to `next`:

> ⚠️ This trait as implemented doesn't work in the [MVP](./mvp.md). See the MVP section for more information. 

```rust
trait Iterable {
    // Type of item yielded up; will be a reference into `Self`.
    type Item<'me>;

    // Type of iterator we return. Will return `Self::Item` elements.
    type Iter<'me>: Iterator<Item = Self::Item>;

    fn iter<'me>(&'me self) -> Self::Iter<'me>;
    //            ^^^                     ^^^
    //
    // Connect the type of `Iter` to `'me`.
}

```

Now I can write the corresponding impl, which works:

```rust
impl<T> Iterable for [T] {
    type Item<'me> = &'me T;
    type Iter<'me> = Iter<'me, T>;
    fn iter<'me>(&'me self) -> Self::Iter<'me> {
        Iter { data: self }
    }
}
```

## What does a GAT *mean*?

Whereas the type `<SomeType as Iterator>::Item` referred to "the `Item` type defined by the iterator `SomeType`", the type `<SomeType as Iterable>::Item<'x>` refers to "the `Item` type when `iter` is called with a `'x` lifetime".



To describe patterns like `Iterable`, we need We *can* describe the `Iterable` pattern by using a 


The `iter`](https://doc.rust-lang.org/std/primitive.slice.html#method.iter) method is meant to work like [`iter`](https://doc.rust-lang.org/std/primitive.slice.html#method.iter) on `[T]` types does -- each time you ca

Sometimes, you have a type that is conceptually an associated type (i.e., determined by which impl is chosen) but which needs to include generic parameters from the method itself. Often this occurs with lifetime parameters, whenever you wish to have a method that returns an associated type that may contain data borrowed from `Self`. Consider trying to write a trait `Iterable`, which would support an [`iter`](https://doc.rust-lang.org/std/primitive.slice.html#method.iter) method


If you want to iterate over something in Rust, you might write the following:

```rust
let v = vec![1, 2, 3];
for elem in v.iter() { ... }
```

The `iter` method returns an iterator of references into the items in the vector. Conceptually, it looks something like this (the actual implementation works differently):

```rust
impl<T> Vec<T> {
    fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { vec: self, position: 0 }
    }
}
```

where `Iter` is a struct that implements the `Iterator` trait:

```rust
pub struct Iter<'a, T> {
    vec: &'a Vec<T>,
    position: usize,
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<&'a T> {
        let i = self.position;
        if i < self.vec.len() {
            self.position += 1;
            Some(&self.vec[i])
        } else {
            None
        }
    }
}
```

Rust's other collections also have similar `iter` methods that work the same way. You might wonder, why not have an `Iterable` trait, that encapsulates this pattern? Something like this:

```rust
trait Iterable {
    type Item;
    type Iterator: Iterator<Item = Self::Item>;

    fn iter<'a>(&'a self) -> Self::Iterator;
}
```

Well, if you look closely at this trait, you'll see that it does't work. The type that an impl needs to provide for `Iterator` needs to be able to reference `'a`, the lifetime of the `self` parameter. But currently, it can't! That lifetime is not in scope:

```rust
impl<T> Iterable for Vec<T> {
    type Item = &'a T;
    //           ^^ 'a is not in scope here!
    type Iterator = Iter<'a, T>;
    //                   ^^ 'a is not in scope here!

    fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { vec: self, position: 0 }
    }
}
```

## Enter: generic associated types

With generic associated types, associated types can have generic parameters. This makes it possible to model the `Iterable` trait by having the `Iterator` type take a generic parameter, `'a`:

```rust
trait Iterable {
    type Item<'a>
    where
        Self: 'a; // <-- see the "required bounds" section for more information

    type Iterator<'a>: Iterator<Item = Self::Item<'a>>
    where
        Self: 'a; // <-- see the "required bounds" section for more information

    fn iter<'a>(&'a self) -> Self::Iterator<'a>;
}
```

Given this trait definition, we can implement `Iterable` like so:

```rust
impl<T> Iterable for Vec<T> {
    type Item<'a> = &'a T
    where
        T: 'a; // <-- see the "required bounds" section for more information

    type Iterator<'a> = Iter<'a, T>
    where
        T: 'a; // <-- see the "required bounds" section for more information

    fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { vec: self, position: 0 }
    }
}
```

