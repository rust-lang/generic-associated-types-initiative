# Minimum Viable Product

The first phase of GATs is to stabilize a "MVP". This MVP contains a number of known shortcomings:

The following section details some of the workarounds required to get the code shown in the explainer to work.

## Workaround: where self bounds

The `Iterable` trait shown in the [explainer intro](./intro.md) needs to have `where Self: 'me` bounds added explicitly:

```rust
trait Iterable {
    type Item<'me>;
    where
        Self: 'me; // <-- bound

    type Iter<'me>: Iterator<Item = Self::Item>
    where
        Self: 'me; // <-- bound

    fn iter<'me>(&'me self) -> Self::Iter<'me>;
}
```

The compiler will dwetect that these bounds are missing and require that you type them. **The explainer intro is written to assume these bounds are added as a default, but this is not a settled question.** The current compromise gives us room to decide later whether to add these bounds by deafult or not; we may find another solution to the problem. See the ["outlives default" design discussion](../design-discussions/outlives-defaults.md) for more background.

