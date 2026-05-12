# Делаем всё обобщенным (Making it all Generic)

Мы уже немного коснулись обобщений (generics) в разговоре об `Option` и `Box`. Однако до
сих пор нам удавалось избегать объявления каких-либо новых типов, которые на самом деле были бы обобщенными
для произвольных элементов.

Оказывается, это очень просто. Давайте сделаем все наши типы обобщенными
прямо сейчас:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

Вы просто делаете все немного более «угловатым» (с использованием угловых скобок), и внезапно ваш код становится
обобщенным. Конечно, мы не можем *просто* сделать это, иначе компилятор будет
Супер Зол.

```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

Проблема вполне понятна: мы говорим об этой штуке `List`, но ее больше не существует в таком виде.
Как и в случае с `Option` и `Box`, теперь мы всегда должны говорить о
`List<Something>` (Список чего-то).

Но что это за «что-то» (Something), которое мы используем во всех этих блоках `impl`? Так же, как и для `List`, мы хотим, чтобы наши
реализации работали со *всеми* типами `T`. Итак, как и в случае с `List`, давайте сделаем наши
`impl` «угловатыми»:


```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...и это всё!


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

Весь наш код теперь полностью обобщен для произвольных значений `T`. Черт возьми,
Rust — это *просто*. Я хотел бы выразить особую благодарность методу `new`, который даже
не изменился:

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

Грейтесь в лучах славы `Self`, хранителя рефакторинга и копипаст-программирования.
Также интересно, что мы не пишем `List<T>`, когда создаем экземпляр
списка. Эта часть выводится (inferred) за нас на основе того факта, что мы возвращаем ее
из функции, которая ожидает `List<T>`.

Хорошо, давайте перейдем к совершенно новому *поведению*!
