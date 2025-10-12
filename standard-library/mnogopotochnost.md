---
hidden: true
---

# Многопоточность

In Rust we can work with OS threads.

We use `thread::spawn` to create a new thread from an anonymous function, and we can use `join` method to wait for the thread to finish.

```
use std::thread;

fn main() {
    let t1: JoinHandle<i32> = thread::spawn(||{ 1 });
    let t2 = thread::spawn(||{ 2 });

    let sum = t1.join().unwrap() + t2.join().unwrap();

    println!("{sum}");
}
```

The execution time of a thread is unpredictable, that’s why thread can use references only on data with static runtime.

```
use std::thread;

const greeting: &'static str = "Hello ";

fn main() {
    let t = thread::spawn(||{
        println!("{} world", greeting);
    });
    t.join();
}
```

If we try use any non static reference, the compiler will complain that the thread can outlive the data it refers to. That’s why, for a thread the only way to use an object is take the ownership on it, by “moving” the object into the thread.

```
use std::thread;

fn main() {
    let greeting: &str = "Hello ";
    let t = thread::spawn(move ||{
        println!("{} world", greeting);
    });
    t.join();
}
```

More complex example:

```
use std::thread::{self, JoinHandle};
use std::fs;

fn main() {
    let files = vec![
        "/etc/fstab".to_string(),
        "/etc/hosts".to_string(),
    ];

    let mut threads: Vec<JoinHandle<Result<String, _>>> = Vec::new();
    for i in 0 .. files.len() {
        let file_clone = files[i].clone(); // copy to be moved into the thread
        let t = thread::spawn(move || {
            fs::read_to_string(file_clone) // returns Result<String,_>
        });
        threads.push(t);
    }

    let mut total_size = 0;
    for t in threads {
        let file_size = t.join()
            .unwrap() // unwrap thread result
            .unwrap() // unwrap result of file read operation
            .len();
        total_size += file_size;
    }

    println!("Total size: {}, Files: {:?}", total_size, files );
}
```
