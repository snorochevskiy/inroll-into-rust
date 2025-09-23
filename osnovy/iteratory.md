---
hidden: true
---

# Итераторы

Трэйт итератор:

```rust
pub trait Iterator {
    type Item;
    
    fn next(&mut self) -> Option<Self::Item>;
    
    ... более 70 других методов
}
```

## Цикл for и итераторы



## API итераторов

Rust offers a convenient API for collections processing that is very similar to Java Stream API.

{% columns %}
{% column %}
Java

```java
// Transformation
var v1 = List.of(1,2,3,4,5,6,7,8,9);
var v2 = v1.stream()
  .filter(a -> a % 2 == 0)
  .map(a -> a + 1)
  .toList(); // [3,5,7,9]


// Folding
var product = List.of(1,2,3).stream()
  .reduce(1, (acc, e) -> acc * e); // 6


// Natural transformation
Optional<Integer> onlyEven(int a) {
  return a % 2 == 0 ?
    Optional.of(a) :
    Optional.empty();
}

var v3 = List.of(1,2,3).stream()
  .flatMap(a -> onlyEven(a).stream())
  .collect(Collectors.toList()); // [2]
```
{% endcolumn %}

{% column %}
Rust

```rust
// Transformation
let v1 = vec![1,2,3,4,5,6,7,8,9];
let v2: Vec<i32> = v1.iter()
  .filter(|&a| a % 2 == 0)
  .map(|a| a + 1)
  .collect(); // [3,5,7,9]


// Folding
let product = vec![1,2,3].iter()
  .fold(1, |acc, e| acc * e); // 6


// Natural transformation
fn only_even(a: i32) -> Option<i32> {
  if a % 2 == 0 { Some(a) }
  else { None }
}


let v3 = vec![1,2,3].into_iter()
  .filter_map(only_even)
  .collect::<Vec<_>>(); // [2]
```
{% endcolumn %}
{% endcolumns %}

