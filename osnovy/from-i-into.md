---
hidden: true
---

# From и Into

Трэйты From и Into — два трэйта собрата, которые используются для конвертации между типами.

## From

Трэйт [**From**](https://doc.rust-lang.org/std/convert/trait.From.html) позволяет задать функциональность для преобразования из некого типа `T` в тип, для которого мы реализуем трэйт. Он имеет вид:

```rust
pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}
```





```rust
pub trait Into<T>: Sized {
    fn into(self) -> T;
}
```



