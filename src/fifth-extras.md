# Всякий лишний хлам (Extra Junk)

Теперь, когда `push` и `pop` написаны, всё остальное на самом деле точно такое же, как и в случае со стеком, как ни странно. Только операции, которые изменяют длину списка, должны затрагивать указатель на хвост (`tail`).

Но, конечно, теперь, когда всё стало небезопасными указателями, нам нужно переписать код для их использования! И раз уж мы собираемся трогать весь код, мы могли бы заодно
воспользоваться шансом и убедиться, что ничего не упустили.

Но в любом случае, давайте начнем копипастить код из реализации стека:

```rust ,ignore
// ...

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

`IntoIter` выглядит нормально, но `Iter` и `IterMut` нарушают наше простое правило никогда больше не использовать безопасные указатели в наших типах. Давайте перестрахуемся и изменим их для использования сырых указателей:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

Выглядит неплохо!

```text
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`
```

Выглядит не очень! Что это за [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html), о которой они толкуют?

> Тип нулевого размера, используемый для маркировки вещей, которые «ведут себя так», как будто они владеют `T`.
>
> Добавление поля `PhantomData<T>` в ваш тип сообщает компилятору, что ваш тип ведет себя так, как будто он хранит значение типа `T`, хотя на самом деле это не так. Эта информация используется при вычислении определенных свойств безопасности.
>
> Для более подробного объяснения того, как использовать `PhantomData<T>`, пожалуйста, обратитесь к [Nomicon](https://doc.rust-lang.org/nightly/nomicon/).

Эй, не спешите, мы читаем книгу, которую написал *Я*. А не ту другую книгу, которую наверняка написал какой-то жуткий *задрот (nerd)*! Готов поспорить, если они и пишут там какую-то структуру данных, то это что-то отстойное вроде стека на массиве (Array Stack), а *не* связанный список.

> Неиспользуемые параметры времени жизни
>
> Возможно, наиболее распространенным вариантом использования `PhantomData` является структура, которая имеет неиспользуемый параметр времени жизни, обычно как часть какого-то небезопасного кода.

А, так мы объявляем время жизни в нашем типе, но на самом деле его не используем. Мы *могли бы* пойти по пути `PhantomData`, но я хочу приберечь это для двусвязного списка в следующей главе, которому она *действительно* понадобится.

Мы находимся в интересной ситуации, когда нам на самом деле не нужна `PhantomData`. *Мне так кажется*. Я просто заявлю об этом и буду верить, что это правда, а если Miri накричит на нас в конце, я признаю свою неправоту, и мы сделаем штуку с `PhantomData`.

Что мы на самом деле сделаем, так это вернем ссылки обратно в эти типы итераторов и будем рады, что всё еще можем использовать ссылки в некоторых местах. Я думаю, это обоснованно, потому что при использовании итератора всё еще сохраняется своего рода правильная вложенность: вы создаете итератор, используете безопасные ссылки какое-то время, а затем уничтожаете итератор.

Только после того, как итератор исчезнет, вы сможете получить доступ к списку и вызывать такие вещи, как `push` и `pop`, которым нужно возиться с указателем на хвост и `Box`'ами. Теперь, во время итерации мы *собираемся* разыменовывать кучу сырых указателей, так что здесь есть своего рода смешивание, но мы должны быть в состоянии думать об этих ссылках как о перезаимствованиях небезопасных указателей.

*Я* даже сам не уверен на 100%, но я просто хочу попробовать и посмотреть!

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

Если мы собираемся хранить ссылки, нам нужно обновить наши сырые указатели до опций-ссылок (`Option<&T>`). Мы *могли бы* проверить, является ли указатель нулевым, но это один из тех невероятно редких случаев, когда, как мне кажется, можно использовать противные методы [ptr::as_ref](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) и [ptr::as_mut](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut).

Я *обычно* рекомендую избегать этих методов как чумы, потому что они делают некоторые удивительные и неприятные вещи и по своей сути снова вводят ссылки, когда всё мое «простое правило» состоит в том, чтобы избегать этого!

Эти методы поставляются с кучей предупреждений, но самое интересное вот это:

> Вы должны соблюдать правила алиасинга Rust, поскольку возвращаемое время жизни `'a` выбирается произвольно и не обязательно отражает фактическое время жизни данных. В частности, на протяжении этого времени жизни к памяти, на которую указывает указатель, нельзя обращаться (для чтения или записи) через любой другой указатель.

Эй, смотрите, это же та вещь, о которой мы говорили 25 страниц! Я уже утверждал, что здесь нам *определенно* будет нормально использовать ссылки, так что с алиасингом разобрались! Другая злая часть — это сигнатура:

```rust ,ignore
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

Вы видите, как это время жизни вообще не привязано к входным данным, потому что `self` передается по значению? Да, это то, что мы называем «несвязанным временем жизни» (unbounded lifetime), и это неприятная штука. Оно готово притвориться настолько большим, насколько мы его попросим, даже `'static`!" Способ *справиться* с этим — поместить его куда-то, что *связано*, что обычно просто означает «вернуть это из функции как можно скорее, чтобы сигнатура функции ограничила его».

Боже, я нервничаю из-за этого, но мы продолжим пробиваться! Давайте украдем несколько реализаций итераторов из стека:

```rust ,ignore
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}
```

Время момента истины...

```text
cargo test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

ДА!!! Выкуси, **РАССКАЗЧИК (NARRATOR)**! Иногда я не совершаю ошибок!

> **РАССКАЗЧИК (NARRATOR)**: но разве не в том был весь смысл, чтобы ошибки учили читателя.

ДА, НУ ТАК ВОТ, ИНОГДА УРОК ЗАКЛЮЧАЕТСЯ В ТОМ, ЧТО Я ПРАВ, И ВСЕ ДОЛЖНЫ СЛУШАТЬ МЕНЯ, КОГДА Я ГОВОРЮ ВЕЩИ О НЕБЕЗОПАСНОМ КОДЕ, ПОТОМУ ЧТО Я ПОТРАТИЛ СЛИШКОМ МНОГО ВРЕМЕНИ, ДУМАЯ О КОРРЕКТНОСТИ (SOUNDNESS) РЕАЛИЗАЦИЙ ИТЕРАТОРОВ?! ОКЕЙ?! ОКЕЙ.

В любом случае, вот `peek` и `peek_mut`.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref()
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut()
    }
}
```

Я даже не собираюсь их тестировать, потому что я больше никогда не совершаю ошибок.

> **РАССКАЗЧИК (NARRATOR)**: `cargo build`

```text
error[E0308]: mismatched types
  --> src\fifth.rs:66:13
   |
25 | impl<T> List<T> {
   |      - this type parameter
...
64 |     pub fn peek(&self) -> Option<&T> {
   |                           ---------- expected `Option<&T>` 
   |                                      because of return type
65 |         unsafe {
66 |             self.head.as_ref()
   |             ^^^^^^^^^^^^^^^^^^ expected type parameter `T`, 
   |                                found struct `fifth::Node`
   |
   = note: expected enum `Option<&T>`
              found enum `Option<&fifth::Node<T>>`

```

ЛАДНО.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

Похоже, я *продолжу* совершать ошибки, поэтому мы будем предельно осторожны и добавим новый тест, который я назову «еда для Miri» (miri food): что-то, что просто дурачится и смешивает наши API в кучу, чтобы помочь Miri поймать наши ошибки.

```rust ,ignore
#[test]
fn miri_food() {
    let mut list = List::new();

    list.push(1);
    list.push(2);
    list.push(3);

    assert!(list.pop() == Some(1));
    list.push(4);
    assert!(list.pop() == Some(2));
    list.push(5);

    assert!(list.peek() == Some(&3));
    list.push(6);
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&30));
    assert!(list.pop() == Some(30));

    for elem in list.iter_mut() {
        *elem *= 100;
    }

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&400));
    assert_eq!(iter.next(), Some(&500));
    assert_eq!(iter.next(), Some(&600));
    assert_eq!(iter.next(), None);
    assert_eq!(iter.next(), None);

    assert!(list.pop() == Some(400));
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&5000));
    list.push(7);

    // Бросаем его на землю и пусть деструктор поупражняется
}
```


```text
cargo test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out



MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Идеально.
