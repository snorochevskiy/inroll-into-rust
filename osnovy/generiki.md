# Генерики

Генерики (generics - обобщение) - это механизм, который позволяет писать функциональность работающую со значениями, абстрагируясь от типов этих значений.

{% hint style="info" %}
Мы будем одинаково использовать названия и "генерик", и "обобщённый".
{% endhint %}

Например, если мы создаём структуру данных "список", то мы будем сфокусорованы на работе с элементами в списке (добавление элемента, удаление, поиск, вставка), а не над тем, какой тип у этих элементов. Функциональонсть списка не изменится в зависимости от того храним мы в нём числа или строки.

Уже знакомый нам, тип вектор - `Vec` как раз является генерик типом: в векторе мы можем хранить значения любого типа данных.

Для того, чтобы разобраться с генериками, давайте создадим максимально простой обобщённый тип — структуру-обёртку. Эта обёртка не имеет никакой специальной функциональности, она просто хранит в себе значение, но при этом такую обёртку можно создать для любого типа.

```rust
struct Holder<T> {
    v: T
}
```

После имени структуры в угловых скобках указывается генерик аргумент - тип, над которым мы абстрагируемся. Мы назвали его `T` - самое популярное имя для генерик типа, но имя может быть абсолютно любым.

Holder\<T> - это скорее не тип, а шаблон для создание типа. Когда компилятор встречает использование Holder\<T> для значениея какого типа, например i32, он генерирует конкретный тип Holder\<i32>.

```rust
fn main() {
    let bool_holder: Holder<bool> = Holder { v: true };
    let i32_holder: Holder<i32> = Holder { v: 5 };
    let string_holder: Holder<String> = Holder { v: "aaa".to_string() };
}
```

Оббщённой может быть не только структура, но и функци. Напишем для нашей генер структуры Holder\<T> соответствующую обобщённую функцию конструктор:

```rust
fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}
```

Как видим, генерик аргумент для функции указывается так же как и для структуры: в угловых скобках после имени.

Теперь, мы можем создавать экземпляры Holder используя эту функцию:

```rust
fn main() {
  let bool_holder: Holder<bool> = make_holder(true);
  let i32_holder: Holder<i32> = make_holder(5);
  let string_holder: Holder<String> = make_holder("aaa".to_string());
}
```

При этом генерик агрумент \`T\` у функции make\_holder никак не связан с генерик аргументом \`T\` у нашей структуры Holder\<T>. Просто, как мы уже сказали, T - самое популярное имя. Если бы мы написали \`fn make\_holder\<T>(v: T) -> Holder\<T>\`, то ничего бы не изменилось.

В качестве завершающего штриха, давайте посмотрим как выглядят методы для генерик структуры: сделаем метод для получение значения из нашей обёртки, и для записи нового значения.

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

В этой конструкции мы видим два генерик аргумента: после impl, и после Holder. Тот, который после impl, объявляет генерик аргумент - говорим, что в последующем блоке имплементации методов будет использовать генерик тип-аргумент T. Генерик аргумент в Holder\<T> просто устанвливает связь между генерик аргументом всего блока impl, и генерик аргументом, в обобщённой структуре Holder. Немного запутано, но станет понятнее в более сложных примерах далее.

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

В языках Java и C# так же есть свои генерики. В этих языках одной из отличительных черт генериков является, так называемое, стирание генерик аргумента при компиляции. Другими словами, в Java генерики используются просто как дополнительный контроль типов, но при компиляции информаци о генерик аргументах изсчезает, и не доступна во время работы программы.\
В Rust генерики больше похожи на шаблоны из C++.

## Generic associated types

For generic traits there is an alternative syntax for defining a generic parameter.

```
trait Describeable {
  type ResultType;
  fn describe(&self) -> Self::ResultType;
}

impl Describeable for i32 {
  type ResultType = String;
  fn describe(&self) -> String {
    self.to_string()
  }
}

fn print_describe(d: impl Describeable<ResultType=String>) {
  println!("{}", d.describe())
}

fn main() {
  print_describe(5)
}
```

This syntax allow to omit the generic parameter is some situations.

## Generic bounds

We can specify which traits the generic type should implement.

```
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

For specifying the a generic bound there is an alternative syntax which can be more readable in some situations.

Instead of

```
fn print_introduction<T: CanIntroduce>(v: T) {
  println!("{}", v.introduce());
}
```

we can also write

```
fn print_introduction<T>(v: T) where T: CanIntroduce {
  println!("{}", v.introduce ());
}
```

and this will actually have the same effect:

```
fn print_introduction(v: impl CanIntroduce) {
  println!("{}", v.introduce ());
}
```
