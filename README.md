# Итоговое домашнее задание под кодовым названием "bingo"

~~Сочиненька на тему "как я провел этим летом"~~

1. Читаю ТЗ ))
2. Скачиваю бинарь на локальную Mac Os, чтобы понять как его запустить
3. Для того чтобы определить на каком из ~~мертвых~~ языков скомпилен файл, применяю ~~черную магию~~ софтину DIE-engine (https://github.com/horsicq/DIE-engine). Бинарь написан на Go - ну штош •͡˘㇁•͡˘
4. Поднимаю локально в Parallels виртуалку с Ubuntu 20.04
5. Скачиваю бинарь на виртуалку wget https://storage.yandexcloud.net/final-homework/bingo
6. Даю права

    ```console
    chmod +x bingo
    ```

7. Пробую запустить

    ```console
    ./bingo
    ```

    > Hello world

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

    > panic: failed to read config data

    Ругается на то что не может прочитать конфигу, чтобы выяснить подробности воспользуемся strace:

    ```console
    strace -e file ./bingo run_server
    ```

    > openat(AT_FDCWD, "/opt/bingo/config.yaml", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)

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

    идем пить чай, минут так на 20 __( •͡˘ _•͡˘)ノ 

10. Пробуем запустить 3

    ```console
    ./bingo run_server
    ```

    > panic: failed to build logger

    чето паника, смотрим стрейсом

    ```console
    strace -e file ./bingo run_server
    ```

    > openat(AT_FDCWD, "/opt/bongo/logs/fe589a1270/main.log", O_WRONLY|O_CREAT|O_APPEND|O_CLOEXEC, 0666) = -1 ENOENT (No such file or directory)

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

    > curl: (7) Failed to connect to localhost port 80 after 0 ms: Connection refused

    Нуу такое, ищем на каком порту работает приложенька:

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

    > pong

    ( ╥﹏╥) ノシ

12. Инфра. Пора развернуть приложеньку на нормальном окружении, и скормить ее Пете 

    И раз уж у нас есть доступ в облако - грех этим не воспользоваться. Поскольку по ТЗ приложение должно иметь 2 ноды (плюс отдельно еще Postgres), буду разворачивать ручками - под каждую ноду своя VM (без контейнеризации), ну и машинка под базу. Итого: 2 VM под инстансы сервиса, в одной целевой группе и сверху Network Load Balancer, 1 VM под базу (база отдельно от приложенек чтобы вдруг чего не вышло). Конфига машинок одинаковая (Intel Cascade Lake. 20% vCPU — прерываемые ВМ, Intel Cascade Lake. RAM 2 Гб, Стандартное сетевое хранилище (HDD) 30 Гб), самый минимум (5%, 1 Гб) брать не стал - есть сомнения что потянет нагрузку.

    Конфига балансера

    > Тип   HTTP \
    Путь    /ping \
    Порт    24575 \
    Время ожидания, c   1 \
    Интервал, c     3 \
    Порог работоспособности     2 \
    Порог неработоспособности   3

    Обработчик

    > listener-f46cd-de7	158.160.130.53	80	24575	TCP

    Повторяю те же действие по разворачиванию приложения как и на локальной машине, с той лишь разницей что БД теперь находится на удаленном хосте и требует некоторых донастроек чтобы приложенька до нее достучалась.

    ```console
    sudo nano /etc/postgresql/14/main/pg_hba.conf
    ```

    добавляем внутренние ip адреса машинок с которых планируем обращаться к базе в конфигу

    ```console
    # IPv4 local connections:
    host    postgres        postgres        10.128.0.1/32          md5
    ```

13. Откалибруем часы

    ```console
    sudo apt install ntpdate
    ```

    ```console
    ntpdate -q ntp.ubuntu.com
    ```

14. Добавим прилу в PATH

    ```console
    sudo cp ./bingo /usr/local/bin/bingo
    ```

15. Отдал потестить Пете

    Нууу, могло быть и лучше. Выявленные проблемы: Приложение периодически отваливается с криками `panic: we're all gonna die` или `killed`, а также после тестовой нагрузки стабильно и необратимо начинает пятисотить с отмазой `i feel bad` (после перезапуска проходит). Приложение долго стартует, следовательно и перезапускается тоже слишком долго

16. Ускорение запуска

    Снифаем что происходит при запуске

    ```console
    sudo netstat -tunpc
    ```

    ```console
    tcp        0      1 10.128.0.12:46826       8.8.8.8:80              SYN_SENT    2949/bingo
    ```

    Бинга шлет запросы на 8.8.8.8:80, но шлет их без уважения 

    Нужно починить порт, заменив 80 на 53

    ```console
    sudo iptables -t nat -A OUTPUT -p tcp -d 8.8.8.8 --dport 80 -j DNAT --to-destination 8.8.8.8:53
    ```

    Закрепим это правило, чтобы оно не исчезало при перезагрузке машинки

    ```console
    sudo apt install iptables-persistent
    ```

    ```console
    sudo su -c "iptables-save > /etc/iptables/rules.v4"
    ```

    получаем еще один сикрет код

    ```console
    Congratulations.
    You were able to figure out why
    the application is slow to start and fix it.
    Here's a secret code that confirms that you did it.
    --------------------------------------------------
    code:         google_dns_is_not_http
    --------------------------------------------------
    ```

17. Процесс мониторинг

    Задача - научиться как-то отслеживать падения приложения и автоматически поднимать его обратно (ведь быстро поднятое упавшим не считается), и второе - постоянно чекать статус ответа /ping и в случае статуса отличного от 200 прибивать процесс (который потом сам автоматически поднимется).

    После некоторого количества проб и ошибок в поисках рабочего решения я остановился на опенсорсной тулзе monit https://bitbucket.org/tildeslash/monit/src/master/
    
    Устанавливаем

    ```console
    sudo apt install monit
    ```
    
    ```console
    sudo systemctl enable monit
    ```

    Настраиваем

    ```console
    sudo nano /etc/monit/monitrc
    ```

    ```console
    set daemon  5
    with start delay 120
    ```

    каждые 5 сек демон чекает стейты сервисов, первоначальная задержка при загрузке машинки 120 сек, чтобы все успело прогрузиться

    ```console
    nano /home/stalien/start-app.sh
    ```

    ```console
    #!/bin/bash
    bingo run_server
    ```

    ```console
    sudo nano /etc/monit/conf-available/bingo
    ```

    ```console
    check process bingo
    matching "bingo"
    start program = "/bin/su -s /bin/bash -c '/home/stalien/start-app.sh' stalien" with timeout 5 seconds
    stop program = "/usr/bin/killall bingo" with timeout 3 seconds
    if failed host localhost port 24575 with protocol http and request "/ping" with timeout 5 seconds th>
    if 5 restarts with 5 cycles then timeout
    ```

    2 условия: если нет процесса с именем bingo - запустить bingo (таймаут 5 сек, чтобы не моросило), если запрос localhost:24575/ping возвращает не 200 - убить, убить, убить !!! (таймаут 3 сек)

    ```console
    sudo monit -t
    ```

    ```console
    sudo ln -s /etc/monit/conf-available/bingo /etc/monit/conf-enabled/
    ```
    
    ```console
    sudo systemctl reload monit
    ```

18. Оптимизация запросов к БД

    По некоторым из запросов ответы не укладываются в тайминги обозначеные в тз. Например GET /api/customer/{id}, проверим план запроса

    ```console
    EXPLAIN (ANALYZE)
    SELECT id, name, surname, birthday, email
    FROM customers
    WHERE id IN(4);
    ```

    Создадим простейшие индексы на столбцы id, одно это значительно ускорит выполнение запросов

    ```console
    CREATE INDEX idx_customers_id ON public.customers (id); 
    ```
    
    ```console
    CREATE INDEX idx_movies_id ON public.movies (id); 
    ```

    ```console
    CREATE INDEX idx_sessions_id ON public.sessions (id); 
    ```

    19. Кеширование GET /long_dummy

    Затрудняюсь точно сформулировать как я пришел к этой конфигурации, но запросы которые приходят от балансировщика - обрабатываются HAProxy на 8080 порту, если запрос дергает GET /long_dummy (который должен кэшироваться) HAProxy его отправляет на 8081 порт, который в свою очередь слушает nginx и уже он выступает в роли кешируешего прокси, остальные запросы соответственно пробрасываются на порты bingo. 
    
    *Изначально HAProxy должен был справляться с кешированием сам, но почемуто не завелся по своей же документации, хотя может это я недостаточно криворук.

    ставим и настраиваем nginx под кеширование

    ```console
    sudo apt update
    sudo apt install nginx
    sudo systemctl enable nginx
    sudo mkdir -p /var/nginx/cache
    sudo nano /etc/nginx/nginx.conf
    ```

    добавляем конфиг в секцию http

    ```console
        proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m use_temp_path=off;

        server {
        listen 8081;

        location /long_dummy {
            proxy_pass http://127.0.0.1:24575;

            proxy_cache my_cache;
            proxy_cache_valid 200 1m;
            proxy_cache_key "$scheme$request_method$host$request_uri";

            proxy_cache_use_stale error timeout invalid_header updating http_500 http_502 http_503 http_504;

            proxy_ignore_headers Cache-Control;
            add_header X-Proxy-Cache $upstream_cache_status;
        }
        }

    ```

    проверяем и применяем конфиг

    ```console
    sudo nginx -t
    sudo service nginx reload
    sudo service nginx restart
    ```

    ставим и настаиваем haproxy

    ```console
    sudo apt install haproxy
    sudo nano /etc/haproxy/haproxy.cfg
    ```

    ```console
    frontend fe_api
        bind :8080
        use_backend nginx if { path_beg /long_dummy }
        default_backend be_api

    backend nginx
        balance source
        server node1 10.128.0.19:8081 check
        server node2 10.128.0.8:8081 check

    backend be_api
        # Get from cache / put in cache
        # http-request cache-use mycache if { path_beg /long_dummy }
        # http-response cache-store mycache

        # server list
        balance leastconn
        server node1 10.128.0.19:24575 check
        server node2 10.128.0.8:24575 check

    cache mycache
        total-max-size 128      # MB
        max-object-size 1000000 # bytes
        max-age 60              # seconds
    ```

    проверяем и применяем конфиг

    ```console
    sudo haproxy -c -f /etc/haproxy/haproxy.cfg
    sudo service haproxy restart
    ```

    20. Мониторинг (Prometheus + Grafana)

    Haproxy сам умеет в Prometheus, поэтому настроить его было довольно просто https://www.haproxy.com/blog/haproxy-exposes-a-prometheus-metrics-endpoint

    добавляем нужный блок в конфиг

    ```console
    sudo nano /etc/haproxy/haproxy.cfg
    ```

    ```console
    frontend stats
        bind *:8404
        option http-use-htx
        http-request use-service prometheus-exporter if { path /metrics }
        stats enable
        stats uri /stats
        stats refresh 10s
    ```

    Ставлю и настраиваю prometheus на той же VM где и база данных, поскольку вероятность что эта виртуалка уйдет в отказ гораздо меньше

    ```console
    sudo apt install prometheus 
    ```

    ```console
    sudo nano /etc/prometheus/prometheus.yml
    ```

    добавляем адреса наблюдаемых машинок

    ```console
    - job_name: node2
        # If prometheus-node-exporter is installed, grab stats about the local
        # machine by default.
        static_configs:
        - targets: ['10.128.0.8:8404']

    - job_name: node1
        # If prometheus-node-exporter is installed, grab stats about the local
        # machine by default.
        static_configs:
        - targets: ['10.128.0.19:8404']
    ```

    Запускаем сервис

    ```console
    sudo systemctl start prometheus
    ```

    Общепринятой является практика использовать Prometheus в связке с Grafana для интерпритации графического представления метрик, поэтому накатываю Grafana

    ```console
    sudo apt-get install -y adduser libfontconfig1 musl
    wget https://dl.grafana.com/enterprise/release/grafana-enterprise_10.2.2_amd64.deb
    sudo dpkg -i grafana-enterprise_10.2.2_amd64.deb
    ```

    Дальше настраиваю импорт данных из Prometheus и настраиваю дашборд (тут хорошая статейка, чтобы не расписывать лишнего https://losst.pro/ustanovka-i-nastrojka-prometheus). Дашбор выбрал этот https://grafana.com/grafana/dashboards/12693-haproxy-2-full/

    

