Процесс загрузки ядра. Часть 1.
===============================

От загрузчика к ядру
--------------------

Если вы читали предыдущие [статьи](http://0xax.blogspot.com/search/label/asm)
моего блога, то могли заметить, что некоторое время назад я начал увлекаться
низкоуровневым программированием. Я написал несколько статей о программировании
под x86\_64 в Linux. В то же время я начал "погружаться" в исходный код ОС
Linux. Было очень интересно, как все работает на низком уровне, как запускаются
программы на моем компьютере, как они помещаются в память, как ядро управляет
процессами и памятью, как работает сеть и многие другие вещи. Поэтому я решил
написать еще одну серию статей о ядре Linux для **x86\_64**.

Замечу, что я не первоклассный специалист со знаниями ядра и не пишу под него
код на работе. Это просто хобби. Мне просто нравятся все эти низкоуровневые
вещи и мне интересно наблюдать за тем, как они работают. Так что, если вас
что-то будет смущать или у вас появятся вопросы или замечания, пишите мне в
твиттер [0xAX](https://twitter.com/0xAX), присылайте письма на
[email](anotherworldofworld@gmail.com) или просто создавайте
[issue](https://github.com/0xAX/linux-insides/issues/new). Я ценю это. Все
статьи также будут доступны на странице
[linux-insides](https://github.com/0xAX/linux-insides), и, если вы обнаружите
какую-нибудь ошибку в моем английском или в содержимом статьи, присылайте pull
request.


*Заметьте, что это не официальная документация, а просто материал для обучения
и обмена знаниями.*

**Требуемые знания**

* Понимание кода на языке C
* Понимание кода на языке ассемблера (синтаксис AT&T)

В любом случае, если вы только начинаете изучать какие-то инструменты, я
постараюсь объяснить некоторые моменты этой и последующих частей. Ладно,
простое введение закончилось, и теперь можно начать "погружение" в ядро и
всякие низкоуровневые штуки.

Весь код, представленный здесь, в основном для ядра версии 3.18. Если есть
какие-то изменения, позже я обновлю статьи соответствующим образом.

Магическая кнопка Старт и что происходит дальше?
-------------------------------------------------------------------------------

Несмотря на то, что это серия статей о ядре Linux, мы не будем начинать с его
исходного кода (по крайней мере не в этом параграфе). Ок, вы нажали магическую
кнопку старта на своем ноутбуке или настольном компьютере, и он начал работать.
После того, как материнская плата отправит сигнал к
[источнику питания](https://en.wikipedia.org/wiki/Power_supply), источник
питания обеспечит компьютер достаточным количеством электричества. Как только
материнская плата получит сигнал
["питание в норме"](https://en.wikipedia.org/wiki/Power_good_signal), она
пытается запустить ЦПУ. ЦПУ перезапускает остаточные данные в своих регистрах и
записывает предустановленные значения каждого из них.


ЦПУ серии [Intel 80386](https://en.wikipedia.org/wiki/Intel_80386) и старше
после перезапуска компьютера заполняют регистры следующими предустановленными
значениями:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

Процессор начинает работать в
[режиме реальных адресов](https://en.wikipedia.org/wiki/Real_mode). Давайте
задержимся здесь ненадолго и попытаемся понять сегментацию памяти в этом
режиме. Режим реальных адресов поддерживается всеми x86-совместимыми
процессорами, от [8086](https://en.wikipedia.org/wiki/Intel_8086) до самых
новых 64-битных ЦПУ Intel. Процессор 8086 имел 20-битную шину адреса, т.е. он
мог работать с адресным пространством в диапазоне 0-0x100000 (1 мегабайт). Но
регистры у него были только 16-битные, а в таком случае максимальный размер
адресуемой памяти составляет 2^16 - 1 или 0xffff (64 килобайта).
[Сегментация памяти](http://en.wikipedia.org/wiki/Memory_segmentation)
используется, чтобы задействовать все доступное адресное пространство. Вся
память делится на небольшие, фиксированного размера сегменты по 65536 байт или
64 Кб. Поскольку мы не можем адресовать память свыше 64 Кб с помощью 16-битных
регистров, был придуман альтернативный метод. Адрес состоит из двух частей:
селектора сегмента, который содержит базовый адрес, и смещение от этого
базового адреса. В режиме реальных адресов базовый адрес селектора сегмента это
`Селектор Сегмента * 16`. Таким образом, чтобы получить физический адрес в
памяти, нужно умножить селектор сегмента на 16 и прибавить смещение:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Например, если `CS:IP` содержит `0x2000:0x0010`, то соответствующий физический
адрес будет:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

Но если мы возьмем максимально доступный селектор сегментов и смещение:
`0xffff:0xffff`, то получим следующее значение:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

что больше первого мегабайта на 65520 байт. Т.к. в режиме реальных адресов
доступен только один мегабайт, с отключенной
[адресной линией A20](https://en.wikipedia.org/wiki/A20_line) `0x10ffef`
становится `0x00ffef`.

Хорошо, теперь мы знаем о режиме реальных адресов и адресации памяти. Давайте
вернемся к обсуждению значений регистров после перезапуска:

Регистр `CS` состоит из двух частей: видимый селектор сегмента и скрытый
базовый адрес. В то время, как базовый адрес рассчитывается умножением значения
селектора сегмента на 16 (во время аппаратного перезапуска), в селектор
сегмента в регистре `CS` записывается 0xf000, а в базовый адрес - 0xffff0000.
Процессор использует этот специальный базовый адрес, пока регистр `CS` не
изменится.

Начальный адрес получается в результате сложения базового адреса и значения
регистра `EIP`:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

Мы получили `0xfffffff0`, т.е. 4 Гб - 16 байт. По этому адресу располагается
т.н. [Вектор прерываний](http://en.wikipedia.org/wiki/Reset_vector). Это
область памяти, в которой ЦПУ ожидает найти первую инструкцию для выполнения
после перезапуска. Она содержит инструкцию
[jump](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29), которая обычно
указывает на точку входа в `BIOS`. Например, если мы взглянем на исходный код
[coreboot](http://www.coreboot.org/), то увидим следующее:

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

Здесь мы можем видеть
[опкод инструкции jmp](http://ref.x86asm.net/coder32.html#xE9) - `0xe9` и его
адрес назначения - `_start - ( . + 2 )`, а также мы видим, что секция `reset`
занимает 16 байт и начинается с `0xfffffff0`:

```
SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}
```

Теперь запускается `BIOS`: после инициализации и проверки оборудования ей нужно
найти загрузочное устройство. Порядок загрузки хранится в конфигурации `BIOS`,
которая определяет, с каких устройств `BIOS` попробует загрузиться. Когда
предпринимается попытка загрузиться с жесткого диска, `BIOS` пытается найти
загрузочный сектор. На размеченных жестких дисках с
[MBR](https://en.wikipedia.org/wiki/Master_boot_record) загрузочный сектор
расположен в первых 446 байтах первого сектора (размер которого 512 байт).
Последние два байта первого сектора это `0x55` и `0xaa`, которые оповещают
`BIOS` о том, что с устройства можно загружаться. Например:

```assembly
;
; Заметка: это пример написан с использованием синтаксиса Intel Assembly
;
[BITS 16]
[ORG  0x7c00]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Собрать и запустить этот код можно таким образом:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

Такая команда оповещает эмулятор [QEMU](http://qemu.org) о необходимости
использовать в качестве образа диска созданный только что бинарный файл. Пока
последний проверяет, удовлетворяет ли загрузочный сектор всем необходимым
требованиям (в `origin` записывается `0x7c00`, а в конце магическая
последовательность), `QEMU` будет работать с бинарным файлом как с главной
загрузочной записью образа (MBR) диска.

Вы увидите:

![Простой загрузчик, который выводит только `!`](http://oi60.tinypic.com/2qbwup0.jpg)

Этот пример показывает, что код будет выполняться в 16 битах в режиме реальных
адресов и начнет выполнение с адреса `0x7c00`. После запуска он вызывает
прерывание [0x10](http://www.ctyme.com/intr/rb-0106.htm), которое просто
выводит символ `!`. Оставшиеся 510 байт заполняются нулями, и код заканчивается
двумя магическими байтами `0xaa` и `0x55`.

Также можно посмотреть на двоичный дамп с помощью утилиты `objdump`:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

На самом деле у загрузочного сектора есть код продолжения процесса загрузки,
таблица разметки вместо набора нулей и восклицательный знак :) Начиная с этого
момента, `BIOS` держит в своих руках процесс загрузки.

**ЗАМЕЧАНИЕ**: Как вы уже прочитали выше - ЦПУ находится в режиме реальных
адресов. В этом режиме вычисление физического адреса в памяти происходит так:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Также, как было упомянуто ранее. У нас есть только 16 битные регистры общего
назначения, максимальное значение которых `0xffff`, поэтому, если брать по
максимуму, результат будет:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

Где `0x10ffef` равен `1 Мб + 64 Кб - 16 байт`. Но у процессора
[8086](https://en.wikipedia.org/wiki/Intel_8086), первого процессора с режимом
реальных адресов, 20-битная шина адресации, а `2^20 = 1048576` это 1 Мб.
Это означает, что фактическое значение доступной памяти 1 Мб.

Основные разделы памяти в режиме реальных адресов:

```
0x00000000 - 0x000003FF - Таблица векторов прерываний
0x00000400 - 0x000004FF - Данные BIOS
0x00000500 - 0x00007BFF - Не используется
0x00007C00 - 0x00007DFF - Наш загрузчик
0x00007E00 - 0x0009FFFF - Не используется
0x000A0000 - 0x000BFFFF - RAM (VRAM) видео памяти
0x000B0000 - 0x000B7777 - Память монохромного видео
0x000B8000 - 0x000BFFFF - Память цветного видео
0x000C0000 - 0x000C7FFF - BIOS ROM видео-памяти
0x000C8000 - 0x000EFFFF - Скрытая память BIOS
0x000F0000 - 0x000FFFFF - Системная BIOS
```

В самом начале статьи я написал, что первая инструкция, выполняемая ЦПУ
расположена по адресу `0xFFFFFFF0`, значение которого намного больше, чем
`0xFFFFF` (1 Мб). Каким образом ЦПУ получает доступ к этому участку в режиме
реальных адресов? Это описано в документации
[coreboot](http://www.coreboot.org/Developer_Manual/Memory_map):

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 Кб ROM указывают на адресное пространство
```

В самом начале выполнения `BIOS` находится не в RAM, а в ROM.

Загрузчик
---------

В Linux есть несколько загрузчиков, такие как
[GRUB 2](https://www.gnu.org/software/grub/) и
[syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project).
У ядра Linux есть
[протокол загрузки](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt),
который определяет все необходимые настройки для правильной работы ОС.
В качестве примера опишем GRUB 2.

Теперь, когда `BIOS` выбрал загрузочный диск и передал контроль управления
коду в загрузочном секторе, начинается выполнение
[boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD).
Этот код очень простой из-за ограничения доступного пространства. Он содержит
указатель, используемый для перехода к основному образу GRUB 2. Основной образ
начинается с
[diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD),
который обычно располагается сразу после первого сектора в неиспользуемой
области перед первым разделом. Код загружает в память оставшуюся часть
основного раздела, который содержит ядро и драйверы GRUB 2 для управления
файловой системой. После этого код выполняет
[grub\_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c).

`grub_main` инициализирует консоль, получает базовый адрес для модулей,
устанавливает корневое устройство, загружает/обрабатывает файл настроек `grub`,
загружает модули и т.д. Закончив работу, `grub_main` переводит `grub` обратно в
нормальный режим. `grub_normal_execute` (из `grub-core/normal/main.c`)
завершает подготовку и отображает меню выбора операционной системы. Когда мы
выбираем один из пунктов меню, запускается `grub_menu_execute_entry`, который
в свою очередь запускает команду `grub` `boot`, загружающую выбранную
операционную систему.

Из протокола загрузки видно, что загрузчик должен читать и заполнять некоторые
поля в заголовке ядра, который начинается со смещения `0x01f1` в коде настроек.
Заголовок ядра
[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S)
начинается с:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

Загрузчик должен заполнить этот и другие заголовки (помеченные как `write`
в протоколе загрузки ядра, например,
[этот](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L354)
) значениями, которые он также получает от команды или вычисленными значениями.
Мы не увидим описание и объяснение всех полей заголовка ядра, но, когда он
будет их использовать, вернемся к этому. Тем не менее вы можете найти полное
описание всех полей в
[протоколе загрузки](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156).

Как мы видим из протокола, после загрузки ядра схема распределения памяти
будет выглядеть следующим образом:

```shell
         | Ядро защищенного режима  |
100000   +--------------------------+
         | Память I/O               |
0A0000   +--------------------------+
         | Резерв для BIOS          | Оставлен максимально допустимый размер
         ~                          ~
         | Список команд            | (Может быть ниже X+10000)
X+10000  +--------------------------+
         | Стек/куча                | Используется кодом ядра в режиме реальных адресов
X+08000  +--------------------------+
         | Настройки ядра           | Код ядра режима реальных адресов.
         | Загрузочный сектор ядра  | Унаследованный загрузочный сектор ядра.
       X +--------------------------+
         | Загрузчик                | <- Точка входа загрузочного сектора 0x7C00
001000   +--------------------------+
         | Резерв под MBR/BIOS      |
000800   +--------------------------+
         | Обычно используется MBR  |
000600   +--------------------------+
         | Используется только BIOS |
000000   +--------------------------+

```

Итак, когда загрузчик передает управление ядру, он запускается с:

```
0x1000 + X + sizeof(KernelBootSector) + 1
```

где `X` это адрес загруженного сектора загрузки ядра. В моем случае `X` это
`0x10000`, как мы можем увидеть в дампе памяти:

![первый адрес ядра](http://oi57.tinypic.com/16bkco2.jpg)

Сейчас загрузчик поместил ядро Linux в память, заполнил поля заголовка и
переключился на него. Теперь мы можем перейти непосредственно к коду настройки
ядра.

Запуск настройки ядра
---------------------

Наконец-то, мы в ядре. Технически ядро еще не запущено, для начала нам нужно
настроить его, менеджер памяти, менеджер процессов и т.д. Настройка ядра
начинается в
[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S),
начиная со
[\_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L293).
На первый взгляд это кажется немного странным, т.к. перед этим еще несколько
инструкций.

Давным-давно у Linux был свой загрузчик, но сейчас, если вы запустите,
например:

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

то увидите:

![Попытка использовать vmlinuz в qemu](http://oi60.tinypic.com/r02xkz.jpg)

Вообще-то `header.S` начинается с
[MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (см. картинку выше),
вывода сообщения об ошибке и
[PE](https://en.wikipedia.org/wiki/Portable_Executable) заголовка:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

Это нужно, чтобы загрузить операционную систему с интерфейсом
[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface).
Прямо сейчас мы не увидим, как это работает; это будет в одной из следующих
глав.

Итак, настоящая настройка ядра начинается с:

```assembly
// header.S line 292
.globl _start
_start:
```

Загрузчик (grub2 или другой) знает об этой метке (смещение `0x200` от `MZ`)
и сразу переходит на нее, несмотря на то, что `header.S` начинается с секции
`.bstext`, которая выводит сообщение об ошибке:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // текущее положение
.bstext : { *(.bstext) }  // поместить секцию .bstext в позицию 0
.bsdata : { *(.bsdata) }
```

Так точка входа настройки ядра:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // остаток заголовка
    //
```

Здесь мы видим опкод инструкции `jmp` - `0xeb` к метке `start_of_setup-1f`.
Нотация `Nf` означает, что `2f` ссылается на следующую локальную метку `2:`.
В нашем случае это метка `1`, которая расположена сразу после инструкции
`jump`. Она содержит оставшуюся часть
[заголовка](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156).
Сразу после заголовка настроек мы видим секцию `.entrytext`, которая начинается
с метки `start_of_setup`.

В общем-то, это первый код, который запускается (отдельно от предыдущей
инструкции `jump`, конечно). После того, как настройщик ядра получил
управление от загрузчика, первая инструкция `jmp` располагалась в смещении
`0x200` (первые 512 байт) от начала реальных адресов. Об этом можно узнать из
протокола загрузки ядра Linux, а также увидеть в исходном коде grub2:

```C
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

Это означает, что после начала настройки ядра регистры сегмента будут иметь
следующие значения:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

Это справедливо для моего случая, когда ядро загружено по адресу `0x10000`.

После перехода на метку `start_of_setup`, необходимо соблюсти условия:

* Быть уверенным, что все значения всех сегментных регистров равны
* Правильно настроить стек, если нужно
* Настроить [bss](https://en.wikipedia.org/wiki/.bss)
* Перейти к C-коду
  [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)

Давайте посмотрим, как эти условия выполняются.

Выравнивание сегментных регистров
---------------------------------

В первую очередь необходимо убедиться, что сегментные регистры `ds` and `es`
указывают на один и тот же адрес, а также что флаг направления очищен
инструкцией `cld`:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

Как я уже писал ранее, grub2 загружает код настройки ядра по адресу `0x10000`
, а `cs` - `0x1020`, потому что запуск происходит не с начала файла, а с:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

инструкции  `jump`, расположение которой смещено на 512 байт от
[4d 5a](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L47).
Также необходимо обновить `cs`: с `0x10200` на `0x10000`, как и остальные
сегментные регистры. После этого мы настраиваем стек:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

кладем значение `ds` на стек по адресу метки
[6](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L494)
и выполняем инструкцию `lretw`. Когда мы вызываем `lretw`, она загружает адрес
метки `6` в регистр
[указателя инструкций](https://en.wikipedia.org/wiki/Program_counter),
а в `cs` - значение `ds`. После этого `ds` и `cs` будут иметь одно и то же
значение.

Настройка стека
---------------

Вообще-то, почти весь код настройки это подготовка для среды языка C в реальном
режиме. Следующий
[шаг](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L467)
состоит в проверке значения регистра `ss` и создании правильного стека, если
значение `ss` неверно:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

Это может привести к трем различны сценариям:

* `ss` имеет верное значение 0x10000 (как и все остальные сегментные регистры
  рядом с `cs`)
* `ss` некорректный и установлен флаг `CAN_USE_HEAP` (см. ниже)
* `ss` некорректный и флаг `CAN_USE_HEAP` не установлен (см. ниже)

Рассмотрим все три сценария:

* `ss` имеет верный адрес (0x10000). В этом случае мы переходим к метке
  [2](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L481):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Здесь мы видим выравнивание сегмента `dx` (содержащего значение `sp`,
полученное загрузчиком) четырью байтами и проверку - является ли полученное
значение нулем. Если ноль, то помещаем `0xfffx` (выровненный четырью байтами
адрес до максимального значения сегмента за вычетом 64 Кб) в `dx`. Если не
ноль, продолжаем использовать `sp`, полученный от загрузчика (в моем случае
`0xf7f4`). После этого мы помещаем значение `ax` в `ss`, который хранит
правильный адрес сегмента `0x10000` и устанавливает правильное значение `sp`.
Теперь наш стек скорректирован:

![стек](http://oi58.tinypic.com/16iwcis.jpg)

* рассмотрим второй сценарий (когда `ss` != `ds`). Во-первых, поместим значение
  [_end](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L52)
  (адрес окончания кода настройки) в `dx` и проверим поле заголовока
  `loadflags` инструкцией `testb`, чтобы проверить, можем ли мы использовать
  кучу (heap) или нет.
  [loadflags](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321)
  это заголовок с битовой маской, который определяется так:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

И, как мы можем узнать из протокола загрузки:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

Если бит `CAN_USE_HEAP` установлен, поместить `heap_end_ptr` в `dx`, который
указывает на `_end` и добавить к нему `STACK_SIZE` (минимальный размер стека -
512 байт). После этого, если `dx` без переноса (так и будет, dx = _end + 512),
переходим на метку `2`, как в предыдущем случае и создаем правильный стек.

![стек](http://oi62.tinypic.com/dr7b5w.jpg)

* Когда флаг `CAN_USE_HEAP` не выставлен, мы просто используем минимальный
  стек от `_end` до `_end + STACK_SIZE`:

![минимальный стек](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
---------

The last two steps that need to happen before we can jump to the main C code,
are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking
the "magic" signature. First, signature checking:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the
[setup_sig](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L39)
with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is
reported.

If the magic number matches, knowing we have a set of correct segment registers
and a stack, we only need to set up the BSS section before jumping into the C
code.

The BSS section is used to store statically allocated, uninitialized data. Linux
carefully ensures this area of memory is first blanked, using the following
code:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First of all the
[__bss_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L47)
address is moved into `di` and the `_end + 3` address (+3 - aligns to 4 bytes)
is moved into `cx`. The `eax` register is cleared (using a `xor` instruction),
and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx`
is divided by four (the size of a 'word'), and the `stosl` instruction is
repeatedly used, storing the value of `eax` (zero) into the address pointed to
by `di`, automatically increasing `di` by four (this occurs until `cx` reaches
zero). The net effect of this code is that zeros are written through all words
in memory from `_\_bss\_start` to `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
------------

That's all, we have the stack and BSS so we can jump to the `main()` C function:

```assembly
    calll main
```

The `main()` function is located in
[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c).
You can read about what this does in the next part.

Conclusion
----------

This is the end of the first part about Linux kernel insides. If you have
questions or suggestions, ping me in twitter [0xAX](https://twitter.com/0xAX),
drop me [email](anotherworldofworld@gmail.com) or just create
[issue](https://github.com/0xAX/linux-internals/issues/new). In the next part we
will see first C code which executes in Linux kernel setup, implementation of
memory routines as `memset`, `memcpy`, `earlyprintk` implementation and early
console initialization and many more.

**Please note that English is not my first language and I am really sorry for
any inconvenience. If you find any mistakes please send me PR to
[linux-insides](https://github.com/0xAX/linux-internals).**

Links
-----

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
