
#### Threads
Node.js изначально является **однопоточной** платформой, что означает, что основной поток (или Event Loop) отвечает за выполнение JavaScript-кода
Начиная с 10 версии Node.js можно использовать **Worker Threads** (рабочие потоки), которые позволяют выполнять задачи в отдельных потоках.

#### Processes
Процесс — это независимая, изолированная единица выполнения программы. У каждого процесса есть собственная память, системные ресурсы и поток выполнения.


**IPC (Inter-Process Communication)** — это механизм, который позволяет процессам обмениваться данными и взаимодействовать друг с другом. В Node.js IPC используется для обмена информацией между основным процессом и дочерними процессами (например, через модуль `child_process`).

### `child_process`

Основные методы:

- `exec` - используется для выполнения команд в оболочке (shell) и возвращает результат их выполнения в виде строки.

```
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

```
const { execFile } = require('child_process');

execFile('node', ['-v'], (error, stdout, stderr) => {
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

- `spawn` - запускает процесс, но в отличие от `exec` и `execFile`, он возвращает потоки (streams) для stdin, stdout и stderr.

```
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

- `fork` предназначен для создания дочерних процессов Node.js, которые могут взаимодействовать с родительским процессом через IPC (Inter-Process Communication)


```
// main.js
const { fork } = require('child_process');

const child = fork('child.js');

child.on('message', (msg) => {
  console.log('Сообщение от дочернего процесса:', msg);
});

child.send({ hello: 'world' });

```

```
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

### Clusters
Пораждают процессы как и child_process ( The worker processes are spawned using the [`child_process.fork()`](https://nodejs.org/api/child_process.html#child_processforkmodulepath-args-options) method, so that they can communicate with the parent via IPC and pass server handles back and forth. )
В отличии от child_process, может автоматически распределять сетевую нагрузку:

```
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

```
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

```
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

```
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

```
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