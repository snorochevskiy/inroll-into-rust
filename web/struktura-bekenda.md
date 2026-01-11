---
hidden: true
---

# Структура бекенда

К этому моменту мы уже умеем создавать HTTP сервер, работать с базой данных, знаем как тестировать эндопоинты и функциональность работающую с БД. Давайте теперь рассмотрим как всё это обычно компонуют в реальных проекта.

## Создание проекта

Мы создадим многомодульный (workspace) проект, состоящий из таких крэйтов:

* persist — слой работы с базой данных
* server — слой веб сервера

1\) В удобном для вас месте создайте новую директорию "test\_backend".

2\) В директории test\_backend создайте файл `Cargo.toml` со следующим содержимым:

```toml
[workspace]
resolver = "3"
```

3\) Откройте консоль в директории test\_backend и создайте модули persist и server:

```
cargo new persist --lib
cargo new server
```

После этого корневой `Cargo.toml` должен иметь содержимое:

```toml
[workspace]
resolver = "3"
members = ["persist", "server"]
```

4\) Объявите зависимости

```toml
[workspace]
resolver = "3"
members = ["adder"]

[workspace.dependencies]
async-trait = "0.1"
thiserror = "1"
tokio = { version = "1", features = ["full"] }
sqlx = {version = "0.8", features = ["postgres", "chrono", "runtime-tokio", "bigdecimal"]}
chrono = "0.4"
bigdecimal = { version = "0.4", features = ["serde"]}

axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Итоговая структура проекта будет такой:

```
test_backend/
├── Cargo.toml
├── persist
│    ├── Cargo.toml
│    ├── src/
│    │   └── lib.rs
│    └── tests/
│        └── account_storage_tests.rs
├── server
│    ├── Cargo.toml
│    └── src/
│        ├── service.rs
│        ├── endpoints.rs
│        ├── lib.rs    
│        └── main.rs
└── migrations/
    ├── 0001_accounts.down.sql
    ├── 0001_accounts.up.sql
    ├── 0002_transactions.down.sql
    └── 0002_transactions.up.sql
```



## Крэйт persist

В `persis/Cargo.toml`:

```toml
[package]
name = "persist"
version = "0.1.0"
edition = "2024"

[dependencies]
async-trait = { workspace = true }
sqlx = { workspace = true }
chrono = { workspace = true }
bigdecimal = { workspace = true }
serde = { workspace = true }
```

Весь код крэйта будет в файле `persist/src/lib.rs`:

```rust
use bigdecimal::BigDecimal;
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool, postgres::PgPoolOptions};

#[derive(Debug, Serialize, Deserialize, FromRow)]
pub struct Account {
    pub id: i64,
    pub owner_name: String,
    pub balance: BigDecimal,
}

// Скроем реализацию хранилища за этим трэйтом
#[async_trait::async_trait]
pub trait Storage {
    async fn fetch_accounts(&self) -> Result<Vec<Account>, sqlx::Error>;
    async fn create_accounts(
        &self, owner_name: &str, initial_balance: BigDecimal
    ) -> Result<Account, sqlx::Error>;
}

pub struct StorageImpl {
    db: PgPool,
}

impl StorageImpl {
    pub fn new(db: PgPool) -> StorageImpl {
        StorageImpl { db }
    }
    pub async fn from_connection_url(url: &str) -> Result<StorageImpl, sqlx::Error> {
        let pool = PgPoolOptions::new()
            .connect(url).await?;
        Ok(StorageImpl { db: pool })
    }
}

#[async_trait::async_trait]
impl Storage for StorageImpl {
    async fn fetch_accounts(&self) -> Result<Vec<Account>, sqlx::Error> {
        sqlx::query_as("SELECT id, owner_name, balance FROM accounts")
            .fetch_all(&self.db)
            .await
    }

    async fn create_accounts(
        &self, owner_name: &str, initial_balance: BigDecimal
    ) -> Result<Account, sqlx::Error> {
        let mut tx = self.db.begin().await?;
        sqlx::query("INSERT INTO accounts(owner_name, balance) VALUES($1, $2)")
            .bind(owner_name)
            .bind(initial_balance)
            .execute(&mut *tx)
            .await?;
        let result = sqlx::query_as(r#"
                SELECT id, owner_name, balance
                FROM accounts
                WHERE id = currval('accounts_seq')
            "#)
            .fetch_one(&mut *tx)
            .await?;
        tx.commit().await?;
        Ok(result)
    }
}
```

Интеграционный текст для хранилища — `persist/tests/account_storage_tests.rs`:

```rust
use bigdecimal::{BigDecimal, FromPrimitive};
use persist::{Storage, StorageImpl};
use sqlx::{migrate::Migrator, postgres::{PgConnectOptions, PgPoolOptions}};
use testcontainers::runners::AsyncRunner;

#[tokio::test]
async fn test_create_and_fetch_account() {
let container = testcontainers_modules::postgres::Postgres::default()
    .with_password("1111")
    .start().await.unwrap();

    let connection_options = PgConnectOptions::new()
        .host(&container.get_host().await.unwrap().to_string())
        .port(container.get_host_port_ipv4(5432).await.unwrap())
        .database("postgres")
        .username("postgres")
        .password("1111");
    let pool = PgPoolOptions::new()
        .connect_with(connection_options).await.unwrap();

    Migrator::new(std::path::Path::new("../migrations")).await.unwrap()
        .run(&pool).await.unwrap();

    // Тестируемый объект хранилища
    let sut = StorageImpl::new(pool);

    // Изначально таблица с аккаунтами пуста
    let accounts = sut.fetch_accounts().await.unwrap();
    assert!(accounts.is_empty());

    // Создаём новй аккаунт
    let created_acc = sut.create_accounts(
            "Test-Account-1", BigDecimal::from_f64(1000.0).unwrap()
        ).await
        .unwrap();

    // Выбираем все аккаунты, чтобы убедиться, что свежесозданный аккаунт присутствует
    let accounts = sut.fetch_accounts().await.unwrap();
    assert_eq!(accounts.len(), 1);
    assert_eq!(accounts[0].id, created_acc.id);
    assert_eq!(accounts[0].owner_name, created_acc.owner_name);
    assert_eq!(accounts[0].balance, created_acc.balance);
}
```

## Крэйт server

В server/Cargo.toml

```toml
[package]
name = "server"
version = "0.1.0"
edition = "2024"

[dependencies]
persist = { path = "../persist" }
thiserror = { workspace = true }
async-trait = { workspace = true }
tokio = { workspace = true }
axum = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
sqlx = { workspace = true }
```

В крэйте server у нас следующие файлы с кодом:

* service.rs — модуль с бизнес логикой, которая построена вокруг вызовов функциональности из крэйта persist
* endpoints.rs — модуль содержащий функции обработчики эндпоинтов, которые вызывают функциональность, определённую в модуле service.rs
* lib.rs — содержит в себе непосредственно создание axum сервера
* main.rs — содержит функцию main, которая запускает функцию создания сервера из lib.rs

Файл `server/src/service.rs`:

```rust
use std::sync::Arc;
use persist::{Account, Storage};
use crate::endpoints::NewAcc;

#[derive(Debug, thiserror::Error)]
pub enum AccountServiceError {
    #[error("Database error: (0)")]
    StorageError(#[from] sqlx::Error),
}

#[async_trait::async_trait]
pub trait AccountService {
    async fn get_all_accounts(&self) -> Result<Vec<Account>, AccountServiceError>;
    async fn create_new_account(&self, new_acc: NewAcc) -> Result<Account, String>;
}

pub struct AccountServiceImpl {
    storage: Arc<dyn Storage + Send + Sync>,
}

impl AccountServiceImpl {
    pub fn new(storage: Arc<dyn Storage + Send + Sync>) -> AccountServiceImpl {
        AccountServiceImpl { storage }
    }
}

#[async_trait::async_trait]
impl AccountService for AccountServiceImpl {
    async fn get_all_accounts(&self) -> Result<Vec<Account>, AccountServiceError> {
        self.storage.fetch_accounts().await
            .map_err(AccountServiceError::from)
    }
    async fn create_new_account(&self, new_acc: NewAcc) -> Result<Account, String> {
        self.storage.create_accounts(&new_acc.owner_name, new_acc.init_balance).await
            .map_err(|e|e.to_string())
    }
}
```

Обратите внимение, что структура `AccountServiceImpl` инкапсулирует хранилище по средствам `Arc<dyn Storage + Send + Sync>`. Это позволит нам иметь возможность заинжектить в `AccountServiceImpl` как реальное хранилище — `StorageImpl`, так и какую-то заглушка для тестов.  Ограничения `Send` и `Sync` необходимы так методы трэйта асинхронный.

Сам же объект `AccountServiceImpl` мы будем хранить в объекте сосотояния, причём тоже не напрямую, а по средствам `Arc<dyn AccountService + Send + Sync>`.

Файл `server/src/lib.rs`:

```rust
pub mod service;
pub mod endpoints;

use std::sync::Arc;
use axum::{Router, routing::{get, post}};
use persist::StorageImpl;
use crate::service::{AccountService, AccountServiceImpl};

pub struct AppState {
    pub account_service: Arc<dyn AccountService + Send + Sync>,
}

pub async fn run_server() {
    let storage = StorageImpl::from_connection_url(
        "postgres://postgres:1111@localhost/mydb"
    ).await.unwrap();

    let account_service = AccountServiceImpl::new(Arc::new(storage));

    let state = AppState { account_service: Arc::new(account_service) };

    let app = Router::new()
        .route("/accounts", get(endpoints::list_accounts))
        .route("/accounts", post(endpoints::create_new_account))
        .with_state(Arc::new(state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}
```

И теперь сами эндпоинты в файле `server/src/endpoints.rs`:

```rust
use std::sync::Arc;
use axum::{Json, extract::State, http::StatusCode};
use serde::Deserialize;
use sqlx::types::BigDecimal;
use persist::Account;
use crate::AppState;

#[derive(Deserialize)]
pub struct NewAcc {
    pub owner_name: String,
    pub init_balance: BigDecimal,
}

pub async fn list_accounts(
    State(state): State<Arc<AppState>>
) -> Result<Json<Vec<Account>>, (StatusCode, String)> {
    match state.account_service.get_all_accounts().await {
        Ok(accounts) => Ok(Json(accounts)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}

pub async fn create_new_account(
    State(state): State<Arc<AppState>>, Json(acc): Json<NewAcc>
) -> Result<Json<Account>, (StatusCode, String)> {
    match state.account_service.create_new_account(acc).await {
        Ok(account) => Ok(Json(account)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}
```

Как видите, эндпоинты просто достают `AccountService` из состояния, и вызывают его методы.

## Миграции

