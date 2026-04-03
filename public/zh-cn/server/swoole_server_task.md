# Swoole\Server\Task

当设置 `event_object => true` 时，[task事件](/server/events?id=task) 回调机制将切换为面向对象模式。本文档将对 `Swoole\Server\Task` 类进行全面讲解。

!> `Swoole\Server\Task`对象只能通过 [task事件](/server/events?id=task)的回调参数被框架自动创建并传入，不能直接实例化。单独 `new Swoole\Server\Task()` 没有任何实际意义，这种对象无法正常工作。

```php
$port = new Task();  // ❌ 无效，无法使用
```


## 属性

### data
- `worker`进程传递给`task`进程的数据`data`，该属性是一个`string`类型的字符串。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\Server\Task;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2, 'event_object' => true]);

$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});

$server->on('task', function (Server $server, Task $task) {
    echo $task->data . PHP_EOL;
});

$server->start();
```

### dispatch_time
- 返回该数据到达`task`进程的时间`dispatch_time`，该属性是一个`double`类型。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\Server\Task;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2, 'event_object' => true]);

$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});

$server->on('task', function (Server $server, Task $task) {
    echo $task->dispatch_time . PHP_EOL;
});

$server->start();
```

### id
- 当前`task`任务的id，该属性是一个`int`的整数。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\Server\Task;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2, 'event_object' => true]);

$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});

$server->on('task', function (Server $server, Task $task) {
    echo $task->id . PHP_EOL;
});

$server->start();
```

### worker_id
- 返回该数据来自哪一个`worker`进程，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\Server\Task;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2, 'event_object' => true]);

$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});

$server->on('task', function (Server $server, Task $task) {
    echo $task->worker_id . PHP_EOL;
});

$server->start();
```

### flags
- 该异步任务的一些标志位信息`flags`，该属性是一个`int`类型的整数。

* **示例**


```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\Server\Task;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2, 'event_object' => true]);

$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});

$server->on('task', function (Server $server, Task $task) {
    echo $task->flags . PHP_EOL;
});

$server->start();
```

?> `flags`返回的结果是以下几种类型：  
  - SWOOLE_TASK_NOREPLY | SWOOLE_TASK_NONBLOCK 表示这不是`Worker`进程发送给`task`进程的，此时如果在`onTask`事件中调用`Swoole\Server::finish()`的话，将会有一个警告发出。  

  - SWOOLE_TASK_CALLBACK | SWOOLE_TASK_NONBLOCK 表示`Swoole\Server::finish()`中最后一个回调函数不是null，`onFinish`事件将不会执行，而只会执行这个回调函数。 

  - SWOOLE_TASK_COROUTINE | SWOOLE_TASK_NONBLOCK 表示将会通过协程的形式处理任务。 
 
  - SW_TASK_NONBLOCK 默认值，当以上三种情况都没有的时候。

## 方法

### finish()

用于在 [Task进程](/learn?id=taskworker进程)中通知`Worker`进程，投递的任务已完成。此函数可以传递结果数据给`Worker`进程。

```php
Swoole\Server\Task->finish(mixed $data): bool
```

  * **参数**

    * `mixed $data`

      * 功能：任务处理的结果内容
      * 默认值：无
      * 其它值：无

  * **提示**
    * `finish`方法可以连续多次调用，`Worker`进程会多次触发[onFinish](/server/events?id=onfinish)事件
    * 在[onTask](/server/events?id=ontask)回调函数中调用过`finish`方法后，`return`数据依然会触发[onFinish](/server/events?id=onfinish)事件
    * `Swoole\Server\Task->finish`是可选的。如果`Worker`进程不关心任务执行的结果，不需要调用此函数
    * 在[onTask](/server/events?id=ontask)回调函数中`return`字符串，等同于调用`finish`

  * **注意**

  !> 使用`Swoole\Server\Task->finish`函数必须为`Server`设置[onFinish](/server/events?id=onfinish)回调函数。此函数只可用于 [Task进程](/learn?id=taskworker进程)的[onTask](/server/events?id=ontask)回调中


### pack()

将给定的数据序列化。

```php
Swoole\Server\Task->pack(mixed $data): string|false
```

  * **参数**

    * `mixed $data`

      * 功能：任务处理的结果内容
      * 默认值：无
      * 其它值：无

  * **返回值**
    * 调用成功返回序列化后的结果。 

### unpack()

将给定的数据反序列化。

```php
Swoole\Server\Task->unpack(string $data): mixed
```

  * **参数**

    * `string $data`

      * 功能：需要反序列化的数据
      * 默认值：无
      * 其它值：无

  * **返回值**
    * 调用成功返回反序列化后的结果。 

## 使用示例
```php
<?php
$server->on('task', function(Swoole\Server $serv, Swoole\Server\Task $task) {
    $task->finish(['result' => true]);
});
```
