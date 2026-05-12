# Тестирование (Testing)

Итак, у нас написаны `push` и `pop`, теперь мы действительно можем протестировать
наш стек! Rust и Cargo поддерживают тестирование как первоклассную функцию, так что это
будет супер-просто. Все, что нам нужно сделать, — это написать функцию и пометить ее атрибутом
`#[test]`.

Как правило, в сообществе Rust мы стараемся держать тесты рядом с кодом, который они тестируют.
Однако мы обычно создаем новое пространство имен (namespace) для тестов, чтобы
избежать конфликтов с «реальным» кодом. Точно так же, как мы использовали `mod`, чтобы указать, что
`first.rs` должен быть включен в `lib.rs`, мы можем использовать `mod`, чтобы, по сути,
создать целый новый файл *внутри текущего (inline)*:

```rust ,ignore
// в first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

И мы вызываем его с помощью `cargo test`.

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

Ура, наш ничего не делающий тест прошел! Давайте заставим его делать хоть что-то. Мы сделаем это
с помощью макроса `assert_eq!`. В этом нет какой-то особой магии тестирования. Все, что он
делает, — это сравнивает две вещи, которые вы ему даете, и вызывает панику (panic) программы, если они не
совпадают. Да-да, вы сообщаете тестовой системе о неудаче, просто впадая в панику!

```rust ,ignore
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // Проверяем, что пустой список ведет себя правильно
        assert_eq!(list.pop(), None);

        // Заполняем список
        list.push(1);
        list.push(2);
        list.push(3);

        // Проверяем нормальное извлечение
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Добавляем еще немного, чтобы убедиться, что ничего не испортилось
        list.push(4);
        list.push(5);

        // Проверяем нормальное извлечение
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Проверяем исчерпание списка
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

Упс! Поскольку мы создали новый модуль, нам нужно явно импортировать `List`, чтобы использовать
его.

```rust ,ignore
mod test {
    use super::List;
    // все остальное без изменений
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

Ура!

Но что не так с этим предупреждением...? Мы явно используем `List` в нашем тесте!

...но только при тестировании! Чтобы успокоить компилятор (и быть вежливыми по отношению к нашим
пользователям), мы должны указать, что весь модуль `test` должен компилироваться только
в том случае, если мы запускаем тесты.


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // все остальное без изменений
}
```

И это все, что касается тестирования!
