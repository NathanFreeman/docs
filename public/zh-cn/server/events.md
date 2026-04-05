# 回调事件

从 [TCP/UDP 服务器](/server/tcp_init)、[HTTP/HTTPS/HTTP2 服务器](/http_server) 和 [WebSocket 服务器](/websocket_server) 章节中，可以看到一些结构相似的代码示例：

```php
$server->on('receive', function (Server $server, int $fd, int $reactorId, string $data) {
    $server->send($fd, 'Hello World');
});
```

这类通过 [Swoole\Server->on()](/server/methods?id=on) 方法注册的，就是**回调事件**。回调事件在整个异步服务器模型中扮演着核心角色。通过它们，可以定义当客户端发送数据、进程启动或退出、甚至客户端连接建立或关闭时，服务器应该执行哪些逻辑。

以上面的代码为例，可以简单地理解为，**一旦客户端发送数据过来，服务器就会自动触发 receive 事件，并执行与之绑定的函数——也就是向客户端回复一句 Hello World**。 

本节将系统介绍 Swoole 异步服务器所支持的所有事件类型。每个事件都绑定一个 PHP 函数（即事件回调），用于响应对应的事件触发。

## start

- 启动后在主进程（master）的主线程触发此函数

```php
function(Swoole\Server $server) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **说明**：Swoole\Server 对象实例
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('start', function(Server $server) {
  echo "master process start";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```

---


* **在此事件之前 Server 已完成以下操作**

  * 创建完成 [Manager 进程](/learn?id=manager进程)
  * 创建完成 [Worker 子进程](/learn?id=worker进程)
  * 监听所有 TCP/UDP/[Unix Socket](/learn?id=什么是IPC) 端口，但尚未开始 Accept 连接和请求
  * 定时器已就绪

* **接下来将执行**

  * [Reactor](/learn?id=reactor线程) 线程开始接收事件，客户端可连接到服务端


**使用说明**

`start` 触发中仅允许执行 `echo`、打印日志、修改进程名称等简单操作，**不得调用 `Swoole\Server` 相关函数**，此时服务尚未完全就绪。`start` 与 `workerStart` 触发在不同进程中并行执行，不存在先后顺序。

建议在 `start` 触发中将 `Swoole\Server->master_pid` 和 `Swoole\Server->manager_pid` 保存至文件，以便编写管理脚本向这两个 PID 发送信号，实现服务的关闭与重启。

> **注意**：在 `start` 中创建的全局资源对象无法在 Worker 进程中使用。因为 `start` 调用时 Worker 进程已创建完成，新对象位于主进程内存空间，Worker 进程无法访问。

> **注意**：[SWOOLE_BASE](/learn?id=swoole_base) 模式下没有 Master 进程，因此不存在 `start` 事件，**请勿在 BASE 模式中使用 `start` 触发**。

---

**安全提示**

- `start` 事件中可使用异步和协程 API，但需注意可能与 `dispatch_func` 和 `package_length_func` 配置存在冲突，**请勿同时使用**。
- 请勿在 `start` 中启动定时器。若代码中执行了 `Swoole\Server::shutdown()`，定时器将导致程序无法正常退出。
- `start` 事件返回前服务端不会接受任何客户端连接，因此可安全使用同步阻塞函数。

```php
<?php
use Swoole\Server;
use Swoole\Timer;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('start', function(Server $server) {
  // 禁止在这里操作定时器，否则主进程永远无法退出
  Timer::add(1000, function() {});
  echo "master process start";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```


## connect
- 有新连接进入时，会在worker进程中触发此事件。

```php
function(Swoole\Server $server, int $fd, int $reactorId) {}
```


* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $fd`**
    * **功能**：连接的文件描述符
    * **默认值**：无
    * **其它值**：无

  * **`int $reactorId`**
    * **功能**：SWOOLE_PROCESS模式下，该值为`TCP`连接所在的[Reactor](/learn?id=reactor线程)线程序号，否则是`worker`进程序号。
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('connect', function(Server $server, int $fd, int $reactorId) {
  echo 'new connection coming' . PHP_EOL;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

!> `HTTP`服务器和`WebSocket`服务器不接受`connect`回调。


## beforeShutdown

- 在进程正常退出前触发此事件

```php
function (Swoole\Server $server) {}
```


* **参数**

  * **`Swoole\Server $server`**
    * **说明**：Swoole\Server 对象实例
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('beforeshutdown', function(Server $server) {
    echo "Server is about to shutdown\n";
    // 在此可执行资源清理、状态保存等操作
    // 支持协程 API
});
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```

---

##### 版本要求

- Swoole 版本 >= `v4.8.0`

---

##### 触发时机

当通过信号或终止服务进程时，以下进程会触发 `beforeshutdown` 事件：

| 进程类型                       | 是否触发 `beforeshutdown` |
|----------------------------|-----------------------|
| **Master 进程**              | ✅ 触发                  |
| **Worker 进程**              | ✅ 触发                  |
| **Task 进程**                | ✅ 触发                  |
| **Manager 进程**             | ✅ SWOOLE_BASE模式下触发    |
| **Swoole\Process 用户自定义进程** | ❌ 不触发                 |


---

##### 注意事项

1. **Manager 进程和Swoole\Process 用户自定义进程不会触发此事件**。
2. 事件触发中支持协程，可安全使用协程 API。
3. 此事件在进程**正常退出**时触发，强制 `kill -9`和 `ctrl + c` 等信号不会触发。

## shutdown

- 该事件在进程正常退出时触发

```php
function(Swoole\Server $server) {}
```

* **参数说明**

  * **`Swoole\Server $server`**
    * **说明**：Swoole\Server 对象实例
    * **默认值**：无
    * **其他值**：无

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('shutdown', function(Server $server) {
    echo "Server is shutdown\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

---

#### **触发时机**

当通过信号或终止服务进程时，以下进程会触发 `shutdown` 事件：

| 进程类型                       | 是否触发 `shutdown` |
|----------------------------|-------------------------|
| **Master 进程**              | ✅ 触发 |
| **Worker 进程**              | ❌ 不触发 |
| **Task 进程**                | ❌ 不触发 |
| **Manager 进程**             | ✅ SWOOLE_BASE模式下触发 |
| **Swoole\Process 用户自定义进程** | ❌ 不触发 |

在此之前，底层已自动完成以下清理工作：

- 关闭所有 **Reactor 线程**、**HeartbeatCheck 线程**、**UdpRecv 线程**
- 关闭所有 **Worker 进程**、**Task 进程**、**User 进程**
- 关闭所有 **TCP/UDP/UnixSocket** 监听端口
- 关闭主 **Reactor 线程**

!> 强制终止进程（如 `kill -9`）不会触发 `shutdown` 事件。  

!> 需使用 `kill -15` 向主进程发送 `SIGTERM` 信号，方可按照正常流程终止程序。  

!> 在命令行中使用 `Ctrl+C` 中断程序时会立即停止，底层同样不会触发 `shutdown`。

!> 请勿在`onShutdown`中调用任何异步或协程相关`API`，触发`onShutdown`时底层已销毁了所有事件循环设施；

!> 此时已经不存在协程环境，如果开发者需要使用协程相关`API`需要手动调用`Co\run`来创建[协程容器](/coroutine?id=什么是协程容器)。


## workerStart 
- 此事件在 Worker进程/ [Task进程](/learn?id=taskworker进程) 启动时发生，这里创建的对象可以在进程生命周期内使用。

```php
function(Swoole\Server $server, int $workerId) {}
```

* **参数说明**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $workerId`**
    * **功能**：进程序号，非系统 PID。
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('workerStart ', function (Server $server, int $workerId) {
    echo "Worker 启动，ID: {$workerId}\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```

---


###### 代码热重载支持
若要支持进程重启机制实现代码热重载，**可以在 `workerStart` 中引入代码文件**。这样进程重启会加载新的代码文件。 

在 `workerStart ` 之前引入的文件（如公共库）不会在进程重启时重新加载，但可在所有进程间共享内存。可以将公用的、不易变的 `php` 文件放置到 `workerStart ` 之前。这样虽然不能重载入代码，但所有 `Worker` 是共享的，不需要额外的内存来保存这些数据。 `workerStart ` 之后的代码每个进程都需要在内存中保存一份。

可以这样理解：

当 Worker 进程或 Task 进程被**重新创建**时（例如通过 `reload` 或进程意外退出后重启），**Manager 进程**会负责 fork 出新的子进程。

- 在 **`workerStart ` 之前**引入的文件（如公共库），是在 **Manager 进程启动时**就已经加载到内存中的。  
  这些文件**不会因为 Worker 进程的重启而重新加载**，因为 Manager 进程本身没有重启。

- 在 **`workerStart ` 触发中**引入的文件（如业务代码），会在**每个 Worker 进程启动时重新加载**，从而实现代码热更新的效果。


```
Manager 进程启动
    ├── 加载公共库（如框架核心、配置等）
    │
    ├── fork Worker 进程 0
    │       └── 执行 workerStart  → 加载业务代码
    │
    ├── fork Worker 进程 1
    │       └── 执行 workerStart  → 加载业务代码
    │
    └── reload 时
            ├── Manager 进程（公共库仍在内存中，不重新加载）
            └── 重新 fork 新 Worker 进程
                    └── 再次执行 onworkerStart  → 重新加载业务代码（实现热更新）
```

```php
require __DIR__ . '/公共库.php'; // 这份文件所有worker进程都能读取到，但是进程重启不会热加载该文件。
$server->on('workerStart ', function ($server, $workerId) {
    require __DIR__ . '/进程A专属文件.php'; // 这份文件只能在进程A读取到，进程B无法读取
});
```

---


###### 协程支持

在 `onworkerStart ` 触发中**会自动创建协程环境**，因此可以直接使用协程 API：
```php
$server->on('workerStart ', function ($server, $workerId) {
    file_put_contents('data.txt', 'hello world');
});
```


!> `workerStart ` 与 `onStart` 是**并发执行**的，没有先后顺序。

!> 当 `worker_num` 或 `task_worker_num` 大于 1 时，**每个进程都会触发一次** `onworkerStart `。

!> `$workerId` 是进程序号，非系统 PID，可通过 `posix_getpid()` 获取实际进程 PID。

!> 若在 `onworkerStart ` 中发生**致命错误**或主动调用 `exit`，当前 Worker/Task 进程会退出，管理进程会重新创建新进程。若频繁发生，可能导致**进程不断创建与销毁**，影响服务稳定性。

!> 可通过 `$server->taskworker` 判断当前是 Worker 进程还是 Task 进程：

```php
<?php
if ($server->taskworker) {
    echo "当前是 Task 进程";
} else {
    echo "当前是 Worker 进程";
}
```

## workerExit
- 仅在开启[reload_async](/server/setting?id=reload_async)特性才会触发`workerExit`，进程退出前会触发。参见 [如何正确的重启服务](/question/use?id=swoole如何正确的重启服务)

```php
function(Swoole\Server $server, int $workerId) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $workerId`**
    * **功能**：进程序号，非系统 PID。
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([
  'reload_async' => true,  // 开启这个才会触发workerExit
  'max_wait_time' => 10
]);
$server->on('workerExit', function (Server $server, int $workerId) {
    echo "Worker 停止，ID: {$workerId}\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```

---


!> `Worker`进程未退出，`workerExit`会持续触发。

!> `workerExit`会在`Worker`进程内触发， [Task进程](/learn?id=taskworker进程)中如果存在[事件循环](/learn?id=什么是eventloop)也会触发。

!> 在`workerExit`中尽可能地移除/关闭异步的`Socket`连接，最终底层检测到[事件循环](/learn?id=什么是eventloop)中事件监听的句柄数量为`0`时退出进程。

!> 当进程没有事件句柄在监听时，进程结束时将不会触发此函数。

!> 等待`Worker`进程退出后才会执行`workerStop`事件触发。

!> 如果进程超过`max_wait_time`秒后仍未退出，系统会强制杀死进程，并且提示`worker exit timeout, forced termination`。

## workerStop

- 此事件在`worker`进程终止时发生。在此函数中可以回收`worker`进程申请的各类资源。

```php
function (Swoole\Server $server, int $workerId) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $workerId`**
    * **功能**：进程序号，非系统 PID。
    * **默认值**：无
    * **其它值**：无

  
---

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('workerStop', function (Server $server, int $workerId) {
    echo "Worker 停止，ID: {$workerId}\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

---


!> 进程异常结束，如被强制`kill`、致命错误、`core dump`时无法执行`workerstop`触发函数。  

!> 请勿在`workerstop`中调用任何异步或协程相关`API`，触发`workerstop`时底层已销毁了所有[事件循环](/learn?id=什么是eventloop)设施。


## workerError
- 此事件在`worker`进程异常退出时触发。

```php
function (Server $server, int $workerId, int $pid, int $exitCode, int $signal) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $workerId`**
    * **功能**：进程序号，非系统 PID。
    * **默认值**：无
    * **其它值**：无

  * **`int $pid`**
    * **功能**：进程 PID。
    * **默认值**：无
    * **其它值**：无

  * **`int $exitCode`**
    * **功能**：进程退出状态码，0~255。
    * **默认值**：无
    * **其它值**：无

  * **`int $signal`**
    * **功能**：进程退出的信号。
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('workerError', function (Server $server, int $workerId, int $pid, int $exitCode, int $signal) {
    echo "Worker 异常退出，ID: {$workerId}\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

---


> 此函数主要用于报警和监控，一旦发现Worker进程异常退出，那么很有可能是遇到了致命错误或者进程Core Dump。通过记录日志或者发送报警的信息来提示开发者进行相应的处理。

> `$signal = 11`：说明`Worker`进程发生了`segment fault`段错误，可能触发了底层的`BUG`，请收集`core dump`信息和`valgrind`内存检测日志，[向Swoole开发组反馈此问题](/other/issue)。

> `$exitCode = 255`：说明Worker进程发生了`Fatal Error`致命错误，请检查PHP的错误日志，找到存在问题的PHP代码，进行解决。

> `$signal = 9`：说明`worker`被系统强行`Kill`，请检查是否有人为的`kill -9`操作，检查`dmesg`信息中是否存在`OOM（Out of memory）`

> 如果存在`OOM`，分配了过大的内存。1.检查`Server`的`setting`配置，是否[socket_buffer_size](/server/setting?id=socket_buffer_size)等分配过大；2.是否创建了非常大的[Swoole\Table](/memory/table)内存模块。

## beforeReload
- worker进程/task进程重启之前触发此事件，该事件在Manager进程中执行

```php
function(Server $server, int $fd, int $reactorId, string $data) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('beforeReload', function (Server $server) {
    echo "Worker before reload\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

---


> `worker`进程重启是可重入的，在上一个重启请求未完成时，下一个重启请求将会被忽略。

!> 禁止在`beforeReload`事件中使用阻塞函数例如`while(1) {}`来阻塞代码，否则进程将无法重启。

> 如果在规定的`max_wait_time`时间中进程没有重启完毕，所有的`worker`进程会被强制杀死。

> 向`master` 进程 / `manager` 进程`发送`SIGUSR1`信号，将平稳地重启所有`worker`进程和`task`进程。

> 向`master` 进程 / `manager` 进程`发送`SIGUSR2`信号，将平稳地重启所有`task`进程。

> `Swoole\Process`进程退出时，不会触发此事件。

## afterReload
- worker进程/task进程重启之后触发此事件，该事件在Manager进程中执行。

```php
function(Swoole\Server $server, int $fd, int $reactorId, string $data) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('afterReload', function (Server $server) {
    echo "Worker after reload\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```



## task
- worker进程向task进程发送数据时，在task进程中触发此事件，该方式用于处理一些耗时任务，避免worker进程阻塞

```php
function (Swoole\Server $server, int $taskId, int $workerId, mixed $data) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $taskId`**
    * **功能**：执行任务的 `task` 进程序号【`$taskId`和`$workerId`组合起来才是全局唯一的，不同的`worker`进程投递的任务`ID`可能会有相同】
    * **默认值**：无
    * **其它值**：无

  * **`int $workerId`**
    * **功能**：投递任务的 `worker` 进程序号【`$taskId`和`$workerId`组合起来才是全局唯一的，不同的`worker`进程投递的任务`ID`可能会有相同】
    * **默认值**：无
    * **其它值**：无

  * **`mixed $data`**
    * **功能**：任务的数据内容
    * **默认值**：无
    * **其它值**：无


---


* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2]);
$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});
$server->on('task', function (Server $server, int $taskId, int $workerId, mixed $data) {
    echo $data . PHP_EOL;
});

$server->start();
```


---



!> 必须设置`task_worker_num`，否则无法创建`task`进程，`worker`进程也没法投递任务。

!> 禁止在`task`事件中使用`Swoole\Server->task()`，底层会检查环境并且抛出`Server->task() cannot use in the task-worker`。

! 若未显式指定目标`task`进程，底层会通过取模方式从所有`task`进程中选择一个进行任务投递。不管该进程是否空闲。

! `task`进程接收到任务，会将自身状态设置为忙碌，这时将不再接收新的Task，如果所有的`task`进程全部忙碌，投递任务时会提示`No idle task worker is available`，需要等待目标`task`进程空闲下来。

! 执行时遇到致命错误退出，或者被外部进程强制`kill`，当前的任务会被丢弃，但不会影响其他正在排队的任务。

## finish
- 此触发函数在worker进程被调用，当`worker`进程投递的任务在`task`进程中完成时， [task进程](/learn?id=taskworker进程)会通过`Swoole\Server->finish()`函数或者`return`操作将任务处理的结果发送给`worker`进程。

```php
function(Swoole\Server $server, int $taskId, mixed $data) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $taskId`**
    * **功能**：执行任务的 `task` 进程序号。
    * **默认值**：无
    * **其它值**：无

  * **`mixed $data`**
    * **功能**：任务处理的结果内容。
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2]);
$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});

$server->on('finish', function(Server $server, int $taskId, mixed $data) {
  echo $data . PHP_EOL; // task进程通过`Swoole\Server->finish()`返回数据  
});

$server->on('task', function (Server $server, int $taskId, int $workerId, mixed $data) use ($server) {
    echo $data . PHP_EOL;
    $server->finish($data);  // return $data; 也是同样的效果
});

$server->start();
```


---




> 如果在 [task](/server/events?id=task) 事件中**没有调用** `Swoole\Server->finish()` 函数，也**没有**通过 `return` 返回结果，则 worker 进程**不会触发** [finish](/server/events?id=finish) 事件。

> 如果 [task](/server/events?id=task) 事件返回了结果，但**没有监听** `finish` 事件，底层会抛出异常：`require 'onFinish' callback`。

> 执行 [finish](/server/events?id=finish) 逻辑的 worker 进程，与下发该 task 任务的 worker 进程是**同一个进程**。

## managerStart
- 当Manager进程启动时触发此事件

```php
function(Swoole\Server $server, int $fd, int $reactorId, string $data) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无


---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('managerStart', function (Server $server) {
    echo "manager start\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

---




> 在这个触发函数中可以修改管理进程的名称。

!> 在`4.2.12`以前的版本中`manager`进程中不能添加定时器，不能投递task任务、不能用协程。在`4.2.12`或更高版本中`manager`进程可以使用基于信号实现的同步模式定时器。

> `manager`进程中可以调用[sendMessage](/server/methods?id=sendMessage)接口向其他工作进程发送消息

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('managerStart', function (Server $server) {
    $server->sendMessage('Hello World', 1); // 发送消息给序号为1的进程
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});


$server->on('pipeMessage', function(Server $server, int $workerId, mixed $message) {
  // 收到manager进程通过Swoole\Server->sendMessage()发送的信息
});

$server->start();
```


## managerStop
- 当Manager进程结束时触发。

```php
function(Swoole\Server $server) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('managerStop', function (Server $server) {
    echo "manager stop\n";
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();
```

---



> `managerStop`触发时，说明`task`和`worker`进程已结束运行，已被`Manager`进程回收。

## pipeMessage
- 当`worker`进程 / [Task进程](/learn?id=taskworker进程)进程收到由 `Swoole\Server->sendMessage()` 发送的[unixSocket](/learn?id=什么是IPC)消息时会触发 `pipeMessage` 事件。`worker/task` 进程都可能会触发 `pipeMessage` 事件。

```php
function(Swoole\Server $server, int $workerId, mixed $message) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $workerId`**
    * **功能**：消息来自哪个进程序号
    * **默认值**：无
    * **其它值**：无

  * **`mixed $message`**
    * **功能**：消息内容，可以是任意PHP类型
    * **默认值**：无
    * **其它值**：无


* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Reponse;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([
  'worker_num' => 2
]);

$server->on('request', function (Request $request, Reponse $response) use ($server) {
    if ($server->worker_id == 0) {
      $server->sendMessage('Hello World', 1); // 发送消息给序号为1的进程
    } else {
      $server->sendMessage('Hello World', 0); // 发送消息给序号为0的进程
    }
});

$server->on('pipeMessage', function(Server $server, int $workerId, mixed $message) {
  var_dump($workerId);
  var_dump($message);
});

$server->start();
```


## receive
- 接收到`TCP`数据时触发此函数，在`worker`进程触发该事件。

```php
function(Swoole\Server $server, int $fd, int $reactorId, string $data) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $fd`**
    * **功能**：连接的文件描述符
    * **默认值**：无
    * **其它值**：无

  * **`int $reactorId`**
    * **功能**：SWOOLE_PROCESS模式下，该值为`TCP`连接所在的[Reactor](/learn?id=reactor线程)线程序号，否则是`worker`进程序号。
    * **默认值**：无
    * **其它值**：无

  * **`string $data`**
    * **功能**：收到的数据内容，可能是文本或者二进制内容
    * **默认值**：无
    * **其它值**：无


---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```



---


> **注意**
>
> - 若**未开启** `open_http_protocol`、`open_websocket_protocol`、`open_http2_protocol`、`open_mqtt_protocol`、`open_redis_protocol` 等协议解析选项，`receive` 触发每次接收到的数据最大为 **64 KB**。
> - 若**开启**上述任一协议，`receive` 将接收完整的应用层数据包，大小受 `package_max_length` 限制。但**不推荐**这种用法，建议改用对应的专用服务端类（如 `Swoole\Http\Server` 处理 HTTP 请求），否则底层会检查并抛出如下警告：
    >
    >   `Swoole\Server::start(): use Swoole\Server class and open http related protocols may lead to some errors (inconsistent class type)`

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set([
  'open_http_protocol' => true,
  'package_max_length' => 2 * 1024 * 1024 // 最多接收2M的文件
]);

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```


## packet
- 接收到`UDP`数据时触发此函数，在`worker`进程触发该事件。

```php
function (Swoole\Server $server, mixed $data, array $clientInfo) {}
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`string $data`**
    * **功能**：收到的数据内容，可能是文本或者二进制内容
    * **默认值**：无
    * **其它值**：无

  * **`array $clientInfo`**
    * **功能**：客户端信息
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS, SWOOLE_SOCK_UDP);
$server->on('packet', function (Server $server, mixed $data, array $clientInfo) {
    echo $data . PHP_EOL;
});

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});
$server->start();
```



## request
- 接收到`HTTP`数据时触发此函数，在`worker`进程触发该事件。

```php
function (Swoole\Http\Request $request, Swoole\Http\Reponse $response) {}
```

* **参数**

  * **`Swoole\Http\Request $request`**
    * **功能**：`HTTP` 请求对象，保存了 `HTTP` 客户端请求的相关信息，包括 `GET`、`POST`、`COOKIE`、`Header` 等。
    * **默认值**：无
    * **其它值**：无

  * **`Swoole\Http\Reponse $response`**
    * **功能**：`HTTP` 响应对象，通过调用此对象的函数，实现 `HTTP` 响应发送。
    * **默认值**：无
    * **其它值**：无


---

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Reponse;

$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('request', function (Request $request, Reponse $response) {
    echo "接收到http请求" . PHP_EOL;
});


$server->start();
```


## beforeHandshakeResponse
- 当`WebSocket`客户端与服务器握手前会触发此函数，如果你不需要自定义握手处理过程，但是又想设置一些`http header`信息到响应头，那么就可以调用这个事件。

```php
function (Swoole\Http\Request $request, Swoole\Http\Response $response) {}
```

* **参数**

  * **`Swoole\Http\Request $request`**
    * **功能**：HTTP请求信息对象，包含客户端请求信息。
    * **默认值**：无
    * **其它值**：无

  * **`Swoole\Http\Response $reponse`**
    * **功能**：HTTP响应信息对象，可以通过这个对象发送响应给客户端。
    * **默认值**：无
    * **其它值**：无


---

* **示例**

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;
use Swoole\Http\Request;
use Swoole\Http\Response;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('beforeHandshakeResponse', function (Request $request, Response $response) {
    var_dump($request);
});

$server->on('message', function (Server $server,  Frame $frame) {
    echo $frame->data . PHP_EOL;
});

$server->start();
```

---

##### 版本要求

- Swoole 版本 >= `v5.0.0`

## handshake
-- 如果用户希望自己进行`websocket`握手，可以设置`handShake`事件触发函数。

```php
function (Swoole\Http\Request $request, Swoole\Http\Response $response) {}
```

* **参数**

  * **`Swoole\Http\Request $request`**
    * **功能**：HTTP请求信息对象，包含客户端请求信息。
    * **默认值**：无
    * **其它值**：无

  * **`Swoole\Http\Response $reponse`**
    * **功能**：HTTP响应信息对象，可以通过这个对象发送响应给客户端。
    * **默认值**：无
    * **其它值**：无

---

* **示例**

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;
use Swoole\Http\Request;
use Swoole\Http\Response;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('handshake', function (Request $request, Response $response) {
    // print_r( $request->header );
    // if (如果不满足我某些自定义的需求条件，那么响应空输出，返回false，握手失败) {
    //    $response->end();
    //    return false;
    // }

    // websocket握手连接算法验证
    $secWebSocketKey = $request->header['sec-websocket-key'];
    $patten = '#^[+/0-9A-Za-z]{21}[AQgw]==$#';
    if (0 === preg_match($patten, $secWebSocketKey) || 16 !== strlen(base64_decode($secWebSocketKey))) {
        $response->end();
        return false;
    }
    echo $request->header['sec-websocket-key'];
    $key = base64_encode(
        sha1(
            $request->header['sec-websocket-key'] . '258EAFA5-E914-47DA-95CA-C5AB0DC85B11',
            true
        )
    );

    $headers = [
        'Upgrade' => 'websocket',
        'Connection' => 'Upgrade',
        'Sec-WebSocket-Accept' => $key,
        'Sec-WebSocket-Version' => '13',
    ];

    // 如果请求头有Sec-WebSocket-Protocol，响应头必须设置相同的Sec-WebSocket-Protocol
    if (isset($request->header['sec-websocket-protocol'])) {
        $headers['Sec-WebSocket-Protocol'] = $request->header['sec-websocket-protocol'];
    }

    foreach ($headers as $key => $val) {
        $response->header($key, $val);
    }

    $response->status(101);
    $response->end();
});

$server->on('message', function (Server $server,  Frame $frame) {
    echo $frame->data . PHP_EOL;
});

$server->start();
```

---



> 如果需要自行处理 `handshake` 的时候，再设置这个触发函数。如果不需要自定义握手过程，那么不要设置该触发，使用`Swoole`默认的握手即可。

!> 设置 `handShake` 触发函数后不会再触发`open`事件。

> `handShake` 中必须调用 `Swoole\Http\Response->status()` 设置状态码为 `101` 并调用 `Swoole\Http\Response->end()` 响应，否则会握手失败。

!> 内置的握手协议为 `Sec-WebSocket-Version: 13`，低版本浏览器需要自行实现握手。

## open
- 当`WebSocket`客户端与服务器建立连接并完成握手后会触发此函数。

```php
function (Swoole\WebSocket\Server $server,  Request $request) {}
```

* **参数**

  * **`Swoole\WebSocket\Server $server`**
    * **功能**：`Swoole\WebSocket\Server` 对象。
    * **默认值**：无
    * **其它值**：无

  * **`Swoole\Http\Request $request`**
    * **功能**：HTTP 请求信息。
    * **默认值**：无
    * **其它值**：无


---

* **示例**

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;
use Swoole\Http\Request;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('open', function (Server $server,  Request $request) {
    var_dump($request);
});

$server->on('message', function (Server $server,  Frame $frame) {
    echo $frame->data . PHP_EOL;
});

$server->start();
```

---


> $request 是一个 HTTP 请求对象，包含了客户端发来的握手请求信息，因为`websocket`是先通过http协议执行握手阶段的。

> `open`事件函数中可以调用 `Swoole\Websocket\Server->push()` 向客户端发送数据或者调用 `Swoole\Websocket\Server->close()` 关闭连接。


## message
- 接收到`websocket`数据时触发此函数，在`worker`进程触发该事件。

```php
function (Swoole\WebSocket\Server $server,  Swoole\WebSocket\Frame $frame) {}
```


* **参数**

  * **`Swoole\WebSocket\Server $server`**
    * **功能**：`Swoole\WebSocket\Server` 对象。
    * **默认值**：无
    * **其它值**：无

  * **`Swoole\WebSocket\Frame $frame`**
    * **功能**：websocket客户端发送过来的帧数据。
    * **默认值**：无
    * **其它值**：无

---


* **示例**

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('message', function (Server $server,  Frame $frame) {
    echo $frame->data . PHP_EOL;
});

$server->start();
```



## disconnect
- `webSocket`关闭连接时会触发该事件。

```php
function (Swoole\WebSocket\Server $server, int $fd) {}
```

* **参数**

  * **`Swoole\WebSocket\Server $server`**
    * **功能**：`Swoole\WebSocket\Server` 对象。
    * **默认值**：无
    * **其它值**：无

  * **`int $fd`**
    * **功能**：已关闭连接的客户端文件描述符（唯一标识）。
    * **默认值**：无
    * **其它值**：无


---


* **示例**

```php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('message', function (Server $server,  Frame $frame) {
    echo $frame->data . PHP_EOL;
});

$server->on('disconnect', function (Server $server, int $fd) {
    echo 'disconnect' . PHP_EOL;
});

$server->start();
```



## close
- `TCP`客户端连接关闭后，在`Worker`进程中触发此函数。

```php
function(Swoole\Server $server, int $fd, int $reactorId) {
```

* **参数**

  * **`Swoole\Server $server`**
    * **功能**：Swoole\Server对象
    * **默认值**：无
    * **其它值**：无

  * **`int $fd`**
    * **功能**：连接的文件描述符
    * **默认值**：无
    * **其它值**：无

  * **`int $reactorId`**
    * **功能**：来自哪个`reactor`线程序号，服务端主动`close`关闭时为负数。
    * **默认值**：无
    * **其它值**：无

---


* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->on('close', function(Server $server, int $fd, int $reactorId) {
  echo "connection close" . PHP_EOL;
});

$server->start();
```

---



> 当服务器主动关闭连接时，底层会设置$reactorId参数为 `-1`，可以通过判断 `$reactorId < 0` 来分辨关闭是由服务器端还是客户端发起的。

> 只有在 `PHP` 代码中主动调用 `Swoole\Server->close()` 函数被视为主动关闭。、

> `close` 触发函数如果发生了致命错误，会导致连接泄漏。通过 netstat 命令会看到大量 CLOSE_WAIT 状态的 TCP 连接。 

> 无论由客户端发起 close 还是服务器端主动调用 `Swoole\Server->close()` 关闭连接，都会触发此事件。因此只要连接关闭，就一定会触发此函数。

>  `close` 中依然可以调用 `Swoole\Server->getClientInfo()` 函数获取到连接信息，在 `close` 触发函数执行完毕后才会调用实际关闭 `TCP` 连接。

> 这里触发 `close` 时表示客户端连接已经关闭，所以无需在`close`事件中执行 `Swoole\Server->close()`。否则会抛出 PHP 错误警告。

## 事件区别

> `receive`、`request` 和 `message` 均用于处理客户端消息，核心区别在于**数据处理层级**：
> *   **`receive`** ：接收原始的 **TCP 字节流**。框架不对数据进行任何解析，需由开发者手动实现协议解码。
> *   **`request` / `message`** ：接收底层框架**自动解析后的结构化数据**，可直接业务逻辑中使用。

> `disconnect`和`close`两者均用于终止连接，但作用域不同：
> *   **`disconnect`** ：专门用于关闭 **WebSocket 逻辑连接**（处理握手状态及协议级断开）。
> *   **`close`** ：用于关闭底层的 **TCP 物理连接**，释放操作系统层面的文件描述符资源。


> `workerExit`和`workerStop`两者均在进程退出触发，但作用不同：
> *   **`workerExit`** ：用于在 `max_wait_time` 规定的时间内执行**柔性关闭**，主动清理并关闭所有事件句柄监听。
> *   **`workerStop`** ：仅作为**进程停止的通知**，不参与柔性关闭，执行时机在 Worker 进程完全退出之后。


## 面向对象风格
当设置 `event_object => true` 时，Swoole 服务器的一些事件回调机制将切换为**面向对象模式**。

#### 核心变化
- **传统模式**：回调函数接收多个分散的参数（如 `$fd`, `$reactor_id`, `$data` 等）。
- **面向对象模式**：回调函数的参数变为一个**封装好的对象实例**。该对象包含了当前事件的所有上下文信息，属性访问更直观，类型提示更友好。


```php
<?php

use Swoole\Server;
use Swoole\Server\Event;
use Swoole\Server\Packet;
use Swoole\Server\PipeMessage;
use Swoole\Server\StatusInfo;
use Swoole\Server\TaskResult;

// 创建服务器实例
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

// 开启面向对象事件模式
$server->set([
    'event_object' => true, 
]);

// 1. 连接事件：接收 Event 对象
$server->on('connect', function (Server $serv, Event $event) {
    // $event->fd, $event->reactor_id 等
    var_dump($event);
});

// 2. 数据接收事件：接收 Event 对象 (包含 data 属性)
$server->on('receive', function (Server $serv, Event $event) {
    // $event->data 即为收到的消息内容
    var_dump($event);
});

// 3. 关闭连接事件：接收 Event 对象
$server->on('close', function (Server $serv, Event $event) {
    var_dump($event);
});

// 4. UDP 数据包事件：接收 Packet 对象
$server->on('packet', function (Server $serv, Packet $packet) {
    // $packet->data, $packet->address, $packet->port
    var_dump($packet);
});

// 5. 管道消息事件：接收 PipeMessage 对象
$server->on('pipeMessage', function (Server $serv, PipeMessage $msg) {
    var_dump($msg);
    
    // 演示如何从对象中获取数据并发送
    $payload = $msg->data; 
    // 假设 $payload 是一个包含 address, port, data, server_socket 的对象/数组
    $serv->sendto(
        $payload->address, 
        $payload->port, 
        $payload->data, 
        $payload->server_socket ?? 0
    );
});

// 6. Worker 错误事件：接收 StatusInfo 对象
$server->on('workerError', function (Server $serv, StatusInfo $info) {
    // $info->worker_id, $info->error_type, $info->errno 等
    var_dump($info);
});

// 7. 异步任务完成事件：接收 TaskResult 对象
$server->on('finish', function (Server $serv, TaskResult $result) {
    // $result->data, $result->task_id
    var_dump($result);
});

$server->start();
```

## 注意事项

#### 1. 事件名称大小写不敏感
底层在处理事件注册时，会自动将事件名称统一转换为**小写**。因此，以下写法均被视为同一事件，效果完全一致：
- `workerStart`
- `workerstart`
- `WoRkerStarT`

> **建议**：为了代码规范和可读性，推荐始终使用文档标准的**驼峰命名法**（如 `workerStart`）。

#### 2. 服务器类型与必选事件
不同的服务器模式或进程类型，**必须**监听对应的核心事件，否则启动时会抛出警告或导致服务无法正常工作。

| 服务器/进程类型 | **必须监听的事件** | 说明 |
| :--- | :--- | :--- |
| **TCP 服务器** | `receive` | 处理客户端发送的原始数据流 |
| **UDP 服务器** | `packet` | 处理客户端发送的数据包 |
| **HTTP 服务器** | `request` | 处理 HTTP 请求并返回响应 |
| **WebSocket 服务器** | `message` | 处理 WebSocket 帧消息 |
| **Task 异步任务进程** | `task`, `finish` | `task`: 接收异步任务<br>`finish`: 接收任务完成结果 |

```php
<?php
// 例1：TCP服务器必须监听receive事件
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$server->start();

// 例2：UDP服务器必须监听packet事件
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS, SWOOLE_SOCK_UDP);

$server->on('packet', function (Server $server, mixed $data, array $clientInfo) {
    echo $data . PHP_EOL;
});

$server->start();

// 例3：HTTP服务器必须监听request事件
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('request', function(Request $request, Response $reponse) {
  var_dump($request);
});

$server->start();


// 例4：WebSocket服务器必须监听message事件
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->on('message', function (Server $server,  Frame $frame) {
    echo $frame->data . PHP_EOL;
});

$server->start();


// 例5：启用task进程时候，必须监听task和finish事件
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
$server = new Server('127.0.0.1', 9501, SWOOLE_PROCESS);
$server->set(['task_worker_num' => 2]);

$server->on('request', function(Request $request, Response $reponse) use ($server) {
  $server->task('Hello Swoole!!!!');
});

$server->on('task', function (Server $server, int $taskId, int $workerId, mixed $data) use ($server) {
    echo $data . PHP_EOL;
    $server->finish($data);
});

$server->on('finish', function(Swoole\Server $server, int $taskId, mixed $data) {
  echo $data . PHP_EOL;
});

$server->start();
```
