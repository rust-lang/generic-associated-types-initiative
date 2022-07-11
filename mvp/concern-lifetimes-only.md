# Concern: we should stabilize lifetimes only

## Summary

This suggestion stems from the concern that [GATs give rise to abstractions that are too complex](./concern-too-complex.md), combined with a recognition that the lack of lifetime GATs means that common patterns like [`Iterable`](../design_patterns/iterable.md) cannot be expressed, which leads to its own form of complexity.

The argument then is that we should stabilize lifetime GATs *only* and avoid types. This limits the kinds of patterns that GATs can be used for. Patterns that abstract over types, like [many modes](../design_patterns/many_modes.md), [pointer types](../design_patterns/pointer_types.md), or [generic scopes](../design_patterns/generic_scopes.md), should instead use alternatives, like code generation or macros.

## Concern: 

Limiting things to types is actually its own kind of learnability hazard. Having to learn "exceptions" 