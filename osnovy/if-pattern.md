# if pattern

We can also use a destructuring patterns with an if expression.

```
let o1: Option<i32> = ...;
if let Some(i) = o1 {
    println!("Number exists: {}", i);
}
```

Construction

```
if let PATTERN = VALUE {
	expression 1
} else {
	expression 2
}
```

is an equivalent to

```
match VALUE {
    PATTERN => expression 1,
    _       => expression 2,
}
```

Same approach works with while loop. It continues looping while elements match a given pattern.

```
use regex::Regex;

const REQUEST: &str =
r#"POST /cgi-bin/process.cgi HTTP/1.1
User-Agent: Mozilla/4.0 (compatible; MSIE5.01; Windows NT)
Content-Type: text/xml; charset=utf-8
Content-Length: length
Connection: Keep-Alive

<?xml version="1.0" encoding="utf-8"?>
<string xmlns="http://clearforest.com/">string</string>"#;

fn main() {
  let lines = REQUEST.lines();
  let mut iterator = lines.skip(1);

  let re = Regex::new(r"([-a-zA-Z]+): (.+)").unwrap();
	
  while let Some((s, [header, value])) =
     iterator.next()                  // Option<&str>
       .and_then(|e| re.captures(e))  // Option<Captures<’_>>
       .map(|c| c.extract()) {        // Option<(&str, [&str; 2])>

    println!("Header={}, value={}", header, value);
  };
}
```

## Let Pattern

TODO
