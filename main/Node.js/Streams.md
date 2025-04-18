
- Зачем нужны Streams в Node.js?
- Почему они лучше обычных `fs.readFile` / `fs.writeFile`?
- Типы стримов?
- С какими типами данных работают стримы?
- Как реализованы стримы изнутри?
- Каким параметром настраивается размер внутреннего буфера?
- Какая проблема может возникнуть при использовании двух стримов?
- Как решается эта проблема "нативно"?
- Writeble Streams:
	- Что возвращает метод для записи?
	- Что значит, если метод вернул `false` и что с этим делать?
	- Что за метод `_writev`?
	- Что делает метод `.cork` и зачем он нужен?
	- Что будет, если несколько раз подряд вызвать метод `.cork`?
	- Как лучше вызывать метод `.uncork`?
	- Что будет, если поток был "corked" и вызывается метод `.end()`?
	- Что принимает в себя метод `end()`?
	- Как узнать, сколько раз был поток "corked"?
- Readable Streams:
	- как (в каких режимах) работают Readable Streams?
	- как вернуть поток в paused режим?
	- как вернуть поток в flowing режим?
	- Что будет с внутренним буффером стрима, если мы не будет читать данные?
	- Что будет, если мы переключили поток в flowing режим, но нету слушателей для этого режима?
	- Почему не стоит их смешивать?
	- Какой аргумент принимает `.read()` и как он работает без него? Что с этим аргументом если в режиме `objectMode`?
	- что делают методы `push` и `unshift`? Что возвращают? Что будет если передать туда `null`?
	- что делает `pipe`? Можно ли запайпить несколько writeble на один readable?
- Duplex Streams:
	- Как работает буфферизация у Duplex стримов?
- Transform Streams:
	- Чем они отличаются от Duplex?
	- Что делает метод `flush`?
- Что предоставляет модуль `node:stream/promises`?



Потоки в Node.js позволяют нам обрабатывать большие данные порционно, чанками. Тем самым не загружая полностью данные в оперативную память.

#### Сравнение с `fs.readFile`
На примере обычного `fs.readFile` файл будет загружен в оперативную память полностью, что плохо при работе с большим объемом данных. Так же не приходится ждать полной загрузки файла, чтобы начать работу с ним. 

Так же, например, если мы стримим пользователю видео, и он хочет начать смотреть его с какой-то минуты, в метод `.createReadStream()` `options`, где указывается параметр `start` в байтах, откуда начать чтение файла. Если указать `start` больше размера файла ошибки не будет, но сразу вызовется событие `'end'` у потока

#### Виды потоков
 Выделяются 4 вида потоков:
- Writable
- Readable
- Duplex
- Transform

### Типы данных

Все потоки, созданные API Node.js, работают исключительно со строками, объектами `Buffer` , `TypedArray` и `DataView` :

- `Strings` и `Buffers` являются наиболее распространенными типами, используемыми с потоками.
- `TypedArray` и `DataView` позволяет обрабатывать двоичные данные с типами вроде `Int32Array` или `Uint8Array`.

Однако реализациям потоков можно работать с другими типами значений JavaScript (за исключением `null`, который служит особой цели в потоках). 
Такие потоки считаются работающими в «**объектном режиме**».

#### Буферизация

Оба потока `Writable` и `Readable` будут хранить данные во внутреннем буфере.

Объем потенциально буферизованных данных зависит от `highWaterMark` параметра, переданного в конструктор потока. 
- Для обычных потоков `highWaterMark` параметр указывает общее количество байтов . 
- Для потоков, работающих в **объектном режиме**, `highWaterMark` указывает общее количество объектов.
- Для потоков, работающих со строками, `highWaterMark` указывает общее количество кодовых единиц UTF-16.


Данные буферизуются в `Readable` потоках, когда реализация вызывает `stream.push(chunk)`. Если потребитель потока не вызывает `stream.read()`, данные будут находиться во внутренней очереди до тех пор, пока не будут потреблены.

Как только общий размер внутреннего буфера чтения достигнет порогового значения, указанного параметром `highWaterMark`, поток временно прекратит чтение данных из базового ресурса до тех пор, пока текущие буферизованные данные не будут потреблены (то есть поток прекратит вызывать внутренний `readable._read()` метод, который используется для заполнения буфера чтения).

Данные буферизуются в `Writable` потоках, когда `writable.write(chunk)` метод вызывается повторно. Пока общий размер внутреннего буфера записи ниже порогового значения, установленного `highWaterMark`, вызовы `writable.write()` вернут `true`. Как только размер внутреннего буфера достигнет или превысит `highWaterMark`,  будет возвращен `false`.

Основная цель API `stream`, в частности `stream.pipe()` метода, заключается в ограничении буферизации данных до приемлемых уровней, чтобы источники и пункты назначения с разными скоростями не перегружали доступную память. (**backpressure** problem)

Эта `highWaterMark` опция является порогом, а не пределом: она определяет объем данных, которые поток буферизует, прежде чем перестанет запрашивать дополнительные данные. Она не налагает строгих ограничений на память в целом. Конкретные реализации потоков могут выбирать для наложения более строгих ограничений, но это необязательно.

Поскольку `Duplex` и `Transform` потоки являются и `Readable` и `Writable`, каждый из них поддерживает два отдельных внутренних буфера, используемых для чтения и записи, что позволяет каждой стороне работать независимо от другой, поддерживая при этом соответствующий и эффективный поток данных. Например, `net.Socket` экземпляры — это `Duplex` потоки, `Readable` сторона которых позволяет потреблять данные, полученные из сокета, и `Writable` сторона которых позволяет записывать данные в сокет. Поскольку данные могут записываться в сокет с более высокой или низкой скоростью, чем они принимаются, каждая сторона должна работать (и буферизировать) независимо от другой.

Механизм внутренней буферизации является внутренней деталью реализации и может быть изменен в любое время. Однако для некоторых расширенных реализаций внутренние буферы могут быть извлечены с помощью `writable.writableBuffer`или `readable.readableBuffer`. Использование этих недокументированных свойств не рекомендуется.
##### *Backpressure*
Предположим, мы читаем данные из быстрого источника и записываем их в медленный.
Из-за этого, может возникнуть проблема backpressure, то есть поток, который вычитывает данные, уже готов их отдать в следующий поток, но этот следующий поток уже переполнил свой буффер и не может пока принимать данные.

Поэтому можно использовать метод `.pause()` на читающем потоке, а когда записывающий поток опустошит свой буффер (кинет событие `drain`), возобновить его:

```js
    source.on('data', (chunk) => {
        // Если Writable Stream переполнен, приостанавливаем Readable Stream
        if (!destination.write(chunk)) {
            source.pause();
        }
    });

    // Когда Writable Stream освобождается, возобновляем Readable Stream
    destination.on('drain', () => {
        source.resume();
    });
```





### Writable Streams
Примеры :
- HTTP requests, on the client
- HTTP responses, on the server
- fs write streams
- zlib streams
- crypto streams
- TCP sockets
- child process stdin
- process.stdout, process.stderr
Некоторые из этих примеров на самом деле являются `Duplex` потоками, реализующими `Writable` интерфейс.

Реализует интерфейсы
#### `stream.Writable`:

##### **`writable.write(chunk, encoding?, callback?)`**  
- `chunk`: `string | Buffer | TypedArray | DataView | any`  
	Необязательные данные для записи. Для потоков, не работающих в объектном режиме, `chunk` должны быть `string | Buffer | TypedArray | DataView` . 
	Для потоков объектного режима `chunk` может быть любым значением JavaScript, кроме `null`.
- `encoding`: `string | null` Кодировка, если `chunk` это строка. По умолчанию: `'utf8'`
-  `callback`: `Function` коллбек, который вызвается, когда чанк записан.
- return: `boolean` - Если чанк попал во внутренний буффер, возвращает `true`. Возвращает `false`, если внутренний буффер переполнен. В этом случае нужно дождаться ивента `drain`, чтобы продолжить запись

Метод `writable.write()` записывает некоторые данные в поток и вызывает предоставленный `callback` после полной обработки данных. Если возникает ошибка, `callback` будет вызван с ошибкой в ​​качестве первого аргумента. `callback` Вызывается асинхронно и перед тем, как `error` будет выдан.

Возвращаемое значение — `true` если внутренний буфер меньше настроенного `highWaterMark` при создании потока после допуска `chunk`. Если `false` возвращается, дальнейшие попытки записи данных в поток должны прекратиться до тех пор, пока  не будет отправлено событие `'drain'`.

##### **`writable.end(chunk, encoding?, callback?)`**  
Вызов `writable.end()` метода сигнализирует о том, что больше данные не будут записаны в `Writable`. Необязательные аргументы `chunk` и `encoding` позволяют записать еще один последний дополнительный фрагмент данных непосредственно перед закрытием потока.

Вызов `stream.write()` метода после вызова `stream.end()` приведет к ошибке.

##### **`writable.cork()`**  
Метод `writable.cork()` заставляет все записанные данные буферизироваться в памяти. Буферизованные данные будут сброшены при вызове методов `stream.uncork()` или `stream.end()`

Основное назначение `writable.cork()` — приспособиться к ситуации, когда несколько небольших фрагментов записываются в поток в быстрой последовательности. Вместо того, чтобы немедленно пересылать их в целевой пункт назначения, `writable.cork()` буферизует все фрагменты до тех пор, пока `writable.uncork()` не будет вызван , который передаст их все в `writable._writev()`, если он присутствует. 
Это предотвращает ситуацию блокировки начала очереди, когда данные буферизуются в ожидании обработки первого небольшого фрагмента. Однако использование `writable.cork()` без реализации `writable._writev()` может оказать неблагоприятное влияние на пропускную способность.

Этот метод полезен, когда нужно записывать несколько чанков данных сразу, минимизируя системные вызовы.

```js
writable.cork(); // Начинаем буферизацию 
writable.write('Первая часть '); 
writable.write('второй части.'); 
writable.uncork(); // Все накопленные данные записываются разом
```

Так же, если не вызывать метод `.uncork()`, а вызвать `.end()`, то данные из внутреннего буфера  будут записаны в destination 

##### **`writable.uncork()`**  
Метод `writable.uncork()` записывает все данные, буферизованные с момента `stream.cork()` вызова.

Если `writable.cork()` метод вызывается в потоке несколько раз, `writable.uncork()` для очистки буферизованных данных необходимо выполнить такое же количество вызовов.

```js
stream.cork();
stream.write('some ');
stream.cork();
stream.write('data ');
process.nextTick(() => {
  stream.uncork();
  // The data will not be flushed until uncork() is called a second time.
  stream.uncork();
}); 
```
Рекомендуется вызывать `uncork()` внутри `process.nextTick()`, чтобы гарантировать сброс буферизованных данных после завершения текущего блока кода. Это помогает избежать неожиданных задержек в обработке потока.

```js
writable.cork();
writable.write('Часть 1 ');
writable.write('Часть 2');
writable.uncork(); // Буфер сбрасывается прямо здесь

writable.write('Часть 3'); // Это отдельный вызов write()
```

```
Записано: Часть 1 Часть 2
Записано: Часть 3
```

Хотя Часть 1 и Часть 2 объединились, Часть 3 записалась отдельно.

```js
writable.cork();
writable.write('Часть 1 ');
writable.write('Часть 2');

process.nextTick(() => {
  writable.uncork(); // Теперь uncork вызывается в следующем цикле событий
});

writable.write('Часть 3');
```

```
Записано: Часть 1 Часть 2 Часть 3
```

Теперь весь буфер накопился и был записан разом.

`writable.writableCorked`
Возвращает число, сколько необходимо раз `writable.uncork()` вызвать, чтобы полностью открыть поток.

##### **`writable.destroy(err?)`**  
Уничтожить поток. При желании сгенерировать `'error'` событие и сгенерировать `'close'` событие (если не `emitClose` установлено значение false).

После этого вызова записываемый поток завершается, и последующие вызовы `write()` или `end()` приведут к `ERR_STREAM_DESTROYED` ошибке. Это разрушительный и немедленный способ уничтожить поток. Предыдущие вызовы `write()` могли не быть слиты и могут вызвать `ERR_STREAM_DESTROYED` ошибку. Используйте `end()` вместо `destroy()`, если данные должны быть записаны перед закрытием, или дождитесь `'drain'` события перед уничтожением потока.

и тд. (https://nodejs.org/api/stream.html#writable-streams)

#### События:

##### `'close'`
Событие `'close'` выдается, когда поток и любой из его базовых ресурсов (например, файловый дескриптор) были закрыты. Событие указывает, что больше не будет выдаваться никаких событий, и никаких дальнейших вычислений не будет.

Поток `Writable` всегда будет генерировать `'close'` событие, если он создан с `emitClose` опцией.

##### `'drain'`
Если вызов `stream.write(chunk)` возвращает `false`, `'drain'` событие будет сгенерировано, когда будет уместно возобновить запись данных в поток.

##### `'error'`
Событие `'error'` выдается, если произошла ошибка при записи или передаче данных. Коллбек слушателя получит в Error аргумент

Поток закрывается при возникновении события `'error'`, если только при создании потока параметр `autoDestroy` не был установлен в значение `false`.

После `'error'` не должно быть никаких других событий, кроме `'close'` (включая события `'error'`).

##### `'finish'`
Событие `'finish'` генерируется после вызова метода `stream.end()` и записи всех данных в целевую систему.

##### `'pipe'`
Событие `'pipe'` генерируется, когда метод `stream.pipe()` вызывается для читаемого потока, добавляя этот записываемый поток в свой набор назначений.

```js
const writer = getWritableStreamSomehow();
const reader = getReadableStreamSomehow();
writer.on('pipe', (src) => {
  console.log('Something is piping into the writer.');
  assert.equal(src, reader);
});
reader.pipe(writer);
```

##### `'unpipe'`
Событие `'unpipe'` генерируется, когда метод `stream.unpipe()` вызывается для потока `Readable`, удаляя этот `Writable` из набора назначений.

```js
const writer = getWritableStreamSomehow();
const reader = getReadableStreamSomehow();
writer.on('unpipe', (src) => {
  console.log('Something has stopped piping into the writer.');
  assert.equal(src, reader);
});
reader.pipe(writer);
reader.unpipe(writer); 
```


Чтобы создать кастомный Wirtable, нужно переопределить минимум `_write`.
Если не вызвать `callback()` (Error first), стрим зависнет.
```js
const { Writable } = require('stream');

class ConsoleWritable extends Writable {
  _write(chunk, encoding, callback) {
    console.log('Получено:', chunk.toString());
    callback();
  }
}

const writable = new ConsoleWritable();

writable.write('Hello\n');
writable.write('world \n');
writable.end();
```

Например, записываем во что-то асинхронное, тогда `callback` следует после того, как все завершится:
```js
const { Writable } = require('stream');
const delay = (ms) => new Promise(res => setTimeout(res, ms))
class ConsoleWritable extends Writable {
    _write(chunk, encoding, callback) {
        console.log(`${Math.floor(performance.now())} Получено:`, chunk.toString());
        delay(2000).then(callback)
    }
}

const writable = new ConsoleWritable();

for(let i = 0; i < 5; i++){
    console.log(`TRY ${i}: ${writable.write(`Test ${i}`)}`);
}

writable.end();

//LOGS:
// 30 Получено: Test 0
// TRY 0: true
// TRY 1: true
// TRY 2: true
// TRY 3: true
// TRY 4: true
// 2044 Получено: Test 1
// 4051 Получено: Test 2
// 6052 Получено: Test 3
// 8054 Получено: Test 4
```

Тоже самое, но с `cork()`
```js
const { Writable } = require('stream');
const delay = (ms) => new Promise(res => setTimeout(res, ms))
class ConsoleWritable extends Writable {
    _write(chunk, encoding, callback) {
        console.log(`${Math.floor(performance.now())} Получено:`, chunk.toString());
        delay(2000).then(callback)
    }
}

const writable = new ConsoleWritable();

writable.cork()
for(let i = 0; i < 5; i++){
    console.log(`TRY ${i}: ${writable.write(`Test ${i}`)}`);
}
writable.uncork()


// LOGS:  
// TRY 0: true  
// TRY 1: true  
// TRY 2: true  
// TRY 3: true  
// TRY 4: true  
// 33 Получено: Test 0  
// 2037 Получено: Test 1  
// 4041 Получено: Test 2  
// 6047 Получено: Test 3  
// 8056 Получено: Test 4
```


`_writev(chunks, callback)`
Метод `._writev()` может быть реализован в дополнение или альтернативно `writable._write()` в реализациях потока, которые способны обрабатывать несколько фрагментов данных одновременно. Если он реализован и есть буферизованные данные от предыдущих записей, вместо `_write()` будет вызван `_writev()`.

```js
const { Writable } = require('stream');
const delay = (ms) => new Promise(res => setTimeout(res, ms))
class ConsoleWritable extends Writable {
    _write(chunk, encoding, callback) {
        console.log(`${Math.floor(performance.now())} Получено:`, chunk.toString());
        delay(2000).then(callback)
    }

    _writev(chunks, callback) {
        const combined = chunks.map(({ chunk }) => chunk.toString()).join(' | ');
        console.log(`${Math.floor(performance.now())} Получены чанки: ${combined}`);
        delay(2000).then(callback)
    }
}

const writable = new ConsoleWritable();

for(let i = 0; i < 5; i++){
    console.log(`TRY ${i}: ${writable.write(`Test ${i}`)}`);
}

writable.end();

// LOGS:  
// 30 Получено: Test 0  
// TRY 0: true  
// TRY 1: true  
// TRY 2: true  
// TRY 3: true  
// TRY 4: true  
// 2046 Получены чанки: Test 1 | Test 2 | Test 3 | Test 4
```

С `cork()`:
```js
const { Writable } = require('stream');
const delay = (ms) => new Promise(res => setTimeout(res, ms))
class ConsoleWritable extends Writable {
    _write(chunk, encoding, callback) {
        console.log(`${Math.floor(performance.now())} Получено:`, chunk.toString());
        delay(2000).then(callback)
    }

    _writev(chunks, callback) {
        const combined = chunks.map(({ chunk }) => chunk.toString()).join(' | ');
        console.log(`${Math.floor(performance.now())} Получены чанки: ${combined}`);
        delay(2000).then(callback)
    }
}

const writable = new ConsoleWritable();

writable.cork()
for(let i = 0; i < 5; i++){
    console.log(`TRY ${i}: ${writable.write(`Test ${i}`)}`);
}
writable.uncork()

// LOGS:  
// TRY 0: true  
// TRY 1: true  
// TRY 2: true  
// TRY 3: true  
// TRY 4: true  
// 33 Получены чанки: Test 0 | Test 1 | Test 2 | Test 3 | Test 4
```

### Readable streams

Примеры:

- HTTP responses, on the client
- HTTP requests, on the server
- fs read streams
- zlib streams
- crypto streams
- TCP sockets
- child process stdout and stderr
- process.stdin

#### Режимы чтения
Readable стримы могут работать в двух режимах **flowing** и **paused**
- **flowing**: данный читаются из источника автоматически и передаются в приложение используя интерфейс EventEmitter.
- **paused**: Для чтения фрагментов данных из потока необходимо явно вызывать метод stream.read().

Чтобы перейти в flowing-режим:
- Добавить слушателя `data`
- Вызвать `resume()`
- Подключить поток к Writable (pipe())

Чтобы вернуться в **paused**-режим:
- Вызвать `pause()`
- добавить обработчик `'readable'`
- У всех стримом в `pipe` вызвать метод `unpipe`

Важно помнить, что `Readable` **не будет генерировать данные**, пока не будет предоставлен механизм для потребления или игнорирования этих данных. Если механизм потребления отключен или удален, Readable попытается прекратить генерировать данные.

Если Readable переключается в режим потока и нет потребителей, доступных для обработки данных, эти данные будут потеряны. Это может произойти, например, когда метод `readable.resume()` вызывается без прослушивателя, прикрепленного к событию 'data', или когда обработчик событий 'data' удаляется из потока.

Добавление обработчика событий `'readable'` автоматически останавливает поток, и данные должны быть потреблены через `readable.read()`. Если обработчик событий `'readable'` удален, поток снова начнет течь, если есть обработчик событий `'data'`.

В частности, в любой момент времени каждый Readable находится в одном из трех возможных состояний:

- `readable.readableFlowing === null`
- `readable.readableFlowing === false`
- `readable.readableFlowing === true`

Когда `readable.readableFlowing` равен `null`, механизм потребления данных потока не предоставляется. Поэтому поток не будет генерировать данные. В этом состоянии подписка прослушивателя для события `'data'`, вызов метода `readable.pipe()` или вызов метода `readable.resume()` переключит `readable.readableFlowing` в значение `true`, заставив `Readable` начать активно испускать события по мере генерации данных.

Вызов `readable.pause()`, `readable.unpipe()` или в случае backpressure приведет к тому, что `readable.readableFlowing` будет установлен как `false`, временно останавливая поток событий, но не останавливая генерацию данных. В этом состоянии подписка слушателя для события `'data'` не переключит `readable.readableFlowing` в true.

```js
const { PassThrough, Writable } = require('node:stream');
const pass = new PassThrough();
const writable = new Writable();

pass.pipe(writable);
pass.unpipe(writable);
// readableFlowing is now false.

pass.on('data', (chunk) => { console.log(chunk.toString()); });
// readableFlowing is still false.
pass.write('ok');  // Will not emit 'data'.
pass.resume();     // Must be called to make stream emit 'data'.
// readableFlowing is now true. 
```

Пока `readable.readableFlowing` имеет значение `false`, данные могут накапливаться во внутреннем буфере потока.

API потока Readable развивалось в нескольких версиях Node.js и предоставляет несколько методов получения потоковых данных. 
В целом, разработчикам следует выбирать один из методов получения данных и никогда не следует использовать несколько методов для получения данных из одного потока. В частности, использование комбинации  `on('data')`, `on('readable')`, `pipe()` или `async iterators` может привести к неочевидному поведению.

Реализует интерфейсы
#### `stream.Readable`:
##### **`readable.read(size?)`**  
- `size` : `number` Необязательный аргумент для указания объема данных для считывания.
- return `string | Buffer | null | any`

Метод `readable.read()` считывает данные из внутреннего буфера и возвращает их.

Если данные для чтения недоступны, возвращается `null`. 

По умолчанию данные возвращаются как объект `Buffer`, если только кодировка не была указана с помощью метода `readable.setEncoding()` или поток не работает в объектном режиме.

Необязательный аргумент `size` указывает определенное количество байтов для чтения. Если `size` байтов недоступно для чтения, будет возвращено значение `null`, если только поток не закончился, в этом случае будут возвращены все данные, оставшиеся во внутреннем буфере.

Если аргумент `size` не указан, будут возвращены все данные, содержащиеся во внутреннем буфере.

Аргумент size должен быть меньше или равен 1 GiB.

Метод `readable.read()` следует вызывать только для потоков Readable, работающих в `paused` режиме. 

В режиме `flowing` readable.read() вызывается автоматически до тех пор, пока внутренний буфер не будет полностью опустошен.


```js
const readable = getReadableStreamSomehow();

// 'readable' may be triggered multiple times as data is buffered in
readable.on('readable', () => {
  let chunk;
  console.log('Stream is readable (new data received in buffer)');
  // Use a loop to make sure we read all currently available data
  while (null !== (chunk = readable.read())) {
    console.log(`Read ${chunk.length} bytes of data...`);
  }
});

// 'end' will be triggered once when there is no more data available
readable.on('end', () => {
  console.log('Reached end of stream.');
});
```

Каждый вызов `readable.read()` возвращает фрагмент данных или `null`, что означает, что в данный момент больше нет данных для чтения. 

Эти фрагменты не объединяются автоматически. Поскольку один вызов `read()` не возвращает все данные (??????? выше же написано, что вовращает, если не передать `size`), может потребоваться использование цикла `while` для непрерывного чтения фрагментов до тех пор, пока не будут извлечены все данные. 

При чтении большого файла `.read()` может временно возвращать `null`, что означает, что он израсходовал все буферизованное содержимое, но могут быть еще данные, которые еще предстоит буферизовать. 

В таких случаях новое событие `'readable'` генерируется, как только в буфере появляется больше данных, а событие `'end'` означает завершение передачи данных.

Поток Readable в объектном режиме всегда будет возвращать один элемент из вызова `readable.read(size)`, независимо от значения аргумента `size`.

##### `readable.readable`

Имеет значение true, если безопасно вызывать `readable.read()`, что означает, что поток не был уничтожен или не выдал `'error'` или `'end'`.


##### `readable.pause()`
Метод `readable.pause()` заставит поток в режиме потока прекратить испускать события `'data'`, выключая режим потока. Любые данные, которые станут доступны, останутся во внутреннем буфере.

Метод `readable.pause()` не действует, если имеется прослушиватель событий `'readable'`.

##### `readable.resume()`

Метод `readable.resume()` заставляет явно приостановленный поток Readable возобновлять отправку событий `'data'`, переключая поток в режим потока.

Метод `readable.resume()` можно использовать для полного использования данных из потока без фактической обработки этих данных.

```
getReadableStreamSomehow()
  .resume()
  .on('end', () => {
    console.log('Reached the end, but did not read anything.');
  }); 
```

Метод `readable.resume()` не действует, если имеется хендлер событий `'readable'`.

##### `readable.pipe(destination[, options])`
прикрепляет `Writable` поток к `readable` , заставляя его автоматически переключаться в `flowing` режим и передавать все свои данные в `Writable`. Поток данных будет автоматически управляться, чтобы целевой `Writable`поток не был перегружен более быстрым `Readable`потоком.

`options`:
- `end` - По умолчанию `stream.end()` вызывается в целевом `Writable` потоке, когда исходный `Readable` поток испускает `'end'`, так что целевой поток больше не может быть записан. Чтобы отключить это поведение по умолчанию, `end` параметр можно передать как `false`, в результате чего целевой поток останется открытым.
```js
reader.pipe(writer, { end: false });
reader.on('end', () => {
  writer.end('Goodbye\n');
});
```
Одно важное предостережение заключается в том, что если `Readable` поток выдает ошибку во время обработки, `Writable`пункт назначения **не закрывается** автоматически. Если возникает ошибка, необходимо будет _вручную_ закрыть каждый поток, чтобы предотвратить утечки памяти.

Можно ли зайпайпить несколько writable на один readable?
GPT и прочие говорят что нет, но я попробовал так:

```js
const fs = require('node:fs');  
  
// const ws = fs.createWriteStream('./text.txt', {highWaterMark: 1000})  
// for(let i = 0; i < 100000; i++){  
// ws.write(`test ${i} \n`)  
// }  
  
  
const rs = fs.createReadStream('./text.txt', {encoding: "utf8", highWaterMark: 32});  
  
const ws1 = fs.createWriteStream('./ws1.txt', {highWaterMark: 1});  
const ws2 = fs.createWriteStream('./ws2.txt', {highWaterMark: 11});  
const ws3 = fs.createWriteStream('./ws3.txt', {highWaterMark: 2});  
  
rs.pipe(ws1);  
rs.pipe(ws2);  
rs.pipe(ws3);  
rs.on('data', console.log)
```

И вроде все все получали нормально. Так же чел пишет что все работает ок:
https://github.com/nodejs/help/issues/2707

Получается, что будет проблема только тогда, когда несколько потоков "подпишутся" позже первого. Первый вычитает часть данных, а следующие получат уже без этой части.


##### `readable.unpipe(destination?)`
- `destination`: `stream.Writable` - Необязательный определенный поток для отсоединения

Метод `readable.unpipe()` отсоединяет записываемый поток, ранее присоединенный с помощью метода `stream.pipe()`.

Если `destination` не указан, то все каналы отсоединяются. 
Если `destination` указан, но для него не настроен канал, то метод ничего не делает.

```js
const fs = require('node:fs');
const readable = getReadableStreamSomehow();
const writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt',
// but only for the first second.
readable.pipe(writable);
setTimeout(() => {
  console.log('Stop writing to file.txt.');
  readable.unpipe(writable);
  console.log('Manually close the file stream.');
  writable.end();
}, 1000); 
```


#####  `readable.push(chunk, encoding?)`
- `chunk`: `Buffer | TypedArray | DataView | string | null | any`. Часть данных для переноса в очередь чтения. Для потоков, не работающих в объектном режиме, `chunk` должен быть `Buffer | TypedArray | DataView | string | null`. Для потоков объектного режима `chunk` может быть любым значением JavaScript.
- `encoding`: `string` Кодировка фрагментов строк. Должна быть допустимая кодировка буфера, например 'utf8' или 'ascii'.

Когда chunk — это `Buffer | TypedArray | DataView | string`, фрагмент данных будет добавлен во внутреннюю очередь для consumers потоков. 
Передача `chunk` как `null` сигнализирует о конце потока (EOF), после чего больше нельзя записывать данные.

Когда Readable работает в режиме паузы, данные, добавленные с помощью `readable.push()`, можно считать, вызвав метод `readable.read()` при возникновении события `'readable'`.

Когда Readable работает в потоковом режиме, данные, добавленные с помощью `'readable.push()'`, будут доставлены путем отправки события `'data'`.

Метод `readable.push()` используется для помещения содержимого во внутренний буфер. Он может управляться методом `readable._read()`.

Для потоков, не работающих в объектном режиме, если параметр `chunk` `readable.push()` не определен, он будет рассматриваться как пустая строка или буфер

##### `readable._read(size)`
Эта функция НЕ ДОЛЖНА вызываться кодом приложения напрямую. Она должна быть реализована дочерними классами и вызываться только внутренними методами класса Readable.

Все реализации потока Readable должны обеспечивать реализацию метода `readable._read()` для извлечения данных из базового ресурса.

При вызове `readable._read()`, если данные доступны из ресурса, реализация должна начать помещать эти данные в очередь чтения с помощью метода `this.push(dataChunk)`. `_read()` будет вызываться снова после каждого вызова `this.push(dataChunk)`, как только поток будет готов принять больше данных. `_read()` может продолжать чтение из ресурса и помещать данные в очередь до тех пор, пока `readable.push()` не вернет `false`. Только когда `_read()` вызывается снова после остановки, он должен возобновить помещение дополнительных данных в очередь.



##### `readable.unshift(chunk, encoding?)`
- `chunk`: `Buffer | TypedArray | DataView | string | null | any`. Часть данных для переноса в очередь чтения. Для потоков, не работающих в объектном режиме, `chunk` должен быть `Buffer | TypedArray | DataView | string | null`. Для потоков объектного режима `chunk` может быть любым значением JavaScript.
- `encoding`: `string` Кодировка фрагментов строк. Должна быть допустимая кодировка буфера, например 'utf8' или 'ascii'.

Передача `chunk` как `null` сигнализирует о конце потока (EOF) и ведет себя так же, как `readable.push(null)`, после чего больше данные не могут быть записаны. 
Сигнал EOF помещается в конец буфера, и любые буферизованные данные все равно будут сброшены.

Метод `readable.unshift()` возвращает часть данных обратно во внутренний буфер. Это полезно в определенных ситуациях, когда поток потребляется кодом, которому нужно «отменить потребление» некоторого количества данных, которые он оптимистично вытащил из источника, чтобы данные можно было передать какой-то другой стороне.

Метод `stream.unshift(chunk)` не может быть вызван после срабатывания события `end`, иначе возникнет ошибка времени выполнения.

```js
class HeaderParser extends Readable {
  constructor(options) {
    super(options);
    this.header = null;
  }

  _read(size) {
    if (!this.header) {
      // Читаем первые 10 байт для заголовка
      const chunk = this.read(10); 
      this.header = chunk.toString();
      
      // Возвращаем оставшиеся данные в буфер
      const remainingData = chunk.slice(10);
      if (remainingData.length > 0) {
        this.unshift(remainingData);
      }
    } else {
      // Обрабатываем оставшиеся данные
      const data = this.read(size);
      if (data) {
        this.push(data);
      }
    }
  }
}
```


и тд (https://nodejs.org/api/stream.html#readable-streams)

#### События:

##### `'data'`
Событие `'data'` генерируется всякий раз, когда поток передает владение фрагментом данных потребителю. Это может произойти всякий раз, когда поток переключается в режим потока путем вызова `readable.pipe()`, `readable.resume()` или путем подписки слушателя к событию `'data'`. Событие `'data'` также будет генерироваться всякий раз, когда вызывается метод `readable.read()` и фрагмент данных доступен для возврата.

Подписка прослушивателя событий `'data'` к потоку, который не был явно приостановлен, переключит поток в режим потока. Затем данные будут переданы, как только они станут доступны.

##### `'end'`
Событие `'end'` генерируется, когда из потока больше нет данных для потребления.

Событие `'end'` не будет отправлено, пока данные не будут полностью потреблены. Этого можно добиться, переключив поток в режим потока или вызвав stream.read() несколько раз, пока все данные не будут потреблены.

```js
const readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data.`);
});
readable.on('end', () => {
  console.log('There will be no more data.');
});
```


##### `'pause'`
Событие `'pause'` генерируется, когда вызывается `stream.pause()` и `readableFlowing` не равен false.


##### `'readable'`
Событие `'readable'` генерируется, когда есть данные, доступные для чтения из потока, вплоть до `state.highWaterMark`. 
Фактически, оно указывает на то, что в буфере потока есть новая информация. Если в этом буфере есть данные, можно вызвать `stream.read()` для их извлечения. Кроме того, событие `'readable'` может также генерироваться при достижении конца потока.

Если достигнут конец потока, вызов `stream.read()` вернет `null` и вызовет событие `'end'`

В некоторых случаях подписка к событию `'readable'` приведет к считыванию некоторого объема данных во внутренний буфер.

В целом, механизмы событий readable.pipe() и 'data' проще понять, чем событие 'readable'. Однако обработка 'readable' может привести к увеличению пропускной способности.

Если одновременно используются и `'readable'`, и `'data'`, `'readable'` имеет приоритет в управлении потоком, т. е. `'data'` будет отправлено только при вызове `stream.read()`. Свойство `readableFlowing` станет `false`. 
Если есть прослушиватели `'data'`, когда `'readable'` удален, поток начнет течь, т. е. события `'data'` будут отправлены без вызова `.resume()`.

```js
const readable = getReadableStreamSomehow();
readable.on('readable', function() {
  // There is some data to read now.
  let data;

  while ((data = this.read()) !== null) {
    console.log(data);
  }
});
```

##### `'resume'`
Событие `'resume'` генерируется, когда вызывается `stream.resume()`, а `readableFlowing` не равен true.


#### paused mode 
```
// test.txt
AAAABBBBCCCCDDDDEEEEFFFF
```


```js
const fs = require("node:fs")
const stream = fs.createReadStream("./test.txt", {encoding: "utf-8", highWaterMark: 8});
stream.on('readable', function(){
    console.log('READABLE')
    let chunk;
    do{
        chunk = this.read(8);
        console.log(chunk);
    } while(chunk != null)
});
```

и так, мы установили read stream, чтобы он вычитывал по 8 байтов, и подписались на readable

Это значит, что каждый раз, когда будут готовы данные для чтения во внутреннем буффере этого стрима, он будет генерировать `readable` ивент, говоря о том, что можно вычитывать

Мы будем читать с той же скоростью

Логи выглядят так:
```
READABLE
AAAABBBB
null
READABLE
CCCCDDDD
null
READABLE
EEEEFFFF
null
READABLE
null
```

Ок, читаем чуть медленнее:
`chunk = this.read(4);`

```
READABLE
AAAA    
BBBB    
null    
READABLE
CCCC    
DDDD    
null    
READABLE
EEEE    
FFFF
null
READABLE
null
```

Чуть быстрее:
`chunk = this.read(12);`
```
READABLE
null
READABLE
AAAABBBBCCCC
DDDDEEEEFFFF
null
READABLE
null

```

Добавим GGGG в text.txt:
```
READABLE
null        
READABLE    
AAAABBBBCCCC
DDDDEEEEFFFF
null        
READABLE    
null        
READABLE    
GGGG
null

```

Почему 
```
AAAABBBBCCCC
DDDDEEEEFFFF
```
То есть 24 байта?

Chat GPT
### Почему поток загружает ещё 8 байт (`EEEEFFFF`)?

Разберёмся глубже. Ключевой момент: **highWaterMark не ограничивает общее количество данных в буфере, а лишь определяет, сколько за раз будет запрашиваться у файловой системы.**

#### 1. Что происходит шаг за шагом?

- `fs.createReadStream` создаёт поток с `highWaterMark = 8`, значит, он **запрашивает данные порциями по 8 байт**.
- Когда файл открыт, поток **загружает первые 8 байт (`AAAABBBB`) в буфер**.
- Событие `readable` срабатывает, потому что буфер не пуст.
- Ты вызываешь `this.read(12)`, но в буфере **всего 8 байт**.
    **Что делает поток в таком случае?**
    - Поток видит, что ты запрашиваешь 12 байт, но в буфере есть только 8.
    - Чтобы удовлетворить запрос, он **подгружает ещё 8 байт (`CCCCDDDD`)** из файла.
    - Теперь в буфере 16 байт, и `read(12)` возвращает `AAAABBBBCCCC`.
    - Остаются 4 байта (`DDDD`).
- Следующий `read(12)` видит, что в буфере **всего 4 байта**, и **запрашивает ещё 8 байт (`EEEEFFFF`)**.
- Теперь в буфере **12 байт** (`DDDDEEEEFFFF`), и `read(12)` возвращает их.

#### 2. Ключевая причина

Поток **не останавливается** после загрузки `highWaterMark`. Он работает так:
1. Если буфер пуст, загружаем `highWaterMark` байт.
2. Если кто-то вызвал `read(N)`, и в буфере **меньше, чем N**, **дозапрашиваем ещё данные** (но всегда кусками `highWaterMark`).

Ок, вроде ясно

Попробуем так:
```js
const fs = require("node:fs")
const stream = fs.createReadStream("./test.txt", {encoding: "utf-8", highWaterMark: 2});
stream.on('readable', function(){
    console.log('READABLE')
    console.log(stream.readableLength)
    console.log(stream.read(16))
});
```

Посмотрим, какой размер буффера на каждом шаге:

```
READABLE
2               
null            
READABLE        
18              
AAAABBBBCCCCDDDD
READABLE        
12              
null            
READABLE        
12              
EEEEFFFFGGGG 
```

Что тоже странно. Почему на 3 readable у нас в буффере 12. Почему он не сделал этот ивент на 8 или 6?
Гпт говорит, что 
- `readable` **не срабатывает просто так каждые 2 байта**. Он **срабатывает, когда буфер переходит из пустого в заполненное состояние**.
- После второго `readable` буфер никогда не был полностью пустым (он постоянно содержал **как минимум 2 байта**).
- `readable` сработал на `12`, потому что именно к этому моменту:
    1. Буфер был пуст.
    2. Поток **загрузил** очередную порцию данных (12 байт).
    3. Это вызвало новое `readable`.

Короче, непонятно

#### Custom Readable
```js
const { Readable } = require('stream');
const delay = (ms) => new Promise(res => setTimeout(res, ms))
class CounterStream extends Readable {
    constructor(options) {
        super(options);
        this.current = 1;
    }

    _read(size) {
        console.log(`${Math.floor(performance.now())} Called _read with size ${size}`);
        const i = this.current++;
        if (i > 5) {
            this.push(null);
        } else {
            delay(1000).then(() => {
                this.push(Buffer.from(`TEST ${i}\n`));
            })

        }
    }
}

const stream = new CounterStream();
stream.pipe(process.stdout);

// LOGS:  
// 31 Called _read with size 16384  
// TEST 1  
// 1048 Called _read with size 16384  
// TEST 2  
// 2060 Called _read with size 16384  
// TEST 3  
// 3067 Called _read with size 16384  
// TEST 4  
// 4070 Called _read with size 16384  
// TEST 5  
// 5083 Called _read with size 16384

```



### Duplex Streams
Это стримы, которые реализуют в себе и `Writable` и `Readable`, при этом, два этих потока **независимы** друг от друга. Пример такого потока TCP-Socket. Да, мы читаем из его Readable, но писать в его Writable нам не обязательно, либо мы пишем туда совершенно другие данные.

`duplex.allowHalfOpen`
Если `false` то поток автоматически вызовет end для writable стороны, когда readable сторона закончится.
По умолчанию значение `true`

### Transform stream
Это duplex потоки, но в отличие от них, поток входа связан с потоком вывода. То есть получая данные на вход (readable), transform stream их преобразовывает и отдает сразу на поток выхода (writable)

например, трансформируем наш стрим:

```js
const fs = require("node:fs")
const { Transform } = require("stream");

const replaceStream = new Transform({
    decodeStrings: false, 
    encoding: 'utf8',
    transform(chunk, encoding, callback) {
        const transformed = chunk
            .replace(/A/g, "@")
            .replace(/B/g, "8")
            .replace(/C/g, "(");

        this.push(transformed);
        callback();
    }
});

const stream = fs.createReadStream("./test.txt", {encoding: "utf-8", highWaterMark: 8});

const transformed = stream.pipe(replaceStream);

transformed.on('readable', function(){
    console.log('READABLE')
    let chunk;
    do{
        chunk = this.read(8);
        console.log(chunk)
    } while(chunk != null)
});

```

Так же, у него есть метод flush, который нужен, если мы не просто "пропускаем" и сразу трансформируем данные, а например, храним какое-то внутреннее состояние. flush забирает из потока абсолютно все.
Docs: В некоторых случаях операция преобразования может потребовать выдачи дополнительного бита данных в конце потока. Например, `zlib`поток сжатия будет хранить объем внутреннего состояния, используемого для оптимального сжатия выходных данных. Однако, когда поток заканчивается, эти дополнительные данные необходимо сбросить, чтобы сжатые данные были полными.

### Streams Promises API
API `stream/promises` предоставляет альтернативный набор асинхронных служебных функций для потоков, которые возвращают `Promise` объекты, а не используют коллбеки.

#### `stream.pipeline(streams: Stream[] | Iterable[] | AsyncIterable[] | Function[], options?: )`
так же есть сигнатура `stream.pipeline(source[, ...transforms], destination[, options])`

Позволяет удобно сделать пайплайн для несколько потоков:

```js
const { pipeline } = require('node:stream/promises');
const fs = require('node:fs');
const zlib = require('node:zlib');

async function run() {
  await pipeline(
    fs.createReadStream('archive.tar'),
    zlib.createGzip(),
    fs.createWriteStream('archive.tar.gz'),
  );
  console.log('Pipeline succeeded.');
}

run().catch(console.error);
```

`options` принимает в себя:
- `signal: AbortSignal` - завершит поток когда сигнал был вызыван. (Promise будет reject'нут c ошибкой AbortSignal)
- `end: boolean` - Завершить целевой поток, когда заканчивается исходный поток. Потоки преобразования всегда завершаются, даже если это значение равно `false`. **По умолчанию:** `true`

```js
const { pipeline } = require('node:stream/promises');
const fs = require('node:fs');

async function run() {
  await pipeline(
    fs.createReadStream('lowercase.txt'),
    async function* (source, { signal }) {
      source.setEncoding('utf8');  // Work with strings rather than `Buffer`s.
      for await (const chunk of source) {
        yield await processChunk(chunk, { signal });
      }
    },
    fs.createWriteStream('uppercase.txt'),
  );
  console.log('Pipeline succeeded.');
}

run().catch(console.error);
```

#### `stream.finished(stream[, options])`
это метод, который позволяет отслеживать **завершение работы потока** (stream) независимо от того, завершился ли он успешно, с ошибкой или был принудительно уничтожен. Это более надёжная альтернатива ручной подписке на события `'end'`, `'finish'` и `'error'`.

```js
const { finished } = require('node:stream/promises');
const fs = require('node:fs');

const rs = fs.createReadStream('archive.tar');

async function run() {
  await finished(rs);
  console.log('Stream is done reading.');
}

run().catch(console.error);
rs.resume(); // Drain the stream.
```

Так же этот существует не промисифицированный метод, а функцию для получения уведомлений, когда поток больше не доступен для чтения, записи или произошла ошибка или произошло событие преждевременного закрытия.

```js
const { finished } = require('node:stream');
const fs = require('node:fs');

const rs = fs.createReadStream('archive.tar');

finished(rs, (err) => {
  if (err) {
    console.error('Stream failed.', err);
  } else {
    console.log('Stream is done reading.');
  }
});

rs.resume(); // Drain the stream.
```

`options` представляет из себя:
- `error:boolean` - Если `false`, ошибки **не** будут передаваться в колбэк. По умолчанию: `true`. (Полезно, когда ошибки обрабатываются отдельно.)

- `readable:boolean` - Если `false`, метод не будет ждать завершения Readable потока.. По умолчанию: `true` (Актуально для **Duplex**-потоков, где нужно отслеживать только запись)

- `writable:boolean` - Если `false`, метод не будет ждать завершения Writable потока. По умолчанию: `true`(Полезно для **Duplex**-потоков, где важно только чтение.)

- `signal: AbortSignal` - позволяет прервать ожидание завершения потока. Базовый поток не будет прерван, если сигнал будет прерван. callback будет вызван с AbortError. Все зарегистрированные слушатели, добавленные этой функцией, также будут удалены

 
Примеры использования:
 1. **Игнорирование ошибок**
```js
finished(stream, { error: false }, () => {
  console.log('Поток завершён (ошибки проигнорированы)');
});
```

 2. **Отслеживание только записи для Duplex-потока**

```js
const duplex = net.connect(80, 'example.com');

finished(duplex, { readable: false }, (err) => {
  console.log('Запись завершена (чтение может продолжаться)');
});
```

 3. **Отмена через AbortController**

```js
const controller = new AbortController();

finished(stream, { signal: controller.signal }, (err) => {
  if (err) console.error('Прервано:', err);
});

setTimeout(() => controller.abort(), 2000);
```


#### `stream.Readable.from(iterable[, options])`
Вспомогательный метод для создания читаемых потоков из итераторов.

```js
const { Readable } = require('stream');

const delay = (ms) => new Promise(res => setTimeout(res, ms))

async function* getDataFromAsyncSource() {
  for (let i = 0; i < 5; i++) {
    await delay(500);
    yield `chunk-${i}\n`;
  }
}

const readable = Readable.from(getDataFromAsyncSource());

readable.pipe(process.stdout);
```