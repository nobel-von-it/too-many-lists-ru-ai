# Итерация (Iteration)

Давайте попробуем итерировать этого плохиша.

## IntoIter

`IntoIter`, как всегда, будет самым простым. Просто оберните стек и
вызывайте `pop`:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop_front()
    }
}
```

Но у нас есть интересное новое развитие событий. Если раньше для наших списков
существовал только один «естественный» порядок итерации, то дека по своей сути
двунаправленная. Что такого особенного в порядке «от начала к концу»? Что, если кто-то захочет
итерироваться в другом направлении?

У Rust на самом деле есть ответ на это: `DoubleEndedIterator`. `DoubleEndedIterator`
*наследуется* от `Iterator` (что означает, что все `DoubleEndedIterator` являются итераторами) и
требует одного нового метода: `next_back`. Он имеет точно такую же сигнатуру, как и
`next`, но предполагается, что он будет возвращать элементы с другого конца. Семантика
`DoubleEndedIterator` очень удобна для нас: итератор становится
декой. Вы можете потреблять элементы с начала и с конца, пока оба конца не
сойдутся, после чего итератор станет пустым.

Как и в случае с `Iterator` и `next`, оказывается, что `next_back` не так уж
сильно волнует потребителей `DoubleEndedIterator`. Скорее, лучшая часть этого интерфейса заключается в том, что он предоставляет метод `rev`, который оборачивает
итератор, создавая новый, который возвращает элементы в обратном порядке.
Семантика этого довольно проста: вызовы `next` для
обратного итератора — это просто вызовы `next_back`.

В любом случае, поскольку мы уже являемся декой, предоставить этот API довольно просто:

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

И давайте протестируем это:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```


```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::iter ... ok
test third::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

Приятно.

## Iter

`Iter` будет чуть менее снисходительным. Нам снова придется иметь дело с этими ужасными
штуками `Ref`! Из-за `Ref` мы не можем хранить `&Node`, как делали раньше.
Вместо этого давайте попробуем хранить `Ref<Node>`:

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```text
> cargo build

```

Пока всё идет хорошо. Реализация `next` будет немного волосатой (hairy), но я думаю,
что здесь используется та же базовая логика, что и в старом `IterMut` для стека, но с дополнительным
безумием `RefCell`:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```text
cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
154 |         |--------- lifetime `'1` appears in the type of `self`
155 |         self.0.take().map(|node_ref| {
156 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
157 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

Черт.

`node_ref` живет недостаточно долго. В отличие от обычных ссылок, Rust не позволяет
нам просто так разделять `Ref`. `Ref`, который мы получаем из `head.borrow()`, может
жить только столько же, сколько `node_ref`, но мы в итоге уничтожаем его в нашем
вызове `Ref::map`.

Функция, которая нам нужна, существует, и она называется *[map_split][]*:

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

Уф (Woof). Давайте попробуем...

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = next.as_ref().map(|head| head.borrow());

        elem
    })
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

Эргх. Нам нужно снова использовать `Ref::map`, чтобы правильно настроить времена жизни. Но `Ref::map`
возвращает `Ref`, а нам нужен `Option<Ref>`, но нам нужно пройти через
`Ref`, чтобы применить `map` к нашему `Option`…

**долго смотрит вдаль**

??????

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = if next.is_some() {
            Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
        } else {
            None
        };

        elem
    })
}
```

```text
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

Ой. Точно. Здесь несколько `RefCell`. Чем глубже мы погружаемся в список, тем больше
уровней вложенности под каждым `RefCell` мы получаем. Нам нужно было бы поддерживать что-то вроде стека
`Ref`, чтобы представлять все выданные займы, которые мы удерживаем, потому что если мы перестаем
смотреть на элемент, нам нужно уменьшить счетчик заимствований на каждом `RefCell`, который
идет перед ним.....................

Я не думаю, что мы можем что-то здесь сделать. Это тупик. Давайте попробуем
выбраться из `RefCell`.

А что насчет наших `Rc`? Кто сказал, что нам вообще нужно хранить ссылки?
Почему бы нам просто не клонировать весь `Rc`, чтобы получить хороший владеющий дескриптор в середине
списка?

```rust ,ignore
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item =
```

Эээ... Подождите, а что мы теперь возвращаем? `&T`? `Ref<T>`?

Нет, ни один из этих вариантов не работает... наш `Iter` больше не имеет времени жизни! И `&T`,
и `Ref<T>` требуют от нас объявить какое-то время жизни заранее, прежде чем мы перейдем к
`next`. Но всё, что нам удастся извлечь из нашего `Rc`, будет заимствованием
итератора... мозг... болит... аааааааахххххх

Может быть, мы можем... отобразить (map)... `Rc`..., чтобы получить `Rc<T>`? Такое возможно? В документации `Rc`, похоже, нет ничего подобного. На самом деле кто-то создал [крейт own-ref][own-ref],
который позволяет это делать.

Но подождите, даже если мы сделаем *это*, мы столкнемся с еще большей проблемой:
ужасным призраком инвалидации итератора (iterator invalidation). Ранее мы были полностью застрахованы
от инвалидации итераторов, потому что `Iter` заимствовал список, оставляя его полностью
неизменяемым. Однако если бы наш `Iter` возвращал `Rc`, они бы вообще не заимствовали список!
Это означает, что люди могут начать вызывать `push` и `pop` для списка, пока
они удерживают указатели на него!

О боже, к чему это приведет?!

Ну, добавление (`push`), на самом деле, это нормально. У нас есть представление о некотором поддиапазоне
списка, и список просто вырастет за пределы нашего поля зрения. Ничего страшного.

Однако `pop` — это совсем другая история. Если они извлекают элементы за пределами нашего
диапазона, все *по-прежнему* должно быть в порядке. Мы не видим этих узлов, так что ничего не
произойдет. Однако если они попытаются извлечь узел, на который мы указываем... всё взорвется! В частности, когда они попытаются вызвать `unwrap` для результата
`try_unwrap`, это фактически завершится ошибкой, и вся программа запаникует.

Это на самом деле довольно круто. Мы можем получить кучу внутренних владеющих указателей на
список и одновременно изменять его, *и это будет просто работать*, пока они
не попытаются удалить узлы, на которые мы указываем. И даже тогда мы не получим
висячих указателей или чего-то подобного, программа детерминированно запаникует!

Но необходимость иметь дело с инвалидацией итераторов вдобавок к маппингу `Rc` кажется...
плохой идеей. `Rc<RefCell>` действительно, по-настоящему окончательно подвел нас. Интересно,
что мы столкнулись с инверсией случая персистентного стека. Если
персистентный стек изо всех сил пытался вернуть владение данными, но мог получать
ссылки целыми днями, то наш список без проблем получал владение, но
действительно изо всех сил пытался выдать ссылки.

Хотя, справедливости ради, большинство наших трудностей было связано с желанием скрыть
детали реализации и иметь приличный API. Мы *могли бы* сделать всё нормально,
если бы хотели просто передавать `Node` повсюду.

Черт возьми, мы могли бы создать несколько параллельных `IterMut`, которые проверялись бы во время выполнения на предмет того, чтобы
не иметь изменяемого доступа к одному и тому же элементу!

На самом деле, такой дизайн больше подходит для внутренней структуры данных, которая
никогда не выходит наружу к потребителям API. Внутренняя изменяемость отлично подходит для
написания безопасных *приложений*. Не так хорошо для безопасных *библиотек*.

В общем, на этом я сдаюсь в попытках реализовать `Iter` и `IterMut`. Мы могли бы их сделать, но *тьфу (ugh)*.


[own-ref]: https://crates.io/crates/owning_ref
[map-split]: https://doc.rust-lang.org/std/cell/struct.Ref.html#method.map_split
