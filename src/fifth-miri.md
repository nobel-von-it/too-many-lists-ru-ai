# Miri

*нервно смеется* Эти небезопасные штуки такие простые, я не знаю, почему все говорят обратное. Наша программа работает идеально.

> **РАССКАЗЧИК (NARRATOR):** 🙂

...верно?

> **РАССКАЗЧИК (NARRATOR):** 🙂

Ну, теперь мы пишем `unsafe` код, поэтому компилятор не может помочь нам отлавливать ошибки так же хорошо. Может быть, тестам просто *повезло* сработать, но на самом деле они делали что-то недетерминированное. Что-то из разряда Неопределенного Поведения (Undefined Behavioury).

Но что мы можем сделать? Мы приоткрыли окна и выскользнули из класса `rustc`. Теперь нам никто не может помочь.

...Подождите, кто это там за углом такой подозрительный?

*"Эй, парень, не хочешь проинтерпретировать немного кода на Rust?"*

Чт... нет? С чего бы,

*"Это безумие, чувак, он может проверить, что реальное динамическое выполнение твоей программы соответствует семантике модели памяти Rust. Мозг взрывает..."*

Что?

*"Он проверяет, не совершаешь ли ты Неопределенное Поведение."*

Думаю, я мог бы попробовать интерпретаторы всего *один разок*.

*"У тебя же установлен rustup, верно?"*

Конечно, это же *тот самый* инструмент для поддержания тулчейна Rust в актуальном состоянии!

```text
> rustup +nightly-2022-01-21 component add miri

info: syncing channel updates for 'nightly-2022-01-21-x86_64-pc-windows-msvc'
info: latest update on 2022-01-21, rust version 1.60.0-nightly (777bb86bc 2022-01-20)
info: downloading component 'cargo'
info: downloading component 'clippy'
info: downloading component 'rust-docs'
info: downloading component 'rust-std'
info: downloading component 'rustc'
info: downloading component 'rustfmt'
info: installing component 'cargo'
info: installing component 'clippy'
info: installing component 'rust-docs'
info: installing component 'rust-std'
info: installing component 'rustc'
info: installing component 'rustfmt'
info: downloading component 'miri'
info: installing component 'miri'
```

Что ты только что установил на мой компьютер!?

*"Годный стафф (The Good Stuff)"*

> **РАССКАЗЧИК (NARRATOR):** Тут происходят странные вещи с версиями тулчейнов:
>
> Инструмент, который мы устанавливаем, `miri`, тесно работает с внутренностями `rustc`,
> поэтому он доступен только для тулчейнов `nightly`.
>
> `+nightly-2022-01-21` сообщает `rustup`, что мы хотим установить `miri` с тулчейном Rust `nightly` за эту дату. Я указываю конкретную дату, потому что иногда
> `miri` отстает и не может быть собран для некоторых `nightly` версий. `rustup`
> автоматически скачает любой тулчейн, который мы укажем с помощью `+`, если он у нас еще не установлен.
>
> 2022-01-21 — это просто `nightly` версия, о которой я знаю, что она поддерживает `miri`, что вы можете проверить
> [на этой странице статуса](https://rust-lang.github.io/rustup-components-history/).
> Вы можете просто использовать `+nightly`, если чувствуете себя удачливым.
>
> Всякий раз, когда мы вызываем `miri` через `cargo miri`, мы также будем использовать этот синтаксис `+`, чтобы
> указать тулчейн, на который мы установили `miri`. Если вы не хотите указывать его каждый раз, вы можете использовать [`rustup override set`](https://rust-lang.github.io/rustup/overrides.html).

```text
> cargo +nightly-2022-01-21 miri test

I will run `"cargo.exe" "install" "xargo"` to install
a recent enough xargo. Proceed? [Y/n]
```

ЭЭЭ, ЧТО ЕЩЕ ЗА XARGO?

*"Всё нормально, не переживай из-за этого."*

```text
> y

    Updating crates.io index
  Installing xargo v0.3.24
...
    Finished release [optimized] target(s) in 10.65s
  Installing C:\Users\ninte\.cargo\bin\xargo-check.exe
  Installing C:\Users\ninte\.cargo\bin\xargo.exe
   Installed package `xargo v0.3.24` (executables `xargo-check.exe`, `xargo.exe`)

I will run `"rustup" "component" "add" "rust-src"` to install 
the `rust-src` component for the selected toolchain. Proceed? [Y/n]
```

ЭЭЭ???

*"Кто же не любит иметь копию исходного кода Rust?"*

```text
> y

info: downloading component 'rust-src'
info: installing component 'rust-src'
```

*"О да, всё готово, вот самая смачная часть."*

```text
   Compiling lists v0.1.0 (C:\Users\ninte\dev\tmp\lists)
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests (lists-5cc11d9ee5c3e924.exe)

error: Undefined Behavior: trying to reborrow for Unique at alloc84055, 
       but parent tag <209678> does not have an appropriate item in 
       the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |
    = help: this indicates a potential bug in the program: 
      it performed an invalid operation, but the rules it 
      violated are still experimental
    = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information

    = note: inside `std::option::Option::<std::boxed::Box<fifth::Node<i32>>>::map::<i32, [closure@src\fifth.rs:31:30: 40:10]>` at \lib\rustlib\src\rust\library\core\src\option.rs:846:18

note: inside `fifth::List::<i32>::pop` at src\fifth.rs:31:9
   --> src\fifth.rs:31:9
    |
31  | /         self.head.take().map(|head| {
32  | |             let head = *head;
33  | |             self.head = head.next;
34  | |
35  | ...   |
39  | |             head.elem
40  | |         })
    | |__________^
note: inside `fifth::test::basics` at src\fifth.rs:74:20
   --> src\fifth.rs:74:20
    |
74  |         assert_eq!(list.pop(), Some(1));
    |                    ^^^^^^^^^^
note: inside closure at src\fifth.rs:62:5
   --> src\fifth.rs:62:5
    |
61  |       #[test]
    |       ------- in this procedural macro expansion
62  | /     fn basics() {
63  | |         let mut list = List::new();
64  | |
65  | |         // Check empty list behaves right
66  | ...   |
96  | |         assert_eq!(list.pop(), None);
97  | |     }
    | |_____^
 ...
error: aborting due to previous error
```

Ого. Вот это ошибка так ошибка.

*"Да, взгляни на это дерьмо. Любо-дорого смотреть."*

Спасибо?

*"На, держи еще флакон эстрадиола, тебе это понадобится позже."*

Подожди, зачем?

*"Ты сейчас начнешь думать о моделях памяти, поверь мне."*

> **РАССКАЗЧИК (NARRATOR):** Таинственный незнакомец затем превратился в лису и шмыгнул в дыру в стене. Автор после этого несколько минут смотрел в пустоту, пытаясь переварить всё, что только что произошло.


-------

Таинственная лиса в переулке была права не только насчет моего пола: `miri` — это действительно Годный Стафф (The Good Shit).

Итак, что же такое [miri](https://github.com/rust-lang/miri)?

> Экспериментальный интерпретатор для среднеуровневого промежуточного представления (MIR) Rust. Он может запускать бинарные файлы и наборы тестов проектов cargo и обнаруживать определенные классы неопределенного поведения, например:
>
> * Доступ к памяти вне границ и использование после освобождения (use-after-free).
> * Некорректное использование неинициализированных данных.
> * Нарушение внутренних предусловий (достижение `unreachable_unchecked`, вызов `copy_nonoverlapping` с перекрывающимися диапазонами и т.д.).
> * Недостаточно выровненный доступ к памяти и ссылки.
> * Нарушение некоторых базовых инвариантов типов (например, `bool`, который не равен 0 или 1, или невалидный дискриминант перечисления).
> * Экспериментально: Нарушения правил Stacked Borrows, регулирующих алиасинг для ссылочных типов.
> * Экспериментально: Состояния гонки данных (но без эффектов слабой памяти).
>
> Кроме того, Miri также сообщит вам об утечках памяти: если в конце выполнения все еще есть выделенная память, и эта память недостижима из глобального статического контекста, Miri вызовет ошибку.
>
> ...
>
> Однако имейте в виду, что Miri не поймает все случаи неопределенного поведения в вашей программе и не может запустить все программы.

TL;DR: он интерпретирует вашу программу и замечает, если вы нарушаете правила *во время выполнения* и совершаете Неопределенное Поведение. Это необходимо, потому что Неопределенное Поведение — это *обычно* вещь, которая происходит во время выполнения. Если бы проблему можно было найти во время компиляции, компилятор просто сделал бы ее ошибкой!

Если вы знакомы с такими инструментами, как `ubsan` и `tsan`: это в основном то же самое, но всё вместе и в более экстремальном виде.

-------

`Miri` теперь висит за окном класса с ножом. С обучающим ножом.

Если мы когда-нибудь захотим, чтобы `miri` проверил нашу работу, мы можем попросить его проинтерпретировать наш набор тестов с помощью

```text
> cargo +nightly-2022-01-21 miri test
```

Теперь давайте поближе посмотрим на то, что он вырезал на нашей парте:

```text
error: Undefined Behavior: trying to reborrow for Unique at alloc84055, but parent tag <209678> does not have an appropriate item in the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |

    = help: this indicates a potential bug in the program: it 
      performed an invalid operation, but the rules it 
      violated are still experimental
    
    = help: see 
      https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information
```

Ну, я вижу, что мы сделали ошибку, но это сбивающее с толку сообщение об ошибке. Что такое "borrow stack" (стек заимствований)?

Мы попытаемся выяснить это в следующем разделе.