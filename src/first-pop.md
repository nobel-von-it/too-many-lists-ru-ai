# Извлечение (Pop)

Как и `push`, `pop` хочет изменять список. В отличие от `push`, мы на самом деле
хотим что-то вернуть. Но `pop` также должен иметь дело со сложным крайним
случаем: что если список пуст? Для представления этого случая мы используем надежный
тип `Option`:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>` — это перечисление (`enum`), которое представляет значение, которое может существовать. Оно может быть либо
`Some(T)`, либо `None`. Мы могли бы создать собственное перечисление для этого, как мы сделали для
`Link`, но мы хотим, чтобы наши пользователи могли понимать, что за чертовщину возвращает наш
тип, а `Option` настолько вездесущ, что его знают *все*. На самом деле, он настолько
фундаментален, что неявно импортируется в область видимости в каждом файле, так же
как и его варианты `Some` и `None` (так что нам не нужно писать `Option::None`).

Угловые скобки в `Option<T>` указывают на то, что `Option` на самом деле является *обобщенным (generic)* по
`T`. Это означает, что вы можете создать `Option` для *любого* типа!

Итак, у нас есть эта штука `Link`, как нам узнать, пуста ли она (`Empty`) или в ней есть
что-то еще (`More`)? Сопоставление с образцом (Pattern matching) с помощью `match`!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

Упс, `pop` должен возвращать значение, а мы этого еще не делаем. Мы *могли бы*
вернуть `None`, но в данном случае, вероятно, лучше вернуть
`unimplemented!()`, чтобы указать, что мы еще не закончили реализацию функции.
`unimplemented!()` — это макрос (`!` указывает на макрос), который вызывает панику (panic) программы,
когда мы до него доходим (~аварийно завершает ее контролируемым образом).

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

Безусловные паники — это пример [расходящейся функции (diverging function)][diverging].
Расходящиеся функции никогда не возвращают управление вызывающей стороне, поэтому их можно использовать в местах,
где ожидается значение любого типа. Здесь `unimplemented!()` используется
вместо значения типа `Option<T>`.

Обратите внимание также, что нам не нужно писать `return` в нашей программе. Последнее
выражение (по сути, строка) в функции неявно является ее возвращаемым значением. Это
позволяет нам выражать очень простые вещи немного более кратко. Вы всегда можете
явно выполнить возврат досрочно с помощью `return`, как в любом другом Си-подобном языке.

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

Да ладно тебе, Rust, отвяжись от нас! Как всегда, Rust чертовски зол на нас. К счастью,
на этот раз он также дает нам полную информацию! По умолчанию сопоставление с образцом пытается
переместить свое содержимое в новую ветку, но мы не можем этого сделать, потому что здесь мы
не владеем `self` по значению.

```text
help: consider borrowing here: `&self.head`
```

Rust говорит, что нам следует добавить ссылку в наш `match`, чтобы исправить это. 🤷‍♀️ Давайте попробуем:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

Ура, снова компилируется! Теперь давайте разберемся с этой логикой. Мы хотим создать
`Option`, так что давайте создадим для этого переменную. В случае `Empty` нам нужно вернуть
`None`. В случае `More` нам нужно вернуть `Some(i32)` и изменить заголовок (head)
списка. Итак, давайте попробуем сделать именно это?

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*головой*

*об стол*

Мы пытаемся переместить данные из `node`, когда все, что у нас есть, — это разделяемая ссылка на него.

Нам, вероятно, следует отступить и подумать о том, что мы пытаемся сделать. Мы хотим:

* Проверить, пуст ли список.
* Если он пуст, просто вернуть `None`
* Если он *не* пуст:
    * удалить заголовок (head) списка
    * забрать его `elem`
    * заменить заголовок списка на его `next`
    * вернуть `Some(elem)`

Ключевое понимание заключается в том, что мы хотим *удалить* вещи, а это значит, что мы хотим получить
заголовок списка *по значению*. Мы определенно не можем сделать это через разделяемую
ссылку, которую мы получаем через `&self.head`. У нас также есть «только» изменяемая ссылка
на `self`, поэтому единственный способ переместить вещи — это *заменить их*. Похоже, мы снова танцуем
танец `Empty`!

Давайте попробуем:


```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

О Б О Ж Е М О Й

Оно скомпилировалось без *единого* предупреждения!!!!!

На самом деле я собираюсь применить здесь свой личный линт (проверку): мы создали это значение `result`
для возврата, но на самом деле нам вообще не нужно было этого делать! Точно так же, как
функция вычисляется до своего последнего выражения, каждый блок также вычисляется до
своего последнего выражения. Обычно мы подавляем это поведение точкой с запятой,
что вместо этого заставляет блок вычисляться в пустой кортеж `()`. Это
фактически то значение, которое возвращают функции, не объявляющие возвращаемое значение (например, `push`).

Поэтому вместо этого мы можем написать `pop` следующим образом:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

Что является немного более кратким и идиоматичным. Обратите внимание, что ветка `Link::Empty`
полностью потеряла свои фигурные скобки, потому что нам нужно вычислить только одно
выражение. Просто приятное сокращение для простых случаев.

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

Отлично, все еще работает!



[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns
