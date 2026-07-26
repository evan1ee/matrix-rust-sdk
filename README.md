<h1 align="center">Matrix Rust SDK</h1>

<div align="center">
  <em>Your all-in-one toolkit for creating Matrix clients with Rust, from simple bots to full-featured apps.</em>
  <br />
  <img src="contrib/logo.svg">
  <hr />
  <a href="https://github.com/matrix-org/matrix-rust-sdk/releases">
    <img src="https://img.shields.io/github/v/release/matrix-org/matrix-rust-sdk?style=flat&labelColor=1C2E27&color=66845F&logo=GitHub&logoColor=white"></a>
  <a href="https://crates.io/crates/matrix-sdk/">
    <img src="https://img.shields.io/crates/v/matrix-sdk?style=flat&labelColor=1C2E27&color=66845F&logo=Rust&logoColor=white"></a>
  <a href="https://codecov.io/gh/matrix-org/matrix-rust-sdk">
    <img src="https://img.shields.io/codecov/c/gh/matrix-org/matrix-rust-sdk?style=flat&labelColor=1C2E27&color=66845F&logo=Codecov&logoColor=white"></a>
  <br />
  <a href="https://docs.rs/matrix-sdk/">
    <img src="https://img.shields.io/docsrs/matrix-sdk?style=flat&labelColor=1C2E27&color=66845F&logo=Rust&logoColor=white"></a>
  <a href="https://github.com/matrix-org/matrix-rust-sdk/actions/workflows/ci.yml">
    <img src="https://img.shields.io/github/actions/workflow/status/matrix-org/matrix-rust-sdk/ci.yml?style=flat&labelColor=1C2E27&color=66845F&logo=GitHub%20Actions&logoColor=white"></a>
</div>

<div align="center">

The Matrix Rust SDK is a collection of libraries that make it easier to build
[Matrix] clients in [Rust].
<br />
<br />

<picture>
  <source srcset="contrib/element-logo-light.png" media="(prefers-color-scheme: dark)">
  <source srcset="contrib/element-logo-dark.png" media="(prefers-color-scheme: light)">
  <img src="contrib/element-logo-fallback.png" alt="Element logo">
</picture>

<br />
<br />

Development of the SDK is proudly sponsored and maintained by
[Element](https://element.io). Element uses the SDK in their next-generation
mobile apps Element X on [iOS](https://github.com/element-hq/element-x-ios) and
[Android](https://github.com/element-hq/element-x-android) and has plans to
introduce it to the web and desktop clients as well.

The SDK is also the basis for multiple Matrix projects and we welcome
contributions from all.

</div>

## Purpose

The SDK takes care of the low-level details like encryption,
syncing, and room state, so you can focus on your app's logic and UI. Whether
you're writing a small bot, a desktop client, or something in between, the SDK
is designed to be flexible, async-friendly, and ready to use out of the box.

## Project structure

The Matrix Rust SDK is made up of several crates that build on top of each
other. The following crates are expected to be usable as direct dependencies:

- [matrix-sdk-ui](https://docs.rs/matrix-sdk-ui/latest/matrix_sdk_ui/) – A
  high-level client library that makes it easy to build full-featured UI clients
  with minimal setup. Check out our reference client,
  [multiverse](https://github.com/matrix-org/matrix-rust-sdk/tree/main/labs/multiverse),
  for an example.
- [matrix-sdk](https://docs.rs/matrix-sdk/latest/matrix_sdk/) – A mid-level
  client library, ideal for building bots, custom clients, or higher-level
  abstractions. You can find example usage in the
  [examples directory](https://github.com/matrix-org/matrix-rust-sdk/tree/main/examples).
- [matrix-sdk-crypto](https://docs.rs/matrix-sdk-crypto/latest/matrix_sdk_crypto/)
  – A standalone encryption state machine with no network I/O, providing
  end-to-end encryption support for Matrix clients and libraries. See the
  [crypto tutorial](https://docs.rs/matrix-sdk-crypto/latest/matrix_sdk_crypto/tutorial/index.html)
  for a step-by-step introduction.

All other crates are effectively internal-only and only structured as crates for
organizational purposes and to improve compilation times. Direct usage of them
is discouraged.

## Status

The library is considered production ready and backs multiple client
implementations such as Element X
[[1]](https://github.com/element-hq/element-x-ios)
[[2]](https://github.com/element-hq/element-x-android),
[Fractal](https://gitlab.gnome.org/World/fractal) and
[iamb](https://github.com/ulyssa/iamb). Client developers should feel confident
to build upon it.

## Bindings

The higher-level crates of the Matrix Rust SDK can be embedded in other
environments such as Swift, Kotlin, JavaScript, and Node.js. Check out the
[bindings/](./bindings/) directory to learn more about how to integrate the SDK
into your language of choice.

## License

[Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## What this fork changes

This fork adds **one method** to upstream `matrix-rust-sdk`. Everything else is
untouched.

| branch | contents |
|---|---|
| `main` | a plain mirror of `matrix-org/matrix-rust-sdk` |
| `develop` | `main` + the change below |

To take upstream updates, move `develop` onto the newer `main` — there is only
the one commit to carry across.

### `Timeline::send_sticker`

One method, in `bindings/matrix-sdk-ffi/src/timeline/mod.rs`:

```rust
pub async fn send_sticker(
    self: Arc<Self>,
    body: String,
    url: String,
    info: ImageInfo,
) -> Result<Arc<SendHandle>, ClientError>
```

**What it does.** Queues an `m.sticker` event on the room's send queue. The
media must already be uploaded — `url` is the `mxc://` URI you got back.

**Why it was needed.** The FFI could send every kind of `m.room.message` —
text, image, video, file — but not `m.sticker`. A sticker is its own event
type rather than a message subtype, so no existing method could produce one,
and a client had to send stickers as plain images instead.

**Why it's written this way.** It hands the event to the same
`Timeline::send` that every other sender uses, rather than talking to the
send queue directly. A sticker therefore gets exactly what a message gets:

- a local echo, so it appears in the timeline immediately instead of after
  the round trip,
- the send queue's retry behaviour when the network fails,
- thread relations, applied by the inner timeline.

Returning a `SendHandle` matches `send_image` and the other media senders, so
a queued sticker can be aborted the same way.

**Using it from Swift.** UniFFI generates the binding automatically:

```swift
let handle = try await timeline.sendSticker(body: body, url: mxcURI, info: imageInfo)
```

### Rebuilding the Swift bindings

[element-x-ios](https://github.com/evan1ee/element-x-ios) consumes this repo as
a local Swift package (`MatrixRustSDK: path: ../matrix-rust-sdk` in its
`project.yml`). That package builds against the **generated xcframework**, not
the Rust source, so editing Rust here changes nothing in the app until the
bindings are regenerated:

```sh
cargo xtask swift build-framework
```

The output lands in `bindings/apple/generated/` and is gitignored, which is why
switching branches in this repo doesn't disturb an app that is already
building. It also means a stale xcframework will keep compiling happily against
the old API — regenerate after every change here.

[Matrix]: https://matrix.org/
[Rust]: https://www.rust-lang.org/
