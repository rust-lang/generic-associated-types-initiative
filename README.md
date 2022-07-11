# Generic Associated Types initiative

<!--
 Status badge advertising the project as being actively worked on. When the
 project has finished be sure to replace the active badge with a badge
 like: https://img.shields.io/badge/status-archived-grey.svg
-->
![initiative status: active](https://img.shields.io/badge/status-active-brightgreen.svg)

This page tracks the work of the Generic Associated Types (GATs) [initiative]! To learn more about what we are trying to do, and to find out the people who are doing it, take a look at the [charter]. 

[charter]: ./CHARTER.md
[initiative]: https://lang-team.rust-lang.org/initiatives.html

## ⚡ Current focus: MVP ⚡

We are currently focused on stabilizing a **Minimum Viable Product** form of GATs in [rust-lang/rust#96709]. [Learn more here!](./mvp.md) You can learn more:

* The [explainer](./explainer.md) covers the design that is being stabilized, and the [rough edges](./explainer/rough_edges.md) section explains some of the ways it is currently difficult to use (see the [shiny future](./shiny_future.md) section for speculation on how to address those).
* The [design patterns](./design_patterns.md) section documents some of the ways GATs are used in the wild.
* The []


[rust-lang/rust#96709]: https://github.com/rust-lang/rust/pull/96709

The following table lists of the stages of an initiative, along with links to the artifacts that will be produced by the start of that stage.

| Stage                                 | State | Artifact(s) |
| ------------------------------------- | ----- | ----------- |
| [Proposal]                            | ✅    | [Charter](./CHARTER.md) |
|                                       |       | [Tracking issue] |
| [Experimental]                        | ✅    | [RFC](./RFC.md) |
| []
| [Development]                         | 🦀    | [Explainer](./explainer.md) | 
| [Feature complete]                    | 💤    | Stabilization report |
| [Stabilized]                          | 💤    | |

[Tracking issue]: https://github.com/rust-lang/rust/issues/44265
[Proposal]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Experimental]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Development]: https://lang-team.rust-lang.org/initiatives/process/stages/development.html
[Feature complete]: https://lang-team.rust-lang.org/initiatives/process/stages/feature-complete.html
[Stabilized]: https://lang-team.rust-lang.org/initiatives/process/stages/stabilized.html

Key:

* ✅ -- phase complete
* 🦀 -- phase in progress
* 💤 -- phase not started yet

## How Can I Get Involved?

* Check for "help wanted" issues on this repository!
* If you would like to help with development, please contact the [owner](./charter.md#membership) to find out if there are things that need doing.
* If you would like to help with the design, check the list of active [design questions](./design-questions/README.md) first. 
* If you have questions about the design, you can file an issue, but be sure to check the [FAQ](./FAQ.md) or the [design-questions](./design-questions/README.md) first to see if there is already something that covers your topic.
* If you are using the feature and would like to provide feedback about your experiences, please [open a "experience report" issue].
* If you are using the feature and would like to report a bug, please open a regular issue.

We also participate on [Zulip][chat-link], feel free to introduce yourself over there and ask us any questions you have.

[open issues]: /issues
[chat-link]: https://rust-lang.zulipchat.com/#narrow/stream/144729-wg-traits
[team-toml]: https://github.com/rust-lang/team/blob/master/teams/initiative-gats.toml

## Building Documentation
This repository is also an mdbook project. You can view and build it using the
following command.

```
mdbook serve
```
