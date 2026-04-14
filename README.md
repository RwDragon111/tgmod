# tgbridge

`tgbridge` — production-ready server-side Fabric мод для Minecraft `1.21.11`, который пересылает события dedicated-сервера в Telegram forum topics через Telegram Bot API.

Целевой стек:

- Minecraft `1.21.11`
- Fabric Loader `0.18.4`
- Fabric API `0.141.3+1.21.11`
- Java `21`
- Gradle `9.2.0` через включённый wrapper

## Что умеет мод

- Отправляет сообщения в один Telegram supergroup с включёнными forum topics
- Использует один `chat_id` и отдельные `message_thread_id` для разных категорий событий
- Разводит чат, входы/выходы, команды, системные сообщения и игровые события по разным темам
- Использует только серверные события Fabric, без клиентского кода
- Отправляет HTTP-запросы в Telegram асинхронно
- Буферизует исходящие сообщения через внутреннюю очередь
- Повторяет неудачные отправки с exponential backoff
- Применяет простой rate limiting, чтобы снизить спам при бурсте событий
- Экранирует Telegram HTML и использует `parse_mode=HTML`
- Убирает дубли повторяющихся системных сообщений в коротком временном окне
- Поддерживает reload конфига без рестарта сервера
- Пишет принятые к пересылке события в локальный JSONL audit-файл

## Маршрутизация событий

Мод использует следующие Fabric events:

- `ServerMessageEvents.CHAT_MESSAGE`
- `ServerMessageEvents.GAME_MESSAGE`
- `ServerMessageEvents.COMMAND_MESSAGE`
- `ServerPlayConnectionEvents.JOIN`
- `ServerPlayConnectionEvents.DISCONNECT`
- `ServerLifecycleEvents.SERVER_STARTING`
- `ServerLifecycleEvents.SERVER_STARTED`
- `ServerLifecycleEvents.SERVER_STOPPING`
- `ServerLifecycleEvents.SERVER_STOPPED`

Формат сообщений в Telegram:

- chat: `<b>[CHAT]</b> PlayerName: text`
- join: `<b>[JOIN]</b> PlayerName`
- quit: `<b>[QUIT]</b> PlayerName`
- commands: `<b>[CMD]</b> formatted command message`
- events: `<b>[EVENT]</b> text`
- system: `<b>[SYSTEM]</b> text`

Если задан `serverName`, tgbridge добавляет `<b>[serverName]</b>` перед тегом категории. Если задан `prefix`, он добавляется после префикса с именем сервера.

## Архитектура

- `TgBridgeMod`: server-only entrypoint Fabric и регистрация событий
- `BridgeService`: центральный слой orchestration для lifecycle, фильтрации, дедупликации, роутинга, audit-логирования и команд
- `MessageRouter`: выбирает нужный Telegram topic и формирует безопасные HTML-сообщения
- `TelegramSender`: асинхронный отправитель с очередью, retry, backoff и rate limiting
- `ConfigManager`: загружает и перезагружает `config/tgbridge.json`
- `JsonlAuditLogger`: пишет пересылаемые события в JSONL, не блокируя основной серверный поток

## Зависимости

Зависимости сборки и рантайма:

- `com.mojang:minecraft:1.21.11`
- `net.fabricmc:yarn:1.21.11+build.4:v2`
- `net.fabricmc:fabric-loader:0.18.4`
- `net.fabricmc.fabric-api:fabric-api:0.141.3+1.21.11`
- `com.google.code.gson:gson:2.11.0`

Инструменты:

- `net.fabricmc.fabric-loom-remap:1.14.1`
- Gradle `9.2.0`

## Настройка Telegram

### 1. Создайте бота через BotFather

Откройте [@BotFather](https://t.me/BotFather), выполните `/newbot` и сохраните токен бота.

Полезные ссылки:

- [Telegram Bot API](https://core.telegram.org/bots/api)
- [sendMessage](https://core.telegram.org/bots/api#sendmessage)
- [getUpdates](https://core.telegram.org/bots/api#getupdates)

### 2. Добавьте бота в supergroup

Создайте нужную supergroup или откройте существующую и добавьте туда бота.

У бота должно быть право отправлять сообщения в группу. Если вы хотите использовать `getUpdates`, чтобы получить ID тем по сообщениям внутри topics, обычно удобнее временно отключить privacy mode у бота через BotFather командой `/setprivacy`, а потом при желании включить обратно.

### 3. Включите forum topics

В Telegram откройте настройки supergroup и включите Topics / Forum Topics для группы.

Создайте по одной теме на каждую категорию, например:

- `chat`
- `joins`
- `commands`
- `system`
- `events`

### 4. Получите `chat_id`

После того как бот добавлен в supergroup, отправьте любое сообщение в группу и затем вызовите:

```text
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
```

В ответе найдите `message.chat.id`. Для supergroup это обычно отрицательное 64-битное число вида `-1001234567890`.

### 5. Получите каждый `message_thread_id`

Отправьте хотя бы по одному сообщению в каждую forum topic, которую мод должен использовать, затем вызовите:

```text
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
```

Для каждого сообщения внутри темы смотрите:

- `message.chat.id` -> общий `chat_id`
- `message.message_thread_id` -> ID темы, который нужно записать в `threadIds.*`

Мод использует один общий `chat_id` и такие per-topic thread IDs:

- `threadIds.chat`
- `threadIds.joins`
- `threadIds.commands`
- `threadIds.system`
- `threadIds.events`

## Конфигурация

Во время работы мод читает:

- `config/tgbridge.json`

Если файл отсутствует, tgbridge автоматически создаст дефолтный конфиг при первом запуске.

Готовый к редактированию пример лежит в `config.example.json`.

### Поддерживаемые поля

- `botToken`: токен Telegram-бота
- `chatId`: ID Telegram supergroup
- `threadIds.chat`: ID темы для обычного игрового чата
- `threadIds.joins`: ID темы для сообщений о входе и выходе игроков
- `threadIds.commands`: ID темы для сообщений команд
- `threadIds.system`: ID темы для lifecycle-сообщений, ошибок моста и уведомлений о reload
- `threadIds.events`: ID темы для death messages, advancements и прочих игровых событий
- `enabled.chat`
- `enabled.joins`
- `enabled.commands`
- `enabled.system`
- `enabled.events`
- `includeAdvancements`
- `includeDeaths`
- `includeServerLifecycle`
- `prefix`
- `serverName`
- `ignoredPlayers`
- `ignoredRegex`
- `whitelistTypes`
- `blacklistTypes`
- `auditLogPath`
- `queueCapacity`
- `maxRetries`
- `baseRetryDelayMs`
- `maxRetryDelayMs`
- `rateLimitPerSecond`
- `dedupWindowMs`
- `requestTimeoutMs`
- `mirrorBridgeErrorsToSystemTopic`

### Семантика фильтров

- `ignoredPlayers`: точные имена игроков, которые нужно игнорировать
- `ignoredRegex`: regex-шаблоны, проверяемые по сырому тексту сообщения
- `whitelistTypes`: разрешать только перечисленные категории или имена событий
- `blacklistTypes`: всегда блокировать перечисленные категории или имена событий

Допустимые имена типов включают как категории, так и более точные event names для join/quit:

- категории: `chat`, `joins`, `commands`, `system`, `events`
- имена событий: `chat`, `join`, `quit`, `command`, `system`, `event`

## Команды

- `/tgbridge status`: показывает путь к конфигу, размер очереди, число отправленных и отброшенных сообщений, а также последнюю ошибку отправителя
- `/tgbridge test`: ставит в очередь по одному тестовому сообщению для каждой настроенной темы
- `/tgbridge reload`: перезагружает `config/tgbridge.json` без рестарта сервера

## Audit log

Каждое принятое к пересылке событие по умолчанию дописывается в JSONL-файл:

- `logs/tgbridge-events.jsonl`

Каждая строка содержит:

- `timestamp`
- `type`
- `player`
- `rawText`
- `telegramThread`

## Сборка

Собирайте проект из корня репозитория:

```powershell
.\gradlew.bat build
```

Готовые jar-файлы:

- `build/libs/tgbridge-<version>.jar`
- `build/libs/tgbridge-<version>-sources.jar`

## Установка

1. Соберите мод.
2. Скопируйте `build/libs/tgbridge-<version>.jar` в папку `mods/` вашего Fabric-сервера.
3. Один раз запустите сервер, чтобы сгенерировался `config/tgbridge.json`.
4. Отредактируйте `config/tgbridge.json`, указав токен бота, `chatId` и thread ID тем.
5. Запустите или перезапустите сервер.

## Проверка работы

1. Выполните `/tgbridge status` и убедитесь, что конфиг загружен.
2. Выполните `/tgbridge test` и убедитесь, что все настроенные Telegram topics получили тестовое сообщение.
3. Зайдите на сервер, напишите в чат, выполните команду и вызовите смерть или advancement.
4. Убедитесь, что сообщение пришло в ожидаемую тему.
5. При необходимости проверьте `logs/tgbridge-events.jsonl` как локальный журнал событий.

## Примечания

- Мод server-only и использует server-only конфигурацию Minecraft jar в Loom.
- Сетевые ошибки логируются и ретраятся; они не должны крашить Minecraft-сервер.
- Ошибки доставки моста логируются в консоль сервера и по возможности зеркалятся в тему `system`.
#   t g m o d  
 #   t g m o d  
 