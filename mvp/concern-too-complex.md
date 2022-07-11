# GATs permit complex patterns that will, on net, make Rust harder to use

## Summary

There is no question that GATs enable [new design patterns](../design_patterns.md). For the most part, these design patterns take the form of enabling a new kind of abstraction. For example, [many modes](../design_patterns/many_modes.md) allows a trait that encapsulates a "mode" in which some other code will be executed; lacking GATs, this can only be done in simple cases. These new design patterns may **locally** improve the code, but **globally** they have the potential to make Rust harder to understand. [As burntsushi wrote:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1168643277)

> The complexity increase comes precisely from the abstraction power given to folks. By increasing the accessibility to that power, that power will in turn be used more frequently and users of the language will need to contend with those abstractions. 

burntsushi continues (emphasis ours):

> Here's where we come to the contentious part of this perspective: **a lot of people have real difficulty understanding abstractions.**

In other words, nobody is contending that the [design patterns](../design_patterns.md) aren't addressing real problems, but they are contending that solving those problems with a trait-based abstraction may not be the right approach. So what are these alternatives? 

* In some cases, it's a "slightly less convenient API" (as in [CSV reader](https://docs.rs/csv/latest/csv/struct.Reader.html#method.read_record)).
* In many others, the answer is some combination of macros and code generation. For example, the [many modes](../design_patterns/many_modes.md) pattern was used to make parser combinators work better, but that could also be addressed via procedural macros and code generation.
* In yet others, it's HRTB; the standard workaround for the [`Iterable`](../design_patterns/iterable.md) pattern is to write `trait Iterable: for<'c> Iterate<'c>`, for example.
* Finally, it is sometimes possible to workaround the lack of GATs by introducing runtime overhead. e.g., foregoing the optimizations that [many modes](../design_patterns/many_modes.md) enabled, or using an enum, or a trait object.

## Counterpoint: GATs often allow library authors to hide complexity

Somewhat counter the above, frequently the goal with GATs is actually to make the interface of the library *simpler*. For example, the common workaround 

## Counterpoint: Macros are not easier

## Counterpoint: Macros are not easier