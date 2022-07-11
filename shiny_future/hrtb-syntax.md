# HRTB Syntax

{{#include ../badges/speculative.md}}

## Summary

Permit `'_` to be used in where-clauses and `impl Trait` input position to mean "higher-ranked trait bounds". When used in output position, `'_` refers back to one of the inputs (if applicable) or errors.

## Examples

### Where clauses

### impl Trait

* `fn gimme_refs(x: impl Iterator<Item = &u32>)`
    * Currently: [errors](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=6d7397a2758523298574ba85db67d7d1)
    * Proposed: ...hmm, tricky case :)
* `fn gimme_refs(x: impl PartialEq<&u32>)`
    * Currently: [errors](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=6d7397a2758523298574ba85db67d7d1)
    * Proposed: ...hmm, tricky case :)
