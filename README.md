1. Какой системный вызов делает команда cd? В прошлом ДЗ мы выяснили, что cd не является самостоятельной программой, это shell builtin, поэтому запустить strace непосредственно на cd не получится. Тем не менее, вы можете запустить strace на /bin/bash -c 'cd /tmp'. В этом случае вы увидите полный список системных вызовов, которые делает сам bash при старте. Вам нужно найти тот единственный, который относится именно к cd. Обратите внимание, что strace выдаёт результат своей работы в поток stderr, а не в stdout.
Ответ: chdir("/tmp")

2. Попробуйте использовать команду file на объекты разных типов на файловой системе. Например:
vagrant@netology1:~$ file /dev/tty
/dev/tty: character special (5/0)
vagrant@netology1:~$ file /dev/sda
/dev/sda: block special (8/0)
vagrant@netology1:~$ file /bin/bash
/bin/bash: ELF 64-bit LSB shared object, x86-64
Используя strace выясните, где находится база данных file на основании которой она делает свои догадки.
Ответ: "/lib/x86_64-linux-gnu/libmagic.so.1

3. Предположим, приложение пишет лог в текстовый файл. Этот файл оказался удален (deleted в lsof), однако возможности сигналом сказать приложению переоткрыть файлы или просто перезапустить приложение – нет. Так как приложение продолжает писать в удаленный файл, место на диске постепенно заканчивается. Основываясь на знаниях о перенаправлении потоков предложите способ обнуления открытого удаленного файла (чтобы освободить место на файловой системе).
Ответ: запустил на терминале запись в файл ping google.ru >& ya2.txt
Проверил, что пишется:
cat ya2.txt
узнал какой это процесс:
ps -a
sudo lsof -p 5105 | grep ya2.txt (без sudo почему-то не выполнялась)
ping    5105 vagrant    1w   REG  253,0     4798 131090 /home/vagrant/ya2.txt
ping    5105 vagrant    2w   REG  253,0     4798 131090 /home/vagrant/ya2.txt
удалил файл: rm ya2.txt
sudo lsof -p 5105 | grep ya2.txt
ping    5105 vagrant    1w   REG  253,0     5934 131090 /home/vagrant/ya2.txt (deleted)
ping    5105 vagrant    2w   REG  253,0     5934 131090 /home/vagrant/ya2.txt (deleted)
Дальше как в презентации, пытаюсь восстановить файл: sudo cat /proc/5105/fd/3 > /home/vagrant/ya2.txt
cat: /proc/5105/fd/3: No such device or address
vagrant@vagrant:~$ ls
ya2.txt

Файл создается, но в него ничего не пишется. Команда cat никак не реагирует.

В чем может быть дело?

4. Занимают ли зомби-процессы какие-то ресурсы в ОС (CPU, RAM, IO)?
Ответ: зомби-процессы освобождают память и не нагружают CPU, они лишь хранят запись для того, чтобы родительсикй процесс мог считать запись о их завершении.

5. В iovisor BCC есть утилита opensnoop:
root@vagrant:~# dpkg -L bpfcc-tools | grep sbin/opensnoop
/usr/sbin/opensnoop-bpfcc
На какие файлы вы увидели вызовы группы open за первую секунду работы утилиты? Воспользуйтесь пакетом bpfcc-tools для Ubuntu 20.04. Дополнительные сведения по установке.
Ответ:PID    COMM               FD ERR PATH
582    irqbalance          6   0 /proc/interrupts
582    irqbalance          6   0 /proc/stat
582    irqbalance          6   0 /proc/irq/20/smp_affinity
582    irqbalance          6   0 /proc/irq/0/smp_affinity
582    irqbalance          6   0 /proc/irq/1/smp_affinity
582    irqbalance          6   0 /proc/irq/8/smp_affinity
582    irqbalance          6   0 /proc/irq/12/smp_affinity
582    irqbalance          6   0 /proc/irq/14/smp_affinity
582    irqbalance          6   0 /proc/irq/15/smp_affinity
793    vminfo              4   0 /var/run/utmp
565    dbus-daemon        -1   2 /usr/local/share/dbus-1/system-services
565    dbus-daemon        18   0 /usr/share/dbus-1/system-services
565    dbus-daemon        -1   2 /lib/dbus-1/system-services
565    dbus-daemon        18   0 /var/lib/snapd/dbus-1/system-services/

6. Какой системный вызов использует uname -a? Приведите цитату из man по этому системному вызову, где описывается альтернативное местоположение в /proc, где можно узнать версию ядра и релиз ОС.
Ответ: uname
Part of the utsname information is also accessible via /proc/sys/kernel/{ostype, hostname, osrelease, version, domainname}.

7. Чем отличается последовательность команд через ; и через && в bash? Например:
root@netology1:~# test -d /tmp/some_dir; echo Hi
Hi
root@netology1:~# test -d /tmp/some_dir && echo Hi
root@netology1:~#
Ответ:
; выполяет команды по очереди
&& выполяет команды до первой ошибки
В первом случае, не смотря на то что такого каталога нет, выводится Hi
Во втором ошибка останавливает выполнение команд

Есть ли смысл использовать в bash &&, если применить set -e?
Ответ:
Нет, это одно и тоже. && и set -e завершают выполнение команды, если возникает ошибка.

8. Из каких опций состоит режим bash set -euxo pipefail и почему его хорошо было бы использовать в сценариях?
set -e (указывает bash немедленно завершить работу, если какая-либо команда имеет ненулевой статус выхода)
set -u (указывает на ошибку в конкретной переменной, кроме $* и $@)
set -x (включает режим оболочки, в котором все выполненные команды выводятся на терминал)
set -o pipefail (Если какая-либо команда в pipeline терпит неудачу, этот код возврата будет использоваться как код возврата всего pipeline. По умолчанию код возврата pipeline - это код последней команды, даже если она выполнена успешно)
Потому что используя set -e, мы бы имели прерванный скрипт, не поняв где в нем произошла ошибка. Данный набор опций дает нам представление о том, как это произошло.

9. Используя -o stat для ps, определите, какой наиболее часто встречающийся статус у процессов в системе. В man ps ознакомьтесь (/PROCESS STATE CODES) что значат дополнительные к основной заглавной буквы статуса процессов. Его можно не учитывать при расчете (считать S, Ss или Ssl равнозначными).
