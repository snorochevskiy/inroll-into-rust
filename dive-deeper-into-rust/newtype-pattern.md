# Newtype паттерн

Одно из самых неудобных ограничений Rust — Orphan rule, о котором мы уже упоминали в главе [Трэйты](../rust-basics/treity.md#realizaciya-treita-dlya-chuzhikh-tipov). Напомним, Orphan Rule гласит: трэйт можно реализовать для типа, только в том случае, если либо трэйт, либо тип (либо оба) принадлежит крэйту (библиотеке или программе) в которой осуществляется реализация.

Другими словами, если мы хотим реализовать трэйт A для типа B, то код реализации должен располагаться либо в крэйте где объявлен тип B, либо в крэйте где объявлен тип A.

А теперь давате представим, что у нас есть задача в рамках которой нам надо сортировать вектор с объектами файлов:

```rust
use std::fs::File;

fn main() {
    let mut v = vec![
        File::open("/etc/fstab").unwrap(),
        File::open("/etc/resolv.conf").unwrap(),
        File::open("/etc/hosts").unwrap(),
    ];
    v.sort();
}
```

{% hint style="info" %}
Для пользователей не знакомых с Linux:

* /etc/fstab — стандартный файл конфигурации разделов жесткого диска
* /etc/resolv.conf — файл с адресами DNS серверов
* /etc/hosts — файл для задания соответствий доменных имён и IP адресов

Автор взял эти файлы без какого-то специального умысла. Проверяя примеры из главы, вы вольны использовать любые имеющийся у вас файлы, или создать новые.
{% endhint %}

Метод `.sort()` требует, чтобы тип сортируемых объектов реализовал трэйт `Ord`. Тип `std::fs::File` не реализует `Ord`, поэтому компиляции завершится с ошибкой:

```
error[E0277]: the trait bound `File: Ord` is not satisfied
   --> src/main.rs:8:7
    |
  8 |     v.sort(); // the trait `Ord` is not implemented for `File`
    |       ^^^^ the trait `Ord` is not implemented for `File`
```

Допустим мы хотим реализовать `Ord` для `File` так, чтобы сортировка происходила на основании размера файла. Здесь и проявляется Orphan Rule: и тип `File` и трэйт `Ord` объявлены не в нашем крэйте, а в стандартной библиотеке.

Стандартным решением этой проблемы является **Newtype паттерн**. Смысл его заключается в том, что мы оборачиваем "чужой" тип в кортежную структуру, и получается, что эта обёртка уже располагается в нашем крэйте.

```rust
struct Обёртка(ЧужойТип);
```

Теперь для этой обёртки мы можем реализовать `Ord`.

Так же, для удобства преобразования "чужого" типа в обёртку и обратно, стоит реализовать  трэйты `From` и `Deref`.

Теперь, вернёмся к нашему примеру с сортировкой файлов, и реализуем эту сортировку при помощи Newtype паттерна.

```rust
use std::{cmp::Ordering, fs::File, ops::Deref};

// Newtype обёртка для File.
struct FileWrapper(File);

impl PartialEq for FileWrapper {
    fn eq(&self, other: &Self) -> bool {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len() == m2.len(),
            _ => false,
        }
    }
}
impl Eq for FileWrapper {}

impl Ord for FileWrapper {
    fn cmp(&self, other: &Self) -> Ordering {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len().cmp(&m2.len()),
            _ => Ordering::Equal,
        }
    }
}

impl PartialOrd for FileWrapper {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl From<File> for FileWrapper {
    fn from(value: File) -> Self {
        FileWrapper(value)
    }
}

impl Deref for FileWrapper {
    type Target = File;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let mut v: Vec<FileWrapper> = vec![
        // Вызовом .into() преобразовываем File в Newtype обёртку
        File::open("/etc/fstab").unwrap().into(),
        File::open("/etc/resolv.conf").unwrap().into(),
        File::open("/etc/hosts").unwrap().into(),
    ];

    println!("Before sorting");
    for file in v.iter() {
        // Ниже, мы вызываем file.metadata(), а не file.0.metadata()
        // так как мы реализовали Deref
        println!("Size: {}", file.metadata().unwrap().len());
    }

    v.sort();

    println!("After sorting");
    for file in v.iter() {
        println!("Size: {}", file.metadata().unwrap().len());
    }
}
```
