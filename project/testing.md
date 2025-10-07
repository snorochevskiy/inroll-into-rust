# Тестирование

## Unit тестирование

В Rust принято писать юнит-тесты в том же файле, в котором находятся тестируемые функции.

Юнит-тест — представляет из себя функцию помеченную аннотацией `#[test]`.

Например, у нас есть некий модуль `src/my_module.rs`, который содержит функцию `inc`, и два юнит-теста к ней: `test_inc_1` и `test_inc_2`.

```rust
fn inc(a: i32) -> i32 {
    a + 1
}

#[test]
fn test_inc_1() {
    assert_eq!(inc(1), 2);
}

#[test]
fn test_inc_2() {
    assert_eq!(inc(7), 8);
}
```

Макрос `assert_eq` сравнивает аргументы, и если они не равны, то инициируется паника.

***

Для того, чтобы запустить все тесты в пакете используется команда `cargo test`:

```
$ cargo test
running 2 tests
test my_module::test_inc_1 ... ok
test my_module::test_inc_2 ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Запустить отдельный тест можно так:

```
$ cargo test -- --exact my_module::test_inc_1
```

С точки зрения организации, все юнит тесты компилируются в отдельный исполняемый бинарный файл.

## Интеграционные тесты

Если юнит-тесты располагаются в тех же модулях, что и тестируемые функции, то интеграционные тесты располагаются в отдельной директории пакета — `tests`. Как следствие, интеграционные тесты имеют доступ только к публичному API тестируемого крэйта.

<pre><code>├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── module1.rs
│   └── module2.rs
<strong>└── tests/
</strong><strong>    ├── integration-tests_1.rs
</strong><strong>    └── integration-tests_2.rs
</strong></code></pre>

В отличии от юнит тестов, каждый `tests/*.rs` файл компилируется в отдельный исполняемый файл. Фактически \*.rs файл с интеграционными тестами является отдельным тестовым крэйтом.&#x20;

Поэтому имеет смысл, по возможности, группировать интеграционные тесты по типу ресурсов, которые им необходимы для исполнения. Например, собрать в один файл все тесты, которым нужна только реляционная БД, в другой файл — все тесты которым нужен только распределённый кеш.

{% hint style="info" %}
Мы пока что не разобрались как создавать статические разделяемые ресурсы. Речь об этом пойдёт в главе TODO.
{% endhint %}

***

Давайте посмотрем на пример, который поможет понять структуру тестов. Объектом нашего интеграционного тестирования будет крэйт библиотека с двумя модулями.

```
cargo new test_project --lib
```

{% code title="src/lib.rs" %}
```rust
pub mod data;
pub mod user_service;
```
{% endcode %}

{% code title="src/data.rs" %}
```rust
// Отражает пользователя, который хранится в неком хранилище.
pub struct User {
    pub first_name: String,
    pub last_name: String,
}

// Объявляет интерфейс получения объекта пользователя из хранилища.
// Конкретная реализация работы с конкретным хранилищем отдаётся на откуп
// пользователю библиотеки, который должен будет реализовать этот трэйт.
pub trait DataSource {
    // Извлекает пользователя с указанным ID из хранилища
    fn find_user_by_id(&self, id: u64) -> Option<User>;
}
```
{% endcode %}

{% code title="src/user_service.rs" %}
```rust
use crate::data::DataSource;

// Функция, которая принимает реализацию работы с хранилищем и ID пользователя,
// и возвращает полное имя этого пользователя
pub fn get_user_full_name(ds: &dyn DataSource, user_id: u64) -> Option<String> {
    ds.find_user_by_id(user_id)
      .map(|user| format!("{} {}", user.first_name, user.last_name))
}
```
{% endcode %}

Теперь напишем интеграционный тест, в котором мы протестируем функцию `get_user_full_name` из модуля `user_service`.

Эта функция требует какую-то реализацию трэйта `DataSource`, чтобы использовать её для извлечения пользователя. В реальной программе, реализацией, скорее всего, была бы интеграция с базой данных, или неким identity сервисом, но для тестовых целей мы просто сделаем заглушку (mock): реализацию, которая возвращает заранее подготовленные тестовые данные.

{% code title="tests/user_service_test.rs" %}
```rust
use test_project::{data::{DataSource, User}, user_service::get_user_full_name};

// Заглушка для тестов
struct DataSourceMock;
impl DataSource for DataSourceMock {
    fn find_user_by_id(&self, id: u64) -> Option<User> {
        Some(User { first_name: "John".to_string(), last_name: "Doe".to_string() })
    }
}

#[test]
fn test_get_user_full_name() {
    let result = get_user_full_name(&DataSourceMock, 1);
    assert_eq!(Some("John Doe".to_string()), result);
}
```
{% endcode %}

Теперь запустим наш тест:

```
$ cargo test
     Running tests/user_service_test.rs (target/debug/deps/user_service_test-9f98ad2948b56b39)

running 1 test
test test_get_user_full_name ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Тест проходит.

***

Если нужны какие-то зависимости только для интеграционных тестов (например, dev containers), то в `Cargo.toml` существует отдельная секция зависимостей, которые доступны только для интеграционных тестов — секция `[dev-dependencies]`.

На данном этапе нам нет смысла углубляться в интеграционные тесты. Мы погорим о них подбробнее, когда будем рассматривать написание бекендов.
