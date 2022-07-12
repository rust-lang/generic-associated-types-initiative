# Concern: we should stabilize lifetimes only

## Summary

This suggestion stems from the concern that [GATs give rise to abstractions that are too complex](./concern-too-complex.md), combined with a recognition that the lack of lifetime GATs means that common patterns like [`Iterable`](../design_patterns/iterable.md) cannot be expressed, which leads to its own form of complexity.

The argument then is that we should stabilize lifetime GATs *only* and avoid types. This limits the kinds of patterns that GATs can be used for. Patterns that abstract over types, like [many modes](../design_patterns/many_modes.md), [pointer types](../design_patterns/pointer_types.md), or [generic scopes](../design_patterns/generic_scopes.md), should instead use alternatives, like code generation or macros.

## Counterpoint: Irregular design is its own learnability hazard

Limiting things to types is actually its own kind of learnability hazard. Having to learn "exceptions" (e.g., this only works for lifetimes) rather than a uniform set of rules (everything can take type parameters) can be quite challenging, particularly when the motivation for those exceptions is simply that we don't want people to be able to express certain abstractions (versus, say, the limitations that motivate dyn-safety, which are that it is simply not possible to general compiled code).

## Counterpoint: Without types, we can't express RPITIT or async fn

To desugar `-> impl Trait`, one must have the ability to write type-level GATs. For example:

```rust
trait Convert {
    fn convert(&self, c: impl Converter) -> impl Display;
}
```

would be translated to something like this:

```rust
trait Convert {
    type convert<T: Converter>: Display;
    fn convert<T: Converter>(&self, c: T) -> Self::convert<T>;
}
```

If we lose the ability to write type-only GATs, then we can no longer fully explain or desugar these more complex features, which is also its own learnability hazard (and gives some indication as to the expressiveness that is being lost).

