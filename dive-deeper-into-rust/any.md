# Any

## Проблема даункаста

Как мы знаем, Rust не является ООП языком. Тем не менее, благодаря системе трэйтов, Rust предлагает мощнные возможности полиморфизма.

Благодаря [трэйт-объектам](../rust-basics/treity.md#dinamicheskaya-dispetcherizaciya), мы в любой момент, можем сделать "upcast": начать общаться с типом через его трэйт абстрагируясь от того, какой конкретно тип скрыт за трэйт-объектом.

```rust
trait Person {}

struct Student {}
impl Person for Student {}

struct Teacher {}
impl Person for Teacher {}

fn work_with_person(person: &dyn Person) {}

fn main() {
    let child1 = Student {};
    work_with_base(&child1); // upcast &Student к &dyn Person
    
    let child2 = Teacher {};
    work_with_base(&child2); // upcast &Teacher к &dyn Person
}
```

Но что делать в ситуации, когда необходимо узнать какой конкретный тип скрывается за трэйт-объектом? То есть, обращаясь к нашему примеру выше, имея ссылку `dyn Person`, получить из ней ссылуку `&Student` или `&Teacher`. Другими словами, сделать downcast.

<details>

<summary>Про upcast и downcast</summary>

Upcast и downcast — термины из мира ООП, где классы могут наследовать друг друга. Приведение указателя на объект дочернего класса к указателю на родительский класс называется upcast. Словно движение вверх (up) по иерархии. Соответственно приведение типа указатель с родительского класса на дочерний называется downcast.

Допустим у нас есть такая иерархия классов на C++:

```cpp
class Shape {
protected:
   float x, y; // location
}

class Circle : public Shape {
    float radius;
}
```

Тогда

```cpp
Circle* circle1 = new Circle();
Shape*  shape   = dynamic_cast<Shape*>(circle1); // upcast
Circle* circle2 = dynamic_cast<Circle*>(shape);  // downcast
```

</details>

К сожалению, на данный момент в Rust нет никакой возможности узнать какой реальный тип скрывается за трэйт объектом. Напомним, трэйт-объект — это просто пара указателей: первый — на сам объект, второй — на vtable. При этом, vtable не содержит название или ID типа, только: указатели на методы, указатель на деструктор (если таков имеется), а так же инофрмацию о размере и выравнивании.

Однако, есть обходной путь. В Rust каждому типу присваивается уникальный идентификатор — [TypeId](https://doc.rust-lang.org/std/any/struct.TypeId.html), который является просто обёрткой над 128-битным числом:

```rust
pub struct TypeId {
    t: (u64, u64),
}
```

Чтобы узнать `TypeId` для некого типа, используется метод-конструктор [TypeId::of](https://doc.rust-lang.org/std/any/struct.TypeId.html#method.of).

Например:

```rust
use std::any::TypeId;

struct Student {}
struct Teacher {}

fn main() {
    println!("{:?}", TypeId::of::<Student>()); // TypeId(0xaf8ebb053b28606d84daaf35f1eae84c)
    println!("{:?}", TypeId::of::<Teacher>()); // TypeId(0x318ec7010fde8de82dedcdc9562881fd)
}
```

Теперь, давайте перепишем самый первый пример из главы, добавив в него downcast.

```rust
use std::any::TypeId;

trait Person where Self: 'static {
    fn exact_type(&self) -> TypeId {
        TypeId::of::<Self>()
    }
}

/// Обратите внимание, что impl не для Person, а для dyn Person - трэйт-объекта
impl dyn Person {
    /// Этот метод даункастит трэйт-объект к ссылке на конкретный тип,
    /// если этот тип соответствует типу объекта сокрытого за трэйт-объектом
    fn downcast<T: 'static>(&self) -> Option<&T> {
        if TypeId::of::<T>() == self.exact_type() {
            unsafe {
                let (data, _vtable): (*const u8, *const u8) =
                    std::mem::transmute(self);
                let data: *const T = std::mem::transmute(data);
                data.as_ref()
            }
        } else {
            None
        }
    }
}

struct Student {
    name: String,
    year_of_education: u32,
}
impl Person for Student {}

struct Teacher {
    name: String,
    subject: String,
}
impl Person for Teacher {}

fn work_with_person(base: &dyn Person) {
    if let Some(s) = base.downcast::<Student>() {
        println!("This is {}, a {}-year student", s.name, s.year_of_education);
    } else if let Some(t) = base.downcast::<Teacher>() {
        println!("This is {}, a teacher of {}", t.name, t.subject);
    }
}

fn main() {
    let student = Student {
        name: "John".to_string(),
        year_of_education: 3,
    };
    work_with_person(&student);

    let teacher = Teacher {
        name: "Ivan".to_string(),
        subject: "Programming".to_string(),
    };
    work_with_person(&teacher);
}
```

Программа работает как от неё и требовалось:

```
$ cargo run
This is John, a 3-year student
This is Ivan, a teacher of Programming
```

Теперь давайт разберёмся в коде.

Для трэйта `Person` мы добавили метод, который возвращает `TypeId` того типа, который этот трэйт реализует. Если `Person` реализуется структурой `Student`, то вернётся `TypeId` для `Teacher`, а если структурой `Teacher`, то соответственно `TypeId` для `Teacher`.

```rust
trait Person where Self: 'static {
    fn exact_type(&self) -> TypeId {
        TypeId::of::<Self>()
    }
}
```

<details>

<summary>Про границу трэйта <code>'static</code>.</summary>

Граница трэйта `'static` говорит, что **объект** реализуюший этот трэйт не должен содержать ссылок, чей лайфтайм короче, чем `'static` лайфтайм.

Рассмотрим пример:

```rust
struct MyIntRef<'a> {
    r: &'a i32,
}

/// Функция, которая принимает объект со статическим лайфтаймом
fn prove_static<T: 'static>(v: T) {}

fn main() {
    // Это работает
    static VAR_STATIC: i32 = 5;
    let my1 = MyIntRef { r: &VAR_STATIC };
    prove_static(my1);

    // Это не скомпилируется
    let var_local = 5;
    let my2 = MyIntRef { r: &var_local }; // doesn not live long enough
    prove_static(my2);
}
```

Получается, что наш пример даункаста от _Person_ к _Student_/_Teacher_ не будет работать, если мы добавим еще одну стукруту, которая реализует трэйт `Person`, но содержит не `'static` ссылки? Да, именно так. Увы, но на сегодняшний существует такое ограничение. Впрочем, на практике, врядли вы когда-либо столкнётесь с ситуаций, когда вам нужно делать downcast для типа содержащего не `static` ссылки.

</details>

Далее в коде, специально для трэйт-объекта `dyn Person` мы определили метод, который проверяет, равно ли значение `TypeId` для типа, к которому хотят даункастить наш трэйт-обект, занчению `TypeId` для типа объекта, который сокрыт за трэйт-объектом (`Student` или `Teacher`). Если да, то мы извлекаем из трэйт-объекта указатель на данные, и приводит к желаемому типу ссылки. Иначе — возвращаем `None`.

```rust
impl dyn Base {
    fn downcast<T: 'static>(&self) -> Option<&T> {
        if TypeId::of::<T>() == self.exact_type() {
            unsafe {
                // Деструктурируем трэйт-объект в пару указателей
                let (data, _vtable): (*const u8, *const u8) =
                    std::mem::transmute(self);
                // Приводим сырой указатель, к указателю на конкретный тип
                let data: *const T = std::mem::transmute(data);
                // Превращаем указатель в ссылку: as_ref() возвращает Option
                data.as_ref()
            }
        } else {
            None
        }
    }
}
```

Чтобы понять каким образом мы полуаем ссылку на объект сокрытый за трэйт-объектом, мы должны еще раз вспомнить что из себя представляет трэйт-объект. Он состоит из пары указателей:

* первый — непосредственно на данные
* второй — на vtable (таблицу виртуальных вызовов).

Поэтому мы сначала преобразоваем `self` (трэйт-объект) в пару указателей, затем берём первый указатель и приводим его к ссылку желаемого типа.

Для приведения `self` к кортежу из двух указателей, мы используем функцию [transmute](https://doc.rust-lang.org/std/mem/fn.transmute.html). Эта функия позволяет как бы посмотреть на одну и ту же память (последовательность байт) через призму другого типа. Разумеется, этой функцией надо пользоваться очень осторожно, и учитывать не только размеры значений, но и возможные выравнивания. К счастью, в нашем случае мы имеем дело с двумя указателями, а они всегда имеют размер равный размеру машинного слова, поэтому не нуждаются в выравнивании.

На самом деле, фрагмент

```rust
let (data, _vtable): (*const u8, *const u8) = std::mem::transmute(self);
let data: *const T = std::mem::transmute(data);
data.as_ref()
```

можно записать куда проще:

```rust
Some(&*(self as *const dyn Person as *const T));
```

Однака, такая запись прячет те детали работы с трэйт-объектом, которые мы хотели продемонстрировать явно.

Остальной код из примера должен быть понятен.

## Даункаст с Any

Разумеется, реализовывать такой даункаст вручную неудобно и опасно, и в примере выше мы сделали это лишь для того, чтобы показать как происходит даункаст на уровне памяти.

На практике же, стандартная библиотека Rust предоставляет удобную (относительно) трэйт обёртку [**Any**](https://doc.rust-lang.org/std/any/trait.Any.html), которая инкапсулирует в себе `TypeId`, тем самым позволяя делать даункаст.

```rust
pub trait Any: 'static {
    fn type_id(&self) -> TypeId;
}

impl<T: 'static + ?Sized> Any for T {
    fn type_id(&self) -> TypeId {
        TypeId::of::<T>()
    }
}
```

Как видно из объявления, абсолютно для любого типа, реализующего `'static`, определена реализация `Any`, которая просто получает `TypeId` типа. По сути, это то же самое, что мы делали методом _exact\_type_ в примере выше.

Так же для `dyn Any` определены такие методы:

*   `pub fn is<T: Any>(&self) -> bool`

    Возвращает `true`, если реальный тип сокрытый за трэйт-объектом — это `T`
* `pub fn downcast_ref<T: Any>(&self) -> Option<&T>`\
  Если реальный тип сокрытый за трэйт-объектом — это `T`, то возвращает `Some(&T)`, иначе `None`.
* `pub fn downcast_mut<T: Any>(&mut self) -> Option<&mut T>`\
  Если реальный тип сокрытый за трэйт-объектом — это `T`, то возвращает `Some(&mut T)`, иначе `None`.

Пример просто даункаста по средствам `Any`:

```rust
use std::any::Any;

fn main() {
    let s = "hello".to_string();
    let any = &s as &dyn Any;

    println!("any is i32: {}", any.is::<i32>());       // any is i32: false
    println!("any is String: {}", any.is::<String>()); // any is String: true

    println!("{:?}", any.downcast_ref::<i32>());    // None
    println!("{:?}", any.downcast_ref::<String>()); // Some("hello")
}
```

Если трэйт наследует `Any`, то его трэйт-объект автоматически получает возможность делать даункаст.

Перепишем с использованием `Any` наш пример с самописным даункастом.

```rust
use std::any::Any;

trait Person: Any {}

struct Student {
    name: String,
    year_of_education: u32,
}
impl Person for Student {}

struct Teacher {
    name: String,
    subject: String,
}
impl Person for Teacher {}

fn work_with_person(person: &dyn Person) {
    let any = person as &dyn Any;
    if let Some(s) = any.downcast_ref::<Student>() {
        println!("This is {}, a {}-year student", s.name, s.year_of_education);
    } else if let Some(t) = any.downcast_ref::<Teacher>() {
        println!("This is {}, a teacher of {}", t.name, t.subject);
    }
}

fn main() {
    let student = Student {
        name: "John".to_string(),
        year_of_education: 3,
    };
    work_with_person(&student);

    let teacher = Teacher {
        name: "Ivan".to_string(),
        subject: "Programming".to_string(),
    };
    work_with_person(&teacher);
}
```

Результат работы программы такой же, как и с нашим собственным даункастом:

```
$ cargo run
This is John, a 3-year student
This is Ivan, a teacher of Programming
```
