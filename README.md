# Итоговое домашнее задание под кодовым названием "bingo"

~~Сочиненька на тему "как я провел этим летом"~~

1. Читаю ТЗ ))
2. Скачиваю бинарь на локальную Mac Os, чтобы понять как его запустить
3. Для того чтобы определить на каком из ~~мертвых~~ языков скомпилен файл, применяю ~~черную магию~~ софтину DIE-engine (https://github.com/horsicq/DIE-engine). Бинарь написан на Go - ну штош •͡˘㇁•͡˘
4. Поднимаю в Parallels виртуалку с Ubuntu 20.04
5. Скачиваю бинарь на виртуалку wget https://storage.yandexcloud.net/final-homework/bingo
6. Пробую запустить 

```console
./bingo
```

    -- Hello world

хм, попробуем зайти с другой стороны:

```console
man ./bingo
```

нее (╥﹏╥) , а что если:

```console
./bingo --help
```

Бинго! ☉ ‿ ⚆

```console
bingo

Usage:
   [flags]
   [command]

Available Commands:
  completion           Generate the autocompletion script for the specified shell
  help                 Help about any command
  prepare_db           prepare_db
  print_current_config print_current_config
  print_default_config print_default_config
  run_server           run_server
  version              version

Flags:
  -h, --help   help for this command

Use " [command] --help" for more information about a command.
```
7. Пробую запустить 2

```console
./bingo run_server
```

