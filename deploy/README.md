# med-prof-chat — Mattermost deploy

Развёртывание Mattermost (team-edition) + Postgres + Nginx по регламенту `depoly_req.txt`.
TLS терминируется на уровне облачного балансировщика; внутри контейнеров — обычный HTTP.

## Структура

```
deploy/
  docker-compose.yml
  Dockerfile           # образ mattermost-nginx (target: mattermost-nginx)
  .env.example
  README.md
nginx/
  nginx.conf           # копируется внутрь mattermost-nginx
volumes/               # bind mounts (создаются на этапе бутстрапа)
```

## Бутстрап (выполняется один раз перед первым запуском)

1. Скопировать `.env.example` в `.env` и заполнить значения:

   ```
   cp deploy/.env.example deploy/.env
   ${EDITOR:-nano} deploy/.env
   ```

   Сгенерировать пароль для БД, например: `openssl rand -base64 24`.

2. Создать директории под bind mounts с правами для UID `nobody` (65534), как требует п. 4.6 регламента (`user: nobody`):

   ```
   sudo mkdir -p volumes/postgres/data \
                 volumes/mattermost/{config,data,logs,plugins,client-plugins,bleve-indexes}
   sudo chown -R 65534:65534 volumes
   sudo chmod 700 volumes/postgres/data
   ```

3. Собрать кастомный образ nginx (директива `build` в compose запрещена п. 4.3 — образ собирается отдельно). Команда из корня репозитория:

   ```
   docker build \
     --tag hub.xtra.vg/med-prof-chat/mattermost-nginx \
     --target mattermost-nginx \
     --file deploy/Dockerfile .
   ```

   Если используется реестр — `docker push hub.xtra.vg/med-prof-chat/mattermost-nginx`.
   Если без реестра — образ останется в локальном кэше docker, compose использует его.

## Запуск

```
docker compose -f deploy/docker-compose.yml up -d
docker compose -f deploy/docker-compose.yml ps
docker compose -f deploy/docker-compose.yml logs -f mattermost
```

После старта Mattermost доступен по адресу `http://127.0.0.1:8064` на хосте.

## TLS и публикация наружу

Порт `8064` опубликован только на `127.0.0.1` (требование п. 4.12). Облачный балансировщик
должен терминировать TLS и проксировать внешний 443 на хостовый `127.0.0.1:8064`.
Если выбранный тариф/балансировщик ходит к VM по приватному/публичному IP, а не по loopback —
понадобится либо локальный туннель (например, cloudflared, tailscale, ssh -R), либо отдельное
согласование исключения по п. 1.5 на публикацию nginx на внешнем интерфейсе.

`MM_SERVICESETTINGS_SITEURL` в `.env` обязательно должен совпадать с публичным URL,
иначе сломаются OAuth-редиректы и WebSocket.

## Сетевая модель

- `db` (internal) — postgres + mattermost. У postgres нет интернета.
- `web` (internal) — mattermost + nginx. У nginx нет исходящего интернета (входящий трафик
  приходит через опубликованный порт, а не через docker network).
- Push-уведомления, marketplace-плагины, link previews и diagnostics отключены — для них
  потребовался бы исходящий интернет.

## Бэкап

База:

```
docker compose -f deploy/docker-compose.yml exec -T postgres \
  pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" | gzip > backups/db-$(date +%F).sql.gz
```

Файлы Mattermost (вложения, конфиг):

```
sudo tar czf backups/mm-files-$(date +%F).tar.gz volumes/mattermost/data volumes/mattermost/config
```

Восстановление БД (на пустую новую установку, до первого старта mattermost):

```
gunzip -c backups/db-YYYY-MM-DD.sql.gz | \
  docker compose -f deploy/docker-compose.yml exec -T postgres \
  psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

## Обновление

1. Поднять тег `image:` сервиса в `docker-compose.yml` (зафиксированный, п. 4.2).
2. Сделать бэкап (см. выше).
3. `docker compose -f deploy/docker-compose.yml pull && docker compose -f deploy/docker-compose.yml up -d`.
4. Mattermost сам мигрирует схему БД при старте (п. 6.1).

## Лимиты ресурсов (под сервер 2c × 3.2 ГГц / 4 ГБ / 50 ГБ)

| сервис     | cpus | mem  |
|------------|------|------|
| postgres   | 0.75 | 1g   |
| mattermost | 1.0  | 1g   |
| nginx      | 0.25 | 128m |
| **итого**  | 2.0  | ~2.1g |

Запас по CPU и RAM — для ОС, бэкапов, всплесков. Лимиты в пределах регламента (≤1 на сервис, ≤3 суммарно).

## Известные отклонения от регламента

- Все сервисы запускаются с `cap_drop: ALL` без `cap_add`. Стандартные образы postgres и
  mattermost ориентированы на запуск от собственных пользователей (postgres uid 999,
  mattermost uid 2000). При `user: nobody` контейнеры стартуют без chown — поэтому
  обязательна предварительная подготовка прав на `volumes/` (см. бутстрап п. 2).
  Если возникнут проблемы со стартом — согласовать исключение по п. 1.5 либо вернуть
  стандартного пользователя сервиса (`user: "999"` для postgres, `user: "2000"` для mattermost).
