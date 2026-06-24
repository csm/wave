# Repro: native dep declared in `cljrs.edn` is not loadable by `require`

> **Update:** the "Could not find namespace" failure this repro was written for
> is **fixed** as of clojurust `8cd7b29`. With the fix, `require` now reaches the
> native-dep dylib builder; this same repro (underscored `:rust/init`) now fails
> one step later with a cargo package-name error — see
> [`../cljrs-dylib-pkgname-bug.md`](../cljrs-dylib-pkgname-bug.md).

Minimal, self-contained reproduction for
[`../cljrs-native-deps-issue.md`](../cljrs-native-deps-issue.md).

A native (`:rust/load :dylib`) dependency declared in `cljrs.edn` and
successfully fetched by `cljrs deps fetch` still cannot be `require`d.

## Files

- `cljrs.edn` — declares `cljrs.base64` as a native dep (the `cljrs-base64`
  crate from the clojurust repo).
- `src/main.cljrs` — `(require '[cljrs.base64 :as base64])` and calls `encode`.

## Run

```sh
cd notes/cljrs-native-deps-repro
cljrs deps fetch     # succeeds; `cljrs deps status` then reports it "cached"
cljrs run src/main.cljrs
```

## Expected

```
dXNlcjpwYXNz
```

(`cljrs.base64/encode "user:pass"` — standard base64.)

## Actual

```
Error: × in demo: runtime error: Could not find namespace cljrs.base64 on source path
```

The dependency is fetched and cached, but the namespace never becomes resolvable
to `require`.

## Verified on

clojurust `963f0aa`, `cljrs` `0.1.0`, default features (`async`, `net`,
`charset`). Output observed:

```
$ cljrs deps fetch
fetching cljrs.base64 (https://github.com/csm/clojurust)...
  ok → /root/.cljrs/cache/git/https___github_com_csm_clojurust
$ cljrs deps status
cljrs.base64: cached (sha: 963f0aa…, url: https://github.com/csm/clojurust)
$ cljrs run src/main.cljrs
Error: × Could not find namespace cljrs.base64 on source path
```

See the parent write-up for root-cause analysis and suggested fixes. The same
failure mode applies to pure-Clojure git deps (e.g. `clojure.tools.cli`): swap
the dep for `{tools.cli {:git/url "https://github.com/csm/tools.cli" :git/sha
"…"}}` and `(require '[clojure.tools.cli :as cli])` — `cljrs deps fetch` caches
it, but `require` still cannot find `clojure.tools.cli`.
