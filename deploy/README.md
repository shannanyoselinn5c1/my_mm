# prof-medics-chat — Mattermost deploy (Caddy + Let's Encrypt)

Развёртывание Mattermost (team-edition) + Postgres + Caddy по регламенту `depoly_req.txt`.
TLS терминируется внутри Caddy через автоматический Let's Encrypt: при первом запросе на
`https://${MM_DOMAIN}` Caddy сам получает и далее обновляет сертификат.

## Структура

```
deploy/
  docker-compose.yml
  Dockerfile           # образ mattermost-caddy (target: mattermost-caddy)
  .env.example
  README.md
caddy/
  Caddyfile            # копируется внутрь mattermost-caddy
volumes/               # bind mounts (создаются на этапе бутстрапа)
```

## Бутстрап (выполняется один раз перед первым запуском)

1. Скопировать `.env.example` в `.env` и заполнить значения:

   ```
   cp deploy/.env.example deploy/.env
   ${EDITOR:-nano} deploy/.env
   ```

   - `POSTGRES_PASSWORD` — сгенерировать через `openssl rand -base64 24`.
   - `MM_DOMAIN` — полное имя без протокола, например `chat.example.com`. Должно резолвиться в публичный IP сервера (см. ниже про DNS).
   - `ACME_EMAIL` — email для уведомлений Let's Encrypt о просрочке (используется только LE, наружу не публикуется).

2. Создать директории под bind mounts с правами для UID `nobody` (65534):

   ```
   sudo mkdir -p volumes/postgres/data \
                 volumes/mattermost/{config,data,logs,plugins,client-plugins,bleve-indexes} \
                 volumes/caddy/{data,config}
   sudo chown -R 65534:65534 volumes
   sudo chmod 700 volumes/postgres/data
   ```

3. Подготовить DNS: A-запись `${MM_DOMAIN}` → публичный IP сервера. Проверка с локальной машины:

   ```
   dig +short @1.1.1.1 chat.example.com
   ```

   Перед запуском compose эта запись **уже должна резолвиться** — иначе Let's Encrypt не пройдёт HTTP-01-валидацию.

4. Открыть в фаерволе VM порты `80/tcp`, `443/tcp`, `443/udp` (HTTP/3) на публичном интерфейсе.
   Для cloud-провайдеров — security group / firewall rules, для bare-metal — `ufw`/`nftables`.

5. Собрать кастомный образ Caddy (директива `build` запрещена п. 4.3 — собираем отдельно).
   Команда из корня репозитория:

   ```
   docker build \
     --tag mattermost-caddy:local \
     --target mattermost-caddy \
     --file deploy/Dockerfile .
   ```

   Образ остаётся в локальном кэше docker, compose использует его по тегу `mattermost-caddy:local`.
   Если позднее понадобится реестр — переименовать тег в формат `hub.xtra.vg/prof-medics-chat/mattermost-caddy`
   (п. 4.2 регламента) и `docker push`.

## Запуск

```
docker compose -f deploy/docker-compose.yml up -d
docker compose -f deploy/docker-compose.yml ps
docker compose -f deploy/docker-compose.yml logs -f caddy
```

В логах Caddy при первом старте увидишь строки `obtaining certificate` и затем
`certificate obtained successfully` — значит, LE отработал. После этого
`https://${MM_DOMAIN}` отвечает рабочим сертификатом.

## TLS

Caddy автоматически:
- получает сертификат при первом обращении к домену (HTTP-01 challenge на 80-м порту);
- обновляет сертификат за ~30 дней до истечения;
- хранит сертификат и состояние ACME в `./volumes/caddy/data` — этот каталог **обязательно**
  должен переживать перезапуски, иначе при каждом рестарте будет новый запрос к LE и риск
  упереться в rate limit (5 сертификатов на домен в неделю).

`MM_SERVICESETTINGS_SITEURL` собирается из `MM_DOMAIN` автоматически (`https://${MM_DOMAIN}`),
правки в env для этого не нужны.

## Сетевая модель

- `db` (internal) — postgres + mattermost. Изолирован от интернета.
- `web` (internal) — mattermost + caddy. Внутренняя связь между reverse proxy и приложением.
- `mail` (internal) — mattermost + postfix. SMTP для email-уведомлений и инвайтов.
- `egress` — caddy + mattermost + postfix. Caddy: ACME (Let's Encrypt) и OCSP.
  Mattermost: push-уведомления. Postfix: исходящая отправка email.
- Marketplace-плагины, link previews и diagnostics в Mattermost отключены.

## SMTP / Email

Для отправки email-уведомлений и инвайтов используется собственный Postfix (send-only).
Postfix автоматически генерирует DKIM-ключи при первом запуске.

### DNS-записи (обязательно)

В панели Timeweb Cloud → DNS для домена `prof-medics.ru`:

| Тип | Имя | Значение |
|---|---|---|
| A | `mail` | `<IP сервера>` |
| TXT | `@` | `v=spf1 a a:mail.prof-medics.ru ~all` |
| TXT | `mail._domainkey` | `v=DKIM1; k=rsa; p=<ключ из контейнера>` |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; rua=mailto:admin@prof-medics.ru` |

PTR-запись (Reverse DNS): настраивается в Timeweb Cloud → Cloud Серверы → Сеть.

### Получение DKIM-ключа

```bash
docker compose -f deploy/docker-compose.yml exec postfix \
  cat /etc/opendkim/keys/prof-medics.ru/mail.txt
```

## Бэкап

База:

```
docker compose -f deploy/docker-compose.yml exec -T postgres \
  pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" | gzip > backups/db-$(date +%F).sql.gz
```

Файлы Mattermost (вложения, конфиг) и сертификаты Caddy:

```
sudo tar czf backups/mm-files-$(date +%F).tar.gz \
    volumes/mattermost/data volumes/mattermost/config volumes/caddy/data
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

| сервис     | cpus | mem   |
|------------|------|-------|
| postgres   | 0.75 | 1g    |
| mattermost | 1.0  | 1g    |
| caddy      | 0.25 | 128m  |
| **итого**  | 2.0  | ~2.1g |

Запас по CPU и RAM — для ОС, бэкапов, всплесков. Лимиты в пределах регламента (≤1 на сервис, ≤3 суммарно).

## Известные отклонения от регламента

1. **Caddy публикует порты на `0.0.0.0`** (80/tcp, 443/tcp, 443/udp), а не на `127.0.0.1`,
   как требует п. 4.12. Без публичного 80 не работает HTTP-01 challenge у Let's Encrypt;
   без публичного 443 чат недоступен снаружи. Требует согласования исключения по п. 1.5.
   Альтернатива — переключиться на DNS-01 challenge с API-токеном DNS-провайдера, тогда
   80 порт можно убрать (443 всё равно нужен).

2. **У Caddy `cap_drop` без `ALL`**: оставлен `NET_BIND_SERVICE`, чтобы процесс под
   `user: nobody` мог открыть порты 80/443. Все остальные дефолтные capabilities
   отброшены явным списком.

3. **Caddy подключён к `egress`-сети с интернетом** — необходимо для запросов к
   `acme-v02.api.letsencrypt.org` и OCSP. Postgres и Mattermost интернет не получают.

4. **Postgres запускается под штатным `user: "999:999"`** (а не `nobody`), без `read_only: true`,
   `cap_drop: ALL` и `no-new-privileges`. Связка строгого регламента с официальным
   postgres-образом на практике вызывает повторяющиеся проблемы с правами на bind mount
   и `initdb chmod`. Отступление от пп. 4.6, 4.7, 4.17 — согласовывается по п. 1.5.
   Mattermost и Caddy ужесточения сохраняют.

5. **Caddy под `user: nobody` + `no-new-privileges=true`**: бинарь Caddy в alpine-образе
   имеет file capability `cap_net_bind_service=+ep`, поэтому может слушать 80/443 от
   непривилегированного пользователя. Если в конкретном ядре эта связка ломается,
   фолбэк — `sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80` на хосте
   (с записью в `/etc/sysctl.d/99-caddy.conf` для постоянства).

6. **Postfix запускается без `read_only: true`**: образ `boky/postfix` при старте генерирует
   конфигурацию Postfix и OpenDKIM в `/etc/postfix`, `/etc/opendkim`, `/var/lib/postfix` и
   `/var/run`. Монтирование всех этих путей через tmpfs ненадёжно и зависит от внутренней
   структуры образа. Остальные ужесточения сохранены: `no-new-privileges`, `cap_drop: ALL`,
   `cap_add: NET_BIND_SERVICE`, лимиты ресурсов. Контейнер подключён к `mail` (internal)
   для связи с Mattermost и к `egress` для исходящей отправки.
   TODO: при переходе на production-образ рассмотреть `read_only: true` с полным набором tmpfs.