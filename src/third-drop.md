# Удаление (Drop)

Как и в случае с изменяемыми списками, у нас есть проблема рекурсивного деструктора.
Надо признать, что для неизменяемого списка эта проблема не так остра: если мы когда-нибудь
достигнем другого узла, который является заголовком другого списка *где-то еще*, мы не
будем рекурсивно удалять его. Однако это все еще вещь, о которой нам следует заботиться, и
то, как с ней бороться, не так очевидно. Вот как мы решили эту проблему раньше:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

Проблема заключается в теле цикла:

```rust ,ignore
cur_link = boxed_node.next.take();
```

Здесь изменяется узел (`Node`) внутри `Box`, но мы не можем делать этого с `Rc`; он дает
нам только разделяемый доступ, потому что на него может указывать любое количество других `Rc`.

Но если мы знаем, что мы последний список, который знает об этом узле, то
было бы вполне нормально извлечь `Node` из `Rc`. Тогда мы также могли бы
понять, когда остановиться: всякий раз, когда мы *не можем* извлечь `Node`.

И посмотрите-ка, у `Rc` есть метод, который делает именно это: `try_unwrap`:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```text
cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ADesires/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Отлично!
Приятно.
