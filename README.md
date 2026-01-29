![quiche](quiche.svg)

[![crates.io](https://img.shields.io/crates/v/quiche.svg)](https://crates.io/crates/quiche)
[![docs.rs](https://docs.rs/quiche/badge.svg)](https://docs.rs/quiche)
[![license](https://img.shields.io/github/license/cloudflare/quiche.svg)](https://opensource.org/licenses/BSD-2-Clause)
![build](https://img.shields.io/github/actions/workflow/status/cloudflare/quiche/stable.yml?branch=master)

[quiche] 是 [IETF] 标准化的 QUIC 传输协议和 HTTP/3 的一个实现。它提供了处理 QUIC 数据包和管理连接状态的底层 API。应用程序需要负责提供 I/O（例如套接字处理）以及支持定时器的事件循环。

想了解更多关于 quiche 的由来及其设计思路的信息，可以阅读 Cloudflare 博客上的一篇[文章][post]，里面有更详细的介绍。

[quiche]: https://docs.quic.tech/quiche/
[ietf]: https://quicwg.org/
[post]: https://blog.cloudflare.com/enjoy-a-slice-of-quic-and-rust/

谁在使用 quiche？
----------------

### Cloudflare

quiche 为 Cloudflare 边缘网络的 [HTTP/3 支持][cloudflare-http3] 提供动力。[cloudflare-quic.com](https://cloudflare-quic.com) 网站可用于测试和实验。

### Android

Android 的 DNS 解析器使用 quiche 来[实现 DNS over HTTP/3][android-http3]。

### curl

quiche 可以[集成到 curl][curl-http3] 中以提供对 HTTP/3 的支持。

[cloudflare-http3]: https://blog.cloudflare.com/http3-the-past-present-and-future/
[android-http3]: https://security.googleblog.com/2022/07/dns-over-http3-in-android.html
[curl-http3]: https://github.com/curl/curl/blob/master/docs/HTTP3.md#quiche-version

快速开始
---------------

### 命令行应用

在深入了解 quiche API 之前，这里有几个关于如何使用作为 [quiche-apps](apps/) crate 一部分提供的 quiche 工具的示例。这些工具不适合生产环境；请参阅[免责声明和说明](#免责声明和说明)。

根据[构建](#构建)部分提到的命令克隆项目后，可以按如下方式运行客户端：

```bash
 $ cargo run --bin quiche-client -- https://cloudflare-quic.com/
```

而服务器可以按如下方式运行：

```bash
 $ cargo run --bin quiche-server -- --cert apps/src/bin/cert.crt --key apps/src/bin/cert.key
```

（注意，提供的证书是自签名的，不应在生产中使用）

使用 `--help` 命令行标志可以获取每个工具选项的更详细描述。

### 配置连接

使用 quiche 建立 QUIC 连接的第一步是创建一个 [`Config`] 对象：

```rust
let mut config = quiche::Config::new(quiche::PROTOCOL_VERSION)?;
config.set_application_protos(&[b"example-proto"]);

// 特定于应用程序和用例的额外配置...
```

[`Config`] 对象控制着 QUIC 连接的重要方面，例如 QUIC 版本、ALPN ID、流控制、拥塞控制、空闲超时以及其他属性或功能。

QUIC 是一种通用传输协议，有几个配置属性没有合理的默认值。例如，任何特定类型允许的并发流数量取决于运行在 QUIC 之上的应用程序以及其他特定于用例的问题。

quiche 将几个属性默认设置为零，应用程序很可能需要使用以下方法将它们设置为其他值以满足需求：
- [`set_initial_max_streams_bidi()`]
- [`set_initial_max_streams_uni()`]
- [`set_initial_max_data()`]
- [`set_initial_max_stream_data_bidi_local()`]
- [`set_initial_max_stream_data_bidi_remote()`]
- [`set_initial_max_stream_data_uni()`]

[`Config`] 也持有 TLS 配置。这可以通过对现有对象的修改器来改变，或者通过手动构建 TLS 上下文并使用 [`with_boring_ssl_ctx_builder()`] 创建配置。

配置对象可以在多个连接之间共享。

### 连接设置

在客户端，可以使用 [`connect()`] 工具函数创建新连接，而 [`accept()`] 用于服务器：

```rust
// 客户端连接。
let conn = quiche::connect(Some(&server_name), &scid, local, peer, &mut config)?;

// 服务器连接。
let conn = quiche::accept(&scid, None, local, peer, &mut config)?;
```

### 处理传入的数据包

使用连接的 [`recv()`] 方法，应用程序可以处理从网络接收到的属于该连接的数据包：

```rust
let to = socket.local_addr().unwrap();

loop {
    let (read, from) = socket.recv_from(&mut buf).unwrap();

    let recv_info = quiche::RecvInfo { from, to };

    let read = match conn.recv(&mut buf[..read], recv_info) {
        Ok(v) => v,

        Err(e) => {
            // 发生错误，处理它。
            break;
        },
    };
}
```

### 生成传出的数据包

传出的数据包使用连接的 [`send()`] 方法生成：

```rust
loop {
    let (write, send_info) = match conn.send(&mut out) {
        Ok(v) => v,

        Err(quiche::Error::Done) => {
            // 写入完成。
            break;
        },

        Err(e) => {
            // 发生错误，处理它。
            break;
        },
    };

    socket.send_to(&out[..write], &send_info.to).unwrap();
}
```

当数据包被发送时，应用程序负责维护定时器以响应基于时间的连接事件。定时器到期时间可以使用连接的 [`timeout()`] 方法获取。

```rust
let timeout = conn.timeout();
```

应用程序需要提供定时器实现，这可以特定于所使用的操作系统或网络框架。当定时器到期时，应该调用连接的 [`on_timeout()`] 方法，之后可能需要在网络上发送额外的数据包：

```rust
// 超时到期，处理它。
conn.on_timeout();

// 超时后根据需要发送更多数据包。
loop {
    let (write, send_info) = match conn.send(&mut out) {
        Ok(v) => v,

        Err(quiche::Error::Done) => {
            // 写入完成。
            break;
        },

        Err(e) => {
            // 发生错误，处理它。
            break;
        },
    };

    socket.send_to(&out[..write], &send_info.to).unwrap();
}
```

#### 数据包发送间隔控制

建议应用程序[控制][pace]传出数据包的发送间隔，以避免创建可能导致网络短期拥塞和数据包丢失的数据包突发。

quiche 通过 [`send()`] 方法返回的 [`SendInfo`] 结构的 [`at`] 字段公开传出数据包的发送间隔提示。此字段表示特定数据包应发送到网络的时间。

应用程序可以通过平台特定机制（例如 Linux 上的 [`SO_TXTIME`] 套接字选项）或自定义方法（例如使用用户空间定时器）人为延迟数据包的发送来利用这些提示。

[pace]: https://datatracker.ietf.org/doc/html/rfc9002#section-7.7
[`SO_TXTIME`]: https://man7.org/linux/man-pages/man8/tc-etf.8.html

### 发送和接收流数据

经过一些来回交互后，连接将完成其握手并准备好发送或接收应用数据。

可以使用 [`stream_send()`] 方法在流上发送数据：

```rust
if conn.is_established() {
    // 握手完成，在流 0 上发送一些数据。
    conn.stream_send(0, b"hello", true)?;
}
```

应用程序可以通过使用连接的 [`readable()`] 方法检查是否有任何可读流，该方法返回一个迭代器，遍历所有有待读取数据的流。

然后可以使用 [`stream_recv()`] 方法从可读流中检索应用数据：

```rust
if conn.is_established() {
    // 遍历可读流。
    for stream_id in conn.readable() {
        // 流可读，读取直到没有更多数据。
        while let Ok((read, fin)) = conn.stream_recv(stream_id, &mut buf) {
            println!("Got {} bytes on stream {}", read, stream_id);
        }
    }
}
```

### HTTP/3

quiche 的 [HTTP/3 模块] 提供了在 QUIC 传输协议之上发送和接收 HTTP 请求和响应的高级 API。

[`Config`]: https://docs.quic.tech/quiche/struct.Config.html
[`set_initial_max_streams_bidi()`]: https://docs.rs/quiche/latest/quiche/struct.Config.html#method.set_initial_max_streams_bidi
[`set_initial_max_streams_uni()`]: https://docs.rs/quiche/latest/quiche/struct.Config.html#method.set_initial_max_streams_uni
[`set_initial_max_data()`]: https://docs.rs/quiche/latest/quiche/struct.Config.html#method.set_initial_max_data
[`set_initial_max_stream_data_bidi_local()`]: https://docs.rs/quiche/latest/quiche/struct.Config.html#method.set_initial_max_stream_data_bidi_local
[`set_initial_max_stream_data_bidi_remote()`]: https://docs.rs/quiche/latest/quiche/struct.Config.html#method.set_initial_max_stream_data_bidi_remote
[`set_initial_max_stream_data_uni()`]: https://docs.rs/quiche/latest/quiche/struct.Config.html#method.set_initial_max_stream_data_uni
[`with_boring_ssl_ctx_builder()`]: https://docs.quic.tech/quiche/struct.Config.html#method.with_boring_ssl_ctx_builder
[`connect()`]: https://docs.quic.tech/quiche/fn.connect.html
[`accept()`]: https://docs.quic.tech/quiche/fn.accept.html
[`recv()`]: https://docs.quic.tech/quiche/struct.Connection.html#method.recv
[`send()`]: https://docs.quic.tech/quiche/struct.Connection.html#method.send
[`timeout()`]: https://docs.quic.tech/quiche/struct.Connection.html#method.timeout
[`on_timeout()`]: https://docs.quic.tech/quiche/struct.Connection.html#method.on_timeout
[`stream_send()`]: https://docs.quic.tech/quiche/struct.Connection.html#method.stream_send
[`readable()`]: https://docs.quic.tech/quiche/struct.Connection.html#method.readable
[`stream_recv()`]: https://docs.quic.tech/quiche/struct.Connection.html#method.stream_recv
[HTTP/3 模块]: https://docs.quic.tech/quiche/h3/index.html

请查看 [quiche/examples/] 目录以获取更完整的使用 quiche API 的示例，包括如何在 C/C++ 应用程序中使用 quiche 的示例（更多信息见下文）。

[examples/]: quiche/examples/

从 C/C++ 调用 quiche
-------------------------

quiche 在 Rust API 之上公开了一个[精简的 C API][thin C API]，可以更轻松地将 quiche 集成到 C/C++ 应用程序中（以及其他允许通过某种形式的 FFI 调用 C API 的语言）。C API 遵循与 Rust API 相同的设计，除了受 C 语言本身施加的限制。

运行 ``cargo build`` 时，一个名为 ``libquiche.a`` 的静态库将与 Rust 库一起自动构建。这是一个完全独立的库，可以直接链接到 C/C++ 应用程序中。

请注意，为了启用 FFI API，必须启用 ``ffi`` 功能（默认禁用），方法是将 ``--features ffi`` 传递给 ``cargo``。

[thin C API]: https://github.com/cloudflare/quiche/blob/master/quiche/include/quiche.h

构建
--------

quiche 需要 Rust 1.85 或更高版本才能构建。可以使用 [rustup](https://rustup.rs/) 安装最新的稳定版 Rust。

设置好 Rust 构建环境后，可以使用 git 获取 quiche 源代码：

```bash
 $ git clone --recursive https://github.com/cloudflare/quiche
```

然后使用 cargo 构建：

```bash
 $ cargo build --examples
```

也可以使用 cargo 运行测试套件：

```bash
 $ cargo test
```

请注意，用于实现基于 TLS 的 QUIC 加密握手的 [BoringSSL] 需要构建并链接到 quiche。使用 cargo 构建 quiche 时会自动完成此操作，但构建过程需要 `cmake` 命令可用。在 Windows 上还需要 [NASM](https://www.nasm.us/)。[官方的 BoringSSL 文档](https://github.com/google/boringssl/blob/master/BUILDING.md) 有更多细节。

或者，您可以通过使用 ``QUICHE_BSSL_PATH`` 环境变量配置 BoringSSL 目录来使用自己的自定义 BoringSSL 构建：

```bash
 $ QUICHE_BSSL_PATH="/path/to/boringssl" cargo build --examples
```

或者，您可以使用 [OpenSSL/quictls]。要启用 quiche 使用此供应商，可以将 ``openssl`` 功能添加到 ``--feature`` 列表中。请注意，如果使用此供应商，则不支持 ``0-RTT``。

[BoringSSL]: https://boringssl.googlesource.com/boringssl/

[OpenSSL/quictls]: https://github.com/quictls/openssl

### 为 Android 构建

可以使用 [cargo-ndk]（v2.0 或更高版本）为 Android（NDK 版本 19 或更高，推荐 21）构建 quiche。

首先需要安装 [Android NDK]，可以通过 Android Studio 或直接安装，并且需要将 `ANDROID_NDK_HOME` 环境变量设置为 NDK 安装路径，例如：

```bash
 $ export ANDROID_NDK_HOME=/usr/local/share/android-ndk
```

然后可以按如下方式安装所需 Android 架构的 Rust 工具链：

```bash
 $ rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android x86_64-linux-android
```

请注意，所有目标架构的最低 API 级别都是 21。

还需要安装 [cargo-ndk]（v2.0 或更高版本）：

```bash
 $ cargo install cargo-ndk
```

最后，可以使用以下过程构建 quiche 库。请注意，`-t <architecture>` 和 `-p <NDK version>` 选项是必需的。

```bash
 $ cargo ndk -t arm64-v8a -p 21 -- build --features ffi
```

更多信息请参见 [build_android_ndk19.sh]。

[Android NDK]: https://developer.android.com/ndk
[cargo-ndk]: https://docs.rs/crate/cargo-ndk
[build_android_ndk19.sh]: https://github.com/cloudflare/quiche/blob/master/tools/android/build_android_ndk19.sh

### 为 iOS 构建

要为 iOS 构建 quiche，您需要以下条件：
- 安装 Xcode 命令行工具。您可以使用 Xcode 或以下命令安装它们：

```bash
 $ xcode-select --install
```

- 为 iOS 架构安装 Rust 工具链：

```bash
 $ rustup target add aarch64-apple-ios x86_64-apple-ios
```

- 安装 `cargo-lipo`：

```bash
 $ cargo install cargo-lipo
```

要构建 libquiche，请运行以下命令：

```bash
 $ cargo lipo --features ffi
```

或者

```bash
 $ cargo lipo --features ffi --release
```

iOS 构建已在 Xcode 10.1 和 Xcode 11.2 中测试。

### 构建 Docker 镜像

要构建 Docker 镜像，只需运行以下命令：

```bash
 $ make docker-build
```

您可以在以下 Docker Hub 存储库中找到 quiche Docker 镜像：

- [cloudflare/quiche](https://hub.docker.com/repository/docker/cloudflare/quiche)
- [cloudflare/quiche-qns](https://hub.docker.com/repository/docker/cloudflare/quiche-qns)

每当 quiche 主分支更新时，`latest` 标签将被更新。

**cloudflare/quiche**

提供安装在 /usr/local/bin 中的服务器和客户端。

**cloudflare/quiche-qns**

提供在 [quic-interop-runner](https://github.com/marten-seemann/quic-interop-runner) 中测试 quiche 的脚本。

免责声明和说明
---------

⚠️ 此存储库包含许多客户端和服务器示例应用程序，用于演示 quiche 库 API 的简单用法。它们不适用于生产环境；不提供性能、安全性或可靠性保证。
⚠️ 由DeepSeek翻译。

版权
---------

版权所有 (C) 2018-2019, Cloudflare, Inc.

许可证请参见 [COPYING]。

[COPYING]: https://github.com/cloudflare/quiche/tree/master/COPYING
