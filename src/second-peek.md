# Просмотр (Peek)

Одна вещь, которую мы даже не удосужились реализовать в прошлый раз, — это просмотр (peeking). Давайте
сделаем это. Все, что нам нужно сделать, — это вернуть ссылку на элемент в
заголовке списка, если он существует. Звучит просто, давайте попробуем:

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*Вздох*. Что теперь, Rust?

`map` принимает `self` по значению, что переместило бы `Option` из того места, где он находится.
Раньше это было нормально, потому что мы только что забрали (`take`) его, но теперь мы действительно
хотим оставить его на месте. *Правильный* способ справиться с этим — использовать метод
`as_ref` для `Option`, который имеет следующее определение:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

Он понижает `Option<T>` до Option со ссылкой на его внутренности. Мы могли бы
сделать это сами с помощью явного `match`, но *фу, нет*. Это означает, что нам
нужно сделать дополнительное разыменование, чтобы пробиться сквозь лишнюю косвенность, но,
к счастью, оператор `.` делает это за нас.


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

В точку.

Мы также можем создать *изменяемую* версию этого метода, используя `as_mut`:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
> cargo build

```

Изи (EZ).

Не забудьте протестировать это:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

Это мило, но мы ведь не совсем проверили, можем ли мы изменять возвращаемое значение `peek_mut`, не так ли? Если ссылка изменяемая, но никто ее не изменяет, действительно ли мы протестировали изменяемость? Давайте попробуем использовать `map` для этого `Option<&mut T>`, чтобы записать туда глубокомысленное значение:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

Компилятор жалуется, что `value` неизменяема, но мы довольно четко написали `&mut value`; в чем дело? Оказывается, такое написание аргумента замыкания не указывает на то, что `value` является изменяемой ссылкой. Вместо этого оно создает шаблон (pattern), который будет сопоставляться с аргументом замыкания; `|&mut value|` означает: «аргумент является изменяемой ссылкой, но просто скопируй значение, на которое он указывает, в `value`, пожалуйста». Если мы просто используем `|value|`, типом `value` будет `&mut i32`, и мы действительно сможем изменить заголовок:

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

Так гораздо лучше!
