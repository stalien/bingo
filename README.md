# Итоговое домашнее задание под кодовым названием "bingo"

~~Сочиненька на тему "как я провел этим летом"~~

1. Читаю ТЗ ))
2. Скачиваю бинарь на локальную Mac Os, чтобы понять как его запустить
3. Для того чтобы определить на каком из ~~мертвых~~ языков скомпилен файл, применяю ~~черную магию~~ софтину DIE-engine (https://github.com/horsicq/DIE-engine). Бинарь написан на Go - ну штош •͡˘㇁•͡˘
4. Поднимаю в Parallels виртуалку с Ubuntu 20.04
5. Скачиваю бинарь на виртуалку wget https://storage.yandexcloud.net/final-homework/bingo
6. Даю права

    ```console
    chmod +x bingo
    ```

7. Пробую запустить

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
8. Пробую запустить 2

```console
./bingo run_server
```

    -- panic: failed to read config data

Ругается на то что не может прочитать конфигу, чтобы выяснить подробности воспользуемся strace:

```console
strace -e file ./bingo run_server
```

    -- openat(AT_FDCWD, "/opt/bingo/config.yaml", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)

как будто по пути "/opt/bingo/config.yaml" должен лежать ямлик с конфигой, а его там нетути - исправляем

```console
sudo mkdir /opt/bingo && \
sudo nano /opt/bingo/config.yaml
```

сама структура нужного нам конфига очевидно возвращается командой print_default_config, смотрим

```console
./bingo print_default_config
```

```console
student_email: test@example.com
postgres_cluster:
  hosts:
  - address: localhost
    port: 5432
  user: postgres
  password: postgres
  db_name: postgres
  ssl_mode: disable
  use_closest_node: false
```

в config.yaml заполняем student_email, остальные параметры оставляем без изменения ( да, я люблю рисковать используя некриптостойкие пароли к db ╭(ʘ̆~◞౪◟~ʘ̆)╮)

проверяем что с конфигом все норм:

```console
/bingo print_current_config
```

9. Накатываем postgresql, задаем пароль дефолтному пользователю

```console
sudo apt install postgresql
```

```console
sudo service postgresql start
```

```console
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'postgres';"
```

заполняем базу, попросив приложение командой prepare_db

```console
./bingo prepare_db
```

идем пить чай, минут так на 20 __( •͡˘ _•͡˘)ノ c[_]

10. Пробуем запустить 3

```console
./bingo run_server
```

    -- panic: failed to build logger

чето паника, смотрим стрейсом

```console
strace -e file ./bingo run_server
```

```console
openat(AT_FDCWD, "/opt/bongo/logs/fe589a1270/main.log", O_WRONLY|O_CREAT|O_APPEND|O_CLOEXEC, 0666) = -1 ENOENT (No such file or directory)
```

создаем директорию "/opt/bongo/logs/fe589a1270/"

```console
sudo mkdir -m 777 -p /opt/bongo/logs/fe589a1270
```

еще разок

```console
./bingo run_server
```

```console
My congratulations.
You were able to start the server.
Here's a secret code that confirms that you did it.
--------------------------------------------------
code:         yoohoo_server_launched
--------------------------------------------------
```

Юхуууу! Готово!! (нет)

11. Пробуем постучаться на helthcheck ручку /ping

```console
curl http://localhost:80/ping
```

    -- curl: (7) Failed to connect to localhost port 80 after 0 ms: Connection refused

Не свезло, ищем на каком порту работает приложенька

ставим тулзу 

```console
sudo apt install net-tools
```

смотрим активность

```console
sudo netstat -tunlp
```

```console
tcp6   0  0 :::24575        :::*            LISTEN      1616/bingo
```

tcp что?? ок, пробуем порт 24575

```console
curl http://localhost:24575/ping
```

```console
pong
```

( ╥﹏╥) ノシ

12. Инфра. Пора развернуть приложеньку на нормальном окружении, и скормить ее Пете )))

И раз уж у нас есть доступ в облако - грех этим не воспользоваться. Поскольку по ТЗ приложение должно иметь 2 ноды (плюс отдельно еще Postgres), буду разворачивать ручками - под каждую ноду своя VM (без контейнеризации), ну и машинка под базу. Итого: 2 VM под инстансы сервиса, в одной целевой группе и сверху Network Load Balancer, 1 VM под базу (база отдельно от приложенек чтобы вдруг чего не вышло). Конфига машинок одинаковая (Intel Cascade Lake. 20% vCPU — прерываемые ВМ, Intel Cascade Lake. RAM 2 Гб, Стандартное сетевое хранилище (HDD) 30 Гб), самый минимум (5%, 1 Гб) брать не стал - есть сомнения что потянет нагрузку.

