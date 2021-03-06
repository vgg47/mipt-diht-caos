# Адресация данных в памяти

## Дригие материалы семинара

* [Reference по ARM](../arm_basics/arm_reference.pdf)
* [Лекция по IEEE754](../../../lectures/fall-2018/Lection03-InstrEncoding_IEEE754.pdf)



## Основные команды

Как свойственно классической RISC-архитектуре, процессор ARM
может выполнять операции только над регистрами. Для доступа к памяти используются отдельные команды *загрузки* (`ldr`) и
*сохранения* (`str`).

Общий вид команд:
```
LDR{условие}{тип} Регистр, Адрес
STR{условие]{тип} Регистр, Адрес
```
где `{условие}` - это условие выполнения команды, может быть
пустым (см. предыдущий семинар); `{тип}` - тип данных:
 * `B` - беззнаковый байт
 * `SB` - знаковый байт
 * `H` - полуслово (16 бит)
 * `SB`- знаковое полуслово
 * `D` - двойное слово.

Если тип в названии команды не указан, то подразумевается
обычное слово. Обратите внимание, что для выполнения операций загрузки/сохранения данных, меньших, чем машинное слово, отдельно выделяются знаковые команды, которые
делают аккуратное расширение бит нулями, сохраняя при этом
старший знаковый бит.

В случае операций загрузки/сохранения пары регистров (двойное слово), регистр должен быть с четным номером. Второе машинное слово подразумевается в соседнем регистре с номером `Rn+1`.

## Адресация

Адрес имеет вид:
`[R_base {, offset}]`
где `R_base` - имя регистра, который содержит базовый адрес в памяти, а необязательный параметр `offset` - смещение относительно адреса. Итоговый адрес определяется как
`*R_base + offset`.

Смещение может быть как именем регистра, так и численной константой, закодированной в команду. Регистры обычно используются для индексации элементов массива, константы - для доступа к полям структуры или локальным переменным и аргументам относительно `[sp]`.

## Адресация полей Си-структур

По стандарту языка Си, поля в памяти структур размещаются по следующим правилам:
 * порядок полей в памяти соответствует порядку полей в описании структуры
 * размер структуры должен быть кратен размеру машинного слова
 * данные внутри машинных слов размещаются таким образом, чтобы быть прижатыми к их границам.

Таким образом, размер структуры не всегда совпадает с суммой размеров отдельных полей. Например:

```
struct A {
  char  f1; // 1 байт
  int   f2; // 4 байта
  char  f3; // 1 байт
};
// 1 + 4 + 1      = 6 байт
// size(struct A) = 12 байт
```

В данном примере поле `f1` занимает часть машинного слова, поле `f2` - имеет размер 4 байта, поэтому занимает уже следующее машинное слово, и для поля `f3` приходится использовать ещё одно. Простая перестановка полей местами позволяет сэкономить 4 байта:

```
struct A {
  char  f1; // 1 байт
  char  f3; // 1 байт
  int   f2; // 4 байта  
};
// 1 + 1 + 4      = 6 байт
// size(struct A) = 8 байт
```
В этом случае поля `f1` и `f3` занимают одно и то же машинное слово.

Компилятор GCC имеет нестандартный аттрибут `packed`, позволяющий создавать "упакованные" структуры, размер которых равен сумме размеров отдельных его полей:

```
struct A {
  char  f1; // 1 байт
  int   f2; // 4 байта
  char  f3; // 1 байт
} __attribute__((packed));
// 1 + 4 + 1      = 6 байт
// size(struct A) = 6 байт
```
