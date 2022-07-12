# Concern: Do we have the right rules for required bounds?

## Summary

QuineDot [writes](https://github.com/rust-lang/rust/pull/96709#issuecomment-1120050703)[^footnote]:

> Are the rules actually how we want them to be? 

[^footnote]: QuineDot also asked about the current rules. The rules are documented on the [required bounds](../explainer/required_bounds.md) page, and they've been brought up to date with nightly.

QuineDot pointed out various grey cases. The rules are reviewed in the [outlives-defaults design discussion](../design-discussions/outlives-defaults.md), which also discussed (in the FAQ) the possibility that we are missing some rules.

QuineDot also pointed out that the implementation seems to have diverged from the rules as discussed. That needs to be evaluated and reconciled.