# ДЗ урока № 16 — Балансировка нагрузки (HTTP)

ДЗ будем делать размещая в Docker и балансировщик, и бэкэнд-хосты. IP хоста с Docker - `192.168.129.131`

1. Продготовим 4 минималистичных web-сервера возвращающих одну из строк:
    * red
    * blue
    * yellow
    * green

    Соответвующим образом назовём конфигурационные файлы и контейнеры, разместим их в каталоге `upstreams`
2. стенд балансировки по варианту `round-robin` разместим в каталоге `round-robin`
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
            red
            blue
            yellow
            green
            red
            blue
            yellow
            green
            red
            blue
        ```
        Ответы приходят в том же порядке, в каком web-серверы указаны в конфиге балансировщика и повторяются по кругу.