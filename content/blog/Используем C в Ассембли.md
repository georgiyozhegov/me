+++
date = "2025-03-28T10:06:37+04:00"
draft = false
title = "Используем C в Ассембли"
+++

![Процесс линковки](/me/CWithAsm.png)

Разрабатывая стандартную библиотеку для своего языка, столкнулся с неочевидной задачей: как связывать код написанный на C с ассембли. Первый подход -- компиляция C в ассембли и ручное копирование кода -- оказался не самым удобным. Две проблемы этого способа это несовместимость синтаксиса GCC и Nasm и постоянное дублирование кода при малейших изменениях.

# Решение

Теперь расскажу о способе, который является оптимальным -- **линковке объектных файлов**.

## Пример

Приведу пример из моего языка -- функция для печати целых чисел.

**_debug.c_**

```c
extern void write_i64(long long value) {
    // тело функции в конце статьи
}
```

Важно, что функция объявлена с модификатором `extern`, то есть доступна глобально.

---

Также, в него нужно включить заголовочный файл, в котором будут объявлены все сигнатуры функций.

**_debug.c_**

```c
#include "debug.h"

// ...
```

**_debug.h_**

```c
#ifndef DEBUG_H
#define DEBUG_H

void write_i64(long long value);

#endif
```

Теперь, создаём объектный файл.

```sh
gcc -nostdlib -no-pie -fno-stack-protector -c debug.c -o debug.o
```

Флаги `-no-pie` и `-fno-stack-protector` нужны для совместимости с ассембли.

**_main.asm_**

```asm
section .text
global _start

_start:
    extern write_i64
    mov rdi, 999
    call write_i64 ; печатает: 999
    mov rax, 60
    mov rdi, 0
    syscall
```

Компилируем и компонуем с объектным файлом стандартной библиотеки

```sh
nasm -f elf64 main.asm -o main.o
gcc -nostdlib -no-pie main.o debug.o -o main
```

Получаем одиночный бинарный файл, в котором включены и стандартная библиотека и главный файл.

---

**P.S:** Тело функции

```c
extern void write_i64(long long value) {
      if (value == 0) {
            write_c('0');
            return;
      }

      char buffer[20];
      int size = 0;
      int negative = value < 0;

      if (negative)
            value = -value;

      while (value > 0) {
            int digit = value % 10;
            buffer[size++] = digit + '0';
            value = (value - digit) / 10;
      }

      if (negative)
            buffer[size++] = '-';

      for (int index = size - 1; index >= 0; index--)
            write_c(buffer[index]);
}
```
