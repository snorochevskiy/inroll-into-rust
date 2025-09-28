---
description: Они же энамы
---

# Перечисления

Перечисления или просто "энамы" (enums) в Rust гораздо мощнее, чем в C, C++, Java других языках более старых поколений, и могут быть представлены в нескольких ипостасьях.

## Перечисление как в C

Если нам нужно перечисление как в C, то синтаксис его объявления следующий:

```rust
enum EnumName {
    Элемент1,
    Элемент2,
    …,
    ЭлементN
}
```

Например:

```rust
enum IpAddrKind {
    V4,
    V6,
}

fn main() {
    let ip_v4 = IpAddrKind::V4;
    let ip_v6 = IpAddrKind::V6;
}
```

Так же как и в Си, с элементами перечисления можно ассоциировать число, которое можно получить путём приведения элемента энама к `usize`:

```rust
enum HttpStatus {
    Ok = 200,
    NotModified = 304,
    NotFound = 404,
}

fn main() {
    println!("{}", HttpStatus::Ok as usize); // 200
}
```



## Перечисление как объединение типов

В отличии от C, в Rust перечисление может включать в себя разные типы (структуры и кортежи), и объекты перечисления могут принадлежать одному из этих типов.

Например, IP4 адрес кодируется 4 байтами, а IP6 адрес - 20 байтами, но IP4 адрес, как правило, записывают при помощи 4 чисел, а IP6 адрес, обычно, записывают строкой. Мы можем сделать перечисление, значение которого будет представлено либо кортежем из 4 байта, либо строкой:

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    let home = IpAddr::V4(127, 0, 0, 1);
    let loopback = IpAddr::V6(String::from("::1"));
}
```

Пример перечисления со структурами:

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}
fn main() {
    let square = Shape::Square { width: 4.0 };
}
```

Одним из преимуществ использования энамов является то, что ими удобно пользоваться в `match` операторе, где компилятор заставит проверить все варианты перечисления.

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}

fn calc_area(shape: &Shape) -> f32 {
    match shape { // Нужно проверить и Square, и Rectangle
        Shape::Square { width } => width * width,
        Shape::Rectangle { width, height } => width * height,
    }
}

fn main() {
    let square = Shape::Square { width: 4.0 };
    println!("{}", calc_area(&square));
}
```

Для перечислений можно добавлять методы точно так же, как и для структур:

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}

impl Shape {
    fn calc_area(&self) -> f32 {
        match self {
            Shape::Square { width } => width * width,
            Shape::Rectangle { width, height } => width * height,
        }
    }
}

fn main() {
    let square = Shape::Square { width: 4.0 };
    println!("{}", square.calc_area());
}
```

## if-let

Как мы уже сказали, оператор `match` заставит на перебрать все возможные варианты перечисления, одна если мы заинтересованы только в одном варианте, то мы можем использовать **if-let** — версия оператора `if` с деструктурирующим шаблоном.

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}
fn main() {
    let s = Shape::Square { width: 4.0 };
    if let Shape::Square { width } = s {
        println!("This is square of width {width}");
    }
}
```



## Лэйаут в памяти

При помощи энамов мы фактически можем хранить значения разных типов в массиве.

```rust
enum MyEnum {
    Byte(u8),
    UInt(u32),
}

fn main() {
    let arr = [MyEnum::Byte(1), MyEnum::UInt(5)];
}
```

Это достигается за счёт того, что под элементы энама резервируется такое количество памяти, которое необходимо для записи наибольшего из возможных ваниантов перечисления. То есть за гибкость приходится платиться потенциальным перерасходом памяти.

Так же, вначале блока хранящего значение энама, содержится дискриминатор — число (может быть от `u8` до `u32`), которое позволяет узнать какой именно из вариантов паречисления хранится здесь.

Массив из примера выше выглядит в памяти примерно так:

<img src="../.gitbook/assets/file.excalidraw (3).svg" alt="" class="gitbook-drawing">

## Еще раз про перечисление как в C

Теперь когда мы знаем как устроены перечисления, мы можем еще раз взглянуть на самый первый пример из этой главы, и понять как он устроен.

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

С семантической точки зрения, V4 и V6 - это никакие не значения, а просто типы синглтоны.
