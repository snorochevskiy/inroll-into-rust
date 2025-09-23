# Генерики

Генерики (generics - обобщение) - это механизм, который позволяет писать функциональность работающую со значениями, абстрагируясь от типов этих значений.

{% hint style="info" %}
Мы будем одинаково использовать названия и "генерик", и "обобщённый".
{% endhint %}

Например, если мы создаём структуру данных "список", то мы будем сфокусорованы на работе с элементами в списке (добавление элемента, удаление, поиск, вставка), а не над тем, какой тип у этих элементов. Функциональонсть списка не изменится в зависимости от того храним мы в нём числа или строки.

Уже знакомый нам, тип вектор - `Vec` как раз является генерик типом: в векторе мы можем хранить значения любого типа данных.

Для того, чтобы разобраться с генериками, давайте создадим максимально простой обобщённый тип — структуру-обёртку. Эта обёртка не имеет никакой специальной функциональности, она просто хранит в себе значение, но эта обёртка подходит для хранения значений любого типа.

```rust
struct Holder<T> {
    v: T
}
```

Здесь, `<T>`  — это, так называемый, генерик тип-аргумент (generic type argument): тип, над которым мы абстрагируемся. В примере, мы назвали наш абстрагированный тип как `T` - самое популярное имя для генерик типа-аргумента, но это имя может быть абсолютно любым, например "Element".

`Holder<T>` - это скорее не тип, а шаблон для создания типа. Когда компилятор встречает использование `Holder<T>` для значения какого типа, например `i32`, он генерирует конкретный тип `Holder<i32>`.

```rust
fn main() {
    let bool_holder: Holder<bool> = Holder { v: true };
    let i32_holder: Holder<i32> = Holder { v: 5 };
    let string_holder: Holder<String> = Holder { v: "aaa".to_string() };
}
```

Обобщёнными могут быть не только структуры, но и функци. Напишем для нашей генерик структуры `Holder<T>` соответствующую обобщённую функцию-конструктор:

```rust
fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}
```

Как видим, генерик аргумент для функции указывается так же как и для структуры: в угловых скобках после имени.

Теперь, мы можем создавать экземпляры `Holder` используя эту функцию:

```rust
fn main() {
  let bool_holder: Holder<bool> = make_holder(true);
  let i32_holder: Holder<i32> = make_holder(5);
  let string_holder: Holder<String> = make_holder("aaa".to_string());
}
```

При этом тип-генерик агрумент `T` у функции `make_holder` никак не связан с генерик аргументом `T` в объявлении структуры `Holder<T>`. Просто, как мы уже сказали, `T` - самое популярное имя. Если бы мы написали `fn make_holder<A>(v: A) -> Holder<A>`, то ничего бы не изменилось.

Чтобы сложить полную картину, давайте посмотрим как выглядят методы для генерик структур. Сделаем два метода: _get_ — для получение значения из нашей обёртки, и _set_ — для записи нового значения.

```rust
impl<T> Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}
```

В этой конструкции мы видим два генерик аргумента: после `impl`, и после `Holder`. Тот, который после `impl`, объявляет генерик аргумент - говорим, что в последующем блоке имплементации методов будет использовать генерик тип-аргумент `T`. Генерик аргумент в `Holder<T>` просто устанвливает связь между генерик аргументом всего блока `impl`, и генерик аргументом, в обобщённой структуре `Holder`. Немного запутано, но станет понятнее в более сложных примерах далее.

А теперь всё вместе:

```rust
struct Holder<T> {
    v: T
}

impl<T> Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}

fn main() {
    let mut h = make_holder(1);
    println!("{}", h.get()); // 1
    h.set(5);
    println!("{}", h.get()); // 5
}
```

## Мономорфизация генериков

Как мыуже сказали, шенерик типы вроде Holder\<T> — это скорее шаблоны типов, а конкретные типы получаются при параметризированнии генерика конкретным типом, например Holder\<i32>.

Каждый раз, когда компилятор встречает использование генерик типа с новым типом, он генерирует конкретный вариант генерика под этот, встреченный, тип. Это генерирование конкретных типов называют **мономорфизацией**.

Например мы использовали наш генерик тип Holder\<T> для типов i32 и bool. Компилятор сгенерирует два конкретных варианта Holder\<T>: для i32 и для bool. Примерно так (в реальности имено будут выглядить сложнее):

```
struct Holder_i32 {
    v: i32
}
struct Holder_bool {
    v: bool
}
```

То же самое произойдёт и с методами, и с функциями: для каждого типа будуте сгенерирован отдельный вариант функции.

<img src="../.gitbook/assets/file.excalidraw (12).svg" alt="" class="gitbook-drawing">

{% hint style="info" %}
В языках Java и C# так же есть свои генерики. В них одной из отличительных черт генериков является, так называемое, "стирание" генерик тип-аргумента при компиляции. Другими словами, в Java генерики используются просто как дополнительный контроль типов, но при компиляции информаци о генерик тип-аргументах изсчезает, и не доступна во время работы программы.\
В Rust генерики больше похожи на шаблоны из C++.
{% endhint %}

## Турборыба ::<>

TODO

## Генерик трэйты

Обобщёнными могут быть не только структуры и функции, но и трэйты.

```rust
trait CanBeAccessed<T> {
    fn get(&self) -> &T;
    fn set(&mut self, new_v: T);
}

trait HasGenericConstructor<T> {
    fn new(value: T) -> Self;
}

struct Holder<T> {
    v: T
}

impl<T> CanBeAccessed<T> for Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

impl<T> HasGenericConstructor<T> for Holder<T> {
    fn new(value: T) -> Self {
        Holder { v: value }
    }
}

fn main() {
    let mut h = Holder::new(5);
    h.set(7);
    println!("{}", h.get());
}
```

Как мы видим из примера, генерик трэйты ведут себя ровно так же, как и обычные, только теперь у них появился генерик тип-аргумент.

## Ассоциированные типы

Для генерик трэйтов существует альтернативный синтаксис для указания генерик тип-аргумента: через специальное поле — ассоциированный типа.

{% columns %}
{% column %}
Трэйт с генерик тип-аргументом

<pre class="language-rust"><code class="lang-rust"><strong>trait Трэйт&#x3C;A> {
</strong>    ...
}


struct S {
    ...
}

impl Трэйт&#x3C;i32> for S {
    ...
}

</code></pre>
{% endcolumn %}

{% column %}
Трэйт с ассоциированным типом

<pre class="language-rust"><code class="lang-rust"><strong>trait Трэйт {
</strong><strong>    type A;
</strong>    ...
}

struct S {
    ...
}

impl Трэйт for S {
    type A = i32;
    ...
}
</code></pre>
{% endcolumn %}
{% endcolumns %}



Перепишем с использованием ассоциированным и типами наш пример из раздела про генерик трэйты

```rust
trait CanBeAccessed {
    type ElementType;
    fn get(&self) -> &Self::ElementType;
    fn set(&mut self, new_v: Self::ElementType);
}

trait HasGenericConstructor {
    type TypeArg;
    fn new(value: Self::TypeArg) -> Self;
}

struct Holder<T> {
    v: T
}

impl<T> CanBeAccessed for Holder<T> {
    type ElementType = T;
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

impl<T> HasGenericConstructor for Holder<T> {
    type TypeArg = T;
    fn new(value: T) -> Self {
        Holder { v: value }
    }
}

fn main() {
    let mut h = Holder::new(5);
    h.set(7);
    println!("{}", h.get());
}
```

Разумеется в  блоке `impl` полю-ассоциированному типу можно присвоить и конкретный тип:

```rust
trait CanBeAccessed {
    type ElementType;
    fn get(&self) -> &Self::ElementType;
    fn set(&mut self, new_v: Self::ElementType);
}

struct I32Holder {
    v: i32
}

impl CanBeAccessed for I32Holder {
    type ElementType = i32;
    fn get(&self) -> &i32 {
        &self.v
    }
    fn set(&mut self, new_v: i32) {
        self.v = new_v;
    }
}
```



## Границы генериков

Когда мы объявляем генерик тип-аргумент для типа или функции, мы можем задать уточнение, что конкретный тип, на который этот генерик тип-аргумент будет заменён, должен реализовывать конкртеный трэйт.

```rust
fn my_func<T: Трэйт>(...) -> ... {
    ...
}
```

Если генерик тип-аргумент должен реализовывать несколько трейтов, то их надо перечислить через знак \`+\`

```
fn my_func<T: Трэйт1 + Трэйт2 + Трэйт3>(...) -> ... {
    ...
}
```

Давайте перепишем наш пример из главы [treity.md](treity.md "mention") с использованием границы генерика.

```rust
trait CanIntroduce {
    fn introduce(&self) -> String;
}

struct Person { name: String }
struct Dog    { name: String }

impl CanIntroduce for Person {
    fn introduce(&self) -> String {
        format!("Hello, I'm {}", self.name)
    }
}

impl CanIntroduce for Dog {
    fn introduce(&self) -> String {
        String::from("Waf-waf")
    }
}

fn print_introduction<T: CanIntroduce>(v: T) {
    println!("{}", v.introduce ());
}

fn main() {
    let person = Person {name: String::from("John")};
    let dog    = Dog    {name: String::from("Bark")};

    print_introduction(person); // Hello, I'm John
    print_introduction(dog);    // Waf-waf
}
```



## where

Для указания границы генерика существует альтернативный синтаксис - при помощи блока **where**.

{% columns %}
{% column %}
Без where блока

```rust
fn my_func<A: Трэйт1, B: Трэйт2>(
    аргумент1: A, аргумент2: B,
) -> Тип {
    ...
}
```
{% endcolumn %}

{% column %}
С where блоком

```rust
fn my_func<A, B>(
    аргумент1: A, аргумент2: B,
) -> Тип
where A: Трэйт1, B: Трэйт2 {
    ...
}
```
{% endcolumn %}
{% endcolumns %}

Блок `where` позволяет сделать сигнатуру функции более читабельной, так как делает объявление генериг тип-агрументов менее громоздкими.

Перепишем с использованием блока where функцию `print_introduction` из предыдущего примера.

```rust
fn print_introduction<T>(v: T) where T: CanIntroduce {
    println!("{}", v.introduce ());
}
```



## Границы генерика и impl Трэйт

Давайте еще раз взглянем на функию `print_introduction` из раздела "Границы генериков".

```rust
fn print_introduction<T: CanIntroduce>(v: T) {
    println!("{}", v.introduce ());
}
```

Очень похожая функияю у нас уже встречала в граве [treity.md](treity.md "mention").

```rust
fn print_introduction(v: &impl CanIntroduce) {
    println!("Value says: {}", v.introduce());
}
```

Как видите, эти две функции, фактически, делают одно и то же. В чём же принципиальное различие? На самом деле только в синтаксисе, и не более. И impl Трэйт, и генерик в итоге компилятором заменяются на конкретный тип, так что это две записи равноценны.

Однако есть отличия для более сложных генериков. Давайте еще раз вглянем на пример, который был в самом начале главы:

```rust
trait CanBeAccessed {
    type ElementType;
    fn get(&self) -> &Self::ElementType;
    fn set(&mut self, new_v: Self::ElementType);
}

struct Holder<T> {
    v: T
}

impl<T> CanBeAccessed for Holder<T> {
    type ElementType = T;
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}
```

Для этого кода, давайте напишем обобщённую функцию _create\_using_, которая будет принимать некое значение, заданное генерик тип-аргументом, и функцию, которая умеет это значение преобразовывать в некий тип, реализующий трэйт `CanBeAccessed`. В нашем примере этот трэйт реализуется только структурой, `Holder`, но теоретически мы можем дописать какие угодно реализации. Итак, давайте попробует написать эту функцию с использованием синтаксиса `impl Трэйт`.

```rust
fn transform_num<T>(
    v: T,
    f: impl Fn(T)->impl CanBeAccessed<ElementType=T>
) -> impl CanBeAccessed<ElementType=T> {
     f(v)
}
```

Теоертически всё верно, однако компилятор Rust не разрешает в сигнатурах функций иметь `impl Трэйт`, который в себе имеет другой `impl Трэйт`, а это как раз то, как выглядит второй аргумент функции. В таких ситуациях единственным вариантом остаётся сигнатара с использованием генериков.

```rust
fn create_using<T, R, F>(v: T, f: F) -> R
where
    F: Fn(T) -> R,
    R: CanBeAccessed<ElementType=T>
{
    f(v)
}
```

