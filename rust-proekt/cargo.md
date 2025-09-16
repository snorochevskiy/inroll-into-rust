# Cargo

Rust distribution comes with a powerful build tool called Cargo.

Similar to Maven, Cargo also able to automatically download required dependencies from a remote repository.

Crates (Rust packages) are distributed in a form of the source code, which makes the first compilation relative slow, but the resulting code is highly optimized.

Out of the box Cargo supports things like:

* building an executable binary
* building a shared library
* running unit / integration tests
* benchmarking
* compiling and testing examples (if we develop a library or a framework)

With the help of Cargo we can create a Rust project just with one command:

```
cargo new hello_world --bin
```

The project layout will look like:

```
├── Cargo.toml
└── src
    └── main.rs
```

Configuration consists if single Cargo.toml file which usually looks like:

```
[package]
name = "my-rust-aws-lambda"
version = "0.1.0"
edition = "2021"

[dependencies]
lambda_runtime = "0.6.0"
serde = "1.0.136"
tokio = { version = "1", features = ["full"] }
```

To build a release executable (with code optimizations):

```
cargo build --release
```
