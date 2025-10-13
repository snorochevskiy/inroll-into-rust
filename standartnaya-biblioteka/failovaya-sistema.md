# Файловая система

Для работы с файловой системой стандартная библиотека Rust предоставляет такие модули:

* [**std::fs**](https://doc.rust-lang.org/std/fs/index.html) — содержит функциональность для работы непосредственно с объектами файловой системы: файлами, директориями, ссылками.
* [**std::io**](https://doc.rust-lang.org/std/io/index.html) — содержит функциональность для работы с операциями ввода/вывода

## Чтение и запись файла

Для примера работы с файлом, напишем простую программу, которая:

* открывает новый файл, записывает в него текст, закрывает файл
* открывает этот же файл, добавляет в него строку, закрывает файл
* опять открывает файл и считывает из него всё содержимое в строку

```rust
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write};

fn main() -> io::Result<()> {
    {
        // Создаёт новый файл (или перезаписывает имеющийся)
        let mut file = File::create("file.txt")?;
        // Записывает в файл байты
        file.write_all("First line\n".as_bytes())?;
        file.flush()?; // Очистка буфера вывода
        file.write_all("Second line\n".as_bytes())?;
    }

    {
        // Открываем файл для добавления
        let mut file = OpenOptions::new()
            .append(true)
            .create(false)
            .open("file.txt")?;
        file.write_all("Third line\n".as_bytes())?;
    }

    {
        // Открываем файл для чтения
        let mut file = File::open("file.txt")?;
        let mut buffer = String::new();
        file.read_to_string(&mut buffer)?;
        println!("{buffer}");
    }

    Ok(())
}
```

Первое, что бросается в глаза: мы открываем файл для записи, но не закрываем его. Дело том, что для типа `File` реализован трэйт `Drop`: деструктор сбросит буфер вывода (вызовом `flush`), а после закроет файл.

Метод `write_all` объявлен в трэйте `std::io::Write`, благодаря чему код чтения и записи выглядит аналогично для любого I/O ресурса: как для файла, так и для сетевого соединения.

Тип `io::Result<T>` — это псевдоним объявленный как:

```rust
pub type Result<T> = result::Result<T, std::io::Error>;
```

то есть, это обычный `Result`, который параметризирован ошибкой специфичной для I/O.

## Чтение директории

Напишем программу, которая выводит имена файлов и директорий, находящихся в текущем каталоге:

```rust
use std::{ffi::OsString, fs::FileType};

fn main() -> std::io::Result<()> {
    for read_entry in std::fs::read_dir(".")? {
        if let Ok(entry) = read_entry {
            let entry_name: OsString = entry.file_name();
            let file_type: FileType = entry.file_type()?;
            println!("{entry_name:?} {file_type:?}");
        }
    }
    Ok(())
}
```

Вывод программы:

```
"target" FileType { is_file: false, is_dir: true, is_symlink: false, .. }
"Cargo.lock" FileType { is_file: true, is_dir: false, is_symlink: false, .. }
"src" FileType { is_file: false, is_dir: true, is_symlink: false, .. }
"Cargo.toml" FileType { is_file: true, is_dir: false, is_symlink: false, .. }
```

## OsString

TODO
