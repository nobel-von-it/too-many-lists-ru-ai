# Iter

Хорошо, давайте попробуем реализовать `Iter`. На этот раз мы не сможем полагаться на то, что
`List` предоставит нам все необходимые функции. Нам придется сделать все самим.
Основная логика, которую мы хотим реализовать, заключается в том, чтобы хранить указатель на текущий узел, который мы хотим выдать
следующим. Поскольку этого узла может не быть (список пуст или мы уже
закончили итерацию), мы хотим, чтобы эта ссылка была обернута в `Option`. Когда мы выдаем
элемент, мы хотим перейти к следующему узлу (`next`) текущего узла.

Итак, давайте попробуем:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

О боже. Времена жизни (Lifetimes). Я слышал о них. Говорят, это сущий кошмар.

Давайте попробуем что-то новенькое: видите эту штуку `error[E0106]`? Это код ошибки компилятора. Мы можем попросить `rustc` объяснить ее с помощью, ну, `--explain`:

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

Это, э-э... на самом деле мало что прояснило (эти документы предполагают, что мы понимаем
Rust лучше, чем сейчас). Но похоже, что нам следует добавить
эти штуки с `'a` в нашу структуру? Давайте попробуем.

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

Ладно, я начинаю видеть здесь закономерность... давайте просто добавим этих маленьких ребят
ко всему, к чему сможем:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

О боже. Мы сломали Rust.

Может быть, нам действительно стоит разобраться, что это за чертовщина — времена жизни `'a`
и что они вообще значат.

Времена жизни могут отпугнуть многих людей, потому что они меняют то, что мы знали и любили с самого рассвета программирования. До сих пор нам удавалось избегать времен жизни, хотя они были незримо вплетены во все наши программы.

Времена жизни не нужны в языках со сборщиком мусора, потому что сборщик мусора гарантирует, что все волшебным образом живет столько, сколько нужно. Большая часть данных в Rust управляется *вручную*, поэтому для этих данных нужно другое решение. C и C++ дают нам яркий пример того, что происходит, если просто позволить людям брать указатели на случайные данные в стеке: повсеместная неуправляемая небезопасность. Это можно грубо разделить на два класса ошибок:

* Хранение указателя на что-то, что вышло из области видимости.
* Хранение указателя на что-то, что было изменено или удалено.

Времена жизни решают обе эти проблемы, и в 99% случаев они делают это совершенно прозрачно.

Так что же такое время жизни?

Проще говоря, время жизни — это имя области (~блока/области видимости) кода где-то в программе. Вот и все. Когда ссылка помечена временем жизни, мы говорим, что она должна быть действительна для *всей* этой области. Различные вещи предъявляют требования к тому, как долго ссылка должна и может быть действительной. Вся система времен жизни, в свою очередь, является просто системой решения ограничений (constraint-solving system), которая пытается минимизировать область действия каждой ссылки. Если она успешно находит набор времен жизни, удовлетворяющий всем ограничениям, ваша программа компилируется! В противном случае вы получаете ошибку, говорящую о том, что что-то жило недостаточно долго.

В теле функции вы обычно не можете говорить о временах жизни, да и не захотели бы *в любом случае*. У компилятора есть вся информация, и он может вывести все ограничения, чтобы найти минимальные времена жизни. Однако на уровне типов и API компилятор *не* обладает всей информацией. Он требует, чтобы вы рассказали ему о взаимосвязи между различными временами жизни, чтобы он мог понять, что вы делаете.

В принципе, эти времена жизни *могли бы* быть опущены, но тогда проверка всех заимствований была бы огромным анализом всей программы, который порождал бы умопомрачительно нелокальные ошибки. Система Rust означает, что вся проверка заимствований может выполняться в теле каждой функции независимо, и все ваши ошибки должны быть достаточно локальными (или ваши типы имеют неверные сигнатуры).

Но мы ведь и раньше писали ссылки в сигнатурах функций, и все было в порядке! Это потому, что существуют определенные случаи, которые настолько распространены, что Rust автоматически выберет времена жизни за вас. Это *сокрытие времен жизни (lifetime elision)*.

В частности:

```rust ,ignore
// Только одна ссылка на входе, поэтому выходные данные должны быть получены из этих входных данных
fn foo(&A) -> &B; // синтаксический сахар для:
fn foo<'a>(&'a A) -> &'a B;

// Много входных данных, предполагаем, что они все независимы
fn foo(&A, &B, &C); // синтаксический сахар для:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Методы, предполагаем, что все выходные времена жизни получены из `self`
fn foo(&self, &B, &C) -> &D; // синтаксический сахар для:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

Так что же *означает* `fn foo<'a>(&'a A) -> &'a B`? На практике это означает лишь то, что входные данные должны жить как минимум столько же, сколько и выходные. Поэтому, если вы сохраняете выходные данные в течение длительного времени, это расширит область, в которой входные данные должны быть действительными. Как только вы перестанете использовать выходные данные, компилятор поймет, что входные данные тоже могут стать недействительными.

Благодаря этой системе Rust может гарантировать, что ничто не будет использовано после освобождения (use after free), и ничто не будет изменено, пока существуют активные ссылки. Он просто следит за тем, чтобы все ограничения выполнялись!

Хорошо. Итак. Iter.

Давайте вернемся к состоянию без времен жизни:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

Нам нужно добавить времена жизни только в сигнатуры функций и типов:

```rust ,ignore
// Iter обобщен по *некоторому* времени жизни, ему все равно
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// Здесь времени жизни нет, у List нет ассоциированных времен жизни
impl<T> List<T> {
    // Мы объявляем новое время жизни здесь для *конкретного* заимствования,
    // которое создает итератор. Теперь &self должен быть действителен столько же,
    // сколько существует Iter.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// Здесь у нас *есть* время жизни, потому что оно есть у Iter и нам нужно его определить
impl<'a, T> Iterator for Iter<'a, T> {
    // Оно нужно и здесь, это объявление типа
    type Item = &'a T;

    // Ничего из этого менять не нужно, все обрабатывается выше.
    // Self продолжает быть невероятно крутым и потрясающим
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

Хорошо, думаю, на этот раз мы справились.

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

ОК. ИТАК. Мы исправили ошибки времен жизни, но теперь получаем новые ошибки типов.

Мы хотим сохранять `&Node`, но получаем `&Box<Node>`. Что ж, это достаточно просто, нам просто нужно разыменовать `Box` перед тем, как брать ссылку:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

Мы забыли `as_ref`, поэтому мы перемещаем бокс в `map`, что означает, что он будет удален, а это значит, что наши ссылки повиснут в воздухе (dangling):

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref` добавил еще один слой косвенности, который нам нужно удалить:


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```text
cargo build

```

🎉 🎉 🎉

Функции `as_deref` и `as_deref_mut` стабильны начиная с Rust 1.40. До этого вам пришлось бы писать `map(|node| &**node)` и `map(|node| &mut**node)`. Вы можете подумать: «вау, эта штука `&**` выглядит костыльно», и вы будете правы, но, как хорошее вино, Rust со временем становится лучше, и нам больше не нужно так делать. Обычно Rust очень хорош в неявном выполнении такого рода преобразований с помощью процесса, называемого *приведением разыменования (deref coercion)*, где он, по сути, может вставлять `*` по всему вашему коду, чтобы он прошел проверку типов. Он может делать это, потому что у нас есть проверка заимствований (borrow checker), гарантирующая, что мы никогда не запутаемся в указателях!

Но в данном случае замыкание в сочетании с тем фактом, что у нас есть `Option<&T>` вместо `&T`, оказывается слишком сложным для компилятора, поэтому нам нужно помочь ему, указав типы явно. К счастью, по моему опыту, такое случается довольно редко.

Просто для полноты картины: мы *могли бы* дать компилятору *другую* подсказку с помощью *турбо-рыбы (turbofish)*:

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

Видите ли, `map` — это обобщенная функция:

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

Турбо-рыба, `::<>`, позволяет нам сказать компилятору, какими, по нашему мнению, должны быть типы этих обобщений. В данном случае `::<&Node<T>, _>` говорит: «она должна возвращать `&Node<T>`, а на другой тип мне все равно/я его не знаю».

Это, в свою очередь, дает компилятору понять, что к `&node` должно быть применено приведение разыменования, поэтому нам не нужно вручную расставлять все эти `*`!

Но в данном случае я не думаю, что это действительно улучшение. Это был просто тонко завуалированный повод похвастаться приведением разыменования и иногда полезной турбо-рыбой. 😅

Давайте напишем тест, чтобы убедиться, что мы ничего не сломали:

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

Чертовски круто.

Наконец, следует отметить, что мы *можем* применить сокрытие времен жизни и здесь:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

эквивалентно:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

Ура, меньше времен жизни!

Или, если вам неудобно «скрывать» то, что структура содержит время жизни, вы можете использовать синтаксис «явно опущенного времени жизни» из Rust 2018 — `'_`:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```
