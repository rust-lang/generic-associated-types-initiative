# Rough edges

{{#include ../badges/nightly.md}} {{#include ../badges/stabilization-96709.md}} {{#include ../badges/seeking-feedback.md}}

The current MVP state includes a number of known rough edges, explained here.

| Edge | Brief explanation |
| --- | --- |
| Clumsy HRTB syntax | The syntax `for<'a> T: Iterable<Item<'a> = ...>` is clumsy |
| [Required bounds] | Compiler requires you to write `where Self: 'me` a lot |
| [No implied bounds on HRTB] | The `for<'a>` syntax should really mean "for any suitable `'a`" to avoid false errors |
| HRTB limited to lifetimes | You cannot write `for<T>` or `for<const C>` to talk about types, lifetimes | 
| Polonius interaction | Some GAT patterns require polonius to pass the borrow checker |
| Not dyn safe | Traits with GATs are not dyn safe, even if those GATs are limited to lifetimes |

[Required bounds]: ./required_bounds.md
[No implied bounds on HRTB]: ./no_implied_bounds.md