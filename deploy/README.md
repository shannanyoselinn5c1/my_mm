# prof-medics-chat — Mattermost deploy (Caddy + Let's Encrypt)

Развёртывание Mattermost (team-edition) + Postgres + Caddy + Postfix по регламенту `deploy_req.txt`.
TLS терминируется внутри Caddy через автоматический Let's Encrypt: при первом запросе на
`https://${MM_DOMAIN}` Caddy сам получает и далее обновляет сертификат.

## Структура

```
deploy/
  docker-compose.yml
  Dockerfile           # образ mattermost-caddy (target: mattermost-caddy)
  .env / .env.example
  README.md
caddy/
  Caddyfile            # копируется внутрь mattermost-caddy
.dockerignore          # минимальный, контекст сборки = корень репо (п. 8)
.deploy/volumes/       # bind mounts на хосте (создаются на бутстрапе, п. 4.8)
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

2. Создать директории под bind mounts (в `.deploy/volumes`, п. 4.8) с правами для UID `nobody` (65534):

   ```
   sudo mkdir -p .deploy/volumes/postgres/data \
                 .deploy/volumes/mattermost/{config,data,logs,plugins,client-plugins,bleve-indexes} \
                 .deploy/volumes/caddy/{data,config} \
                 .deploy/volumes/postfix/dkim
   sudo chown -R 65534:65534 .deploy/volumes
   sudo chmod 700 .deploy/volumes/postgres/data
   ```

   Postgres теперь работает под `user: nobody`, поэтому каталог его данных должен
   принадлежать UID 65534 (`chown` выше) — иначе `initdb` при первом старте упадёт на правах.

3. Подготовить DNS: A-запись `${MM_DOMAIN}` → публичный IP сервера. Проверка с локальной машины:

   ```
   dig +short @1.1.1.1 chat.example.com
   ```

   Перед запуском compose эта запись **уже должна резолвиться** — иначе Let's Encrypt не пройдёт HTTP-01-валидацию.

4. Открыть в фаерволе VM порты `80/tcp`, `443/tcp`, `443/udp` (HTTP/3) на публичном интерфейсе.
   Для cloud-провайдеров — security group / firewall rules, для bare-metal — `ufw`/`nftables`.

5. Собрать кастомный образ Caddy (директива `build` запрещена п. 4.2 — собираем отдельно).
   Команда из корня репозитория (контекст сборки = корень, п. 3.3):

   ```
   docker build \
     --tag hub.xtra.vg/prof-medics-chat/mattermost-caddy:2.8.4 \
     --target mattermost-caddy \
     --file deploy/Dockerfile .
   ```

   Имя образа уже в формате `hub.xtra.vg/{slug}/{image-name}` (п. 4.2). Реестра `hub.xtra.vg`
   пока нет, поэтому образ остаётся в локальном кэше docker и compose берёт его оттуда по этому
   тегу; `docker push` отложен до появления реестра (временное отклонение, см. «Известные отклонения»).

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
- хранит сертификат и состояние ACME в `./.deploy/volumes/caddy/data` — этот каталог **обязательно**
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
    .deploy/volumes/mattermost/data .deploy/volumes/mattermost/config .deploy/volumes/caddy/data
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

| сервис     | cpus | mem    |
|------------|------|--------|
| postgres   | 0.75 | 1g     |
| mattermost | 1.0  | 1g     |
| caddy      | 0.25 | 128m   |
| postfix    | 0.25 | 128m   |
| **итого**  | 2.25 | ~2.26g |

Запас по CPU и RAM — для ОС, бэкапов, всплесков. Лимиты в пределах регламента
(≤1 на сервис, ≤3 суммарно — пп. 4.14, 4.15). `mem_limit` = `memswap_limit` у всех сервисов,
чтобы исключить своп.

## Известные отклонения от регламента (согласовываются по п. 1.5)

1. **Caddy публикует порты на `0.0.0.0`** (80/tcp, 443/tcp, 443/udp), а не на `127.0.0.1`
   (п. 4.12). Caddy — интернет-фронт: без публичного 80 не работает HTTP-01 challenge
   Let's Encrypt, без публичного 443 чат недоступен снаружи. Альтернатива на будущее —
   DNS-01 challenge с API-токеном DNS-провайдера, тогда 80 можно убрать (443 всё равно нужен).

2. **У Caddy `cap_drop` без `ALL`**: оставлен `NET_BIND_SERVICE`, чтобы процесс под
   `user: nobody` мог открыть порты 80/443; остальные capabilities отброшены явным списком
   (отступление от духа п. 4.16, но безопаснее, чем полный набор). Если в конкретном ядре
   связка `nobody` + `no-new-privileges` всё же ломается на привязке портов — фолбэк
   `sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80` (с записью в `/etc/sysctl.d/99-caddy.conf`).

3. **Образ Caddy собирается и хранится локально**, `docker push` отложен — реестра `hub.xtra.vg`
   пока нет. Формат имени образа соблюдён (`hub.xtra.vg/prof-medics-chat/mattermost-caddy`, п. 4.2).
   Временное отклонение до появления реестра.

4. **Postfix: root + `cap_add` + без `read_only`** (пп. 4.6, 4.7, 4.19). Это свойство
   архитектуры Postfix (демон `master` стартует под root, слушает привилегированные порты,
   раздаёт работу процессам под разными uid) и образа `boky/postfix` (генерирует конфиг и
   DKIM-ключи на старте в `/etc/postfix`, `/etc/opendkim`, `/var/lib/postfix`, `/var/run`;
   монтировать всё это через tmpfs хрупко и зависит от версии образа).
   **Почему риск приемлем:** контейнер postfix **не публикует порты на хост** (нет `ports:`) —
   из интернета напрямую недоступен; **не подключён к сети `db`** — доступа к Postgres и
   перепискам у него нет; достижим только латерально из уже скомпрометированного Mattermost
   по internal-сети `mail`. Компенсирующие меры: `cap_drop: ALL` + минимальный `cap_add`,
   `no-new-privileges`, запиненный (`boky/postfix:v5.1.0`) и регулярно обновляемый образ, лимиты ресурсов.
   Альтернатива на будущее — внешний SMTP-relay вместо self-hosted Postfix, что убрало бы это отклонение.

**Приведены к норме:** Postgres, Mattermost и Caddy запускаются под `user: nobody`,
`read_only: true`, `security_opt: no-new-privileges`, `cap_drop: ALL` (у Caddy — с явным
списком, см. п. 2). Bind mounts вынесены в `.deploy/volumes` (п. 4.8), внешние образы
запинены (п. 4.2), сети сегментированы: `db`/`web`/`mail` — `internal`, интернет (`egress`)
только у Caddy, Mattermost и Postfix.

### Следствия `read_only` у Mattermost

- Прекомпилированные плагины отключены: `MM_PLUGINSETTINGS_AUTOMATICPREPACKAGEDPLUGINS=false`
  **плюс** каталог `/mattermost/prepackaged_plugins` перекрыт пустым `tmpfs`. Одного флага мало:
  Mattermost распаковывает ~20 встроенных плагинов (boards, calls, jira…) в `/tmp` ещё до проверки
  флага и упирается в размер tmpfs (`no space left on device`), а раздувать `/tmp` нельзя
  (п. 4.18 ограничивает tmpfs 512 МБ). Пустой `tmpfs` поверх каталога убирает сами бандлы —
  распаковывать нечего. Плагины этому деплою не нужны (marketplace и загрузка тоже выключены).
- В логе при старте возможна строка `Failed to update root.html ... read-only file system` — это
  попытка переписать статику под subpath на read-only ФС. При корневом `SITEURL`
  (`https://${MM_DOMAIN}`, наш случай) ошибка безвредна.