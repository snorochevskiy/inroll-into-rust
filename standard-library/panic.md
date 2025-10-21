---
hidden: true
---

# Паника

## panic

Когда в программе происходит какой-то недопустимое действие, например, деление на ноль, или вызова `unwrap()` на объекте `None`, то происходит паника.

Например:

{% code lineNumbers="true" %}
```rust
fn main() {
    let a: Option<i32> = None;
    a.unwrap();
}
```
{% endcode %}

выведет

```
$ cargo run
thread 'main' panicked at src/main.rs:3:7:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Паника авариано завершает программу и печетает на стандартный вывод информацию о соучившейся ошибке.

Панику можно инициировать вручную при помощи макроса [panic](https://doc.rust-lang.org/std/macro.panic.html). Синтаксис его использования такой же как у макроса `println!`, то вместо вывода на консоль, он завершит программу и напечатает переданное сообщение.

Например, такой код:

```rust
fn main() {
    let a = 5;
    panic!("Ending the program here. Num: {}", a);
}
```

выведет:

```
$ cargo run
thread 'main' panicked at src/main.rs:3:5:
Ending the program here. Num: 5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Макрос `panic` принимает на вход форматирующую строку, но если необходимо пробросить не строковое значение через панику, это можно сделать при помощи функции [panic\_any](https://doc.rust-lang.org/std/panic/fn.panic_any.html).

```
pub fn panic_any<M: 'static + Any + Send>(msg: M) -> !
```



## Перехват паники



```rust
use std::panic;

fn main() {
    let result = panic::catch_unwind(|| {
        panic!("My panic msg");
    });
    println!("Result: '{:?}'", result);
    println!("Continue working...");
}
```

напечатает

<pre><code>$ cargo run
thread 'main' panicked at src/main.rs:5:9:
My panic msg
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
<strong>Result: 'Err(Any { .. })'
</strong>Continue working...
</code></pre>



todo, unreachable
