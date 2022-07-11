# MVP Stabilization

We are currently focused on stabilizing a **Minimum Viable Product** form of GATs in [rust-lang/rust#96709]. That's a long github thread, so this page summarizes the key points to understand.

* Understanding GATs in general:
    * The [explainer](./explainer.md) covers the design that is being stabilized in tutorial form.
    * The [design patterns](./design_patterns.md) section covers ways that GATs used in the wild.
    * The [rough edges](./explainer/rough_edges.md) section explains some of the ways GATs are currently difficult to use; the [shiny future](./shiny_future.md) section contains speculation on how to address those.
* Highlights from the thread:
    * The [stabilization report](https://github.com/rust-lang/rust/pull/96709#issue-1225460272) lays the groundwork.
    * This [summary comment](https://github.com/rust-lang/rust/pull/96709#issuecomment-1129311660) identified the core concerns. We have detailed them here in more depth, and provided some counterpoints:
        * [GATs permit complex patterns that will, on net, make Rust harder to use](./mvp/concern-too-complex.md)
        * [Stabilizing GATs with lifetimes only would help prevent this](./mvp/concern-lifetimes-only.md)
        * [GATs are useful but the current state is too rough and should not be stabilized](./mvp/concern-too-rough.md)
        * [Given the number of papercuts, how do we know that the current MVP is forwards compatible with the fixes?](./mvp/concern-backcompat.md)

    
[rust-lang/rust#96709]: https://github.com/rust-lang/rust/pull/96709
[rust-lang/rust#96709]: https://github.com/rust-lang/rust/pull/96709



