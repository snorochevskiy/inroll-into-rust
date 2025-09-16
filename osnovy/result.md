# Result

Another ubiquitous generic enum is Result. When Option is used to represent a possible lack of the value, the Result type contains either expected value, or an error.

(It is similar to Either type from Scala or Haskell)

```
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

This type is very important because there are no exceptions in Rust. That’s why all operations that can fail, has return type wrapped in Result.

For example:

```
fn safe_div(a: f32, b: f32) -> Result<f32, String> {
    if b == 0 {
        Err("Cannot divide by zero")
    } else {
        Ok(a / b)
    }
}
```

## Composing Result

Let’s look how to use multiple functions that return Result, with a program that prints a text file:

```
use std::fs::File;
use std::io::{Error, prelude::*};

fn read_text_file(file_name: &str) -> Result<String, Error> {
    // open file
    let mut file = match File::open(file_name) {
		Ok(file) => file,
		Err(e)   => return Err(e),
	};

    // read text from file
    let mut contents = String::new();
    match file.read_to_string(&mut contents) {
		Ok(read_bytes) => Ok(contents),
		Err(e)         => return Err(e),
	}
} 

fn main() {
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

{% hint style="info" %}
Those who familiar with Golang, can notice it is similar to if err != nil.
{% endhint %}

Another way is to use combinators map and and\_then:

```
use std::fs::File;
use std::io::{Error, prelude::*};

fn read_text_file(file_name: &str) -> Result<String, Error> {
    File::open(file_name).and_then(|mut file| {
        let mut contents = String::new();
        file.read_to_string(&mut contents).map(|_| {
            contents
        })
    })
}

fn main() {
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

But in truth, Rust has a special syntax operator for unwrapping Result type - ?

If the Result contains a “good” value then ? just returns it, but if it contains error, then ? Immediately exits the function and propagates the error to the caller code.

```
use std::fs::File;
use std::io::{Error, prelude::*};

fn read_text_file(file_name: &str) -> Result<String, Error> {
    let mut file = File::open(file_name)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
} 

fn main() {
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

