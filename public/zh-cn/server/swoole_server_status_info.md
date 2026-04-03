# Swoole\Server\StatusInfo

当设置 `event_object => true` 时，[workerError](/server/events?id=workererror)事件 回调机制将切换为面向对象模式。本文档将对 `Swoole\Server\StatusInfo` 类进行全面讲解。

!> `Swoole\Server\StatusInfo`对象只能通过 [workerError](/server/events?id=workererror)事件的回调参数被框架自动创建并传入，不能直接实例化。单独 `new Swoole\Server\StatusInfo()` 没有任何实际意义，这种对象无法正常工作。

```php
$port = new StatusInfo();  // ❌ 无效，无法使用
```

## 属性

### worker_id
- 返回当前`worker`进程id，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
use Swoole\Server\StatusInfo;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([ 'event_object' => true ]);

$server->on('workerError', function (Server $serv, StatusInfo $info) {
    echo $info->worker_id;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```



### worker_pid
- 返回当前`worker`进程父进程id，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
use Swoole\Server\StatusInfo;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([ 'event_object' => true ]);

$server->on('workerError', function (Server $serv, StatusInfo $info) {
    echo $info->worker_pid;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

### status
- 返回进程状态`status`，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
use Swoole\Server\StatusInfo;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([ 'event_object' => true ]);

$server->on('workerError', function (Server $serv, StatusInfo $info) {
    echo $info->status;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

### exit_code
- 返回进程退出状态码`exit_code`，该属性是一个`int`类型的整数，范围是`0-255`。

* **示例**

```php
<?php
use Swoole\Server;
use Swoole\Server\StatusInfo;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([ 'event_object' => true ]);

$server->on('workerError', function (Server $serv, StatusInfo $info) {
    echo $info->exit_code;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

### signal
- 进程退出的信号`signal`，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
use Swoole\Server\StatusInfo;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([ 'event_object' => true ]);

$server->on('workerError', function (Server $serv, StatusInfo $info) {
    echo $info->signal;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```
