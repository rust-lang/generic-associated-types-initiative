# Should GATs only support lifetime parameters?

## Question

One possibility is to only stabilize GATs with lifetime parameters:

```rust
trait Iterable {
    type Item<'a>;
}
```

## Conclusion