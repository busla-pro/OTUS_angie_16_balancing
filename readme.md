# ДЗ урока № 16 — Балансировка нагрузки (HTTP)

ДЗ будем делать размещая в Docker и балансировщик, и бэкэнд-хосты. IP хоста с Docker - `192.168.129.131`

Структура каталогов:
* `upstreams` - общие для всех стендов/сценариев/вариантов концигурации backend-серверов
* `round-robin` - docker compose и http-конфиг angie для варианта "Равномерная балансировка (round-robin)"
* `hash` - docker compose и http-конфиг angie для варианта "Балансировка по хэшу с использованием переменных (на выбор)"
* `random` - docker compose и http-конфиг angie для варианта "Произвольная балансировка (random)"
* `backup` - docker compose и http-конфиг angie для варианта с резервным бэкэнд серрвером

1. Продготовим 4 минималистичных web-сервера возвращающих одну из строк:
    * red
    * blue
    * yellow
    * green

    и `uri` запроса.

    Соответвующим образом назовём конфигурационные файлы и контейнеры, разместим их в каталоге `upstreams`
2. стенд балансировки по варианту "Равномерная балансировка (round-robin)" разместим в каталоге `round-robin`
    1. в конфиге angie в директиве `upstream` просто перечислим 4 web-сервера - по умолчанию используется `round-robin`:
        ```
        upstream backend {
            server backend-red:80;
            server backend-blue:80;
            server backend-yellow:80;
            server backend-green:80;
        }
        ```
    2. в `docker-compose.yml` запустим все пять контейнеров, наружу опубликуем только tcp-порт 80 баланировщика.
    3. сделаем снаружи 10 запросов:
        ```powershell
        1..10 | % { Invoke-webRequest -UseBasicParsing -Uri 'http://192.168.129.131/' } | % Content
        ```
        тестовый стенд возвращает результат:
        ```
        red ( / )
        blue ( / )
        yellow ( / )
        green ( / )
        red ( / )
        blue ( / )
        yellow ( / )
        green ( / )
        red ( / )
        blue ( / )
        ```
        Ответы приходят в том же порядке, в каком web-серверы указаны в конфиге балансировщика и повторяются по кругу.
3. стенд балансировки по варианту "Балансировка по хэшу с использованием переменных (на выбор)" разместим в каталоге `hash`
    1. возьмём конфигурацию стенда `round-robin`, в директиву `upstream` добавим балансировку по хэшу от встроенной переменной `$request_uri`:
        ```
        hash $request_uri;
        ```
    2. сделаем снаружи 10 запросов с разным `uri`:
        ```powershell
        1..10 | % { Invoke-webRequest -UseBasicParsing -Uri "http://192.168.129.131/$_.html" } | % Content
        ```
        запросы примерно равномерно распределились по бэкэндам:
        ```
        green ( /1.html )
        green ( /2.html )
        green ( /3.html )
        yellow ( /4.html )
        yellow ( /5.html )
        yellow ( /6.html )
        yellow ( /7.html )
        red ( /8.html )
        red ( /9.html )
        yellow ( /10.html )
        ```
        и при повторных запросах распределяются идентичным образом независимо от порядка и количества запросов:
        ```powershell
        10..1 | % { Invoke-webRequest -UseBasicParsing -Uri "http://192.168.129.131/$_.html" } | % Content
        ```
        ```
        yellow ( /10.html )
        red ( /9.html )
        red ( /8.html )
        yellow ( /7.html )
        yellow ( /6.html )
        yellow ( /5.html )
        yellow ( /4.html )
        green ( /3.html )
        green ( /2.html )
        green ( /1.html )
        ```
4. стенд балансировки по варианту "Произвольная балансировка (random)" разместим в каталоге `random`
    1. возьмём конфигурацию стенда `round-robin`, в директиву `upstream` добавим случайную балансировку:
        ```
        random;
        ```
    2. сделаем снаружи 10 запросов с разным `uri`:
        ```powershell
        1..10 | % { Invoke-webRequest -UseBasicParsing -Uri "http://192.168.129.131/$_.html" } | % Content
        ```
        запросы примерно равномерно распределились по бэкэндам:
        ```
        yellow ( /1.html )
        red ( /2.html )
        green ( /3.html )
        blue ( /4.html )
        red ( /5.html )
        blue ( /6.html )
        blue ( /7.html )
        green ( /8.html )
        yellow ( /9.html )
        green ( /10.html )
        ```
        повторные запросы дают похожее равномерное распределение, но каждый раз разное:
        ```
        green ( /1.html )
        red ( /2.html )
        blue ( /3.html )
        red ( /4.html )
        green ( /5.html )
        red ( /6.html )
        blue ( /7.html )
        yellow ( /8.html )
        blue ( /9.html )
        yellow ( /10.html )
        ```
5. стенд балансировки по варианту "резервный бэкэнд с отключением одного из бэкэндов" разместим в каталоге `backup`
    1. возьмём конфигурацию стенда `round-robin`, в директиве `upstream` двум серверам пропишем `backup`
    2. сделаем снаружи 10 запросов с разным `uri`:
    ```
    1..10 | % { Invoke-webRequest -UseBasicParsing -Uri "http://192.168.129.131/$_.html" } | % Content
    ```
    запросы по кругу распределяются по двум основным серверам:
    ```
    red ( /1.html )
    blue ( /2.html )
    red ( /3.html )
    blue ( /4.html )
    red ( /5.html )
    blue ( /6.html )
    red ( /7.html )
    blue ( /8.html )
    red ( /9.html )
    blue ( /10.html )
    ```
    3. остановим контейнер `backend-red` и повторим 10 запросов - все их обработал один оставшийся основной сервер:
    ```
    blue ( /1.html )
    blue ( /2.html )
    blue ( /3.html )
    blue ( /4.html )
    blue ( /5.html )
    blue ( /6.html )
    blue ( /7.html )
    blue ( /8.html )
    blue ( /9.html )
    blue ( /10.html )
    ```
    4. остановим контейнер `backend-blue` и повторим 10 запросов
    ```
    yellow ( /1.html )
    green ( /2.html )
    yellow ( /3.html )
    green ( /4.html )
    yellow ( /5.html )
    green ( /6.html )
    yellow ( /7.html )
    green ( /8.html )
    yellow ( /9.html )
    green ( /10.html )
    ```
    При выходе из строя всех основных бэкэндов трафик начал распределяться распределилися на все резервные.