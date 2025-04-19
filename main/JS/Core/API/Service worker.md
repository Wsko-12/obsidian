- Что такое ServiceWorker?
- Какие стадии жизненного цикла у ServiceWorker?
- Зачем нужен `event.waitUntil()`
- Что рекомендуется делать на каждой стадии?
- Когда обновляется SW?
- Как будет идти перехват запросов при первой установке/переустановке?
- Как это поведение контролировать?
- Как пользоваться кешом?
- Что еще можно делать с помощью SW?

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


## Установка
В этом событии можно:
- Предварительно закэшировать необходимые файлы.

Так же следует упомянуть, что в sw используются ExpandableEvents,
а именно `event.waitUntil(Promise)`
Это нужно, если в `install` будет использоваться асинхронное API, чтобы установка завершилось тогда, когда все прошло успешно и activate не вызывался раньше времени

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
  
## Активация
На этом этапе следует делать очистку старых кэшей, то есть удалять устаревшие данные из кэша, которые использовались предыдущими версиями Service Worker.

Для того, чтобы `fetch` начал отрабатывать сразу, после **самого первого** `activate`, следуют вызвать `event.waitUntil(self.clients.claim());`.

Eсли `activate`  был вызван немедленно с помощью `self.skipWaiting()` в `install`, и была открыта старая вкладка и открывается новая, где загружается обновленный SW, то на старой вкладке запросы будут тоже идти через новый `fetch`  даже **БЕЗ** `event.waitUntil(self.clients.claim());`. На ней так же вызовется `oncontrollerchange`
#### Возможные кейсы:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<button id="button">CLICK</button>
<script>
  navigator.serviceWorker.register("./sw.js", {scope: "./"}).then((reg) => {
    console.log(reg)
  })

  button.onclick = function (){
    document.createElement("img").src = "texture.png"
  }
</script>
</body>
</html>
```

```js
//sw.js

const version = "V1"
self.addEventListener('fetch', (e) => {
    console.log(version, e.request);
    e.respondWith(fetch(e.request))
})
self.addEventListener('install', (ev) => {
    console.log(`install ${version}`)
    // self.skipWaiting();
})

self.addEventListener('activate', (ev) => {
    console.log(`activate ${version}`)
    // ev.waitUntil(self.clients.claim())
})
```


Самый обычный, простая установка без скипов:
```
SERVICE WORKER V1 DEPLOED
Open TAB A
INSTALL V1
ACTIVATE V1
- NO FETCH
Reload TAB A
FETCH V1
```

Самая первая установка, но открывается вторая вкладка после первой:
```
SERVICE WORKER V1 DEPLOED 
Open TAB A
INSTALL V1
ACTIVATE V1                 
-                               OPEN TAB B
- NO FETCH                      FETCH V1
Reload TAB A
FETCH V1
```

Самая первая установка, но открыта вкладка до того, как SW задеплоен
```
Open TAB A
SERVICE WORKER V1 DEPLOED 
-                            OPEN TAB B
-                            INSTALL V1
-                            ACTIVATE V1   
-                            NO FETCH
Reload TAB A
FETCH V1                     NO FETCH
```

Если так же зеркально релоаднуть другую, то все равно на первой ничего не будет:

```
Open TAB A
SERVICE WORKER V1 DEPLOED 
-                            OPEN TAB B
-                            INSTALL V1
-                            ACTIVATE V1   
-                            Reload TAB B
- NO FETCH                   FETCH V1  
```

Точно такое же поведение будет, если использовать `self.skipWaiting()` в `install`.
Чтобы начать перехват запросов сразу, следует вызвать `ev.waitUntil(self.clients.claim())` в `activate`

```
SERVICE WORKER V1 DEPLOED
Open TAB A
INSTALL V1
ACTIVATE V1
ev.waitUntil(self.clients.claim())
FETCH V1
```

Понятное дело, что если вторая вкладка откроется после первой, то ее fetch тоже будет перехватываться.

Теперь, первая вкладка открылась до деплоя SW и вызывается `ev.waitUntil(self.clients.claim())` в `activate`

```
Open TAB A
SERVICE WORKER V1 DEPLOED 
-                            OPEN TAB B
-                            INSTALL V1
-                            ACTIVATE V1  
-                            ev.waitUntil(self.clients.claim())
FETCH V1                     FETCH V1
```

Теперь переезд на новую версию SW
```
SERVICE WORKER V2 DEPLOED
Open TAB A
INSTALL V2
FETCH V1
Reload TAB A
FETCH V1
Close TAB A
Open TAB A
INSTALL V2 (IDK)
ACTIVATE V2
FETCH V2
```

Точно так же будет, если открыть вторую вкладку. SW будет ждать, пока все вкладки использующие его не закроются полностью.

Если изменить это поведение с помощью `self.skipWaiting()` в `install`.
```
SERVICE WORKER V2 DEPLOED
Open TAB A
INSTALL V2
self.skipWaiting()
ACTIVATE V2
FETCH V2
```

Если вкладка была открыта до деплоя SW V2  `self.skipWaiting()` в `install`.

```
Open TAB A
FETCH V1
SERVICE WORKER V2 DEPLOED          Open TAB B
-                                  INSTALL V2
-                                  self.skipWaiting()
-                                  ACTIVATE V2
FETCH V2                           FETCH V2 
```

## Перехват запросов
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

      // В случае, если данные не были найдены в кеше, получаем их с сервера. и кладем в кеш
      const fresh = await fetch(event.request);
      cache.put(event.request, fresh.clone());
	  return fresh;
    })(),
  );
});
```

Если мы хотим положить response в кеш, нужно его клонировать. При этом, можно сначала ответить браузеру, затем с помошью `ev.waitUntil` класть ответ в кеш


## Background Synchronization API
https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API

## Push API
```js

// main.js
async function setUpPush(){
	const registration = await navigator.serviceWorker.register('push-sw.js');
	const subscription = await registration.pushManager.getSubscription();
	
	if(subscription){
		return;
	}
	
	const newSubscription = await registration.pushManager.subscribe({
		userVisibleOnly: true,
		// generated by require("web-push").generateVAPIDKeys()
		applicationServerKey: "some-key" 
	});
	
	sendSubscriptionToServer(newSubscription);
}

setUpPush()

// push-sw.js
self.addEventListener('push', function(event) {
	event.waitUntil(
		self.registration.showNotification('Title', {
			body: event.data ? event.data.text() : 'no payload',
		})
	);
});

//server.js
const webPush = require("web-push");

// generate by webPush.generateVAPIDKeys()
const VAPID_PUBLIC_KEY = "some-key";
const VAPID_PRIVATE_KEY = "some-key";

// Set the keys used for encrypting the push messages.
webPush.setVapidDetails(
  "https://example.com/",
  VAPID_PUBLIC_KEY,
  VAPID_PRIVATE_KEY
);

function sendPush(){
	// Subscription from client
	const subscription = {
	  "endpoint": "....",
	  "expirationTime": null,
	  "keys": {
	      "p256dh": "....",
	      "auth": "....."
	  }
	}

	const payload = "Hello world";
	const options = {
	  TTL: 10000,
	};

	webPush.sendNotification(subscription, payload, options)
}
sendPush();

```


https://developer.mozilla.org/ru/docs/Web/API/Push_API
https://github.com/mdn/serviceworker-cookbook/tree/master/push-payload
https://www.youtube.com/watch?v=2zHqTjyfIY8&ab_channel=Ashotofcode
https://github.com/web-push-libs/web-push
https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Tutorials/js13kGames/Re-engageable_Notifications_Push#push


Links:
https://github.com/mdn/serviceworker-cookbook/
https://www.youtube.com/playlist?list=PLyuRouwmQCjl4iJgjH3i61tkqauM-NTGj





