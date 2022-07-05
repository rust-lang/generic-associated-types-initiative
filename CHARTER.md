# "Generic Associated Types" Charter

## Goals

Extend traits by permitting associated types to have generic parameters (types, lifetimes).

This enables writing traits that capture more complex patterns:

* an [`Iterable`] trait for collections that support an `iter` method, yielding references into themselves
* a [`Mode`] trait

It also provides the technical foundation of [async functions in traits](https://rust-lang.github.io/async-fundamentals-initiative/explainer/async_fn_in_traits.html) and [return-position impl Trait in traits](https://rust-lang.github.io/impl-trait-initiative/explainer/rpit_trait.html) (although this dependence is internal to the compiler and exposed to users).

* Concretely, extend associated types with generic parameters.
* This makes it possible to write traits describing more complex patterns, e.g.
    * methods with arguments or return values that borrow from other arguments
    * 
* Extend Rust traits to describe patterns that currently cannot be described...
    * associated types that reference 
* Extend associated types on traits to have type and lifetime parameters.

[generic associated types]: https://github.com/rust-lang/rfcs/pull/1598

## Membership

| Role | Github |
| ---  | --- |
| [Owner] | [jackh726](https://github.com/jackh726/) | 
| [Liaison] | [nikomatsakis](https://github.com/nikomatsakis/) | 

[Owner]: https://lang-team.rust-lang.org/initiatives/process/roles/owner.html
[Liaison]: https://lang-team.rust-lang.org/initiatives/process/roles/liaison.html
