+++
date = "2025-03-12T16:35:41+04:00"
draft = false
title = "Шаг 1: Lexing"
+++

![](/LexCover.png)

Это первая статья из серии про процесс компиляции, в которой каждый этап описывается по отдельности. Перед прочтением советую прочитать предыдущую обзорную статью, описывающую весь процесс в общих чертах, которую можно [найти здесь](https://georgiyozhegov.github.io/blog/%D0%BF%D1%83%D1%82%D1%8C-%D0%BE%D1%82-%D0%BA%D0%BE%D0%B4%D0%B0-%D0%B4%D0%BE-%D0%B1%D0%B8%D0%BD%D0%B0%D1%80%D0%BD%D0%BE%D0%B3%D0%BE-%D1%84%D0%B0%D0%B9%D0%BB%D0%B0/).

Относительно других этапов, лексический анализ -- самый простой, хотя и может показаться сложным на первый взгляд, поэтому эта статья будет самой короткой из серии.

Весь код взят из [этого репозитория](https://github.com/georgiyozhegov/mellow/tree/master/lib/syntax/src/lex) и упрощен, для лучшего понимания.

# Код

**_source.txt_**

Для примера возьмем небольшой фрагмент кода на вымышленном языке программирования.

```
let a = 10 + 2
if a > 8 then
    debug "A больше 8"
else
    debug "А либо меньше, либо равно 8"
end
```

**_src/main.rs_**

Читаем исходный код из файла и создаём итератор по символам. Метод `.peekable()` даёт возможность "заглядывать" на один символ вперёд.

```rust
fn main() {
    let source = fs::read_to_string("source.txt").unwrap();
    let source = source.chars().peekable();
}
```

Определим тип токена.

```rust
#[derive(Debug)]
pub enum Token {
    Integer(i64),
    Identifier(String),
    String(String),
    Equal,
    Plus,
    Greater,
    Let,
    If,
    Then,
    Else,
    End,
    Debug,
}
```

Создадим структуру, которая будет содержать всю логику лексера. `Source` -- символы, которые подаются на вход.

```rust
type Source<'s> = Peekable<Chars<'s>>;

struct Lex<'l> {
    source: Source<'l>,
}

impl<'l> Lex<'l> {
    pub fn new(source: Source<'l>) -> Self {
        Self { source }
    }
}
```

Для улучшения читаемости кода определим макросы различных категорий символов.

```rust
macro_rules! numeric {
    () => {
        '0'..='9' // паттерн чисел
    };
}

macro_rules! alphabetic {
    () => {
        'a'..='z' | 'A'..='Z' | '_' // паттерн идентификаторов
    };
}

macro_rules! skip {
    () => {
        ' ' | '\t' | '\n' // паттерн пробела и других пропускаемых символов
    };
}
```

Реализуем `Iterator` для лексера, чтобы можно было обрабатывать выходные токены в виде потока.

```rust
impl Iterator for Lex<'_> {
    type Item = Token;

    fn next(&mut self) -> Option<Self::Item> {
        match self.source.peek()? {
            numeric!() => Some(self.numeric()),
            alphabetic!() => Some(self.alphabetic()),
            '"' => Some(self.string()),
            skip!() => self.skip(),
            _ => Some(self.single()),
        }
    }
}
```

Здесь мы проверяем следующий символ и определяем какой токен будем обрабатывать. Например, если следующий символ -- `"`, выходным токеном будет `String`, то есть строка.

Добавим вывод токенов при итерации по `Lex`.

```rust
fn main() {
    let source = fs::read_to_string("source.txt").unwrap();
    let source = source.chars().peekable();
    for token in Lex::new(source) {
        println!("{token:?}");
    }
}
```

Теперь напишем методы для сбора токенов из символов.

**Функция для целых чисел.**

```rust
impl Lex<'_> {
    fn numeric(&mut self) -> Token {
        let buffer = self.take_while(|c| matches!(c, numeric!()));
        Token::Integer(buffer.parse().unwrap())
    }
```

В коде используется функция `take_while`, которая собирает в буфер все символы, которые попадают под правило, передаваемое как аргумент.

```rust
impl Lex<'_> {
    fn take_while(&mut self, predicate: fn(&char) -> bool) -> String {
        let mut output = String::new();
        while let Some(c) = self.source.peek() {
            if !predicate(c) {
                break;
            }
            output.push(*c);
            self.source.next();
        }
        output
    }
}
```

**Функция для идентификаторов и ключевых слов.**

```rust
    fn keyword(buffer: &str) -> Option<Token> {
        match buffer {
            "let" => Some(Token::Let),
            "if" => Some(Token::If),
            "then" => Some(Token::Then),
            "else" => Some(Token::Else),
            "end" => Some(Token::End),
            "debug" => Some(Token::Debug),
            _ => None,
        }
    }

    fn alphabetic(&mut self) -> Token {
        let buffer = self.take_while(|c| matches!(c, alphabetic!() | numeric!()));
        if let Some(token) = Self::keyword(&buffer) {
            token
        } else {
            Token::Identifier(buffer)
        }
    }
```

Если `buffer` соответствует какому-то ключевому слову, возвращаем его, а иначе -- идентификатор.

**Функция для обработки строк.**

```rust
    fn string(&mut self) -> Token {
        self.source.next(); // пропускаем '"'
        let buffer = self.take_while(|c| *c != '"');
        self.source.next(); // пропускаем '"'
        Token::String(buffer)
    }
```

Собираем все символы между кавычками, пропуская их самих.

**Пропуск незначимых символов.**

```rust
    fn skip(&mut self) -> Option<Token> {
        self.take_while(|c| matches!(c, skip!()));
        self.next()
    }
```

Пропускаем все пробельные символы и рекурсивно вызываем `self.next()` для того, чтобы получить следующий токен.

**Обработка одиночных символов.**

```rust
    fn single(&mut self) -> Token {
        match self.source.next().unwrap() {
            '+' => Token::Plus,
            '>' => Token::Greater,
            '=' => Token::Equal,
            c => panic!("Неизвестный символ: '{c}'"),
        }
    }
}
```

Для однозначных токенов, таких как операторы, просто сопоставляем символ с соответствующим токеном. А если символ неизвестен -- паникуем (не делайте так в проде).

# Запуск!

Компилируем код и запускаем.

```sh
cargo run
```

**Выход**

```sh
Let
Identifier("a")
Equal
Integer(10)
Plus
Integer(2)
If
Identifier("a")
Greater
Integer(8)
Then
Debug
String("A больше 8")
Else
Debug
String("А либо меньше, либо равно 8")
End
```

Все работает!

# Заключение

Надеюсь, статья была интересной, а самое главное -- полезной. Если что-то непонятно -- пишите в комментарии или мне в личные сообщения чтобы я мог вам это объяснить.
