# Generic scopes

## Summary

APIs like [`std::thread::scope`](https://doc.rust-lang.org/std/thread/fn.scope.html) follow a "scope" pattern:

```rust
in_scope(|scope| {
    ... /* can use `scope` in here */ ...
})
```

In this pattern, the closure takes a `scope` argument whose type is somehow limited to the closure body, often by including a fresh generic lifetime (`'env`, in the case of [`std::thread::scope`]). The closure is then able to invoke methods on `scope`. This pattern makes sense when there is some setup and teardown required both/after the scope (e.g., blocking on all the threads that were spawned to terminate).

The "generic scopes" pattern encapsulates this "scoped closure" concept:

```rust
owner.with_scope(|scope| {
    ...
})
```

Here, the type of the `scope` that the closure will depend on the `with_scope` call, but it needs to include some fresh lifetime that is tied to the `with_scope` call itself.

## Details

The generic scopes pattern arise from [smithay](https://github.com/rust-lang/rust/pull/96709#issuecomment-1120354039) and was demonstrated by this [playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=a23b6a846aa1a506c199f7792e1abd3e)) snippet.

In this case, the "owner" object is a renderer, and the "scope" call is called `render`. Clients invoke...

```rust
r.render(|my_frame| { ... })
```

...where the precise type of `my_frame` depends on the renderer. Frames often include thread-local information which should only be accessible during that callback.

