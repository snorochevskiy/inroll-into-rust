---
hidden: true
---

# Умные указатели

If we want to move object to the heap, we can use std::boxed::Box type.

Box consists of a pointer to the heap where the actual data is stored, but the Box itself resides on the stack.

Box owns the data it points to, and we can say it is a direct analogy for _unique\_ptr_ in C++.

<img src="../.gitbook/assets/file.excalidraw (4).svg" alt="" class="gitbook-drawing">

The syntax:

```
struct Point2D { x: i32, y: i32 }

fn main() {
    let p: Point2D = Point2D {x: 5, y: 2}; // Creating object on the stack
    let b: Box<Point2D> = Box::new(p);     // Moving from the stack to the heap
}
```

Box – is a, so called “zero cost abstraction”, meaning it compiles into a regular pointer.

To keep a value on the stack, the compiler should be able to determine the size of the value at the compile type, which is impossible for types like String and Vec.

E.g. it is not possible to know the size of a linked list at the compile time

\`\`

```rust
enum List<T> {
    Nil,
    Elem(T, List<T>),
}           ------- recursive without indirection
// error[E0072]: recursive type `List` has infinite size
// insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
//   Elem(T, Box<List<T>>),
//           ++++       +
```

Wrapping into a Box (i.e. moving to the heap) solves this problem.

```rust
#[derive(Debug)]
enum List<T> {
    Nil,                   // empty list
    Elem(T, Box<List<T>>), // an element and pointer to the tail
}

use List::*;

fn main() {
    let list: List<i32> =
        Elem(1, Box::new(
            Elem(2, Box::new(Nil))
        ));
    println!("{:?}", list); // Elem(1, Elem(2, Nil))
}
```

Note that this list doesn’t allow to mutate elements, because each element is already referenced by a reference, meaning we cannot take one more mutable reference.

We will see how to solve this problem later.

## Trait Deref

Box allows to use indirection operator `*` as if the box is a regular reference, and use `&` operator to get direct reference to the data “inside” the box;

```
fn main() {
    let mut b = Box::new(1);
    *b = 2;
    println!("{b}"); // 2

    increment(&mut b);
    println!("{b}"); // 3
}

fn increment(i: &mut i32) {
    *i += 1;
}
```

This is possible, because Box implements special trait – Deref.

```
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```

Compiler replaces `*mybox` with `*(mybox.deref())`

## Sharing data with Rc

Rust ownership principles doesn’t allow to share data (each object is owned only by one variable, and the object is deleted when the owned goes out of the scope), but in reality the are types of functionalities that require data sharing.

To share data we use Rc type, which is similar to Box, but in addition to the pointer to the heap, it also has a counter internally shared between all Rc copies.

* If we clone an Rc object, we just create a copy of the Rc pointer and increment the reference counter
* When the Rc object is destroyed, the reference counter is decremented
* When the reference counter hits zero, the object in heap is deleted

<img src="../.gitbook/assets/file.excalidraw (5).svg" alt="" class="gitbook-drawing">

An equivalent of RC in C++ is _shared\_ptr_.

Example:

```
use std::{rc::Rc, borrow::BorrowMut};
#[derive(Debug)]
struct City {
    name: String,
    country: Rc<String>
}

fn main() {
    let us = Rc::new("USA".to_string());
    let ny = City {name: "New York".to_string(), country: us.clone() };
    let odessa = City {name: "Odessa".to_string(), country: us.clone() };

    println!("{:?}", odessa); // City { name: "Odessa", country: "USA" }
}
```

## Mutable Rc

`Cell<T>` is a wrapper type that behaves like an immutable value, but allows to replace the value it hold inside with a new value.

```
use std::cell::Cell;

fn main() {
    let c1 = Cell::new(1);
    println!("c1={}", c1.get()); // c1 = 1

    c1.set(2);
    println!("c1={}", c1.get()); // c1 = 2
    
    let prev = c1.replace(3);
    println!("prev={}, c1={}", prev, c1.get()); //prev = 2, c1 = 3

    let c2 = Cell::new(5);
    c1.swap(&c2);
    println!("c1={}, c2={}", c1.get(), c2.get()); // c1 = 5, c2 = 3
}
```

Cell is not thread safe.

Now using a combination of Rc and Cell we can have an object that can be both shared among multiple owners (not in different threads) an mutated.

```
use std::{cell::Cell, rc::Rc};

fn main() {
    let rc1 = Rc::new(Cell::new(1));
    let rc2 = rc1.clone();

    rc2.as_ref().set(5);

    println!("{:?}", rc1); // Cell { value: 5 }
}
```

## RefCell

When Cell allows only the replacement of the nested data, RefCell\<T> allows the editing.

RefCell\<T> can loan a reference to the nested data to the caller code. And this reference can be used to read or mutate data (if we have borrowed a mutable reference).

```
use std::cell::{RefCell, RefMut};

fn main() {
    let ref_cell = RefCell::new(1);
    {
        let mut mut_ref: RefMut<'_, i32> = ref_cell.borrow_mut();
        *mut_ref = 5;
    }
    println!("{:?}", ref_cell);
}
```

RefCell keeps track of borrowed references, because RefCell doesn’t allow to borrow immutable and mutable references simultaneously. That’s why it still provides borrow-checker like safety, but just moves this check from the compile time to the run time.

```
use std::cell::{RefCell, RefMut, Ref};

fn main() {
    let ref_cell = RefCell::new(1);

    let immut_ref: Ref<'_, i32> = ref_cell.borrow(); // borrowing immutable

    let mut mut_ref: RefMut<'_, i32> = ref_cell.borrow_mut();
    *mut_ref = 5;                      ^^^ already borrowed: BorrowMutError
    
    println!("{:?}", ref_cell);
}
```

Now we can rewrite the Linked list implementation from example from Box paragraph.

```
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
enum List<T> {
    Elem(Rc<RefCell<T>>, Rc<List<T>>),
    Nil,
}

use List::*;

fn main() {
    let v = Rc::new(RefCell::new(1));

    let a = Rc::new(Elem(Rc::clone(&v), Rc::new(Nil)));

    let b = Elem(Rc::new(RefCell::new(2)), Rc::clone(&a));
    let c = Elem(Rc::new(RefCell::new(3)), Rc::clone(&a));

    *v.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

## Arc

Rc allows to share an object, but it is not thread safe. If we need to share an object between concurrent flows, we need to use Arc – thread safe version of Rc.

```
use std::sync::{Arc, Mutex, MutexGuard};
use std::thread;

fn main() {
  let counter: Arc<Mutex<i32>> = Arc::new(Mutex::new(0));

  let mut threads = Vec::new();
  for _ in 0 .. 10 {
    let counter_clone = counter.clone();
    let t = thread::spawn(move || {
      for _ in 0 .. 50 {
        let mut mut_guard: MutexGuard<'_, i32> = counter_clone.lock().unwrap();
        *mut_guard += 1; // MutexGuard provides reference like access to data
      } // MutexGuard goes out of the scope, and releases the mutex
    });
    threads.push(t);
  }
  for t in threads {
    let _ = t.join();
  }
  println!("{:?}", counter);
}
```

## Cow

TODO
