изменения в ejabberd.yml можно применять без перезапуска командой:

```bash
docker exec xmpp_ejabberd ejabberdctl reload_config
```
# Конфигурация ejabberd (ejabberd.yml)

Файл конфигурации ejabberd имеет формат YAML и определяет поведение XMPP-сервера.  
В проекте используется Docker-версия ejabberd, поэтому файл либо монтируется из хост-системы, либо редактируется внутри контейнера (не рекомендуется). Обычно он расположен по пути `/etc/ejabberd/ejabberd.yml` внутри контейнера, а снаружи может быть доступен в томах Docker.

Ниже описаны основные секции конфигурации, которые чаще всего требуется изменять под свои нужды.

---

## 1. Основные параметры сервера

### `hosts`
Список доменных имён, которые обслуживает сервер.
```yaml
hosts:
  - "xmpp.example.com"
  - "example.com"
loglevel
Уровень логирования: none | emergency | alert | critical | error | warning | notice | info | debug.

yaml
loglevel: info
```
## 2. Слушающие порты (listen)
Секция listen определяет, на каких интерфейсах и портах ejabberd принимает соединения, и какие протоколы/модули обрабатывают трафик.

В вашей установке большую часть трафика принимает Caddy на порту 443 и перенаправляет XMPP-соединения на внутренний порт ejabberd (обычно 5222 для c2s и 5269 для s2s с TLS). Поэтому в ejabberd слушатели могут быть настроены на локальные порты без внешнего TLS (starttls или direct tls уже обеспечивается Caddy).

Пример типовой настройки:

```yaml
listen:
  - 
    port: 5222
    ip: "::"
    module: ejabberd_c2s
    max_stanza_size: 262144
    shaper: c2s_shaper
    access: c2s
    starttls: false   # TLS уже терминирован upstream, поэтому starttls выключен
    protocol: tcp
  -
    port: 5269
    ip: "::"
    module: ejabberd_s2s_in
  -
    port: 5280
    ip: "::"
    module: ejabberd_http
    request_handlers:
      "/admin": ejabberd_web_admin
      "/api": mod_http_api
    captcha: true
    http_bind: true
    web_admin: true
 ```
Что можно менять:

 - Добавлять/удалять слушателей под свои нужды (например, включить веб-админку на порту 5280, если она не используется за reverse proxy).

 - Включать/отключать STARTTLS, если вы решили терминировать TLS непосредственно в ejabberd (в вашем случае он отключён, так как TLS обрабатывается Caddy).

## 3. Аутентификация и база данных
**auth_method**
Метод аутентификации: internal (встроенная Mnesia), sql (через БД), ldap, external и др.

```yaml
auth_method: sql
sql_type, sql_server, sql_database, sql_username, sql_password
Параметры подключения к базе данных (в вашем проекте — PostgreSQL).

yaml
sql_type: pgsql
sql_server: "postgres"         # имя сервиса в Docker Compose
sql_database: "ejabberd"
sql_username: "ejabberd"
sql_password: "yourpassword"
sql_port: 5432
sql_query_timeout, sql_keepalive_interval
```
Настройки таймаутов и поддержания соединения с БД.

## 4. Модули (modules)
Секция modules содержит список загружаемых модулей с их конфигурацией. Именно здесь включается функционал: хранение офлайн-сообщений, ведение списка комнат, публикация заметок, архивация и т.д.

Пример:

```yaml
modules:
  mod_adhoc: {}
  mod_admin_extra: {}          # дополнительные админские команды
  mod_announce: {}              # объявления
  mod_blocking: {}              # блокировка пользователей (XEP-0191)
  mod_bosh: {}                   # BOSH (если нужен)
  mod_caps: {}
  mod_carboncopy: {}             # копии сообщений (XEP-0280)
  mod_client_state: {}           # управление состоянием клиента
  mod_configure: {}              # on-the-fly конфигурация
  mod_disco: {}                   # service discovery
  mod_echo: {}                    # эхо-тест
  mod_http_api: {}                # HTTP API
  mod_last: {}                    # последняя активность
  mod_mam:                        # архивация сообщений (XEP-0313)
    assume_mam_usage: true
    default: always
  mod_muc:                         # многопользовательские комнаты
    access: muc
    access_admin: muc_admin
    access_create: muc_create
    access_persistent: muc_create
  mod_muc_admin: {}                # административные команды для MUC
  mod_offline:                     # хранение офлайн-сообщений
    access_max_user_messages: max_user_offline_messages
  mod_ping: {}                      # пинг
  mod_privacy: {}                   # списки приватности
  mod_private: {}                   # приватные хранилища XML
  mod_pubsub:                       # публикация-подписка
    access_createnode: pubsub_createnode
    ignore_pep_from_offline: true
    last_item_cache: false
    plugins:
      - flat
      - pep
  mod_push: {}                       # push-уведомления
  mod_push_keepalive: {}             # поддержание сессии для push
  mod_register:                      # регистрация новых пользователей
    access: register
    welcome_message:
      subject: "Welcome!"
      body: "Hi, welcome to this XMPP server."
  mod_roster: {}                      # список контактов
  mod_s2s_dialback: {}                # s2s dialback
  mod_shared_roster: {}               # общие ростеры
  mod_stream_mgmt:                    # управление потоками (XEP-0198)
    cache_size: 1000
    cache_life: 3600
    max_resume_timeout: 300
  mod_vcard: {}                        # vCard
  mod_vcard_xupdate: {}                # обновления vCard
  mod_version: {}                      # версия сервера
```
Что можно менять:

 - Добавлять или удалять модули в зависимости от требуемой функциональности.

 - Настраивать параметры модулей (например, для mod_mam указать срок хранения архива).

## 5. ACL и права доступа (access)
ACL (Access Control Lists) определяют правила доступа к различным функциям сервера.

Пример:

```yaml
acl:
  admin:
    user:
      - "admin@xmpp.example.com"
  local:
    user_regexp: ""
  loopback:
    ip:
      - "127.0.0.0/8"
      - "::1/128"

access:
  max_user_offline_messages:
    admin: infinity
    all: 100
  c2s:
    deny: blocked
    all: allow
  register:
    all: allow
  muc:
    all: allow
  pubsub_createnode:
    all: allow
```
Что можно менять:

Добавлять администраторов, задавать ограничения на количество офлайн-сообщений, определять, кто может создавать комнаты и т.д.

## 6. Shaper (ограничение трафика)
Определяет профили ограничения скорости передачи данных.

```yaml
shaper:
  normal: 1000        # байт/сек
  fast: 50000
  c2s_shaper: fast
  s2s_shaper: normal
```
## 7. TLS (если не вынесен на reverse proxy)
Если TLS терминируется не на Caddy, а непосредственно в ejabberd, то указываются пути к сертификатам в секции listen и глобальные настройки:

```yaml
certfiles:
  - "/etc/ejabberd/certs/server.pem"
  - "/etc/ejabberd/certs/*.pem"

define_macro:
  'TLS': 'tls'
  'CIPHERS': 'HIGH'
  'PROTOCOLS': 'tlsv1.2'
```
В вашем случае эти настройки, скорее всего, не нужны, так как TLS обрабатывает Caddy.

## 8. Интеграция с внешними сервисами (TURN/STUN)
Если вы используете coturn для передачи медиа, укажите настройки в модуле mod_stun_disco (если он включён) или через mod_turn:

```yaml
mod_stun_disco:
  credentials: "turn"
  services:
    - host: "turn.example.com"
      port: 3478
      type: stun
    - host: "turn.example.com"
      port: 5349
      type: turns
```
## 9. Веб-админка и API
Если вы хотите управлять сервером через веб-интерфейс, убедитесь, что в секции listen присутствует обработчик ejabberd_web_admin на порту 5280 (или другом). Также можно включить HTTP API:

```yaml
listen:
  - port: 5280
    module: ejabberd_http
    request_handlers:
      "/admin": ejabberd_web_admin
      "/api": mod_http_api
```
## 10. Специфические для Docker настройки
При использовании Docker важно указать корректные имена сервисов в параметрах подключения к БД (например, sql_server: "postgres"). Также стоит обратить внимание на сетевые интерфейсы — часто используют ip: "::" для приёма подключений на всех интерфейсах.

Где искать и как применить изменения
В вашем проекте файл конфигурации ejabberd, скорее всего, лежит в отдельной папке (например, ./config/ejabberd/ejabberd.yml) и монтируется в контейнер через volumes в docker-compose.yml.

После редактирования файла необходимо перезапустить контейнер ejabberd:

```bash
docker-compose restart ejabberd
```
или, если используете простой docker:

```bash
docker restart ejabberd
```
Проверьте логи на наличие ошибок:

```bash
docker logs ejabberd
```
Примечание: Полный список всех доступных опций и их описание можно найти в официальной документации ejabberd: https://docs.ejabberd.im/admin/configuration/