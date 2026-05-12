# Скучная комбинаторика (Boring Combinatorics)

Итак, возвращаемся к нашим регулярным связанным спискам!

Сначала давайте разберемся с `Drop`, что тривиально сделать с помощью `pop`:

```rust ,ignore
impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Извлекаем, пока не придется остановиться
        while let Some(_) = self.pop_front() { }
    }
}
```

Нам нужно заполнить кучу действительно скучных комбинаторных реализаций, таких как `front`, `front_mut`, `back`, `back_mut`, `iter`, `iter_mut`, `into_iter` ...

Вы могли бы сделать их с помощью макросов или чего-то подобного, но, честно говоря, это худшая участь, чем копипаст. Мы просто собираемся сделать много копипаста. Я *очень тщательно* проработал предыдущие реализации `push`/`pop` так, чтобы мы могли *буквально* просто поменять местами `front` и `back`, и код делал бы/говорил то, что нужно! Ура болезненному опыту! (Так велик соблазн говорить о «prev и next» для узлов, но я считаю, что действительно стоит последовательно говорить о «front» и «back» как можно чаще, чтобы избежать ошибок.)

Итак, первый пошел, `front`:

```rust ,ignore
pub fn front(&self) -> Option<&T> {
    unsafe {
        self.front.map(|node| &(*node.as_ptr()).elem)
    }
}
```

Эй, вообще-то эта книга довольно старая, и с тех пор были добавлены некоторые приятные новые вещи, такие как оператор `?`, который делает ранний возврат при `Option::None`. Делает ли это наш код красивее?


```rust ,ignore
pub fn front(&self) -> Option<&T> {
    unsafe {
        Some(&(*self.front?.as_ptr()).elem)
    }
}
```

Может быть? Для чего-то столь простого это примерно то же самое, а предыдущий раздел был посвящен тому, как ранние возвраты немного пугают нас, так что, возможно, нам стоит предпочесть быть чуть более явными здесь (я придерживаюсь реализации с `map`). Перейдем к `front_mut`:

```rust ,ignore
pub fn front_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.front.map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

Я просто вывалю все версии для `back` в конце.

Далее — итераторы. В отличие от всех наших предыдущих списков, мы *наконец-то* разблокировали возможность сделать [DoubleEndedIterator](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html), и если мы стремимся к промышленному качеству, мы сделаем также и [ExactSizeIterator](https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html).

Поэтому в дополнение к `next` и `size_hint` мы собираемся поддерживать `next_back` и `len`.

Самые бдительные из вас могут заметить, что `IterMut` выглядит гораздо более подозрительно (sketchy) с двунаправленной итерацией, но на самом деле это всё еще корректно (sound)!

... боже, это будет куча шаблонного кода (boilerplate). Может, мне действительно стоит написать макрос... нет, нет, это всё же худшая участь.

```rust ,ignore
pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

impl<T> LinkedList<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }
}


impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    
    fn next(&mut self) -> Option<Self::Item> {
        // Хотя проверка условия `self.front == self.back` здесь заманчива,
        // она не сработает правильно для выдачи последнего элемента! Такого
        // рода вещи работают только для массивов из-за указателей «за конец».
        if self.len > 0 {
            // Мы могли бы вызвать unwrap для front, но это безопаснее и проще
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}
```

...это всего лишь `.iter()`...

мы вставим `IterMut` в конце, это буквально тот же самый код с `mut` во многих местах, давайте сначала разберемся с `into_iter`. К счастью, мы всё еще можем опереться на наше проверенное решение: просто обернуть нашу коллекцию и использовать `pop` для `next`:

```rust ,ignore
pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}


impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}
```

Всё еще куча шаблонного кода, но по крайней мере это *удовлетворяющий* шаблонный код.

Итак, вот весь наш код со всеми заполненными комбинаторными реализациями:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}

pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: это связанный список, чего вы хотите?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Помещаем новый передний узел перед старым
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // Если переднего узла нет, значит, список пуст, и нам нужно
                // установить также и задний узел.
                self.back = Some(new);
            }
            // Эти вещи происходят всегда!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn push_back(&mut self, elem: T) {
        // SAFETY: это связанный список, чего вы хотите?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                back: None,
                front: None,
                elem,
            })));
            if let Some(old) = self.back {
                // Помещаем новый задний узел перед старым
                (*old.as_ptr()).back = Some(new);
                (*new.as_ptr()).front = Some(old);
            } else {
                // Если заднего узла нет, значит, список пуст, и нам нужно
                // установить также и передний узел.
                self.front = Some(new);
            }
            // Эти вещи происходят всегда!
            self.back = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Нам нужно что-то делать только если есть передний узел для извлечения.
            self.front.map(|node| {
                // Возвращаем Box к жизни, чтобы мы могли забрать его значение и
                // дропнуть его (Box продолжает магическим образом понимать это за нас).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Делаем следующий узел новым передним узлом.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Очищаем его ссылку на удаленный узел
                    (*new.as_ptr()).front = None;
                } else {
                    // Если передний узел теперь null, то этот список теперь пуст!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box неявно освобождается здесь и знает, что там нет T.
            })
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        unsafe {
            // Нам нужно что-то делать только если есть задний узел для извлечения.
            self.back.map(|node| {
                // Возвращаем Box к жизни, чтобы мы могли забрать его значение и
                // дропнуть его (Box продолжает магическим образом понимать это за нас).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Делаем следующий узел новым задним узлом.
                self.back = boxed_node.front;
                if let Some(new) = self.back {
                    // Очищаем его ссылку на удаленный узел
                    (*new.as_ptr()).back = None;
                } else {
                    // Если задний узел теперь null, то этот список теперь пуст!
                    self.front = None;
                }

                self.len -= 1;
                result
                // Box неявно освобождается здесь и знает, что там нет T.
            })
        }
    }

    pub fn front(&self) -> Option<&T> {
        unsafe {
            self.front.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.front.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn back(&self) -> Option<&T> {
        unsafe {
            self.back.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn back_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.back.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Извлекаем, пока не придется остановиться
        while let Some(_) = self.pop_front() { }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        // Хотя проверка условия `self.front == self.back` здесь заманчива,
        // она не сработает правильно для выдачи последнего элемента! Такого
        // рода вещи работают только для массивов из-за указателей «за конец».
        if self.len > 0 {
            // Мы могли бы вызвать unwrap для front, но это безопаснее и проще
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type IntoIter = IterMut<'a, T>;
    type Item = &'a mut T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        // Хотя проверка условия `self.front == self.back` здесь заманчива,
        // она не сработает правильно для выдачи последнего элемента! Такого
        // рода вещи работают только для массивов из-за указателей «за конец».
        if self.len > 0 {
            // Мы могли бы вызвать unwrap для front, но это безопаснее и проще
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for IterMut<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}


#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Пробуем сломать пустой список
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Пробуем сломать список из одного элемента
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Дурачимся
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}