# Server: Backend модули

## Общее описание

Backend обрабатывает все операции с `.inpx`-данными: парсинг, индексацию, поиск и скачивание книг.
Построен на Express + WebSocket (`ws`). База данных — jembadb (в отдельном потоке через `JembaDbThread`).

Точка входа: `server/index.js` — инициализирует конфиг, логгер, создаёт HTTP+WS сервер, подключает OPDS и WebAccess.

## Структура директории

```
server/
├── index.js                  # Точка входа: инициализация и запуск сервера
├── createWebApp.js           # Production: обслуживание собранного SPA
├── static.js                 # Сервинг статических файлов (/book, /public-files)
├── dev.js                    # Dev-режим: webpack HMR middleware
├── core/                     # Ядро — вся бизнес-логика
│   ├── AppLogger.js          # Логгер (singleton)
│   ├── AsyncExit.js          # Graceful shutdown (singleton)
│   ├── DbCreator.js          # Создание и заполнение jembadb из .inpx
│   ├── DbSearcher.js         # Поиск по jembadb (с кешированием)
│   ├── FileDownloader.js     # Скачивание файлов из удалённых источников
│   ├── HeavyCalc.js          # Worker thread для тяжёлых вычислений (Node.js worker_threads)
│   ├── InpxHashCreator.js    # Хэш .inpx + filter для инвалидации кеша БД
│   ├── InpxParser.js         # Парсер .inpx формата (ZIP + XML)
│   ├── LockQueue.js          # Очередь-мьютекс для асинхронного доступа
│   ├── Logger.js             # Базовый логгер (файл + консоль)
│   ├── RemoteLib.js          # Управление удалённой библиотекой (singleton)
│   ├── utils.js              # Утилиты: хэши, случайные строки, файловый обход
│   ├── WebAccess.js          # HTTP API: аутентификация (accessPassword + токены), роутинг
│   ├── WebSocketConnection.js# Клиент-side WS обёртка: send()/message() с очередью и таймаутами
│   ├── WebWorker.js          # Оркестратор (singleton): БД, поиск, скачивание
│   ├── WorkerState.js        # Трекинг состояния долгих операций (singleton)
│   ├── ZipReader.js          # Чтение zip-архивов (node-stream-zip)
│   ├── fb2/                  # FB2-обработка
│   │   ├── Fb2Helper.js      # Извлечение обложки, конвертация FB2
│   │   ├── Fb2Parser.js      # Парсер FB2 (XML → объект)
│   │   └── textUtils.js      # Утилиты для текстовой обработки
│   ├── genres/               # Жанры
│   │   ├── index.js          # Экспорт дерева жанров
│   │   └── genresText.js     # Тексты жанров (ru/en)
│   ├── opds/                 # OPDS-сервер (RFC 4287 + OpenSearch)
│   │   ├── index.js          # Express middleware для OPDS
│   │   ├── BasePage.js       # Базовый класс страницы OPDS (Atom feed)
│   │   ├── RootPage.js       # /opds — корневой каталог
│   │   ├── SearchPage.js     # /opds/search — поиск
│   │   ├── AuthorPage.js     # /opds/author/...
│   │   ├── BookPage.js       # /opds/book/...
│   │   ├── GenrePage.js      # /opds/genre/...
│   │   ├── SeriesPage.js     # /opds/series/...
│   │   ├── TitlePage.js      # /opds/title/...
│   │   ├── OpensearchPage.js # OpenSearch description
│   │   └── SearchHelpPage.js # Справка по синтаксису поиска
│   └── xml/                  # XML-утилиты
│       ├── XmlParser.js      # Потоковый SAX-парсер (обёртка над sax.js)
│       ├── sax.js            # SAX-парсер (встроенный)
│       └── ObjectInspector.js# Утилита инспекции объектов
├── controllers/
│   ├── index.js              # Экспорт всех контроллеров
│   └── WebSocketController.js# WS endpoint: принимает сообщения от клиента → WebWorker
└── config/
    ├── index.js              # ConfigManager: загрузка и объединение конфигов
    ├── base.js               # Базовые значения всех параметров
    ├── development.js        # Overrides для dev-режима (branch: 'development')
    ├── production.js         # Overrides для production
    └── application_env       # Описание переменных окружения (11 переменных)
```

## Модули core/ — подробное описание

### `WebWorker.js` — главный оркестратор (singleton)
- Инициализирует и пересоздаёт jembadb при старте или изменении `.inpx`
- Управляет `DbCreator` (создание БД) и `DbSearcher` (поиск)
- Обрабатывает WS-запросы от клиента: поиск, скачивание, информация о книге
- Периодически проверяет `.inpx` на изменения (`inpxCheckInterval`)
- Состояния сервера: `normal`, `db_loading`, `db_creating`
- Отслеживает прогресс через `WorkerState`

### `DbCreator.js` — создание базы данных
- Парсит `.inpx` через `InpxParser` и индексирует данные в jembadb
- Таблицы: `books`, `authors`, `series`, `titles`, `genres`, `dbInfo`
- Поддерживает `filter.json` для фильтрации авторов/книг/языков
- При `lowMemoryMode` ограничивает использование памяти

### `DbSearcher.js` — поиск по базе
- Поддерживает поиск: по автору, серии, названию, жанру, языку, дате, оценке, типу файла
- Расширенный поиск (раздел `</>`): полнотекстовый по всем полям
- Двухуровневый кеш запросов: в памяти (`queryCacheMemSize`) + на диске (`queryCacheDiskSize`)
- Синтаксис поиска: `=точное`, `*содержит`, обычный — поиск по префиксу
- `maxLimit = 1000` записей в ответе

### `WebSocketController.js` — WS-сервер
- Принимает JSON-сообщения от клиента, передаёт в `WebWorker`
- Закрывает неактивные соединения через 5 минут
- Логирует входящие/исходящие сообщения при `logQueries: true`

### `WebAccess.js` — HTTP API и аутентификация
- Управляет токенами сессий (хранит в jembadb таблице `access`)
- При смене пароля в конфиге сбрасывает все токены
- `periodicClean()` — удаляет токены старше `accessTimeout`
- `freeAccess = true` если `accessPassword === ''`

### `WorkerState.js` — трекинг долгих операций (singleton)
- `getControl(workerId)` → объект с методами `set()`, `finish()`, `get()`
- Используется для отображения прогресса создания БД и скачивания
- Автоочистка устаревших состояний каждые 3600 сек

### `HeavyCalc.js` — тяжёлые вычисления
- Запускает Node.js `worker_threads.Worker` для выполнения произвольных функций
- Используется для операций, которые могут заблокировать event loop
- API: `calc(fn, args)` → Promise с результатом

### `InpxHashCreator.js` — инвалидация кеша БД
- Хэш = `dbVersion` + хэш `filter.json` (если есть) + хэш `.inpx`
- При изменении хэша БД пересоздаётся

### `LockQueue.js` — мьютекс-очередь
- Ограничивает параллельный доступ к ресурсу
- `get()` → занять блокировку, `ret()` → освободить
- Размер очереди по умолчанию: 100

### `WebSocketConnection.js` (в core/)
- Кросс-платформенная WS-обёртка для использования на сервере (библиотека `ws`)
- `send(action, args, timeout)` → Promise
- Обрабатывает очередь сообщений и таймауты

### `RemoteLib.js` (singleton)
- Скачивает `.inpx` файл с удалённого сервера через WebSocket
- Режим: `config.remoteLib = { url, accessPassword }`

### `FileDownloader.js`
- HTTP-скачивание файлов из удалённых источников (axios)
- Используется для скачивания книг в режиме удалённой библиотеки

### `ZipReader.js`
- Обёртка над `node-stream-zip` для чтения zip-архивов с книгами
- Поддерживает cp866 в именах файлов

### `InpxParser.js`
- Парсит `.inpx` (ZIP с XML-файлами) через `XmlParser`
- Читает структуру: заголовки (`structure.xml`), книги, жанры

## Конфигурация (`server/config/`)

### Важные параметры `base.js`

| Параметр | По умолчанию | Описание |
|----------|-------------|----------|
| `accessPassword` | `''` | Пароль (пусто = без пароля) |
| `accessTimeout` | `0` | Таймаут сессии в минутах (0 = бесконечно) |
| `extendedSearch` | `true` | Расширенный поиск |
| `bookReadLink` | `/reader/?${DOWNLOAD_LINK}` | Шаблон ссылки для читалки |
| `dbVersion` | `'12'` | Версия схемы БД (инвалидирует кеш при изменении) |
| `dbCacheSize` | `5` | Размер кеша jembadb |
| `queryCacheMemSize` | `50` | Кеш запросов в памяти (MB) |
| `queryCacheDiskSize` | `500` | Кеш запросов на диске (MB) |
| `inpxCheckInterval` | `60` | Интервал проверки .inpx (мин) |
| `maxPayloadSize` | `500` | Макс. WS-сообщение (MB) |
| `lowMemoryMode` | `false` | Режим экономии памяти |
| `remoteLib` | `false` | Режим удалённой библиотеки |
| `server.port` | `'22380'` | Порт (в README указан 12380) |

### Директории (задаются при старте)

| Переменная | Путь по умолчанию |
|-----------|------------------|
| `dataDir` | `<execDir>/.inpx-web` |
| `tempDir` | `<dataDir>/tmp` |
| `logDir` | `<dataDir>/log` |
| `publicDir` | `<dataDir>/public` |
| `bookDir` | `<dataDir>/public-files/book` |

## Рецепты разработки

### Добавление нового WS-метода

1. В `WebWorker.js` добавить обработчик в `switch(action)` или соответствующий метод
2. В `DbSearcher.js` добавить метод поиска (если нужен новый тип запроса)
3. В клиенте (`client/components/Api/Api.vue`) вызвать новый action через WS

### Добавление нового типа поиска

1. `DbCreator.js` — создать нужный индекс при наполнении БД
2. `DbSearcher.js` — добавить метод с `db.select({table, where, ...})`
3. `WebWorker.js` — подключить метод к WS-роутингу

### Работа с jembadb

```js
const { JembaDbThread } = require('jembadb');
const db = new JembaDbThread();
await db.lock({ dbPath, create: true });
await db.openAll();

// Запросы
const rows = await db.select({ table: 'books', where: `@@id('someId')` });
await db.insert({ table: 'books', rows: [{...}] });
await db.update({ table: 'books', where: `...`, mod: `{field: 'newVal'}` });
await db.delete({ table: 'books', where: `...` });

await db.unlock();
```

### Добавление внешнего конвертера

1. Добавить класс в `server/core/`
2. Зарегистрировать в `WebWorker.js`
3. Добавить конфигурацию в `server/config/base.js` → `external`
4. Создать `external_tools.json` рядом с `config.json`

### Логирование

```js
const log = new (require('./AppLogger'))().log; // singleton
log('Обычное сообщение');
log(LM_ERR, 'Ошибка');
log(LM_FATAL, 'Критическая ошибка');
log(LM_WARN, 'Предупреждение');
```

## Сигналы и выход

`AsyncExit` (singleton) собирает cleanup-коллбэки через `asyncExit.add(fn)`.
При завершении процесса все коллбэки вызываются последовательно (закрытие БД и т.д.).
