---
hidden: true
---

# Юнит тесты

В Rust принято писать юнит-тесты в том же файле, в котором находятся тестируемые функции.

Юнит-тест — представляет из себя функцию помеченную аннотацией `#[test]`.

```
fn inc(a: i32) -> i32 {
    a + 1
}

#[test]
fn test_inc() {
    assert_eq!(inc(1), 2);
}

#[test]
#[should_panic = (expected = "divide by zero")]
fn test_div_by_zero() {
    1 / 0;
}
```

To run unit test we need to execute:

```
cargo test
```

