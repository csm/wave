# Native-code dependencies declared in `cljrs.edn` cannot be loaded by `require`

## Summary

A dependency that ships **native Rust code** (a crate exposing a `cljrs_init`
registry function, e.g. `cljrs-base64`) cannot be brought into a program by
declaring it in `cljrs.edn` `:deps` and `require`-ing its namespace. There is no
deps configuration that makes

```clojure
(require '[cljrs.base64 :as base64])
(base64/encode "user:pass")
```

resolve `cljrs.base64` when that namespace is provided by a dependency rather
than compiled into the running `cljrs` binary. `require` fails with:

```
Could not find namespace cljrs.base64 on source path
```

This came up bootstrapping a real program (`wave`, an HTTP CLI) entirely on top
of `cljrs`: HTTP Basic auth wants base64, `cljrs-base64` already exists in the
workspace, but there is no way to consume it as a dependency — the only options
are to hand-roll the encoder or to fork/rebuild the `cljrs` binary with the
crate statically linked. Pure-Clojure git deps are affected by a related gap
(see Root Cause #2), so this is really "deps declared in `cljrs.edn` are not
brought in for a plain `require`," with the native case being the sharpest
example.

## Environment

- clojurust `963f0aa` (`crates/cljrs` version `0.1.0`)
- `cljrs` built with default features (`default = ["async", "net", "charset"]`)
- Linux x86_64

## Repro

`cljrs.edn`:

```clojure
{:deps {cljrs.base64 {:git/url    "https://github.com/csm/clojurust"
                      :git/sha    "963f0aaffbfb5c89991c95b515d57d811b95c81d"
                      :rust/crate "crates/cljrs-base64"
                      :rust/init  "cljrs_base64::cljrs_init"
                      :rust/load  :dylib}}
 :paths ["src"]}
```

`src/main.cljrs`:

```clojure
(ns demo)
(require '[cljrs.base64 :as base64])
(defn -main [& _]
  (println (base64/encode "user:pass")))   ; expect dXNlcjpwYXNz
```

```sh
cljrs deps fetch     # clones the repo into ~/.cljrs/cache/git/<slug> (bare repo)
cljrs run src/main.cljrs
```

### Expected

`dXNlcjpwYXNz` is printed — the native dependency's `cljrs.base64` namespace is
registered and `encode` is callable.

### Actual

```
Error: × in demo: runtime error: Could not find namespace cljrs.base64 on source path
```

The same failure occurs for a **pure-Clojure** git dep (e.g. `clojure.tools.cli`
via `csm/tools.cli`): after `cljrs deps fetch`, `(require '[clojure.tools.cli …])`
still reports "Could not find namespace … on source path". The only way to make
it resolve today is to put the dependency's checked-out source on `:paths`
manually (an absolute, machine-specific cache path), or to statically link the
crate into the binary.

## Root cause

Two independent gaps; together they mean nothing declared only in `:deps` is
reachable from a plain `require`.

### 1. The pinned-native loader is only reached via versioned *symbol* resolution

`cljrs-dylib::install` registers a pinned-native loader hook
(`set_pinned_native_loader` → `load_pinned`, `crates/cljrs-dylib/src/lib.rs`).
`load_pinned` does the right thing — it finds a `:rust/load :dylib` dep covering
the base namespace, builds the wrapper cdylib at the pinned commit, and registers
its exports.

But that hook is only invoked from `pinned_native_or_head_fallback`, which is
only called from `resolve_versioned_value` in
`crates/cljrs-env/src/versioned.rs` — i.e. when resolving a **versioned symbol**
`ns/sym@commit`. It is never reached by:

- a plain `(require '[cljrs.base64 :as base64])` — the unversioned namespace
  loader `do_load` (`crates/cljrs-env/src/loader.rs`) only consults
  `globals.builtin_source` and then `find_source_file(rel_path, source_paths)`;
  it never consults `deps_config` or the pinned-native loader, so it fails with
  "Could not find namespace … on source path"; nor by
- a *versioned namespace* require `(require '[cljrs.base64@<sha> …])` — that goes
  through `do_versioned_load`, which calls `fetch_versioned_source` and requires
  **Clojure source** for the namespace (it `find_source_file`s, then reads the
  file from git history). A pure-native crate has no `.cljrs`, so this errors too.

Net: the only thing that triggers native dep loading is writing a versioned
symbol literal such as `cljrs.base64/encode@<sha>` directly in code. There is no
plain-`require` path, and no way to alias it ergonomically.

### 2. Git deps are never added to the source path, and the fetch is a bare repo

`apply_deps_config` (`crates/cljrs/src/main.rs`) appends only the local
`config.paths` (the `:paths` from `cljrs.edn`) to `globals.source_paths`. It does
**not** add any fetched git-dep directories. So even a pure-Clojure git dep's
namespaces are never on the source path.

Compounding this, `cljrs deps fetch` populates `~/.cljrs/cache/git/<slug>` as a
**bare** clone (no working tree — only `objects/`, `refs/`, `HEAD`,
`packed-refs`). And `fetch_versioned_source`
(`crates/cljrs-env/src/versioned.rs`) resolves a versioned namespace by first
locating the namespace's file via `find_source_file(rel_path, source_paths)` and
then `find_repo_root` on it — i.e. it needs the file present on a source path,
inside a working tree. A bare cache repo satisfies neither, so even the
versioned path can't reach a git-dep's source.

## Why it matters

This blocks the natural "stand up a program on bare `cljrs`" workflow: declare a
dependency in `cljrs.edn`, `require` it, use it. It's most acute for native deps
(no source-only fallback at all — you must fork the binary), but it equally
prevents plain-`require` use of ordinary Clojure git deps such as
`clojure.tools.cli`.

## Suggested directions

1. **Add fetched git-dep source roots to `source_paths`.** In
   `apply_deps_config`, for each declared dep, resolve its on-disk location
   (checking out a working tree, or materializing the pinned tree from the bare
   cache) and append the dep's own `:paths` (from its `cljrs.edn`) to
   `globals.source_paths`. This alone fixes pure-Clojure deps like `tools.cli`.

2. **Consult the pinned-native loader from the unversioned `require` path.** When
   `do_load` cannot find a namespace on the source path, look up `deps_config`
   for a `:rust/load :dylib` dep covering the namespace and invoke the same
   `load_pinned` machinery (treating the dep's pinned `:git/sha` as the commit),
   so a plain `(require '[cljrs.base64 :as base64])` brings the native code in.

3. **Allow a non-bare (or worktree-materializing) fetch** so that
   `find_source_file` / `find_repo_root` can locate dep sources, making the
   existing versioned path usable for git deps too.

A minimal fix for the native case is (1)+(2): a dep declared with `:rust/load
:dylib` should be loadable by a plain `require` of its namespace.

## Workaround used in the meantime

`wave` calls `cljrs.base64/encode` and assumes the `cljrs.base64` namespace is
provided by the `cljrs` binary (the `cljrs-base64` crate linked in). Validated by
rebuilding `cljrs` with `cljrs-base64` statically registered at startup:
`(base64/encode "user:pass")` → `dXNlcjpwYXNz`.
