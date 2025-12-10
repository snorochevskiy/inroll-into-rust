---
hidden: true
---

# SQLx

Очень часто бекенд приложения взаимодействуют с реляционными базами данных, поэтому давайте поговорим о работе с реляционными БД.

Для работы с реляционными СУБД мы будет использовать популярную библиотеку [SQLx](https://crates.io/crates/sqlx), которая поддерживает PostgreSQL, MySQL, SQLite и MS SQL Server.

SQLx предоставляет:

* драйверы для СУБД
* пулл соединений
* API для выполнения SQL и и получения ответов
* мапперы, преобразующие ответ от БД в объекты Rust структур

## Тестовая БД

Для примера будем использовать СУБД PostgreSQL. Мы можете поставить дистрибуети PostgreSQL локально, или использовать docker образ.

```
docker run --name my_pg_container \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=1111 \
  -d postgres

docker exec -it my_pg_container bash

psql  -U postgres

\list - показать схемы
\c postgres - выбрать схему postgres
\dt - показать таблицы в выбраной схеме

CREATE DATABASE mydb;
\c mydb;
```



```
┌────────────┐         ┌────────────────┐
│  accounts  │         │  transactions  │
├────────────┤         ├────────────────┤
│*id         │───┐     │*id             │
│ owner_name │   │     │ amount         │
│ balance    │   ├────*│ src_account_id │
└────────────┘   └────*│ dst_account_id │
                       │ tx_timestamp   │
                       └────────────────┘
```



```sql
CREATE TABLE accounts ( -- mydb.public.accounts 
    id SERIAL PRIMARY KEY,
    owner_name VARCHAR(255) NOT NULL UNIQUE,
    balance NUMERIC(10, 2) DEFAULT 0.00
);

CREATE TABLE transactions ( -- mydb.public.transactions 
    id SERIAL PRIMARY KEY,
    amount NUMERIC(10, 2) DEFAULT 0.00,
    src_account_id INTEGER NOT NULL,
    dst_account_id INTEGER NOT NULL,
    FOREIGN KEY (src_account_id) REFERENCES accounts (id),
    FOREIGN KEY (src_account_id) REFERENCES accounts (id),
    tx_timestamp TIMESTAMP NOT NULL
);
```

## Подключение к СУБД

Создадим новый проект

```
cargo new test_sqlx
```

И добавим sqlx в Cargo.toml

```toml
[package]
name = "test_sqlx"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["mysql", "chrono", "runtime-tokio"]}
chrono = "0.4"
```



## Выполнение запросов



## sqlx макросы



## Миграция

???

