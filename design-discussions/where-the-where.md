# Where does the where clause go?

## UPDATE

This document is retained for historical purposes. See the [Where the Where](./where-the-where-1.md) conclusion for the most up-to-date conversation.

## Summary

Proposed: to alter the syntax of where clauses on type aliases so that they appear *after* the value:

```
type StringMap<K> = BTreeMap<K, String>
where
    K: PartialOrd
```

This applies both in top-level modules and in trats (associated types, generic or otherwise).

## Background

The current syntax for where to place the "where clause" of a generic associated types is awkward. Consider this example ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=bdb55a5d5cb17e20d73e22a3f2db0e57)):

```rust
trait Iterable {
    type Iter<'a> where Self: 'a;

    fn iter(&self) -> Self::Iter<'_>;
}

impl<T> Iterable for Vec<T> {
    type Iter<'a>
    where 
        Self: 'a = <&'a [T] as IntoIterator>::IntoIter;

    fn iter(&self) -> Self::Iter<'_> {
        self.iter()
    }
}
```

Note the impl. Most people expect the impl to be written as follows (indeed, the author wrote it this way in the first draft):

```rust
impl Iterable for Vec<T> {
    type Iter<'a>  = <&'a [T] as Iterator>::Iter
    where 
        Self: 'a;

    fn iter(&self) -> Self::Iter<'_> {
        self.iter()
    }
}
```

However, this placement of the where clause is in fact rather inconsistent, since the `= <&'a [T] as Iterator>::Iter` is in some sense the "body" of the item.

The same current syntax is used for where clauses on type aliases ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=74eeed1795b693f238150f825a0e8438)):

```rust
type Foo<T> where T: Eq = Vec<T>;

fn main() { }
```

## Top-level type aliases

Currently, we accept where clauses in top-level type aliases, but they are deprecated (warning) and semi-ignored:

```
type StringMap<K> where
    K: PartialOrd
= BTreeMap<K, String>
```

Under this proposal, this syntax remains, but is deprecated. The newer syntax for type aliases (with `where` coming after the type) would remain feature gated until such time as we enforce the expected semantics.

## Alternatives

### Keep the current syntax.

In this case, we must settle the question of how we expect it to be formatted (surely not as I have shown it above).

```rust
impl<T> Iterable for Vec<T> {
    type Iter<'a> where Self: 'a 
        = <&'a [T] as IntoIterator>::IntoIter;

    fn iter(&self) -> Self::Iter<'_> {
        self.iter()
    }
}
```

### Accept either

What do we do if both are supplied?