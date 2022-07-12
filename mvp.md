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
    * One detailed question was raised, about the specifics of our rules for [required bounds](explainer/required_bounds.md):
        * [Do we have the right rules for required bounds?](./mvp/concern-right-rules-for-required-bounds.md)
    
[rust-lang/rust#96709]: https://github.com/rust-lang/rust/pull/96709
[rust-lang/rust#96709]: https://github.com/rust-lang/rust/pull/96709



## Places where conversation has been happening?

Want to read the firehose? Check out these places where conversation has been happening:

* [rust-lang/rust#96709][] issue thread
* [Zulip thread on GATs and complexity](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/GATs.20and.20complexity)
* [Zulip thread on "type-GATs vs lifetime-GATs"](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/type-GATs.20vs.20lifetime-GATs.20.5Bfrom.20GATs.20and.20complexity.5D)
* [Zulip thread on renaming `for`](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/renaming.20.02klzzwxh.3A0000.03) -- side topic that spun out