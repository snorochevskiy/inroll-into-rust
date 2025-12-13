---
hidden: true
---

# Версионирование структуры БД

SQLx предлагает утилиту командной строки [sqlx-cli](https://crates.io/crates/sqlx-cli), которая позволяет:

* вычитывать метаданные из БД, и валидировать SQL запросы на этапе компиляции
* быстро создавать и удалять тестовую БД
* версионировать структуры БД, которое часто называют **миграцией**

Именно версионирование структуры БД мы и разберём.

Для начала нам потребуется установить саму утилиту при помощи команды:

&#x20;`cargo install sqlx-cli`&#x20;

Теперь давайте перенесём наш SQL для создания таблиц, в файлы миграции. Создадим первый файл при помощи команды:

```
cargo sqlx migrate add accounts -r --sequential
```

Эта команда создаст два пустых файла:

* `migrations/0001_accounts.up.sql` — файл для изменения структуры БД
* `migrations/0001_accounts.down.sql` — файл для отката изменений

Откроем `0001_accounts.up.sql` и напишем в нём следующее:

```sql
CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    owner_name VARCHAR(255) NOT NULL UNIQUE,
    balance NUMERIC(10, 2)  NOT NULL DEFAULT 0.00
);
```

Далее `0001_accounts.down.sql`:

```sql
DROP TABLE accounts;
```

Теперь создадим следующую пару файлов для таблицы transactions.

Можно так же сделать это командой `cargo sqlx migrate add transactions -r --sequential`, но можно и создать файлы `0002_transactions.up.sql` и `0002_transactions.down.sql` вручную.

Заполним `0002_transactions.up.sql`:

```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    amount NUMERIC(10, 2) DEFAULT 0.00,
    src_account_id BIGINT NOT NULL,
    dst_account_id BIGINT NOT NULL,
    tx_timestamp TIMESTAMP NOT NULL,
    FOREIGN KEY (src_account_id) REFERENCES accounts (id),
    FOREIGN KEY (src_account_id) REFERENCES accounts (id)
);
```

И затем `0002_transactions.down.sql`:

```sql
DROP TABLE transactions;
```

Теперь, чтобы протестировать нашу миграцию, давайте создадим новую БД — my\_migration.

После создания БД, запустим миграцию:

```
$ cargo sqlx migrate run \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 1/migrate accounts (3.99736ms)
Applied 2/migrate transactions (4.774571ms)
```

В базе данных должны появиться три таблицы:

* `accounts`
* `transactions`
* `_sqlx_migrations`

Последяя — метаданные миграции. SQLx использует эту таблицу, чтобы знать какие миграции были уже выполнены. Таблица хранит номера применённых миграций, имена выполненых файлов миграции, время выполнения, а так же контрольную сумму файла миграции (нужна для проверки не был ли изменён файл уже применённой миграции).

```
> select * from _sqlx_migrations;
 version | description  |  installed_on       | success | checksum | execution_time
---------+------------- +---------------------+---------+---------------------------
       1 | accounts     | 2025-12-12 02:19:32 | t       | \x3124df | 3997360
       2 | transactions | 2025-12-12 02:19:32 | t       | \xaded4e | 4774571
```

Как мы видим, обе наши миграции были успешно применены.

Утилита `cargo sqlx migrate` позволяет не только применять миграции, но откатывать их. Например, давайте откатим структуру БД до версии 1, то есть откатим создание таблицы transactions.

```
$ cargo sqlx migrate revert \
    --target-version 1 \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 2/revert transactions (3.658186ms)
Skipped 1/revert accounts (0ns)
```

Если мы проверим какие таблицы есть в БД my\_migration, то мы увидим что таблиа transactions пропала.

Если мы запустим миграцию еще раз, то таблица transaction будет создана опять.

```
$ cargo sqlx migrate run \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 2/migrate transactions
```

