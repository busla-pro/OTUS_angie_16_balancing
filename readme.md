# ДЗ урока № 16 — Балансировка нагрузки (HTTP)

ДЗ будем делать размещая в Docker и балансировщик, и бэкэнд-хосты. IP хоста с Docker - `192.168.129.131`

1. Продготовим 4 минималистичных web-сервера возвращающих одну из строк:
    * red
    * blue
    * yellow
    * green

    И `uri` запроса.

    Соответвующим образом назовём конфигурационные файлы и контейнеры, разместим их в каталоге `upstreams`
2. стенд балансировки по варианту "Равномерная балансировка (round-robin)" разместим в каталоге `round-robin`
    1. в конфиге angie в директиве `upstream backend` просто перечислим 4 web-сервера - по умолчанию используется `round-robin`:
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
    1. возьмём конфигурацию стенда `round-robin`, в в директиву `upstream backend` добавим балансировку по хэшу от встроенной переменной `$request_uri`:
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