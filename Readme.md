# Набор программных компонентов для реализации автоматической синхронизации доменных данных и состояний клиентских приложений между (clilent <-> server)

Состоит из:

- Cache (для работы с доменными данными)
- AppState (для работы с состояниями приложений)

## Требования

- Максимальное снижение бойлерплейта
- Основанно на схемах
- Минимум настроек для начала работы
- Декларативная конфигурируемость (при необходимости)
- Поддержка маппинга данных
- Поддержка работы без интернета
- Поддержка работы с перебоями интернет соединения
- Поддержка сервера одновременной работы с несколькими приложениями
  - для разных типов приложений - web, mobile
  - для приложений с разным назначением

## Особенности работы

Основная конфигурация на сервере, клиентское конфигурирование расширяет серверную настройку

> Ключи описанные на сервере, после установки соединения доступны на клиенте с автоматической синхронизацией client <-> server

- Если доступ к данным нужен до установки соединения с сервером, донастраиваем на клиенте.
- Если ключи(fragments) нужны только на клиенте (локальные) - объявить нужно в конфиге на клиенте
- Если данные должны сохраняться локально (локал сторедж) - нужно настроить в конфиге на клиенте

```javascript

// '<server>/application/config/SyncStorage.js'
([
  {
    scope: "mobile/ShopWindow",
    cache: {
      autoUpdatedTime: '500ms', // время автообновления данных для всех "fragments" (можно переназначить внутри фрагмента)
      updateIntervalLimit: '100ms' // запрос данных не чаще чем 1раз в 100 миллесекунд,
      fragments: {
        "products": {
          fetchHandler: "/application/api/shop/produtsQuery", // данные из эндпоинта будут положены в кеш и отправлены на клиент
          item: { // настройки для кеширования елемента из списка
            uniqueField: "meta.id", // уникальный ключь , будет взят из указанного поля
            stillTime: '180s', // время кеширования элемента - 180сек
          },
          retry: {
            timeout: 200,  // таймаут между попытками retry  (по умолчанию рост экспоненциальный)
            count: 3
          },
        },

        "locationList": { // автоматически доступен на клиенте (в конфиге на клиенте мы его не видим)
          updateHandler: "application/api/town/locationsQuery", // данные из эндпоинта будут положены в кеш и отправлены на клиент
          transform: { // настройки отображения данных (в случае если нужен тонкий клиент)
            forClient: '/application/lib/locations/transformDataForClient', // данные которые вернет будут положены в кеш на клиенте
          }
      }
    },

    state: {
      "currentLocation": {
        retry: {
          timeout: 200,  // таймаут между попытками retry  (по умолчанию рост экспоненциальный)
          count: 3
        },
        linked: {
          // настройки обмена данными между разными scope
          // например нужно из этого поля обновлять поле стейта внутри {scope: "mobile/Runner}
          // скорее всего лучше будет сделать через автовызов api endpoint
        }
      }
    }
  }
  { scope: "web/ShopAdministration", /* ... */ }
  { scope: "modile/Runner", /* ... */ }
])

```

``` javascript

// '<client>/config/SyncStorage.js'
export const {Cache, AppState} = new SyncStorage({
    scope: "mobile/ProductShop",
    localStorage: StorageAdapter, // import another storage
    chache: {
      autoUpdatedTime: '300ms', // запрос свежих данных каждые 300 миллесекунд (перезаписали конфиг сервера)
      fragments: {
        'products': {
          default: [], // начальное значение (доступно до установления соединения с сервером)
          local: { // настройки хранения в локальном хранилище
            saved: true // сохранять в локальное хранилище
            cripted: false // шифровать сохраняемые данные
          }
          transofrm: {
            forClient: lib.transformFunction // трансформация данных на клиенте
          }
        }
      }
    },
  state: {
    "currentLocation": {
      autoUpdatedTime: '500ms', // запрос свежих данных каждые 300 миллесекунд (перезаписали конфиг сервера)
    },
    "currentLocationName": {
      default: "", // начальное значение (доступно до установления соединения с сервером)
      local: { // настройки хранения в локальном хранилище
        saved: true // сохранять в локальное хранилище
        cripted: false // шифровать сохраняемые данные
      }
    }, // поле не синхронизируется с сервером (только на клиенте храниться)
    "latestScreen": {} // поле не синхронизируется с сервером (только на клиенте храниться)
  },
  "orderForms" : { // не синхронизируется с сервером (только на клиенте доступен)
    item: { // настройки для кеширования елемента из списка
      uniqueField: "groupName", // уникальный ключь , будет взят из указанного поля
      stillTime: '180s', // время кеширования элемента - 180сек
    }
  }
})
```

### Cache

#### Хранилище доменных данных

Особенности

- Автоматическая синхронизация данных с сервером
  - Источник данных - результат вызова API ендпоинта.
  - На клиенте можем только читать данные и инвалидировать кеш.
- Можем настроить отображение данных
  - Отображение делается на сервере
    (server -> transform -> client -> cache)
  - Обображение делается на клиенте
    (server -> client -> transform -> chache)
- Можем настроить автоматическое сохранение данных в локальном хранилище (localStorage)
  - По умолчанию не сохраняет в локальное хранилище
- Если в кеше хранится список, а нам нужно быстро получить доступ к некоторым его елементам:
        эти элементы автоматически кешируются как самостоятельные
  - необходимо настроить (указать уникальный ключ, для доступа к элементу)
  - при получении данных с сервера эти элементы автоматически перекешируются

``` javascript
// <client>
// Cache - получаем после конфигурации (выше по файлу)

// получение данных из кеша
const products = await Cache.getValue('products')

Cache.subscribe('products', ({eventType}) => {
  // EventTypes: getValue | fetchValue | updateValue | updateError | invalidated | serverSync

  /* Special params for event
    getValue: {value, isDefault, isLoading}
    updatedValue: {value}
    serverSync: {isSynced}
    updateError: {retryCount, currentTimeout}
  */

})

// получение элемента (в конфиге указали по какому полю искать -  "meta.id")
const clickProductId = "aa-11-bb"
const simpleProductData = await Cache.getValue(['products', clickProductId])
Cache.subscribe(['products', clickProductId], ({eventType}) => {})

// инвалидация
await Cache.invalidate('products')
await Cache.invalidateAll()

// общие подписки
Cache.onError(({key, error}) => {})
Chache.onUpdate(({key, value}) => {}) // сработает после обновления поля в кеше
Cache.onInvalidate(({key}) => {})

// Отписки
Cache.unsubscribe('products')
Cache.unsubscribeAll()

// получить список всех доступных ключей (синхронизируемые + локальные)
await Cache.introspection()


```

``` javascript
  //<server>/application/api/Other/AnotherApiEndpoint.js

  ({
    method: async () => {
      SyncStorage.Cache.invalidate('products') // запустит хендлер для обновления данных
    }
  })
```

### AppState

#### Хранилище состояний систем приложения

- set, get данных
- нет инвалидации
- нет трансформаций данных

``` javascript
// <client>

const locationName = await AppState.get("currentLocationName")

Cache.subscribe('currentLocationName', ({eventType}) => {
  // EventTypes: getValue | fetchValue | updateValue | updateError | toDefault | serverSync

  /* Special params for event
    getValue: {value, isDefault, isLoading}
    updatedValue: {value}
    serverSync: {isSynced}
    updateError: {retryCount, currentTimeout}
  */

})

// получение элемента (в конфиге указали по какому полю искать -  "name")
const formGroupName = "name0989"
const formData = await Cache.getValue(['orderForms', formGroupName])
Cache.subscribe(['orderForms', formGroupName], ({eventType}) => {})

const newLocationName = "Another name"
await AppState.set("currentLocationName", newLocationName)

await AppState.toDefault()

// общие подписки
AppState.onError(({key, error}) => {})
CAppState.onUpdate(({key, value}) => {}) // сработает после обновления поля в кеше

// Отписки
AppState.unsubscribe('orderForms')
AppState.unsubscribeAll()

// получить список всех доступных ключей (синхронизируемые + локальные)
await AppState.introspection()


```
