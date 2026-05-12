# Связанный список, выделенный в стеке (The Stack-Allocated Linked List)

Эта книга в основном сосредоточена на связанных списках, *выделяемых в куче* (heap-allocated), потому что они наиболее распространены и практичны, но мы не *обязаны* использовать выделение памяти в куче. Выделение в куче приятно тем, что оно позволяет легко динамически выделять память. Выделение в стеке в этом отношении менее дружелюбно — такие вещи, как `alloca` в C, широко известны как Очень Проклятые И Проблемные (Very Cursed And Problematic).

Так что давайте выделим память в стеке простым способом: вызвав функцию и получив новый фрейм стека с дополнительным пространством! Это очень глупое решение нашей проблемы, но в то же время оно искренне практично и полезно. Это делается постоянно, потенциально даже без мысли о том, что это связанный список!

Каждый раз, когда вы делаете что-то рекурсивно, вы можете просто передать указатель на состояние текущего шага следующему шагу. Если этот указатель сам по себе является *частью* состояния, то вы создали связанный список, выделенный в стеке!

Теперь, конечно, мы находимся в *глупой* части книги, поэтому мы сделаем это глупым способом: сделаем связанный список главной звездой и заставим весь пользовательский код жить в болоте коллбэков (callbacks). Все ведь любят вложенные коллбэки!

Наш тип `List` будет просто узлом со ссылкой на другой узел:

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a List<'a, T>>,
}
```

И у него будет только одна операция — `push`, которая будет принимать старый список, состояние для текущего узла и коллбэк. Новый список будет создан в коллбэке. Мы также позволим коллбэкам возвращать любое значение, которое `push` вернет по завершении:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&List<'a, T>) -> U,
    ) -> U {
        let list = List { data, prev };
        callback(&list)
    }
}
```

Вот и всё! Мы можем использовать его так:

```rust ,ignore
List::push(None, 3, |list| {
    println!("{}", list.data);
    List::push(Some(list), 5, |list| {
        println!("{}", list.data);
        List::push(Some(list), 13, |list| {
            println!("{}", list.data);
        })
    })
})
```

Это прекрасно. 😿

Пользователь уже может обходить этот список, используя `while-let` для перехода по значениям `prev`, но просто забавы ради давайте реализуем итератор, который является вполне обычным:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev;
            &node.data
        })
    }
}
```

Давайте протестируем его:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn elegance() {
        List::push(None, 3, |list| {
            assert_eq!(list.iter().copied().sum::<i32>(), 3);
            List::push(Some(list), 5, |list| {
                assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
                List::push(Some(list), 13, |list| {
                    assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
                })
            })
        })
    }
}
```

```text
> cargo test

running 18 tests
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::basics ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test second::test::iter_mut ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test second::test::peek ... ok
test third::test::iter ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

Теперь на этом этапе вы можете спросить: «Эй, а могу ли я изменять данные, хранящиеся в узле?». Возможно! Давайте попробуем заставить список использовать изменяемые ссылки вместо разделяемых:


```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a mut List<'a, T>>,
}

pub struct Iter<'a, T> {
    next: Option<&'a List<'a, T>>,
}

impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a mut List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&mut List<'a, T>) -> U,
    ) -> U {
        let mut list = List { data, prev };
        callback(&mut list)
    }

    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev.as_ref().map(|prev| &**prev);
            &node.data
        })
    }
}

```


```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:47:32
   |
46 |  List::push(Some(list), 13, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
47 |      assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:45:28
   |
44 |  List::push(Some(list), 5, |list| {
   |                             ----
   |                             |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |      assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here


<ad infinitum>
```

Ой-ой. Похоже, ему не нравится наш итератор. Может, мы там что-то напутали? Давайте немного упростим тест, чтобы проверить:


```rust ,ignore
#[test]
fn elegance() {
    List::push(None, 3, |list| {
        assert_eq!(list.data, 3);
        List::push(Some(list), 5, |list| {
            assert_eq!(list.data, 5);
            List::push(Some(list), 13, |list| {
                assert_eq!(list.data, 13);
            })
        })
    })
}
```

```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:46:17
   |
44 |   List::push(Some(list), 5, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |       assert_eq!(list.data, 5);
46 | /     List::push(Some(list), 13, |list| {
47 | |         assert_eq!(list.data, 13);
48 | |     })
   | |______^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:44:13
   |
42 |   List::push(None, 3, |list| {
   |                        ----
   |                        |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
43 |       assert_eq!(list.data, 3);
44 | /     List::push(Some(list), 5, |list| {
45 | |         assert_eq!(list.data, 5);
46 | |         List::push(Some(list), 13, |list| {
47 | |             assert_eq!(list.data, 13);
48 | |         })
49 | |     })
   | |______________^ `list` escapes the closure body here
```

Хм, нет, это всё еще какая-то ерунда.

Проблема в том, что наш список случайно (😉) полагается на *вариантность* (variance). [Вариантность — это сложная тема](https://doc.rust-lang.org/nomicon/subtyping.html), но давайте посмотрим на нее в упрощенном виде:

Каждый список содержит ссылку на `List` с *точно таким же типом, как и он сам*. С точки зрения самого внутреннего списка это означает, что все списки используют то же время жизни (lifetime), что и он сам, но это *объективно* ложно: каждый узел в списке живет строго дольше следующего, потому что они буквально находятся во вложенных областях видимости!

Так... почему же код скомпилировался, когда мы использовали разделяемые ссылки? Потому что во многих случаях компилятор знает, что безопасно иметь что-то, что живет «слишком долго»! Когда мы запихиваем ссылку на список в следующий список, компилятор тихо «сжимает» времена жизни, чтобы они соответствовали тому, что ожидает новый список. Это сжатие времени жизни и есть *вариантность*.

Это точно такой же трюк, как в языках с наследованием, который позволяет вам передать `Cat` (Кошку) туда, где ожидается `Animal` (Животное — супертип `Cat`). Интуитивно мы понимаем, что нормально передать `Cat`, когда ожидается `Animal`, потому что Кошка — это просто Животное *и кое-что еще*. Ведь *нормально* на время забыть про часть «и кое-что еще», верно?

Аналогично, большее время жизни — это просто меньшее время жизни *и кое-что еще*. Так что здесь тоже вполне можно забыть про «и кое-что еще»!

Но вы, конечно, теперь спросите: тогда почему не работает версия с изменяемыми ссылками?!

Ну, вариантность *не всегда* безопасна. Если бы наш код *действительно* скомпилировался, мы могли бы написать use-after-free вот так:

```rust ,ignore
List::push(None, 3, |list| {
    List::push(Some(list), 5, |list| {
        List::push(Some(list), 13, |list| {
            // АХАХАХА все времена жизни одинаковы, поэтому компилятор
            // позволит мне перезаписать моего родителя, чтобы он содержал изменяемую ссылку на меня самого!
            // Я создам все use-after-free!!
            *list.prev.as_mut().unwrap().prev = Some(list);
        })
    })
})
```

Проблема с забыванием деталей заключается в том, что *кто-то другой может помнить эти детали и ожидать, что они останутся верными*. Это очень большая проблема, как только вы вводите *мутацию* (изменение). Если вы не будете осторожны, код, который не помнит про «и кое-что еще», от которого мы избавились, может подумать, что всё в порядке, и записать что-то туда, где «помнят» и *ожидают*, что это «и кое-что еще» всё еще там.

Если говорить в терминах наследования, этот код должен быть недопустимым:

```rust ,ignore
let mut my_kitty = Cat;                  // Создаем Кошку (длинное время жизни)
let animal: &mut Animal = &mut my_kitty; // Забываем, что это Кошка (укорачиваем время жизни)
*animal = Dog;                           // Записываем Собаку (короткое время жизни)
my_kitty.meow();                         // Мяукающая собака! (Use After Free)
```

Так что, хотя вы *можете* укоротить время жизни изменяемой ссылки, как только вы начинаете делать их *вложенными*, всё становится «инвариантным» (invariant), и вам больше не разрешается укорачивать времена жизни.

В частности, `&mut &'big mut T` не может быть преобразован в `&mut &'small mut T`, где `'big` больше, чем `'small`. Или более формально: `&'a mut T` ковариантен по `'a`, но инвариантен по `T`.

Интересный факт: Java на самом деле специально *позволяет* делать подобные вещи, но она [делает проверки во время выполнения, чтобы предотвратить появление мяукающих собак](https://docs.oracle.com/javase/7/docs/api/java/lang/ArrayStoreException.html).

----

Так что же мы можем сделать, чтобы мутировать данные? Использовать внутреннюю мутабельность (interior mutability)! Это позволяет нам сказать компилятору, что мы просто хотим иметь возможность мутировать *данные*, но не будем трогать ссылки.

Мы можем просто вернуться к предыдущей версии нашего кода с разделяемыми ссылками и использовать `Cell` в новом тесте:

```rust ,ignore
#[test]
fn cell() {
    use std::cell::Cell;

    List::push(None, Cell::new(3), |list| {
        List::push(Some(list), Cell::new(5), |list| {
            List::push(Some(list), Cell::new(13), |list| {
                // Умножаем каждое значение в списке на 10
                for val in list.iter() {
                    val.set(val.get() * 10)
                }

                let mut vals = list.iter();
                assert_eq!(vals.next().unwrap().get(), 130);
                assert_eq!(vals.next().unwrap().get(), 50);
                assert_eq!(vals.next().unwrap().get(), 30);
                assert_eq!(vals.next(), None);
                assert_eq!(vals.next(), None);
            })
        })
    })
}
```

```text
> cargo test

running 19 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter_mut ... ok
test fifth::test::iter ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test first::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fifth::test::miri_food ... ok
test silly2::test::cell ... ok
test third::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test third::test::basics ... ok
test second::test::iter ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

Просто как рекурсивный пирог! ✨
