# Мидлваре

Мидлваре (middleware) — функциональность, которая вызывается перед функцией-обработчиком, а так же имеет возможность обработать результат функции обработчика, перед тем как отправить его в сеть. Можно сказать, что мидлваре оборачивает вызов функции-обработчика запроса.

<img src="../.gitbook/assets/file.excalidraw (1) (1).svg" alt="" class="gitbook-drawing">

Если мы хотим выполнять какие-то действия перед обработкой любого запроса, то мидлваре — как раз тот механизм, который позволит нам это сделать.

При этом мы можем создать несколько мидлваре, которые будут цепочкой выполняться перед обработчиком запроса, и после него.

<img src="../.gitbook/assets/file.excalidraw (1) (1) (1).svg" alt="" class="gitbook-drawing">

По диаграмме сверху видно, что мидлвари выстраиваются подобно слоям, оборачивающим обработчик запроса. Именно поэтому, зарегистрированное в роутере мидлваре называют слоем (layer).

## Простейшее милдваре

Самый простой вариант мидлваре выглядит как просто функция, принимающая объект [Request](https://docs.rs/axum/latest/axum/extract/type.Request.html) — HTTP запрос, и объект [Next](https://docs.rs/axum/latest/axum/middleware/struct.Next.html) — ссылка не следующий в цепочке мидлваре, или обработчик запроса.

```rust
async fn мидлваре(request: Request, next: Next) -> Response {
    // ... действия перед обработкой запроса
    
    // вызываем следующий в цепочке обработчик
    let response = next.run(request).await;
    
    // ... действия после обработки запроса

    // возвращаем результат из мидлваре
    response
}
```

Такую функцию надо зарегистрировать в роутере, при помощи метода `layer`, после чего мидлваре начнёт отрабатывать для всех эндпоинтов, зарегистрированных в этом роутере.

```rust
let app = Router::new()
    .route("/путь", get(функция-обработчик))
    .layer(middleware::from_fn(милдваре));
```

Чтобы понять как это работает, напишем классический пример мидлваре, который логирует время работы обработчика:

```rust
use axum::{Router, extract::Request, response::Response, routing::get};
use axum::middleware::{self, Next};
use tokio::time::Instant;

async fn log_exec_time(request: Request, next: Next) -> Response {
    let start = Instant::now();
    let response = next.run(request).await;
    tracing::info!("Request took: {} micros", start.elapsed().as_micros());
    response
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let app = Router::new()
        .route("/hello", get(hello))
        .layer(middleware::from_fn(log_exec_time));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

Запустим сервер и перейдём в броузере на [http://localhost:8080/hello](http://localhost:8080/hello)

```
$ cargo run
   Compiling test_axum v0.1.0 (/home/stas/dev/proj/rust/test_axum)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.60s
     Running `target/debug/test_axum`
2025-12-02T23:09:30.342625Z  INFO test_axum: Request took: 52 micros
```

Как видите, в консоле появилась лог-запись с уровнем `INFO` от приложениея test\_axum.

## Слои

Теперь давайте посмотрим на практике, как взаимодействуют между собой разные мидлваре.

Добавим в наш hello сервер два мидлваре, каждый из которых печатает в лог одно сообщение перед вызовом следюущего обработчика по цепочке, и другое сообщение после того, как следующий по цепочке обработчик закончил свою работу:

```rust
use axum::middleware::{Next, from_fn};
use axum::{Router, extract::Request, routing::get};

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let app = Router::new()
        .route("/hello", get(hello))
        .layer(from_fn(async |request: Request, next: Next| {
            tracing::info!("Middleware-1: before call");
            let response = next.run(request).await;
            tracing::info!("Middleware-1: after call");
            response
        }))
        .layer(from_fn(async |request: Request, next: Next| {
            tracing::info!("Middleware-2: before call");
            let response = next.run(request).await;
            tracing::info!("Middleware-2: after call");
            response
        }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    tracing::info!("Request handler");
    "Hello!"
}
```

Запустим сервер и перейдём по адресу [http://localhost:8080/hello](http://localhost:8080/hello)

```
stas@slim:~/dev/proj/rust/test_axum$ cargo run
   Compiling test_axum v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.16s
     Running `target/debug/test_axum`
2025-12-03T22:36:42.563720Z  INFO test_axum: Middleware-2: before call
2025-12-03T22:36:42.563776Z  INFO test_axum: Middleware-1: before call
2025-12-03T22:36:42.563800Z  INFO test_axum: Request handler
2025-12-03T22:36:42.563823Z  INFO test_axum: Middleware-1: after call
2025-12-03T22:36:42.563834Z  INFO test_axum: Middleware-2: after call
```

Как видите, мидлваре оборачивают друг друга подобно матрёшке. При этом, тот мидлваре, который был добавлен в роутер последним, становится первым в цепочке.

## Работа со стэйтом

Нередко для логики работы мидлваре может понадобится доступ к данным, хранящимся в состоянии. В этом случае функция-мидлваре будет иметь дополнительный аргумент — `State`.

```rust
async fn мидлваре(state: State<Стэйт>, request: Request, next: Next) -> Response {
    // Действия перед обработкой запроса
    let response = next.run(request).await;
    // действия после обработки запроса
    response
}
```

Регистрируется такая мидлваре-функция в роутере при помощи [middleware::from\_fn\_with\_state](https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html):

```rust
let app = Router::new()
    .route("/путь", get(функция-обработчик))
    .with_state(state.clone())
    .layer(middleware::from_fn_with_state(state, милдваре));
```

Как вы могли заметить, состояние приходится передавать отдельно и в `with_state`, и в `from_fn_with_state`. С одной стороны это не очень удобно, но с другой стороны позволяет иметь разные объекты состояния для мидлваре, и для роутера.

В качестве примера, напишем мидлваре, который извлекает из HTTP заголовка ID сессии, потом достаёт из состояния соответствующий объект сессии пользователя, и помещает его в [task-local переменную](../async/tokio.md#task-local). Далее эта task-local переменная используется из функции-обработчика запроса.

```rust
use std::{collections::HashMap, sync::Arc};
use axum::{Router, extract::{Request, State}, http::StatusCode, routing::get};
use axum::middleware::{self, Next};
use axum::response::{IntoResponse, Response};
use tokio::sync::{Mutex, RwLock};

tokio::task_local! {
    pub static SESSION: Arc<Mutex<SessionData>>;
}

// Сессия пользователям
struct SessionData {
    user_name: String,
}

// Состояние бекенда, хранящее сессии
struct AppState {
    sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>>,
}

async fn set_session_for_request(
    State(state): State<Arc<AppState>>, request: Request, next: Next
) -> Response {
    // получаем объект сессии
    let session = if let Some(value) = request.headers().get("sessionid") {
        if let Ok(string_value) = value.to_str() {
            let session_id = string_value.to_string();
            let read_guard = state.sessions.read().await;
            if let Some(session) = read_guard.get(&session_id) {
                session.clone()
            } else {
                return StatusCode::UNAUTHORIZED.into_response();
            }
        } else {
            return StatusCode::BAD_REQUEST.into_response();
        }
    } else {
        return StatusCode::UNAUTHORIZED.into_response();
    };

    // Присваиваем объект сессии task-local переменной
    let response = SESSION.scope(session, async {
            next.run(request).await // вызываем обработчик по цепочке
        }).await;
    response
}

#[tokio::main]
async fn main() {
    let sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>> = {
        let mut data = HashMap::new();
        data.insert( // Создаём тестовую сессию пользователя
            "1111-1111-1111".to_string(),
            Arc::new(Mutex::new(SessionData {
                user_name: "John Doe".to_string(),
            })),
        );
        RwLock::new(data)
    };
    let app_state = Arc::new(AppState { sessions });

    let app = Router::new()
        .route("/hello", get(hello))
        .with_state(app_state.clone())
        .layer(middleware::from_fn_with_state(app_state, set_session_for_request));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> String {
    // Извлекаем сессию из task-local переменной
    let session = SESSION.with(|session| session.clone());
    // используем сессию
    format!("Hello, {}!", session.lock().await.user_name)
}
```

Простестируем, чтобы убедиться, что мидлваре корректно помещает сессию в task-local переменную, доступную из функции-обработчика.

```
$ curl -i -H "sessionid: 1111-1111-1111" http://localhost:8080/hello                                                                             HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 15
date: Wed, 03 Dec 2025 14:03:22 GMT

Hello, John Doe!
```

## Стандартные мидлваре

Экосистема Axum предоставляет ряд стандартных милдваре для таких вещей как: CORS, компрессия, таймауты и т.д., но перед тем как мы посмотрим как ими пользоваться, мы должны познакомится с библиотекой [Tower](tower.md).
