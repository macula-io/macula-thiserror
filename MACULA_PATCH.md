# Macula fork of `thiserror` 2.0.18

Vendored fork of [dtolnay/thiserror](https://github.com/dtolnay/thiserror)
at version `2.0.18`, with a single patch flipping the default feature
set from `["std"]` to `[]` so `no_std` consumers (via Cargo's
`[patch.crates-io]`) do not get `std` re-enabled through feature
unification.

Used by [macula-kernel](https://codeberg.org/macula-internal/macula-kernel)
to satisfy `quinn-proto`'s transitive dependency on `thiserror` without
pulling `std` into the kernel target.

## The patch

```diff
 [features]
-default = ["std"]
+default = []
 std = []
```

Applied to both `Cargo.toml` (cargo-normalized) and `Cargo.toml.orig`
(upstream hand-written).

## Why this is needed

Upstream `thiserror` correctly gates `extern crate std;` and
`extern crate std as core;` (an old-edition polyfill) on
`feature = "std"`. The crate's source compiles fine in a `no_std`
environment when `std` is off.

The problem is at the cargo-feature layer: `default = ["std"]` plus
`quinn-proto` pulling `thiserror` with default features unifies `std`
back on. Source-level gating is then bypassed and the two
`extern crate std` lines at `src/lib.rs:277` and `src/lib.rs:279`
both emit `E0463: can't find crate for std` against the kernel target.

Flipping `default = []` makes `std` strictly opt-in. Without `std`,
`thiserror` falls back to formatting `core::fmt`-only error paths,
which is exactly what no_std callers need.

The bundled `thiserror-impl` proc-macro crate is unaffected (it builds
host-side for the build machine, not the kernel target).

## Upstream contribution

File an upstream PR proposing `default = []`. Upstream may resist for
backwards-compatibility reasons. If accepted, retire this fork.

## Versioning policy

Track `thiserror` upstream releases of 2.x. Re-roll by:

1. Downloading `thiserror-X.Y.Z.crate` tarball from crates.io.
2. Extracting into a fresh checkout.
3. Re-applying the `default = []` patch to both `Cargo.toml` and `Cargo.toml.orig`.
4. Tagging as `vX.Y.Z-macula1`.
5. Bumping the git ref in `macula-kernel`'s `Cargo.toml`.

## License

Inherited from upstream `thiserror` (Apache-2.0 OR MIT). No new code,
only the Cargo.toml feature-default flip.
