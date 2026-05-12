# Добавление (Push)

Итак, давайте напишем добавление значения в список. `push` *изменяет* список,
поэтому нам понадобится `&mut self`. Нам также нужно принять `i32` для добавления:

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

Перво-наперво нам нужно создать узел (node) для хранения нашего элемента:

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

Что идет в `next`? Ну, весь старый список! Можем ли мы... просто сделать это?

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

Не-а. Rust говорит нам правильные вещи, но определенно не очевидно,
что именно это значит и что с этим делать:

> cannot move out of borrowed content
> (нельзя переместить из заимствованного содержимого)

Мы пытаемся переместить поле `self.head` в `next`, но Rust не хочет,
чтобы мы это делали. Это оставило бы `self` только частично инициализированным, когда мы закончим
заимствование и «вернем его обратно» его законному владельцу. Как мы говорили ранее, это
*единственная* вещь, которую вы не можете сделать с `&mut`: это было бы супер-грубо,
и Rust очень вежлив (это также было бы невероятно опасно, но наверняка
не поэтому он заботится об этом).

Что если мы вернем что-то обратно? А именно, узел, который мы создаем:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

Без шансов (No dice). В принципе, это то, что Rust мог бы принять, но он
этого не сделает (по разным причинам — самая серьезная из которых — [безопасность исключений (exception safety)][]). Нам нужен
какой-то способ получить `head` так, чтобы Rust не заметил, что он пропал. За советом мы
обращаемся к печально известному хакеру на Rust Индиане Джонсу:

![Инди готовится к mem::replace](img/indy.gif)

Ах да, Инди предлагает маневр с `mem::replace`. Эта невероятно полезная
функция позволяет нам украсть значение из заимствования, *заменив* его другим
значением. Давайте просто подключим `std::mem` вверху файла, чтобы `mem` был в
локальной области видимости:

```rust ,ignore
use std::mem;
```

и используем его соответствующим образом:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

Здесь мы временно `заменяем (replace)` `self.head` на `Link::Empty` перед тем, как заменить его
новым заголовком (head) списка. Не буду врать: это довольно досадная
необходимость. К сожалению, мы вынуждены так поступить (пока что).

Но эй, с `push` покончено! Наверное. Честно говоря, нам следовало бы это протестировать.
Прямо сейчас самый простой способ сделать это — написать `pop` и убедиться,
что он выдает правильные результаты.


[exception safety]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html
