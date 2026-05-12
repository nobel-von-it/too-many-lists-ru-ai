# Просмотр (Peeking)

Хорошо, мы справились с `push` и `pop`. Не буду лгать, это было
немного эмоционально. Корректность во время компиляции (Compile-time correctness) — та еще штука (hell of a drug).

Давайте остынем, сделав что-то простое: давайте просто реализуем `peek_front`.
Раньше это всегда было очень просто. Должно быть просто и сейчас, верно?

Верно?

На самом деле, я думаю, что могу просто скопипастить это!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

Подождите. Не в этот раз.

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // ЗАИМСТВУЕМ!!!!
        &node.borrow().elem
    })
}
```

ХАХ.

```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

Ладно, я просто сожгу свой компьютер.

Это точно такая же логика, как и в нашем односвязном стеке. Почему всё иначе? ПОЧЕМУ?

Ответ на самом деле и есть вся мораль этой главы: `RefCell` превращает всё
в грусть. До сих пор `RefCell` был просто помехой. Теперь он станет кошмаром.

Итак, что происходит? Чтобы понять это, нам нужно вернуться к определению
`borrow`:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

В разделе о структуре мы сказали:

> Вместо того чтобы обеспечивать соблюдение этих правил статически, `RefCell` делает это во время выполнения.
> Если вы нарушите правила, `RefCell` просто запаникует и аварийно завершит программу.
> Почему он возвращает эти штуки `Ref` и `RefMut`? Ну, они ведут себя в основном
> как `Rc`, но для заимствования. Они также удерживают `RefCell` заимствованным, пока не выйдут
> из области видимости. **Мы вернемся к этому позже.**

Это время пришло (It's later).

`Ref` и `RefMut` реализуют `Deref` и `DerefMut` соответственно. Так что для большинства
целей они ведут себя *точно так же*, как `&T` и `&mut T`. Однако
из-за того, как работают эти типажи, возвращаемая ссылка связана
со временем жизни `Ref`, а не самого `RefCell`. Это означает, что `Ref`
должен существовать до тех пор, пока мы сохраняем ссылку.

Это фактически необходимо для корректности. Когда `Ref` удаляется, он сообщает
`RefCell`, что он больше не заимствован. Поэтому, если бы нам *удалось* сохранить нашу
ссылку дольше, чем существовал `Ref`, мы могли бы получить `RefMut`, пока ссылка
еще существовала, и полностью сломать систему типов Rust пополам.

Так где же мы остались? Мы хотим вернуть только ссылку, но нам нужно
сохранить эту штуку `Ref`. Но как только мы возвращаем ссылку из
`peek`, функция завершается, и `Ref` выходит из области видимости.

😖

Насколько я знаю, здесь мы фактически зашли в тупик. Вы не можете
полностью инкапсулировать использование `RefCell` подобным образом.

Но... что, если мы просто откажемся от полного сокрытия деталей нашей реализации?
Что, если мы будем возвращать `Ref`?

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

Ой-ой (Blurp). Нужно импортировать кое-что.


```rust ,ignore
use std::cell::{Ref, RefCell};
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

Хм... верно. У нас есть `Ref<Node<T>>`, но нам нужен `Ref<T>`. Мы могли бы
оставить всякую надежду на инкапсуляцию и просто вернуть это. Мы также могли бы
усложнить ситуацию и обернуть `Ref<Node<T>>` в новый тип, чтобы предоставить
доступ только к `&T`.

Оба эти варианта *как-то* отстойны (lame).

Вместо этого мы пойдем еще глубже. Давайте
повеселимся (*fun*). Источником нашего веселья станет *этот зверь*:

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> Создает новый `Ref` для компонента заимствованных данных.

Да: точно так же, как вы можете применять `map` к `Option`, вы можете применять `map` к `Ref`.

Я уверен, что кто-то где-то действительно взволнован, потому что *монады* или что-то в этом роде, но
меня всё это не волнует. Кроме того, я не думаю, что это настоящая монада, так как
здесь нет случая, подобного `None`, но я отвлекся.

Это круто, и это всё, что для меня имеет значение. *Мне это нужно*.

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

Ооо дааа (Awww yissss)

Давайте убедимся, что это работает, подправив тест из нашего стека. Нам нужно
сделать кое-какие исправления, чтобы справиться с тем фактом, что `Ref` не реализуют сравнения.

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

Отлично!
