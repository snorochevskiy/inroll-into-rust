# Основные трейты

From the “Rust basics” presentation we already know about such standard traits as:

* `Debug` – defines “debug to string” method for the type
* `Clone` – defines _clone()_ method, that create a copy of the object
* `Copy` – marker trait, trait allows compiler to implicitly place _clone()_ in places where the object is moved.
* `Hash` – defines hash code to the type.
* `PartialEq`, `Eq` – defines comparison method for the type.
* `PartialOrd`, `Ord` – defines ordering for sorting.
* `Deref` – allows take a reference of a given type from the object.
* `AsRef`\<T> – defines _as\_ref()_ method that provides \&T
* `Iterator` – allows to iterate over object elements

There are also following commonly used traits:

* `Default` – defines method that created a default value for the type
* `From` / `Into` – defines explicit and implicit converting into a value of a given type
* `Drop` – defines method that is called when the object goes out of the scope
* `Sized` – marker trait that is added by the compiler if the type sized is know at compile time
* `Sync` – added by compiler to types that can be safely accessed concurrently
* `Send` – added by compiler to types that can be send to another thread

## Eq and PartialyEq

Eq requires == and != operations to be:

* reflexive: a == a
* symmetric: a == b implies b == a
* transitive: a == b and b == c implies a == c

PartialyEq – for types that don’t have full equality.

E.g. f32 and f64 are IEEE 754 compliant which means that a variable of f32 can have NAN (not a number) value, and the specification stands that NAN != NAN.

```
let a = f32::NAN;
let b = f32::NAN;
println!("{}", a == b) // false
```

