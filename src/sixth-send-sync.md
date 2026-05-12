# Тесты Send, Sync и компиляции (Send, Sync, and Compile Tests)

Окей, на самом деле у нас есть еще одна пара трейтов, о которой стоит подумать, но они особенные. Нам предстоит иметь дело со Священной Римской Империей Rust: Небезопасными Встраиваемыми Трейтами, Требующими Явного Включения (Unsafe Opt-In Built-In Traits — OIBIT): [Send и Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html), которые на самом деле требуют явного отключения (opt-out) и встраиваются автоматически (1 из 3 — это довольно неплохо!).

Как и `Copy`, эти трейты не имеют абсолютно никакого связанного с ними кода и являются просто маркерами того, что ваш тип обладает определенным свойством. `Send` говорит о том, что ваш тип безопасно отправлять в другой поток. `Sync` говорит о том, что вашим типом безопасно делиться между потоками (`&Self: Send`).

Тот же аргумент в пользу ковариантности `LinkedList` применим и здесь: обычно нормальные коллекции, которые не используют хитрых трюков с внутренней изменяемостью (interior mutability), безопасно делать `Send` и `Sync`.

Но я сказал, что они требуют *явного отключения* (opt out). Так что, на самом деле, являемся ли мы ими уже сейчас? Как нам это узнать?

Давайте добавим немного новой магии в наш код: случайный приватный мусор, который не скомпилируется, если наши типы не обладают ожидаемыми свойствами:

```rust ,ignore
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    is_send::<Cursor<i32>>();
    is_sync::<Cursor<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> { x }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> { x }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> { x }
}
```

```text
cargo build
   Compiling linked-list v0.0.3 
error[E0277]: `NonNull<Node<i32>>` cannot be sent between threads safely
   --> src\lib.rs:433:5
    |
433 |     is_send::<LinkedList<i32>>();
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ `NonNull<Node<i32>>` cannot be sent between threads safely
    |
    = help: within `LinkedList<i32>`, the trait `Send` is not implemented for `NonNull<Node<i32>>`
    = note: required because it appears within the type `Option<NonNull<Node<i32>>>`
note: required because it appears within the type `LinkedList<i32>`
   --> src\lib.rs:8:12
    |
8   | pub struct LinkedList<T> {
    |            ^^^^^^^^^^
note: required by a bound in `is_send`
   --> src\lib.rs:430:19
    |
430 |     fn is_send<T: Send>() {}
    |                   ^^^^ required by this bound in `is_send`

<еще миллион ошибок>
```

О боже, в чем дело! У меня ведь была такая отличная шутка про Священную Римскую Империю!

Что ж, я солгал вам, когда сказал, что у сырых указателей есть только одна защитная функция: вот и вторая. `*const` И `*mut` явно отказываются от `Send` и `Sync` ради безопасности, поэтому нам *действительно* нужно включить их обратно вручную:

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

Обратите внимание, что мы должны писать здесь `unsafe impl`: это *небезопасные трейты*! Небезопасный код (например, библиотеки конкурентности) полагается на то, что мы реализуем эти трейты правильно! Поскольку здесь нет фактического кода, гарантия, которую мы даем, заключается лишь в том, что да, нас действительно безопасно отправлять (`Send`) или делиться (`Sync`) между потоками!

Не навешивайте их легкомысленно, но я, как Сертифицированный Профессионал, готов сказать: да, здесь всё совершенно нормально. Обратите внимание, что нам не нужно реализовывать `Send` и `Sync` для `IntoIter`: он просто содержит `LinkedList`, поэтому он автоматически выводит (auto-derives) `Send` и `Sync` — я же говорил вам, что они на самом деле отключаемые (opt out)! (Вы отключаетесь с помощью забавного синтаксиса `impl !Send for MyType {}`.)

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

Окей, отлично!

...Погодите, на самом деле было бы очень опасно, если бы вещи, которые *не должны* быть таковыми, ими оказывались. В частности, `IterMut` *определенно* не должен быть ковариантным, потому что он «подобен» `&mut T`. Но как мы можем это проверить?

С помощью Магии! Ну, на самом деле, с помощью `rustdoc`! Окей, нам не обязательно использовать `rustdoc` для этого, но это самый забавный способ. Видите ли, если вы пишете документационный комментарий и включаете в него блок кода, то `rustdoc` попытается скомпилировать и запустить его. Таким образом, мы можем использовать это для создания свежих анонимных «программ», которые не влияют на основную:


```rust ,ignore
    /// ```
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
error[E0308]: mismatched types
 --> src\lib.rs:461:86
  |
6 | fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
  |                                                                                      ^ lifetime mismatch
  |
  = note: expected struct `linked_list::IterMut<'_, &'a T>`
             found struct `linked_list::IterMut<'_, &'static T>`
```

Отлично, мы доказали, что он инвариантен, но эээ, теперь наши тесты фейлятся. Без паники, `rustdoc` позволяет сказать, что это ожидаемо, если аннотировать блок кодом `compile_fail`!

(На самом деле мы доказали только то, что он «не ковариантен», но, честно говоря, если вам удастся сделать тип «случайно и ошибочно контрвариантным», то... поздравляю?)

```rust ,ignore
    /// ```compile_fail
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src\lib.rs

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

Ура! Я рекомендую всегда сначала делать тест без `compile_fail`, чтобы вы могли убедиться, что он не компилируется *по правильной причине*. Например, этот тест также зафейлится (и, следовательно, пройдет проверку), если вы забудете написать `use`, а это не то, чего мы хотим! Хотя концептуально привлекательно иметь возможность «требовать» определенную ошибку от компилятора, это было бы абсолютным кошмаром, который фактически сделал бы любое улучшение сообщений об ошибках компилятора ломающим изменением. Мы хотим, чтобы компилятор становился лучше, так что нет, вы этого не получите.

(Ой, подождите, мы на самом деле можем указать код ошибки, который мы ожидаем, рядом с `compile_fail`, **но это работает только на nightly-версии и на это плохая идея полагаться по причинам, указанным выше. На стабильной версии это будет молча проигнорировано.**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...кстати, вы заметили ту часть, где мы действительно сделали `IterMut` инвариантным? Ее было легко пропустить, так как я «просто» скопировал `Iter` и вставил его в конец. Это последняя строка здесь:

```rust ,ignore
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

Давайте попробуем удалить этот `PhantomData`:

```text
 cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
error[E0392]: parameter `'a` is never used
  --> src\lib.rs:30:20
   |
30 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, or using a marker such as `PhantomData`
```

Ха! Компилятор прикрывает нашу спину и просто не позволит нам *не* использовать время жизни. Давайте попробуем вместо этого использовать *неправильный* пример:

```rust ,ignore
    _boo: PhantomData<&'a T>,
```

```text
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

Он собирается! Поймают ли наши тесты проблему теперь?

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
Test compiled successfully, but it's marked `compile_fail`.

failures:
    src\lib.rs - assert_properties::iter_mut_invariant (line 458)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s
```

Эээй!!! Система работает! Мне нравится иметь тесты, которые действительно делают свою работу, чтобы мне не приходилось так сильно ужасаться грядущих ошибок!
