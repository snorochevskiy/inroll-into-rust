# Concurrency

Rust concurrency approach is very flexible because it separates an abstraction of asynchronous code and the way how it executed.

The key components are:

* async function
* Future trait
* async runtime

## Async

“async” keyword allows to define a function that is to be executed concurrently.

```
async fn function_name(arguments) -> ResultType {
	...
}
```

E.g.

```
async fn get_1() -> i32 {
    1
}
```

Rust compiler will translate this function into something like:

```
fn get_1() -> impl Future<Output=i32>+Sized {
    1
}
```

The concept of Future in Rust is different from futures in Java.

* In Java/Scala if you have a Future – it is already being executed
* In Rust Future is lazy, and wont compute until you force them and provide a runtime

We can think of Future as of a wrapped computation (like Mono in Spring Reactor, or IO in Cats-Effects).

The std::future::Future trait is defined as following:

```
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

The poll method signature is pretty complex, but we need to know only 2 things:

* future implementation is lazy and poll based
* at the moment of a call to the poll method, the result may not be ready, but the _Context_ parameter contains a callback that can be used later to notify the runtime it can poll for the result again

## Executing a future

The simplest possible runtime is a runtime that executes everything in a single thread.

Such single-thread is provided by _futures_ library.

This library provide a function called block\_on that executes a future on the current thread:

```
use futures::executor::block_on;

async fn get_1() -> i32 {
  1
}

fn main() {
  let fut: impl Future<Ouput=i32>+Sized = get_1();
  let result = block_on(fut);
  println!("{}", result);
}
```

We will take a look at more complex runtimes later.

## Composing Async

The most beautiful thing about async functions in Rust: they can be compose in a manner like if it a synchronous linear code.

The key word .await allows to compose async function in a wait/notify style (similar to await on promise in JS).

```
struct User     { user_id: u64, name: String,   addr_id: u64 }
struct Address  { addr_id: u64, location: String }
struct UserInfo { user: User,   addr: Address }

async fn get_user_by_id(user_id: u64) -> User {
    … 
}

async fn get_address_by_id(addr_id: u64) -> Address {
    …
}

async fn get_user_info(user_id: u64) -> UserInfo {
  let user: User    = get_user_by_id(user_id).await;
  let addr: Address = get_address_by_id(user.addr_id).await;
  UserInfo { user, addr }
}
```

.await can be called only from an async function

Similar Java code would look like:

```
CompletableFuture<User> fetchUserById(Long userId) { ... }

CompletableFuture<Address> fetchAddressById(Long addrId) { … }

CompletableFuture<UserInfo> getUserInfo(Long userId) {
  return fetchUserById(userId)
    .thenCompose(user ->
       fetchAddressById(user.getAddrId())
         .thenApply(addr ->
           new UserInfo(user, addr)
         )
    );
}
```
