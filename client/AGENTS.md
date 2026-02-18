# Client: Frontend (Vue 3 + Quasar)

## Общее описание

SPA на Vue 3 + Quasar v2. Общается с сервером **только через WebSocket** (поиск, скачивание, данные).
HTTP используется только для загрузки статики и скачивания файлов книг по прямой ссылке.

Сборка: Webpack 5 (`build/webpack.*.config.js`). Dev-режим: HMR через webpack-dev-middleware (без отдельного dev-сервера, встроено в Express backend).

## Структура директории

```
client/
├── main.js                   # Точка входа: createApp, router, store, Quasar, монтирование
├── router.js                 # Vue Router (hash history): /, /author, /series, /title, /extended
├── quasar.js                 # Quasar инициализация и конфиг плагинов
├── index.html.template       # HTML-шаблон для Webpack HtmlWebpackPlugin
├── components/
│   ├── App.vue               # Корневой компонент (монтирует Api + Search)
│   ├── vueComponent.js       # Базовый класс для компонентов (class-based → defineComponent)
│   ├── Api/
│   │   ├── Api.vue           # WS API: busy-диалог, прогресс, аутентификация, все WS-вызовы
│   │   └── webSocketConnection.js  # Экземпляр WS-соединения (singleton, автоURL из location)
│   ├── Search/
│   │   ├── Search.vue        # Главный экран: панель поиска, переключение разделов
│   │   ├── BaseList.js       # Базовый класс для списков (пагинация, поиск, события)
│   │   ├── AuthorList/
│   │   │   └── AuthorList.vue    # Список авторов с раскрытием книг
│   │   ├── SeriesList.vue    # Список серий
│   │   ├── TitleList.vue     # Список книг (раздел "Книги")
│   │   ├── ExtendedList.vue  # Расширенный поиск (</>)
│   │   ├── BookView.vue      # Карточка/строка книги (скачать, ссылка, читалка)
│   │   ├── BookInfoDialog.vue    # Диалог "Информация о книге" (обложка, жанры, метаданные)
│   │   ├── SettingsDialog.vue    # Диалог настроек UI
│   │   ├── LoadingMessage.vue    # Индикатор загрузки
│   │   ├── PageScroller.vue  # Кнопка "вверх" для длинных списков
│   │   ├── SelectDateDialog.vue  # Диалог выбора диапазона дат
│   │   ├── SelectExtDialog.vue   # Диалог выбора формата файла
│   │   ├── SelectExtSearchDialog.vue # Расширенный выбор формата
│   │   ├── SelectGenreDialog.vue # Дерево жанров
│   │   ├── SelectLangDialog.vue  # Выбор языка
│   │   ├── SelectLibRateDialog.vue   # Выбор оценки
│   │   ├── authorBooksStorage.js     # In-memory кеш книг авторов (избегает повторных WS-запросов)
│   │   └── assets/           # Картинки (logo.png и др.)
│   └── share/                # Переиспользуемые мелкие компоненты
│       ├── Dialog.vue        # Базовый диалог (Quasar q-dialog обёртка)
│       ├── StdDialog.vue     # Стандартный диалог с кнопками OK/Cancel
│       ├── DivBtn.vue        # Кнопка-div с иконкой (Quasar-стиль)
│       ├── Notify.vue        # Уведомления (Quasar Notify)
│       └── NumInput.vue      # Числовой ввод с валидацией
├── store/
│   ├── index.js              # createStore + VuexPersistence (localStorage)
│   └── root.js               # Корневой модуль (namespaced: true)
│                             #   state: config (с сервера), settings (локальные)
│                             #   mutations: setConfig, setSettings
└── share/                    # Утилиты (не Vue-компоненты)
    ├── utils.js              # Общие хелперы (форматирование, строки)
    ├── cryptoUtils.js        # Крипто-утилиты (для accessToken)
    ├── diffUtils.js          # Diff-утилиты
    └── sjcl.js / sjclWrapper.js  # Stanford JS Crypto Library + обёртка
```

## Ключевые паттерны

### Class-based компоненты (`vueComponent.js`)

Вместо стандартного Vue Options API используется класс с `defineComponent`:

```js
import vueComponent from '../vueComponent.js';

class MyComponent {
    _options = { components: {}, watch: {}, emits: [] };
    _props = { someProp: String };

    // data (instance fields)
    myData = 'value';

    // methods (class methods)
    myMethod() { ... }

    // computed (геттеры)
    get myComputed() { return this.myData + '!'; }
}

export default { components: { MyComponent: vueComponent(MyComponent) } };
```

`vueComponent.js` преобразует класс: поля → `data()`, методы → `methods`, геттеры → `computed`.

### WebSocket API (`Api.vue`)

Все запросы к серверу — через `Api.vue`. Компонент используется как provide/inject или через ref в родителе.

```js
// Внутри Api.vue — пример вызова
async search(query) {
    return await this.request('search', query);
}

// request() отправляет WS-сообщение и ждёт ответа
```

`webSocketConnection.js` — singleton `WebSocketConnection` (из `server/core/WebSocketConnection.js`),
URL вычисляется автоматически из `window.location` (ws:// или wss://).

> **Важно**: `client/components/Api/webSocketConnection.js` **импортирует серверный код**
> `server/core/WebSocketConnection.js` напрямую. Это намеренно — один класс работает и в браузере (через `window.WebSocket`), и на Node.js (через `ws`).

### Vuex Store

Единый модуль `root` (namespaced):

| Состояние | Тип | Описание |
|-----------|-----|----------|
| `config` | Object | Конфиг с сервера (`webConfigParams`) |
| `settings.accessToken` | String | Токен сессии для аутентификации |
| `settings.limit` | Number | Кол-во результатов на страницу |
| `settings.downloadAsZip` | Boolean | Скачивать как ZIP |
| `settings.showCounts` | Boolean | Показывать счётчики |
| `settings.showRates` | Boolean | Показывать оценки |
| `settings.langDefault` | String | Язык поиска по умолчанию |

Состояние персистируется в localStorage через `VuexPersistence` (при обновлении страницы сохраняется).

### Маршрутизация (router.js)

Hash-история (`createWebHashHistory`). Все маршруты рендерят `Search.vue`:

| Путь | Раздел |
|------|--------|
| `/` | Авторы (по умолчанию) |
| `/author` | Авторы |
| `/series` | Серии |
| `/title` | Книги |
| `/extended` | Расширенный поиск |

## Зависимости (ключевые)

| Пакет | Назначение |
|-------|-----------|
| `vue@3` | Реактивный UI-фреймворк |
| `quasar@2` + `@quasar/extras` | UI компоненты, иконки |
| `vue-router@4` | Клиентская маршрутизация |
| `vuex@4` + `vuex-persist` | Стейт-менеджмент + персистентность |
| `axios` | HTTP-запросы (скачивание файлов) |
| `lodash` | Утилиты |
| `localforage` | IndexedDB/localStorage (через VuexPersistence) |
| `dayjs` | Форматирование дат |

## Рецепты разработки

### Добавить новый WS-запрос

1. В `server/WebWorker.js` добавить обработчик action
2. В `client/components/Api/Api.vue` добавить метод, вызывающий `this.request(action, args)`
3. В нужном компоненте вызвать метод через ref к `Api`

### Добавить новый диалог

1. Создать компонент в `client/components/Search/` или `client/components/share/`
2. Использовать `Dialog.vue` или `StdDialog.vue` как основу
3. Управлять видимостью через `v-model` (boolean в родительском data)

### Добавить поле в настройки UI

1. `client/store/root.js` → добавить поле в `state.settings` с дефолтом
2. `client/components/Search/SettingsDialog.vue` → добавить UI-элемент
3. Использовать `this.$store.commit('root/setSettings', { newField: value })`

### Добавить новый раздел поиска

1. `client/router.js` → добавить маршрут
2. `client/components/Search/Search.vue` → добавить опцию в `listOptions` и логику переключения
3. Создать компонент-список (наследовать `BaseList.js`)
4. `server/WebWorker.js` → добавить обработчик для нового типа поиска

## Dev-режим

```bash
# В корне проекта (запускает и сервер, и webpack HMR через nodemon)
npm run dev
```

Webpack HMR middleware встроен в Express сервер (см. `server/dev.js`).
Клиентский код пересобирается автоматически при изменениях.
Для смены порта: `--port=<port>` в CLI или `server.port` в `config.json`.
