# Дублирование файловых дескрипторов. Каналы

## Дублирование файловых дескрипторов

Системный вызов `fcntl` позволяет настраивать различные манипуляции над открытыми файловыми дескрипторами. Одной из команд манипуляции является `F_DUPFD` - создание *копии* дескриптора в текущем процессе, но с другим номером.

Копия подразумевает, что два разных файловых дескриптора связаны с одним открытым файлом в процессе, и разделяют следующие его аттрибуты:
 * сам файловый объект;
 * блокировки, связанные с файлом;
 * текущая позиция файла;
 * режим открытия (чтение/запись/добавление).

При этом, не сохраняется флаг `CLOEXEC`, который предпиывает автоматическое закрытие файла при выполнении системного вызова `exec`.

Упрощённой семантикой для создания копии файловых дескрипторов являются системные вызовы POSIX: `dup` и `dup2`:
```
#include <unistd.h>

/* Возвращает копию нового файлового дескриптора, при этом, по аналогии
   с open, численное значение нового файлового дескриптора - минимальный
   не занятый номер. */
int dup(int old_fd);

/* Создаёт копию нового файлового дескриптора с явно указанным номером new_fd.
   Если ранее файловый дескриптор new_fd был открыт, то закрывает его. */
int dup2(int old_fd, int new_fd);
```

## Неименованные каналы

Канал - это пара связанных между собой файловых дескрипторов, один из которых предназначен для только для чтения, а другой - только для записи.

Канал создается с помощью системного вызова `pipe`:
```
#include <unistd.h>

int pipe(int pipefd[2]);
```

В качестве аргумента системному вызову `pipe` передается указатель на массив и двух целых чисел, куда будут записаны номера файловых дескрипторов:
 * `pipefd[0]` - файловый дескриптор, предназначенный для чтения;
 * `pipefd[1]` - файловый дескриптор, предназначенный для записи.

### Запись данных в канал

Осуществляется с помощью системного вызова `write`, первым аргументом которого является `pipefd[1]`. Канал является буферизованным, под Linux обычно его размер 65К. Возможные сценарии поведения при записи:

 * системный вызов `write` завершается немедленно, если размер данных меньше размера буфера, и в буфере есть место;
 * системный вызов `write` приостанавливает выполнение до тех пор, пока не появится место в буфере, то есть предыдущие данные не будут кем-то прочитаны из канала;
 * системный вызов `write` завершается с ошибкой `Broken pipe` (доставляется через сигнал `SIGPIPE`), если с противоположной стороны канал был закрыт, и данные читать некому.

### Чтение данных из канала

Осуществляется с помощью системного вызова `read`, первым аргументом которого является `pipefd[0]`. Возможные сценарии поведения при чтении:

 * если в буфере канала есть данные, то `read` читает их, и завершает свою работу;
 * если буфер пустой и есть **хотя бы один** открытый файловый дескриптор с противоположной стороны, то выполнение `read` блокируется;
 * если буфер пустой и все файловые дескрипторы с противоположной стороны каналы закрыты, то `read` немедленно завершает работу, возвращая `0`.


### Проблема dead lock

При выполнении системных вызовов `fork`, `dup` или `dup2` создаются копии файловых дескрипторов, связанных с каналом. Если не закрывать все лишние (неиспользуемые) копии файловых дескрипторов, предназначенных для записи, то это приводит к тому, что при очередной попытке чтения из канала, `read` вместо того, чтобы завершить работу, будет находиться в ожидании данных.

```
int fds_pair[2];
pipe(fds_pair);

if ( 0!=fork() )  // теперь у нас существует неявная копия файловых дескрипторов
{
    // немного записываем в буфер
    static const char Hello[] = "Hello!";
    write(fds_pair[1], Hello, sizeof(Hello));
    close(fds_pair[1]);

    // а теперь читаем обратно
    char buffer[1024];
    read(fds_pair[0], buffer, sizeof(buffer)); // получаем dead lock!
}
else while (1) shched_yield();
```

Для того, чтобы избежать этой проблемы, необходимо тщательно следить за тем, в какие моменты создаются копии файловых дескрипторов, и закрывать их тогда, когда они не нужны.
