# Реализация курсоров (Implementing Cursors)

Окей, мы будем заморачиваться только с `CursorMut` из `std`, потому что неизменяемая версия на самом деле не интересна. Как и в моем оригинальном дизайне, здесь есть «призрачный» (ghost) элемент, который содержит `None` для обозначения начала/конца списка, и вы можете «перешагнуть через него», чтобы перейти на другую сторону списка. Чтобы реализовать его, нам понадобятся:

* Указатель на текущий узел
* Указатель на список
* Текущий индекс

Погодите, а каков индекс, когда мы указываем на «призрака»?

*хмурит брови* ... *проверяет std* ... *не нравится ответ std*

Окей, вполне разумно, что `index` у `Cursor` возвращает `Option<usize>`. Реализация в `std` делает кучу всякой ерунды, чтобы не хранить его как `Option`, но... мы же связанный список, так что всё нормально. Кроме того, в `std` есть штуки вроде `cursor_front`/`cursor_back`, которые запускают курсор на переднем/заднем элементах, что кажется интуитивным, но затем приходится делать что-то странное, когда список пуст.

Вы можете реализовать эти вещи, если хотите, но я собираюсь сократить всё это повторяющееся барахло и граничные случаи и просто сделать чистый метод `cursor_mut`, который начинает с призрака, а люди могут использовать `move_next`/`move_prev`, чтобы добраться до нужного элемента (а затем вы можете обернуть это в `cursor_front`, если очень захочется).

Давайте приступим:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

Довольно просто: по одному полю для каждого пункта нашего списка! Теперь метод `cursor_mut`:

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}
```

Поскольку мы начинаем с призрака, мы можем просто начать со всего равного `None`, мило и просто! Далее, движение:


```rust ,ignore
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // Мы находимся на реальном элементе, переходим к следующему (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // Мы только что перешли на призрака, индекса больше нет
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // Мы у призрака, и есть реальное начало списка, так что переходим к нему!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // Мы у призрака, но это единственный элемент... ничего не делаем.
        }
    }
}
```

Итак, здесь есть 4 интересных случая:

* Обычный случай
* Обычный случай, но мы доходим до призрака
* Случай с призраком, когда мы переходим в начало списка
* Случай с призраком, но список пуст, поэтому ничего не делаем

`move_prev` — это точно такая же логика, но с инвертированием `front`/`back` и инвертированием изменений индекса:

```rust ,ignore
pub fn move_prev(&mut self) {
    if let Some(cur) = self.cur {
        unsafe {
            // Мы находимся на реальном элементе, переходим к предыдущему (front)
            self.cur = (*cur.as_ptr()).front;
            if self.cur.is_some() {
                *self.index.as_mut().unwrap() -= 1;
            } else {
                // Мы только что перешли на призрака, индекса больше нет
                self.index = None;
            }
        }
    } else if !self.list.is_empty() {
        // Мы у призрака, и есть реальный конец списка, так что переходим к нему!
        self.cur = self.list.back;
        self.index = Some(self.list.len - 1)
    } else {
        // Мы у призрака, но это единственный элемент... ничего не делаем.
    }
}
```

Далее давайте добавим несколько методов для просмотра элементов вокруг курсора: `current`, `peek_next` и `peek_prev`. **Очень важное примечание:** эти методы должны заимствовать наш курсор по `&mut self`, и результаты должны быть привязаны к этому заимствованию. Мы не можем позволить пользователю получить несколько копий изменяемой ссылки, и мы не можем позволить ему использовать любые наши API вставки/удаления/разделения/склейки, пока он удерживает такую ссылку!

К счастью, это дефолтное предположение, которое делает Rust, когда вы используете элизиум времен жизни (lifetime elision), так что мы просто сделаем всё правильно по умолчанию!

```rust ,ignore
pub fn current(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

Голова пуста, методы `Option` и (опущенные) ошибки компилятора теперь думают за меня. Я скептически относился к теме с `Option<NonNull>`, но, черт возьми, это действительно позволяет мне писать этот код на автопилоте. Я провел слишком много времени, создавая коллекции на основе массивов, где никогда не получается использовать `Option` — вау, это так приятно! (`(*node.as_ptr())` по-прежнему выглядит жалко, но таковы уж сырые указатели в Rust...)

Далее у нас есть выбор: мы можем либо сразу перейти к разделению (split) и склейке (splice), в чем и заключается весь смысл этих API, либо мы можем сделать маленький шаг с вставкой/удалением одного элемента. У меня такое ощущение, что мы просто захотим реализовать вставку/удаление через разделение и склейку... так что давайте просто сделаем их первыми и посмотрим, как лягут карты (честно говоря, понятия не имею, пока пишу это).




# Разделение (Split)

Сначала `split_before` и `split_after`, которые возвращают всё, что находится до/после текущего элемента, в виде `LinkedList` (останавливаясь на элементе-призраке, если только вы не находитесь на призраке, и в этом случае мы просто возвращаем весь список, а курсор теперь указывает на пустой список):

*прищуривается* окей, здесь на самом деле какая-то нетривиальная логика, так что нам придется разбирать ее шаг за шагом.

Я вижу 4 потенциально интересных случая для `split_before`:

* Обычный случай
* Обычный случай, но `prev` — это призрак
* Случай с призраком, когда мы возвращаем весь список и становимся пустыми
* Случай с призраком, но список пуст, поэтому ничего не делаем и возвращаем пустой список

Давайте начнем с граничных случаев. Третий случай, я полагаю, это просто

```rust
mem::replace(self.list, LinkedList::new())
```

Верно? Мы становимся пустыми, возвращаем весь список, а наши поля уже были `None`, так что обновлять нечего. Мило. О, эй, это также делает правильные вещи и в четвертом случае!

Итак, теперь обычные случаи... окей, мне понадобятся ASCII-диаграммы для этого. В самом общем случае у нас есть что-то вроде этого:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

И мы хотим получить вот это:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

Так что нам нужно разорвать связь между `cur` и `prev`, и... боже, так много всего нужно изменить. Окей, мне просто нужно разбить это на шаги, чтобы убедить себя, что это имеет смысл. Это будет немного избыточно, но зато я смогу в этом разобраться:

```rust ,ignore
pub fn split_before(&mut self) -> LinkedList<T> {
    if let Some(cur) = self.cur {
        // Мы указываем на реальный элемент, поэтому список не пуст.
        unsafe {
            // Текущее состояние
            let old_len = self.list.len;
            let old_idx = self.index.unwrap();
            let prev = (*cur.as_ptr()).front;
            
            // Каким станет self
            let new_len = old_len - old_idx;
            let new_front = self.cur;
            let new_back = self.list.back;
            let new_idx = Some(0);

            // Каким станет результат
            let output_len = old_len - new_len;
            let output_front = self.list.front;
            let output_back = prev;

            // Разрываем связи между cur и prev
            if let Some(prev) = prev {
                (*cur.as_ptr()).front = None;
                (*prev.as_ptr()).back = None;
            }

            // Формируем результат:
            self.list.len = new_len;
            self.list.front = new_front;
            self.list.back = new_back;
            self.index = new_idx;

            LinkedList {
                front: output_front,
                back: output_back,
                len: output_len,
                _boo: PhantomData,
            }
        }
    } else {
        // Мы у призрака, просто заменяем наш список пустым.
        // Никакое другое состояние менять не нужно.
        std::mem::replace(self.list, LinkedList::new())
    }
}
```

Обратите внимание, что этот `if-let` обрабатывает ситуацию «обычный случай, но `prev` — это призрак»:

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
}
```

Если *вы* хотите, вы можете всё это сжать и применить такие оптимизации, как:

* свернуть два обращения к `(*cur.as_ptr()).front` в одно `(*cur.as_ptr()).front.take()`
* заметить, что `new_back` — это no-op (пустая операция), и просто удалить и то, и другое

Насколько я могу судить, всё остальное просто случайно делает правильные вещи. Посмотрим, когда напишем тесты! (копипастим, чтобы сделать `split_after`)

Я закончил Совершать Ошибки и собираюсь просто попытаться написать как можно более надежный и дуракоустойчивый код. Вот как я *на самом деле* пишу коллекции: просто разбиваю всё на тривиальные шаги и случаи, пока это не уложится в моей голове и не покажется надежным. Затем пишу тонну тестов, пока не убежусь, что мне всё-таки не удалось ничего испортить.

Поскольку большая часть работы с коллекциями, которую я делал, была *крайне небезопасной (extremely unsafe)*, я обычно не мог полагаться на то, что компилятор поймает ошибки, а Miri в те времена еще не существовало! Так что мне просто нужно было щуриться на проблему, пока не заболит голова, и изо всех сил стараться Никогда-Никогда-Никогда Не Ошибаться.

Не пишите небезопасный код на Rust (Unsafe Rust)! Безопасный Rust (Safe Rust) намного лучше!!!!




# Склейка (Splice)

Еще только один босс остался — `splice_before` и `splice_after`, которые, как я ожидаю, будут самыми изобилующими граничными случаями. Две эти функции *принимают* `LinkedList` и вживляют его содержимое в наш. Наш список может быть пустым, их список может быть пустым, нам приходится иметь дело с призраками... *вздох* давайте просто пойдем шаг за шагом со `splice_before`.

* Если их список пуст, нам ничего делать не нужно.
* Если наш список пуст, то наш список просто становится их списком.
* Если мы указываем на призрака, то это добавляет элементы в конец (изменяется `list.back`)
* Если мы указываем на первый элемент (0), то это добавляет элементы в начало (изменяется `list.front`)
* В общем случае мы делаем кучу всякой фигни с указателями.

Общий случай выглядит так:

```text
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

Превращается в это:

```text
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

Окей? Окей. Давайте запишем это... *ДЕЛАЕТ ГЛУБОКИЙ ВДОХ И ПУСКАЕТСЯ ВО ВСЕ ТЯЖКИЕ*:

```rust ,ignore
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Входной список пуст, ничего не делаем.
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // Мы добавляем в начало, см. добавление в конец
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Индекс сдвигается вперед на длину входного списка
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // Общий случай, никаких границ, просто внутренние исправления
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Индекс сдвигается вперед на длину входного списка
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // Мы на призраке, но список не пуст, добавляем в конец
                // Мы можем либо `take` указатели входного списка, либо `mem::forget`
                // его. Использование `take` более ответственно на случай, если мы сделаем кастомные
                // аллокаторы или что-то еще, что также нуждается в очистке!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Не обязательно, но вежливо сделать
                input.len = 0;
            } else {
                // Мы пусты, становимся входным списком, остаемся на призраке
                *self.list = input;
            }
        }
    }
```

Окей, этот вариант действительно ужасен, и теперь действительно чувствуется эта боль от `Option<NonNull>`. Но мы можем сделать много чисток. Во-первых, мы можем вынести этот код в самый конец, потому что мы всегда хотим его выполнять. Мне не очень нравится (хотя иногда это пустая операция, и установка `input.len` — это скорее вопрос паранойи по поводу будущих расширений кода):

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> Use of moved value: `input`

Ах, верно, в случае «мы пусты» мы перемещаем список. Давайте заменим это на swap:

```rust ,ignore
// Мы пусты, становимся входным списком, остаемся на призраке
std::mem::swap(self.list, &mut input);
```

В этом случае записи будут бессмысленными, но они всё равно сработают (мы также, вероятно, могли бы сделать ранний возврат в этой ветке, чтобы умилостивить компилятор).

Этот unwrap — просто следствие того, что я думал о случаях задом наперед, и это можно исправить, заставив `if-let` задать правильный вопрос:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

Корректировка индекса дублируется внутри веток, поэтому ее также можно вынести наружу:

```rust
*self.index.as_mut().unwrap() += input.len;
```

Окей, собрав всё это вместе, мы получаем следующее:

```rust
if input.is_empty() {
    // Входной список пуст, ничего не делаем.
} else if let Some(cur) = self.cur {
    // Оба списка не пусты
    if let Some(prev) = (*cur.as_ptr()).front {
        // Общий случай, никаких границ, просто внутренние исправления
        let in_front = input.front.take().unwrap();
        let in_back = input.back.take().unwrap();

        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // Мы добавляем в начало, см. добавление в конец ниже
        (*cur.as_ptr()).front = input.back.take();
        (*input.back.unwrap().as_ptr()).back = Some(cur);
        self.list.front = input.front.take();
    }
    // Индекс сдвигается вперед на длину входного списка
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // Мы на призраке, но список не пуст, добавляем в конец
    // Мы можем либо `take` указатели входного списка, либо `mem::forget`
    // его. Использование `take` более ответственно на случай, если мы сделаем кастомные
    // аллокаторы или что-то еще, что также нуждается в очистке!
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
    self.list.back = input.back.take();

} else {
    // Мы пусты, становимся входным списком, остаемся на призраке
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Не обязательно, но вежливо сделать
input.len = 0;

// Input дропается здесь
```

Ладно, это всё еще отстой, но в основном из-за — так, стоп, только что заметил баг:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

Мы делаем `take` для `input.front`, а затем вызываем `unwrap` для него же на следующей строке! *вздох* и мы делаем то же самое в эквивалентном зеркальном случае. Мы бы мгновенно поймали это в тестах, но мы ведь пытаемся быть Идеальными сейчас, и я просто делаю это вживую, и это именно тот момент, когда я это увидел. Вот что я получаю за то, что не веду себя как обычно занудно и не делаю всё поэтапно. Больше конкретики!

```rust
// Мы можем либо `take` указатели входного списка, либо `mem::forget`
// его. Использование `take` более ответственно на случай, если мы сделаем кастомные
// аллокаторы или что-то еще, что также нуждается в очистке!
if input.is_empty() {
    // Входной список пуст, ничего не делаем.
} else if let Some(cur) = self.cur {
    // Оба списка не пусты
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    if let Some(prev) = (*cur.as_ptr()).front {
        // Общий случай, никаких границ, просто внутренние исправления
        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // Нет prev, мы добавляем в начало
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
        self.list.front = Some(in_front);
    }
    // Индекс сдвигается вперед на длину входного списка
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // Мы на призраке, но список не пуст, добавляем в конец
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    (*back.as_ptr()).back = Some(in_front);
    (*in_front.as_ptr()).front = Some(back);
    self.list.back = Some(in_back);
} else {
    // Мы пусты, становимся входным списком, остаемся на призраке
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Не обязательно, но вежливо сделать
input.len = 0;

// Input дропается здесь
```

Ладно, вот это, вот это я могу терпеть. Единственные претензии, которые у меня есть — это то, что мы не дедуплицировали `in_front`/`in_back` (вероятно, мы могли бы перестроить наши условия, но да ладно). На самом деле это практически то же самое, что вы написали бы на C, но с барахлом в виде `Option<NonNull>`, делающим процесс утомительным. Я могу с этим жить. Ну нет, нам просто нужно сделать сырые указатели лучше для таких вещей. Но это выходит за рамки данной книги.

В любом случае, после этого я совершенно истощен, так что `insert`, `remove` и все остальные API можно оставить в качестве упражнения для читателя.

Вот финальный код для нашего `Cursor` с моей попыткой скопипастить комбинаторику. Всё ли я сделал правильно? Я узнаю об этом только тогда, когда напишу следующую главу и протестирую это чудовище!


```rust ,ignore
pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // Мы находимся на реальном элементе, переходим к следующему (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // Мы только что перешли на призрака, индекса больше нет
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // Мы у призрака, и есть реальное начало списка, так что переходим к нему!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // Мы у призрака, но это единственный элемент... ничего не делаем.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // Мы находимся на реальном элементе, переходим к предыдущему (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // Мы только что перешли на призрака, индекса больше нет
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // Мы у призрака, и есть реальный конец списка, так что переходим к нему!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // Мы у призрака, но это единственный элемент... ничего не делаем.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // У нас есть это:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        // 
        //
        // И мы хотим получить вот это:
        // 
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        // 
        //    return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // Мы указываем на реальный элемент, поэтому список не пуст.
            unsafe {
                // Текущее состояние
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;
                
                // Каким станет self
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // Каким станет результат
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Разрываем связи между cur и prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Формируем результат:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // Мы у призрака, просто заменяем наш список пустым.
            // Никакое другое состояние менять не нужно.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // У нас есть это:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // И мы хотим получить вот это:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //    return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // Мы указываем на реальный элемент, поэтому список не пуст.
            unsafe {
                // Текущее состояние
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;
                
                // Каким станет self
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // Каким станет результат
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Разрываем связи между cur и next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Формируем результат:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // Мы у призрака, просто заменяем наш список пустым.
            // Никакое другое состояние менять не нужно.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // У нас есть это:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Превращается в это:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // Мы можем либо `take` указатели входного списка, либо `mem::forget`
            // его. Использование `take` более ответственно на случай, если мы сделаем кастомные
            // аллокаторы или что-то еще, что также нуждается в очистке!
            if input.is_empty() {
                // Входной список пуст, ничего не делаем.
            } else if let Some(cur) = self.cur {
                // Оба списка не пусты
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // Общий случай, никаких границ, просто внутренние исправления
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // Нет prev, мы добавляем в начало
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Индекс сдвигается вперед на длину входного списка
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // Мы на призраке, но список не пуст, добавляем в конец
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // Мы пусты, становимся входным списком, остаемся на призраке
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Не обязательно, но вежливо сделать
            input.len = 0;
            
            // Input дропается здесь
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // У нас есть это:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Превращается в это:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // Мы можем либо `take` указатели входного списка, либо `mem::forget`
            // его. Использование `take` более ответственно на случай, если мы сделаем кастомные
            // аллокаторы или что-то еще, что также нуждается в очистке!
            if input.is_empty() {
                // Входной список пуст, ничего не делаем.
            } else if let Some(cur) = self.cur {
                // Оба списка не пусты
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // Общий случай, никаких границ, просто внутренние исправления
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // Нет next, мы добавляем в конец
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Индекс не меняется
            } else if let Some(front) = self.list.front {
                // Мы на призраке, но список не пуст, добавляем в начало
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // Мы пусты, становимся входным списком, остаемся на призраке
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Не обязательно, но вежливо сделать
            input.len = 0;
            
            // Input дропается здесь
        }        
    }
}
```