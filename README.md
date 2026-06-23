# wave

A URL-fetching command-line tool in the spirit of `curl`/`wget`, written in
[Clojurust](https://github.com/csm/clojurust) (`.cljrs`). It is fully
asynchronous (built on `cljrs-net` over `clojure.core.async`) and speaks every
HTTP version up to HTTP/3.

## Status

Initial implementation. The HTTP/1.0, HTTP/1.1 and HTTP/2 clients are
implemented from the byte-stream layer up; HTTP/3 is delegated to cljrs-net's
native QUIC/HTTP-3 client.

| Version | Transport | Notes |
|---|---|---|
| HTTP/1.0 | TCP / TLS | `Connection: close`, one request per connection |
| HTTP/1.1 | TCP / TLS | Content-Length and chunked transfer decoding |
| HTTP/2   | TLS (`h2`) / cleartext (prior knowledge) | own framing + HPACK (static table + Huffman) |
| HTTP/3   | QUIC | via `clojure.rust.net.h3` |

## Building / running

`wave` depends on a cljrs build with the default `net`, `async` and `charset`
features and the `cljrs.base64` library (used for HTTP Basic auth), plus the
`clojure.tools.cli` git dependency declared in `cljrs.edn`.

```sh
# Fetch the git dependency (tools.cli) into ~/.cljrs/cache
cljrs deps fetch

# Run
cljrs run src/main.cljrs -- https://example.com/
```

## Usage

```
wave [options] <url> [url...]
```

| Option | Description |
|---|---|
| `-X, --request METHOD` | Request method (default GET, or POST with `-d`) |
| `-H, --header HEADER` | Add a request header (repeatable) |
| `-d, --data DATA` | Send a request body (implies POST) |
| `-o, --output FILE` | Write the body to FILE |
| `-O, --remote-name` | Write the body to a file named like the URL path |
| `-i, --include` | Include response headers in the output |
| `-I, --head` | Fetch headers only (HEAD) |
| `-L, --location` | Follow redirects |
| `--max-redirs N` | Maximum redirects (default 50) |
| `-A, --user-agent NAME` | Set the User-Agent |
| `-e, --referer URL` | Set the Referer |
| `-u, --user USER:PASS` | HTTP Basic authentication |
| `-k, --insecure` | Skip TLS certificate verification |
| `-s, --silent` | Suppress diagnostics |
| `-v, --verbose` | Print response status/headers as diagnostics |
| `--connect-timeout SECONDS` | Connection timeout |
| `--http1.0` / `--http1.1` | Force HTTP/1.0 / 1.1 (default 1.1) |
| `--http2` | Use HTTP/2 |
| `--http2-prior-knowledge` | HTTP/2 over cleartext without upgrade |
| `--http3` | Use HTTP/3 |
| `-h, --help` / `-V, --version` | Help / version |

### Examples

```sh
wave https://example.com/
wave -I https://example.com/
wave --http2 -H 'Accept: application/json' https://api.example.com/
wave --http3 https://cloudflare-quic.com/
wave -X POST -d 'a=1&b=2' http://httpbin.org/post
wave -L -o page.html http://example.com/redirect
```

## Architecture

Source lives under `src/`, one namespace per file:

| Namespace | Responsibility |
|---|---|
| `wave` (`main.cljrs`) | CLI entry point, version dispatch, redirects, output |
| `wave.cli` | Argument parsing (`clojure.tools.cli`) → request config |
| `wave.url` | URL parsing |
| `wave.http` | Shared header helpers, Basic-auth base64, defaults |
| `wave.http1` | HTTP/1.0 + 1.1 client |
| `wave.http2` | HTTP/2 framing client |
| `wave.hpack` | HPACK encode/decode (incl. Huffman) for HTTP/2 |
| `wave.http3` | HTTP/3 client (wraps `clojure.rust.net.h3`) |
| `wave.bytes` | Byte-array utilities over the async byte streams |

Each version client exposes an `^:async request` function returning a uniform
response map `{:version :status :reason :headers :body}` (or `{:error msg}`),
which `main` renders.

### Notes & limitations

- One request per connection (no keep-alive pooling yet).
- HTTP/2's HPACK encoder emits literal, non-Huffman fields (always valid); the
  decoder handles the full static/dynamic table and Huffman strings.
- No transparent decompression (`--compressed` is not yet implemented).
- Binary bodies printed to stdout are decoded as UTF-8; use `-o` to save bytes
  verbatim.
- `-main` is a thin synchronous wrapper over an `^:async` driver (the current
  cljrs runtime does not pass arguments to a variadic `^:async -main`).
