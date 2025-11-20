# Многопоточность

Когда речь заходит о параллельном программирования, Rust предоставляет:&#x20;

* традиционную многопоточность, основанную на потоках, управляемых операционной системой
* асинхронность, позволяющую выполнять функциональность на асинхронном рантайме, работающем в пользовательском пространстве (user space)

В это главе мы рассмотрим работу с потоками.

## Создание потока

Для создания нового потока используется функция [**std::thread::spawn**](https://doc.rust-lang.org/std/thread/fn.spawn.html):

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

В качестве аргумента функция `spawn` принимает замыкание `FnOnce`, которое запускается на новом потоке. Разумеется, можно передать не только `FnOnce`, но и `FnMut`, и `Fn` и `fn()`. Про трэйт `Send` мы поговорим немного позже.

Функция `spawn` возвращает объект типа [JoinHandle](https://doc.rust-lang.org/std/thread/struct.JoinHandle.html), который позволяет:

* получить информацию о потоке
* ожидать завершения потока
* получить результат, с которым завершилась функция потока

Рассмотрим простой пример: создадим два потока из функций, которые просто возвращают число:

```rust
use std::thread::{self, JoinHandle};

fn main() {
    let t1: JoinHandle<i32> = thread::spawn(||{ 1 });
    let t2: JoinHandle<i32> = thread::spawn(||{ 2 });

    let sum = t1.join().unwrap() + t2.join().unwrap();

    println!("{sum}"); // 3
}
```

Поток можно создать и из обычной функции:

```rust
use std::thread;

fn print_nums() {
    let thread_id = thread::current().id(); // ID текущего потока
    for i in 1 .. 5 {
        println!("thread: {thread_id:?}, num: {i}");
        thread::sleep(std::time::Duration::from_millis(100));
    }
}

fn main() {
    let t1 = thread::spawn(print_nums);
    let t2 = thread::spawn(print_nums);
    let _ = t1.join();
    let _ = t2.join();
}
```

Эта программа печатает:

```
thread: ThreadId(2), num: 1
thread: ThreadId(3), num: 1
thread: ThreadId(2), num: 2
thread: ThreadId(3), num: 2
thread: ThreadId(2), num: 3
thread: ThreadId(3), num: 3
thread: ThreadId(2), num: 4
thread: ThreadId(3), num: 4
```

На самом деле, функция `std::thread::spawn` просто перевызывает `std::thread::Builder::new().spawn()` — билдер, создающий поток. Если необходимо задать имя потока, или указать размер стека, то это можно сделать создав поток непосредственно при помощи билдера.

```rust
fn main() {
    let t = std::thread::Builder::new()
        .name("my_thread".to_string())
        .stack_size(8192)
        .spawn(|| {
            println!("thread name: {:?}", std::thread::current().name()) 
        })
        .unwrap();
    let _ = t.join();
}
```

## трэйт Send

Настало время поговорить о перемещении объектов между потоками, и о роли трэйта [**Send**](https://doc.rust-lang.org/std/marker/trait.Send.html).

Давайте взглянет на такой пример:

```rust
use std::thread;

fn main() {
    let user = "John Doe".to_string();
    let t1 = thread::spawn(move || {
        println!("{}", user);
    });
    let _ = t1.join();
}
```

Здесь всё просто: в главном потоке мы создаём объект строки, а дальше используем этот объект в порождённом потоке.

С ключевым словом `move` мы уже знакомы: оно означает, что если какое-то значение из внешнего контекста используется замыканием по ссылке, то владение этим объектом должно быть перемещено к замыканию.

Теперь давайте взглянем на небольшую модификацию этой программы: обернём строку в `Rc`.&#x20;

{% code lineNumbers="true" %}
```rust
use std::{rc::Rc, thread};

fn main() {
    let user = Rc::new("John Doe".to_string());
    let t1 = thread::spawn(move || {
        println!("{}", user);
    });
    let _ = t1.join();
}
```
{% endcode %}

Компиляция завершится с ошибкой:

```
error[E0277]: `Rc<String>` cannot be sent between threads safely
   --> src/main.rs:5:28
    |
5   |       let t1 = thread::spawn(move || {
    |                ------------- ^------
    |                |             |
    |  ______________|_____________within this `{closure@src/main.rs:5:28: 5:35}`
    | |              |
    | |              required by a bound introduced by this call
6   | |         println!("{}", user);
7   | |     });
    | |_____^ `Rc<String>` cannot be sent between threads safely
    |
    = help: within `{closure@src/main.rs:5:28: 5:35}`,
      the trait `Send` is not implemented for `Rc<String>`
```

Ошибка говорит, что объект типа `Rc` нельзя пересылать между потоками, так как тип `Rc` не реализует трэйт `Send`.

`Send` — маркерный трэйт, который указывает на то, что объект данного типа можно пересылать между потоками.

Объекты типа `Rc` не безопасно пересылать между потоками. Чтобы понять почему, давайте представим себе такой сценарий:

1. В главном потока мы создаём объект `Rc`. Как мы помним, [Rc состоит из двух полей](../rust-basics/smart-pointers.md#rc-sovmestnoe-vladenie): указатель на данные в куче и указатель на счётчик (количество владельцев), который так же расположен в куче.
2. Мы клонируем `Rc`, и получаем уже два объекта `Rc` ссылающихся на один и тот же объект в куче.&#x20;
3. Мы передаём второй объект `Rc` в другой поток.
4. Одновременно в главном потоке и во втором потоке мы клонируем объекты `Rc`.

Так как счётчик, находящийся в куче, никак не синхронизирован для многопоточного доступа, есть вероятность, что одновременное инкрементирование счётчика из разных потоков приведёт к записи некорректного значения. Отсюда легко понять, что `Rc` небезопасно пересылать между потоками, поэтому для `Rc` не `Send`:

Трэйт `Send` автоматически реализуется компилятором для любого типа, если он не содержит полей, которые не `Send`, например полей типа `PhontomData` или `*mut T`.

Сделаем свой тип, который не `Send`:

```rust
struct User {
    name: String,
    ptr: *mut u32, // *mut T указатель - не Send
}

// Эту функцию можно вызвать только параметризировав Send типом
fn prove_send<T: Send>() {}

fn main() {
    prove_send::<User>(); // ошибка, User не Send
}
```

Если нам всё таки понадобится иметь иметь возможность пересылать объекты типа `User` между потоками, т.е. сделаеть его `Send`, то мы всегда можем явно реализовать трэйт `Send` так:

```rust
struct User {
    name: String,
    ptr: *mut u32, // *mut T указатель - не Send
}

unsafe impl Send for User { } // явно реализуем Send

fn prove_send<T: Send>() {}

fn main() {
    prove_send::<User>(); // работает без ошибок
}
```

Разумеется, unsafe имплементация трэйта, явно говорит о том, что теперь вся ответственность по корректной работе с указателями ложится на плечи разработчика.

## Sync

Другой важный, в контексте многопоточности, трэйт — [**Sync**](https://doc.rust-lang.org/std/marker/trait.Sync.html). Этот трэйт тоже маркерный, и он автоматически реализуется компилятором для любого типа, если ссылку на значение этого типа можно безопасно использовать из разных потоков.

Формально говоря: `T` является `Sync`, если немутабельная ссылка `&T` является `Send`.

Большая часть стандратных типов реализует `Sync`. Как и в случае с `Send`, исключение составляют типы, которые инкапсулируют в себе указатель и при этом не предусматривают никакого механизма синхронизации работы с этим указателем из разных потоков. Например, такие  типы как `Rc` и `Cell`.

Рассмотрим пример, который демонстрирует, что `String` реализует `Sync`.

```rust
fn main() {
    static s: String = String::new();
    let r1 = &s;
    let r2 = &s;

    let t1 = std::thread::spawn(move || {
        println!("{}", r1);
    });
    let t2 = std::thread::spawn(move || {
        println!("{}", r2);
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

Так как `String` является `Sync`, мы може использовать ссылки на один и тот же объект из разных потоков.

Так же, в этом примере нам пришлось объявить строку как статическую переменную. Это необходимо ввиду того, что в текущей реализации компилятор не может отследить, что запущенный поток не переживёт скоуп, которому принадлежит переменная владеющая захваченным объектом `String`. А объявив переменную как `static`, мы продлеваем время её жизни до момента завершения программы.

Теперь давайте заменим `String` на `Cell`, который не реализует `Sync` и попытаемся скомпилировать программу.

```rust
use std::cell::Cell;

fn main() {
    static c: Cell<i32> = Cell::new(5);

    let t1 = std::thread::spawn(move || {
        println!("{:?}", &c);
    });
    let t2 = std::thread::spawn(move || {
        println!("{:?}", &c);
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

Ожидаемо, компилятор выдал ошибку сообщающую, что `Cell` не реализует `Sync`\`:

```
error[E0277]: `Cell<i32>` cannot be shared between threads safely
 --> src/main.rs:4:15
  |
4 |     static c: Cell<i32> = Cell::new(5);
  |               ^^^^^^^^^ `Cell<i32>` cannot be shared between threads safely
  |
  = help: the trait `Sync` is not implemented for `Cell<i32>`
  = note: shared static variables must have a type that implements `Sync
```

Как и в случае с `Send` мы можем явно реализовать `Sync` для нашего типа, при этом вся ответственность за безопасную работу с несинхронизированными полями ляжет на нас.

```rust
unsafe impl Sync for НашТип {}
```

## Механизмы синхронизации

Теперь когда мы познакомились с `Send` и `Sync`, мы готовы рассмотреть механизмы синхронизации доступа к данным из разных потоков. Стандартная библиотека Rust предлагает такие механизмы синхронизации:

* [**Mutex**](https://doc.rust-lang.org/std/sync/struct.Mutex.html) — позволяет в любую единицу времени иметь эксклюзивный доступ к ресурсу только для одного потока
* [**RwLock**](https://doc.rust-lang.org/std/sync/struct.RwLock.html) — позволяет либо множественный доступ для чтения, либо эксклюзивный доступ для записи
* [**Condvar**](https://doc.rust-lang.org/std/sync/struct.Condvar.html) — позволяет одному потоку уснуть и ждать пока другой поток не пробудит его
* [**Barrier**](https://doc.rust-lang.org/std/sync/struct.Barrier.html) — позволяет синхронизировать между собой несколько потоков в некоторой точке

### Mutex

Как и в других язык программирования в Rust, мьютекс — механизм, который позволяет только одному потоку получить эксклюзивный доступ к ресурсу. Все остальные потоки, желающие в этот момент получить доступ к ресурсу, находятся в состоянии ожидания до момента, пока поток захвативший ресурс, не отпустит его.

Мьютекс представлен типом [`std::sync::Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html). Чтобы обернуть объект в мьютекс, используется метод конструктор `new`:

```rust
use std::sync::Mutex;

fn main() {
    let m: Mutex<i32> = Mutex::new(5);
}
```

Для того, чтобы получить доступ к объекту внутри мьютекса, используется метод `lock()`. Это метод позволяет получить объект умного указателя [**MutexGuard**](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html), которыйпредоставляет мутабельную ссылку на объект (`MutexGuard` реализует трэйт `DerefMut`).

```rust
use std::sync::{Mutex, MutexGuard, PoisonError};

fn main() {
    let m: Mutex<i32> = Mutex::new(5);
    {
        // захватываем ресурс мьютекса
        let lock_attempt: Result<MutexGuard<'_, i32>, PoisonError<_>> = m.lock();
        let mut guard = lock_attempt.unwrap();
        *guard = 6;
    } // отпускаем мьютекс
}
```

Мы не зря вызвали `lock` в отдельном блоке из фигурных скобок — скоупе. Дело в том, что объект `MutexGuard` представляет из себя эксклюзивно захваченный ресурс. Пока он существует, ресурс мьютекса остаётся захваченным, как только он уничтожается, мьютекс освобождается. Именно поэтому желательно минимизировать область, в которой существует объект `MutexGuard`.

Теперь поговорим о работе с мьютексом из разных потоков. Как мы знаем, на потоке выполняется функция или замыкание. В обоих случаях, чтобы поток мог работать с неким объектом, этот объект надо отдать потоку во владение. Проблема в том, что мы не можем отдать один объект мьютекса во владение двум потокам. Здесь на помощь приходит умный указатель `Arc` (Atomic Reference Counting) — потокобезопасная версия `Rc`. Мы просто обрачиваем мьютекс в Arc, и это позволяет передавать разделяему ссылку на мьютекс в разные потоки.

```rust
use std::{sync::{Arc, Mutex}, thread};

fn main() {
    let m_original: Arc<Mutex<i32>> = Arc::new(Mutex::new(5));

    let m_clone = m_original.clone();
    let t = thread::spawn(move|| {
        if let Ok(mut guard) = m_clone.lock() {
            *guard = 6;
        }
    });
    let _ = t.join();

    println!("{m_original:?}"); // Mutex { data: 6, poisoned: false, .. }
}
```

Рассмотрим пример счётчика, который одновременно инкрементируется из двух разных потоков.

```rust
use std::{sync::{Arc, Mutex}, thread::{self, JoinHandle}};

fn start_counter_thread(counter: Arc<Mutex<i32>>) -> JoinHandle<()> {
    thread::spawn(move || {
        for _ in 0 .. 1000 {
            if let Ok(mut guard) = counter.lock() {
                *guard += 1;
            }
        }
    })
}

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let t1 = start_counter_thread(counter.clone());
    let t2 = start_counter_thread(counter.clone());

    let _ = t1.join();
    let _ = t2.join();

    println!("{counter:?}"); // Mutex { data: 2000, poisoned: false, .. }
}
```

{% hint style="warning" %}
#### Про лайфтайм mutex guard

В случае, когда со значением завёрнутым в мьютекс, надо выполнить только одно действие, можно обойтись одним выражением `lock()` без открытия нового скоупа. Например, если мы хотим извлечь элемент из вектора, завёрнутого в мьютекс, то мы можем написать такое:

{% code lineNumbers="true" %}
```rust
let list: Mutex<Vec<i32>> = ...;
let last: Option<i32> = list.lock().unwrap().pop();
if let Some(element) = last {
    process_element(element);
}
```
{% endcode %}

Здесь в выражении на строке (2), вызовом `.lock()` создаётся объект guard, который сразу же и уничтожается по завершении выражения. Таким образом, область захвата мьютекса ограничивается только этим одним выражением.

Может появится соблазн, сократить этот код таким образом:

{% code lineNumbers="true" %}
```rust
let list: Mutex<Vec<i32>> = ...;
if let Some(element) = list.lock().unwrap().pop() {
    process_element(element);
}
```
{% endcode %}

Однако, если вы ожидали, что мьютекс будет использован только получения элемента, а затем сразу же освободиться, то это работает не так. Объекты созданные в заголовке выражений if-let и while-let живут до самого конца скоупа этих выражение. То есть в течении всего выполнения функции `process_element`, мьютекс будет оставаться захваченным.
{% endhint %}

#### Отравленый мьютекс

Если поток, уже захвативший мьютекс, завершается с паникой, то мьютекс помечается как **отравленный** (poisoned).\
Если другой поток попытается захватить отравленный мьютекс, то вызов метода `.lock()` вернет ошибку `PoisonError`.

```rust
use std::{sync::{Arc, Mutex}, thread};

fn main() {
    let m = Arc::new(Mutex::new(5));

    let m1 = m.clone();
    let t1 = thread::spawn(move|| {
        let guard = m1.lock().unwrap();
        // Инициируем панику, чтобы отрпавить мьютекс
        panic!("poisoning mutex...");
    });
    let _ = t1.join();

    // Проверяем отравлен ли мьютекс
    println!("{}", m.is_poisoned()); // true

    let lock_attempt_1 = m.lock();
    // Отравленный мьютекс вместо Ok(Guiard) возвращает PoisonError
    println!("{lock_attempt_1:?}"); // Err(PoisonError { .. })

    // При этом, из PoisonError всё равно можно извлечь объект Guard.
    // Извлекая Guard из PoisonError мы явно понимаем, что мьютекс отправлен.
    if let Err(e) = lock_attempt_1 {
        let guard = e.into_inner();
        println!("Value: {}", *guard); // 5
    }

    // Отравленный мьютерс можно вернуть к нормальному состоянию.
    m.clear_poison();
    println!("{}", m.is_poisoned()); // false
}
```

Как мы видим, основнное назначение `PoisonError` — явно проинформировать, что другой поток, захвативший мьютекс, завершился с паникой. В некоторых сценариях, эта информация поможет нам корректно завершить, или откатить операцию, которую не смог выполнить поток завершившийся с паникой.

### RwLock

Если мьютекс предоставляет только эксклюзивный доступ к ресурсу, то [**RwLock**](https://doc.rust-lang.org/std/sync/struct.RwLock.html) разделяет доступ на чтение и доступ на запись. Сколько угодно потоков могут одновременно захватывать ресурс на чтение, но захват на записать происходит эксклюзивно как у мьютекса.

```rust
use std::sync::RwLock;

fn main() {
    let rw_lock = RwLock::new(5);

    { // Захват на чтение
        let lock_attempt = rw_lock.read();
        if let Ok(guard) = lock_attempt {
            println!("Read: {}", *guard);
        }
    }

    { // Захват на запись
        let lock_attempt = rw_lock.write();
        if let Ok(mut guard) = lock_attempt {
            *guard = 10;
            println!("Updated: {}", *guard);
        }
    }

    println!("{rw_lock:?}");
}
```

Аналогично мьютексу, `RwLock` тоже надо "заворачивать" в `Arc`, чтобы передать в несколько потоков.

`RwLock` становится отравленным, только если поток завершившийся с паникой, произвёл захват для записи. Паника при захвате для чтения, не приводит к отравлению `RwLock`.

### Condvar

Тип [**Condvar**](https://doc.rust-lang.org/std/sync/struct.Condvar.html) позволяет одному потоку ожидать, пока другой поток изменит значение некой переменной на ожидаемое.&#x20;

`Condvar` всегда используется в паре с мьютексом, и принцип этого взаимодействия проще понять на примере.

В качестве примера возьмём классическую для `Condvar` задачу: один поток должен ожидать пока другой поток, не выполнит какое-то действие.

<pre class="language-rust" data-line-numbers><code class="lang-rust">use std::sync::{Arc, Mutex, Condvar};
use std::thread;

fn main () {
    // Создаём Convdar с булевым флагом
    let cond = Arc::new((Mutex::new(false), Condvar::new()));

    let cond_copy = Arc::clone(&#x26;cond);
    thread::spawn(move || {
        // Этот поток эмулирует некие приготовления,
        // в конце которых condvar флаг будет выставлен в true
        let (mutex, cvar) = &#x26;*cond_copy;
        let mut flag_guard = mutex.lock().unwrap();
        *flag_guard = true;
<strong>        cvar.notify_one();
</strong>    });

    let (mutex, cvar) = &#x26;*cond;
    let mut flag_guard = mutex.lock().unwrap();
    // Здесь мы ожидаем пока в порождённом потоке флаг не будет выставлен в true
    while !(*flag_guard) {
<strong>        flag_guard = cvar.wait(flag_guard).unwrap();
</strong>    }
}
</code></pre>

Здесь в строке (22) главный поток "засыпает" на вызове `Condvar::wait`, и ожидает пока на этом же объекте `Condvar` не будет вызвано `notify_one`. После вызова `notify_one` во втором потоке, главный поток просыпается, и проверяет обновлённое значение флага, скрытое за объектом `MutexGuard`.

Обратите внимание, что передача `MutexGuard` в вызов `wait` освобождает мьютекс. Без этого, второй поток не смог бы захватить мьютекс в строке (13), и прозошёл бы, так называемый, dead lock.

Кроме `notify_one`, у Condvar существует еще метод `notify_all`, который можно использовать, если несколько потоков вызвали `wait` на одном объекте Condvar. То есть, `notify_one` пробуждает только один поток вызвавший `wait`, а `notify_all` пробуждает все.

{% hint style="info" %}
Вы могли заметить, что нам приходится склонировать переменную типа Arc, чтобы передать её в другой поток.
{% endhint %}

### Barrier

[**Barrier**](https://doc.rust-lang.org/std/sync/struct.Barrier.html) **(барьер)** — механизм, который позволяет набору потоков ожидать, пока все из них не будут готовы начать работу.

```rust
use std::sync::{Arc, Barrier, Mutex};
use std::thread;

const WORKERS_NUM: usize = 10;

fn main() {
    let data = (0..100).collect::<Vec<_>>();
    let mutex = Arc::new(Mutex::new(data));
    let barrier = Arc::new(Barrier::new(WORKERS_NUM));

    let mut workers = Vec::new();
    for _ in 0 .. WORKERS_NUM {
        let mutex_clone = mutex.clone();
        let barrier_clone = barrier.clone();
        let t = thread::spawn(move || {
            loop {
                barrier_clone.wait();
                let Some(element) = mutex_clone.lock().unwrap().pop() else {
                    break;
                };
                println!("Processing {element} by {:?}", thread::current().id());
            }
        });
        workers.push(t);
    }
    workers.into_iter().for_each(|t| t.join().unwrap());
}
```

Запустив этот код, мы можем убедиться, что наш массив данных (вектор из 100 элементов) обрабатывается порцями по 10 элементов, и каждый элемент обрабатывается отдельным потоком.

## scoped thread

Одним из неприятных ограничений потоков является то, что даже если поток запускается в теле функции, и прекращает свою работу до того как эта функция завершилась (потому что на потоке вызывается `join`), поток всё равно не может обращаться к локальным переменным функции по ссылке, без перемещения владениями над ними. Например, как в примере выше для барьера.

Фактически, этот сценарий когда время жизни потоков гаратировано ограничено скоупом функции, в которой эти потоки создаются. Специально для таких ситуаций существую [scoped thread](https://doc.rust-lang.org/std/thread/fn.scope.html) (потоки принадлежащие скоупу).

Scoped потоки:

* погут обращаться к локальным переменным родительской функции непосредственно по ссылке, без `move` и `Arc`
* гаратировано завершаются в конце блока _scope_, внутри которого они созданы (как если перед выходом из скоупа для них вызывается `join`)

Давайте перепишем наш пример для барьера, с использованием scoped потоков.

```rust
use std::sync::{Barrier, Mutex};
use std::thread;

const WORKERS_NUM: usize = 10;

fn main() {
    let data = (0..100).collect::<Vec<_>>();
    let mutex = Mutex::new(data);
    let barrier = Barrier::new(WORKERS_NUM);

    thread::scope(|s| {
        // скоуп для запуска потоков
        for _ in 0..WORKERS_NUM {
            s.spawn(|| {
                loop {
                    barrier.wait();
                    let Some(element) = mutex.lock().unwrap().pop() else {
                        break;
                    };
                    println!("Processing {element} by {:?}", thread::current().id());
                }
            });
        }
    });
}
```

## Атомики

Для примитивных типов данных, в Rust есть атомарные обёртки, которые можно безопасно использовать в многопоточной среде.

<table><thead><tr><th width="246.66668701171875" valign="top">Примитивный тип</th><th valign="top">Атомик</th></tr></thead><tbody><tr><td valign="top"><code>bool</code></td><td valign="top"><code>std::sync::atomic::AtomicBool</code></td></tr><tr><td valign="top"><code>u8</code><br><code>u16</code><br><code>u32</code><br><code>u64</code><br><code>i8</code><br><code>i16</code><br><code>i32</code><br><code>i64</code></td><td valign="top"><code>std::sync::atomic::AtomicU8</code><br><code>std::sync::atomic::AtomicU16</code><br><code>std::sync::atomic::AtomicU32</code><br><code>std::sync::atomic::AtomicU64</code><br><code>std::sync::atomic::AtomicI8</code><br><code>std::sync::atomic::AtomicI16</code><br><code>std::sync::atomic::AtomicI32</code><br><code>std::sync::atomic::AtomicI64</code></td></tr><tr><td valign="top"><code>usize</code><br><code>isize</code></td><td valign="top"><code>std::sync::atomic::AtomicUsize</code><br><code>std::sync::atomic::AtomicIsize</code></td></tr><tr><td valign="top"><code>*mut T</code></td><td valign="top"><code>std::sync::atomic::AtomicPtr&#x3C;T></code></td></tr></tbody></table>

Все числовые атомик типы позволяют атомарно читать и записывать значение, а так же атомарно выполнять арифметические и логические операции.

Например, сделаем на базе [AtomicI32](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicI32.html) счётчик, который будем инкрементировать из разных потоков.

```rust
use std::{
    sync::atomic::{AtomicI32, Ordering},
    thread,
};

fn main() {
    let a = AtomicI32::new(0); // инициализируем атомик нулём
    thread::scope(|s| {
        for _ in 0..1000 {
            s.spawn(|| {
                for _ in 0..1000 {
                    a.fetch_add(1, Ordering::Relaxed); // инкрементируем
                }
            });
        }
    });
    println!("{}", a.load(Ordering::Relaxed)); // 1000000
}
```

Совершив 1000 инкрементов из 1000 потоков, мы ожидаемо получим значение счётчика равно 100000.

<details>

<summary>Тот же счётчик с указателем вместо атомика</summary>

```rust
use std::thread;

fn main() {
    let mut a = 0;
    let address = (&raw mut a).addr();
    thread::scope(|s| {
        for _ in 0..1000 {
            s.spawn(|| {
                for _ in 0..1000 {
                    unsafe {
                        *(address as *mut i32) += 1;
                    }
                }
            });
        }
    });
    println!("{}", a); // 862886
}
```

</details>

Метод [fetch\_add](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicI32.html#method.fetch_add), реализующий сложение, кроме слагаемого так же принимает некий [Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) в качестве аргумента. Этот аргумент отвечает за, так называемый, memory ordering, который определяет какие гарантии мы получаем касательно порядка синхронизации операций с атомиками между потоками.

Дело в том, что компилятор может переставлять местами инструкции таким образом, что их порядок не будет соответствовать тому порядку, который они имеют в исходном коде. Это делается с целью оптимизации вычислений, и не влияет на логику программы. Если компилятор видит, что значение череменной `y` вычисляется на основе значения переменной `x`, то он проследит, чтобы значение `x` было вычислено до того как начнётся вычисление `y`. Однако компилятор может отслеживать такие зависимости только рамках вычислений для одного потока, но когда дело касается атомиков, то компилятор не в стостоянии отследить в каком порядке происходит взаимодействием между разными атомиками в разных разных потоков. Именно для решения этой проблемы и задаётся параметр `Ordering`, который может принимать такие значения:

* `Relaxed` — всё еще гарантирует консистентность одной атомик переменной, но ничего не обещает про относительный порядок операций между разными атомик переменными. Это значит, что два потока могут по разному видеть операции на двух разных атомик переменных. Например, если первый поток записывает значение в атомик `a`, и затем в атомик `b`, то второй поток может увидеть эти записи в противоположном порядке.

{% hint style="info" %}
Скорее всего, в бекенд приложениях, вы будете использовать атомики только в качестве счётчиков для некой статистики, и всегда только с `Relaxed` ордерингом.
{% endhint %}

* `Release` — используется для операций записи. При записи атомика с `Release` ордерингом, все предшествующие операции (включая не атомики и `Relaxed` атомик записи), должны быть видимы другим потокам, которые читают этот атомик с `Acquire` ордерингом.\
  Например, если мы записываем переменную `a`, а потом записываем атомик `b` с ордерингом `Release`, то другой поток, который читает атомик `b` с `Acquire` ордерингом, должен обязательно увидеть и изменения для `a`, предшествующие записи в атомик `b`.
* `Acquire` — используется для операций чтения из атомика, запись в который была сделана с ордерингом `Release`.
* `AcqRel` — используется для операций, которые имеют одновременно семантику чтения и записи, например `fetch_add` (сначала читает значение, потом  вычисляет сложение, а потом делает запись при помощи [compare and swap](https://ru.wikipedia.org/wiki/%D0%A1%D1%80%D0%B0%D0%B2%D0%BD%D0%B5%D0%BD%D0%B8%D0%B5_%D1%81_%D0%BE%D0%B1%D0%BC%D0%B5%D0%BD%D0%BE%D0%BC)). При выполнении `fetch_add` с ордерингом `AcqRel`, чтение значения произойдёт с `Acquire` ордерингом, а запись с `Release`.
* `SeqCst` — используется и для чтения, и для записи. Это наивысшая гарантия синхронизации изменения происходящих перед записью атомика. Используйте этот вид ордеринга, когда у вас логика завязана на последовательность изменений атомиков. Однако имейте ввиду, что этот вид ордеринга имеет наихудшую производительность.

***

Кроме `fetch_add`, числовые атомики так же поддерживают: вычитание (`fetch_sub`), побитовое ИЛИ (`fetch_or`), побитовое И (`fetch_and`), побитовое И НЕ (`fetch_nand`) и побитовое исключающее ИЛИ (`fetch_xor`).

***

Так же, атомики поддерживают [Compare And Exchange](https://ru.wikipedia.org/wiki/%D0%A1%D1%80%D0%B0%D0%B2%D0%BD%D0%B5%D0%BD%D0%B8%D0%B5_%D1%81_%D0%BE%D0%B1%D0%BC%D0%B5%D0%BD%D0%BE%D0%BC) операцию, которая используется в случаях, когда нужно считать значение атомика, далее вычислить новое значение, и после этого записать новое значение в атомик. Просто комбинация методов `load` и `save` в таком сценарии не подойдёт, так как между вызовами `load` и `save` (во время вычисления нового значения), другой поток может произвести запись в этот же атомик, и получится, что текущий поток вычисляет новое значение основываясь на уже устаревших данных.

Операция compare and exchange (по факту, это то же, что и compare and swap) представлена методом:

```rust
pub fn compare_exchange(
    &self,
    expected: тип_значения_атомика,
    new: тип_значения_атомика,
    success_ordering: Ordering,
    failure_ordering: Ordering,
) -> Result<тип, тип>
```

Этот метод принимает ожидаемое текущее значение атомика — `expected` и новое значение — `new`, и записывает в атомик новое значение только в том случае, если ожидаемое текущее значение равно реальному текущему значению, содержащемуся в атомике.

Метод возвращает `Ok(реальное текущее значение до записи)`, если запись была произведена, или `Err(реальное текущее значение)`, если запись не была произведена.

Парамтры memory ordering-а:

* `success_ordering` указывает memory ordering для успешной read-write-modify операции, которая происходит, если ожидаимое текущее значение атомика совпадёт с реальным текущим значение.
* `failure_ordering` указывает memory ordering для load операции, которой будет считано реальное текущее значение атомика, если оно не совпадёт с ожидаемым.

Пример использования `compare_exchange`:

```rust
use std::{
    sync::atomic::{AtomicI32, Ordering},
    thread,
};

fn main() {
    let a = AtomicI32::new(0);
    thread::scope(|s| {
        for _ in 0..1000 {
            s.spawn(|| {
                for _ in 0..1000 {
                    let mut old_val = a.load(Ordering::Relaxed);
                    loop {
                        let new_val = old_val + 1;
                        let r = a.compare_exchange(
                            old_val,
                            new_val,
                            Ordering::Relaxed,
                            Ordering::Relaxed,
                        );
                        if let Err(actual_val) = r {
                            old_val = actual_val;
                        } else {
                            break;
                        }
                    }
                }
            });
        }
    });
    println!("{:?}", a); // 1000000
}
```

Для эксперимента можете попробовать заменить внутренний цикл на

```rust
for _ in 0..1000 {
    let old_val = a.load(Ordering::Relaxed);
    let new_val = old_val + 1;
    a.store(new_val, Ordering::Relaxed);
}
```

и посмотреть на результат.

## Thread local storage

Thread-local storage — механизм, представляющий из себя локальное для потока хранилище.

Идея заключается в том, что в коде мы работает thread-local переменной так, словно она глобальная. Однако для каждого потока эта "глобальная" переменная имеет своё значение.

Thread-local переменная объявляется при помощи макроса [thread\_local](https://doc.rust-lang.org/std/macro.thread_local.html):

```rust
thread_local! {
    static ПЕРЕМЕННАЯ: Тип = значение;
}
```

После этого можно работать с thread-local переменной так, словно это это обычная глобальная переменная. При этом, изменения переменной будут видны только в рамках того же потока.

Рассмотрим простой пример, в котором хорошо видно, что значение thread-local переменной у каждого потока своё:

```rust
use std::{cell::Cell, thread};

thread_local! {
    pub static NUM: Cell<u32> = const { Cell::new(0) };
}

fn print_num() {
    println!("{}", NUM.get());
}

fn main() {
    let t1 = thread::spawn(|| {
        NUM.set(1);
        print_num();
    });
    let t2 = thread::spawn(|| {
        NUM.set(2);
        print_num();
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

Программа напечатает:

```
1
2
```

Thread-local переменные часто используют в веб серверах. Например, в самом начале обработки запроса, мы помещаем в thread-local информацию о сессии пользователя сделавшего запроса. Дальше эта информация становится доступной во всей цепочке вызовов обработки запроса. Без thread-local нам бы пришлось пробрасывать объект сессии "сквозь" все вызовы функций.

## Каналы

Для общения между потоками, стандратная библиотека Rust предоставляет каналы.

По сути канал является синхронизированной очередью: один поток добавляет элементы в конец очереди, а другой поток извлекает элементы из значала.

Стандартная библиотека предоставляет канал [**mpsc**](https://doc.rust-lang.org/std/sync/mpsc/index.html) (multiple producers, single consumer), который позволяет множеству потоков добавлять сообщения в канал, но только одному потоку считывать сообщения.

{% hint style="info" %}
Если необходимо, чтобы множество потоков могли читать из канала, то библиотека [crossbeam-channel](https://crates.io/crates/crossbeam-channel) предоставляет mpmc (multiple producers, multiple consumers) канал.

Так же в стандартной библиотеке Rust есть свой канал [**mpmc**](https://doc.rust-lang.org/std/sync/mpmc/index.html), однако на данный момет (Rust 1.89) он доступен только в ночной сборке Rust.
{% endhint %}

Канал создаётся одной из двух функций:

* [channel](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html) — создаёт канал неограниченного размера
* [sync\_channel](https://doc.rust-lang.org/std/sync/mpsc/fn.sync_channel.html) — создаёт канал заданного размера. Если канал заполнен, то попытка добавить сообщение в канал приводит к блокировке пишущего потока до тех пор, пока читающий поток не извлечет из канала сообщение, тем самым освободив место.

И `channel` и `sync_channel` возвращают кортеж из двух элементов: [Sender](https://doc.rust-lang.org/std/sync/mpsc/struct.Sender.html) и [Receiver](https://doc.rust-lang.org/std/sync/mpsc/struct.Receiver.html).

```rust
pub fn channel<T>() -> (Sender<T>, Receiver<T>)
pub fn sync_channel<T>(bound: usize) -> (SyncSender<T>, Receiver<T>)
```

Объект `Sender` используется для добавления сообщений в канал, а `Receiver` — для чтения из канала.

Рассмотрим простой пример: один поток отправляет числа в канал, а другой поток их оттуда достаёт и печатает на консоль.

```rust
use std::{sync::mpsc, thread};

// Обёртка для чисел, которые будем передавать в канал
enum Element {
    Num(i32), // очередное число
    Finish,   // флаг завершения работы
}

fn main() {
    let (producer, receiver) = mpsc::channel::<Element>();
    let t1 = thread::spawn(move || {
        for i in 0..5 {
            let _ = producer.send(Element::Num(i));
        }
    });
    let t2 = thread::spawn(move || {
        while let Ok(msg) = receiver.recv() {
            match msg {
                Element::Num(i) => println!("{i}"),
                Element::Finish => break,
            }
        }
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

В примере выше мы использовали канал неограниченного размера. Теперь давайте посмотрим как работает канал ограниченного размера.

Создадим канал вместитьностью в 3 сообщения. Один поток будет отправлять сообщения в канал, и печатать сколько времени заняла отправка, а другой поток будет в цикле ждать 1 секунду, а потом извлекать очередное сообщение.

{% code lineNumbers="true" %}
```rust
use std::{sync::mpsc, thread, time::{Duration, Instant}};

fn main() {
    let (snd, rcv) = mpsc::sync_channel::<i32>(3);
    let t1 = thread::spawn(move || {
        for i in 0..5 {
            let start = Instant::now();
            let _ = snd.send(i);
            println!("Took {} millis to send msg", start.elapsed().as_millis());
        }
    });
    let _ = thread::spawn(move || {
        loop {
            thread::sleep(Duration::from_secs(1));
            match rcv.recv() {
                Ok(_) => (),
                Err(_) => break,
            }
        }
    });
    let _ = t1.join();
}
```
{% endcode %}

Программа печатает:

```
Took 0 millis  to send msg
Took 0 millis  to send msg
Took 0 millis  to send msg
Took 1000 millis  to send msg
Took 1000 millis  to send msg
```

Так как вместительность канала — 3 сообщения, то первые 3 сообщения отправляются в канал мгновенно. Далее поток отправитель засыпает и ожидает, когда в канале появится место. А так как поток читатель делает паузу в 1 секунду между чтениями сообщения, то канал отправитель начинает ждать по 1 секунде, пока сообщение будет отправлено в канал.

## Что почитать

В этой главе мы сделали беглый обзор многопоточности в Rust. Скорее всего, для написания бекенд приложений этих знаний будет достаточно, однако если вы хотите углубиться в это тему, то обязательно обратите внимание на бесплатную книгу Rust Atomics and Locks за авторством Mara Bos.

[https://marabos.nl/atomics/](https://marabos.nl/atomics/)
