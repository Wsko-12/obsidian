
Service worker - это как прокси между веб приложением и браузером/сетью. Позволяет перехватывать запросы и тем самым, например, кешировать их.

Service Worker будет следовать следующему жизненному циклу:

1. Загрузка
2. Установка
3. Активация

##### Загрузка/Регистрация
https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register

Когда браузер встречает вызов `navigator.serviceWorker.register()`, он загружает файл Service Worker. Если файл успешно загружен, браузер сравнивает его с предыдущей версией.
Если файл отличается, начинается процесс установки нового Service Worker, а если нет, браузер продолжает использовать текущую версию.

После этого он будет загружаться каждые 24 часа или около того. Он _может_ загружаться и чаще, но он **должен** загружаться как минимум каждые 24 часа, чтобы предотвратить использование старой версии кода клиентом. (https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API)

Вторым аргументом метод register принимает options:

`scope` - Про него подробнее ниже
`type` - `classic` | `module`
`updateViaCache` - `all` | `imports` | `node` (https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register#updateviacache) это относится только к скрипту Service Worker и его импортам, а не к другим ресурсам, извлекаемым этими скриптами.

###### Значения `updateViaCache` и их поведение

| Значение               | Описание                                                                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `'all'` (по умолчанию) | Использует кэш браузера для загрузки как самого файла Service Worker, так и импортируемых ресурсов.                                                           |
| `'none'`               | Всегда обходит кэш браузера и загружает все ресурсы с сервера. Это полезно для более агрессивного обновления.                                                 |
| `'imports'`            | Использует кэш браузера только для импортируемых ресурсов внутри Service Worker (например, через `importScripts()`). Сам Service Worker загружается без кэша. |

###### Значения `scope` 
**`scope`** задает **область действия (контекст)** Service Worker, то есть определяет, какие URL-адреса на сайте будут обслуживаться этим Service Worker.

Например `{ scope: '/app/' }`:
- `/app/index.html` — обслуживается Service Worker
- `/app/style.css` — обслуживается Service Worker
- `/about.html` — **не обслуживается**, так как не входит в указанный `scope`

Так же будет, если сервис воркер загружен из `/app/sw.js`, так как Service Worker не может иметь область действия шире своего собственного местоположения.

Но это можно расширить, если дать `{scope: "/"}`, однако для этого нужен заголовок на `sw.js` `Service-Worker-Allowed: <scope>`. Если сервер не установил заголовок, регистрация сервисного работника завершится неудачей, так как запрошенный объект `scope`слишком широк.

```js
if ("serviceWorker" in navigator) {
  // Declaring a broadened scope
  navigator.serviceWorker.register("./sw.js", { scope: "/" }).then(
    (registration) => {
      // The registration succeeded because the Service-Worker-Allowed header
      // had set a broadened maximum scope for the service worker script
      console.log("Service worker registration succeeded:", registration);
    },
    (error) => {
      // This happens if the Service-Worker-Allowed header doesn't broaden the scope
      console.error(`Service worker registration failed: ${error}`);
    },
  );
} else {
  console.error("Service workers are not supported.");
}
```

Можно удалить все регистрации используя:
```js
navigator.serviceWorker.getRegistrations().then((regs) => {  
	regs.forEach((r) => r.unregister())  
})
```


##### Установка
В этом событии можно:
- Предварительно закэшировать необходимые файлы.

Если в `install` будет использоваться асинхронное API, следует использовать `event.waitUntil`, чтобы установка завершилось тогда, когда все прошло успешно

```js
const SW = 1  
  
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));  
self.addEventListener('install', (ev) => {  
	ev.waitUntil(delay(10000))  
})
```

при этом в браузер в дев тулзе 10s будет писать:
```
#0000 trying to install
```


После успешной установки есть два варианта событий:
1. Это самый первый Service Worker. В этом случае SW сразу перейдет в активацию
2. Если service worker уже существует, новая версия устанавливается в фоновом режиме, но не активируется — worker переходит в состояние _в ожидании_. Новая версия активируется только тогда, когда больше не останется загруженных страниц, использующих старый service worker.

Активация может произойти раньше при использовании `self.skipWaiting()`
  
##### Активация
На этом этапе следует делать очистка старых кэшей, то есть удалять устаревшие данные из кэша, которые использовались предыдущими версиями Service Worker.

Для того, чтобы `fetch` начал отрабатывать сразу, после первого `activate`, следуют вызвать `event.waitUntil(self.clients.claim());`.

Eсли `activate`  был вызван немедленно с помощью `self.skipWaiting()` в `install`, и была открыта старая вкладка и открывается новая, где загружается обновленный SW, то на старой вкладке запросы будут тоже идти через новый `fetch`  даже **БЕЗ** `event.waitUntil(self.clients.claim());`. На ней так же вызовется `oncontrollerchange`

Хотя в доке написано *Вызывает событие "`controllerchange`" на [`navigator.serviceWorker`](https://developer.mozilla.org/ru/docs/Web/API/ServiceWorkerContainer "navigator.serviceWorker") всех клиентских страниц, контролируемых сервис-воркером.* (https://developer.mozilla.org/ru/docs/Web/API/Clients/claim), оно отрабатывает и без него

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<img id="img" src="./img.png">
<script>
console.log("REFRESH")
  document.getElementById("img").addEventListener('click', () => {
    const img2 = new Image();
    img2.src = "./img2.png"
  })

  navigator.serviceWorker.register("./sw.js");

  const id = Math.random();
  console.log("ID", id);
  navigator.serviceWorker.oncontrollerchange = () => {
    console.log('CONTROLLER CHANGED', id)
  };
</script>
</body>
</html>
```

```js
const SW = 1
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
self.addEventListener('install', (ev) => {
    self.skipWaiting();
    console.log(`install ${SW}`)
    ev.waitUntil(delay(1000))
})

self.addEventListener('activate', (ev) => {
    // ev.waitUntil(self.clients.claim())
    console.log(`activate ${SW}`)
})


self.addEventListener('fetch', (ev) => {
    console.log(`fetch ${SW}`, ev.clientId)
})
```


```
// На второй вкладке, без релоада
REFRESH
ID 0.48171652829106426
fetch 0 395e8533-b5e7-421b-897d-4deaad223710 (sw.js)
fetch 0 b6536b25-b35c-4a55-aae6-e2ef071b3ff4 (sw.js)
...
// обновлили первую вкладку, тут пишет:
install 1 (sw.js)
CONTROLLER CHANGED 0.48171652829106426
activate 1 (sw.js)
// кликаем по картинке
fetch 1 395e8533-b5e7-421b-897d-4deaad223710 (sw.js)
```


##### Перехват запросов
Для того, чтобы начать перехватывать запросы, добавляем `fetch` event listenter. При этом в коллбек будет приходить fetchEvent. Этот реквест мы можем как просто пропустить через себя и ничего с ним не делать:

```js
self.addEventListener('fetch', (e) => {
	e.respondWith(fetch(e.request))
})
```


Так и достать реквест из кеша (если он там есть). Ну а дальше уже можно принимать разные стратегии, в зависимости от нужного поведения:
https://habr.com/ru/companies/2gis/articles/345552/

```js
self.addEventListener("fetch", (event) => {
  // В случае не-GET запроса браузер должен сам обрабатывать его
  if (event.request.method != "GET") return;

  // Обрабатываем запрос с помощью логики service worker
  event.respondWith(
    (async function () {
      // Пытаемся получить ответ из кеша.
      const cache = await caches.open("dynamic-v1");
      const cachedResponse = await cache.match(event.request);

      if (cachedResponse) {
        // Если кеш был найден, возвращаем данные из него
        // и запускаем фоновое обновление данных в кеше.
        event.waitUntil(cache.add(event.request));
        return cachedResponse;
      }

      // В случае, если данные не были найдены в кеше, получаем их с сервера.
      return fetch(event.request);
    })(),
  );
});
```

Если мы хотим положить response в кеш, нужно его клонировать. При этом, можно сначала ответить браузеру, затем с помошью `ev.waitUntil` класть ответ в кеш

Links:
https://github.com/mdn/serviceworker-cookbook/
https://www.youtube.com/playlist?list=PLyuRouwmQCjl4iJgjH3i61tkqauM-NTGj





