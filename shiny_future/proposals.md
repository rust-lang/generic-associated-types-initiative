# Proposals

{{#include ../badges/speculative.md}}

Specific improvements being used in the shiny future section include:

| Improvement | Summary |
| --- | --- |
| [Concise HRTB syntax] | permit `T: Iterable<Item<'_> = &u32>` or `T::Item<'_>: Send` instead of `for<'a> T: Iterable<Item<'a> = &'a u32>` |
| [HRTB implied bounds] | The `for<'a>` syntax in HRTB means "any suitable `'a`" and not "any `'a` at all" |
| Polonius | Polonius-style borrow checking |
| Default outlives bounds | Add default bounds for `where Self: 'a` when appropriate rather than requiring users to write them automatically; add those same defaults to the impl |

[Concise HRTB syntax]: ./hrtb-syntax.md
[HTB implied bounds]: ./hrtb-implied-bounds.md