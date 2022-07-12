# Pointer types

{{#include ../badges/nightly.md}} {{#include ../badges/stabilization-96709.md}}

## Summary

GATs allow code to be generic over "references to data". This can include `Rc` vs `Arc`, but also other more abstract situations.

## Details

(To be written.)

## References

* Pythonesque's [comment](https://github.com/rust-lang/rust/pull/96709#issuecomment-1150127168) covered one case where they *wanted* something like a pointer types pattern (I think) but had to work around it, as well as [commits from Veloren](https://github.com/amethyst/specs/blob/master/src/storage/generic.rs#L114-L150) that may be this pattern (but could also be "many modes").
* evenyag [writes](https://github.com/rust-lang/rust/pull/96709#issuecomment-1170276610) about `type-exercise-in-rust`