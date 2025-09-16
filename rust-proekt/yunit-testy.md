# Юнит тесты

It is a common practice to write test in the same file with the source code.

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

