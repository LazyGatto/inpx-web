# inpx-web: Веб-сервер поиска по .inpx-коллекциям

## Общее описание

Веб-сервис для поиска и скачивания книг из `.inpx`-коллекций (индексных файлов сетевых библиотек типа MyHomeLib/freeLib).
Предоставляет веб-интерфейс и OPDS-сервер. Поддерживает работу в режиме "удалённой библиотеки".

- **Версия**: 1.5.8.1
- **Лицензия**: CC0-1.0
- **Репозиторий**: bookpauk/inpx-web
- **Node.js**: >= 16.16.0
- **Порт по умолчанию**: 12380 (README) / 22380 (base.js)
- **OPDS**: `/opds`

## Технологический стек

| Слой | Технология |
|------|-----------|
| Backend | Node.js, Express, WebSocket (`ws`) |
| БД | jembadb (встроенная) |
| Frontend | Vue 3, Quasar v2, Vuex 4, Vue Router 4 |
| Сборка | Webpack 5, `pkg` (упаковка в исполняемый файл) |
| Packaging | pkg (Linux x64/arm64, Windows x64, macOS x64) |

## Архитектура

```
inpx-web/
├── server/                  # Node.js backend
│   ├── index.js             # Точка входа: инициализация, Express + WebSocket сервер
│   ├── createWebApp.js      # Создание production web app (статика)
│   ├── static.js            # Сервинг статических файлов
│   ├── dev.js               # Dev-режим: webpack HMR middleware
│   ├── nodemon.json         # Конфиг nodemon для dev-режима
│   ├── core/                # Ядро приложения
│   ├── controllers/         # HTTP/WS контроллеры
│   └── config/              # Конфигурация
├── client/                  # Vue 3 SPA (Quasar)
│   ├── main.js              # Точка входа Vue приложения
│   ├── router.js            # Vue Router (hash history)
│   ├── quasar.js            # Quasar инициализация
│   ├── index.html.template  # HTML шаблон
│   ├── components/          # Vue компоненты
│   │   ├── App.vue          # Корневой компонент
│   │   ├── Api/             # WebSocket API клиент
│   │   │   ├── Api.vue      # Основной API компонент
│   │   │   └── webSocketConnection.js  # WS клиент (браузер)
│   │   ├── Search/          # Поисковый интерфейс
│   │   └── share/           # Общие мелкие компоненты (Dialog, Notify и др.)
│   ├── store/               # Vuex store
│   │   ├── index.js         # Создание store (VuexPersistence)
│   │   └── root.js          # Корневой модуль: config, settings
│   └── assets/              # Статические ресурсы (шрифты, картинки)
├── build/                   # Webpack конфигурации и скрипты сборки
│   ├── webpack.base.config.js
│   ├── webpack.dev.config.js
│   ├── webpack.prod.config.js
│   ├── prepkg.js            # Подготовка к pkg-упаковке
│   ├── release.js           # Скрипт создания релизов
│   └── appdir.js
├── examples/                # Примеры конфигурации
├── Dockerfile               # Docker-образ
├── docker_entrypoint.sh     # Точка входа Docker
├── makefile                 # Make-задачи
├── package.json             # Зависимости и npm-скрипты
└── nodemon.json             # Конфиг nodemon
```

## Основной поток данных

```
Браузер (Vue/WS) ──► WebSocketController ──► WebWorker ──► DbSearcher ──► jembadb
                                                       ──► DbCreator (при запуске)
                                                       ──► FileDownloader (скачивание)
```

1. Клиент открывает WebSocket соединение с сервером
2. `WebSocketController` принимает сообщения и делегирует `WebWorker`
3. `WebWorker` (singleton) управляет состоянием БД, поиском и скачиванием
4. `DbSearcher` выполняет запросы к jembadb
5. Прогресс долгих операций отслеживается через `WorkerState`

## Ключевые принципы

- **Вся логика — на сервере**: клиент только отображает данные и отправляет запросы
- **WebSocket** — основной протокол для поиска и скачивания (не REST)
- **REST/HTTP** — только для статики и OPDS
- **jembadb** — встроенная NoSQL БД (работает в отдельном потоке через JembaDbThread)
- **Singleton-паттерн**: WebWorker, WorkerState, AppLogger, AsyncExit — одиночки
- **pkg** — упаковывает Node.js приложение в самодостаточный исполняемый файл

## Команды для работы с проектом

```bash
# Dev-режим (nodemon + webpack HMR)
npm run dev

# Сборка клиента (production)
npm run build:client

# Сборка релиза для конкретной платформы
npm run build:linux
npm run build:win
npm run build:macos
npm run build:linux-arm64

# Сборка всех релизов
npm run build:all
```

## Конфигурация

- Конфиг при первом запуске создаётся в `<execDir>/.inpx-web/config.json`
- Базовые значения: `server/config/base.js`
- CLI-параметры перекрывают config.json
- Ключевые параметры: `accessPassword`, `opds`, `remoteLib`, `bookReadLink`, `uiDefaults`

## Соглашения по коду

- **CommonJS** на backend (`require`/`module.exports`)
- **ES Modules** на frontend (`import`/`export`)
- Backend-логи через синглтон `AppLogger`: `log(LM_ERR, ...)`, `log(LM_FATAL, ...)`
- Конфиг передаётся в классы через конструктор: `new MyClass(config)`
- Таблицы jembadb: `books`, `authors`, `series`, `titles`, `genres`, `query_cache`, `query_time`

## Связанные AGENTS-файлы

- `server/AGENTS.md` — подробное описание backend-модулей
- `client/AGENTS.md` — подробное описание frontend (Vue 3 + Quasar)
