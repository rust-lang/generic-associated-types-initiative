# Concern: GATs are useful but the current state is too rough and should not be stabilized

## Examples

[nrc wrote:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1118275010)

> There are numerous cases of small bugs or small gaps in expressivity which have prevented people using GATs for the use cases they want to use them for (see e.g., the blog post linked from the OP, or this reddit thread). These are the sort of things which must be addressed before stabilisation so that we can be sure that they are in fact small and not hiding insurmountable issues.

and [burntsushi wrote:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1118627760)

> I understand the idea of making incremental progress, but as a total outsider to lang development, this feature looks like it has way too many papercuts and limitations to stabilize right now. Not being able to write a [basic `filter` implementation on a `LendingIterator` trait](https://github.com/rust-lang/rust/issues/92985) in safe code is a _huge_ red flag to me. Namely, if I can't write a filter adaptor in safe straight-forward code, then... what else can't I do? 

clintfred chimes in with a "intermediate Rust user" perspective [similar to this](https://github.com/rust-lang/rust/pull/96709#issuecomment-1120054138):

> As a solidly intermediate Rust user I can say that I have tried wrap my mind around GATs several times over the past 3-4 years, and I don't really "get" it. To be completely fair, I have never tried using them on nightly, so maybe this something best groked hands-on.

Sabrina also wrote a [blog post showing some of the problems with lifetime GATs in their current state](https://sabrinajewson.org/blog/the-better-alternative-to-lifetime-gats).

## Details

GATs in their current form have some pretty [severe limitations](../explainer/rough_edges.md). These limitations block one of the most compelling use cases, lending iterators, and fixing that will require polonius (c.f. [#92985]). Worse, these limitations are hard to convey to users: things just kind of randomly don't work, and it's not obvious that this is the compiler's fault (this is kind of a "worst case" for learnability, since it can trick you into thinking you don't understand the model, when in fact you do).

[#92985]: https://github.com/rust-lang/rust/issues/92985


## Responses

### Aren't GATs too hard to learn in their current state (or maybe in general)?

There is no doubt that "papercuts" can have a major impact on learnability. However, the lack of GATs, even in rudimentary form, causes its share of papercuts as well. Things that feel like they *ought* to be expressable (e.g., an [`Iterable`](../explainer/iterable.md) trait) are not. For example, [CLEckhardt writes](https://github.com/rust-lang/rust/pull/96709#issuecomment-1130190157):

> The complexity is either in the language or the code. There are a lot of complex things that Rust lets me express cleanly, which is why I love it. But when I run into things that Rust can't express, I have to write complex code. So the complexity doesn't leave, it just moves.

[mijamo wrote something similar](https://github.com/rust-lang/rust/pull/96709#issuecomment-1131885602):

> For what it's worth I am a rust beginner coming from mostly Python / Typescript. There are MANY things in Rust that make my head hurt but GAT just seem natural after some introduction and I actually get stuck trying to go around the lack of GAT nearly every time I try to write Rust code (I have given up in every single of my rust projects so far and I would say lack of GAT and annoying lifetime handling were my 2 main problems).

[audunhalland left a similar comment:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1119760258)

> I'm no "type theorist" at all, I have no deep insight into how the Rust typesystem works internally, or what is the deeper cause of some random compile error. With GATs I just tend to keep flicking on the code until the compiler is somehow happy. Just like classic non-GAT Rust code :) I have stumbled across some of the HRTB limitations, but have managed to work around them quite easily. My feeling is that for my use-cases GATs now feel like a natural part of the language.

Pzixel describes how they [wind up using GATs in most projects](https://github.com/rust-lang/rust/pull/96709#issuecomment-1120175346):

> It all starts similarly: I have bunch of traits that return Vecs or something. Then I profile and see allocs in tight loops I cannot afford. \[..] Every time you want to have zero-cost in traits GATs come into play. Iterators and futures are just some examples, there are more. And I bet that every program written in rust are using these concepts and people there either don't care and put boxes in traits or write their own super niche fragile implementations to fit their needs. \[..] I'd like to put some thought on the matter "No one using GATs" and "GATs are just too complicated" - it reminds me of generics discussions in Go. While GATs may be complicated making it work without them is even harder. It worth always keeping it in mind

### Will we really do the follow-up required?

One specific danger is that we would stabilize GATs and then "declare victory", leaving the feature unchanged for years. There are a few reasons to think that won't happen. 

For one, the new [types team](https://github.com/rust-lang/types-team) is ramping up and focused on precisely these kinds of improvements.

Second, having features on master leads to more attention and volunteers, not less (although that alone is not enough when the problems are deeper). [jam1garner writes:](https://github.com/rust-lang/rust/pull/96709#issuecomment-1118868165)

> And yeah the UX could be better :( I can't speak for everyone but as someone who tries to make diagnostics PRs when I hit issues and have the time/energy, I personally can't really kick the tires if I can't use the feature in my libraries, and thus don't have a very natural path to find pain points to.

More generally, Rust's entire history has been one of taking complex features, exposing them, and sanding down the edges over time -- and sometimes stalling out for an extended period of time. And yet, it's hard to come up with an example where it would've been better to hold off on stabilization. Take async fundtions: when stabilized, they had many diagnostic rough edges, and still do. 

This is very much true of async functions,and yet. Trying to get *everything