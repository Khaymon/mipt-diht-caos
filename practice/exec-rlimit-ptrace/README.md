# fork-МАГИЯ-exec

## Системный вызов exec

Формальное описание системного вызова exec: [man 2 execve](http://ru.manpages.org/execve/2)

Системный вызов `exec` предназначен для замены программы текущего процесса. Как правило, используется совместно с `fork`, но не обязательно.

Си-оболочки для системного вызова `exec` имеют несколько разных сигнатур.

```
int execve(const char *filename,
           char *const argv[],
           char *const envp[]);           
int execvpe(.....) // параметры аналогично execve

int execv(const char *filename, char *const argv[])
int execvp(......) // параметры аналогично execv

int execle(const char *filename,
           const char arg0, ..., /* NULL */,
           const char env0, ..., /* NULL */);

int execl(const char *filename,
          const char arg0, ..., /* NULL */);
int execlp(......) // параметры аналогично execl

```

Различные буквы в суффиксах названий `exec` означают?
 * `v` или `l` - параметры передаются в виде массивов (`v`), заканчивающихся элементом `NULL`, либо в виде переменного количества аргументов (`l`), где признаком конца перечисления аргументов является значение `NULL`.
 * `e` - кроме аргументов программы передаются переменные окружения в виде строк `КЛЮЧ=ЗНАЧЕНИЕ`.
 * `p` - именем программы может быть не только имя файла, но и имя, которое нужно найти в одном из каталогов, перечисленных в переменной окружения `PATH`.

Возвращаемым значением `exec` может быть только значение `-1` (признак ошибки). В случае успеха, возвращаемое значение уже в принципе не имеет никакого смысла, поскольку будет выполняться другая программа.

Аргументы программы - это то, что передаётся в функцию `main` (на самом деле, они доступны из `_start`, поскольку располагаются на стеке). Первым аргументом (с индексом `0`), как правило, является имя программы, но это не является обязательным требованием.

Классическим способом запуска новой программы является пара системных вызовов:

```
if (0 == fork) {
  execlp(program, program, NULL);
  perror("exec"); exit(1);
}
```

Замена выполняемой программы с помощью `exec` оставляет неизменным многие аттрибуты процесса, например, открытые файловые дескрипторы, лимиты, и переменные окружения, установленные через `setenv`.

Таким образом, между вызовами `fork` и `exec` можно провести дополнительную настройку программы перед выполнением.

```
if (0 == fork) {
  // Заменить стандартные потоки ввода/вывода на файлы
  // (имитация операции >ВЫХОД <ВХОД в bash)
  close(0);
  close(1);
  /* 0 = */ open(in_file, O_RDONLY);
  /* 1 = */ open(out_file, O_WRONLY|O_CREAT|O_TRUNC, 0640);

  execlp(program, program, NULL);
  perror("exec"); exit(1);
}
```

Для того, чтобы случайно (в достаточно больших программах) не передать открытый файловый дескриптор новой программе, в системном вызове `open` предусмотрен флаг открытия `O_CLOEXEC`, который означает, что файл должен быть закрыт при вызове `exec`.


## Лимиты

С процессом связаны некоторые лимиты (ограничения) на используемое процессорное время, максимальный объём памяти, количество файловых дескрипторов, процессов и т. д.

Лимиты подразделяются на *жёсткие*, которые обычные пользователи могут только уменьшать (хотя `root` может и увеличивать), и *мягкие*, которые де-факто являются значениями по умолчанию, и их можно увеличить до жёсткого лимита.

Примерами жёстких лимитов являются ограничения на количество процессов или объём доступной памяти. Пример мягкого лимита - это размер стека, который по умолчанию в Linux равен 8Мб, но может быть изменен произвольным образом, в том числе в большую сторону.

Поскольку изменение размера стека - очень опасная операция, которая может нарушить структуру размещения данных в памяти, изменять этот лимит можно только до запуска функции `main`, либо непосредственно перед выполнением `exec`. В противном случае, поведение программы не определено.

Получение и установка лимитов осуществляются с помощью системных вызовов `getrlimit` и `setrlimit`.

Примеры лимитов см. в [get_limits.c](get_limits.c).

Пример изменения размера стека см. в [shell_with_custom_stack_size.c](shell_with_custom_stack_size.c).


## Трассировка выполнения программы

В Linux (не POSIX!) имеется системный вывов `ptrace`, назначение которого - обеспечить возможность отладки.

Первым аргументом `ptrace` является команда, а дальше - некоторое количество аргументов команды.

Если между вызовами `fork` и `exec` выполнить `ptrace(PTRACE_TRACEME,0,0,0)`, то процесс будет приостановлен до тех пор, пока к нему не подключится отладчик, либо программа, которая ведёт себя подобно отладчику, и не разрешит продолжить выполнение дальше.

Некоторые команды, которые программа-"отладчик" может посылать исследуемому процессу:

 * `PTRACE_CONT, pid, 0, signo` - продолжить выполнение процесса. Если `signo` не 0, то отправляется сигнал.
 * `PTRACE_SINGLESTEP, pid, 0, 0` - выполнить одну инструкцию.
 * `PTRACE_SYSCALL, pid, 0, 0` - продолжить выполнение до системного вызова.
 * `PTRACE_GETREGS, pid, 0, &state)` - получить значение регистров процессора и сохранить в структуру `user_regs_state`, объявленную в файле `<sys/user.h>`.
 * `PTRACE_SETREGS, pid, 0, &state)` - модифицировать регистры процессора.
 * `PTRACE_PEEKDATA, pid, addr, 0` - прочитать одно машинное слово из памяти процесса по указанному адресу `addr`.
 * `PTRACE_POKEDATA, pid, addr, data` - записать одно машинное слово `data` в память процесса по указанному адресу `addr`.

Пример перехвата системного вызова `write`, который цензурирует нехорошее английское слово в тексте вывода см. в [ptrace_catch_string.c](ptrace_catch_string.c).
