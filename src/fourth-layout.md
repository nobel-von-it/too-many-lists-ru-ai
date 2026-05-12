# Структура (Layout)

Ключом к нашему дизайну является тип `RefCell`. Сердцем
`RefCell` является пара методов:

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

Правила для `borrow` и `borrow_mut` точно такие же, как для `&` и `&mut`:
вы можете вызывать `borrow` столько раз, сколько захотите, но `borrow_mut` требует
монопольного доступа (exclusivity).

Вместо того чтобы обеспечивать соблюдение этих правил статически, `RefCell` делает это во время выполнения (at runtime).
Если вы нарушите правила, `RefCell` просто запаникует и аварийно завершит программу.
Почему он возвращает эти штуки `Ref` и `RefMut`? Ну, они ведут себя в основном
как `Rc`, но для заимствования. Они также удерживают `RefCell` заимствованным, пока не выйдут
из области видимости. Мы вернемся к этому позже.

Теперь с `Rc` и `RefCell` мы можем стать... невероятно многословным языком
с повсеместной изменяемостью и сборкой мусора, который не умеет собирать циклы! У-ррррааа...

Хорошо, мы хотим сделать список *двусвязным*. Это означает, что каждый узел имеет указатель на
предыдущий и следующий узлы. Кроме того, сам список имеет указатель на
первый и последний узлы. Это дает нам быструю вставку и удаление на *обоих*
концах списка.

Поэтому мы, вероятно, хотим что-то вроде этого:

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

Эй, оно скомпилировалось! Полно предупреждений о мертвом коде, но скомпилировалось! Давайте попробуем это использовать.
