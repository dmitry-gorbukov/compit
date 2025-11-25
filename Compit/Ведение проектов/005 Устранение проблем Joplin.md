
Нашел проблему! В вашем docker-compose файле отсутствуют критически важные настройки:

## 1. Проблемы в текущем файле:

- **Нет проброса портов** - контейнер не доступен с хоста
- **Нет указания порта приложения** - Joplin не знает на каком порту работать
- **Неправильное имя сервиса БД** - должно быть `joplin-db`, а не `postgres`

## 2. Исправленный docker-compose.yml

Замените содержимое файла `/opt/docker-services/joplin/docker-compose.yml`:

```yaml
version: '3.8'

services:
  joplin-db:
    image: postgres:13
    container_name: joplin-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: joplin
      POSTGRES_PASSWORD: joplin
      POSTGRES_DB: joplin
    volumes:
      - ./db:/var/lib/postgresql/data
    nenaontworks:
      - joplin-net

  joplin-server:
    image: joplin/server:latest
    container_name: joplin-server
    restart: unless-stopped
    environment:
      APP_PORT: 22300
      APP_BASE_URL: http://joplin.localhost
      DB_CLIENT: postgres
      POSTGRES_DATABASE: joplin
      POSTGRES_USER: joplin
      POSTGRES_PASSWORD: joplin
      POSTGRES_PORT: 5432
      POSTGRES_HOST: joplin-db
    ports:
      - "22300:22300"
    volumes:
      - ./data:/var/lib/joplin
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.joplin.rule=Host(`joplin.localhost`)"
      - "traefik.http.routers.joplin.entrypoints=web"
      - "traefik.http.services.joplin.loadbalancer.server.port=22300"
    depends_on:
      - joplin-db
    networks:
      - web
      - joplin-net

networks:
  web:
    external: true
  joplin-net:
    external: false
```

## 3. Применяем исправления

```bash
# Переходим в директорию
cd /opt/docker-services/joplin

# Переименуем старый файл если нужно
mv compose.yml docker-compose.yml

# Останавливаем и удаляем старые контейнеры
docker compose down

# Запускаем с новой конфигурацией
docker compose up -d

# Проверяем
docker ps | grep joplin
docker port joplin-server

# Ждем запуска и проверяем
sleep 10
curl -v http://localhost:22300/
```

## 4. Проверяем логи

```bash
# Смотрим логи после перезапуска
docker logs joplin-server --tail 20
```

## 5. Обновляем systemd сервис

Если использовали systemd, перезапустим:

```bash
sudo systemctl restart joplin.service
sudo systemctl status joplin.service
```

## 6. Проверка в браузере

После успешного запуска:
- **Прямой доступ**: http://localhost:22300
- **Через Traefik**: http://joplin.localhost

## Ключевые изменения:

1. ✅ **Добавлен `APP_PORT: 22300`** - Joplin знает на каком порту работать
2. ✅ **Добавлен `ports: - "22300:22300"`** - проброс порта на хост
3. ✅ **Исправлено имя БД** - `joplin-db` вместо `postgres`
4. ✅ **Добавлена метка порта для Traefik** - `loadbalancer.server.port=22300`
5. ✅ **Раздельные сети** - для лучшей изоляции

**Выполните команды из пункта 3** - это должно решить проблему с доступностью Joplin.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTMyODQzODY0MiwtMTU1OTUzODgxMl19
-->