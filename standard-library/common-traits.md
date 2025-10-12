---
hidden: true
---

# Основные трейты

Мы уже успели познакомится со следующими трэйтами:

* [**Debug**](https://doc.rust-lang.org/std/fmt/trait.Debug.html) — объявляет функциональность, для получения отладочного текстового описания объекта. Используется при выводе макросом `println!` по средствам форматирующей комбинации `{:?}`
* [**Clone**](https://doc.rust-lang.org/std/clone/trait.Clone.html) — определяет метод `clone()`, который делает глубокую копию объекта
* [**Copy**](https://doc.rust-lang.org/std/marker/trait.Copy.html) — маркерный трэйт, который указывает компилятору, что при операции присваивания нужно, вместо перемещения владения, производить копирование путём вызова метода `clone()`
* [**Hash**](https://doc.rust-lang.org/std/hash/trait.Hash.html) — определяет метод для вычисления хэш-кода для значения типа
* [**PartialEq**](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html), [**Eq**](https://doc.rust-lang.org/std/cmp/trait.Eq.html) — определяет методы для сравнения объектов на равенство
* [**PartialOrd**](https://app.gitbook.com/u/dkQPoX1YHFcD5pDtm1LhqZnDeAW2), [**Ord**](https://doc.rust-lang.org/std/cmp/trait.Ord.html) — определяет методы для определения какой из объектов больше другого (испольуется при сортировке)
* [**Default**](https://doc.rust-lang.org/std/default/trait.Default.html) — задаёт метод-конструктор, который создаёт объект по умолчанию
* [**Deref**](https://doc.rust-lang.org/std/ops/trait.Deref.html) — позволяет брать ссылку на некие данные, которыми владеет объект. Компилятор поменяет выражение `&объект` на `объект.deref()`.
* [**Iterator**](https://doc.rust-lang.org/std/iter/trait.Iterator.html) — позволяет итерироваться по элементам объекта

Кроме вышеперечисленных, существуют еще такие широкоиспользуемые трэйты:

* [**From**](https://doc.rust-lang.org/std/convert/trait.From.html) / [**Into**](https://doc.rust-lang.org/std/convert/trait.Into.html) — определяют конвертирование из одного типа в другой
* [**AsRef\<T>**](https://doc.rust-lang.org/std/convert/trait.AsRef.html) — позволяет получить ссылку на внутренние данные, из ссылки на родительский объект
* [**Borrow**](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) — позволяет одалживать ссылку на значение, которым владее объект
* [**ToOwned**](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html) — позволяет получать объект (для владения) из ссылки (как правило, путём клонирования)
* [**Drop**](https://doc.rust-lang.org/std/ops/trait.Drop.html) — обявляет метод-деструктор, который вызывается при выходе объекта из скоупа
* [**Sized**](https://doc.rust-lang.org/std/marker/trait.Sized.html) — маркерный трэйт, который автоматически имплементируется компилятором, если размер типа известен на этапе компиляции
* [**Sync**](https://doc.rust-lang.org/std/marker/trait.Sync.html) — маркерный трэйт, который автоматически добавляется компилятором к типам, к значениям которых безопасно обращаться из нескольких потоков. О нём мы подробнее поговорим в главе про [mnogopotochnost.md](mnogopotochnost.md "mention").
* [**Send**](https://doc.rust-lang.org/std/marker/trait.Send.html) — маркерный трэйт, который автоматически добавляется компилятором для типов, чьи значения могут передаваться в другой поток. О нём мы так же поговорим в главе про [mnogopotochnost.md](mnogopotochnost.md "mention").

## Подробнее о Eq и PartialEq

Как мы упоминали ранее, в Rust существует два трэйта, которые используются для проверки на равенство:

*   `PartialEq` — частичное равенство

    ```rust
    pub trait PartialEq<Rhs = Self> where Rhs: ?Sized {
        fn eq(&self, other: &Rhs) -> bool;        // оператор ==
        fn ne(&self, other: &Rhs) -> bool { ... } // оператор !=
    }
    ```
*   `Eq` — равенство

    ```rust
    pub trait Eq: PartialEq { }
    ```

Трэйт `Eq` требует, чтобы методы `eq` и `ne`, которые соответствуют операторам `==` и `!=` были:

* рефлексивны: `a == a`
* симметричны: если `a == b`, то `b == a`
* транзитивны: если `a == b` и `b == c`, то `a == c`

Трэйт `PartialEq` не имеет таких ограничений, поэтому может быть использован для типов, для которых не определено полное равенство.

Например, `f32` и `f64` соответствуют спецификации IEEE 754, которая декларирует, что переменных этих типов могут содержать значение `NAN` (not a number), причём в соответствии с IEEE 754, `NAN != NAN`.

```rust
let a = f32::NAN;
let b = f32::NAN;
println!("{}", a == b) // false
```

Стандартные типы, которые реализуют `Eq`:

* целые числа: `i8` - `i128` и `u8` - `u128`
* булевый тип: `bool`
* строки: `&str` и `String`

Реализуют только `PartialEq`:

* числа с плавающей запятой: `f32`, `f64`.

## PartialOrd и Ord

Эти два трэйта используются для реализации сравнения на "больше, меньше или равно".

Как правило, реализация этих трэйтов необходима для использования объектов в функциональности, подразумевающей сортировку значений по возрастанию/убыванию.

Трэйт [PartialOrd](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html) используется для сравнения значений, которые не всегда могут быть сравнимы.

```rust
pub trait PartialOrd<Rhs = Self>: PartialEq<Rhs> where Rhs: ?Sized {
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;
    // Реализации по умолчанию
    fn lt(&self, other: &Rhs) -> bool { ... }
    fn le(&self, other: &Rhs) -> bool { ... }
    fn gt(&self, other: &Rhs) -> bool { ... }
    fn ge(&self, other: &Rhs) -> bool { ... }
}
```

Как мы видим, основной метод `partial_cmp` возвращает тип `Option<Ordering>`.

```rust
pub enum Ordering {
    Less = -1, Equal = 0 Greater = 1,
}
```

Если значения являются сравнимыми, то `partial_cmp` возвращает `Some(Ordering)`.\
Если значения являются несравнимыми, то `partial_cmp` возвращает `None`. За примером снова обратимся к типу `f32`.

```rust
println!("{:?}", 4.0.partial_cmp(&5.0));           // Some(Less)
println!("{:?}", f32::NAN.partial_cmp(&f32::NAN)); // None
```

Соответственно, трэйт [Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html) используется для задания строгого сравнения.

```rust
pub trait Ord: Eq + PartialOrd {
    fn cmp(&self, other: &Self) -> Ordering;
    // Реализации по умолчанию
    fn max(self, other: Self) -> Self where Self: Sized { ... }
    fn min(self, other: Self) -> Self where Self: Sized { ... }
    fn clamp(self, min: Self, max: Self) -> Self where Self: Sized { ... }
}
```

Для примера, создадим тип "Груз" с двумя полями: масса и флаг — является ли хрупким. Мы хотим иметь возможность сортировать вектор с грузами так, чтобы груз с меньшей массой оказывался перед грузом с большей массов, но все хрупкие грузы были перед всему нехрупкими.

```rust
#[derive(Debug, PartialEq)]
struct Cargo {
    weght: f32,
    fragile: bool,
}

impl Eq for Cargo {}

impl PartialOrd for Cargo {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for Cargo {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        if self.fragile && !other.fragile {
            return std::cmp::Ordering::Less;
        }
        if !self.fragile && other.fragile {
            return std::cmp::Ordering::Greater;
        }
        if self.weght < other.weght { std::cmp::Ordering::Less }
        else if self.weght > other.weght { std::cmp::Ordering::Greater }
        else { std::cmp::Ordering::Equal }
    }
}

fn main() {
    let mut v = vec![
        Cargo { weght: 2.0, fragile: false },
        Cargo { weght: 1.0, fragile: false },
        Cargo { weght: 3.0, fragile: true },
    ];
    v.sort(); // Требует, чтобы тип элемента реализовал Ord
    println!("{v:?}");
//Cargo{weght:3.0,fragile:true},Cargo{weght:1.0,fragile:false},Cargo{weght:2.0,fragile:false}
}
```

Обычно, логику сравнения пишут в реализации `PartialOrd` только в том случае, если реализация `Ord` для этого же типа не планируется. В противном случае реализация `PartialOrd` должна заключаться просто в перевызове `Ord::cmp`, как в примере выше.

## Default

Этот трэйт декларирует метод-конструктор, который позволяет создать объект по умолчанию.

```rust
pub trait Default: Sized {
    fn default() -> Self;
}
```

Трэйт `Default` определён для многих стандартных типов:

```rust
fn main() {
    println!("{}", i32::default());          // 0
    println!("{}", f32::default());          // 0
    println!("{}", bool::default());         // false
    println!("{}", String::default());       // ""
    println!("{:?}", Vec::<i32>::default()); // []
}
```

Многие функции стандартной бибилиотки работают с трэйтом `Default`. Например, тип `Option` имеет метод `unwrap_of_default()`, который возвращает значение по умолчанию, в случае вызова на `None`.

```rust
let o: Option<i32> = None;
let v = o.unwrap_or_default();
println!("{v}"); // 0
```

Реализовать `Default` для своего типа можно либо при помощи аннотации `derive`

```rust
#[derive(Debug, Default)]
struct Ip4Addr(u8, u8, u8, u8);

fn main() {
    let default_ip_addr = Ip4Addr::default();
    println!("{default_ip_addr:?}"); // Ip4Addr(0, 0, 0, 0)
}
```

либо вручную

```rust
#[derive(Debug)]
struct Ip4Addr(u8, u8, u8, u8);

impl Default for Ip4Addr {
    fn default() -> Self {
        Ip4Addr(127, 0, 0, 1)
    }
}

fn main() {
    let default_ip_addr = Ip4Addr::default();
    println!("{default_ip_addr:?}"); // Ip4Addr(127, 0, 0, 1)
}
```

## From и Into

Трэйты `From` и `Into` — два трэйта "собрата", которые используются для конвертации между типами.

Трэйт [**From**](https://doc.rust-lang.org/std/convert/trait.From.html) позволяет задать функциональность для преобразования из некого типа `T` в тип, для которого мы реализуем трэйт `From`. Трэйт декларирует метод-конструктор `from`, который используется для построения одного значения из другого:

```rust
pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}
```

Например, сделаем реализацию `From`, которая позволяет создать объект структуры "IP4 адрес" из массива.

```rust
#[derive(Debug)]
struct Ip4Addr(u8, u8, u8, u8);

fn ping(addr: Ip4Addr) {
    println!("Ping {addr:?}");
}

impl From<[u8; 4]> for Ip4Addr {
    fn from(value: [u8; 4]) -> Self {
        let [a, b, c, d] = value;
        Ip4Addr(a, b, c, d)
    }
}

fn main() {
    let arr = [127, 0, 0, 1];
    let addr = Ip4Addr::from(arr);
    ping(addr);
}
```

Трэйт [Into](https://doc.rust-lang.org/std/convert/trait.Into.html) делает тоже самое, только "с другой стороны":

```rust
pub trait Into<T>: Sized {
    fn into(self) -> T;
}
```

Если `From` обычно используется как конструктор — для явного создания объекта нужного нам типа из объекта другого типа, то `Into` используется при передаче аргументов, то есть: `arg: impl Into<T>`. Например:

<pre class="language-rust"><code class="lang-rust">#[derive(Debug)]
struct Ip4Addr(u8, u8, u8, u8);

<strong>fn ping(into_addr: impl Into&#x3C;Ip4Addr>) { // Принимаем аргумент как Into
</strong>    println!("Ping {:?}", into_addr.into());
}

impl Into&#x3C;Ip4Addr> for [u8; 4] {
    fn into(self) -> Ip4Addr {
        let [a, b, c, d] = self;
        Ip4Addr(a, b, c, d)
    }
}

fn main() {
    let arr = [127, 0, 0, 1];
    ping(arr);
}
</code></pre>

Однако, трэйт `Into` очень редко реализуют явно. Дело в том, что когда мы реализуем `From<X> for Y`, мы автоматически получаем и реализацию `Into<Y> for X`.

## AsRef

Трэйт [AsRef\<T>](https://doc.rust-lang.org/std/convert/trait.AsRef.html) декларирует метод `as_ref()`, который используется для того, чтобы из ссылки на объект получить другую ссылку — на внутреннее поле объекта, или другие данные которыми объект  владеет.

```rust
pub trait AsRef<T> where T: ?Sized {
    fn as_ref(&self) -> &T;
}
```

Этот трэйт используют для того, чтобы получать ссылку на данные в том виде, в котором они уже хранятся в объекте. Какие-то преобразования данных не предполагаются. Как правило, этот трэйт используется для указания типа аргумента функций, т.е. в функции тип аргумента задаётся как `impl AsRef<T>`.

Рассмотрим пример: имеется тип "Отправление", который инкапсулирует в себе "Товар" и "Адрес". Мы используем `AsRef` для получения доступа к этим внутренним полям.

```rust
struct Product(String);

struct Address(String);

struct Shipment {
    product: Product,
    address: Address,
}

impl AsRef<Product> for Shipment {
    fn as_ref(&self) -> &Product {
        &self.product
    }
}

impl AsRef<Address> for Shipment {
    fn as_ref(&self) -> &Address {
        &self.address
    }
}

fn process_product(prod: impl AsRef<Product>) {
    // ...
}

fn process_address(addr: impl AsRef<Address>) {
    // ...
}

fn main() {
    let s = Shipment {
        product: Product("laptop".to_string()),
        address: Address("In the middle of the nowhere".to_string()),
    };
    process_product(&s);
    process_address(&s);
}
```

{% hint style="info" %}
Теоретически, мы можем использовать `as_ref()` и не из другой функции, например:

```rust
let addr: &Address = s.as_ref();
```

или даже

```rust
let addr = AsRef::<Address>::as_ref(&s);
```

но, обычно, на практике это не имеет смысла.
{% endhint %}

Кроме `AsRef`, который предоставляет немутабельную ссылку, имеется соответствующий трэйт [AsMut](https://doc.rust-lang.org/std/convert/trait.AsMut.html), который возвращает мутабельную ссылку.

## Borrow

Сигнатурно, трэйт [Borrow](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) очень похож на `AsRef`: он так же служит для предоставления ссылки на некий внутренний объект.

```rust
pub trait Borrow<Borrowed> where Borrowed: ?Sized {
    fn borrow(&self) -> &Borrowed;
}
```

Единственная разница заключается в том, что вся функциональность определённая в стандартной библиотеке для трэйта `Borrow`, предполагает определённую связь реализаций некоторых трэйтов для владеющего объекта и для одолженной ссылки.

А именно: для неких объктов `x` и `y` типа `T`, который реализует `Borrow<R>`

* если `T` и `R` реализуют трэйт `Eq`, выражение `x.borrow() == y.borrow()` должно возвращать то же, что и `x == y`
* если `T` и `R` реализуют трэйт `Ord`, выражение `x.borrow().cmp(&y.borrow())` должно возвращать то же, что и `x.cmp(&y)`
* если `T` и `R` реализуют трэйт `Hash`, то хэш-код от `x` должен быть равен хэш-коду от `x.borrow()`

Существует так же трэйт [BorrowMut](https://doc.rust-lang.org/std/borrow/trait.BorrowMut.html), который возвращает мутабельную ссылку.

## ToOwned

Трэйт [ToOwned](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html) декларирует метод `to_owned()`, который позволяет из ссылки получить новый объект во владение.

```rust
pub trait ToOwned {
    type Owned: Borrow<Self>;

    fn to_owned(&self) -> Self::Owned;

    // Реализация по умолчанию
    fn clone_into(&self, target: &mut Self::Owned) { ... }
}
```

Часто `ToOwned` делает то же самое что и `Clone`, но в некоторых ситуациях, он возвращает другой тип.

Например, реализация `ToOwned` для `&str` возвращает объект `String`.

```rust
let slice: &str = "aaa";
let owned: String = slice.to_owned();
```

## Drop

Трэйт [Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) используется для задания функциональности, которая должна быть выполнена при выходе значения из скоупа. Как правило, при помощи этого трэйта задают деструктор.

Для типов, которые захватывают I/O ресурс, `Drop` можно использовать для освобождения этого ресурса. Для типов, которые владеют объектом в куче, `Drop` используют для освобождения памяти.

Например, сделаем структуру — примитивный аналог `Box`, которая инкапсулирует в себе указатель на кучу, и воспользуемся `Drop` для того, чтобы освободить аллоцированную память.

```rust
struct MyBox<T> {
    ptr: *mut T,
}

impl <T> MyBox<T> {
    fn new(val: T) -> MyBox<T> {
        let ptr = unsafe {
            let layout = std::alloc::Layout::for_value(&val);
            let ptr = std::alloc::alloc(layout) as *mut T;
            *ptr = val;
            ptr
        };
        MyBox {
            ptr
        }
    }
    fn get(&self) -> &T {
        unsafe { self.ptr.as_ref().unwrap() }
    }
    fn set(&self, new_val: T) {
        unsafe { *self.ptr = new_val; }
    }
}

impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe {
            std::alloc::dealloc(self.ptr as *mut u8, std::alloc::Layout::new::<T>());
        }
        println!("Released memory");
    }
}

fn main() {
    {
        let my_box = MyBox::new(5);
        println!("Boxed num: {}", my_box.get());
    } // my_box выходит из скоупа
    println!("Box memory is already released here");
}
```

Программа печатает:

```
Boxed num: 5
Released memory
Box memory is already released here
```

{% hint style="info" %}
Не акцентируйте внимание на работе с кучей из данного примера (особенно учитывая, что, обычно, работают с ней не совсем так). Высока вероятность, что при написании бекендов на Rust, вам никогда не придётся работать с кучей напрямую.
{% endhint %}

## Sized

Трэйт [Sized](https://doc.rust-lang.org/std/marker/trait.Sized.html) — маркерный трэйт, который автоматически реализуется компилятором для типов, чей размер известен на этапе компиляции.

Примером не `Sized` типа является тип `str`. Не `&str`, а именно `str`. Да, на самом деле тип строки литерала — это `str`, но мы всегда работаем с ними по средствам слайс-ссылки `&str`. А слайс-ссылка, как мы знаем, имеет известный и постоянный размер (два поля: указатель и длина). Таким образом `str` — не `Sized`, а `&str` — `Sized`.

Аналогично `str`, не `Sized` типом является и `[T]` — последовательность элементов в памяти неопределённого размера. Точто так же как и с `str`, с `[T]` мы работаем по средствам слайса `&[T]`, который — `Sized`.

Еще один пример не `Sized` типа — трэйт-объект (`dyn Трэйт`). Именно поэтому мы работает с трэйт объектами либо по ссылке `&dyn Трэйт`, либо путём упаковки в умный указатель `Box<dyn Трэйт>`.

При этом, тип `Vec` — `Sized`: хоть и размер буфера, хранимого в куче, не известен на этапе компиляции, но с точки зрения системы типов, тип `Vec` — это только та часть, которая хранится на стеке, а её размер известен. Аналогичным образом, тип `String` тоже `Sized`.

Как мы видим, все типы которые мы можем самостоятельно создать используя безопасный Rust (без unsafe и работы с памятью через указатель), являются `Sized`, даже если они хранят данные в куче.

***

Так же надо упомянуть, что по умолчанию, для всех генерик тип-аргументов компилятор задаёт неявную границу `Sized`.

То есть, когда мы пишем код:

```rust
fn my_func<T>() { ... }
```

компилятор расценивает его как:

```rust
fn my_func<T: Sized>() { ... }
```

Иногда  это ограничение нужно снять, для чего указывается явная трэйт граница `?Sized`:

```rust
fn my_func<T: ?Sized>() { ... 
```

Просматривая генерик код в сторонних бибилотеках или стандартной библиотеке, можно часто встретить такой послебление трэёт границы через `?Sized`.





