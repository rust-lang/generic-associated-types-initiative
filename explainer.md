# 📚 Explainer

> The "explainer" is "end-user readable" documentation that explains how to use the feature being deveoped by this initiative.
> If you want to experiment with the feature, you've come to the right place.
> Until the feature enters "feature complete" form, the explainer should be considered a work-in-progress.

## ⚠️ Suboptimal syntax alert ⚠️

In general, the GAT feature is currently in a "minimum viable product" phase. That means that there are a number of places where the syntax is kind of clumsy, and more annotations are required than we would like to have. The explainer describes the feature that we are aiming to stabilize but you can see the [Future Directions](./explainer/future_directions.md) section for some notes on the kinds of changes we would like to make.

## The problem in a nutshell

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

