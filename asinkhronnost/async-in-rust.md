# Асинхронность в Rust

Когда дело касается реализации потоков пользовательского пространства, необходимо решить две задачи:

* Как реализовать файберы так, чтобы с ними было удобно работать?
* Как спроектировать экзекьютор для файберов так, чтобы он был и гибким, и производительным?

Rust предлагает довольно элегантное решение: язык и стандартная библиотека задают максимально гибкий интерфейс для асинхронных функция (которые по сути являются файберами), а реализация экзекьюторов для их исполнения отдаётся на откуп сторонним разработчикам. Таким образом получается, что мы имеем единый интерфейс для написания файберов, при этом можем вибирать тот или другой рантайм, в зависимости от специфики нашего приложения.

{% hint style="info" %}
В документации, названия "экзекьюто" (executor) и "синхронный рантайм" (async runtime) часто используются взаимозаменяемо.
{% endhint %}

## Асинхронные функции

Объявляя функцию с ключевым словом `async`, мы указываем, что эта функция является асинхронной, и должна быть исполнена конкурентно на неком рантайме.

```rust
async fn имя_функци(аргументы) -> ТипРезультата {
    ...
}
```

Например:

```rust
async fn get_1() -> i32 {
    1
}
```

Вызов асинхронной функции возвращает не её значение, а объект файбера. Например, вызов вышеобъявленной async функции `get_1()` создаёт объект файбера, который при вычислении возвращает `1`.

```rust
fn main() {
    let my_fiber = get_1();
}
```

Далее, чтобы получить результат этого файбера, нам надо исполнить файбер на неком рантайме.

{% hint style="info" %}
Читая документацию по async функциям или асинхронным рантайм, вы врядли встретите слово "файбер" (fiber). Скорее всего вы будете встречать названия "async function" или "future". Первое используется по понятной причине: из-за ключевого слова `async`. Второе же используется из-за трэйта, который лежит в основе асинхронного исполнения в Rust, но об этом позже.\
В любом случае, и "async function" и "future" в данном контексте являются синонимами файберов.
{% endhint %}

## Исполнение async функции

Стандартная библиотека Rust предоставляет функциональность для создания файберов, однако не содержит никакой реализации экзекьютора для их исполнения.

Экосистема Rust предлагает несколько высокопроизводительных асинхронных экзекьюторов, но на данный момент мы воспользуемся самым примитивным экзекьютором из библиотеки [futures](https://crates.io/crates/futures).

{% tabs %}
{% tab title="src/main.rs" %}
```rust
use futures::executor::block_on;

async fn get_1() -> i32 {
  1
}

fn main() {
  let my_fiber = get_1();
  let result = block_on(my_fiber);
  println!("{}", result);
}
```
{% endtab %}

{% tab title="Cargo.toml" %}
```toml
[package]
name = "test_rust_async"
version = "0.1.0"
edition = "2024"

[dependencies]
futures = "0.3"
```
{% endtab %}
{% endtabs %}

Функция `block_on` принимает файбер в качестве аргумента, и исполняет его на экзекьюторе. Этот экзекьютор просто использует текущий поток для того чтобы исполнить файбер: он не имеет умного планировщика и сложной системы пулов потоков. При этом этои простой экзекьютор является отличной демонстрацией того, что экзекьюторы могут быть устроены очень по-разному.

Мы рассмотрим более сложные экзекьюторы далее.

## Композиция асинхронных функций

Мы уже знаем, что вызов асинхронной функции порождает файбер. Но что делать если файбер должен состоять из последовательности вызовов async функций?

Например, у нас есть две async функции:

```rust
async fn load_number() -> i32 {
    1
}

async fn transform_number(a: i32) -> i32 {
    a + 1
}
```

и мы хотим получить файбер, который сначала получает число вызовом `load_number`, а потом преобразует его при помощи `transform_number`.

Для композиции вызовов асинхронных функций, в Rust используется подход **async/await**, с которым вы уже могли встречаться в языках Javascript, C#, Python, Swift.

```rust
async fn my_flow() -> i32 {
    let number = load_number().await;
    let transformed = transform_number(number).await;
    transformed
}
```

Здесь мы создаём третью async функцию, которая объединяет в себе вызовы других async функций. Теперь вызов функции `my_flow` вернёт экземпляр файбера, который в качестве своих составляющих содержит под-файберы, порождённые вызовами `load_number` и `transform_number`.

Важно отметить, что вызов `await` можно совершать **только** в теле async функции.

{% hint style="info" %}
Если вы встречали оператор await в других языках программирования, то могли заметить, что там await ставится перед вызовом функции, в то время как в Rust — после. Так было сделано для того, чтобы было удобнее увязывать вызовы async функций в цепочки:

```rust
get_value().await
    .call_method_1().await
    .call_method_2().await
```
{% endhint %}

А теперь самое главное: помните, мы говорили, что файбер содержит в себе места, в которых экзекьютор может приостановить выполнение файбера? Вызовы `.await` как раз являются теми самыми местами. Именно на них экзекьютор может отправлять файбер в очередь ожидания, перекидывать на другой ОС поток и т.д.

Как работает `await` внутри станет понятнее, когда мы разберёмся с внутренним устройством файбера, а пока что давайте взглянем на еще один пример, который приближен к реалиям написания бекендов.

Допустим, у нас есть два отдельных сервиса: один отвечает за хранение пользователей, а другой за хранение адресов. Мы ходим написать фукциональность, которая возвращает информацию о пользователе и его адресе.

```rust
// Информация о пользователей
struct User     { user_id: u64, name: String,   addr_id: u64 }
// Информация об адресе
struct Address  { addr_id: u64, location: String }
// Информация о пользователей и адресе.
struct UserInfo { user: User,   addr: Address }

// Сервис, который возвращает пользователя по его ID
async fn get_user_by_id(user_id: u64) -> User {
    // делает запрос с user сервису
}

// Сервис, который возвращает адрес по ID записи адреса
async fn get_address_by_id(addr_id: u64) -> Address {
    // делает запрос с address сервису
}

async fn get_user_with_address(user_id: u64) -> UserInfo {
  let user: User    = get_user_by_id(user_id).await;
  let addr: Address = get_address_by_id(user.addr_id).await;
  UserInfo { user, addr }
}
```

<details>

<summary>Java программистам на заметку</summary>

Чтобы понять насколько красива композиция через async/await, давайте рассмотрим как выглядела бы подобная композиция функций основанных на [CompletableFuture](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html):

```java
CompletableFuture<User> fetchUserById(Long userId) { ... }

CompletableFuture<Address> fetchAddressById(Long addrId) { … }

CompletableFuture<UserInfo> getUserInfo(Long userId) {
  return fetchUserById(userId)
    .thenCompose(user ->
       fetchAddressById(user.getAddrId())
         .thenApply(addr ->
           new UserInfo(user, addr)
         )
    );
}
```



</details>

## async замыкания

Фьючер можно создавать не только вызовом async функции, но и при помощи async замыкания.

```rust
let closure = async || { тело };
let fiber = closure();
```

Например:

```rust
fn main() {
    let closure = async|| { 1 };
    let my_fiber = closure();
    let result = futures::executor::block_on(my_fiber);
    println!("{}", result); // 1
}
```

Также существуют async блоки. Они подобны обычному скоупу (блок кода в фигурных скобках), только возвращают не значение, а фьючер.

```rust
let fiber = async { тело };
```

Например:

```rust
fn main() {
    let my_fiber = async { 1 };
    let result = futures::executor::block_on(my_fiber);
    println!("{}", result); // 1
}
```

## Анатомия файберов — Future

Давайте теперь разберёмся как файберы устроены внутри.

Если в вашем редакторе кода имеется поддержка Rust LSP, благодаря чему отображаются выведенные типы, то скорее всего этот пример из секции [#ispolnenie-async-funkcii](async-in-rust.md#ispolnenie-async-funkcii "mention"), у вас выглядит так:

```rust
use futures::executor::block_on;

async fn get_1() -> i32 {
    1
}

fn main() {
  let my_fiber: impl Future<Output = i32> = get_1();
  let result: i32 = block_on(my_fiber);
  println!("{}", result);
}
```

Как видите, тип файбера — это нечто, реализующее трэйт [Future](https://doc.rust-lang.org/std/future/trait.Future.html). А трэйт `Future`, в свою очередь, является основным интерфейсом для файберов.

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

Да, все файберы в Rust должны реализовать именно этот генерик трэйт, который содержит всего один метод. Давайте разбираться с ним.

С ассциированным типом `Output` всё просто: это тип результата работы всего фьючера (файбера).

Метод `poll` используется экзекьютором, для того, чтобы получить значение файбера. Метод возвращает обёртку-перечисление [Poll](https://doc.rust-lang.org/std/task/enum.Poll.html):

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

`Ready(результата фьючера)` символизирует, что фьючер закончил свою работу, а `Pending` указывает на то, что фьючер еще не "в процессе".

Аргумент типа [Context](https://doc.rust-lang.org/std/task/struct.Context.html) метода `poll` используется для того, чтобы передать во фьючер ссылку на объект типа [Waker](https://doc.rust-lang.org/std/task/struct.Waker.html). А `Waker`, в свою очередь, используется фьючером, чтобы сигнализировать свой экзекьютор о том, что он (фьючер) завершил работу.

Вот как это работает:

Когда экзекьютор запрашивает значение фьючера, он передаёт в `poll` (через `Context`) ссылку на `Waker`. Если значение фьючера уже готово (или может быть быстро посчитано прямо в рамках вызова `poll`), то значение сразу возвращается завёрнутым в `Poll::Ready`. Если же значение не может быть вычислено сразу (т.е. `poll` вернёт `Poll::Pending`), то фьючер сохраняет ссылку на `Waker`, и когда фьючер завершит свою работу, использует вызов `Waker::wake()`, чтобы сигнализировать экзекьютор о том, что можно повторно вызвать `poll` и получить результат.

Предполагется, что на новом объекте фьючера экзекьютор вызывает `poll` только один раз, и если фьючер вернул `Poll::Pending`, то фьючер помещается в очередь для "еще исполняющихся", и не трогается пока не поступит соответствующее оповещение от этого фьючера через `Waker`.

Визуализируем примерную схему взаимодействия экзекьютора и фьючеров. Здесь _Future 1_ соответствует простой async функции, которая не зависит от других async функции, и не выполняет операции ввода/вывода. Future 2 выполняет операцию неблокирующего ввода/вывода с помощью отдельного потока осуществляющего ввод/вывод через epoll.

<img src="../.gitbook/assets/file.excalidraw (21).svg" alt="" class="gitbook-drawing">

Такая реализация ввода/вывода не является каким-то требованием или стандартной функциональность. Мы просто продемострировали один из возможных вариантов реализации.

{% hint style="info" %}
Так же, как вы могли заметить, в методе `poll`, тип аргумента `self` — `Pin<&mut Self>`. Если не вдаваться в детали, то [Pin](https://doc.rust-lang.org/std/pin/struct.Pin.html) — это обёртка, которая "прикалывает" (как булавка) память объекта фьючера к одному месту, и не позволяет её перемещать.

`Pin` практически не используется нигде кроме фьючеров, и врядли вы столкнётесь с ним напряму в рамках бекенд приложений, поэтому мы не будет заострять на нём внимание.
{% endhint %}

## Пишем свой экзекютор

К этому моменту вас уже должно было сложиться примерное представление как в Rust работают фьючеры, и по каким принципам построены экзекьюторы файберов.

Однако если вам не хватает ощущения как это всё работает внутри, то ниже мы рассмотрим пример максимально примитивного экзекьютора. Но если вы просто перейдёте к следующей главе, то ничего важного вы не пропустите.

<pre class="language-rust" data-title="src/my_executor.rs"><code class="lang-rust">use std::{
    collections::HashSet, future::Future, pin::Pin,
    sync::{Arc, Mutex, atomic::{AtomicBool, AtomicU64, Ordering}, mpsc},
    task::{Context, Poll, Wake},
    thread::{self, sleep},
    time::Duration
};

type BoxFuture = Pin&#x3C;Box&#x3C;dyn Future&#x3C;Output = ()> + Send + 'static>>;

// Обёртка для фьючера, сгенерированного комипилятором из async функции
struct SpawnedTask {
    id: u64,
    future: Mutex&#x3C;Option&#x3C;BoxFuture>>,
}

// Реализация фьючера, которая делает паузу (как функция thread::sleep)
pub struct Sleep {
    interval: Duration,
    is_ready: Arc&#x3C;AtomicBool>,
}

impl Sleep {
    pub fn new(interval: Duration) -> Sleep {
        Sleep {
            interval, is_ready: Arc::new(AtomicBool::new(false))
        }
    }
}

impl Future for Sleep {
    type Output = ();

    fn poll(self: Pin&#x3C;&#x26;mut Self>, cx: &#x26;mut Context&#x3C;'_>) -> Poll&#x3C;Self::Output> {
        if self.is_ready.load(Ordering::SeqCst) {
            return Poll::Ready(());
        } else {
            let waker = cx.waker().clone();
            let ready_flag = self.is_ready.clone();
            let interval_to_sleep = self.interval.clone();
            // самая примитивная реализация - стартовать новый поток для ожидания
            thread::spawn(move || {
                sleep(interval_to_sleep);
                ready_flag.store(true, Ordering::SeqCst);
                // извещаем экзекьютор об окончании работы фьючера
                waker.wake();
            });
            Poll::Pending
        }
    }
}

// Интерфейс для работы с экзекьютором
pub struct Executor {
    runtime: ExecutorRuntime,
    last_task_id: AtomicU64,
}

impl Executor {
    pub fn new() -> Executor {
        Executor {
            runtime: ExecutorRuntime::new(),
            last_task_id: AtomicU64::new(1),
        }
    }
    // Используется для добавления async функции в очередь экзекьютора
    pub fn spawn&#x3C;F>(&#x26;self, fut: F) where F: Future&#x3C;Output = ()> + Send + 'static {
        let task = Arc::new(SpawnedTask {
            id: self.last_task_id.fetch_add(1, Ordering::SeqCst),
            future: Mutex::new(Some(Box::pin(fut))),
        });
        let _ = self.runtime.task_producer.send(task);
    }

    // Запускает вычисление фьючеров (файберов) из очереди экзекьютора
    pub fn exec_blocking(&#x26;mut self) {
        self.runtime.run();
    }
}

// Инкапсулирует код для непосредственного вычисление фьючеров
pub struct ExecutorRuntime {
    // Sender, который выдаётся другим компонентам (Executor и Taks)
    // чтобы они могли добавления async функции в очередь
    task_producer: mpsc::Sender&#x3C;Arc&#x3C;SpawnedTask>>,
    // Receiver используемый рантаймом для извлечения следующей async-функции
    task_queue: mpsc::Receiver&#x3C;Arc&#x3C;SpawnedTask>>,
    // Хранилище для фьючеров, которые при первом вызове poll вернули
    // Poll::Pending. Нужно, чтобы не завершить работу до того как выполнены
    // все фьючеры.
    task_pending: HashSet&#x3C;u64>,
}

impl ExecutorRuntime {
    pub fn new() -> ExecutorRuntime {
        let (sender, receiver) = mpsc::channel::&#x3C;Arc&#x3C;SpawnedTask>>();
        ExecutorRuntime {
            task_producer: sender,
            task_queue: receiver,
            task_pending: HashSet::new(),
        }
    }

    // Запуск исполнения фьючеров
    pub fn run(&#x26;mut self) {
        loop {
            match self.task_queue.recv_timeout(Duration::from_secs(1)) {
                Ok(task) => self.process_task(task),
                Err(_) =>
                    // Если очередь фьючеров пуста, и нет фьючеров, которые
                    // в процессе исполнения, тогда обработка завершается
                    if self.task_pending.is_empty() {
                        break;
                    },
            }
        }
    }

    fn process_task(&#x26;mut self, task: Arc&#x3C;SpawnedTask>) {
        let mut future_guard = task.future.lock().unwrap();
        // Извлекаем объект Pin&#x3C;Box&#x3C;dyn Future>> из таска, потому что для
        // вызова poll нужен именно объект (по значению), а не ссылка
        let Some(mut fut) = future_guard.take() else {
            return; // already finished
        };

        // Создаём Waker на случай, если фьючер не сможет выполниться сразу
        // и вернёт Poll::Pending.
        let spawned_task_waker = SpawnedTaskWaker {
            task: task.clone(),
            sender: self.task_producer.clone(),
        };
        let waker = Arc::new(spawned_task_waker).into();
        let mut cx = Context::from_waker(&#x26;waker);

        // Выполняем фьючер
        let poll_result = fut.as_mut().poll(&#x26;mut cx);

        match poll_result {
            Poll::Pending => {
                // Засовываем фьючер обратно в таск, так как этот таск придётся
                // обрабатывать снова, после того как фьючер вызове waker
                *future_guard = Some(fut);
                // Запоминаем, что таск с таким ID выполняется на фоне
                self.task_pending.insert(task.id);
            }
            Poll::Ready(()) => {
                // Удаляем (если надо) ID таска из списка тасков,
                // выполняющихся на фоне
                self.task_pending.remove(&#x26;task.id);
            }
        }
    }
}

// Простейший Waker, который просто еще раз добавляет таск в очередь рантайма
<strong>struct SpawnedTaskWaker {
</strong>    sender: mpsc::Sender&#x3C;Arc&#x3C;SpawnedTask>>,
    task: Arc&#x3C;SpawnedTask>,
}

impl Wake for SpawnedTaskWaker {
    fn wake(self: Arc&#x3C;Self>) {
        let _ = self.sender.send(self.task.clone());
    }
}
</code></pre>

Пример использования экзекьютора:

{% code title="main.rs" %}
```rust
mod my_executor;

use std::time::Duration;
use crate::my_executor::{Executor, Sleep};

async fn func_with_sleep() {
    println!("Async function: before sleep");
    Sleep::new(Duration::from_secs(1)).await;
    println!("Async function: after sleep");
}

async fn calc_5() -> i32 {
    5
}

fn main() {
    let mut ex = Executor::new();

    // Получаем фьючер (файбер)
    let fut = func_with_sleep();
    // Закидываем фьючер в очередь экзекьютора
    ex.spawn(fut);

    // Вместо async функции используем async замыкание
    ex.spawn(async {
        println!("Async closure: start");
        async fn inner() {
            println!("Async closure: inner function");
        }
        inner().await;

        let a = calc_5().await;
        println!("Async closure: async func call result {a}");
        println!("Async closure: end");
    });

    ex.exec_blocking();

    println!("All done");
}
```
{% endcode %}

Эта программа напечатает:

```
Async function: before sleep
Async closure: start
Async closure: inner function
Async closure: async func call result 5
Async closure: end
Async function: after sleep
All done
```

Если вам интересно увидеть более универсальный, но простой экзекьютор, то можете изучить исходный код экзекьютора из библотека futures:\
[https://github.com/rust-lang/futures-rs/tree/master/futures-executor](https://github.com/rust-lang/futures-rs/tree/master/futures-executor)
