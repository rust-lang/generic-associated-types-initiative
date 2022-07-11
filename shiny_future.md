# 🔮 Shiny future

{{#include badges/speculative.md}}

> 🚨 **Warning: Speculation ahead.** This section is a rewrite of the [explainer](./explainer.md) that includes various improvements. These improvements are still in proposal form and will require RFCs and design work before they are stabilized. 

Specific improvements being used here (some of which are references to other ongoing initiatives...)

| Improvement | Summary |
| --- | --- |
| Concise HRTB syntax | permit `T: Iterable<Item<'_> = &u32>` or `T::Item<'_>: Send` instead of `for<'a> T: Iterable<Item<'a> = &'a u32>` |
| Implied bounds | The `for<'a>` syntax in HRTB means "any suitable `'a`" and not "any `'a` at all" |
| Polonius | Polonius-style borrow checking |
| Default outlives bounds | Add default bounds for `where Self: 'a` when appropriate rather than requiring users to write them automatically; add those same defaults to the impl |
