---
hidden: true
---

# Axum

## Создание сервера

```
cargo new test_axum
```

Cargo.toml

```
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```



```
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(hello));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

## Path parameters



```
use axum::{Router, extract::Path, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/hello", get(hello))
        .route("/hello", get(hello_path));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn handle_hello() -> &'static str {
    "Hello!"
}

async fn hello_path(Path(name): Path<String>) -> String {
    format!("Hello {}", name)
}
```



## Query parameters



```
use std::collections::HashMap;

use axum::{
    Router,
    extract::{Path, Query},
    routing::get,
};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/hello", get(hello))
        .route("/hello/{name}", get(hello_path));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello(Query(params): Query<HashMap<String, String>>) -> String {
    match params.get("name") {
        Some(name) => format!("Hello, {}!", name),
        None => "Hello!".to_owned(),
    }
}

async fn hello_path(Path(name): Path<String>) -> String {
    format!("Hello {}", name)
}
```







