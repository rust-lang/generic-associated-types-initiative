# How do we know that the current MVP is forwards compatible with the fixes?

## Summary

GATs in their current form have some pretty [severe limitations](../explainer/rough_edges.md). As we address those limitations, we may find we wish to make backwards incompatible changes. Keeping things unstable ensures we have room to change the details.

## Counterpoints

The GAT syntax being stabilized is leveraging existing syntactic constructs, like associated types and `for<'a>` bounds. The problems we are running into exist, for the most part, with those constructs as well. The fixes discussed in the [shiny future](../shiny_future.md) section are not typically specific to GATs.

One area where we've specifically concerned backcompat is the question of [required bounds](../explainer/required_bounds.md), and in that area we particularly chose the most forwards compatible option (requiring users to write things out explicitly, thus ensuring they can be made optional or defaulted later on). We will always have to permit users to write where-clauses, so there is no real danger there.

