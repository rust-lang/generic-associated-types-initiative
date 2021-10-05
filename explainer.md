# 📚 Explainer

> The "explainer" is "end-user readable" documentation that explains how to use the feature being deveoped by this initiative.
> If you want to experiment with the feature, you've come to the right place.
> Until the feature enters "feature complete" form, the explainer should be considered a work-in-progress.

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

With generic associated types, associated types can have generic parameters. This makes it possible to model the `Iterable` trait:

```rust
trait Iterable {
    type Item<'a>;
    type Iterator<'a>: Iterator<Item = Self::Item<'a>>;
    //           ^^^^^ this changed

    fn iter<'a>(&'a self) -> Self::Iterator<'a>;
    //                                     ^^^^ this changed
}
```

Given this trait definition, we can implement `Iterable` like so:

```rust
impl<T> Iterable for Vec<T> {
    type Iterator<'a> = Iter<'a, T>;

    fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { vec: self, position: 0 }
    }
}
```

## Bounds

Suppose I want to write a function that accepts any iterable and creates a vector by cloning its elements. I can now write such a thing like this:

```rust
fn into_vec<T>(
    iterable: &impl for<'a> Iterable<Item<'a> = &'a T>,
) -> Vec<T>
where
    T: Clone
{
    let mut out = vec![];
    for elem in iterable.iter() {
        out.push(elem.clone());
    }
    out
}
```

Let's look at this function more closely. The most interesting part is the type of the `iterable` parameter:

```rust
iterable: &impl for<'a> Iterable<Item<'a> = &'a T>,
//              ^^^^^^^          ^^^^^^^^
```

What this type says is "some `Iterable` which yields up values of type `&'a T`". The `'a` here represents the lifetime of the collection you pass in.

You might be surprised by the `for<'a>` here. What this is saying is, given some 

### Suboptimal syntax alert

It is clear that this syntax is suboptimal. Generic associated types in this form remain a bit clumsy and we expect to invest effort in finding better syntax for expressing this sort of bound! Stay tuned.

## Bounds for specific lifetimes

In the previous