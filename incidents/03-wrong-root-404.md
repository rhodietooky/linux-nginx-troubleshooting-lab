# Инцидент 3: неправильный root/path — 404 Not Found

## Цель

Проверить ситуацию, когда nginx запущен, конфигурация синтаксически корректная, но сайт не открывается, потому что в конфиге указан неправильный путь к директории сайта.

## Что было сломано

В конфиге сайта:

```text
/etc/nginx/sites-available/site01.conf
```

была изменена директива `root`.

Было:

```nginx
root /var/www/site01;
```

Стало для тестовой поломки:

```nginx
root /var/www/site001;
```

То есть nginx начал искать файлы сайта в неправильной директории.

## Симптом

После применения конфигурации nginx вернул ошибку:

```text
HTTP/1.1 404 Not Found
```

Проверка выполнялась командой:

```bash
curl -i -H "Host: rhodesian-lab.test" http://127.0.0.1/
```

## Диагностика

Для диагностики использовались команды:

Проверка синтаксиса конфигурации nginx.

```bash
sudo nginx -t
```

Проверка HTTP-ответа конкретного virtual host.

```bash
curl -i -H "Host: rhodesian-lab.test" http://127.0.0.1/
```

Просмотр входящих запросов и HTTP-статусов.

```bash
tail -n 20 /var/log/nginx/access.log
```

Просмотр ошибок nginx.

```bash
tail -n 20 /var/log/nginx/error.log
```

Проверка активного конфига сайта: порт, `server_name` и `root`.

```bash
grep -n "root\|server_name\|listen" /etc/nginx/sites-enabled/site01.conf
```

Проверка, существует ли ошибочная директория.

```bash
ls -ld /var/www/site001
```

Проверка, существует ли правильный файл сайта.

```bash
ls -l /var/www/site01/index.html
```

## Важное наблюдение

Команда:

```bash
sudo nginx -t
```

могла показать:

```text
syntax is ok
test is successful
```

Это нормально, потому что с точки зрения синтаксиса строка:

```nginx
root /var/www/site001;
```

написана корректно.

Но логически путь неправильный: в этой директории нет нужного `index.html`.

## Причина

Причина была в неправильной директиве `root`.

Nginx пытался найти файл:

```text
/var/www/site001/index.html
```

Но правильный файл находился здесь:

```text
/var/www/site01/index.html
```

Из-за этого nginx возвращал:

```text
404 Not Found
```

## Исправление

В конфиге был возвращён правильный путь:

```nginx
root /var/www/site01;
```

После этого конфигурация была проверена:

```bash
sudo nginx -t
```

И применена:

```bash
sudo systemctl reload nginx
```

## Проверка после исправления

После исправления был выполнен запрос:

```bash
curl -i -H "Host: rhodesian-lab.test" http://127.0.0.1/
```

Ожидаемый результат:

```text
HTTP/1.1 200 OK
```

## Разница между 403 и 404

```text
403 Forbidden
Файл существует, но nginx не имеет прав его прочитать.

404 Not Found
Файл или путь не найден.
```

В этом инциденте была ошибка `404`, потому что nginx искал файл в неправильной директории.

## Вывод

Ошибка `404 Not Found` может быть связана не только с отсутствием файла, но и с неправильным путём в директиве `root`.

Важно понимать, что `nginx -t` проверяет синтаксис конфигурации, но не всегда выявляет логические ошибки вроде неправильного пути к директории сайта.

Для диагностики полезны команды:

```bash
sudo nginx -t
curl -i -H "Host: rhodesian-lab.test" http://127.0.0.1/
tail -n 20 /var/log/nginx/access.log
tail -n 20 /var/log/nginx/error.log
grep -n "root\|server_name\|listen" /etc/nginx/sites-enabled/site01.conf
ls -ld /var/www/site001
ls -l /var/www/site01/index.html
```
