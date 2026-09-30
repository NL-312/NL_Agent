# Техническое задание: удалённый Telegram-агент с MCP

## 1. Цель проекта

Создать Telegram-бота, через которого пользователь сможет удалённо работать с MCP-сервером и Telegram-аккаунтом без Claude Desktop и без зависимости от конкретного компьютера.

Бот, MCP-сервер и фоновые процессы размещаются на VPS. Пользователь взаимодействует с системой через Telegram.

## 2. Целевая схема

```text
Пользователь Telegram
        |
        v
Telegram Bot API
        |
        v
Bot Gateway на VPS
        |
        +--> Claude API / агентный оркестратор
        |          |
        |          v
        |      MCP-клиент
        |          |
        |          v
        |      MCP read/actions server
        |          |
        |          v
        |      Telegram user session
        |
        +--> аудит, лимиты, approvals, idempotency
```

## 3. Обязательный результат

Система должна позволять авторизованному пользователю отправить боту запрос на естественном языке, получить ответ на основе MCP-инструментов и, при явном подтверждении, выполнить разрешённое действие в Telegram.

Примеры запросов:

- «Покажи новые сообщения в группе за последние 24 часа»;
- «Найди участников с username, содержащим `example`»;
- «Подготовь сообщение для группы X»;
- «Отправь подготовленное сообщение» — только после отдельного подтверждения и прохождения политики.

## 4. Объём первой версии (MVP)

### 4.1 Telegram-бот

- long polling или webhook; для MVP использовать long polling;
- обработка команд `/start`, `/help`, `/status`, `/cancel`;
- приём текстовых сообщений;
- ответы с ограничением длины Telegram-сообщения и разбиением длинных результатов;
- whitelist пользователей по Telegram user ID;
- запрет обслуживания неизвестных пользователей;
- ограничение частоты запросов;
- correlation ID для каждого запроса;
- понятные сообщения о статусе: «принято», «выполняется», «готово», «ошибка».

### 4.2 Agent Gateway

Gateway должен:

1. принять сообщение от Telegram;
2. проверить пользователя и лимиты;
3. создать контекст диалога;
4. передать запрос агенту Claude;
5. предоставить агенту список MCP-инструментов;
6. выполнить только разрешённые MCP-вызовы;
7. вернуть пользователю итоговый ответ;
8. сохранить метаданные операции без секретов.

На первом этапе допускается один последовательный запрос на пользователя. Параллельные MCP-вызовы должны быть явно разрешены только для read-only операций.

### 4.3 MCP

MCP read-сервер должен использовать существующие инструменты чтения, включая:

- `tg_auth_status`;
- `tg_list_sessions`;
- `tg_get_my_dialogs`;
- `tg_get_group_info`;
- `tg_get_messages`;
- `tg_get_messages_since`;
- `tg_search_messages`;
- `tg_get_participants`;
- `tg_search_participants`;
- `tg_get_stats`.

Actions-сервер подключается отдельным процессом и отдельной Telegram-сессией. Read и actions не должны использовать один SQLite session-файл.

## 5. Политика действий

Все действия, изменяющие Telegram, должны выполняться только через actions MCP и существующие защитные механизмы проекта.

Для каждого write-действия обязательно:

1. сформировать dry-run с целью, чатом, текстом и параметрами;
2. показать пользователю краткое резюме;
3. получить отдельное явное подтверждение;
4. передать `approval_code` из dry-run;
5. передать `confirm=true` и требуемый `confirmation_text`;
6. записать результат и idempotency key.

Бот не должен:

- отправлять сообщения по одному лишь намерению пользователя без подтверждения;
- выполнять действия для чатов вне allowlist;
- обходить rate limit, anti-spam или write guard;
- принимать команды от пользователей вне whitelist;
- показывать секреты, API hash, bot token, session paths и approval data в полном виде.

## 6. Авторизация и безопасность

### 6.1 Секреты

Секреты хранятся только в environment/secret provider на VPS:

- `TELEGRAM_BOT_TOKEN`;
- `ANTHROPIC_API_KEY`;
- `TG_API_ID`;
- `TG_API_HASH`;
- пути к session-файлам;
- whitelist Telegram user IDs и групп.

Секреты не передаются через Telegram и не записываются в логи.

### 6.2 Доступ пользователей

Минимальная конфигурация:

```text
BOT_ALLOWED_USER_IDS=123456789,987654321
TG_ACTIONS_ALLOWED_GROUPS=-1001234567890,example_group
```

Дополнительно рекомендуется проверять username только как вспомогательный атрибут; основным идентификатором должен быть numeric Telegram user ID.

### 6.3 Изоляция процессов

- bot gateway запускается отдельным Unix-пользователем;
- MCP read и actions запускаются отдельными процессами;
- session-файлы имеют права `600`;
- каталог сессий имеет права `700`;
- write actions не выполняются из bot gateway напрямую;
- для production использовать systemd с `Restart=on-failure`, `NoNewPrivileges=yes` и ограничением доступа к файловой системе.

## 7. Модель данных

Для MVP допускается SQLite.

Минимальные сущности:

### `users`

- `telegram_user_id` — primary key;
- `username`;
- `enabled`;
- `created_at`;
- `updated_at`.

### `conversations`

- `id`;
- `telegram_user_id`;
- `status`;
- `created_at`;
- `updated_at`.

### `requests`

- `id` / correlation ID;
- `telegram_user_id`;
- `conversation_id`;
- `request_text_hash`;
- `status`;
- `started_at`;
- `finished_at`;
- `error_code`.

### `approvals`

- `approval_code_hash`;
- `telegram_user_id`;
- `action_type`;
- `target`;
- `expires_at`;
- `used_at`.

В базу нельзя сохранять полные bot token, API hash, приватные ключи или содержимое секретных переменных.

## 8. Конфигурация

Добавить `.env.sample` с параметрами:

```dotenv
TELEGRAM_BOT_TOKEN=
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=
BOT_ALLOWED_USER_IDS=
BOT_ADMIN_USER_IDS=
BOT_MAX_REQUESTS_PER_MINUTE=10
BOT_REQUEST_TIMEOUT_SEC=180
BOT_MAX_REPLY_CHARS=3500
BOT_ENABLE_ACTIONS=0
MCP_READ_COMMAND=/usr/local/bin/tgmcp-read
MCP_ACTIONS_COMMAND=/usr/local/bin/tgmcp-actions
BOT_AUDIT_FILE=data/logs/bot_audit.jsonl
```

Production defaults:

```dotenv
BOT_ENABLE_ACTIONS=0
TG_ACTIONS_REQUIRE_ALLOWLIST=1
TG_ACTIONS_REQUIRE_CONFIRMATION_TEXT=1
TG_ACTIONS_REQUIRE_APPROVAL_CODE=1
TG_BLOCK_DIRECT_TELETHON_WRITE=1
```

## 9. Обработка ошибок

Пользователь должен получать безопасное сообщение без traceback и секретов.

Коды ошибок:

- `AUTH_REQUIRED` — Telegram-сессия не авторизована;
- `ACCESS_DENIED` — пользователь не в whitelist;
- `RATE_LIMITED` — превышен лимит;
- `MCP_UNAVAILABLE` — MCP-процесс недоступен;
- `TELEGRAM_FLOOD_WAIT` — Telegram ограничил запросы;
- `APPROVAL_REQUIRED` — нужно подтверждение;
- `ACTION_EXPIRED` — approval истёк;
- `INTERNAL_ERROR` — внутренняя ошибка.

Технические детали пишутся в серверный лог с correlation ID.

## 10. Наблюдаемость

Обязательные поля аудита:

- timestamp;
- correlation ID;
- Telegram user ID;
- тип операции;
- MCP tool name;
- target без лишних персональных данных;
- статус;
- длительность;
- error code.

Не логировать:

- bot token;
- Anthropic API key;
- TG API hash;
- приватные SSH-ключи;
- полные тексты личных сообщений без отдельной необходимости.

## 11. Развёртывание

### Этап 1 — read-only

- установить bot gateway на VPS;
- настроить `.env`;
- подключить `tgmcp-read` через stdio;
- авторизовать Telegram session;
- добавить systemd unit;
- проверить `/status`, `tg_auth_status`, чтение диалогов;
- отключить actions.

### Этап 2 — approvals

- добавить dry-run/approval flow;
- хранить approval state;
- добавить `/cancel`;
- покрыть idempotency и TTL;
- провести тесты без реальной отправки сообщений.

### Этап 3 — controlled actions

- подключить `tgmcp-actions`;
- настроить allowlist групп;
- включить actions только для admin whitelist;
- провести тестовую отправку в специально созданной группе;
- проверить аудит и повторный запуск.

## 12. Тестирование

Обязательные тесты:

- неизвестный пользователь получает отказ;
- разрешённый пользователь получает `/status`;
- Telegram API timeout корректно обрабатывается;
- MCP process restart не теряет запрос навсегда;
- длинный ответ разбивается корректно;
- markdown/HTML из результата безопасно экранируется;
- read tool не может вызвать write operation;
- write без approval отклоняется;
- неправильный approval code отклоняется;
- просроченный approval отклоняется;
- повторный write блокируется idempotency;
- секреты отсутствуют в логах;
- одновременно работающие read/actions не делят одну session SQLite.

Команды проверки проекта:

```bash
PYTHONPATH=tganalytics:. python -m pytest tests/ -q
python scripts/check_anti_spam_compliance.py
python scripts/check_env.py
```

## 13. Критерии приёмки

Работа считается выполненной, если:

1. бот работает на VPS без Claude Desktop на пользовательском компьютере;
2. разрешённый пользователь может получить статус Telegram-сессии;
3. read-запросы вызывают MCP и возвращают корректный ответ;
4. неизвестный пользователь не может вызвать MCP;
5. write-действия отключены по умолчанию;
6. write-действия проходят dry-run, approval и allowlist;
7. после перезапуска VPS бот и MCP восстанавливаются автоматически;
8. секреты не попадают в репозиторий и логи;
9. есть тесты на авторизацию, MCP timeout, approval и idempotency;
10. имеется инструкция развёртывания и отката.

## 14. Вне первой версии

- голосовые сообщения;
- обработка изображений и документов;
- несколько независимых Telegram-аккаунтов на одного пользователя;
- веб-панель администратора;
- публичный HTTP MCP endpoint;
- автоматическая рассылка без ручного approval;
- обучение модели на истории переписки.

