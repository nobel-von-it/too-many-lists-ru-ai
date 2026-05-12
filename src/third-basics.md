# Основы (Basics)

Мы уже знаем много основ Rust, поэтому можем снова сделать много простых
вещей.

Для конструктора мы можем снова просто скопипастить:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

`push` и `pop` больше не имеют особого смысла. Вместо этого мы можем предоставить
`prepend` (добавление в начало) и `tail` (хвост), которые делают примерно то же самое.

Давайте начнем с добавления в начало. Оно принимает список и элемент и возвращает
`List`. Как и в случае с изменяемым списком, мы хотим создать новый узел, у которого старый
список будет значением `next`. Единственное новшество заключается в том, как *получить* это следующее значение,
потому что нам не разрешено ничего изменять.

Ответом на наши молитвы является типаж `Clone`. `Clone` реализуется почти
каждым типом и предоставляет обобщенный способ получить «еще один такой же», который
логически не связан с оригиналом, имея лишь разделяемую ссылку. Это похоже на конструктор
копирования в C++, но он никогда не вызывается неявно.

`Rc`, в частности, использует `Clone` как способ увеличения счетчика ссылок. Таким образом,
вместо перемещения `Box` в подсписок, мы просто клонируем заголовок (head)
старого списка. Нам даже не нужно делать `match` по заголовку, потому что `Option` предоставляет
реализацию `Clone`, которая делает именно то, что нам нужно.

Итак, давайте попробуем:

```rust ,ignore
pub fn prepend(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

Вау, Rust действительно строг (hard-nosed) в отношении фактического использования полей. Он понимает, что ни один
потребитель никогда не сможет увидеть использование этих полей! Тем не менее, пока все идет хорошо.

`tail` — это логическая противоположность этой операции. Она принимает список и возвращает
весь список без первого элемента. Всё, что для этого нужно — клонировать *второй*
элемент в списке (если он существует). Давайте попробуем:

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

Хм, мы напортачили. `map` ожидает, что мы вернем `Y`, но здесь мы возвращаем
`Option<Y>`. К счастью, это еще один распространенный паттерн для `Option`, и мы можем просто
использовать `and_then`, чтобы иметь возможность вернуть `Option`.

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

Отлично.

Теперь, когда у нас есть `tail`, мы, вероятно, должны предоставить `head`, который возвращает
ссылку на первый элемент. Это просто `peek` из изменяемого списка:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```text
> cargo build

```

Приятно.

Этого функционала достаточно, чтобы мы могли его протестировать:


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.prepend(1).prepend(2).prepend(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Убеждаемся, что пустой хвост работает
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

Идеально!

`Iter` также идентичен тому, каким он был для нашего изменяемого списка:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
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

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().prepend(1).prepend(2).prepend(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

```

Кто вообще сказал, что динамическая типизация проще?

(глупцы сказали)

Обратите внимание, что мы не можем реализовать `IntoIter` или `IterMut` для этого типа. У нас есть только
разделяемый доступ к элементам.
