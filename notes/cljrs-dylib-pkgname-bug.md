# `:rust/load :dylib` wrapper fails for crates whose package name has a hyphen

## Summary

With the deps-loading fix in place, a `:rust/load :dylib` dependency is now
correctly *reached* by `require` (the loader invokes the pinned-native dylib
builder). But the generated wrapper crate fails to build when the dependency's
**cargo package name contains a hyphen** (e.g. `cljrs-base64`):

```
[cljrs-dylib] building pinned native package cljrs_base64@<commit>…
error: no matching package found
searched package name: `cljrs_base64`
perhaps you meant:      cljrs-base64
location searched: …/crates/cljrs-base64
required by package `cljrs-pinned-wrapper`
```

The wrapper's `Cargo.toml` uses the **underscored Rust ident** derived from
`:rust/init` as the cargo dependency key, but a cargo dependency key must match
the dependency's `[package] name` (hyphenated). They differ for any crate whose
package name uses `-`.

## Environment

- clojurust `8cd7b29` (`cljrs` `0.1.0`, default features)

## Repro

`cljrs.edn` (note the documented, underscored `:rust/init`):

```clojure
{:deps {cljrs.base64 {:git/url    "https://github.com/csm/clojurust"
                      :git/sha    "8cd7b299210a5f1cf96e67828c31a93fbbabde16"
                      :rust/crate "crates/cljrs-base64"
                      :rust/init  "cljrs_base64::cljrs_init"   ; underscored (matches docs)
                      :rust/load  :dylib}}
 :paths ["src"]}
```

`src/main.cljrs`:

```clojure
(ns demo)
(require '[cljrs.base64 :as base64])
(defn -main [& _] (println (base64/encode "user:pass")))   ; expect dXNlcjpwYXNz
```

```sh
cljrs deps fetch
cljrs run src/main.cljrs
```

### Expected

`dXNlcjpwYXNz`.

### Actual

`cargo build` of the generated wrapper fails with `no matching package found
… searched package name: cljrs_base64 … perhaps you meant: cljrs-base64`.

## Root cause

In `crates/cljrs-dylib/src/lib.rs`:

- `build_pinned_wrapper` derives the crate name from `:rust/init`:
  ```rust
  let crate_name = init_fn.split("::").next().unwrap_or(init_fn); // "cljrs_base64"
  let pkg_ident  = crate_name.replace('-', "_");                  // "cljrs_base64"
  ```
- `write_wrapper_crate` then emits the dependency line in the wrapper's
  `Cargo.toml` using `crate_name` as the **dependency key**:
  ```text
  [dependencies]
  …
  cljrs_base64 = { path = "…/crates/cljrs-base64" }
  ```
  Cargo looks for a package literally named `cljrs_base64`, but the crate's
  `[package] name` is `cljrs-base64`, so resolution fails. (The generated
  `src/lib.rs` is fine — it calls `{pkg_ident}::{init_tail}`, and cargo maps the
  hyphenated package to the underscored crate ident automatically.)

The dependency key should be the crate's real **package name**, not the
underscored Rust ident from `:rust/init`.

## Suggested fix

Read `[package] name` from `crate_dir/Cargo.toml` and use it as the wrapper's
dependency key (optionally with `package = "<name>"` if a different key is
desired), keeping `pkg_ident` only for the `extern crate` / call path in the
generated `src/lib.rs`. Then the documented underscored `:rust/init` works for
hyphenated package names.

## Workaround

Spell `:rust/init` with the **hyphenated package name**:

```clojure
:rust/init "cljrs-base64::cljrs_init"
```

`crate_name` then becomes `cljrs-base64` (correct cargo dependency key) while
`pkg_ident` is still `cljrs_base64` (correct Rust ident). Verified end to end:
the wrapper builds, `[cljrs-dylib] loaded native dep cljrs.base64`, and
`base64/encode "user:pass"` → `dXNlcjpwYXNz`. `wave` uses this workaround in its
`cljrs.edn`.

## Note on validation environment

This was validated with the clojurust git remote unreachable (only the cached
commit `963f0aa` available, whose `cljrs-base64` source is byte-identical to
`8cd7b29`). Two env-only adjustments were needed and are **not** part of the bug:
pointing the cache remote at a local mirror (`fetch_remote` always re-contacts
the remote even when the commit is already cached), and `CLJRS_WORKSPACE_ROOT`
set to the cached worktree so the wrapper's workspace crates came from one
consistent checkout. With a normal network and a pin matching the running
binary, neither is needed.
