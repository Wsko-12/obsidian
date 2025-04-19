Для того, чтобы открыть кеш, следует вызвать
https://developer.mozilla.org/ru/docs/Web/API/CacheStorage/open

```js
const cache = await caches.open("cache-name");
```


У объекта `cache` помимо других есть два метода:
- add()
- put()

add - это как fetch() + put(), поэтому принимает `url`
p**u**t - когда у нас **уже** есть Response