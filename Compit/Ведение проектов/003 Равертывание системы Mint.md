
Отличный выбор! Давайте развернем эту связку через Docker Compose. Это будет максимально чисто и переносимо.

## 1. Структура проекта

Создадим общую структуру для всех сервисов:

```bash
mkdir ~/my-workspace
cd ~/my-workspace
mkdir wiki-js openproject joplin traefik
```

## 2. Traefik для reverse proxy (опционально, но рекомендуется)

`~/my-workspace/traefik/docker-compose.yml`:
```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    command:
      - --api.dashboard=true
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - web

networks:
  web:
    external: false
```

## 3. Wiki.js — База знаний

`~/my-workspace/wiki-js/docker-compose.yml`:
```yaml
version: '3.8'

services:
  wiki:
    image: ghcr.io/requarks/wiki:2
    container_name: wiki-js
    environment:
      DB_TYPE: postgres
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: wikijsrocks
      DB_NAME: wiki
    volumes:
      - ./data:/var/wiki/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wiki.rule=Host(`wiki.localhost`)"
      - "traefik.http.routers.wiki.entrypoints=web"
    depends_on:
      - postgres
    networks:
      - web
      - internal

  postgres:
    image: postgres:15
    container_name: wiki-postgres
    environment:
      POSTGRES_DB: wiki
      POSTGRES_USER: wikijs
      POSTGRES_PASSWORD: wikijsrocks
    volumes:
      - ./db:/var/lib/postgresql/data
    networks:
      - internal

networks:
  web:
    external: true
  internal:
    external: false
```

## 4. OpenProject — Управление проектами

`~/my-workspace/openproject/docker-compose.yml`:
```yaml
version: '3.8'

services:
  openproject:
    image: openproject/community:12
    container_name: openproject
    environment:
      OPENPROJECT_SECRET_KEY_BASE: secret
      OPENPROJECT_HOST__NAME: openproject.localhost
      OPENPROJECT_HTTPS: "false"
    volumes:
      - ./data:/var/openproject/assets
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.openproject.rule=Host(`openproject.localhost`)"
      - "traefik.http.routers.openproject.entrypoints=web"
    networks:
      - web
      - internal
    depends_on:
      - postgres

  postgres:
    image: postgres:13
    container_name: openproject-postgres
    environment:
      POSTGRES_USER: openproject
      POSTGRES_PASSWORD: openproject
      POSTGRES_DB: openproject
    volumes:
      - ./db:/var/lib/postgresql/data
    networks:
      - internal

networks:
  web:
    external: true
  internal:
    external: false
```

## 5. Joplin Server — Синхронизация заметок

`~/my-workspace/joplin/docker-compose.yml`:
```yaml
version: '3.8'

services:
  joplin:
    image: joplin/server:latest
    container_name: joplin-server
    environment:
      APP_BASE_URL: http://joplin.localhost
      DB_CLIENT: postgres
      POSTGRES_DATABASE: joplin
      POSTGRES_USER: joplin
      POSTGRES_PASSWORD: joplin
      POSTGRES_PORT: 5432
      POSTGRES_HOST: postgres
    volumes:
      - ./data:/var/lib/joplin
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.joplin.rule=Host(`joplin.localhost`)"
      - "traefik.http.routers.joplin.entrypoints=web"
    depends_on:
      - postgres
    networks:
      - web
      - internal

  postgres:
    image: postgres:13
    container_name: joplin-postgres
    environment:
      POSTGRES_USER: joplin
      POSTGRES_PASSWORD: joplin
      POSTGRES_DB: joplin
    volumes:
      - ./db:/var/lib/postgresql/data
    networks:
      - internal

networks:
  web:
    external: true
  internal:
    external: false
```

## 6. Общий docker-compose.yml (альтернативный вариант)

Если хотите всё в одном файле `~/my-workspace/docker-compose.yml`:

```yaml
version: '3.8'

services:
  # Traefik
  traefik:
    image: traefik:v2.10
    container_name: traefik
    command:
      - --api.dashboard=true
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  # Wiki.js
  wiki:
    image: ghcr.io/requarks/wiki:2
    container_name: wiki-js
    environment:
      DB_TYPE: postgres
      DB_HOST: wiki-db
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: wikijsrocks
      DB_NAME: wiki
    volumes:
      - ./wiki-js/data:/var/wiki/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wiki.rule=Host(`wiki.localhost`)"
    depends_on:
      - wiki-db

  wiki-db:
    image: postgres:15
    container_name: wiki-postgres
    environment:
      POSTGRES_DB: wiki
      POSTGRES_USER: wikijs
      POSTGRES_PASSWORD: wikijsrocks
    volumes:
      - ./wiki-js/db:/var/lib/postgresql/data

  # OpenProject
  openproject:
    image: openproject/community:12
    container_name: openproject
    environment:
      OPENPROJECT_SECRET_KEY_BASE: secret
      OPENPROJECT_HOST__NAME: openproject.localhost
      OPENPROJECT_HTTPS: "false"
    volumes:
      - ./openproject/data:/var/openproject/assets
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.openproject.rule=Host(`openproject.localhost`)"
    depends_on:
      - openproject-db

  openproject-db:
    image: postgres:13
    container_name: openproject-postgres
    environment:
      POSTGRES_USER: openproject
      POSTGRES_PASSWORD: openproject
      POSTGRES_DB: openproject
    volumes:
      - ./openproject/db:/var/lib/postgresql/data

networks:
  default:
    name: web
```

## Запуск системы

### Способ 1: Раздельные сервисы (рекомендуется)
```bash
# Создаем сеть
docker network create web

# Запускаем Traefik
cd ~/my-workspace/traefik
docker compose up -d

# Запускаем Wiki.js
cd ~/my-workspace/wiki-js
docker compose up -d

# Запускаем OpenProject
cd ~/my-workspace/openproject
docker compose up -d

# Запускаем Joplin Server
cd ~/my-workspace/joplin
docker compose up -d
```

### Способ 2: Всё в одном (если выбрали общий файл)
```bash
cd ~/my-workspace
docker compose up -d
```

## Настройка hosts файла

Добавьте в `/etc/hosts`:
```
127.0.0.1 wiki.localhost
127.0.0.1 openproject.localhost
127.0.0.1 joplin.localhost
```

## Доступ к сервисам

- **Wiki.js**: http://wiki.localhost
- **OpenProject**: http://openproject.localhost
- **Joplin Server**: http://joplin.localhost
- **Traefik Dashboard**: http://localhost:8080

## Рекомендации для Linux Mint

1. **Docker Compose V2** уже установлен с Docker Engine
2. **Используйте `docker compose`** (без дефиса) — это современная версия
3. **Порты 80 и 8080** могут быть заняты — измените в `ports` если нужно
4. **Для продакшена** настройте правильные домены и SSL

## Управление сервисами

```bash
# Просмотр логов
docker compose logs -f

# Остановка
docker compose down

# Перезапуск
docker compose restart

# Обновление образов
docker compose pull
docker compose up -d
```

Эта конфигурация даст вам полноценную экосистему для управления знаниями и проектами!
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwNTg2NDQ1MzNdfQ==
-->