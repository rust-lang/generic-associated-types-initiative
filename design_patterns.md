# Design patterns

A natural question with GATs is to ask "what are they used for?" Because GATs represent a kind of "fundamental capability" of traits, though, that question can be difficult to answer in a short summary -- they can be used for all kinds of things! Therefore, this section attempts to answer by summarizing "design patterns" that we have seen in the wild that are enabled by GATs. These patterns are described through a "deep dive" into a particular example, often of a crate in the wild; but they represent patterns that could be extracted and applied in other cases.

## List of projects using GATs

Over the years, many people have posted examples of how they would like to use GATs. compiler-errors compiled a mostly complete list which was [posted to the stabilization issue](https://github.com/rust-lang/rust/pull/96709#issuecomment-1173170243). We reproduce that list here:

### Projects using GATs

* [`connector-x`](https://github.com/sfu-db/connector-x) - uses GATs to provide zero-copy interfaces to load data from DBs.
* [`kas`](https://github.com/kas-gui/kas/pull/57) - uses Generic Associated Types to avoid the unsafety around draw_handle and size_handle, fixing a possible memory-safety violation in the process.
    * "generic associated types remove a usage of unsafe (revealing a bug in the process), and are almost certainly the way forward (once the compiler properly supports this)"
* [`raft-engine`](https://github.com/tikv/raft-engine/pull/96) - uses GATs in a generic builder pattern

### Blocked (at least in part) by GATs:
* Rust in the linux kernel - https://github.com/Rust-for-Linux/linux/issues/2
* [`udoprog/audio`](https://github.com/udoprog/audio/issues/3#issuecomment-1021805121) - "My goal is to author a set of traits and data structures that can be used across audio libraries in the ecosystem. I'm currently holding off on GATs landing, since that's needed to provide proper abstractions without incurring a runtime overhead."
* [`graphene`](https://github.com/Emoun/graphene/issues/7) - "This would allow the result types of most Graph methods to be impl Iterator, such that implementors can use whatever implementation they want. To achieve this currently we are using `Box<Iterator>` as return types which is inefficient."
* [`ion-rust`](https://github.com/amzn/ion-rust/issues/98) - "Currently, delegation via our current APIs are difficult because we cannot have an associated type that can be constructed with a per invocation lifetime. This example illustrates this, and shows how we can work around this by the judicious use of unsafe code[...]"
* [`proptest`](https://github.com/AltSysrq/proptest/issues/9) - GATs could be used to represent non-owned values
* [`ink`](https://github.com/paritytech/ink/pull/1217#issuecomment-1124049962) - GATs could be used to simplify macro codegen around trait impl
* [`objc2`](https://github.com/madsmtm/objc2/pull/37) - Could use GATs to abstract over a generic `Reference` type, simplifying two methods into one
* [`mpris-rs`](https://github.com/Mange/mpris-rs/issues/64) - GATs could be used to abstract over an iterator type
* [`dioxus`](https://github.com/DioxusLabs/dioxus/pull/412#issue-1239019905) - "It allows some property of a node to depend on both the state of it's children and it's parent. Specifically this would make text wrapping, and overflow possible"
* [`sophia_rs`](https://github.com/pchampin/sophia_rs/issues/19#issuecomment-563972666) - An other way to go would be to have an "rdf-api" crate similar to what RDF/JS is doing for the RDF models and its commons extensions. And have Oxigraph and Sophia and hopefully the other RDF related libraries in Rust use it. But it might be hard to build a nice and efficient API without GAT.
    
### Other miscellaneous mentions of GATs, or GATs blocked a rewrite but workarounds were found

* [`veracruz`](https://github.com/veracruz-project/veracruz/pull/269) - The workaround here is to require the associated-type implementations to all be "lifetime-less", which probably requires unsafe code in the implementations.
* [`embedded-nal`](https://github.com/rust-embedded-community/embedded-nal/issues/31#issuecomment-736676502) - Using a typestate pattern to represent UDP trait's states
* [`linfa`](https://github.com/rust-ml/linfa/issues/119#issuecomment-884628664) - For now though, there are several limitations to the implementation due to a lack of type-system features. For instance, it has to return a Vec of references to points because the lifetime to &self has to show up somewhere in the return type, and we don't have access to GATs yet. Basically, I get around these issues by performing more allocation than we should [...]
* [`heed`](https://github.com/meilisearch/heed/pull/92) - Initial rewrite of a trait relied on GATs, was eventually worked around but has its own limitations?
* `ockam` - https://github.com/build-trust/ockam/issues/1564
* [`rust-imap`](https://github.com/jonhoo/rust-imap/pull/208#discussion_r680427811) - could benefit with a GATified `Index` trait
* [`anchors`](https://github.com/lord/anchors/issues/4) - "GAT will let us skip cloning during map"
* [`capnproto-rust`](https://github.com/capnproto/capnproto-rust/pull/201#issuecomment-731585030) - The main obstacle is that getting this to work (particularly with `capnp::traits::Owned`) probably requires generic associated types, and it's unclear when those will land on stable rust.
* [`gamma`](https://github.com/metamolecular/gamma/issues/8) - Could use GATs to return iterators instead of having to box them, possibly providing a more general API without the slowdown of boxing
* [`dicom-rs`](https://github.com/Enet4/dicom-rs/pull/152#discussion_r681182316) - GAT could be used to representing lifetime of borrowed data
* [`rust-multihash`](https://github.com/multiformats/rust-multihash/pull/116) - Apparently could use GATs to get around const-generics limitations 
* [`libprio-rs`](https://github.com/divviup/libprio-rs/pull/202#discussion_r839932483) - Doing better will require a feature called "generic associated types" (per the SO above). Unfortunately, GATs are not stabilized yet; it looks like they are set to be stabilized in a few months.
* [`gluesql`](https://github.com/gluesql/gluesql/issues/483) - Could use GATs to turn a trait into an associated type<sup>(i think)</sup>
* [`pushgen`](https://github.com/AndWass/pushgen/issues/44#issuecomment-890302372) - mentioned that things could be simplified by GATs (or RPITIT)
* [`plexus`](https://github.com/olson-sean-k/plexus/issues/16#issuecomment-498786388) - "Until GATs land in stable Rust, this change requires boxing iterators to put them behind StorageProxy."
* [`tensorflow/rust`](https://github.com/tensorflow/rust/issues/162#issuecomment-587226628) - "It would be most natural to provide an iterator over the records, but no efficient implementation is possible without GATS which I think I was hoping would already be in the language by now."

### General themes for why folks want GATs

The general themes for why folks want GATs include...

* GATs to avoid boxing/cloning (achieving a more performant, zero-copy API)
* GATs to represent lifetime relationships that can't be expressed using regular associated types (e.g. `fn(&self) -> Self::Gat<'_>`)
  * Some of these get around it by using unsafe code which could be removed with GATs
* GATs as a manual desugaring of RPITIT
* GATs to offer a more abstract/pluggable API
* GATs to provide a cleaner, DRY-er codebase

