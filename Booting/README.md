# Процесс загрузки ядра

Эта глава описывает процесс загрузки ядра Linux. Здесь вы увидите несколько
статей, описывающих полный цикл процесса загрузки ядра:

* [От загрузчика к ядру](https://github.com/olshevskiy87/linux-insides-ru/blob/master/Booting/linux-bootstrap-1.md)
  - описывает все стадии от включения компьютера до запуска первой инструкции
  ядра;
* [Первые шаги в коде настройки ядра](https://github.com/olshevskiy87/linux-insides-ru/blob/master/Booting/linux-bootstrap-2.md)
  - описывает первые шаги в исходном коде настройки ядра. Вы ознакомитесь с
  инициализацией кучи, запросы различных параметров, таких как EDD, IST и др.
* [Инициализация видео-режима и переход в защищённый режим](https://github.com/olshevskiy87/linux-insides-ru/blob/master/Booting/linux-bootstrap-3.md)
  - описывает инициализацию видео-режима в коде настройки ядра и переход к
  защищённому режиму.
* [Переход к 64-битному режиму](https://github.com/olshevskiy87/linux-insides-ru/blob/master/Booting/linux-bootstrap-4.md)
  - описывает подготовку к переходу в 64-битный режим и детали перехода.
* [Распаковка ядра](https://github.com/olshevskiy87/linux-insides-ru/blob/master/Booting/linux-bootstrap-5.md)
  - описывает подготовку перед распаковкой ядра и детали самой распаковки.
