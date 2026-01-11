# async\_trait

В этой главе мы рассмотрим решение одной проблемы, которая возникает при использовании async методов в трэйтах.

Допустим мы разрабатывает бекенд для работы с некими товарами. Мы решили организовать код в виде двух компонентов: компонент который отвечает за работы с базой данных, и компонент, который отвечает за бизнес логику.

```rust
fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService { product_storage };
    println!("All products: {:?}", product_service.get_all_products())
}

#[derive(Debug)]
struct Product { } // Некий тип товара

// Бизнес логика работы с продуктами
struct ProductService {
    product_storage: ProductDbStorage,
}
impl ProductService {
    fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products()
    }
}

// Работа с таблица в БД, которые хранят продукты
struct ProductDbStorage {
    // содержит соединение к БД
}
impl ProductDbStorage {
    fn list_products(&self) -> Vec<Product> {
        // представим, что здесь происходит обращение к реальной базе данных
        Vec::new()
    }
}
```

Однако, такая жёсткая связанность компонентов не очень удобна. Например, если мы захотим написать юнит-текст для `ProductService`, то мы не сможем проинициализировать поле `storage` ничем, кроме `ProductDbStorage`, который работает с реальной базой данных. А ведь юнит-тест не подразумевает работу с реальными системами, и все обращения к ним должны быть заменены заглушками.

Чтобы ослабить связанность между компонентами:

* абстрагируем тип `ProductDbStorage` через трэйт `ProductStorage`
* в `ProdcutService` в полек `storage`, вместо непосредственно объекта `ProductDbStorage`, будем хранить `Box<dyn ProductStorage>`.

```rust
fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService {
        product_storage: Box::new(product_storage)
    };
    println!("All products: {:?}", product_service.get_all_products())
}

#[derive(Debug)]
struct Product { }

struct ProductService {
    product_storage: Box<dyn ProductStorage>,
}
impl ProductService {
    fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products()
    }
}

trait ProductStorage {
     fn list_products(&self) -> Vec<Product>;
} 

struct ProductDbStorage { }

impl ProductStorage for ProductDbStorage {
    fn list_products(&self) -> Vec<Product> { Vec::new() }
}

#[test]
fn test_product_service() {
    // Заглушка для ProductStorage
    struct ProductStorageMock;
    impl ProductStorage for ProductStorageMock {
        fn list_products(&self) -> Vec<Product> {
            vec![Product {}]
        }
    }

    let sut = ProductService {
        product_storage: Box::new(ProductStorageMock) // заглушка
    };

    assert_eq!(sut.get_all_products().len(), 1);
}
```

И вот теперь, давайте попытаемся переписать `ProductService` и `ProductStorage` так, чтобы их методы были асинхронными.

```rust
struct ProductService {
    product_storage: Box<dyn ProductStorage>,
}                    // Error: the trait `ProductStorage` is not dyn compatible
impl ProductService {
    async fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products().await
    }
}

trait ProductStorage {
    async fn list_products(&self) -> Vec<Product>;
} 

struct ProductDbStorage { }

impl ProductStorage for ProductDbStorage {
    async fn list_products(&self) -> Vec<Product> { Vec::new() }
}
```

Увы, но мы увидим ошибку компиляции:

> the trait \`ProductStorage\` is not dyn compatible

Эта ошибка гласит о том, что компилятор не может создать трэйт-объект для трэйта с async методами. Чтобы понять почему так происходит, мы должны вспомнить, что функция вида:

```rust
async fn myfync() -> MyType {}
```

Превращается компилятором в

```rust
fn myfync() -> impl Future<Output = MyType> {}
```

В этом и кроется проблема. Чтобы создать трэйт-объект, компилятор должен построить таблицу виртуальных вызовов vtable. Однако одно из условий для этого состоит в том, что компилятор должен знать размер типов, возвращаемых методами (чтобы иметь возможность аллоцировать место на стеке), но это невозможно, так как `impl Future` имеет неизвестный размер.

Стандартное решение этой проблемы формирования vtable: завернуть возвращаемый тип в `Box`, чей размер на стеке известен. А так как мы имеем дело с асинхронным кодом (а `Box` не безопасен для многопоточности), то вместо `Box`, следует использовать `Pin<Box>`.

Вооружившись этим знанием, давайте перепишем наш пример:

```rust
use std::pin::Pin;

#[tokio::main]
async fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService {
        product_storage: Box::new(product_storage)
    };
    println!("All products: {:?}", product_service.get_all_products().await)
}

#[derive(Debug)]
struct Product { }

struct ProductService {
    product_storage: Box<dyn ProductStorage + Send + Sync>,
}
impl ProductService {
    async fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products().await
    }
}

trait ProductStorage {
    fn list_products<'a, 't>(
        &'a self
    ) -> Pin<Box<dyn Future<Output = Vec<Product>> + Send + 't>>
    where 'a: 't, Self: 't;
} 

struct ProductDbStorage { }

impl ProductStorage for ProductDbStorage {
    fn list_products<'a, 't>(
        &'a self,
    ) -> Pin<Box<dyn Future<Output = Vec<Product>> + Send + 't>>
    where 'a: 't, Self: 't {
        Box::pin(async move { Vec::new() })
    }
}
```

Этот код успешно компилируется и запускается. Но согласитесь: выглядит от не слишком элегантно.

К счастью, существуется библиотека [async-trait](https://crates.io/crates/async-trait), которая поможет исправить ситуацию. Просто пометте и трэйт, и его реализацию аннотацией `#[async_trait]`, и пишите код с использованием обычных async методов без всех этих `Pin<Box>`. Во время компиляции, обработчик аннотации сам заменит async методы на методы возвращающие `Pin<Box>`.

```rust
#[tokio::main]
async fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService {
        product_storage: Box::new(product_storage)
    };
    println!("All products: {:?}", product_service.get_all_products().await)
}

#[derive(Debug)]
struct Product { }

struct ProductService {
    product_storage: Box<dyn ProductStorage + Send + Sync>,
}
impl ProductService {
    async fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products().await
    }
}

#[async_trait::async_trait]
trait ProductStorage {
     async fn list_products(&self) -> Vec<Product>;
} 

struct ProductDbStorage { }

#[async_trait::async_trait]
impl ProductStorage for ProductDbStorage {
    async fn list_products(&self) -> Vec<Product> { Vec::new() }
}
```

