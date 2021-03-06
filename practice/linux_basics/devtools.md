# Инструменты разработчика

## Компиляторы `gcc` и `clang`

В стандартную поставку современных UNIX-систем входит один из компиляторов: либо `gcc`, либо `clang`. В случае с Linux, по умолчанию обычно используется `gcc`, а в BSD-системах - `clang`. Далее будет описана работа с компилятором `gcc`, имея ввиду, что работа с `clang` ничем принципиально не отличается: у обоих компиляторов много общего, в том числе опции командной строки.

Кроме того, существует команда `cc`, которая является символической ссылкой на используемый по умолчанию компилятор языка Си (`gcc` или `clang`), и команда `c++`, - символическая ссылка на используемый по умолчанию компилятор для C++.

Рассмотрим простейшую программу на языке C++:
```
// файл hello.cpp
#include <iostream>

int
main() {
  std::cout << "Hello, World!" << std::endl;
  return 0;
}
```

Скомпилировать эту программу можно с помощью команды:
```
> c++ -o program.jpg hello.cpp
```

Опция компилятора `-o ИМЯ_ФАЙЛА` указывает имя выходного файла, который нужно создать. По умолчанию используется имя `a.out`. Обратите внимание, что файл `program.jpg` является обычным выполняемым файлом!

### Стадии сборки программы на Си или C++

При выполнении команды `c++ -o program.jpg hello.cpp` выполняется достаточно сложная цепочка действий:

 1. Выполняется *препроцессинг* текстового файла `hello.cpp`. На этом этапе обрабатываются *директивы препроцессора* (которые начинаются с символа `#`), и получается новый текст программы. Если запустить компилятор с опцией `-E`, то будет выполнен только этот шаг, и на стандартный поток вывода будет выведен преобразованный текст программы.

 2. Выполняется *трансляция* одного или нескольких текстов на Си или C++ в объектные модули, содержащие машинный код. Если указать опцию `-c`, то на этом сборка программы будет приостановлена, и будут созданы объектные файлы с суффиксом `.o`. Объектные файлы содержат *бинарный* исполняемый код, которому в точности соотвествует некоторый текст на языке ассемблера. Этот текст можно получить с помощью опции `-S`, - в этом случае, вместо объектных файлов будут созданы текстовые файлы с суффиксом `.s`.

 3. Компоновка одного или нескольких объектных файлов в исполняемый файл, и связываение его со стандартной библиотекой Си/С++ (ну и другими библиотеками, если требуется). Для выполнения компоновки компилятор вызывает стороннюю программу `ld`.

### Программы на Си v.s. программы на C++

Компилятор `gcc` имеет опцию `-x ЯЗЫК`, для указания языка исходного текста программы: Си (`c`), C++ (`c++`) или Фортран (`fortran`). По умолчанию, язык исходного текста определяется в соответствии с именем файла: `.c`, - это программы на языке Си, а файлы, оканчивающиеся на `.cc`, `.cpp` или `.cxx`, - это тексты на языке C++. Таким образом, имя файла является существенным.

Это относится к стадиям препроцессинга и трансляция, но может вызвать проблемы на стадии компоновки. Например, используя команду `gcc` вместо `g++` (или `cc` вместо `c++`), можно успешно скомпилировать исходный текст программы на C++, но при этом возникнут ошибки на стадии связывания, поскольку компоновщику `ld` будут переданы опции, подразумевающие связывание только со стандартной библиотекой Си, но не C++. Поэтому, при сборке программ на C++ нужно использовать команду `c++` или `g++`.

### Указание стандартов

Опция компилятора `-std=ИМЯ` позволяет явным образом указать используемый стандарт языка. Рекомендуется явным образом указывать используемый стандарт, поскольку поведение по умолчанию зависит от используемого дистрибутива Linux. Допустимые имена:
 * `c89`, `c99`, `c11`, `gnu99`, `gnu11` для языка Си;
 * `c++03`, `c++11`, `c++14`, `c++17`, `gnu++11`, `gnu++14`, `gnu++17` для языка C++.

Двузначное число в имени стандарта указывает на его год. Если в имени стандарта присутствует `gnu`, то подразумеваются GNU-расширения компилятора, специфичные для UNIX-подобных систем, и кроме того, считается определенным макрос `#define _DEFAULT_SOURCE`, который в некоторых случаях меняет поведение отдельных функций стандартной библиотеки.

В дальнейшем мы будем ориентироваться на стандарт `c11`, а в некоторых задачах, где будет про это явно указано - его расширением `gnu11`.

## Объектные файлы, библиотеки и исполняемые файлы

### Модуль `ctypes` интерпретатора Python

Рассмотрим [программу](my-first-program.c) на языке Си:
```
/* my-first-program.c */
#include <stdio.h>

static void
do_something()
{
    printf("Hello, World!\n");
}

extern void
do_something_else(int value)
{
    printf("Number is %d\n", value);
}

int
main()
{
    do_something();
}
```

Скомпилируем эту программу в объектный файл, а затем - получим из него: (1) выполняемую программу; (2) разделяемую библиотеку. Обратите внимание на опцию `-fPIC`, предназначенную для генерации позиционно-независимого кода, о чем будет рассказано на одном из последующих семинарах.

```
> gcc -c -fPIC my-first-program.c
> gcc -o program.jpg my-first-program.o
> gcc -shared -o library.so my-first-program.o
```

В результате мы получим программу `program.jpg`, которая
выводит на экран строку `Hello, World!`, и *библиотеку* с именем `library.so`, которую можно использовать как из Си/C++ программы, так и динамически подгрузить для использования интерпретатором Python:

```
> python3
Python 3.6.5 (default, Mar 31 2018, 19:45:04) [GCC] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from ctypes import cdll
>>> lib = cdll.LoadLibrary("./library.so")
>>> lib.do_something_else(123)
>>> retval = lib.do_something_else(123)
Number is 123
>>> print(retval)
14
```
Обратите внимание, что результатом работы функции `do_something_else` является какое-то загадочное число `14` (возможно, будет какое-то другое при попытке воспроизвети этот эксперимент), хотя функция возвращает `void`.

Причина заключается в том, что разделяемые библиотеки хранят только **имена** функций, но не их сигнатуры (типы параметров и возвращаемого значения).

Попытка вызвать функцию `do_something` не увенчается успехом:
```
>>> lib.do_something()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib64/python3.6/ctypes/__init__.py", line 361, in __getattr__
    func = self.__getitem__(name)
  File "/usr/lib64/python3.6/ctypes/__init__.py", line 366, in __getitem__
    func = self._FuncPtr((name_or_ordinal, self))
AttributeError: ./library.so: undefined symbol: do_something
```

В этом случае имя `do_something` не найдено, поскольку в исходном тексте на языке Си модификатор `static` перед именем функции явно запрещает использование функции где-либо вне текущего исходного текста.

### Просмотр таблицы символов

Для исследования объектных файлов, в том числе и скомпонованных, используется утилита `objdump`.

Опция `--syms` или `-t` отображает отдельные секции исполняемого объектного файла, которым присвоены имена - *символы*.

Некоторые имена имеют пометку `*UND*`, - это означает, что имя используется в объектном файле, но его расположение неизвестно. Задача компоновщика состоит как раз в том, чтобы найти требуемые имена в разных объектных файлах или динамических библиотеках, а затем - подставить правильный адрес.

Некоторые символы помечены как глобальные (символ `g` во втором столбце вывода), а некоторые - как локальные (символ `l`). Те символы, которые не являются глобальными, считаются *не экспортируемыми*, то есть (теоретически) не должны быть доступны извне.


## Отладчик

Если скомпилировать программу с опцией `-g`, то размер программы увеличится, поскольку в ней появляются дополнительные секции, которые содержат *отладочную информацию*.

Отладочная информация содержит сведения о соответствии отдельных фрагментов программы исходным текстам, и включает в себя номера строк, имена исходных файлов, имена типов, функций и переменных.

Эти информация используется только отладчиком, и почти никак не влияет на поведение программы. Таким образом, отладочную информацию можно совмещать с оптимизацией, в том числе достаточно агрессивной (опция компилятора `-O3`).

Для запуска программы под отладчиком используется команда `gdb`, в качестве аргумента к которой указывается имя выполняемого файла или команды.

Основные команды `gdb`:
 * `run` - запуск программы, после `run` можно указать аргументы;
 * `break` - установить точку останова, параметрами этой команды может имя функции или пара `ИМЯ_ФАЙЛА:НОМЕР_СТРОКИ`;
 * `ni`, `si` - шаг через строку или шаг внутрь функции соответственно;
 * `return` - выйти из текущей функции наверх;
 * `continue` - продолжить выполнение до следующей точки останова или исключительной ситуации;
 * `print` - вывести значение переменной из текущего контекста выполнения.

Взаимодействие с отладчиком производится в режиме командной строки. Различные интегрированные среды разработки (CLion, CodeBlocks, QtCreator) являются всего лишь графической оболочкой, использующей именно этот отладчик, и визуализируя взаимодействие с ним.

Более подробный список команд можно посмотреть в [CheatSheet](https://www.cheatography.com/fristle/cheat-sheets/closed-source-debugging-with-gdb/).
