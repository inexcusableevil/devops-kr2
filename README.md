# Контрольная работа №2

Тараканов Леонид Валентинович, ЭФБО-11-24. Номер в списке 25, вариант1: api.devops.local, порт8001.

Исходный шаблон: https://gitverse.ru/dgimatdinov/kr2-template, main 3ee6c4b. Рабочая ветка kr2/variant-1. Main сохраняет исходную историю шаблона; Pull Request до защиты не объединяется.

## Исправления

Таблица дефектов: [docs/defects.md](docs/defects.md). Логика app.py и состав requirements.txt не изменены; добавлен обязательный комментарий студента.

Dockerfile использует уже загруженный python:3.12-slim, отдельный слой зависимостей, непривилегированного app, exec CMD и bind0.0.0.0:8001 внутри контейнера. Файл.env исключён из Git и контекста сборки. Пароль, опубликованный в исходном шаблоне, не используется.

## Развёртывание

На Mac запись hosts: `192.168.56.2 api.devops.local`. Все ресурсы работают на локальной учебной VM; внешний хостинг не используется.

Из корня репозитория сначала пробный запуск, затем доставка:

```sh
rsync -avz --delete --exclude=.env --exclude=.git --dry-run ./ devops:kr2/
rsync -avz --delete --exclude=.env --exclude=.git ./ devops:kr2/
```

Только при первоначальной установке на VM, без трассировки команд:

```sh
cd ~/kr2
umask 077
printf 'REDIS_PASSWORD=%s\n' "$(openssl rand -hex 16)" > .env
chmod 600 .env
unset REDIS_PASSWORD
docker compose up -d --build
```

Существующий.env не перезаписывать: он содержит действующий серверный пароль. Не экспортировать REDIS_PASSWORD в рабочей оболочке. Не публиковать вывод docker inspect контейнера Redis и docker compose config: там появляется подставленный пароль.

Redis7alpine запускается официальной точкой входа с командой redis-server, что понижает UID до redis999. Оба сервиса имеют restart:unless-stopped. Сетьbackend, DNSredis, томredis_data в/data; включён AOF. App стартует после PING/PONG. Публикация только127.0.0.1:8001:8001; Redis без ports.

## Nginx и TLS

```sh
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
 -keyout /etc/ssl/private/kr2.key -out /etc/ssl/certs/kr2.crt \
 -subj '/CN=api.devops.local' -addext 'subjectAltName=DNS:api.devops.local'
sudo chown root:root /etc/ssl/private/kr2.key /etc/ssl/certs/kr2.crt
sudo chmod 600 /etc/ssl/private/kr2.key
sudo chmod 644 /etc/ssl/certs/kr2.crt
sudo install -o root -g root -m 644 config/nginx/kr2 /etc/nginx/sites-available/kr2
sudo ln -s /etc/nginx/sites-available/kr2 /etc/nginx/sites-enabled/kr2
sudo nginx -t && sudo systemctl reload nginx
```

Ресурс отдельный, devops-site не меняется. Журналыkr2.access.log/kr2.error.log. На Mac передать только открытый сертификат по проверенному SSH:

```sh
scp devops:/etc/ssl/certs/kr2.crt ~/kr2.crt
curl -4 --cacert ~/kr2.crt https://api.devops.local/
curl -4 --cacert ~/kr2.crt https://api.devops.local/whoami
```

Применение -4 исключает задержки разрешения.local на данном Mac. whoami возвращает scheme:https,client:192.168.56.1,host:api.devops.local. Адрес передан Nginx в X-Real-IP; TCP-соединение приложения приходит от прокси.

## Контроль и восстановление

```sh
docker compose ps
docker compose top
docker compose port app 8001
docker compose port redis 6379
docker compose exec -T app getent hosts redis </dev/null
docker compose exec -T redis redis-cli ping </dev/null
sudo ss -tlnp
sudo ~/devops-course-2026/scripts/audit.sh
```

Расширенный audit.sh находится в scripts/ этого репозитория, на VM установлен в~/devops-course-2026/scripts/audit.sh. Исходные14проверок сохранены, раздел[6] добавляет5проверок КР2. Аудит запускать после sudo-v в интерактивном терминале или весь через sudo при SSH без TTY.

Проверено: UIDобоих999, Redishealthy, NOAUTHбезпароля, .env отсутствует в образе, публикацияRedis:0, app127.0.0.1:8001; счётчик4до down и5после up. Флагdown-v не использовался; именованный том сохранён.

При остановленномapp Nginx отдаёт502 и пишет connect()failed111 к127.0.0.1:8001. Восстановление: docker compose start app, дождаться /health. Перезагрузка и откат для такой диагностики не требуются.

Снимки06-before-kr2 и07-kr2 сохраняются в VirtualBox. UFW неизменён. SHA256devops-site до/после:942253604228ee051ffe7c552f9121564d00f82f79e5c3f94e2b4b8a0c924a91.
