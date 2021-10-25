# Future directions

There are some points about GATs which we expect to change. Deciding on the best solution, however, requires more information about the way GATs are used in practice. Therefore, we encourage you to [file a new issue] sharing your experiences and giving feedback on these points.

[open issues]: [file an issue]: https://github.com/rust-lang/generic-associated-types-initiative/issues/new/choose

## Proposal A: Supply [required bounds] by default

The first planned change is to make the [required bounds] on associated types be supplied by default, rather than requiring that they be typed explicitly. We may or may not include some form of annotation to "opt-out" from this default. In short, the expectation is that -- in practice -- virtually all traits will want these where clauses, and there will be negligibly few cases that do not. We are therefore looking for examples where having to add the required bounds was a problem, and you were forced to [adopt the workaround].

[required bounds]: ./required_bounds.md
[adopt the workaround]: ./required_bounds.md#workaround