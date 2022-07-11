# Defining and implementing the Iterable trait with GATs

To express traits like `Iterable`, we can make use generic associated types -- that is, associated types with generic parameters. Here is the complete `Iterable` trait:

```rust
trait Iterable {
    // Type of item yielded up; will be a reference into `Self`.
    type Item<'collection>;

    // Type of iterator we return. Will return `Self::Item` elements.
    type Iterator<'collection>: Iterator<Item = Self::Item<'collection>>;

    fn iter(&self) -> Self::Iterator<'_>;
    //      ^                        ^^
    //
    // Returns a `Self::Iter` derived from `self`.
}
```

Let's walk through it piece by piece...

* We added a `'collection` parameter to `Item`. This represents "the specific collection that the `Item` is borrowed from" (or, if you prefer, the lifetime for which that collection is borrowed). 
* The same `'collection` parameter is added to `Iterator`, indicating the collection that the iterator borrows its items from.
* In the `iter` method, the value of `'collection` comes from `self`, indicating that `iter` returns an `Iterator` linked to `self`.

## Implementing the trait

Let's write an implementation of this trait. We'll implement it for the `Vec<T>` type; a `&Vec<T>` can be coerced into a `&[T]` slice, so we can re-use the slice `Iter` that we defined before (the [playground] link includes an impl of `Iterable` for `[T]` as well, but we'll use `Vec` here because it's more convenient).

```rust
// from before
struct Iter<'c, T> {
    data: &'c [T],
}

impl<T> Iterable for Vec<T> {
    type Item<'c> = &'c T;
    
    type Iterator<'c> = Iter<'c, T>;

    fn iter<'c>(&'c self) -> Self::Iterator<'c> {
        Iter { data: self }
    }
}
```

## Invoking it

Now that we have the `Iterable` trait, we can reference it in our "count twice" function.

```rust
fn count_twice<I: Iterable>(collection: &I) {
    let mut count = 0;
    for _ in collection.iter() {
        count += 1;
    }

    for elem in collection.iter() {
        process(elem, count);
    }
}
```

and we can invoke that by writing code like `count_twice(&vec![1, 2, 3, 4, 5, 6])`.

[Play with the code from this section on the Rust playground.][playground]