# Tokio

Теперь когда мы познакомились с фьючерами и экзекьюторами, мы можем порассуждать о том, как должен быть устроен эффективный экзекьютор для бекенд приложений. Бекенд приложения являются приложениями с интенсивным вводом/выводом (особоенно по сетевой части), поэтому эффектиный экзекьютор должен делать упор на работу с неблокирующим вводом/выводом.

Экзекьютор должен:

1. ... иметь пул потоков для выполнения файберов, которые занимаются только вычислением. Этот пул не должен быть больше, чем количество ядер процессора.
2. ... предоставлять свой неблокирующий API, как минимум, для операций работы с сетью
3. ... также иметь пул с одним потоком с наивысшим приоритетом, который будет обрабатывать неблокирующие операции ввода/вывода по средствам epoll, io\_uring, IOCP, kqueue.
4. ... для операций ввода/вывода, которые не имеют неблокирующего варианта (например, DNS вызовы), иметь пул потоков неограниченного размера.

Итак получилось, что мы примерно описали структуру экзекьютора из библиотеки [Tokio](https://tokio.rs/).

**Tokio** — это высокопроизводительный асинхронный рантайм для приложений с интенсивным вводом/выводом. Он содержит как экзекьютор с несколькими пулами потоков (в том числе, и для неблоклокирующих операции ввода/вывода), так и большую библиотеку функций для работы с сестью, файловой системой, таймерами и т.д. в асинхронном стиле.

{% hint style="info" %}
У библиотеки Tokio есть очень детальная документация, которая настоятельно рекомендуется к прочтению: [https://tokio.rs/tokio/tutorial](https://tokio.rs/tokio/tutorial)
{% endhint %}

## Tokio API для ввода/вывода

Первое что хочется заметить при изучении Tokio: авторы сделали API для асинхронной работы с сетью и файловой системой очень похожим на аналогичный синхронный API из стандартной библиотеки.

Возьмём для примера простую программу, которая считывает содержимое текстового файла в строку.

Сначала напишем реализацию с использованием синхронного API из стандартной библиотеки.

```rust
use std::fs::File;
use std::io::Read;

fn main() {
    let mut file = File::open("/etc/fstab")
        .unwrap();

    let mut contents = String::new();
    file.read_to_string(&mut contents)
        .unwrap();

    println!("{}", contents);
}
```

А теперь то же самое, только с использованием асинхронного API из Tokio.

Теперь сама программа (`src/main.rs`):

```rust
use tokio::fs::File;
use tokio::io::AsyncReadExt;

#[tokio::main]
async fn main() {
    let mut file = File::open("/etc/fstab")
        .await
        .unwrap();

    let mut contents = String::new();
    file.read_to_string(&mut contents)
        .await
        .unwrap();

    println!("{}", contents);
}
```

Нам так же потребуется подключить Tokio в `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = {version = "1", features = ["full"]}
```

Как видите, код работы с файлом абсолютно идентичен, за исключением вызовов `.await`.

## tokio::main

Теперь, когда мы увидели как выглядит программа с использованием Tokio, давайте разбираться подробнее.

Первое на чём следует остановиться — функция `main`. Как вы могли заметить, она стала async, и над ней появилась аннотация `#[tokio::main]`. Здесь нет никакой магии: главная функция в любой программе на Rust — это та функция `main` с которой мы уже давно знакомы, не асинхронная.

Просто в библиотеке Tokio определён процедурный макрос, который превращает такой код:

```rust
#[tokio::main]
async fn main() {
    // Код асинхронной программы
}
```

в такой:

```rust
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // Код асинхронной программы
    })
}
```

Если вернуться к прошлой главе, и посмотреть как мы [исполняли асинхронную функцию на экзекьюторе](async-in-rust.md#ispolnenie-async-funkcii), то мы увидим, что там тоже присутствовал вызов `block_on`, который имел тот же смысл.

То есть экзекьютор Tokio работает точно так же, как и любой другой: компилятор превращает async функцию во фьючер, а этот фьючер далее передаётся экзекьютору для исполнения.

## Task

Подобно тому, как со традиционной многопоточностью можно создавать ОС потоки, Tokio позволяет вручную создавать файберы, которые в терминологии Tokio называются тасками (task - задача).

Создание таска осуществляется при помощи функции [tokio::spawn](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html), которая очень похожа на функцию [thread::spawn](https://doc.rust-lang.org/std/thread/fn.spawn.html), используемую для создания ОС потоков.

```rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output> ⓘwhere
    F: Future + Send + 'static,
    F::Output: Send + 'static,
```

Например:

```rust
use tokio::task::{JoinError, JoinHandle};

async fn get_1() -> i32 {
    1
}

#[tokio::main]
async fn main() {
    let t1: JoinHandle<i32> = tokio::spawn(get_1());
    let result1: Result<i32, JoinError> = t1.await;
    println!("{result1:?}"); // Ok(1)

    let t2: JoinHandle<i32> = tokio::spawn(async { 5 });
    let result2: Result<i32, JoinError> = t2.await;
    println!("{result2:?}"); // Ok(5)
}
```

Как видите, с точки зрения API, работа с тасками очень напоминает работу с ОС потоками. Разница лишь в том, что ОС потоки обслуживаются операционной системой, а таски — рантаймом Tokio.

Когда мы вызываем `tokio::spawn`, создаётся новый таск (являющийся обёрткой для фьючера сгенерированного компилятором из async функции или async блока), который сразу же помещается в очередь экзекьютора на выполнение. В какой-то момент планировщик Tokio поместит этот таск на один из ОС поток, и попытается его исполнить путём вызова метода `poll` у фьючера.

Вызов `tokio::spawn` возвращает объект [tokio::task::JoinHandle](https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html), который очень похож на [std::thread::JoinHandle](https://doc.rust-lang.org/std/thread/struct.JoinHandle.html), возвращаемый вызовом `thread::spawn`. Он так же позволяет дождаться выполнения таска, и получить его результат.

## task vs thread

Мы много говорили о том, что потоки пользовательского пространства и неблокирующий API работают эффективнее, чем ОС потоки и блокируюий API. Теперь, когда мы наконец добрались до Tokio, давайте проведём небольшое сравнение.

Напишем программу, которая создаём миллион тасков, каждый из которых просто засыпает на 1 секунд, а потом завершает свою работу.

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let mut handles = Vec::new();
    for _ in 1..1000000 {
        let t = tokio::spawn(async {
            tokio::time::sleep(Duration::from_secs(1)).await;
        });
        handles.push(t);
    }
    for handle in handles {
        let _ = handle.await;
    }
}
```

Запустим программу через Linux утилиту time (не путать со встроенной shell командой), чтобы посмотреть сколько ресурсов потребляет такая программа.

```
$ /bin/time cargo run
10.28user 4.07system 0:02.83elapsed 506%CPU (0avgtext+0avgdata 386308maxresident)k
0inputs+256outputs (0major+103536minor)pagefaults 0swaps
```

На ноутбуке автора (с 8-ядерномым процессором Ryzen 7840HS и 32 гигабайтами оперативной памяти) программа выполнилась за 2.83 секунды и использовала 386308 килобайт оперативной памяти в пике.

Теперь напишем такую же программу, только вместо Tokio тасков мы будем использовать обычные ОС потоки.

```rust
use std::time::Duration;
use std::thread;

fn main() {
    let mut handles = Vec::new();
    for _ in 1..1000000 {
        let t = thread::spawn(|| {
            thread::sleep(Duration::from_secs(1));
        });
        handles.push(t);
    }
    for handle in handles {
        let _ = handle.join().unwrap();
    }
}
```

Этот вариант отработал за 1 минуту 16 секунд, и в пике использовал 4.2 гигабайта оперативной памяти.

```
$ /bin/time cargo run
thread '<unnamed>' (583640) panicked at library/std/src/sys/pal/unix/stack_overflow.rs:222:13:
failed to set up alternative stack guard page: Cannot allocate memory (os error 12)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

thread 'main' (91293) panicked at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/thread/mod.rs:729:29:
failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }
fatal runtime error: failed to initiate panic, error 5, aborting

Command terminated by signal 6
4.25user 71.05system 1:16.74elapsed 98%CPU (0avgtext+0avgdata 4219816maxresident)k
0inputs+0outputs (0major+1062256minor)pagefaults 0swaps
```

Так же в логе можно заметить, что несколько попыток создать поток потерпели неудачу.

{% hint style="info" %}
Возможно при запуске, вместо того чтобы работать долго, программа сразу завершится с подобной ошибкой:

```
$ cargo run
thread 'main' (52703) panicked at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/thread/mod.rs:729:29:
failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Command exited with non-zero status 101
```

Если вы используете Gnome Terminal, то скорее всего так и произойдёт (для удачного запуска автор использовал эмулятор терминала Alacritty).

Эта ошибка указывает, что программа попыталась превысить максимальный лимит количства потоков.
{% endhint %}

Сравнив результаты работы этих примеров, можно с уверенностью сказать, что для некоторых задач Tokio гораздо эффективнее, чем ОС потоки.

## Синхронизация тасков

Tokio пытается копировать из стандартной библиотеки не только API для работы с файловой системой и сетью, но и API для механизмов синхронизации. Именно поэтому для синхронизации работы с данными из разных тасков, Tokio предлагает типы Mutex, RwLock и Barrier, которые очень похожи на своих собратьев работающих с ОС потоками.

Например, давайте посмотрим на Tokio мьютекс:

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let m: Arc<Mutex<i32>> = Arc::new(Mutex::new(1));

    let t = tokio::spawn({
        let m = m.clone();
        async move {
            let mut guard = m.lock().await;
            *guard = 2;
        }
    });
    let _ = t.await;

    println!("{m:?}");
}
```

Как видите, этот пример очень похож на пример мьютекса из главы про [многопоточность](../dive-deeper-into-rust/multithreading.md#mutex), с той разницей, что вызов `lock()` возвращает фьючер, на котором нужно вызвать `.await`.

{% hint style="warning" %}
Будте осторожны: если вы случайно используете мьютекс, предназначенный для работы с ОС потоками, в Tokio таске, то получите серьезное падение производительности.
{% endhint %}

## Каналы

Tokio предоставляет не только свои варианты механизмов синхронизации, но и свои каналы.

#### mpsc

[mpsc](https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html) — Tokio аналог для multiple producers single consumer [канала](../dive-deeper-into-rust/multithreading.md#kanaly) из стандартной библиотеки.

Рассмотрим пример:

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    // Создаём канал
    let (snd, mut rcv) = mpsc::unbounded_channel::<i32>();

    // Из нового таска отправляем число  в канала
    tokio::spawn({
        let snd = snd.clone();
        async move {
            let _ = snd.send(1);
        }
    });
    tokio::spawn({
        let snd = snd.clone();
        async move {
            let _ = snd.send(2);
        }
    });

    // В цикле считываем из канала числа, и печатаем их на консоль.
    // Если в течении секунды не пришло новых сообщений, то завершаем цикл.
    while let Some(msg) = tokio::select! {
        msg = rcv.recv()                    => msg,
        _   = sleep(Duration::from_secs(1)) => None,
    } {
        println!("Received: {msg}");
    }
}
```

Здесь мы сначала создаём канал без ограничения на размер — `mpsc::unbounded_channel`.

Далее мы стартуем два таска, которые отправляют в канал по одному сообщению.

В конце у нас идёт интересный блок, аналогов которому в стандартной библиотеке нет. Мы используем макрос [tokio::select](https://tokio.rs/tokio/tutorial/select), который принимает в себя несколько фьючеров (в нашем случае два), и возвращает результат того, который завершится первым.

Синтаксис использования этого макроса в какой-то степени похож на оператор `match`:

```rust
let переменная = tokio::select! {
    шаблон_1 = фьючер_1 => выражение_1,
    шаблон_2 = фьючер_2 => выражение_2,
};
```

`select!` принимает несколько фьючеров, и ожидает завершения одного из них. Резуль первого завершившегося фьючера присваивается шаблону, а дальше выполняется выражение связанное с этим фьючером. Результат этого выражения — результат всего `select!` блока. При этом результаты остальных фьючеров просто отбрасываются.

Так как функция `tokio::time:sleep` возвращает фьючер, её часто используют в связке с макросом `select!` в качестве таймаута для какой-то другой операции. Таким образом, в примере выше в блоке `select!` мы читаем сообщения из канала с таймаутом в 1 секунду.

#### oneshot

[oneshot](https://docs.rs/tokio/latest/tokio/sync/oneshot/index.html) — канал, в который можно послать только одно сообщение.

Реализация очень элегантная:&#x20;

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (snd, rcv) = oneshot::channel::<i32>(); // создаём канал
    
    // Метод send забирает владение над self, тем самым уничтожая его.
    // Поэтому send можно вызвать только однажды.
    // Так же, ресивер для oneshot не реализует Clone.
    let _ = snd.send(1);

    // Так как oneshot расчитан на приём только одного сообщения,
    // вместо метода receive() используется просто .await
    let r = rcv.await;

    println!("{r:?}"); // Ok(1)
}
```

#### broadcast

[broadcast](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html) — multi-producer, multi-consumer канал, в котором каждое сообщение от любого из отправителей, отправляется всем потребителям.

```rust
use std::{thread::sleep, time::Duration};
use tokio::sync::broadcast::{self, Receiver};

fn spawn_receiver(name: &'static str, rcv: &Receiver<i32>) {
    let mut rcv = rcv.resubscribe();
    tokio::spawn({
        async move {
            println!("{name} > msg: {:?}", rcv.recv().await);
        }
    });
}

#[tokio::main]
async fn main() {
    let (snd, rcv) = broadcast::channel::<i32>(100);

    spawn_receiver("rcv-1", &rcv);
    spawn_receiver("rcv-2", &rcv);
    spawn_receiver("rcv-3", &rcv);

    let _ = snd.send(5);

    sleep(Duration::from_secs(1));
}
```

Программа напечатает:

```
rcv-2 > msg: Ok(5)
rcv-1 > msg: Ok(5)
rcv-3 > msg: Ok(5)
```

Как видите, каждый из слушателей получил свою копию сообщения.

#### watch

Канал [watch](https://docs.rs/tokio/latest/tokio/sync/watch/index.html) позволяет отправлять сообщения из разных тасков, а так же из разных тасков сообщения читать. При этом для читателя, канал содержит только одно последнее записанное значение.

Этот тип канала обычно используется для информирования слушателей об изменениях.

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (snd, rcv) = watch::channel::<i32>(0);

    // has_changed() показывает приходили ли сообщения не просмотренные ЭТИМ ресивером
    println!("Has new messages: {}", rcv.has_changed().unwrap());

    let _ = snd.send(1);
    let _ = snd.send(2);
    let _ = snd.send(3);

    {
        let mut rcv1 = rcv.clone();
        println!("Receiver 1 changed: {}", rcv1.has_changed().unwrap());
        {
            // borrow_and_update() возвращает ссылку на последнее полученное значение
            // и помечает это значение как просмотренное на этом ресивере
            let guard = rcv1.borrow_and_update();
            println!("Receiver 1 value: {:?}", *guard);
        }
        println!("Receiver 1 changed: {}", rcv1.has_changed().unwrap());
    }

    {
        let mut rcv2 = rcv.clone();
        println!("Receiver 2 changed: {}", rcv2.has_changed().unwrap());
        {
            let guard = rcv2.borrow_and_update();
            println!("Receiver 2 value: {:?}", *guard);
        }
        println!("Receiver 2 changed: {}", rcv2.has_changed().unwrap());
    }
}
```

Программа напечатает:

```
Has new messages: false
Receiver 1 changed: true
Receiver 1 value: 3
Receiver 1 changed: false
Receiver 2 changed: true
Receiver 2 value: 3
Receiver 2 changed: false
```

#### async\_channel

Иногда в бекенд приложениях нужно создать функциональность, где обработчики запросов генерируют заявки на какую-то ресурсоёмкую работу (например, обработку изображения), а далее эти заявки обрабатываются пулом обработчиков.

<img src="../.gitbook/assets/file.excalidraw (23) (1).svg" alt="" class="gitbook-drawing">

Казалось бы: это отличный сценарий использования для каналов. Однако, проблема в том, что в Tokio нет типа канала, который позволяет множеству тасков отправлять сообщения, и множеству тасков вынимать сообщения. То есть, некого аналога [MPMC](https://doc.rust-lang.org/std/sync/mpmc/index.html) в Tokio нет.

К счастью, существует сторонняя библиотека [async\_channel](https://crates.io/crates/async-channel), которая работает вместе с Tokio и предоставляет такой вид канала.

```rust
use async_channel::{Receiver, Sender};
use std::time::Duration;
use tokio::{task::JoinHandle, time::sleep};

// Создаёт таск из которого отсылает заданное значение в канал
fn make_producer(snd: &Sender<i32>, value: i32) {
    let snd = snd.clone();
    tokio::spawn(async move {
        let _ = snd.send(value).await;
    });
}

// Создаёт таск - worker, который в цикле считывает сообщения из канала и печатает их.
// Если в течении секунды не поступает новых сообщений, то worker завершается.
fn make_worker(rcv: &Receiver<i32>, name: &'static str) -> JoinHandle<()> {
    let rcv = rcv.clone();
    tokio::spawn(async move {
        loop {
            tokio::select! {
                msg = rcv.recv() => {
                    println!("{name} > received: {:?}", msg.unwrap());
                    sleep(Duration::from_millis(100)).await;
                }
                _ = sleep(Duration::from_secs(1)) => break,
            }
        }
    })
}

#[tokio::main]
async fn main() {
    let (snd, rcv) = async_channel::unbounded::<i32>();
    make_producer(&snd, 1);
    make_producer(&snd, 2);
    make_producer(&snd, 3);
    make_producer(&snd, 4);

    let t1 = make_worker(&rcv, "worker-1");
    let t2 = make_worker(&rcv, "worker-2");
    let _ = t1.await;
    let _ = t2.await;
}
```

Программа напечатает:

```
worker-2 > received: 1
worker-1 > received: 2
worker-1 > received: 4
worker-2 > received: 3
```

## LocalSet

Экзекьютор Tokio работает таким образом, что один и тот же таск может быть частично выполнен на одном ОС потоке, потом сотановлен и помещён в очередь ожидания ответа от операции ввода/вывода, а после возобновлён уже на другом ОС потоке.

Именно поэтому данные, захватываемые таском должны реализовать `Send`: ведь таск всегда может быть переброшен на другой поток для выполнения.

Однако в Tokio есть механизм, который позволяет указать, что таск должен выполняться на одном и том же ОС потоке, и не перебрасываться на другие. Этот механизм называется [LocalSet](https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html).

```rust
let local_set = LocalSet::new();
local_set.spawn(async { тело таска });
```

Рассмотрим пример, в котором мы инкрементируем счётчик из несколько тасков.

При работе с обычныйми тасками, мы должны были бы завернуть счётчик в `Mutex` (реализующий `Sync`, но не `Send`), а мьютекс в свою очередь завернуть в `Arc`, который реализует `Send`. Однако `LocalSet` позволяет использовать типы не реализующие `Send`, а значит вместо `Arc` можем использовать просто `Rc`.

```rust
use std::{rc::Rc, time::Duration};

use tokio::{sync::Mutex, task::LocalSet, time::sleep};

#[tokio::main]
async fn main() {
    let counter = Rc::new(Mutex::new(0)); // Rc не реализует Send
    let local_set = LocalSet::new();
    for _ in 0..100 {
        let counter = counter.clone();
        local_set.spawn_local(async move { // порождаем таск на текущем ОС потоке
            sleep(Duration::from_secs(1)).await;
            *counter.lock().await += 1;
        });
    }
    local_set.await;
    println!("{}", *counter.lock().await);
}
```

Чтобы убедиться, что таски работают параллельно, запустим программу с помощью утилиты time:

```
$ /bin/time cargo run
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.04s
     Running `target/debug/test_rust`
100
0.09user 0.05system 0:01.14elapsed 13%CPU (0avgtext+0avgdata 30524maxresident)k
0inputs+0outputs (0major+8035minor)pagefaults 0swaps
```

Как мы видим, время выполнения сотни тасков составило 1.14 секунды.

## Task local

Имея дело с ОС потоками, мы можем использовать хранилище привязанное к потоку — [Thread local](../dive-deeper-into-rust/multithreading.md#thread-local-storage). Для Tokio тасков существует аналогичный механизм Task Local, который позволяет создать "глобальную" переменную, которая для каждого таска будет своя.

Переменные, локальные для потока, объявляются при помощи макроса [task\_local](https://docs.rs/tokio/latest/tokio/macro.task_local.html).

```rust
tokio::task_local! {
    pub static ПЕРЕМЕННАЯ: Тип;
}
```

Имея подобное объявоение, мы можем создать скоуп с локальной переменной.

```rust
ПЕРЕМЕННАЯ.scope(значение, async {
   // Код в этом скоупе, а так же все функции, которые вызваны из скоупа,
   // имеют доступ к task local переменной так, словно это глобальная переменная
   // типа LocalKey<Тип>
   my_func()
});

fn my_func() {
    let v = ПЕРЕМЕННАЯ.get();
    // ...
}
```

Все функции вызванные внутри этого скоупа, будут иметь доступ с этой переменной так, словно она является глобальной. Тип этой глобальной переменной — [LocalKey\<T>](https://docs.rs/tokio/latest/tokio/task/struct.LocalKey.html), где `T` — тип, с которым мы объявляли переменную в макросе `task_local!`.

Если мы объявили нашу task local переменную с типом, который реализует `Copy` (например `i32`), то на объекте `LocalKey` мы можем использовать метод `get()`, который просто извлекает копию значения.

Например:

```rust
tokio::task_local! {
    pub static NUM: i32;
}

fn print_num() {
    println!("NUM = {}", NUM.get());
}

#[tokio::main]
async fn main() {
    NUM.scope(1, async { print_num() }).await;
    NUM.scope(2, async { print_num() }).await;
}
```

Программа напечатает:

```
NUM = 1
NUM = 2
```

Если же нам надо получить ссылку на значение хранимое в task local переменной, то нам поможет метод [with](https://docs.rs/tokio/latest/tokio/task/struct.LocalKey.html#method.with):

```rust
pub fn with<F, R>(&'static self, f: F) -> R where F: FnOnce(&T) -> R
```

Метод принимает замыкание, которое ожидает ссылку на task local переменную в качестве аргумента.

```rust
tokio::task_local! {
    pub static NAME: String;
}

fn print_name() {
    NAME.with(|name| println!("NAME = {name}"))
}

#[tokio::main]
async fn main() {
    NAME.scope("A".to_string(), async { print_name() }).await;

    NAME.scope("B".to_string(), async { print_name() }).await;
}
```

***

Какое же применение может быть у task local переменных?

Говоря о бекенд приложениях, часто используют такое понятие как "сквозная проблема" (cross-cutting concern). Это такой вид функциональности, который насквозь пронизывает логику программы, и поэтому не может быть изолирован в отдельный модуль. Классическими примерами сквозной проблемы служат аутентификация/авторизация и логирование.

Например, в приложении может быть много различной функциональности, которая должна выполняться только в том случае, если её выполнение было инициировано запросом от пользователя, который авторизован для выполнения этой функциональности. Вопрос: как нам передавать информацию о пользователе и его правах?

Первый способ: во всех функциях, которые участвуют в обработке запроса, ввести аргумент, который будет хранить информацию о пользователе. Такой аргумент придётся добавить во все функции: как в те, которые проверяют информацию о пользователе непосредственно, так и в те, которые просто вызывают другие функции, которым эта информация может потребоваться. Согласитесь, не самое изящное решение.

Второй способ: в самом начале обработки запроса, положить в task local информацию о пользователе, а дальше, во всех местах где она потребуется, считывать из task local.

Давайте напишем простоей пример, который демонстрирует это подход.

```rust
use std::cell::RefCell;

tokio::task_local! {
    pub static USER: RefCell<UserInfo>;
}

struct UserInfo {
    id: String,           // ID пользователя, которое должно браться из запроса
    role: Option<String>, // Роль определяется по ID на сервере
}

async fn handle_request(request: &str) -> String {
    USER.with(|u| println!("Request from: {}", u.borrow().id));
    fill_role();
    process_client(request).await
}

fn fill_role() {
    USER.with(|u| {
        // эмулируем поиск пользователя в БД по ID и заполняем его роль
        let mut user = u.borrow_mut();
        if user.id.as_str() == "ID_1234" {
            user.role = Some("CLIENT".to_string());
        }
    });
}

async fn process_client(request: &str) -> String {
    // функциональность доступная только пользователям с ролью CLIENT
    if USER.with(|u| u.borrow().role != Some("CLIENT".to_string())) {
        format!("UNAUTHORIZED TO PROCESS: {request}")
    } else {
        format!("PROCESSED: {request}")
    }
}

#[tokio::main]
async fn main() {
    // Эмулируем обработку запроса от пользователя
    let response_1 = USER.scope(
        RefCell::new(UserInfo { id: "ID_1234".to_string(), role: None }),
        async move { handle_request("Test 1").await },
    )
    .await;
    println!("Response: {response_1}");

    // Эмулируем обработку запроса от другого пользователя
    let response_2 = USER.scope(
        RefCell::new(UserInfo { id: "ID_5678".to_string(), role: None }),
        async move { handle_request("Test 2").await },
    )
    .await;
    println!("Response: {response_2}");
}

```

Программа выводит:

```
Request from: ID_1234
Response: PROCESSED: Test 1
Request from: ID_5678
Response: UNAUTHORIZED TO PROCESS: Test 2
```
