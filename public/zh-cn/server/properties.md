# 属性

- `Swoole\Http\Server`、`Swoole\Websocket\Server` 以及 `Swoole\Redis\Server` 均继承自 `Swoole\Server` 类。因此，这些子类不仅共享以下属性，同时也完全继承了父类的所有公共方法（如 `set`、`close` 等）。

!> 以下所有属性本质上是只读快照。修改属性值不会影响服务的运行状态。例如，服务启动后修改其中的进程数配置，并不会实际增加或减少进程。

### setting

- 通过 [Swoole\Server->set()](/server/methods?id=set) 方法配置的所有参数，最终都会被保存在 `Swoole\Server->$setting` 属性中。该属性是一个数组（`array`），允许开发者在回调函数中随时访问和读取当前的运行时配置。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->set([  'worker_num' => 4  ]);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->setting);
});

$server->start();
```

### connections

- 这是一个由连接的文件描述符（fd）组成的数组迭代器。你可以使用 foreach 遍历服务器当前所有的连接。此属性的功能与[Swoole\Server->getClientList()](/server/methods?id=getclientlist)是一致的，但是更加友好。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  foreach($server->connections as $fd) {  // 遍历当前连接
    var_dump($fd);
  }
});

$server->start();
```

!> `$connections`属性是一个迭代器对象，不是PHP数组，所以不能用`var_dump`或者数组下标来访问，只能通过`foreach`进行遍历操作。

!> 只有在[SWOOLE_PROCESS](/learn?id=swoole_process)模式下，该属性会保存所有 `worker` 进程的连接；在其他模式下，该属性为每个 `worker` 进程独立持有，互不干扰。

!> 自 `Swoole 5.0+` 起，默认运行模式已由 [SWOOLE_PROCESS](/learn?id=swoole_process) 调整为 [SWOOLE_BASE](/learn?id=swoole_base)。

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  foreach($server->connections as $fd) {  // SWOOLE_PROCESS模式下，$server->connections保存了所有`worker`进程的连接（fd），其他进程可以访问彼此的连接（fd）。
    var_dump($fd);
  }
});
```

### host

- 返回当前服务器监听的主机地址的`host`，该属性是一个`string`类型的字符串。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->host);  // 输出127.0.0.1
});

$server->start();
```

### port

- 返回当前服务器监听的端口的`port`，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->port);  // 输出9501
});

$server->start();
```

### type

- 返回当前服务器的 `Socket` 类型，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;

// 示例一：TCP 服务
$server = new Server('127.0.0.1', 9501, SWOOLE_BASE, SWOOLE_TCP);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->type);  // SWOOLE_SOCK_TCP
});

$server->start();

// 示例二：UDP 服务
$server = new Server('127.0.0.1', 9501, , SWOOLE_BASE, SWOOLE_UDP);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->type);  // SWOOLE_SOCK_UDP
});

$server->start();
```


- 该属性会返回以下常量值之一：

  - `SWOOLE_SOCK_TCP`：TCP IPv4 Socket
  
  - `SWOOLE_SOCK_TCP6`：TCP IPv6 Socke
  
  - `SWOOLE_SOCK_UDP`：UDP IPv4 Socket
  
  - `SWOOLE_SOCK_UDP6`：UDP IPv6 Socket
  
  - `SWOOLE_SOCK_UNIX_DGRAM`：Unix Socket Dgram
  
  - `SWOOLE_SOCK_UNIX_STREAM`：Unix Socket Stream


### ssl

- 返回当前服务器是否启动`ssl`，该属性是一个`bool`类型。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

// 示例：HTTP 服务
$server = new Server('127.0.0.1', 443, SWOOLE_BASE, SWOOLE_TCP | SWOOLE_SSL);
$server->on('request', function(Request $request, Response $response) use ($server) {
  var_dump($server->ssl);  // 输出 true
});

$server->start();
```

### mode

- 返回当前服务器的进程模式`mode`，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

// 示例：HTTP 服务
$server = new Server('127.0.0.1', 9501, SWOOLE_BASE);
$server->on('request', function(Request $request, Response $response) use ($server) {
  var_dump($server->mode);  // 输出 SWOOLE_BASE
});

$server->start();
```

> 该属性返回以下值之一：
>
> - `SWOOLE_BASE` — 单进程模式
> - `SWOOLE_PROCESS` — 多进程模式
> - `SWOOLE_THREAD` — 多线程模式



### ports

监听端口数组，如果服务器监听了多个端口可以遍历`Server::$ports`得到所有`Swoole\Server\Port`对象。

其中`swoole_server::$ports[0]`为构造方法所设置的主服务器端口。

  * **示例**

```php
$ports = $server->ports;
$ports[0]->set($settings);
$ports[1]->on('Receive', function () {
    //callback
});
```

### master_pid

- 返回`master`进程的`PID`。

* **示例**

```php
use Swoole\Server;
$server = new Server("127.0.0.1", 9501);
$server->on('start', function ($server){
    echo $server->master_pid;
});
$server->on('receive', function ($server, $fd, $reactor_id, $data) {
});
$server->start();
```

!> `master` 进程仅存在于 [SWOOLE_PROCESS](/learn?id=swoole_process) 模式下，其他模式下该属性无效。



### manager_pid

- 返回当前`manager`进程的`PID`，该属性是一个`int`类型的整数。

* **示例**

```php
use Swoole\Server;
$server = new Server("127.0.0.1", 9501);
$server->on('start', function ($server) {
    echo $server->manager_pid;
});

$server->on('receive', function ($server, $fd, $reactor_id, $data) {

});
$server->start();
```

!> 只能在`Start/WorkerStart`之后获取到


    
### worker_id

- 得到当前`worker`进程或者[task进程](/learn?id=taskworker进程)的编号，编号从0开始，该属性是一个`int`类型的整数。

* **示例**

```php
$server = new Swoole\Server('127.0.0.1', 9501);
$server->set([
    'worker_num' => 8,
    'task_worker_num' => 4,
]);
$server->on('WorkerStart', function ($server, int $workerId) {
    if ($server->taskworker) {
        echo "task workerId：{$workerId}\n";
        echo "task worker_id：{$server->worker_id}\n";
    } else {
        echo "workerId：{$workerId}\n";
        echo "worker_id：{$server->worker_id}\n";
    }
});
$server->on('Receive', function ($server, $fd, $reactor_id, $data) {
});
$server->on('Task', function ($serv, $task_id, $reactor_id, $data) {
});
$server->start();
```

- **提示**
  - 该属性与 [`onWorkerStart`](/server/events?id=onworkerstart) 回调中的 `$workerId` 参数含义相同。

  - `worker`进程和[Task 进程](/learn?id=taskworker进程)重启后`worker_id`的值是不变的。

  - 若 `worker_num` 设为 4，则 worker 进程编号范围为 `[0, worker_num - 1]`，即 `[0, 3]`。

  - 若 `worker_num` 为 4，`task_worker_num` 为 8，则 [task 进程](/learn?id=taskworker进程) 的编号范围为 `[worker_num, worker_num + task_worker_num - 1]`，即 `[4, 11]`。
  

### taskworker

- 当前进程是否是 `Task` 进程，`true`表示当前的进程是`Task`工作进程，`false`表示当前的进程是其他进程，该属性是一个`bool`类型。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->set(['task_worker_num' => 2]);

$server->on('task', function(Server $server, int $taskId, int $workerId, mixed $data) {
  var_dump($server->taskworker); // task进程输出true
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->taskworker); // worker进程输出false
});

$server->start();
```


### worker_pid

- 得到当前`Worker`进程的`PID`，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->worker_pid);
});

$server->start();
``` 

!>  [SWOOLE_BASE](/learn?id=swoole_base)和[SWOOLE_PROCESS](/learn?id=swoole_process)模式下，该属性的值与`posix_getpid()`的返回值相同。

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501， SWOOLE_BASE);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->worker_pid == posix_getpid());
});

$server->start();
``` 

!> [SWOOLE_THREAD](/learn?id=swoole_thread)模式下，该属性的值与`Swoole\Thread::getId()`的返回值相同。

```php
<?php
use Swoole\Server;
use Swoole\Thread;

$server = new Server('127.0.0.1', 9501， SWOOLE_THREAD);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  var_dump($server->worker_pid == Thread::getId());
});

$server->start();
``` 
