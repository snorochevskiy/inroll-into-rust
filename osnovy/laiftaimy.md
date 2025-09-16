# Лайфтаймы

When we work with multiple references, sometimes Rust cannot automatically guarantee that value references are pointing stay alive to needed moment.

E.g. this code won’t compile:

```
fn take_longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let l = take_longest("aaa", "bbbb");
}
```

t fails with error:

> error\[E0106]: missing lifetime specifier\
> help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from \`x\` or \`y\`\
> help: consider introducing a named lifetime parameter

Potential problem:

```
let s1 = String::from("aaa");
let longest;
{
    let s2 = String::from("bbbb");
    longest = take_longest(s1.as_str(), s2.as_str());
}
```

To solve this problem we need to explicitly specify lifetime relations for these references:

```
fn take_longest<’a>(x: &’a str, y: &’a str) -> &’a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let l = take_longest("aaa", "bbbb");
}
```

This can be read as: given a lifetime “a” (arbitrary long) that is not shorter than the lifetime of function “longest”, and the lifetime of all the arguments and result type should correspond to same scope.

Because we have lifetime defined, this following code won’t compile:

```
fn main() {
  let s1 = String::from("aaa");
  let longest;
  {
    let s2 = String::from("bbbb");
    longest = take_longest(s1.as_str(), s2.as_str()); // does not live long enough
  }
  println!("The longest string is {}", result);
}
```

## How to live with lifetimes

If you have problems with complex lifetimes, than you are probably using too much of references.

It can be reasonable if you write an OS driver, or a firmware for a device with a limited amount of RAM.

In all other cases just clone the value instead of taking a reference, and you won’t face lifetime related problems.\
