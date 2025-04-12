
#### Threads
Node.js изначально является **однопоточной** платформой, что означает, что основной поток (или Event Loop) отвечает за выполнение JavaScript-кода
Начиная с 10 версии Node.js можно использовать **Worker Threads** (рабочие потоки), которые позволяют выполнять задачи в отдельных потоках.

#### Processes
Процесс — это независимая, изолированная единица выполнения программы. У каждого процесса есть собственная память, системные ресурсы и поток выполнения.


**IPC (Inter-Process Communication)** — это механизм, который позволяет процессам обмениваться данными и взаимодействовать друг с другом. В Node.js IPC используется для обмена информацией между основным процессом и дочерними процессами (например, через модуль `child_process`).

### `child_process`

Основные методы:

#### `exec` 
используется для выполнения команд в оболочке (shell) и возвращает результат их выполнения stdout/stderr, которые являются строками или буфферами.

```javascript
const { exec } = require('child_process');

exec('ls -lh', (error, stdout, stderr) => {
  if (error) {
    console.error(`Ошибка: ${error.message}`);
    return;
  }
  if (stderr) {
    console.error(`stderr: ${stderr}`);
    return;
  }
  console.log(`stdout: ${stdout}`);
});
```

- `execFile` - запускает **исполняемый файл напрямую**, без использования оболочки.

```javascript
const { execFile } = require('child_process');

execFile('/usr/bin/ls', ['-la'], (error, stdout, stderr) => { ... });

```

DeepSeek:
- `exec()` — как если бы вы открыли терминал и вручную ввели команду.
- `execFile()` — как если бы программа запустилась напрямую, минуя терминал.




#### `spawn` 
запускает процесс, но в отличие от `exec` и `execFile`, он возвращает потоки (streams) для stdin, stdout и stderr.

```javascript
const { spawn } = require('child_process');  
  
const child = spawn('node', ['-v']);  
  
child.stdout.on('data', (data) => {  
	console.log(`stdout: ${data}`);  
});  
  
child.stderr.on('data', (data) => {  
	console.error(`stderr: ${data}`);  
});  
  
child.on('close', (code) => {  
	console.log(`Процесс завершился с кодом ${code}`);  
});
```

Так же дополнительным параметром в spawn мы можем настраивать потоки stdin, stdout, stderr, через `options.stdio`:

```javascript
const subprocess = spawn('prg', [], {
  detached: true,
  stdio: 'inherit',
});
```

*Параметр options.stdio используется для настройки каналов, которые устанавливаются между родительским и дочерним процессами. По умолчанию stdin, stdout и stderr дочернего процесса перенаправляются в соответствующие потоки , и на subprocess.stdinобъекте subprocess.stdout. Это эквивалентно установке равного .subprocess.stderrChildProcessoptions.stdio['pipe', 'pipe', 'pipe']*

Для удобства `options.stdio`может быть одна из следующих строк:
- `'pipe'`: эквивалентно `['pipe', 'pipe', 'pipe']`(по умолчанию)
- `'overlapped'`: эквивалентно`['overlapped', 'overlapped', 'overlapped']`
- `'ignore'`: эквивалентно`['ignore', 'ignore', 'ignore']`
- `'inherit'`: эквивалентно `['inherit', 'inherit', 'inherit']`или`[0, 1, 2]`

Рассмотрим возможные опции:
###### `'pipe'`:
(по умолчанию) Создать канал между дочерним и родительским процессами. Родительский конец канала предоставляется родителю как свойство объекта `child_process`как [`subprocess.stdio[fd]`](https://nodejs.org/api/child_process.html#subprocessstdio). Каналы, созданные для fds 0, 1 и 2, также доступны как [`subprocess.stdin`](https://nodejs.org/api/child_process.html#subprocessstdin), [`subprocess.stdout`](https://nodejs.org/api/child_process.html#subprocessstdout)и [`subprocess.stderr`](https://nodejs.org/api/child_process.html#subprocessstderr), соответственно. Это не настоящие каналы Unix, и поэтому дочерний процесс не может использовать их с помощью своих файлов дескрипторов, например `/dev/fd/2`или `/dev/stdout`.
###### `'overlapped'` ???
То же самое, что и `'pipe'`за исключением того, что `FILE_FLAG_OVERLAPPED` флаг установлен на дескрипторе. Это необходимо для перекрывающегося ввода-вывода на дескрипторах stdio дочернего процесса. Подробности см. [в документации](https://docs.microsoft.com/en-us/windows/win32/fileio/synchronous-and-asynchronous-i-o) 
Это тоже самое что и  `'pipe'` на non-Windows системах.

###### `'ignore'`
указывает Node.js игнорировать fd в дочернем процессе. Хотя Node.js всегда будет открывать fd 0, 1 и 2 для порождаемых им процессов, установка fd в `'ignore'`заставит Node.js открыть `/dev/null`и прикрепить его к fd дочернего процесса.

###### `'inherit'`: 
Использовать потоки родительского процесса. В первых трех позициях это эквивалентно `process.stdin`, `process.stdout`, и `process.stderr`, соответственно. В любой другой позиции эквивалентно `'ignore'`.

###### `'ipc'`: 
создать канал IPC для передачи сообщений/файловых дескрипторов между родительским и дочерним процессами.  Установка этого параметра включает [`subprocess.send()`](https://nodejs.org/api/child_process.html#subprocesssendmessage-sendhandle-options-callback)метод. Если дочерний процесс является экземпляром Node.js, наличие канала IPC включит методы [`process.send()`](https://nodejs.org/api/process.html#processsendmessage-sendhandle-options-callback)и [`process.disconnect()`](https://nodejs.org/api/process.html#processdisconnect), а также [`'disconnect'`](https://nodejs.org/api/process.html#event-disconnect)и [`'message'`](https://nodejs.org/api/process.html#event-message)события внутри дочернего процесса.

То есть мы можем сделать `fork` через `spawn`, и настроить там ipc канал вот так:
```javascript
const { spawn } = require('child_process');
const path = require('path');

const child = spawn(process.execPath, [path.join(__dirname, 'child.js')], {
  stdio: ['pipe', 'pipe', 'pipe', 'ipc']
});

child.on('message', (msg) => {
  console.log('From child (spawn):', msg);
});

child.send({ hello: 'world (spawn)' });
```

###### `Object<Stream>`:
Можно передать туда stream, чтобы процесс писал/читал из потока. Это удобно, если мы хотим создать `detached` процесс, при этом не игнорировать его stdout

```js
const { openSync } = require('node:fs');
const { spawn } = require('node:child_process');
const out = openSync('./out.log', 'a');
const err = openSync('./out.log', 'a');

const subprocess = spawn('prg', [], {
  detached: true,
  stdio: [ 'ignore', out, err ],
});

subprocess.unref();
```

 `options.detached` - позволяет дочернему процессу продолжать работу после выхода из родительского.
 То есть можно создать абсолютно независимый процесс, при этом его следует полностью отделить от родительского. Для этого следует игнорировать его stdio.
 По умолчанию родитель будет ждать завершения отсоединенного дочернего процесса. Чтобы родительский процесс не ждал `subprocess`завершения заданного процесса, используется `subprocess.unref()`. Это приведет к тому, что цикл событий родительского процесса не будет включать дочерний процесс в свой счетчик ссылок, что позволит родительскому процессу завершиться независимо от дочернего процесса, если только не установлен канал IPC между дочерним и родительским процессами.

#### `fork` 
предназначен для создания дочерних процессов Node.js, которые могут взаимодействовать с родительским процессом через IPC (Inter-Process Communication)

При этом логи из child процесса будут выводится в консоль родителя, это значит, ему отдается его stdin/out ('inherit')


```javascript
// main.js
const { fork } = require('child_process');

const child = fork('child.js');

child.on('message', (msg) => {
  console.log('Сообщение от дочернего процесса:', msg);
});

child.send({ hello: 'world' });

```

```javascript
// child.js
process.on('message', (msg) => {
  console.log('Получено от родительского процесса:', msg);
  process.send({ response: 'ok' });
});
```

| **Метод**    | **Особенности**                                                | **Когда использовать**                                                               |
| ------------ | -------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **exec**     | Выполняет команду в оболочке, возвращает строки stdout/stderr. | Простые команды оболочки с небольшим объемом данных.                                 |
| **execFile** | Выполняет файл напрямую, без использования оболочки.           | Запуск бинарных файлов или скриптов без необходимости работы с оболочкой.            |
| **spawn**    | Запускает процесс с потоковым вводом/выводом.                  | Для процессов, которые требуют потокового взаимодействия с большими объемами данных. |
| **fork**     | Создает дочерние процессы Node.js с IPC.                       | Для выполнения JavaScript-кода в отдельном процессе с обменом данными.               |

# Clusters
Пораждают процессы как и child_process ( The worker processes are spawned using the [`child_process.fork()`](https://nodejs.org/api/child_process.html#child_processforkmodulepath-args-options) method, so that they can communicate with the parent via IPC and pass server handles back and forth. )
В отличии от child_process, может автоматически распределять сетевую нагрузку:

```js
const cluster = require('cluster');  
const http = require('http');  
const os = require('os');  
  
if (cluster.isMaster) {  
	console.log(`Главный процесс ${process.pid} запущен`);  
	  
	const numCPUs = os.cpus().length;  
	console.log(numCPUs)  
	  
	for (let i = 0; i < numCPUs; i++) {  
		cluster.fork();  
	}  
	  
	cluster.on('exit', (worker, code, signal) => {  
		console.log(`Воркер ${worker.process.pid} завершился`);  
		// Можно перезапустить воркер, если требуется  
		cluster.fork();  
	});  

} else {  
	// Воркеры обрабатывают HTTP-запросы  
	http
	.createServer((req, res) => {  
		res.writeHead(200, {'Content-Type': 'text/txt; charset=utf-8'});  
		res.end(`Ответ от воркера ${process.pid}\n`);  
	})
	.listen(8000);  
	  
	console.log(`Воркер ${process.pid} запущен`);  
}
```

child_process и clusters работают в рамках одной операционной системы, чтобы распределить процессы на разные машины, можно использовать TCP соединение через библиотеку `net`

```js
// client.js

const net = require('net');

const socket = new net.Socket();

socket.connect({
 host: '127.0.0.1',
 port: 3000
}, () => {
	socket.write('Client connected')
	socket.on('data', () => {
		const message = data.toString();
		console.log("Data:", data);
	})
})

```

```js
// server.js

const net = require('net');

const server = net.createServer((socket) => {
	socket.write("data")
	socket.on("data", (data) => {
		const message = data.toString();
		console.log("Data:", data);
	})
})

server.listen(3000)

```

Так же эту библиотеку можно использовать на одной машине через Unix-сокеты (Linux) или Named Pipes (Windows)

```js
// server.js
const net = require('net');  
const pipeName = '\\\\.\\pipe\\mypipe';  
  
const server = net.createServer((connection) => {  
	console.log('Клиент подключился.');  
	  
	// Обработка входящих данных от клиента  
	connection.on('data', (data) => {  
		console.log('Получено от клиента:', data.toString());  
		connection.write('Привет, клиент!');  
	});  
	  
	// Обработка отключения клиента  
	connection.on('end', () => {  
		console.log('Клиент отключился.');  
	});  
});  
  
// Запуск сервера  
server.listen(pipeName, () => {  
	console.log(`Сервер запущен на ${pipeName}`);  
});
```

```js
// child.js

const net = require('net');  
const pipeName = '\\\\.\\pipe\\mypipe';  
  
const client = net.connect(pipeName, () => {  
	console.log('Подключено к серверу.');  
	  
	// Отправка данных серверу  
	client.write('Привет, сервер!');  
});  
  
// Обработка данных, полученных от сервера  
client.on('data', (data) => {  
	console.log('Получено от сервера:', data.toString());  
	client.end(); // Закрыть соединение после получения данных  
});  
  
// Обработка завершения соединения  
client.on('end', () => {  
	console.log('Соединение с сервером закрыто.');  
});

```


Links:
https://www.youtube.com/playlist?list=PLL1UEcDHVPjkGyNpXH4Ep4QPXkVDJNetL