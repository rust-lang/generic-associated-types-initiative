# Concern: GATs are useful but the current state is too rough and should not be stabilized

## Examples

[nrc wrote:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1118275010)

> There are numerous cases of small bugs or small gaps in expressivity which have prevented people using GATs for the use cases they want to use them for (see e.g., the blog post linked from the OP, or this reddit thread). These are the sort of things which must be addressed before stabilisation so that we can be sure that they are in fact small and not hiding insurmountable issues.

and [burntsushi wrote:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1118627760)

> I understand the idea of making incremental progress, but as a total outsider to lang development, this feature looks like it has way too many papercuts and limitations to stabilize right now. Not being able to write a [basic `filter` implementation on a `LendingIterator` trait](https://github.com/rust-lang/rust/issues/92985) in safe code is a _huge_ red flag to me. Namely, if I can't write a filter adaptor in safe straight-forward code, then... what else can't I do? 

## Details

GATs in their current form have some pretty [severe limitations](../explainer/rough_edges.md). These limitations block one of the most compelling use cases, lending iterators, and fixing that will require polonius (c.f. [#92985]). Worse, these limitations are hard to convey to users: things just kind of randomly don't work, and it's not obvious that this is the compiler's fault (this is kind of a "worst case" for learnability, since it can trick you into thinking you don't understand the model, when in fact you do).

[#92985]: https://github.com/rust-lang/rust/issues/92985

Another concern with stabilizing with known limitations is that fixing, in the course of trying to address those limitations, we may find we wish to make backwards incompatible changes. Keeping things unstable ensures we have room to change the details.

## Responses

### Aren't GATs too hard to learn in their current state (or maybe in general)?

There is no doubt that "papercuts" can have a major impact on learnability. However, the lack of GATs, even in rudimentary form, causes its share of papercuts as well. Things that feel like they *ought* to be expressable (e.g., an [`Iterable`](../explainer/iterable.md) trait) are not. For example, [CLEckhardt writes](https://github.com/rust-lang/rust/pull/96709#issuecomment-1130190157):

> The complexity is either in the language or the code. There are a lot of complex things that Rust lets me express cleanly, which is why I love it. But when I run into things that Rust can't express, I have to write complex code. So the complexity doesn't leave, it just moves.

[mijamo wrote something similar](https://github.com/rust-lang/rust/pull/96709#issuecomment-1131885602):

> For what it's worth I am a rust beginner coming from mostly Python / Typescript. There are MANY things in Rust that make my head hurt but GAT just seem natural after some introduction and I actually get stuck trying to go around the lack of GAT nearly every time I try to write Rust code (I have given up in every single of my rust projects so far and I would say lack of GAT and annoying lifetime handling were my 2 main problems).

## Counter

One specific danger is that we would stabilize GATs and then "declare victory", leaving the feature unchanged for years. The best counter

## Why do we believe future changes will be backwards compatible?

The GAT syntax being stabilized is leveraging existing syntactic constructs, like associated types and `for<'a>` bounds. The problems we are running into exist, for the most part, with those constructs as well. The fixes discussed in the [shiny future](../shiny_future.md) section are not typically specific to GATs.

One area where we've specifically concerned backcompat is the question of [required bounds](../explainer/required_bounds.md), and in that area we particularly chose the most forwards compatible option (requiring users to write things out explicitly, thus ensuring they can be made optional or defaulted later on). We will always have to permit users to write where-clauses, so there is no real danger there.

