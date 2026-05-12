# Итоговый код (The Final Code)

Итак, 6000 слов спустя, вот и весь код, который нам удалось написать:

```rust
use std::mem;

pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, Link::Empty),
        });

        self.head = Link::More(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, Link::Empty) {
            Link::Empty => None,
            Link::More(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);

        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
        }
    }
}

#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Проверяем, что пустой список ведет себя правильно
        assert_eq!(list.pop(), None);

        // Заполняем список
        list.push(1);
        list.push(2);
        list.push(3);

        // Проверяем нормальное извлечение
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Добавляем еще немного, чтобы убедиться, что ничего не испортилось
        list.push(4);
        list.push(5);

        // Проверяем нормальное извлечение
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Проверяем исчерпание списка
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

Божечки. 80 строк, и половина из них — тесты! Что ж, я ведь говорил, что с этой первой главой
придется повозиться!
