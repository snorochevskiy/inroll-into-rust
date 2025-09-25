# Циклы

## while

Цикл while в Rust работает и выглядит так же как и в других императивных языках:

```rust
while условие {
  тело цикла
}
```

Например, вывод на консоль чисел от `5` до `0` (не включительно).

```rust
fn main() {
    let mut n = 5;
    while n > 0 {
        println!("{n}");
        n -= 1;
    }
}
```

***

## do-while

В Rust нет do-while цикла, но при большой необходимости его можно сэмулировать через while:

{% columns %}
{% column %}
Такой код

```rust
fn main() {
  let mut start = 0;
  let mut sum = 0;

  while {
    sum += start;
    start += 1;
    start < 10
  } {};

  println!("sum: {}", sum);
}
```
{% endcolumn %}

{% column %}
работает словно

```rust
fn main() {
  let mut start = 0;
  let mut sum = 0;

do {
    sum += start;
    start += 1;
} while start < 10;


  println!("sum: {}", sum);
}
```
{% endcolumn %}
{% endcolumns %}

***

## loop

В Rust есть специальный "бесконечный" цикл - `loop`. По сути, это просто аналог `while true`.

```
loop {
  тело цикла
}
```

Как и из `while true` цикла, из `loop` можно выйти при помощи оператора `break`.

Например, программа, которая выводит числа последовательности Фиббоначи, меньше указанного (подразумеваем, что указанное число всегда больше 1).

```rust
fn main() {
  let maximum = 30;
  let mut a = 0;
  let mut b = 1;
  print!("{a} {b}");
  loop {
    let next = a + b;
    if next > maximum {
      break;
    }
    print!(" {next}");
    a = b;
    b = next;
  }
}
```

Оператор `break` для цикла `loop` иимеет еще одну функциланльность: он возвращает значение.

```rust
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2; // возвращаем из цикла
    }
};
println!("The result is {}", result); // 20
```

Тип оператора break - never type.

## for

Аппелируя к сложившийся терминологии, в Rust нет "классического" цикла **for**, только **for-each**, который предназначен для перебора последовательностей.

Синтаксис:

```rust
for переменная in последовательность {
  тело цикла
}
```

Например, перебор элементов массива выглядит так:

<pre class="language-rust"><code class="lang-rust"><strong>fn main() {
</strong><strong>    let arr = [10, 20, 30, 40, 50];
</strong>    
    for element in arr {
        println!("the value is: {}", element);
    }
}
</code></pre>

При помощи `for` можно перебирать элементы массивов, слайсов, векторов, и еще целого ряда коллекций.

## Перебор диапазона

В Rust нет "классического" цикцла for, однако цикл вида `for (int i=0; i<N; i++)` требуется довольно часто. Последовательный перебор всех целых чисел в заданном диапазоне можно сделать так:

```rust
for i in 0 .. 20 {
    println!("{}", i);
}
```

Этот цикл перебирает числа от 0 до 20 (не включительно).

Чтобы перебрать числа от 0 до 20 (включительно), надо вместо `0 .. 20` указать `0 ..= 20`.

{% hint style="info" %}
Детальнее о цикле for мы поговорим в главах [vladenie.md](vladenie.md "mention") и [iteratory.md](iteratory.md "mention")
{% endhint %}

