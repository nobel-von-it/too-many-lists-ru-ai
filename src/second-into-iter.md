# IntoIter

Итерация по коллекциям в Rust осуществляется с помощью типажа *Iterator*. Он немного сложнее,
чем `Drop`:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Новым элементом здесь является `type Item`. Это объявление того, что каждая
реализация `Iterator` имеет *ассоциированный тип (associated type)* под названием `Item`. В данном случае
это тип, который функция может выдать вам при вызове `next`.

Причина, по которой `Iterator` возвращает `Option<Self::Item>`, заключается в том, что интерфейс
объединяет понятия `has_next` (есть ли следующий) и `get_next` (получить следующий). Когда у вас есть следующее значение,
вы возвращаете
`Some(value)`, а когда его нет — `None`. Это делает
API в целом более эргономичным, безопасным в использовании и реализации, позволяя избежать
избыточных проверок и логики между `has_next` и `get_next`. Отлично!

К сожалению, в Rust (пока) нет ничего похожего на оператор `yield`, поэтому нам придется
реализовывать логику самостоятельно. Кроме того, на самом деле существует 3 различных вида
итераторов, которые должна стараться реализовать каждая коллекция:

* `IntoIter` — `T` (по значению)
* `IterMut` — `&mut T` (по изменяемой ссылке)
* `Iter` — `&T` (по разделяемой ссылке)

У нас уже есть все инструменты для реализации
`IntoIter` с использованием интерфейса `List`: просто вызывайте `pop` снова и снова. Поэтому мы
просто реализуем `IntoIter` как обертку нового типа (newtype wrapper) вокруг `List`:


```rust ,ignore
// Кортежные структуры (Tuple structs) — это альтернативная форма структур,
// полезная для создания простых оберток вокруг других типов.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // доступ к полям кортежной структуры осуществляется по номерам
        self.0.pop()
    }
}
```

И давайте напишем тест:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

Отлично!
