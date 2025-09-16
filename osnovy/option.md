# Option

One of the commonly used enum from the standard library – Option, which is used as types wrapper for values that can lack value.

```
enum Option<T> {
    Some(T),
    None,
}
```

Comparing to Java, in Rust Option type is very important, since in Rust there is no null.

E.g. that’s how we can define a database entity:

```
struct PersonDbRecord {
    id: Option<i32>, //None before persisted
    name: String,
}
```

Same as in Java, in Rust we can transform Option values, and compose them with other Options.

```
fn main() {
    let o1: Option<i32> = Some(1);

    let o2: Option<i32> = o1.map(|a| { a + 1 }); // Some(2)

    let o3: Option<i32> = o2.and_then(|a| { Some(a + 1) }); // Some(3)

    let o4: Option<i32> = Some(Some(4)).flatten(); // Some(4)


    let i1 = o1.unwrap(); // 1 (may panic)

    let i2 = o2.unwrap_or(0); // 2

    let i3 = match o3 {
        Some(i) => i,
        None    => 0,
    };
}
```
