# Использование Option (Using Option)

Особенно наблюдательные читатели могли заметить, что мы фактически заново изобрели
очень плохую версию `Option`:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

`Link` — это просто `Option<Box<Node>>`. Конечно, приятно не писать
`Option<Box<Node>>` повсюду, и, в отличие от `pop`, мы не выставляем это
внешнему миру, так что, возможно, все в порядке. Однако у `Option` есть несколько *очень
приятных* методов, которые мы реализовывали вручную. Давайте *не будем*
этого делать и заменим все на `Option`. Сначала мы сделаем это наивно,
просто переименовав все для использования `Some` и `None`:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// ура, псевдонимы типов (type aliases)!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

Это немного лучше, но главные преимущества мы получим от методов `Option`.

Во-первых, `mem::replace(&mut option, None)` — это настолько невероятно
распространенная идиома, что в `Option` ее просто взяли и сделали методом: `take`.

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

Во-вторых, `match option { None => None, Some(x) => Some(y) }` — это настолько
невероятно распространенная идиома, что ее назвали `map`. `map` принимает функцию для
выполнения над `x` в `Some(x)`, чтобы получить `y` в `Some(y)`. Мы могли бы
написать полноценную функцию (`fn`) и передать ее в `map`, но мы бы предпочли написать то, что нужно
сделать, *прямо на месте (inline)*.

Способ сделать это — использовать *замыкание (closure)*. Замыкания — это анонимные функции с
дополнительной суперсилой: они могут ссылаться на локальные переменные *вне* замыкания!
Это делает их суперполезными для выполнения всевозможной условной логики. Единственное
место, где мы используем `match` — это `pop`, так что давайте просто перепишем его:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

Ах, так гораздо лучше. Давайте убедимся, что мы ничего не сломали:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

Отлично! Давайте перейдем к фактическому улучшению *поведения* кода.
