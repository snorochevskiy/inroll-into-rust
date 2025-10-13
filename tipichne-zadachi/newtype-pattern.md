---
hidden: true
---

# Newtype pattern

One on the most annoying Rust rules – is so called “Orphan rule”. It states that when you want to implement a trait for a type, this should be done in a crate that owns either the trait, or the type (or both).

In other words, if you want to implement trait A for type B, this should be done in the crate where defined either A or B.

```
use std::fs::File;

fn main() {
    let mut v = vec![
        File::open("/etc/fstab").unwrap(),
        File::open("/etc/resolv.conf").unwrap(),
    ];
    v.sort(); // the trait `Ord` is not implemented for `File`
}
```

f you need to implement an interface for a type in a crate that doesn’t own the trairt or the type, the standard workaround is to wrap the type into a wrapper – new type.

Method sort:

```
impl<T> [T] {
    pub fn sort(&mut self) where T: Ord { ... }
}
```

```
pub trait Ord: Eq + PartialOrd { ... }

pub trait PartialOrd<Rhs = Self>: PartialEq<Rhs> { ... }
```

```
use std::{cmp::Ordering, fs::File};

fn main() {
    let mut v = vec![
        FileWrapper(File::open("/etc/fstab").unwrap()),
        FileWrapper(File::open("/etc/resolv.conf").unwrap()),
    ];
    v.sort();
}

struct FileWrapper(File);

impl PartialEq for FileWrapper {
    fn eq(&self, other: &Self) -> bool {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len() == m2.len(),
            _ => false,
        }
    }
}
impl Eq for FileWrapper { }

impl PartialOrd for FileWrapper {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for FileWrapper {
    fn cmp(&self, other: &Self) -> Ordering {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len().cmp(&m2.len()),
            _ => Ordering::Equal,
        }
    }
}
```
