# Лэйаут проекта

A common Rust project layout usually looks like following:

```
.
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── module1.rs
│   └── module2/
│       ├── mod.rs
│       ├── module2_file1.rs
│       └── module2_file2.rs
└── tests/
    └── integration-tests.rs
```

## Многомодульный проект



```
my-multimodule-project/
  ├── Cargo.toml
  ├── core/
  │     ├── src/
  │     │     ├── lib.rs
  │     │     ├── config.rs
  │     │     ├── dao.rs
  │     │     ├── service.rs
  │     │     └── util.rs
  │     └── Cargo.toml
  ├── web/
  │     ├── src/
  │     │     ├── main.rs
  │     │     ├── routs.rs
  │     │     └── auth.rs
  │     └── Cargo.toml
  └── cli/
        ├── src/
        │     └── main.rs
        └── Cargo.toml

```
