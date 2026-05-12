# Основы (Basics)

Ладно, это та часть книги, которая отстойна, и именно поэтому мне потребовалось 7 лет, чтобы написать эту главу! Время просто пробежаться по куче действительно скучных вещей, которые мы делали уже раз пять, но на этот раз всё будет экстра-многословно и длинно, потому что нам приходится делать всё дважды и с `Option<NonNull<Node<T>>>`!

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }
}
```

`PhantomData` — это странный тип без полей, поэтому вы просто создаете его, указывая имя типа. *пожимает плечами*

```rust ,ignore
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
            // установить также и задний узел. Также здесь есть несколько
            // проверок целостности для тестирования, на случай, если мы накосячим.
            debug_assert!(self.back.is_none());
            debug_assert!(self.front.is_none());
            debug_assert!(self.len == 0);
            self.back = Some(new);
        }
        self.front = Some(new);
        self.len += 1;
    }
}
```

```text
error[E0614]: type `NonNull<Node<T>>` cannot be dereferenced
  --> src\lib.rs:39:17
   |
39 |                 (*old).front = Some(new);
   |                 ^^^^^^
```


Ах да, я искренне ненавижу своих детишек-указателей. Нам нужно явно получить сырой указатель из `NonNull` с помощью `as_ptr`, потому что `DerefMut` определен через `&mut`, а мы не хотим случайно вводить безопасные ссылки в наш небезопасный код!


```rust ,ignore
            (*old.as_ptr()).front = Some(new);
            (*new.as_ptr()).back = Some(old);
```

```text
   Compiling linked-list v0.0.3
warning: field is never read: `elem`
  --> src\lib.rs:16:5
   |
16 |     elem: T,
   |     ^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default

warning: `linked-list` (lib) generated 1 warning (1 duplicate)
warning: `linked-list` (lib test) generated 1 warning
    Finished test [unoptimized + debuginfo] target(s) in 0.33s
```

Отлично, теперь займемся `pop` (и `len`):

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    unsafe {
        // Нам нужно что-то делать только если есть передний узел для извлечения.
        // Обратите внимание, что нам больше не нужно возиться с `take`,
        // потому что всё является Copy и нет деструкторов, которые запустятся,
        // если мы ошибемся... верно? :) Веееерно? :)))
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
                debug_assert!(self.len == 1);
                self.back = None;
            }

            self.len -= 1;
            result
            // Box неявно освобождается здесь и знает, что там нет T.
        })
    }
}

pub fn len(&self) -> usize {
    self.len
}
```

```text
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
```

Выглядит вполне законно, время написать тест!

```rust ,ignore
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
```


```text
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.40s
     Running unittests src\lib.rs

running 1 test
test test::test_basic_front ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Ура, мы идеальны!

...Верно же?