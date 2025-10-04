---
hidden: true
---

# Многомодульный проект



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
