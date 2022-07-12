# GATs permit complex patterns that will, on net, make Rust harder to use

## Summary

There is no question that GATs enable [new design patterns](../design_patterns.md). For the most part, these design patterns take the form of enabling a new kind of abstraction. For example, [many modes](../design_patterns/many_modes.md) allows a trait that encapsulates a "mode" in which some other code will be executed; lacking GATs, this can only be done in simple cases. These new design patterns may **locally** improve the code, but, if it becomes commonplace to use more complex abstractions in Rust code bases, Rust code overall will become harder for people to understand and read. [As burntsushi wrote:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1168643277)

> The complexity increase comes precisely from the abstraction power given to folks. By increasing the accessibility to that power, that power will in turn be used more frequently and users of the language will need to contend with those abstractions. 

burntsushi continues (emphasis ours):

> Here's where we come to the contentious part of this perspective: **a lot of people have real difficulty understanding abstractions.**

Nick Cameron also [wrote a blog post](https://www.ncameron.org/blog/complexity/) covering this theme. One of Nick's comments on Zulip points out that duplicated code in an API surface is annoying to maintain, but could be easier for consumers:

> [Nick Cameron](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/GATs.20and.20complexity/near/287854656): I think the risk of features like GATs is that encourages smaller, more abstract APIs which are more satisfying for library authors but which are harder for the median user to comprehend. For a non-GAT example, consider Result and Option compared to a more abstract monadic type. The latter reduces code duplication in the library and perhaps permits more interoperability, but we prefer having separate Result and Option in Rust because for most users, it's easier to understand an API with concrete values and functions like Ok and unwrap, rather than abstract concepts like unit and bind.

## What are the alternatives?

Nobody is contending that the [design patterns](../design_patterns.md) aren't addressing real problems, but they are contending that **solving those problems with a trait-based abstraction may not be the right approach**. So what are these alternatives? 

* In some cases, it's a "slightly less convenient API" (as in [CSV reader](https://docs.rs/csv/latest/csv/struct.Reader.html#method.read_record)).
* In others, the answer is some combination of macros and code generation. For example, the [many modes](../design_patterns/many_modes.md) pattern was used to make parser combinators work better, but that could also be addressed via procedural macros and code generation.
* In yet others, it's HRTB; the standard workaround for the [`Iterable`](../design_patterns/iterable.md) pattern is to write `for<'c> Iterate<'c>`, for example.
* Finally, it is sometimes possible to workaround the lack of GATs by introducing runtime overhead. e.g., foregoing the optimizations that [many modes](../design_patterns/many_modes.md) enabled, or using an enum, or a trait object.

## Counterpoint: GATs often allow library authors to hide complexity

Somewhat counter the above, frequently the goal with GATs is actually to make the interface of the library *simpler*:

> [oli:](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/GATs.20and.20complexity/near/287855607) Yes, and in user libraries right now you do stumble across those very complex APIs that work around the lack of GATs

> [vultix](https://github.com/rust-lang/rust/pull/96709#issuecomment-1170092720): I've personally seen two use-cases where GATs would have been useful, both times while writing libraries. We found an ugly workaround for the first use-case that wasn't horrible, but the second workaround made the user-facing api severely worse.

> [almann](https://github.com/amzn/ion-rust/issues/98): Currently, delegation via our current APIs are difficult because we cannot have an associated type that can be constructed with a per invocation lifetime. This [example](https://gist.github.com/almann/d21da1856d2a3c9f95dd04248a17d3ce) illustrates this, and shows how we can work around this by the judicious use of `unsafe` code, by essentially emulating the reference with pointers we know to be scoped appropriately. We also work around the lack of GAT by simplifying the value writer used in the implementation of `IonCAnnotationsFieldWriter` for `IonCWriterHandle` by refering to `Self` which works, but makes the API potentially easier to misuse.

The [many modes](../design_patterns/many_modes.md) pattern shows how the Chumsky parser generator used GATs internally. They don't appear in the public facing methods.

Jake Goulding points out that we hear very few stories of people who tried GATs but backed off:

> [Jake Goulding](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/GATs.20and.20complexity/near/287858483): It would be wonderful if there were experience reports of people saying "I thought I needed GATs but after writing them I realized I could do X instead and that was much clearer". My one experience was that I could use one of the stable GAT workarounds, but it felt wrong and I was happy to trial-run the nightly GAT version instead.



Nick Cameron:
IMO, this is complicated. However, many uses of it do not require understanding every nuance of what's going on in there:

That's fair. I think in general it is true, like sum is easier to understand than fold, or cloned is easier than map, but that doesn't necessarily mean that the more abstract thing is too difficult to understand. And the more abstract one is likely more general, so there's a trade-off for each case

## Counterpoint: Macros are not easier

Many folks chimed in to express their feeling that proc macros, or duplicated code, are not easier to understand or maintain, e.g. [Alice Cecile here](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/type-GATs.20vs.20lifetime-GATs.20.5Bfrom.20GATs.20and.20complexity.5D/near/288732525):

> Alice Cecile: Macros are so much worse to read / write / maintain than pretty much any type magic I've ever seen. Even very simple stuff is rough


## Counterpoint: Arguing against a feature because it could be misused would block many parts of Rust

https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/GATs.20and.20complexity/near/287765775

[Ralf Jung](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/GATs.20and.20complexity/near/287841239) writes:

> yes, that I think is the main argument to me. not having a feature because it could be used to write unnecessarily complicated APIs sounds like a bad argument to me -- it effectively means preventing some people from writing good APIs (assuming we accept there are APIs where the GAT version is clearly superior) for fear of other people writing bad APIs. one can already write unnecessarily complicated APIs in Rust in a lot of ways -- e.g. we didnt block proc-macros just because they can be used for bad APIs, though anyone who had to debug other people's sea of macros knows it can easily lead to terribly opaque API surfaces.
heck, we have unsafe code, where the consequences of using the feature in the wrong way are (IMO) a lot worse than GATs. I dont understand why GATs are suddenly considered a problem when they are (IMO) a feature much less in danger of being misused than unsafe code or proc macros. Rust has empowerment literally in its one-sentence slurb: we give people powerful tools and all the help we can to use them, and we accept that this means some people will misuse them, and we do what we can (technically and socially) to mitigate the consequences of that.
