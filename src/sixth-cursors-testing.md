# Тестирование курсоров (Testing Cursors)

Пришло время узнать, сколько ужасно постыдных ошибок я совершил в предыдущем разделе!

О боже, мы сделали наш API непохожим ни на `std`, ни на старую реализацию. Ладно, я просто наспех слеплю что-нибудь из них обеих. Да, давайте «позаимствуем» эти тесты из `std`:

```rust ,ignore
    #[test]
    fn test_cursor_move_peek() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 1));
        assert_eq!(cursor.peek_next(), Some(&mut 2));
        assert_eq!(cursor.peek_prev(), None);
        assert_eq!(cursor.index(), Some(0));
        cursor.move_prev();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 2));
        assert_eq!(cursor.peek_next(), Some(&mut 3));
        assert_eq!(cursor.peek_prev(), Some(&mut 1));
        assert_eq!(cursor.index(), Some(1));

        let mut cursor = m.cursor_mut();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 6));
        assert_eq!(cursor.peek_next(), None);
        assert_eq!(cursor.peek_prev(), Some(&mut 5));
        assert_eq!(cursor.index(), Some(5));
        cursor.move_next();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 5));
        assert_eq!(cursor.peek_next(), Some(&mut 6));
        assert_eq!(cursor.peek_prev(), Some(&mut 4));
        assert_eq!(cursor.index(), Some(4));
    }

    #[test]
    fn test_cursor_mut_insert() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.splice_before(Some(7).into_iter().collect());
        cursor.splice_after(Some(8).into_iter().collect());
        // check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[7, 1, 8, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        cursor.splice_before(Some(9).into_iter().collect());
        cursor.splice_after(Some(10).into_iter().collect());
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[10, 7, 1, 8, 2, 3, 4, 5, 6, 9]);
        
        /* remove_current не реализован
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(7));
        cursor.move_prev();
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), Some(9));
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(10));
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[1, 8, 2, 3, 4, 5, 6]);
        */

        let mut cursor = m.cursor_mut();
        cursor.move_next();
        let mut p: LinkedList<u32> = LinkedList::new();
        p.extend([100, 101, 102, 103]);
        let mut q: LinkedList<u32> = LinkedList::new();
        q.extend([200, 201, 202, 203]);
        cursor.splice_after(p);
        cursor.splice_before(q);
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        let tmp = cursor.split_before();
        assert_eq!(m.into_iter().collect::<Vec<_>>(), &[]);
        m = tmp;
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        let tmp = cursor.split_after();
        assert_eq!(tmp.into_iter().collect::<Vec<_>>(), &[102, 103, 8, 2, 3, 4, 5, 6]);
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[200, 201, 202, 203, 1, 100, 101]);
    }

    fn check_links<T>(_list: &LinkedList<T>) {
        // было бы хорошо это сделать!
    }
```

Момент истины!

```text
cargo test

   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 1.03s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_debug ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_ord ... ok
test test::test_cursor_move_peek ... FAILED
test test::test_cursor_mut_insert ... FAILED
test test::test_iterator ... ok
test test::test_mut_iter ... ok
test test::test_eq ... ok
test test::test_rev_iter ... ok
test test::test_iterator_double_end ... ok
test test::test_hashmap ... ok
test test::test_ord_nan ... ok

failures:

---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `None`,
 right: `Some(1)`', src\lib.rs:1079:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1153:9


failures:
    test::test_cursor_move_peek
    test::test_cursor_mut_insert

test result: FAILED. 12 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Признаюсь, во мне взыграла гордыня, и я надеялся, что справился с этим с первого раза. Вот почему мы пишем тесты (но, возможно, я просто плохо портировал тесты..?).

В чем первая ошибка?

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 2, 3, 4, 5, 6]);
let mut cursor = m.cursor_mut();

cursor.move_next();
assert_eq!(cursor.current(), Some(&mut 1));
assert_eq!(cursor.peek_next(), Some(&mut 2));
assert_eq!(cursor.peek_prev(), None);
assert_eq!(cursor.index(), Some(0));

cursor.move_prev();
assert_eq!(cursor.current(), None);
assert_eq!(cursor.peek_next(), Some(&mut 1)); // УМИРАЕТ ЗДЕСЬ
```

Черт, я действительно испортил базовую функциональность. Погодите,

> Голова пуста, методы `Option` и (опущенные) ошибки компилятора теперь думают за меня.

Что ж, я как минимум честен.

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

...Да, это просто неправильно. Если `self.cur` равен `None`, мы не должны просто сдаваться, нам нужно проверить еще и `self.list.front`, потому что мы находимся на призраке! Так что нам просто нужно добавить `or_else` в цепочку:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .or_else(|| self.list.front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .or_else(|| self.list.back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

Это помогло?

```text
---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `Some(6)`,
 right: `None`', src\lib.rs:1078:9
```

Погодите, теперь всё неправильно *еще раньше*. Окей, мне нужно перестать писать `peek` с пустой головой, потому что, очевидно, это намного сложнее, чем я готов был признать. Попытка вслепую связать эти случаи в цепочку — это катастрофа, давайте сделаем нормальный `if` для случаев с призраком и без:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        let next = if let Some(cur) = self.cur {
            // Обычный случай, пытаемся перейти по указателю back текущего узла
            (*cur.as_ptr()).back
        } else {
            // Случай с призраком, пытаемся использовать указатель front списка
            self.list.front
        };

        // Выдаем элемент, если следующий узел существует
        next.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        let prev = if let Some(cur) = self.cur {
            // Обычный случай, пытаемся перейти по указателю front текущего узла
            (*cur.as_ptr()).front
        } else {
            // Случай с призраком, пытаемся использовать указатель back списка
            self.list.back
        };

        // Выдаем элемент, если предыдущий узел существует
        prev.map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

Чувствую уверенность в этом варианте!

```text
failures:

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1168:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    test::test_cursor_mut_insert

test result: FAILED. 13 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Даааа. Окей, осталась еще одна ошибка... ох.

Вы заметили ту часть, где я закомментировал код для тестирования `remove_current`? Да, я не обратил внимания на то, что этот тест сохраняет состояние (stateful). Давайте просто создадим новый список с состоянием, в котором нас оставила бы часть с `remove_current`:

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 8, 2, 3, 4, 5, 6]);
```

```text
 cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.70s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_cursor_move_peek ... ok
test test::test_eq ... ok
test test::test_cursor_mut_insert ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_ord_nan ... ok
test test::test_mut_iter ... ok
test test::test_hashmap ... ok
test test::test_debug ... ok
test test::test_ord ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

Хеееей, посмотрите на это... окей, теперь у меня паранойя. Давайте нормально заполним `check_links` и протестируем под Miri:

```rust ,ignore
fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
    let from_front: Vec<_> = list.iter().collect();
    let from_back: Vec<_> = list.iter().rev().collect();
    let re_reved: Vec<_> = from_back.into_iter().rev().collect();

    assert_eq!(from_front, re_reved);
}
```

Является ли это лучшим способом сделать это? Нет. Всё ли нормально? Да.

```text
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo miri test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic ... ok
test test::test_basic_front ... ok
test test::test_cursor_move_peek ... ok
test test::test_cursor_mut_insert ... ok
test test::test_debug ... ok
test test::test_eq ... ok
test test::test_hashmap ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_mut_iter ... ok
test test::test_ord ... ok
test test::test_ord_nan ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

ГОТОВО.

Готово.

Мы сделали это. Мы создали чертовски качественный (production-quality) `LinkedList` практически со всей той же функциональностью, что и в `std`. Не хватает ли нам мелких удобных методов тут и там? Абсолютно. Добавлю ли я их в финальную опубликованную версию крейта? Возможно!

Но я Так Сильно Устал.

Итак. Мы победили.

Погодите, черт. Мы же стремимся к промышленному качеству. Окей, последний финальный босс: `clippy`.

```text
cargo clippy

cargo clippy
    Checking linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:189:19
    |
189 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: `#[warn(clippy::redundant_pattern_matching)]` on by default
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: method `into_iter` can be confused for the standard trait method `std::iter::IntoIterator::into_iter`
   --> src\lib.rs:210:5
    |
210 | /     pub fn into_iter(self) -> IntoIter<T> {
211 | |         IntoIter {
212 | |             list: self
213 | |         }
214 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::should_implement_trait)]` on by default
    = help: consider implementing the trait `std::iter::IntoIterator` or choosing a less ambiguous method name
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#should_implement_trait

warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:228:19
    |
228 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: re-implementing `PartialEq::ne` is unnecessary
   --> src\lib.rs:275:5
    |
275 | /     fn ne(&self, other: &Self) -> bool {
276 | |         self.len() != other.len() || self.iter().ne(other)
277 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::partialeq_ne_impl)]` on by default
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#partialeq_ne_impl

warning: `linked-list` (lib) generated 4 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
```

Ладно, `clippy`, давай сделаем это.

Претензия 1 (и 3): мы используем `while let Some(_) = ` вместо `while .is_some()`. Цикл пустой, так что это действительно не имеет значения, но ладно, `clippy`, я сделаю по-твоему.

Претензия 2: У нас есть собственный метод `into_iter`. Погодите, что *проверяет std* окей, очко в пользу `clippy`. `IntoIterator` находится в прелюдии (и, по сути, является элементом языка), так что нам не нужна еще и собственная (inherent) версия.

Претензия 4: мы скопировали странный карго-культ из `std`. *пожимает плечами* ладно, я его удалю.

```text
cargo clippy
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

Отлично. Осталось сделать только одну вещь перед тем, как назвать это промышленным качеством: `fmt`.

```text
cargo fmt
```

...да, он добавил несколько пустых строк и удалил несколько висячих пробелов. Ничего интересного.

**ТЕПЕРЬ МЫ ДЕЙСТВИТЕЛЬНО ОКОНЧАТЕЛЬНО ЗАКОНЧИЛИ!!!!!!!!!!!!!!!!!!!!!**
