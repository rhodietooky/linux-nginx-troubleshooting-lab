# Linux Nginx Troubleshooting Lab

Учебный проект по базовой диагностике nginx на Ubuntu Server.

В рамках проекта был настроен nginx с несколькими виртуальными хостами, локальными доменными именами и отдельными директориями сайтов. После базовой настройки были разобраны типовые инциденты: ошибка в конфигурации, ошибка прав доступа и неправильный путь `root`.

## Цель проекта

Отработать практические навыки диагностики nginx и Linux-сервисов:

- проверка состояния сервиса через `systemctl`;
- проверка конфигурации через `nginx -t`;
- анализ логов через `journalctl`;
- анализ `access.log` и `error.log`;
- проверка HTTP-ответов через `curl`;
- диагностика ошибок `403 Forbidden` и `404 Not Found`;
- понимание связки `server_name`, `root`, `index.html` и `hosts`.

## Среда и инструменты

- Ubuntu Server
- nginx
- systemd
- curl
- Linux permissions
- Windows hosts file

## Настроенные сайты

На одном nginx были настроены три локальных сайта:

| Домен | Директория сайта |
|---|---|
| `rhodesian-lab.test` | `/var/www/site01` |
| `rhodesian-support.test` | `/var/www/site02` |
| `rhodesian-dev.test` | `/var/www/site03` |

Для локального доступа с Windows домены были добавлены в файл `hosts`:

```text
192.168.1.139 rhodesian-lab.test
192.168.1.139 rhodesian-support.test
192.168.1.139 rhodesian-dev.test
```

## Структура nginx на Ubuntu

Основные директории и файлы, использованные в проекте:

Главный конфигурационный файл nginx.

```text
/etc/nginx/nginx.conf
```

Директория с доступными конфигурациями сайтов.

```text
/etc/nginx/sites-available/
```

Директория с включёнными конфигурациями сайтов. Обычно содержит символические ссылки на файлы из `sites-available`.

```text
/etc/nginx/sites-enabled/
```

Директория с файлами сайтов.

```text
/var/www/
```

Лог входящих HTTP-запросов.

```text
/var/log/nginx/access.log
```

Лог ошибок nginx.

```text
/var/log/nginx/error.log
```

## Основные команды диагностики

Проверка состояния nginx:

```bash
systemctl status nginx --no-pager -l
```

Проверка конфигурации nginx:

```bash
sudo nginx -t
```

Применение конфигурации без полной остановки сервиса:

```bash
sudo systemctl reload nginx
```

Просмотр логов systemd-unit nginx:

```bash
journalctl -u nginx -n 30 --no-pager
```

Проверка, слушает ли nginx порт 80:

```bash
ss -tulpn | grep :80
```

Проверка HTTP-ответа конкретного virtual host:

```bash
curl -i -H "Host: rhodesian-lab.test" http://127.0.0.1/
```

Просмотр access log:

```bash
tail -n 20 /var/log/nginx/access.log
```

Просмотр error log:

```bash
tail -n 20 /var/log/nginx/error.log
```

## Инциденты

### 1. Ошибка в конфигурации nginx

Была намеренно допущена синтаксическая ошибка в конфиге nginx. После этого `nginx -t` показал ошибку, а сервис не смог корректно применить конфигурацию.

Использованные команды:

```bash
sudo nginx -t
systemctl status nginx --no-pager -l
journalctl -u nginx -n 30 --no-pager
```

Вывод:

- `nginx -t` помогает быстро найти синтаксические ошибки в конфигурации;
- перед `reload` или `restart` nginx нужно всегда проверять конфиг;
- `systemctl status` показывает текущее состояние сервиса;
- `journalctl -u nginx` помогает посмотреть подробные логи службы.

### 2. Ошибка прав доступа — 403 Forbidden

У файла сайта были изменены права так, что nginx не мог прочитать `index.html`.

Пример поломки:

```bash
sudo chmod 000 /var/www/site01/index.html
```

Симптом:

```text
HTTP/1.1 403 Forbidden
```

Диагностика:

```bash
curl -i -H "Host: rhodesian-lab.test" http://127.0.0.1/
tail -n 20 /var/log/nginx/error.log
ls -l /var/www/site01/index.html
ls -ld /var/www/site01
```

Исправление:

```bash
sudo chmod 644 /var/www/site01/index.html
```

Вывод:

- `403 Forbidden` часто связан с правами доступа;
- nginx может видеть файл, но не иметь прав его прочитать;
- `error.log` помогает подтвердить ошибку `permission denied`;
- не стоит исправлять права через `chmod 777` без необходимости.

### 3. Неправильный root/path — 404 Not Found

В конфиге nginx был указан неправильный путь к директории сайта.

Было:

```nginx
root /var/www/site01;
```

Стало для тестовой поломки:

```nginx
root /var/www/site001;
```

Симптом:

```text
HTTP/1.1 404 Not Found
```

Диагностика:

```bash
sudo nginx -t
curl -i -H "Host: rhodesian-lab.test" http://127.0.0.1/
tail -n 20 /var/log/nginx/access.log
tail -n 20 /var/log/nginx/error.log
grep -n "root\|server_name\|listen" /etc/nginx/sites-enabled/site01.conf
ls -ld /var/www/site001
ls -l /var/www/site01/index.html
```

Вывод:

- `nginx -t` проверяет синтаксис, но не всегда ловит логические ошибки;
- если `root` указывает на неправильную директорию, nginx может вернуть `404`;
- `404 Not Found` означает, что нужный файл или путь не найден;
- важно отличать `403` от `404`.

## Разница между 403 и 404

```text
403 Forbidden
Файл существует, но nginx не имеет прав его прочитать.

404 Not Found
Файл или путь не найден.
```

## Что я отработал

В этом проекте я отработал:

- установку и базовую проверку nginx;
- настройку virtual hosts;
- работу с `server_name`;
- работу с `root` и `index.html`;
- включение сайтов через `sites-enabled`;
- проверку конфигурации через `nginx -t`;
- диагностику сервиса через `systemctl`;
