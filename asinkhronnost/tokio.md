---
hidden: true
---

# Tokio

Tokio is a coroutines based runtime that is highly efficient for I/O bound applications.

Tokio provides an async runtime and the full set of non-blocking network and FS API (based on epoll and io\_uring).

```rust
use tokio::fs::File;
use tokio::io::AsyncReadExt;

#[tokio::main]
async fn main() {
    let mut file = File::open("/etc/fstab")
        .await
        .unwrap();

    let mut contents: Vec<u8> = Vec::new();
    file.read_to_end(&mut contents)
        .await
        .unwrap();

    println!("{}", String::from_utf8_lossy(&contents));
}
```

The code is absolutely linear, yet in lines 7 and 12, the fiber is taken off from the system thread.

## tokio::main

During the compilation, “async main” function (marked with #\[tokio::main]) is translated into a regular main function, that first creates an instance of Tokio runtime, and executes the async code.

E.g. the code

```
#[tokio::main]
async fn main() {
    // The code
}
```

will be translated to

```
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // The code
    })
}
```

## Scoped data

```
tokio::task_local! {
    pub static SESSION: SessinData;
}

#[derive(Debug, Clone)]
pub struct SessinData {
    name: String,
}

#[tokio::main]
async fn main() {

    let a = SESSION.scope(SessinData { name: "ololo".to_string() }, async move {
        proxy().await
    }).await;

    println!("{a}");
}

async fn proxy() -> String {
    get_from_scope().await
}

async fn get_from_scope() -> String {
    SESSION.get().name.clone()
}
```
