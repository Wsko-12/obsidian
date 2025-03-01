На заре интернета был придуман http протокол, который по свой сути является stateless.
И все было хорошо, пока интернет состоял чисто из html файлов как википедия.

Однако позже, появились сайты, на которых нужно было авторизироаться.
Как поддерживать сессию, если протокол без состояния?

Для этого придумали куки. То есть, когда пользователь первый раз заходит на сайт и авторизируется там, сервер отправлял команду браузеру установить куки с помощью заголовка Set-Cookie.

 И теперь, когда пользователь заходил на сайт, браузер прикреплял эти куки к запросу, и сервер знал, что к нему обращается авторизованный пользователь. И мог, например, вернуть html страничку, где в хедере было написано "Привет, John!"


Как **раньше** можно было отслеживать пользователя  между сайтами:

1. Пользователь посещает `https://a.com`, который внедряет контент из `https://tracker.com`. ``https://tracker.com`` устанавливает файл cookie на устройстве пользователя.
2. Пользователь посещает `https://b.com`, который также встраивает `https://tracker.com`. И теперь `https://tracker.com` может получить куку, которая была установлена на `https://a.com`

Это работало, потому что исторически файлы cookie хранились с ключом, основанным на имени хоста или домена сайта, который их установил, также известным как **ключ хоста** . В приведенном выше случае файл cookie будет храниться с ключом `("tracker.com")`

Теперь браузеры начали блокировать third-party cookie. 

Если ничего не менять, то теперь на втором пункте, когда запрос с `https://b.com` пойдет на `https://tracker.com`, браузер заблокирует cookie, которая была установлена, когда пользователь был на `https://a.com`. Заблокирует - значит просто не будет прикладывать этот cookie.


Окей, предположим, мы добросовестные разработчики и, который делают чат. И наш чат `https://chat.com` встраивают на сайте `https://a.com`. Нам бы хотелось, как-то хранить сессию пользователя, чтобы при каждом заходе на `https://a.com` пользователю не пришлось бы логиниться в нашем окошке чата.

Поэтому, были придуманы **CHIPS** (**Cookies Having Independent Partitioned State**)
Теперь, когда наш сервер хочет установить куку, должен добавить `Partitioned`

```
Set-Cookie: __Host-example=34d8g; SameSite=None; Secure; Path=/; Partitioned;
```

Теперь, ключом хранения для нашей куки будет `{("https://a.com"), ("chat.com")}`

Это значит, что когда `https//b.com` установит наш чат себе, кука отправлена не будет, так как для нее ключ это `{("https://b.com"), ("chat.com")}`

Это как механизм ***State Partitioning*

Тут важно заметить, что ключи формируются не **same-origin**, a **same-site** 

**same-origin**  - протокол (https) + домен (a.com) + порт (443);
**same-site** - **eTLD+1**. то есть: 
1. effective Top-Level Domain (.com, .github.io, .co.jp) 
2. domain (a)
3. Sheme (протокол) тоже учитывается, поэтому существует **Schemeless same-site**

https://web.dev/articles/same-site-same-origin

То есть если `https://a.com` создаст sub-domain `https://sub.a.com` и там установит наш `https://chat.com`, то наши куки все равно будут нам доступны


**НУЖНО УТОЧНЕНИЕ ПРАВИЛЬНО ЛИ Я ПОНЯЛ:**
Но что делать, если мы все же хотим использовать например один и тот же localStorage для `chat.com` на `a.com` и `b.com`

Пример:
- в своем `https://chat.com` в `localStorage` я храню установленную пользователем тему 
- сначала мой чат встроил `https://a.com`, туда зашел пользователь **John**, который выбрал темную тему 
 - Затем мой чат установил себе `https://b.com` И когда туда зайдет `John`, ему снова придется выбрать темную тему. 
 
 Но если бы я использовал Storage Access API и там и там, то при установке Джоном темной темы на `https://a.com`, я бы записал в localStorage theme: "dark" И это было бы доступно на сайте `https://b.com` после того, как я получил бы явное разрешение от него.

То есть, если John зашел на сайт `https://a.com`, где есть iframe `https://chat.com`, и в своем чате я буду делать запросы к стораджам, то ключом для моих стораджей будет partitioned
`a.com|chat.com`
и так же на `https://b.com`, там ключ будет `b.com|chat.com`,
но если там и там John дал разрешение, то ключ будет `chat.com`, то есть один и тот же localStorage (unpartitioned)

https://developer.mozilla.org/en-US/docs/Web/API/Storage_Access_API/Using#checking_and_requesting_storage_access

```javascript
function doThingsWithCookies() {
  document.cookie = "foo=bar"; // set a cookie
}

function doThingsWithLocalStorage(handle) {
  handle.localStorage.setItem("foo", "bar"); // set a local storage key
}

async function handleCookieAccess() {
  if (!document.hasStorageAccess) {
    // This browser doesn't support the Storage Access API
    // so let's just hope we have access!
    doThingsWithCookies();
  } else {
    const hasAccess = await document.hasStorageAccess();
    if (hasAccess) {
      // We have access to third-party cookies, so let's go
      doThingsWithCookies();
      // If we want to modify unpartitioned state, we need to request a handle.
      const handle = await document.requestStorageAccess({
        localStorage: true,
      });
      doThingsWithLocalStorage(handle);
    } else {
      // Check whether third-party cookie access has been granted
      // to another same-site embed
      try {
        const permission = await navigator.permissions.query({
          name: "storage-access",
        });

        if (permission.state === "granted") {
          // If so, you can just call requestStorageAccess() without a user interaction,
          // and it will resolve automatically.
          const handle = await document.requestStorageAccess({
            cookies: true,
            localStorage: true,
          });
          doThingsWithLocalStorage(handle);
          doThingsWithCookies();
        } else if (permission.state === "prompt") {
          // Need to call requestStorageAccess() after a user interaction
          btn.addEventListener("click", async () => {
            try {
              const handle = await document.requestStorageAccess({
                cookies: true,
                localStorage: true,
              });
              doThingsWithLocalStorage(handle);
              doThingsWithCookies();
            } catch (err) {
              // If there is an error obtaining storage access.
              console.error(`Error obtaining storage access: ${err}.
                            Please sign in.`);
            }
          });
        } else if (permission.state === "denied") {
          // User has denied third-party cookie access, so we'll
          // need to do something else
        }
      } catch (error) {
        console.log(`Could not access permission state. Error: ${error}`);
        doThingsWithCookies(); // Again, we'll have to hope we have access!
      }
    }
  }
}
```
 


Links:
https://habr.com/ru/articles/773260/
https://developer.mozilla.org/en-US/docs/Web/Privacy/Guides/Privacy_sandbox/Partitioned_cookies
https://developer.mozilla.org/en-US/docs/Web/Privacy/Guides/State_Partitioning
https://developer.mozilla.org/en-US/docs/Web/API/Storage_Access_API