# Разрушение (Breaking Down)

Логика `pop_front` должна быть такой же, как и у `push_front`, но в обратном порядке. Давайте
попробуем:

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // нужно взять старую голову, убедившись, что она -2
    self.head.take().map(|old_head| {                         // -1 old
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new
                // список не пустеет
                new_head.borrow_mut().prev.take();            // -1 old
                self.head = Some(new_head);                   // +1 new
                // итого: -2 old, +0 new
            }
            None => {
                // список пустеет
                self.tail.take();                             // -1 old
                // итого: -2 old, (нового нет)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

Ой. *RefCell*. Полагаю, нужно снова вызвать `borrow_mut`…

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```

```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*вздох*

> cannot move out of borrowed content (невозможно переместить из заимствованного содержимого)

Хм... Кажется, `Box` нас *действительно* разбаловал. `borrow_mut` дает нам только
`&mut Node<T>`, но мы не можем переместить значение из него!

Нам нужно что-то, что принимает `RefCell<T>` и возвращает `T`. Давайте проверим
[документацию][refcell] на наличие чего-то подобного:

> `fn into_inner(self) -> T`
>
> Потребляет `RefCell`, возвращая обернутое значение.

Это выглядит многообещающе!

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

Ах, черт. `into_inner` хочет переместить `RefCell`, но мы не можем, потому что он
находится в `Rc`. Как мы видели в предыдущей главе, `Rc<T>` позволяет нам получать только разделяемые ссылки
на свои внутренности. Это имеет смысл, потому что в этом *вся суть*
указателей с подсчетом ссылок: они разделяемые!

Это было проблемой для нас, когда мы хотели реализовать `Drop` для нашего списка
с подсчетом ссылок, и решение здесь то же самое: `Rc::try_unwrap`, который извлекает
содержимое `Rc`, если его счетчик ссылок равен 1.

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap` возвращает `Result<T, Rc<T>>`. `Result` — это, по сути,
обобщенный `Option`, где с вариантом ошибки (`Err`) связаны данные. В
данном случае это `Rc`, который вы пытались развернуть. Поскольку нас не интересует случай,
когда это не удается (если мы правильно написали программу, это *должно* работать), мы
просто вызываем `unwrap` для него.

В любом случае, давайте посмотрим, какую ошибку компилятора мы получим следующей (давайте признаем, что она
обязательно будет).

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

УХ. `unwrap` для `Result` требует, чтобы вы могли вывести ошибку в отладочном виде (debug-print).
`RefCell<T>` реализует `Debug` только в том случае, если его реализует `T`. `Node` не реализует `Debug`.

Вместо этого давайте просто обойдем это ограничение, преобразовав `Result` в
`Option` с помощью `ok`:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

ПОЖАЛУЙСТА.

```text
cargo build

```

ДА.

*фух*

Мы сделали это.

Мы реализовали `push` и `pop`.

Давайте протестируем это, украв базовый тест `stack` (потому что это всё, что
мы реализовали на данный момент):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Проверяем, что пустой список ведет себя правильно
        assert_eq!(list.pop_front(), None);

        // Заполняем список
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Проверяем нормальное извлечение
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Добавляем еще немного, чтобы убедиться, что ничего не испортилось
        list.push_front(4);
        list.push_front(5);

        // Проверяем нормальное извлечение
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Проверяем исчерпание списка
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```

*В точку (Nailed it)*.

Теперь, когда мы можем правильно удалять элементы из списка, мы можем реализовать `Drop`.
`Drop` на этот раз немного интереснее с концептуальной точки зрения. В то время как
ранее мы беспокоились о реализации `Drop` для наших стеков только во избежание неограниченной
рекурсии, теперь нам нужно реализовать `Drop`, чтобы вообще хоть что-то происходило.

`Rc` не умеет работать с циклами. Если есть цикл, все элементы будут удерживать друг друга
в живых. Двусвязный список, как выясняется, — это просто большая цепь крошечных
циклов! Поэтому, когда мы удаляем наш список, у двух конечных узлов их счетчики ссылок
уменьшатся до 1... и больше ничего не произойдет. Что ж, если наш список
содержит ровно один узел, то все в порядке. Но в идеале список должен работать правильно,
если он содержит несколько элементов. Может, это только мое мнение.

Как мы видели, удаление элементов было немного болезненным. Поэтому самое простое, что мы можем
сделать, — это просто вызывать `pop`, пока не получим `None`:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(На самом деле мы могли бы сделать это и с нашими изменяемыми стеками, но короткие пути — для
людей, которые во всем разбираются!)

Мы могли бы рассмотреть реализацию версий `push` и `pop` с суффиксом `_back`, но
это просто работа по копированию и вставке, которую мы отложим на потом в этой главе. А пока
давайте посмотрим на более интересные вещи!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
