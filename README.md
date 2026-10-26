# 🚗 AutoSniper — мониторинг объявлений о продаже авто + Telegram-бот

Асинхронная full-stack система на Python 3.11+: собирает объявления о продаже автомобилей
из Telegram-каналов, сообществ VK, Avito / Auto.ru / Drom.ru и RSS-лент, сопоставляет их
с фильтрами пользователей и присылает карточки в Telegram.

> Пошаговая инструкция для пользователя (установка, токены, запуск, меню бота,
> автозапуск и решение проблем) — в [MANUAL.md](MANUAL.md).
> Подробное подключение Telegram, VK, Avito/Auto.ru/Drom и RSS — в [SOURCES.md](SOURCES.md).
> История изменений проекта — в [CHANGELOG.md](CHANGELOG.md).

```
Telegram-каналы ─┐
VK-сообщества ───┼──► Scrapers ──► Parser engine ──► SQLAlchemy ──► Matcher ──► Notifier ──► aiogram
Avito / Drom ────┤   (курсоры,      (цена, город,      (SQLite /      (фильтры      (HTML +
Auto.ru / RSS ───┘    retry,         марка, модель,     PostgreSQL)    подписок)      фото)
                      backoff)       год)                                  │
                                                                           └── dedup по hash_id
```

---

## Возможности

- **Четыре источника данных**: Telegram (Telethon, пользовательская сессия), VK API
  (`wall.get` + `newsfeed.search` + региональные группы), веб-площадки Avito/Auto.ru/Drom
  и RSS (JSON-LD + embedded-state + fallback-парсинг, опционально Playwright).
- **Локальные агенты проверки**: исправляют опечатки в марке/модели и тексте
  объявления, понимают сленг и народные названия (`семёрка`, `марковник`,
  `буханка`, `солярис`), предлагают варианты. Без платных API.
- **Безопасность**: админ-роутер по `ADMIN_IDS`, антифлуд, суточный лимит поисков,
  холодный старт, XSS-экранирование, белый список ссылок и `/delete_me`.
- **Интерактивный бот на aiogram 3.x**: inline-меню, пошаговый FSM-визард, пагинация подписок,
  пауза/удаление, разовый поиск, статус системы.
- **Умный парсинг русских объявлений**: `2.350.000`, `1,2 млн`, `1200k`, `120 тыс.`;
  города и их падежи/сокращения (`СПб`, `МСК`, `в Питере`); марки и модели с алиасами
  (`бмв`, `ладу`, `камри`, `320i` → `BMW 3 Series`).
- **Несколько моделей и радиус города**: до 10 моделей в одной подписке, расстояние
  от города (только город, 50/100/200/300/500 км или без ограничения).
- **Только свежие объявления**: подписки присылают публикации за последние
  `SUBSCRIPTION_MAX_AGE_DAYS` дней; разовый поиск — за месяц, неделю, сутки или всё время.
- **Дедупликация**: SHA-256 от нормализованных заголовка, цены и URL + журнал доставки
  `(user_id, listing_id)` — ни одного повторного уведомления.
- **Устойчивость**: экспоненциальный backoff, `Retry-After`, FloodWait, ротация User-Agent
  и прокси, ограничение скорости, лимит скачивания фото.
- **SQLite из коробки, PostgreSQL для продакшена**, Alembic-миграции, 112 offline-тестов, Docker.

---

## Содержание

1. [Быстрый старт](#быстрый-старт)
2. [Получение доступов](#получение-доступов)
3. [База данных и миграции](#база-данных-и-миграции)
4. [Запуск](#запуск)
5. [Как пользоваться ботом](#как-пользоваться-ботом)
6. [Конфигурация](#конфигурация)
7. [Как это работает внутри](#как-это-работает-внутри)
8. [Структура проекта](#структура-проекта)
9. [Тесты](#тесты)
10. [Docker и systemd](#docker-и-systemd)
11. [Диагностика](#диагностика)
12. [Ограничения и правовые заметки](#ограничения-и-правовые-заметки)

---

## Быстрый старт

Требуется **Python 3.11+**.

```powershell
# Windows PowerShell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

```bash
# Linux / macOS
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Минимально заполните `.env`:

```dotenv
BOT_TOKEN=               # @BotFather
DATABASE_URL=sqlite+aiosqlite:///./data/autosniper.db

# хотя бы один источник:
TELEGRAM_API_ID=
TELEGRAM_API_HASH=
TELEGRAM_CHANNELS=@auto_sale_msk
```

Затем:

```bash
python main.py --migrate     # создать схему (Alembic)
python main.py --doctor      # проверить конфигурацию, БД и токен бота
python main.py               # запустить бота + фоновый мониторинг
```

> Разработка на SQLite: вместо `--migrate` можно один раз выполнить
> `python main.py --init-db` (создаёт таблицы из метаданных). Не смешивайте два способа
> на одной базе: `init-db` не заполняет `alembic_version`.

Проверка без Telegram и без отправки уведомлений:

```bash
python main.py --once --no-notify
```

---

## Получение доступов

### 1. Токен бота

1. Откройте [@BotFather](https://t.me/BotFather) → `/newbot`.
2. Придумайте имя и username, скопируйте токен в `BOT_TOKEN`.
3. Команды бота и меню регистрируются автоматически при старте.

### 2. Telegram API для скрапера каналов

Скрапер читает **публичные каналы и чаты от имени пользователя**, поэтому нужен
`api_id`/`api_hash`, а не токен бота.

1. Зайдите на <https://my.telegram.org> → **API development tools**.
2. Создайте приложение (название и платформа — любые).
3. Скопируйте `api_id` и `api_hash` в `.env`.
4. Один раз авторизуйтесь:

```bash
python scripts/tg_login.py
```

Скрипт спросит номер телефона и код из Telegram (и пароль 2FA, если включён).
Будет создан файл `data/telegram_scraper.session` — **он даёт полный доступ к аккаунту**,
не публикуйте его и не коммитьте.

5. Добавьте каналы в `TELEGRAM_CHANNELS`:

```dotenv
TELEGRAM_CHANNELS=@auto_sale_msk,https://t.me/cars_sale_spb,-1001234567890
```

Аккаунт должен состоять в приватных чатах, иначе их не прочитать.

> Используйте отдельный аккаунт. Массовое чтение и подписки на десятки каналов могут
> привести к ограничениям Telegram.

### 3. VK

1. <https://vk.com/dev> → **Создать приложение** → тип *Standalone*.
2. Получите пользовательский токен (Implicit Flow) с правами `wall`, `groups`, `offline`:

```
https://oauth.vk.com/authorize?client_id=ВАШ_ID&display=page&redirect_uri=https://oauth.vk.com/blank.html&scope=wall,groups,offline&response_type=token&v=5.199
```

3. После подтверждения скопируйте `access_token` из адресной строки в `VK_TOKEN`.
4. Укажите сообщества:

```dotenv
VK_GROUPS=-12345678,auto_sale_group,https://vk.com/public123
```

Если `VK_GROUPS` пуст, используется глобальный `newsfeed.search` по вашим подпискам
(нужен пользовательский токен; сервисный токен для этого метода не подходит).

### 4. Веб-площадки и RSS

`WEB_TARGETS` — список `провайдер:URL` через запятую:

```dotenv
WEB_TARGETS=avito:https://www.avito.ru/moskva/avtomobili,auto_ru:https://auto.ru/moskva/cars/used/?sort=fresh_relevance_1-desc,drom:https://auto.drom.ru/moscow/,rss:https://example.com/cars/feed.xml
```

Поддерживаются провайдеры:

| Провайдер | Что делает | Поисковый параметр |
|---|---|---|
| `avito` | Avito, JSON-LD + fallback по ссылкам | `?q=` |
| `auto_ru` | Auto.ru, JSON-LD объявлений | `?text=` |
| `rss` | RSS 2.0 / Atom (в т.ч. фиды Avito/Auto.ru) | — |

Avito aggressively защищается от ботов: при `403`/пустой выдаче включите Playwright
(см. ниже) или используйте RSS/официальный API. Система **не обходит CAPTCHA**.

```bash
pip install playwright
playwright install chromium
# .env: WEB_USE_PLAYWRIGHT=true
```

---

## База данных и миграции

По умолчанию — SQLite (`./data/autosniper.db`), включены WAL и внешние ключи.

### PostgreSQL

```bash
pip install asyncpg
```

```dotenv
DATABASE_URL=postgresql+asyncpg://autosniper:password@localhost:5432/autosniper
```

Схема:

| Таблица | Назначение |
|---|---|
| `users` | пользователи бота; `is_active` = уведомления включены |
| `subscriptions` | фильтры: марка, модели, город, расстояние, максимальная цена, год от |
| `listings` | уникальные объявления (`hash_id`), распознанные поля, фото, исходный текст |
| `notifications` | журнал доставки, unique `(user_id, listing_id)`, повторные попытки |
| `scraper_states` | курсоры и здоровье источников между перезапусками |

### Alembic

```bash
python main.py --migrate            # = alembic upgrade head
alembic current                    # текущая ревизия
alembic revision --autogenerate -m "add field"   # новая миграция после правки моделей
alembic downgrade base             # откатить всё (удалит данные!)
```

Миграции — `alembic/versions/0001_initial_schema.py` и
`0002_subscription_models_distance_age.py`.

---

## Запуск

### Обычный режим

```bash
python main.py
```

Одновременно работают:

- long polling aiogram (`asyncio`-задача `aiogram-polling`);
- фоновый цикл скраперов (`scraper-loop`) с интервалом `POLL_INTERVAL_SECONDS` + jitter.

Остановка — `Ctrl+C` (SIGINT/SIGTERM обрабатываются, клиенты и БД закрываются корректно).

### Разовые команды

```bash
python main.py --doctor                  # проверка конфигурации / БД / токена
python main.py --once                    # один цикл сбора и отправки уведомлений
python main.py --once --no-notify        # один цикл без Telegram-доставки
python main.py --init-db                 # создать таблицы (dev, SQLite)
python main.py --migrate                 # применить миграции
python main.py --log-level DEBUG         # переопределить уровень логов
```

`--once` удобно вызывать из cron, если не нужен постоянный процесс, но тогда
планировщик внутри не используется.

### Режим «только бот»

Отключите все источники — бот запустится, но объявлений не будет:

```dotenv
TELEGRAM_ENABLED=false
VK_ENABLED=false
WEB_ENABLED=false
```

---

## Как пользоваться ботом

`/start` открывает меню:

```
[ ➕ Добавить подписку ]
[ 📋 Мои подписки ]  [ ⚙️ Настройки ]
[ 🔍 Разовый поиск сейчас ]  [ ❓ Помощь / Статус ]
```

### Добавление подписки (FSM, 5 шагов ввода)

1. **Марка** — кнопки популярных марок или свой текст (`BMW`, `Лада`, `Mercedes`).
2. **Модели** — одна или несколько через запятую (`3 Series, X5`, до 10) или «⏭ Любая модель».
3. **Город** — кнопки или свой текст; «🌍 Любой город».
4. **Расстояние** — только город, 50/100/200/300/500 км или без ограничения.
5. **Максимальная цена** — число (`2 500 000`) или `2,5 млн`; «♾ Без ограничения».
6. **Подтверждение** — `✅ Подтвердить` / `❌ Отмена`.

Дубликаты и лимит `MAX_SUBSCRIPTIONS_PER_USER` проверяются до сохранения.
Расстояние доступно для городов из встроенного справочника координат; для остальных
шаг пропускается.

### Управление

- «📋 Мои подписки» — по 5 на страницу, у каждой: `⏸ Пауза` / `▶️ Включить` и `🗑 Удалить`
  (с подтверждением).
- «⚙️ Настройки» — включение/выключение уведомлений, удаление всех подписок.
- «🔍 Разовый поиск» — выбрать подписку и период (месяц, неделя, сутки, всё время),
  опросить источники сейчас; результаты **не сохраняются** и не влияют на уведомления.
- «❓ Помощь / Статус» — статистика, аптайм, состояние каждого источника; админам
  показываются последние ошибки.

### Формат уведомления

Приходит HTML-карточка с фото (если оно есть) и ссылкой:

```
🚗 НАЙДЕН АВТОМОБИЛЬ!

📌 Модель: BMW 3 Series
💰 Цена: 2 350 000 ₽
📍 Город: Москва
🌐 Источник: Telegram (@auto_sale_msk)

📝 Описание: Продам тройку, 2018 год, идеальное состояние...

🔗 Открыть объявление
```

### Команды

| Команда | Действие |
|---|---|
| `/start`, `/menu` | главное меню |
| `/add` | новая подписка |
| `/subs` | список и управление подписками |
| `/settings` | настройки уведомлений |
| `/search` | разовый поиск |
| `/status` | состояние системы |
| `/help` | справка |
| `/cancel` | отменить текущий ввод |

---

## Конфигурация

Все параметры читаются из `.env`; полный список с комментариями — в
[`.env.example`](.env.example). Ключевые:

| Переменная | По умолчанию | Смысл |
|---|---|---|
| `BOT_TOKEN` | — | токен @BotFather |
| `ADMIN_IDS` | — | ID админов через запятую; им доступен `/admin` |
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/autosniper.db` | DSN БД |
| `TELEGRAM_API_ID` / `_HASH` | — | доступ к user-сессии |
| `TELEGRAM_SESSION` | `data/telegram_scraper` | файл сессии |
| `TELEGRAM_CHANNELS` | — | каналы/чаты |
| `TELEGRAM_DOWNLOAD_PHOTOS` | `true` | скачивать первое фото |
| `VK_TOKEN` | — | токен VK |
| `VK_GROUPS` | — | сообщества; пусто → `newsfeed.search` |
| `WEB_TARGETS` | — | `avito:…`, `auto_ru:…`, `drom:…`, `rss:…` |
| `WEB_USE_PLAYWRIGHT` | `false` | рендер JS-страниц |
| `PROXY_URL` / `PROXY_LIST` | — | один прокси / ротация |
| `POLL_INTERVAL_SECONDS` | `180` | период опроса |
| `MAX_RETRIES` | `4` | повторы HTTP-запросов |
| `BACKOFF_BASE_SECONDS` / `_MAX` | `2` / `120` | экспоненциальный backoff |
| `MATCH_REQUIRE_KNOWN_PRICE` | `true` | не присылать без распознанной цены |
| `MATCH_REQUIRE_KNOWN_CITY` | `false` | требовать распознанный город |
| `MATCH_SAME_REGION` | `true` | Москва и Подмосковье — один регион |
| `SUBSCRIPTION_MAX_AGE_DAYS` | `7` | подписки присылают объявления не старше N дней (`0` — без ограничения) |
| `LOG_LEVEL`, `LOG_FILE` | `INFO`, `logs/autosniper.log` | логи (ротация 5 МБ × 5) |

Пути к SQLite/логам/медиа создаются автоматически при старте.

---

## Как это работает внутри

### Цикл сбора

1. `ScraperOrchestrator` загружает активные подписки пользователей с включёнными
   уведомлениями и превращает их в `SearchQuery` (марка + модели + город + бюджет).
2. Каждый включённый скрапер получает свой сохранённый курсор:
   - Telegram — `min_id` по каждому каналу;
   - VK — `since` (unix time);
   - веб — URL-цели (дедупликация по хэшу).
3. Скраперы работают параллельно, парсинг и запись в БД — последовательно
   (бережём SQLite от конкурирующих писателей).
4. Сырые посты (`RawListing`) нормализуются, дедуплицируются и сохраняются.
5. Для новых объявлений matcher проверяет подписки; на каждую пару «пользователь ×
   объявление» создаётся запись в `notifications`.
6. **После коммита** уведомления отправляются; результат (`sent_at` / ошибка) пишется
   отдельной транзакцией. Недоставленные повторяются в следующем цикле.
7. Курсор и здоровье источника сохраняются в `scraper_states`.

### Парсинг текста

`services/parser_engine.py` — чистый stdlib, покрыт тестами:

- **Цена**: `1 200 000`, `1.200.000`, `1,2 млн`, `2.35 млн`, `1200k`, `120 тыс.`,
  `1 250 000 рублей`; игнорирует годы, пробег, телефоны, л.с.
- **Город**: ~100 городов с падежами и сокращениями, включая СНГ; учитывает
  `г.`, `город:`, `📍`; сравнение с регионами (Москва ↔ Подольск).
- **Марка**: ~60 брендов и русские алиасы (`бмв`, `мерс`, `ладу`, `фольц`, `чери`…).
- **Модель**: извлекается после марки, отсекаются стоп-слова и год; алиасы
  (`камри` → Camry, `веста` → Vesta); `320i` распознаётся как `3 Series`.
- **Год**: приоритет у чисел с маркером «год/г.».

### Сопоставление

Марка — каноническое сравнение; модели — совпадение любой из выбранных (токены/Jaccard);
цена — `≤ max_price`; город — точное совпадение, один регион или радиус от центра
(`max_distance_km`); год — `≥ year_min`; свежесть — дата публикации не старше
`max_age_days`. Если цена не распознана, по умолчанию объявление не отправляется
(`MATCH_REQUIRE_KNOWN_PRICE=true`), город — можно не требовать.

### Дедупликация и доставка

`hash_id = SHA256(normalized_title | price | normalized_url)`; из URL удаляются
`utm_*`, `from`, `ref` и т.п. Повторный импорт того же поста не создаёт дубль.
Уникальный индекс `(user_id, listing_id)` гарантирует одно уведомление на пользователя,
даже если под него подходят несколько фильтров.

### Устойчивость

- HTTP: до `MAX_RETRIES` попыток, экспоненциальная задержка с jitter, учёт `Retry-After`,
  ротация User-Agent и прокси, ограничение запросов в секунду.
- Telegram: `flood_sleep_threshold=60`; при более длинном FloodWait канал пропускается
  до следующего цикла, ошибка логируется.
- Скрапер: после ошибки — собственный cooldown (5 с → 120 с, удвоение), состояние
  сохраняется в БД.
- Уведомления: `TelegramRetryAfter` повторяется, `403 blocked` деактивирует пользователя,
  прочие ошибки фиксируются и повторяются позже.

---

## Структура проекта

```
.
├── main.py                     # CLI, запуск бота + скраперов, doctor/once/migrate
├── MANUAL.md                   # пошаговая инструкция для пользователя
├── SOURCES.md                  # подробное подключение Telegram / VK / Avito / RSS
├── CHANGELOG.md                # история изменений проекта
├── start.bat / doctor.bat / tg_login.bat / tg_dialogs.bat / vk_check.bat
├── config/
│   ├── settings.py             # типизированные настройки из .env
│   └── logger.py               # консоль + ротируемый файл, UTF-8
├── database/
│   ├── models.py               # SQLAlchemy 2.0 модели
│   ├── session.py              # async engine, сессии, PRAGMA SQLite
│   └── repositories.py         # весь SQL (users, subs, listings, notifications, states)
├── scrapers/
│   ├── base.py                 # RawListing, ScrapeBatch, SearchQuery, BaseScraper
│   ├── telegram.py             # Telethon: каналы, фото, FloodWait, курсоры
│   ├── vk.py                   # wall.get / newsfeed.search, фото, geo
│   ├── web.py                  # Avito / Auto.ru / Drom / RSS, JSON-LD, ротация
│   ├── browser.py              # опциональный Playwright
│   └── registry.py             # собирает только настроенные источники
├── services/
│   ├── parser_engine.py        # цена/город/марка/модель/год
│   ├── geo.py                  # координаты городов и расстояние
│   ├── matching.py             # правила сопоставления
│   ├── agents.py               # локальные агенты проверки/опечаток
│   ├── synonyms_dict.py        # сленг и народные названия авто
│   ├── hashing.py              # нормализация и hash_id
│   ├── normalizer.py           # RawListing → Listing
│   ├── pipeline.py             # dedup → store → match → notify → journal
│   ├── notifier.py             # HTML-карточки, фото, ошибки Telegram
│   ├── orchestrator.py         # циклы, backoff, разовый поиск, статус
│   ├── net.py                  # HTTP-клиент, retry, rate limit, прокси, UA
│   └── formatting.py           # форматирование цен/дат/подписок
├── bot/
│   ├── app.py                  # Bot/Dispatcher, middleware, роутеры
│   ├── middlewares.py          # сессия БД, пользователь, throttle, error handler
│   ├── states.py               # FSM визарда
│   ├── keyboards.py            # inline-клавиатуры
│   ├── callbacks.py            # типизированные callback-data
│   ├── texts.py                # тексты и статус
│   ├── utils.py                # safe_edit, uptime
│   └── handlers/               # start, subscriptions, search, help, fallback
├── alembic/                    # env.py + 0001 + 0002 (models/radius/age)
├── tests/                      # 112 offline-тестов
├── scripts/tg_login.py         # интерактивная авторизация Telethon
├── Dockerfile / docker-compose.yml
├── requirements.txt / requirements-dev.txt
├── .env.example
├── alembic.ini
└── pyproject.toml              # pytest + ruff
```

---

## Тесты

```bash
pip install -r requirements-dev.txt
pytest -q
ruff check .
```

Покрыто без сети и без Telegram:

- парсинг цен/городов/марок/моделей/года (включая «ловушки»: пробег, телефоны, млн/k);
- правила сопоставления и политики неизвестных полей;
- дедупликация и нормализация URL;
- полный pipeline на in-memory SQLite: создание, повторный импорт, блокировка бота,
  один алерт при двух подходящих фильтрах, повтор недоставленных уведомлений;
- парсеры Avito JSON-LD, RSS, VK-маппинг, конфиг веб-целей;
- клавиатуры, callback-data ≤ 64 байт, экранирование и лимит длины caption;
- полный FSM-визард «добавить → подтвердить → пауза → удалить» через фейковую
  Telegram-сессию (без сети).

Проверка миграций:

```bash
python main.py --migrate
alembic check          # "No new upgrade operations detected."
```

---

## Docker и systemd

### Docker Compose (PostgreSQL)

```bash
cp .env.example .env
# заполните BOT_TOKEN, TELEGRAM_*, VK_*, WEB_TARGETS
docker compose up -d --build
docker compose logs -f bot
```

Внутри контейнера выполняется `python main.py --migrate && python main.py`.

Авторизация Telethon (файл сессии сохраняется в `./data`):

```bash
docker compose run --rm bot python scripts/tg_login.py
docker compose restart bot
```

Смените пароль PostgreSQL в `docker-compose.yml` и не публикуйте порт 5432 наружу.

### systemd

```ini
# /etc/systemd/system/autosniper.service
[Unit]
Description=AutoSniper Telegram bot
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=autosniper
WorkingDirectory=/opt/autosniper
EnvironmentFile=/opt/autosniper/.env
ExecStart=/opt/autosniper/.venv/bin/python main.py
Restart=always
RestartSec=10
KillSignal=SIGINT
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now autosniper
journalctl -u autosniper -f
```

Запускайте **один** экземпляр с одним `BOT_TOKEN`: long polling не поддерживает
несколько реплик.

---

## Диагностика

| Симптом | Причина / решение |
|---|---|
| `No scraper is configured` | Заполните `TELEGRAM_*`, `VK_*` или `WEB_TARGETS`; либо отключите все источники для bot-only режима |
| `Telegram session is not authorized` | `python scripts/tg_login.py`; проверьте `TELEGRAM_SESSION` |
| `FloodWait` в логах | Telegram просит подождать; увеличьте `POLL_INTERVAL_SECONDS`, уменьшите число каналов |
| Avito отдаёт 403 / пусто | Включите Playwright, используйте прокси/регион в URL или RSS; CAPTCHA не обходится |
| `Database schema is missing` | `python main.py --migrate` |
| `database is locked` (SQLite) | Не запускайте несколько процессов; WAL включён, но SQLite — не для нагрузки |
| Уведомлений нет | Проверьте `/status`, `MATCH_REQUIRE_KNOWN_PRICE`, цену/город в объявлении, `is_active` пользователя, лог `notifications.last_error` |
| Фото не приходит | Локальный файл удалён/нет прав; Telegram не смог скачать внешний URL — карточка уйдёт текстом |
| Бот молчит | Проверьте `BOT_TOKEN`, `getMe` через `--doctor`, логи `logs/autosniper.log` |
| Краказябры в консоли Windows | Логи пишутся в UTF-8; используйте Windows Terminal или `chcp 65001` |

Полезные места:

```bash
tail -f logs/autosniper.log
python main.py --doctor
```

---

## Ограничения и правовые заметки

- **Правила площадок.** Avito, Auto.ru и VK ограничивают автоматический сбор. Используйте
  официальные API/фиды, соблюдайте `robots.txt`, лимиты и пользовательское соглашение.
  Проект не обходит CAPTCHA и не маскируется под человека.
- **Telegram-аккаунт.** User-сессия = доступ к аккаунту. Риски ограничений и блокировки
  лежат на владельце; используйте отдельный аккаунт и умеренный polling.
- **Эвристики.** Цена/город/модель могут не распознаться при необычном тексте; политики
  `MATCH_REQUIRE_*` позволяют выбрать строгость. Все распознанные поля видны в БД.
- **Один процесс.** FSM хранится в памяти (перезапуск сбрасывает незавершённый визард),
  long polling не масштабируется горизонтально. Для нескольких реплик нужны webhook,
  Redis FSM и отдельный воркер.
- **Веб-скрейпинг** хрупок к изменениям вёрстки; JSON-LD-парсер устойчивее селекторов,
  но прикрывайте его RSS-фидами.
- **Данные.** В БД хранятся тексты объявлений, URL и Telegram ID пользователей. Настройте
  ротацию/бэкапы и удаление по запросу.
- **Не является инвестиционной/юридической рекомендацией.** Проверяйте автомобиль и
  продавца самостоятельно; система только показывает объявления.

---

## Лицензия

Проект предоставлен «как есть» для образовательных и личных целей, без гарантий.
Ответственность за соблюдение законов и правил площадок несёт пользователь.
