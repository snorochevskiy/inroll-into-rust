---
description: Они же энамы
---

# Перечисления

Enum in Rust is a very powerful type.

```
enum EnumName {
    TypeElement1, TypeElement2, …, TypeElementN
}
```

It can be used just similarly to Java enums:

```
enum IpAddrKind {
    V4,
    V6,
}

// usage
let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```

It is also possible to associate a value with each enum element:

```
enum HttpStatus {
    Ok = 200,
    NotModified = 304,
    NotFound = 404,
}
```

These associated values can even be of different types:

```
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);
let loopback = IpAddr::V6(String::from("::1"));
```

Enums also can contain “nested” structs:

```
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}

fn calc_area(shape: Shape) -> f32 {
	match shape {
		Shape::Square { width } => width * width,
		Shape::Rectangle { width, height } => width * height,
	}
}

fn main() {
    let square = Shape::Square { width: 4.0 };
    println!("{}", calc_area(square));
}
```

In Java it would be usually implemented as a hierarchy:

```
abstract class Shape
class Square extends Shape { ... }
class Rectangle extends Shape { ... }
```



## if let

