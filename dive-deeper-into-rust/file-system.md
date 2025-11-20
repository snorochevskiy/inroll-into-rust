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

Почти все функци дла работы с файловой системой в качестве строк используют не `String`, а [**OsString**](https://doc.rust-lang.org/std/ffi/struct.OsString.html). Этот тип хранит строки в том представлении, в котором строки хранятся в текущей операционной системе.

Отдельный тип строки используется потому что:

* В Unix подобных системах, имена файлов хранятся в виде последовательностей ненулевых байт, в которой хранится строка в UTF-8 или другой кодировке.
* На Windows имена файлов представлены последовательностями ненулевых двухбайтных значений (UTF-16).
* Rust строки всегда хранятся в UTF-8 кодировке, причем нулевые символы допустимы.

Для `OsString` реализованы `From<String>` и `From<&str>`, который позволяет легко преобразовать "обычную" строку в `OsString`.

```rust
use std::ffi::OsString;

fn main() {
    let string = "text";
    let os_string = OsString::from(string);
}
```

Как для типа `String` есть парный тип — строковый-слайс `&str`, так и для `OsString` есть соответствующий тип — `&OsStr`.

```rust
use std::ffi::{OsStr, OsString};

fn main() {
    let string = "text";
    let os_string = OsString::from(string);
    let os_str: &OsStr = &os_string;
}
```

***

В большинстве функций для работы с именами файлов используется тип [**Path**](https://doc.rust-lang.org/std/path/struct.Path.html). Фактически, этот тип является просто обёрткой над `OsStr`:

```rust
pub struct Path {
    inner: OsStr,
}
```

Стандартная библиотека предоставляет `From`/`Into` и `AsRef` преобразования, которые позволяют бесшовно конвертировать Rust строки в `Path`.

Например, сигнутура метода `File::create`, который мы использовали в примере создания файла, имеет такой виде:

```rust
pub fn create<P: AsRef<Path>>(path: P) -> io::Result<File>
```

И именно потому что для `&str` определён `AsRef<Path>`, мы смогли вызвать этот метод как:

```rust
let mut file = File::create("file.txt")?;
```
