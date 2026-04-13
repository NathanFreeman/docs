# 配置

- [Swoole\Server->set()](/server/methods?id=set) 方法用于设置 `Swoole\Server`、`Swoole\Http\Server`、`Swoole\WebSocket\Server` 和 `Swoole\Redis\Server` 运行时的各项参数。

- 如果您已经阅读过[多端口监听](/server/port)章节，其中提到的 [Swoole\Server\Port->set()](/server/port) 方法的配置项说明也可以在这里找到。

* **示例**

```php
<?php
use Swoole\Server;

$server = new Server('127.0.0.1', 9501);
$server->set([
    'worker_num' => 4
]);

$server->on('receive', function (Server $server, int $fd, int $reactorId, string $data) {
    $server->send($fd, 'Hello World');
});
$server->start();
```

> 自 [v4.5.5](/version/log?id=v455) 版本起，底层会检测配置项的有效性。如果设置了 Swoole 不支持的配置项，将产生一个警告提示。

```shell
PHP Warning:  unsupported option [foo] in @swoole-src/library/core/Server/Helper.php
```

### worker_num
- 设置启动的`Worker`进程数。默认值为`CPU`核数。

- `worker` 进程负责处理实际的业务逻辑。合理设置该值直接影响服务器的处理能力。

---

* **示例：**

```php
$server->set(['worker_num' => 4]); 
```

---

* **注意**

  * 如果`1`个请求耗时`100ms`，要提供`1000QPS`的处理能力，那必须配置`100`个进程或更多，但开的进程越多，占用的内存就会大大增加，而且进程间切换的开销就会越来越大。所以这里适当即可。不要配置过大。
  * 如果业务代码为[异步IO](/learn?id=同步io异步io)的，这里设置为`CPU`核数最合理。
  * 如果业务代码为[同步IO](/learn?id=同步io异步io)，需要根据请求响应时间和系统负载来调整，例如：`CPU`核数 * （2 - 4）倍甚至更大。
  * 默认设置为服务器的 `CPU` 核数，最大不得超过为服务器的 `CPU` 核数 * 1000。
  * 假设每个进程占用`40M`内存，`100`个进程就需要占用`4G`内存。
  * 如果是[SWOOLE_THREAD](/learn?id=swoole_thread)模式，该参数则代表创建的线程数量。


### reactor_num
- 设置启动的 [Reactor](/learn?id=reactor线程) 线程数。默认值为 `CPU` 核数。该配置仅适用于 [SWOOLE_PROCESS](/learn?id=swoole_process) 模式。

- `reactor` 线程负责接收客户端连接和网络请求，是 `Swoole` 的"网络事件监听器"。每个 `reactor` 线程都独立维持一个 [EventLoop](/learn?id=什么是eventloop)，线程之间无锁运行，可以在 128 核 CPU 上并行执行，然后将接收到的请求数据转发给 `worker` 进程处理。

---

* **示例：**

```php
$server->set(['reactor_num' => 4]); 
```

---

* **注意**

  * `reactor_num`建议设置为`CPU`核数的`1-4`倍。
  * `reactor_num`最大不得超过 `CPU`核数 * `4`。
  * `reactor_num`必须小于或等于`worker_num`，如果设置的`reactor_num`大于`worker_num`，会自动调整使`reactor_num`等于`worker_num`。
  * `reactor_num`在超过`8`核的机器上默认设置为`8`。


### task_worker_num
- 配置 [task进程](/server/process_thread?id=task) 将启用耗时任务投递功能，此时必须同时注册 `[task](/server/events?id=task)` 和 `[finish](/server/events?id=finish)` 两个事件回调函数，否则服务器程序无法启动。

---

* **示例：**

```php
$server->set(['task_worker_num' => 4]); 
```

---

* **注意**

  * `task`进程数最大不得超过为服务器的 `CPU` 核数 * 1000。
  * 假设单个`task`进程的处理耗时为`100ms`，那一个进程1秒就可以处理`1/0.1=10`个任务。
  * `task`进程内不能使用`Swoole\Server->task()`方法

### enable_coroutine

- 控制是否在[事件回调](/server/events)中自动创建协程来执行业务逻辑，默认值为 `true`。

---

* **示例：**

```php
$server->set(['enable_coroutine' => true]); 
```

---

* **配置方式**

  * 在 `php.ini` 中设置 `swoole.enable_coroutine = 'Off'`（详见 [ini配置文档](/other/config.md)）
  * 通过 `$server->set(['enable_coroutine' => false]);` 设置的优先级高于 `php.ini`

---

* **注意**

  * 当 `enable_coroutine = true` 时，底层会在 `request` 等回调中自动创建协程。
  * 当 `enable_coroutine = false` 时，底层不会自动创建协程。如果业务中不需要使用协程，关闭该选项可提升一定性能。如需使用协程，需自行通过 `go()` 创建；若不需要协程特性，行为与 Swoole 1.x 完全一致。
  * 开启该选项仅表示会通过协程方式处理请求。如果回调中包含了阻塞操作（如 `sleep`、`mysqlnd` 扩展等），还需额外配置 `hook_flags` 以实现阻塞函数和扩展的协程化。

---

* **受 `enable_coroutine` 影响的事件回调有**

  * [workerStart](/server/events?id=workerStart)
  * [connect](/server/events?id=connect)
  * [open](/server/events?id=open)
  * [receive](/server/events?id=receive)
  * [setHandler](/redis_server?id=sethandler)
  * [packet](/server/events?id=packet)
  * [request](/server/events?id=request)
  * [message](/server/events?id=message)
  * [pipeMessage](/server/events?id=pipeMessage)
  * [finish](/server/events?id=finish)
  * [close](/server/events?id=close)

### task_enable_coroutine

- 开启[task进程](/server/process_thread?id=task)协程化。开启后自动在[task事件回调](/server/events?id=task)创建协程和[协程容器](/coroutine/scheduler)，可以直接使用协程`API`。

---

* **示例：**

```php
$server->set([
  'enable_coroutine' => true
  'task_enable_coroutine' => true
]); 
```

--- 

* **注意**

  * `task_enable_coroutine`必须在[enable_coroutine](/server/setting?id=enable_coroutine)为`true`时才可以使用
  * 自 `Swoole 4.2.12` 版本起支持该选项。


### hook_flags

- 用于设置哪些阻塞函数或扩展需要被替换为协程版本，以实现异步非阻塞的协程调度。默认不替换任何阻塞函数或者扩展。

---

* **示例：**

```php
$server->set([
    'hook_flags' => SWOOLE_HOOK_SLEEP | SWOOLE_HOOK_NATIVE_CURL,
]);
```

---


* **支持的协程化选项：**

可使用 `SWOOLE_HOOK_ALL` 一键开启全部协程化项，也可按需组合使用以下常量：

| 常量名                             | 说明                                                         |
|---------------------------------|------------------------------------------------------------|
| `SWOOLE_HOOK_TCP`               | TCP 操作（如 `stream_socket_client`）                           |
| `SWOOLE_HOOK_UNIX`              | Unix Socket 操作                                             |
| `SWOOLE_HOOK_UDP`               | UDP 操作                                                     |
| `SWOOLE_HOOK_UDG`               | Unix 数据报套接字                                                |
| `SWOOLE_HOOK_SSL`               | SSL 封装层                                                    |
| `SWOOLE_HOOK_TLS`               | TLS 封装层                                                    |
| `SWOOLE_HOOK_SLEEP`             | `sleep`、`usleep`、`time_nanosleep`、`time_sleep_until` 等睡眠函数 |
| `SWOOLE_HOOK_FILE`              | 文件操作（如 `fread`、`fwrite`、`file_get_contents` 等）             |
| `SWOOLE_HOOK_STREAM_FUNCTION`   | 流函数（如 `stream_select`、`stream_set_blocking`）               |
| `SWOOLE_HOOK_BLOCKING_FUNCTION` | 各类阻塞函数（如 `gethostbyname`、`exec`、`shell_exec` 等）            |
| `SWOOLE_HOOK_PROC`              | 进程操作（如 `proc_open`）                                        |
| `SWOOLE_HOOK_CURL`              | CURL 协程化（基础版）                                              |
| `SWOOLE_HOOK_NATIVE_CURL`       | CURL 原生协程化（更完整、推荐）                                         |
| `SWOOLE_HOOK_SOCKETS`           | sockets 扩展相关函数                                             |
| `SWOOLE_HOOK_STDIO`             | 标准输入输出（如 `stdin`、`stdout`、`stderr`）                        |
| `SWOOLE_HOOK_PDO_PGSQL`         | PDO PostgreSQL 驱动协程化                                       |
| `SWOOLE_HOOK_PDO_ODBC`          | PDO ODBC 驱动协程化                                             |
| `SWOOLE_HOOK_PDO_ORACLE`        | PDO Oracle 驱动协程化                                           |
| `SWOOLE_HOOK_PDO_SQLITE`        | PDO SQLite 驱动协程化                                           |
| `SWOOLE_HOOK_ALL`               | 开启以上所有协程化项                                                 |

!> Swoole版本为 `v4.5+` 或 [4.4LTS](https://github.com/swoole/swoole-src/tree/v4.4.x) 可用，详情参考[一键协程化](/runtime)

### enable_reuse_port

- 设置端口复用，启用后，支持多个进程或服务同时监听同一个端口（需内核支持）。默认值：`false`。

---

* **示例：**

```php
$server->set([
    'enable_reuse_port' => true,
]);
```


!> 仅在`Linux-3.9.0`以上版本的内核可用 `Swoole4.5`以上版本可用


### max_request

- 设置 [worker进程](/server/process_thread?id=worker) 可以执行的最大任务数。当 worker 进程处理的任务数达到该值时，进程会自动重启，以释放占用的内存和资源。默认值为`0`，表示不设限制，进程不会退出。

---

* **示例：**

```php
$server->set([
    'max_request' => 10000,
]);
```

---

* **注意**

  * 达到`max_request`不一定马上关闭进程，参考[max_wait_time](/server/setting?id=max_wait_time)。
  * [SWOOLE_BASE](/server/process_thread?id=swoole_base)下，达到`max_request`后重启进程会导致客户端连接断开。
  * 这个参数的主要作用是解决由于程序编码不规范导致的PHP进程内存泄露问题。PHP应用程序有缓慢的内存泄漏，但无法定位到具体原因、无法解决，可以通过设置`max_request`临时解决，需要找到内存泄漏的代码并修复，而不是通过此方案，可以使用Swoole Tracker发现泄漏的代码。


### task_max_request

- 设置 [task进程](/server/process_thread?id=task) 可以执行的最大任务数。当 task 进程处理的任务数达到该值时，进程会自动重启，以释放占用的内存和资源。默认值为`0`，表示不设限制，进程不会退出。

---

* **示例：**

```php
$server->set([
    'task_max_request' => 10000,
]);
```

### max_connection / max_conn

- 设置服务器的最大连接数。当服务器已建立的连接数达到该值时，新进入的连接将被拒绝。默认值为 `ulimit -n`（系统当前用户最大打开文件数）。

---

* **示例**

```php
$server->set([
  'max_connection' => 10000
]);
```

---

* **注意**

  * 如果应用层未显式设置 `max_connection`，底层将自动使用 `ulimit -n` 的值作为默认值。
  * `max_connection` 的值不能超过操作系统的 `ulimit -n` 值。若设置过大，底层会发出警告并自动将配置重置为 `ulimit -n` 的值。
  * 在 `Swoole 4.2.9` 及以上版本中，当检测到 `ulimit -n` 设置过大（如 `100 万`）时，底层会将默认值调整为 `100000`。这是因为过大的 `ulimit -n` 需要分配大量内存用于连接信息存储，可能导致进程启动失败。
  * 若设置的值过小，底层也会发出警告，并将其调整为 `ulimit -n` 的值。配置的最小有效值为 `(worker_num + task_worker_num) * 2 + 32`。
  * `max_connection` 不应盲目调大，应根据服务器实际可用内存合理设置。Swoole 会按照该数值预先分配一块内存用于保存连接信息，每个 TCP 连接约占用 `224` 字节。

### max_coroutine / max_coro_num

- 设置当前工作进程最大协程数量。默认值：`100000`，Swoole版本小于`v4.4.0-beta` 时默认值为`3000`。

---

* **示例**

```php
$server->set([
  'max_coroutine' => 10000
]);
```

---

* **注意**
  * 超过`max_coroutine`底层将无法创建新的协程，底层会抛出`exceed max number of coroutine`错误，`TCP Server`会直接关闭连接，`Http Server`会返回Http的503状态码。
  * 在服务器程序中实际最大可创建协程数量等于 `worker_num * max_coroutine`，[task进程](/server/process_thread?id=task)和[user进程](/server/process_thread?id=user)的协程数量单独计算。

### max_concurrency
- 限制`HTTP`，`HTTP2`服务器最大并发请求数量，超过该值之后，以后的请求会返回`503`错误，默认值为`4294967295`，即为无符号 `int` 的最大值。

---

* **示例**

```php
$server->set([
  'max_concurrency' => 10000
]);
```

### worker_max_concurrency

- 开启一键协程化之后，`worker` 进程会源源不断的接受请求，为了避免压力过大，我们可以设置 `worker_max_concurrency` 限制 `worker` 进程的请求执行数。

- 当请求数超过该值时，`worker` 进程会将多余的请求暂存于队列，默认值为 `4294967295`，即为无符号 `int` 的最大值。

- 如果没有设置 `worker_max_concurrency`，但是设置了 `max_concurrency` 的话，底层会自动设置 `worker_max_concurrency` 等于 `max_concurrency`。


---

* **示例**

```php
$server->set([
  'worker_max_concurrency' => 10000
]);
```

---

* **说明**
  * 如果进程重启时队列中仍有未处理的请求，底层会遍历整个队列，并向其中的每个请求逐一返回 `503 Service Unavailable`。
  * 可以这样理解：`worker_max_concurrency` 限制的是**单个进程**的并发处理能力，而 `max_concurrency` 限制的是**整个服务器**的总并发处理能力，即所有进程正在处理的请求总数不能超过该值。

!> Swoole版本 >= `5.0.0` 可用。

### max_idle_time

- 该配置用于设置服务器 `TCP` 连接的读写超时阈值。当服务器连接在读写操作时间时候超过`max_idle_time`，底层将主动关闭该连接。默认值为：`0`，表示不进行超时检查。

---

* **示例**

```php
$server->set([
  'max_idle_time' => 10
]);
```

---

* **注意**
  * 新连接加入事件循环后，若在 `max_idle_time` 时间内未发送任何数据，底层将关闭该连接。
  * 读取客户端发送的数据过程中，若从开始读取到完成的耗时超过 `max_idle_time`尚未完成，底层将关闭该连接。
  * 写入数据发送给客户端过程中，若写入操作的耗时超过 `max_idle_time` 尚未完成，底层将关闭该连接。

### heartbeat_idle_time
- 客户端`TCP`连接最大允许空闲的时间，单位为秒。表示一个客户端连接如果`heartbeat_idle_time`秒内未向服务器发送任何数据，此连接将被强制关闭。

---

* **示例**

```php
$server->set([
  'heartbeat_idle_time' => 3,         // 表示一个连接如果3秒内未向服务器发送任何数据，此连接将被强制关闭
  'heartbeat_check_interval' => 60,   // 表示每60秒遍历一次
]);
```

* **注意**
  * 启用 `heartbeat_idle_time` 后，服务器并不会主动向客户端发送数据包。
  * 如果只设置了 `heartbeat_idle_time` 未设置 `heartbeat_check_interval` 底层将不会创建心跳检测线程。
  * 如果没有设置`heartbeat_idle_time`，只设置了`heartbeat_check_interval`，那么`heartbeat_idle_time`会设置为`heartbeat_check_interval`的两倍。


### heartbeat_check_interval
- 每个多少秒发一次`TCP`连接心跳检测，默认值为：0，表示不开启心跳检测。

---

* **示例**

```php
$server->set([
  'heartbeat_idle_time' => 3,         // 表示一个连接如果3秒内未向服务器发送任何数据，此连接将被强制关闭
  'heartbeat_check_interval' => 60,   // 表示每60秒遍历一次
]);
```
---

* **注意**
  * 服务端并不会主动向客户端发送心跳包，而是被动等待客户端发送心跳。服务器端的 `heartbeat_check` 仅仅是检测连接上一次发送数据的时间，如果超过限制，将切断连接。
  
  * 此选项表示每隔多久轮循一次，单位为秒。如 heartbeat_check_interval => 60，表示每 60 秒，遍历所有连接，如果该连接在 `heartbeat_idle_time` 秒内没有向服务器发送任何数据，此连接将被强制关闭。
  
  * 若未配置，则不会启用心跳，该配置默认关闭。
  
  * 被心跳检测切断的连接依然会触发[close事件回调](/server/events?id=close)。

### package_max_length

- 设置能接收的最大数据包尺寸，单位为字节。默认值：`2M` 即 `2 * 1024 * 1024`，最小值为`64K`。

---

* **示例**

```php
$server->set([
  'package_max_length' => 2 * 1024 * 1024
]);
```

---

* **注意**

  * 此参数不宜设置过大，否则会占用很大的内存。
  
  * 开启[open_length_check](/server/setting?id=open_length_check)/[open_eof_check](/server/setting?id=open_eof_check)/[open_eof_split](/server/setting?id=open_eof_split)/[open_http_protocol](/server/setting?id=open_http_protocol)/[open_http2_protocol](/http_server?id=open_http2_protocol)/[open_websocket_protocol](/server/setting?id=open_websocket_protocol)/[open_mqtt_protocol](/server/setting?id=open_mqtt_protocol)等协议解析后，Swoole 底层会进行数据包拼接。在完整接收一个数据包之前，所有数据均保存在内存中。因此必须设置 `package_max_length` 来限制单个数据包的最大内存占用。例如，若有 1 万个 TCP 连接同时发送数据，每个数据包大小为 `2M`，在极端情况下内存占用将达到 `20G`。
  
  * `open_length_check`：当发现包长度超过`package_max_length`，将直接丢弃此数据，并关闭连接，不会占用任何内存。
  
  * `open_eof_check`：因为无法事先得知数据包长度，所以收到的数据还是会保存到内存中，持续增长。当发现内存占用已超过`package_max_length`时，将直接丢弃此数据，并关闭连接。
  
  * `open_http_protocol`：`GET`请求最大允许`8K`，而且无法修改配置。`POST`请求会检测`Content-Length`，如果`Content-Length`超过`package_max_length`，将直接丢弃此数据，发送`http 400`错误，并关闭连接。

### open_eof_check 
- 此选项用于检测客户端发来的数据：仅当`数据包结尾`匹配指定分隔符`（如 \r\n）`时，底层才会停止接收数据；否则，数据将被持续拼接，直到超出缓存区限制或接收超时才会中止。

- 若检测过程发生错误，底层会将其视为恶意连接，直接丢弃数据并强制关闭连接。参考 [TCP 数据包边界问题](/learn?id=tcp数据包边界问题)。

- 性能极高，几乎无额外开销。

---

* **示例**

```php
$server->set([
  'open_eof_check' => true,
  'package_eof' => '\r\n'
]);
```

!> 此配置仅适用于 **STREAM 类型** 的 Socket（例如 TCP、Unix Socket Stream）。对于指定分隔符检测，系统**不会**在数据流中间查找指定分隔符，因此[worker进程](/process_thread?id=worker)可能会一次性收到多个数据包。你需要在应用层代码中自行拆分数据包，例如使用 `explode("\r\n", $data)` 进行分包处理。

### open_eof_split
- 底层逐字节扫描数据流，每次遇到指定分隔符`（如 \r\n）`就自动切分并交付一个完整数据包给[worker进程](/process_thread?id=worker)。

---

* **示例**

```php
$server->set([
  'open_eof_split' => true,
  'package_eof' => '\r\n'
]);
```

---

* **注意**
  * 启用 `open_eof_split` 后，底层会在数据流中查找指定分隔符并自动拆分数据包，确保 [receive回调](/server/events?id=receive) 每次都只收到一个以该分隔符结尾的完整数据包。
  * `open_eof_check` 仅检查数据包末尾是否包含指定分隔符，性能极佳、几乎无额外开销，但无法解决多包合并问题：当客户端连续发送多个带 EOF 的数据包时，底层可能一次性全部返回，需要业务层自行拆包。
  * `open_eof_split`采用从左到右的逐字节扫描方式查找指定分隔符来拆分数据包，性能较差，且每次仅返回一个数据包。
  * `open_eof_split`优先级大于`open_eof_check`。

### package_eof
- 设置数据包指定分隔符。

---

* **示例**

```php
$server->set([
  'open_eof_split' => true,
  'package_eof' => '\r\n'
]);
```

!> `package_eof` 最大只允许传入 `8` 个字节的字符串。

### open_length_check
- 启用数据包长度检测协议解析功能。默认值为：`false`。参考 [TCP 数据包边界问题](/learn?id=tcp数据包边界问题)。

---

* **示例**

```php
$server->set([
  'open_length_check' => true
]);
```

---

* **注意**
  * 开启后，服务器底层会自动根据你设定的规则，可以保证[reactor线程](/server/process_thread?id=master) 或者 [worker进程](/server/process_thread?id=worker) 每次都会收到一个完整的数据包。

### package_length_type
- 指定长度字段的类型（如 'N' 表示 4 字节无符号长整型）。与 `PHP` 的 `pack` 函数一样。告诉服务器：长度字段占几个字节，以及这些字节的排列顺序（字节序）。

---

* **示例**

```php
$server->set([
  'package_length_type' => 'N'
]);
```

---

* **注意，目前 Swoole 支持 10 种长度字段类型：**

| 字符参数 | 作用                 |
|------|--------------------|
| c    | 有符号、1字节            |
| C    | 无符号、1字节            |
| s    | 有符号、主机字节序、2字节      |
| S    | 无符号、主机字节序、2字节      |
| n    | 无符号、网络字节序、2字节      |
| N    | 无符号、网络字节序、4字节      |
| l    | 有符号、主机字节序、4字节（小写L） |
| L    | 无符号、主机字节序、4字节（大写L） |
| v    | 无符号、小端字节序、2字节      |
| V    | 无符号、小端字节序、4字节      |

### package_length_offset

- 定位长度字段位置的核心参数。它告诉服务器：“长度字段这个值，存放在整个数据包的第几个字节”。

- 因为有些协议设计不一定总是把长度字段放在数据包的最开头。可能包头还有其他固定信息。长度字段可能偏移几个字节后才出现。

---

* **示例**

- 假设你的协议格式如下：

```text
[ magic: 2字节 ] [ command: 2字节 ] [ length: 4字节 ] [ body: N字节 ]
字节0-1           字节2-3           字节4-7         字节8开始
```

- 长度字段 length 位于字节 4、5、6、7（共4字节），那么配置就是：

```php
$server->set([
    'open_length_check'     => true,
    'package_length_type'   => 'N',      // 4字节无符号长整型
    'package_length_offset' => 4,        // 长度字段从第4字节开始 
    'package_body_offset'   => 8,        // 包体从第8字节开始（跳过整个包头）
    'package_max_length'    => 81920,
]);
```

* **注意**
  * 偏移量一定是从`0`开始的。

### package_body_offset / package_body_start

- 定位包体数据起始位置的偏移量。告诉服务器：真正的业务数据（包体）从数据包的第几个字节开始。

---

* **示例1：**

- 假设你的协议格式如下：

```text
[ magic: 2字节 ] [ command: 2字节 ] [ length: 4字节 ] [ body: N字节 ]
字节0-1           字节2-3           字节4-7         字节8开始
```

- body从第8个字节开始，那么配置就是：

```php
$server->set([
    'open_length_check'     => true,
    'package_length_type'   => 'N',      // 4字节无符号长整型
    'package_length_offset' => 4,        // 长度字段从第4字节开始 
    'package_body_offset'   => 8,        // 包体从第8字节开始（跳过整个包头）
    'package_max_length'    => 81920,
]);
```

---

* **示例2：**

- 假设你的协议格式如下：

```text
[ magic: 2字节 ] [ command: 2字节 ] [ length: 4字节 ] [ type: 4字节 ] [ body: N字节 ]
字节0-1           字节2-3           字节4-7            字节8-11       字节12开始
```

- body从第12个字节开始，那么配置就是：

```php
$server->set([
    'open_length_check'     => true,
    'package_length_type'   => 'N',      // 4字节无符号长整型
    'package_length_offset' => 4,        // 长度字段从第4字节开始 
    'package_body_offset'   => 12,        // 包体从第12字节开始（跳过整个包头）
    'package_max_length'    => 81920,
]);
```

---

* **注意**
  * 偏移量一定是从`0`开始的。

### package_length_func

- 自定义一个 `PHP` 函数来告诉 `Swoole`一个完整数据包究竟有多长。

- 这个函数会接收当前接收到的部分数据（存储在一个缓冲区中），然后你需要在这个函数里实现你的核心逻辑：解析并返回整个数据包的长度。

---

```php
function(string $data): int {}
```

* **参数**

  * **`string $data`**
    * **功能**：当前接收到的部分数据
    * **默认值**：无
    * **其它值**：无

* **返回值**

  *  返回 `0`：数据不足，需等待并接收更多数据。
  *  返回 `-1`：数据错误，底层将自动关闭连接。
  *  返回 `>0`：成功获取包长（该值即完整数据包的长度）。


---

* **示例**

- 下面这个例子模拟了一个简单文本协议，其中数据包的格式为 "LENGTH:xxx\nBODY:yyy"，你需要从数据中解析出 xxx 作为长度。

```php
$server->set([
    'open_length_check'   => true,          // 1. 必须开启
    'package_max_length'  => 81920,         // 2. 设置最大包长限制
    'package_length_func' => function ($data) {
        // 3. 定义你的解析逻辑
        // 检查是否已经接收到足够的数据，以便解析出长度值
        if (strlen($data) < 16) {
            return 0; // 数据不够，返回0，继续等待更多数据
        }

        // 假设协议是 "LENGTH:123\n"，用正则从收到的数据开头提取长度
        if (preg_match('/^LENGTH:(\d+)\\\n/', $data, $match)) {
            $length = (int)$match[1];
            
            // 获取整个包头的长度，以便计算完整包长
            $header_length = strlen($match[0]);
            
            // 返回整个数据包的总长度 = 包头长度 + 包体长度
            return $header_length + $length;
        }

        // 如果连包头都不符合协议规范，返回 -1，让 Swoole 关闭这个连接
        return -1;
    }
]);
```

---

* **注意**
  * `package_length_func`主要用来处理那些"无法用一个固定偏移量就读取到长度"的复杂协议。
  * 请勿在长度解析函数中执行阻塞 IO 操作，可能导致所有 [reactor线程](/server/process_thread?id=master) 或者 [worker进程](/server/process_thread?id=worker)发生阻塞。
  * 由于 `ZendVM` 不支持运行在多线程环境，因此底层会自动使用 `Mutex` 互斥锁对 `PHP` 长度函数进行加锁，避免并发执行 `PHP` 函数。

!> Swoole版本 >= 1.9.3 可用

### upload_max_filesize
- 设置 `HTTP`，`HTTP2` 服务器允许上传的文件大小上限。防止大文件上传耗尽服务器内存，提升服务稳定性。

---

* **示例**

```php
<?php
use Swoole\Http\Server;
$http = new Server('127.0.0.1', 9501);
$http->set([
  'package_max_length' => 2 * 1024 * 1024,
  'upload_max_filesize' => 4 * 1024 * 1024
]);
```

---

* **注意**
  * `upload_max_filesize` 用于允许 `HTTP` 服务器接收大文件。当接收到的请求报文大小超过 `package_max_length` 但未超过 `upload_max_filesize` 时，服务器不会丢弃数据，而是将请求体内容写入磁盘临时文件，不在内存中保留副本。后续处理时再从文件中读取数据，从而避免内存溢出。

  * 必须配合 `package_max_length` 和 `php` 的 `memory_limit` 使用（两者是硬性上限）。

!> `Swoole 5.0.0` 以上可以使用。

### aio_core_worker_num

- 设置异步线程池最小线程数，默认为 `CPU` 核数。

---

* **示例**

```php
$server->set([
  'aio_core_worker_num' => 10
]);
```

### aio_worker_num

- 设置异步线程池最大线程数，默认为 `CPU` 核数 * 8。

---

* **示例**

```php
$server->set([
  'aio_worker_num' => 80
]);
```

### aio_max_wait_time


- 设置任务最大等待时间，单位为秒。

---

* **示例**

```php
$server->set([
  'aio_max_wait_time' => 80
]);
```

---

* **注意**
  * 如果每个线程都有任务在处理，但是队列积压的任务的等待时间已经超过了`aio_max_wait_time`，底层会扩容出新的线程来处理任务。
  * 扩容的新线程 + 现有线程数不会超过`aio_worker_num`。

### aio_max_idle_time


- 设置异步线程池线程的空闲时间，单位为秒，超过该时间没有接收到任务的线程将会被回收。

---

* **示例**

```php
$server->set([
  'aio_max_idle_time' => 10
]);
```

---


* **注意**
  * 在底层回收空闲线程时，底层会保证线程数不会少于`aio_core_worker_num`。

### iouring_entries

- 设置 `io_uring` 的队列大小时，默认值为 8192。若传入的值不是 2 的幂次方，内核会将其调整为**大于该值且最接近的 2 的幂次方数**。

---

* **示例**

```php
<?php
$server->set([
  'iouring_entries' => 8192
]);
```

---

* **注意**
  * 如果传入的值过大，内核会抛出异常并且终止程序。
  * 当系统安装了`liburing`和编译`Swoole`开启了`--enable-iouring`之后才能使用。

!> `Swoole 6.0`以上可以使用。

### iouring_workers

- 设置 `io_uring` 的工作线程数，默认值是 `CPU 核数 * 4`。

---

* **示例**

```php
<?php
$server->set([
  'iouring_workers' => 128
]);
```

---

* **注意**
  * 如果传入的值过大，内核会抛出异常并且终止程序。
  * 当系统安装了`liburing`和编译`Swoole`开启了`--enable-iouring`之后才能使用。

!> `Swoole 6.0`以上可以使用。

### iouring_flag

- 设置`io_uring`的工作模式，默认值为`SWOOLE_IOURING_DEFAULT`。

---

* **示例**

```php
<?php
$server->set([
  'iouring_flag' => SWOOLE_IOURING_DEFAULT
]);
```

---

* **注意**

  * 如果传入的模式不正确，内核会统一使用`SWOOLE_IOURING_DEFAULT`中断驱动模式。
  * `SWOOLE_IOURING_DEFAULT`，中断驱动模式，可通过系统调用`io_uring_enter`提交`I/O`请求，然后直接检查完成队列状态判断是否完成。
  * `SWOOLE_IOURING_SQPOLL`，内核轮询模式，内核会创建内核线程用于提交和收割`I/O`请求，几乎完全消除用户态内核态上下文切换，性能较好。

!> `Swoole 6.0`以上可以使用。

### reload_async

- 设置异步柔性重启开关。默认值：`true`

---

* **示例**

```php
$server->set([
  'reload_async' => true
]);
```


---

* **注意**

  * 将启用异步安全重启特性，[worker进程](/server/process_thread?id=worker)会等待异步事件完成后再退出。详细信息请参见 [如何正确的重启服务](/question/use?id=swoole如何正确的重启服务)
  * `reload_async` 开启的主要目的是为了保证服务重载时，协程或异步任务能正常结束。
  * 在`4.x`版本中开启 [enable_coroutine](/server/setting?id=enable_coroutine)时，底层会额外增加一个协程数量的检测，当前无任何协程时进程才会退出，开启时即使`reload_async => false`也会强制打开`reload_async`。


### bootstrap

- 多线程模式下的程序入口文件，默认是当前执行的脚本文件名。

---

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_THREAD);
$server->set([
    'worker_num' => 4
    'bootstrap' => __FILE__,
]);
```

---

* **注意**

  * 在[SWOOLE_THREAD](/server/process_thread?id=swoole_thread)模式下，由于 `PHP ZTS（Zend Thread Safe）`模式下的线程是相互隔离的，无法像进程那样通过 `fork()` 直接复制一份独立的内存数据。因此，每个线程都需要重新执行当前脚本文件来初始化自身环境。

!> Swoole版本 >= `v6.0` ， `PHP`为`ZTS`模式，编译`Swoole`时开启了`--enable-swoole-thread`可用

### init_arguments
- 用于设置多线程模式下的共享数据。该配置项需要传入一个回调函数，在服务器启动时会自动执行该函数，其返回值可作为线程间共享的数据对象。。

---

* **示例**

```php
<?php
use Swoole\Server;
use Swoole\Thread\Map;

$server = new Server('127.0.0.1', 9501, SWOOLE_THREAD);
$server->set([
    'init_arguments' => function() {return new Map();}
    'bootstrap' => __FILE__,
]);

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
    $map = Swoole\Thread::getArguments(); // 这里会返回执行回调函数 function() {return new Map();} 的返回值
});
```

---

* **注意**
  * Swoole内置了许多线程安全容器，[并发Map](/thread/map)，[并发List](/thread/arraylist)，[并发队列](/thread/queue)。
  * 请确保回调函数返回的是线程安全的变量，不要返回非线程安全的普通变量。

!> Swoole版本 >= `v6.0` ， `PHP`为`ZTS`模式，编译`Swoole`时开启了`--enable-swoole-thread`可用


### single_thread

- 设置[master进程](/server/process_thread?id=master)中的`reactor`线程数为1。

---

* **示例**

```php
$server->set([
    'single_thread' => true
]);
```

---

* **注意**
  * 在 `PHP ZTS` 下，如果使用 [SWOOLE_PROCESS 模式](/server/process_thread?id=swoole_process)，一定要设置该值为 `true`。

### max_wait_time

- 设置 [worker进程](/server/process_thread?id=worker)收到停止服务通知后最大等待时间，默认值：`3`，超过该时间进程还没退出，该进程会被强制杀死。

---

* **示例**

```php
$server->set([
  'max_wait_time' => 30
]);
```

---

* **注意**

  * 经常会碰到由于`worker`阻塞卡顿导致`worker`无法正常`重启`, 无法满足一些生产场景，例如发布代码热更新需要`reload`进程。所以，Swoole 加入了进程重启超时时间的选项。详细信息请参见 [如何正确的重启服务](/question/use?id=swoole如何正确的重启服务)。
  * **管理进程收到重启、关闭信号后或者达到`max_request`时，管理进程会重起该`worker`进程。分以下几个步骤：**

    * 底层会增加一个(`max_wait_time`)秒的定时器，触发定时器后，检查进程是否依然存在，如果是，会强制杀掉，重新拉一个进程。
    * 需要在`onWorkerStop`回调里面做收尾工作，需要在`max_wait_time`秒内完成收尾。
    * 依次向目标进程发送`SIGTERM`信号，杀掉进程。

### ssl_cert_file / ssl_key_file

- 设置SSL隧道加密。

---

* **示例**

```php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501, SWOOLE_BASE, SWOOLE_TCP | SWOOLE_SSL);
$server->set(array(
  'ssl_cert_file' => __DIR__.'/config/ssl.crt',
  'ssl_key_file' => __DIR__.'/config/ssl.key',
));
```

---

* **注意**

  * 使用 `HTTPS` 服务时，浏览器必须信任所配置的 SSL 证书，否则页面会显示不安全。

  * 使用 `wss`（WebSocket over SSL）时，发起连接的页面也必须通过 `HTTPS` 访问。

  * 浏览器不信任`SSL`证书将无法使用 `wss` 。

  * 文件必须为`PEM`格式，不支持`DER`格式，可使用`openssl`工具进行转换。

  * **`PEM`转`DER`格式**

    ```shell
    openssl x509 -in cert.crt -outform der -out cert.der
    ```

  * **`DER`转`PEM`格式**

    ```shell
    openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
    ```

!> 该配置需要编译`Swoole`时加入`--enable-openssl`选项，但在`Swoole 6.2.0`已经默认支持`openssl`。

### ssl_compress

- 设置是否启用 `SSL/TLS` 压缩。

---

* **示例**

```php
$server->set([
  'ssl_compress' => true
]);
```

### ssl_protocols

- 设置OpenSSL隧道加密的协议。默认值：`0`，支持全部协议。

- 支持的协议类型有`SWOOLE_SSL_TLSv1` ,`SWOOLE_SSL_TLSv1`,`SWOOLE_SSL_TLSv1`,`SWOOLE_SSL_TLSv1`,`SWOOLE_SSL_SSL`,`SWOOLE_SSL_SSLv3`。

---

* **示例**

```php
$server->set(array(
    'ssl_protocols' => SWOOLE_SSL_TLSv1,
));
```

!> Swoole版本 >= `v4.5.4` 可用

### ssl_verify_peer

- 是否验证服务端 SSL 证书的对端证书。默认值：`false`

---

* **示例**

```php
$server->set(array(
  'ssl_client_cert_file' => '/path/to/cert_file'
  'ssl_verify_peer' => true,
));
```

!> 默认关闭，即不验证客户端证书。若开启，必须同时设置 `ssl_client_cert_file` 选项


### ssl_client_cert_file

- 根证书文件路径，用于验证客户端证书（双向 SSL 认证中的 CA 根证书）。

---

* **示例**

```php
$server->set(array(
    'ssl_cert_file'         => __DIR__ . '/config/ssl.crt',
    'ssl_key_file'          => __DIR__ . '/config/ssl.key',
    'ssl_verify_peer'       => true,
    'ssl_allow_self_signed' => true,
    'ssl_client_cert_file'  => __DIR__ . '/config/ca.crt',
));
```

!> `TCP`服务若验证失败，会底层会主动关闭连接。

### ssl_allow_self_signed

- 是否允许自签名证书。默认值：`false`。

---

* **示例**

```php
$server->set([
  'ssl_allow_self_signed' => true
]);
```

### ssl_cafile

- 当设置 `ssl_verify_peer` 为 `true` 时，用来验证远端证书所用到的 `CA` 证书。本选项值为 `CA` 证书在本地文件系统的全路径及文件名。

---

* **示例**

```php
$server->set([
  'ssl_cafile' => '/etc/CA',
]);
```

### ssl_capath

- 如果未设置 `ssl_cafile`，或者 `ssl_cafile` 所指的文件不存在时，会在 `ssl_capath` 所指定的目录搜索适用的证书。该目录必须是已经经过哈希处理的证书目录。

---

* **示例**

```php
$server->set([
  'ssl_capath' => '/etc/capath/',
]);
```

### ssl_verify_depth

- 如果证书链条层次太深，超过了本选项的设定值，则终止验证。

---

* **示例**

```php
$server->set([
  'ssl_verify_depth' => 5
]);
```

### ssl_prefer_server_ciphers

- 决定当客户端（浏览器）和服务器建立加密连接时，由谁来选择使用哪种加密算法（密码套件）。

---

* **示例**

```php
$server->set([
  'ssl_prefer_server_ciphers' => true
]);
```

---

* **注意**

  * `true`表示优先使用服务器端配置的加密套件顺序。
  * `false`优先使用客户端（浏览器）提供的加密套件顺序。

### ssl_ciphers

- 设置 `Openssl` 加密算法，默认值为：`EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH`

---

* **示例**

```php
$server->set([
  'ssl_ciphers' => 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH'
]);
```

* **注意**

  * `ssl_ciphers`为空时，由 `openssl` 自行选择加密算法

### ssl_ecdh_curve

- 专门控制 `ECDH` 密钥交换过程中所使用的椭圆曲线。

---

* **示例**

```php
$server->set([
  'ssl_ecdh_curve' => 'X25519:prime256v1:secp384r1'
]);
```

### ssl_dhparam

- 用于指定 Diffie-Hellman (DH) 密钥交换 所使用的强参数。当服务器使用老式的 DHE 加密套件时，这个指令告诉服务器应该使用哪个“大质数文件”来生成临时的会话密钥，从而保证安全性。

---

* **示例**

```php
$server->set([
  'ssl_dhparam' => ‘/etc/dhparam.pem’
]);
```
### ssl_sni_certs

- 设置 SNI (Server Name Identification) 证书。

---

* **示例**

```php
$server->set([
  'ssl_sni_certs' => [
        'cs.php.net' => [
            'ssl_cert_file' => __DIR__ . '/sni_server_cs_cert.pem',
            'ssl_key_file' => __DIR__ . '/sni_server_cs_key.pem',
        ],
        'uk.php.net' => [
            'ssl_cert_file' =>  __DIR__ . '/sni_server_uk_cert.pem',
            'ssl_key_file' => __DIR__ . '/sni_server_uk_key.pem',
        ],
        'us.php.net' => [
            'ssl_cert_file' => __DIR__ . '/sni_server_us_cert.pem',
            'ssl_key_file' => __DIR__ . '/sni_server_us_key.pem',
        ],
    ]
]);
```


### websocket_compression
- 是否启用 `WebSocket` 帧数据的压缩功能。启用后，仍需配合 `SWOOLE_WEBSOCKET_FLAG_COMPRESS` 标志位，对具体的帧单独进行压缩。

---

* **示例1：单帧压缩（自动压缩）。**

- 当 `websocket_compression` 设置为 `true`，并在发送帧时携带 `SWOOLE_WEBSOCKET_FLAG_COMPRESS`，`Swoole` 底层会自动对该帧数据进行 `zlib` 压缩后发送给客户端。

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;

$server = new Server('127.0.0.1', 9501);
// 开启 WebSocket 压缩支持
$server->set(['websocket_compression' => true]);

$server->on('message', function (Server $server, Frame $frame) {
  // 发送时带上 SWOOLE_WEBSOCKET_FLAG_COMPRESS 标志，底层会自动使用 zlib 压缩该帧数据
  $server->push(
      $frame->fd,
      'Hello Swoole',
      SWOOLE_WEBSOCKET_OPCODE_TEXT,
      SWOOLE_WEBSOCKET_FLAG_FIN | SWOOLE_WEBSOCKET_FLAG_COMPRESS
  );
});

$server->start();
```

* **示例2：连续帧压缩（手动压缩）**

- 当 `websocket_compression` 为 `true` 时，如果需要发送多个连续帧（分片消息），底层不会自动压缩，必须由开发者自行压缩数据并设置 `RSV1` 标志。

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;

$server = new Server('127.0.0.1', 9501);
// 开启 WebSocket 压缩支持
$server->set(['websocket_compression' => true]);

$server->on('message', function (Server $server, Frame $frame) {
  $data1 = bin2hex(random_bytes(10 * 1024));
  $data2 = bin2hex(random_bytes(20 * 2048));
  $data3 = bin2hex(random_bytes(40 * 4096));
  $data4 = bin2hex(random_bytes(40 * 4096));
  
  $context = deflate_init(ZLIB_ENCODING_RAW);
  $server->push($frame->fd, deflate_add($context, $data1, ZLIB_NO_FLUSH), SWOOLE_WEBSOCKET_OPCODE_TEXT, SWOOLE_WEBSOCKET_FLAG_COMPRESS | SWOOLE_WEBSOCKET_FLAG_RSV1);
  $server->push($frame->fd, deflate_add($context, $data2, ZLIB_NO_FLUSH), SWOOLE_WEBSOCKET_OPCODE_CONTINUATION, 0);
  $server->push($frame->fd, deflate_add($context, $data3, ZLIB_NO_FLUSH), SWOOLE_WEBSOCKET_OPCODE_CONTINUATION, 0);
  $server->push($frame->fd, deflate_add($context, $data4, ZLIB_FINISH), SWOOLE_WEBSOCKET_OPCODE_CONTINUATION, SWOOLE_WEBSOCKET_FLAG_FIN);
});

$server->start();
```

---

* **注意**
  * 要求服务端与客户端在握手阶段均明确支持压缩；若任一方不支持，则整个通信流程中的消息均不会自动压缩。
  * `Ping`，`Pong`和关闭帧这三个控制帧不会受到该配置的影响，控制帧不允许压缩。
  * 有关分片消息的处理机制及详细说明，可以查看[Swoole\WebSocket\Server->push()](/server/methods?id=push)。


### websocket_subprotocol
- 在 `WebSocket` 连接建立时，客户端和服务器协商确定双方都支持的消息格式或应用层协议。

---

* **示例**

```php
$server->set([
    'websocket_subprotocol' => 'json',
]);
```

---

* **说明**
  * 设置后握手响应的 HTTP 头会增加 `Sec-WebSocket-Protocol: {$websocket_subprotocol}`。具体使用方法请参考 `WebSocket` 协议相关 `RFC` 文档。
  
  * `WebSocket` 只提供了一个"全双工通信的管道"，但不管管道里传输的数据长什么样（是 JSON、是 XML、还是自定义二进制格式）。`Subprotocol` 就是告诉对方："我发的消息是按某某协议组织的，你能理解吗？"。

### open_websocket_ping_frame
- 该选项用于控制是否由使用者自行处理`websocket`客户端发来的 `Ping` 帧。默认值为 `false`，此时服务器会自动响应一个`Pong` 帧给客户端。

---

* **示例**

```php
$websocket->set([
  'open_websocket_ping_frame' => true
]);
```

---

* **注意**

  * 设置为`true`后，由客户端发送的`Ping`帧不会由底层自动处理，而是会触发[message事件](/server/events?id=message)，由使用者自行处理。

### open_websocket_pong_frame

- 该选项用于控制是否由使用者自行处理`websocket`客户端发来的 `Pong` 帧。默认值为 `false`。

---

* **示例**

```php
$websocket->set([
  'open_websocket_pong_frame' => true
]);
```

---

* **注意**

  * 设置为`true`后，由客户端发送的`Pong`帧不会由底层自动处理，而是会触发[message事件](/server/events?id=message)，由使用者自行处理。

### open_websocket_close_frame

- 该选项用于控制是否由使用者自行处理`websocket`客户端发来的 `关闭` 帧。默认值为 `false`。

---

* **示例**

```php
$websocket->set([
  'open_websocket_close_frame' => true
]);
```

---

* **注意**

  * 设置为`true`后，由客户端发送的`关闭`帧不会由底层自动处理，而是会触发[message事件](/server/events?id=message)，由使用者自行处理。

### http_compression

- 是否开启`HTTP`响应压缩，默认值为：`true`。

---

* **示例**

```php
$http->set([
  'http_compression' => true
]);
```

---

* **注意**
  * 该参数生效的前提是服务器至少支持 `brotli`、`gzip`、`deflate`、`zstd` 中的一种压缩算法。否则，由于缺乏可用的压缩功能，此参数将不会产生任何作用。

### http_compression_level / compression_level / http_gzip_level

- 设置`HTTP`响应压缩等级，范围是 1-9，等级越高压缩后的尺寸越小，但 CPU 消耗更多。默认为 1, 最高为 9。

---

* **示例**

```php
$http->set([
  'http_compression' => true,
  'http_compression_level' => 6
]);
```

### http_compression_min_length / compression_min_length

- 设置`HTTP`响应压缩的最小字节，超过该选项值才开启压缩。默认 20 字节。

---

* **示例**

```php
$http->set([
  'http_compression' => true,
  'http_compression_level' => 6,
  'http_compression_min_length' => 4096
]);
```
### http_compression_types / compression_types

- 设置需要压缩的响应类型。

---

* **示例**

```php
$http->set([
  'http_compression_types' => ['text/html', 'application/json']
]);
```

!> Swoole 版本 >= v4.8.12 可用

### http_parse_cookie

- 设置底层在解析 `HTTP` 报文时是否启用 `Cookies` 解析，默认值为 `true`。若设置为 `false`，则不会对 `Cookies` 进行解析，原始的 `Cookies` 信息将保留在[Swoole\Http\Request->header](/server/swoole_http_request?id=header) 中。

---

* **示例**

```php
$http->set([
  'http_parse_cookie' => true,
]);
```

### http_parse_post

- 设置底层在解析 `HTTP` 报文时是否启用 `POST` 解析，默认值为 `true`。若设置为 `false`，则不会对 `POST` 进行解析。

---

* **示例**

```php
$http->set([
  'http_parse_post' => true,
]);
```

---

* **说明**
  * 设置为 `true` 时自动将 `Content-Type为x-www-form-urlencoded` 的请求包体解析到 [Swoole\Http\Request->post](/server/swoole_http_request?id=post) 中。
  * 设置为 `false` 是可以通过[Swoole\Http\Request->getContent()](/server/swoole_http_request?id=getcontent)获取请求包体。

### http_parse_files


- 设置底层在解析 `HTTP` 报文时是否对用户上传文件进行解析，默认值为 `true`。若设置为 `false`，则不会对用户上传文件进行解析。

---

* **示例**

```php
$http->set([
  'http_parse_files' => true,
]);
```

### http2_header_table_size

- 设置 HTTP/2 协议中 HPACK 压缩动态表的最大大小的参数。

---

* **示例**

```php
$http2->set([
  'http2_header_table_size' => 0x1
]);
```

### http2_enable_push
- 控制 HTTP/2 服务端是否启用 Server Push（服务端推送）功能的开关。

---

* **示例**

```php
$http2->set([
  'http2_enable_push' => 0x2
]);
```

### http2_max_concurrent_streams
- 限制在单个 HTTP/2 连接上同时处于活跃状态的流（Stream）的最大数量。

---

* **示例**

```php
$http2->set([
  'http2_max_concurrent_streams' => 256
]);
```


### http2_init_window_size

- 用于配置 HTTP/2 流量控制中初始窗口大小的参数。

---

* **示例**

```php
$http2->set([
  'http2_init_window_size' => '256K'
]);
```

### http2_max_frame_size
- 配置 HTTP/2 协议中单个帧（Frame）最大有效载荷（Payload）大小的参数。

---

* **示例**

```php
$http2->set([
  'http2_max_frame_size' => '256K'
]);
```

### http2_max_header_list_size

- 配置 HTTP/2 连接中允许接收的最大头部列表大小的参数。

---

* **示例**

```php
$http2->set([
  'http2_max_header_list_size' => '32K'
]);
```

### document_root
- 配置静态文件服务器根目录。

---

* **示例**

```php
$http->set([
  'document_root' => '/home/files/'
  'enable_static_handler' => true
]);
```

- 当你访问`http://127.0.0.1:9501/hello.txt`时，如果`/home/files/`存在这份`hello.txt`文件，就会返回这份文件的内容给客户端。

- 当你访问`http://127.0.0.1:9501/hello/hello.txt`时，如果`/home/files/hello/`存在这份`hello.txt`文件，就会返回这份文件的内容给客户端。

---

* **注意**

  * 设置 `document_root` 并设置 `enable_static_handler` 为 `true` 后，底层收到 `Http` 请求会先判断 `document_root` 路径下是否存在此文件，如果存在会直接发送文件内容给客户端，不再触发 [request](/server/events?id=request) 回调。
  * 使用静态文件处理特性时，应当将动态 `PHP` 代码和静态文件进行隔离，静态文件存放到特定的目录。

!> 这个功能比较简易，只能用于测试，禁止在互联网中使用。

### enable_static_handler

- 启动静态文件服务器。默认为：`false`。

---

* **示例**

```php
$http->set([
  'document_root' => '/home/files/'
  'enable_static_handler' => true
]);
```

!> 这个功能比较简易，只能用于测试，禁止在互联网中使用。

### static_handler_locations

- 设置静态处理器的路径。类型为数组，默认不启用。


---

* **示例**

```php
$http->set([
  'document_root' => '/home/files/'
  'enable_static_handler' => true,
  'static_handler_locations' => ['/static', '/app/images'],
]);
```

---

* **说明**

  * 类似于 `Nginx` 的 `location` 指令，可以指定一个或多个路径为静态路径。只有 `URL` 在指定路径下才会启用静态文件处理器，否则会视为动态请求。
  * `location` 项必须以 `/` 开头。
  * 支持多级路径，如 `/app/images`。
  * 启用 `static_handler_locations` 后，如果请求对应的文件不存在，将直接返回 404 错误。、

!> 这个功能比较简易，只能用于测试，禁止在互联网中使用。

### url_rewrite_rules

- 静态文件服务器URL重写规则的配置采用数组形式。

- 每个规则包含一个键值对，其中键（key）为正则表达式，用于匹配请求路径；值（value）为目标目录路径。

- 当请求路径与某个键的正则表达式匹配成功时，该请求将被重写至对应的目标目录。

---

* **示例**

```php
$http->set([
  'document_root' => '/home/files/'
  'enable_static_handler' => true,
  'url_rewrite_rules' => [
    '~^/view/post/(\d+)$~' => '/static/$1.html',
    '/article/' => '/static/article/'
   ]
]);
```

- 当访问 `http://127.0.0.1:9501/view/post/123` 时，该 URL 会匹配正则表达式 `~^/view/post/(\d+)$~`。匹配成功后，系统会在 `/home/files/static/` 目录下查找与捕获的数字（即 `123`）对应的 `123.html` 文件。如果该文件存在，则将其内容返回给客户端。

!> 这个功能比较简易，只能用于测试，禁止在互联网中使用。

### http_autoindex
- 开启目录浏览功能，默认为：`false`。

- 当访问服务器上的一个目录，且该目录下没有默认的索引文件（如 index.html、index.php）时，服务器会自动生成一个该目录下的文件和子目录列表（目录索引），并以网页形式返回给客户端。类似`Nginx`的`autoindex`指令。

---

* **示例**

```php
$http->set([
  'document_root' => '/home/files/'
  'enable_static_handler' => true,
  'http_autoindex' => true,
]);
```

- 当你直接访问`http://127.0.0.1:9501`时，就会返回整个`/home/files/`的文件路径。

![http_autoindex](../images/http_autoindex.png)

!> 这个功能比较简易，只能用于测试，禁止在互联网中使用。

### http_index_files

- 用于指定当客户端访问一个目录时，默认返回该目录下的哪个文件。类似`Nginx`的`index`指令。


---

* **示例**

```php
$http->set([
  'document_root' => '/home/files/'
  'enable_static_handler' => true,
  'http_index_files' => ['index.php'],
]);
```

- 访问`http://127.0.0.1:9501/`，如果`/home/files/`存在`index.php`，会将该文件返回给客户端。

!> 这个功能比较简易，只能用于测试，禁止在互联网中使用。

### log_file
### log_level
### log_date_format
### log_date_with_microseconds
### log_rotation

### open_http_protocol
- 启用`HTTP`协议报文解析。

### open_websocket_protocol
- 启用`WebSocket`协议报文解析。

### open_http2_protocol
- 启用`HTTP2`协议报文解析。

### open_mqtt_protocol
- 启用`mqtt`协议报文解析。

### open_redis_protocol
- 启用`redis`协议报文解析。

### socket_dns_timeout
- 控制域名解析超时时间，单位为秒。

---

* **示例**

```php
$server->set([
  'socket_dns_timeout' => 10
]);
```
### socket_connect_timeout
- 控制 TCP 连接建立阶段的超时时间。它指的是客户端从发起 connect() 系统调用开始，到成功与服务器建立 TCP 三次握手连接为止的最大等待时间。单位为秒。

---

* **示例**

```php
$server->set([
  'socket_connect_timeout' => 10
]);
```

### socket_timeout
- 
### socket_write_timeout / socket_send_timeout
- 控制客户端向服务器发送数据的最大允许时间。单位为秒。

---

* **示例**

```php
$server->set([
  'socket_write_timeout' => 10
]);
```

### socket_read_timeout / socket_recv_timeout
- 控制的是客户端等待接收数据的最大允许时间。它监控的是从发送完请求后，到完全接收完响应数据之间的整个过程。单位为秒。

---

* **示例**

```php
$server->set([
  'socket_read_timeout' => 10
]);
```

### socket_buffer_size
- 配置进程间通信的缓存区长度。默认值：2M。

- 用于设置 [worker进程](/server/process_thread?id=worker) 和 [master进程](/server/process_thread?id=master) 进程间通讯 buffer 总的大小，参考 [SWOOLE_PROCESS模式](/server/process_thread?id=SWOOLE_PROCESS) 。

---

* **示例**

```php
$server->set([
  'socket_buffer_size' => 128 * 1024 *1024
]);
```

### open_tcp_nodelay
- 关闭 `Nagle` 算法，让小数据包能立刻被发送出去，而不是等待合并。默认值：`false`。

---

* **示例**

```php
$server->set([
  'open_tcp_nodelay' => false
]);
```

### open_tcp_keepalive
- 开启 TCP 的 KeepAlive 机制，让操作系统内核自动检测连接是否仍然有效，并自动清理已失效的连接。默认值：`false`。

---

* **示例**

```php
$server->set([
  'open_tcp_keepalive' => false
]);
```

* **注意**
  * 需要配合`tcp_keepidle`，`tcp_keepinterval`和`tcp_keepcount`一起使用，

### tcp_keepidle
- TCP 连接在空闲多长时间后，系统内核才开始发送第一个 `KeepAlive` 探测包，检测这个连接是否还活着。单位为秒。

---

* **示例**

```php
$server->set([
  'tcp_keepidle' => 5
]);
```

### tcp_keepinterval
- TCP KeepAlive 探测包的发送间隔。它决定了在第一次探测没有收到响应后，每隔多久重新发送一次探测包。单位为秒。

---

* **示例**

```php
$server->set([
  'tcp_keepinterval' => 5
]);
```

### tcp_keepcount
- TCP KeepAlive 探测失败后，关闭连接前的最大重试次数。它决定了在收到探测包的响应之前，系统愿意尝试多少次。

---

* **示例**

```php
$server->set([
  'tcp_keepinterval' => 100
]);
```

### tcp_defer_accept

- `tcp_defer_accept` 参数用于控制服务器在新建 TCP 连接时，是否延迟分配套接字和应用层资源。默认值为 `0`，表示采用标准行为：三次握手完成后，服务器立即分配资源并唤醒应用程序处理。

- 标准行为存在一个效率问题：若客户端建立连接后长时间不发送数据，服务器仍需维持该连接、占用资源，这就是“全连接攻击”的典型场景。

- 当设置了 `tcp_defer_accept`（单位：秒）后，服务器仅在收到客户端实际数据时才会分配完整资源。如果超过该时间仍未收到任何数据，连接将被主动丢弃，从而避免资源空耗。

---

* **示例**

```php
$server->set([
  'tcp_defer_accept' => 5
]);
```

### tcp_user_timeout

- 当发送方发送数据后，如果迟迟收不到对方的 `ACK` 确认，TCP 会按指数退避算法重传数据。

- 累计重传时间一旦超过`tcp_user_timeout`的值，连接会被关闭。默认值为：0。单位为毫秒。

---

* **示例**

```php
$server->set([
  'tcp_user_timeout' => 5000 // 5秒
]);
```

!> Swoole 版本 >= `v4.5.3-alpha` 可用。

### tcp_fastopen

- 开启 TCP 快速握手特性，它允许在三次握手过程中就携带应用数据，从而减少一个 RTT（往返时间）的延迟。默认值为：`false`。

---

* **示例**

```php
$server->set([
  'tcp_fastopen' => true
]);
```

### debug_mode
### trace_flags
### enable_signalfd
### enable_kqueue
### display_errors
### print_backtrace_on_error
### dns_server
### enable_deadlock_check
### enable_preemptive_scheduler
### c_stack_size
### name_resolver
### chroot
### user
### group
### daemonize
### pid_file
### max_queued_bytes
### send_timeout
### dispatch_mode
### send_yield
### dispatch_func
### discard_timeout_request
### enable_unsafe_event
### enable_delay_receive
### task_use_object/task_object
### event_object
### task_ipc_mode
### task_tmpdir
### task_max_request_grace
### start_session_id
### max_request_grace
### open_cpu_affinity
### cpu_affinity_ignore
### upload_tmp_dir
### input_buffer_size/buffer_input_size
### output_buffer_size/buffer_output_size
### message_queue_key
### backlog
### buffer_high_watermark
### buffer_low_watermark





### debug_mode

?> 设置日志模式为`debug`调试模式，只有编译时开启了`--enable-debug`才有作用。

```php
$server->set([
  'debug_mode' => true
])
```

### trace_flags

?> 设置跟踪日志的标签，仅打印部分跟踪日志。`trace_flags` 支持使用 `|` 或操作符设置多个跟踪项。，只有编译时开启了`--enable-trace-log`才有作用。

底层支持以下跟踪项，可使用`SWOOLE_TRACE_ALL`表示跟踪所有项目：

* `SWOOLE_TRACE_SERVER`
* `SWOOLE_TRACE_CLIENT`
* `SWOOLE_TRACE_BUFFER`
* `SWOOLE_TRACE_CONN`
* `SWOOLE_TRACE_EVENT`
* `SWOOLE_TRACE_WORKER`
* `SWOOLE_TRACE_REACTOR`
* `SWOOLE_TRACE_PHP`
* `SWOOLE_TRACE_HTTP2`
* `SWOOLE_TRACE_EOF_PROTOCOL`
* `SWOOLE_TRACE_LENGTH_PROTOCOL`
* `SWOOLE_TRACE_CLOSE`
* `SWOOLE_TRACE_HTTP_CLIENT`
* `SWOOLE_TRACE_COROUTINE`
* `SWOOLE_TRACE_REDIS_CLIENT`
* `SWOOLE_TRACE_MYSQL_CLIENT`
* `SWOOLE_TRACE_AIO`
* `SWOOLE_TRACE_ALL`

### log_file

?> **指定`Swoole`错误日志文件**

?> 在`Swoole`运行期发生的异常信息会记录到这个文件中，默认会打印到屏幕。  
开启守护进程模式后`(daemonize => true)`，标准输出将会被重定向到`log_file`。在PHP代码中`echo/var_dump/print`等打印到屏幕的内容会写入到`log_file`文件。

  * **提示**

    * `log_file`中的日志仅仅是做运行时错误记录，没有长久存储的必要。

    * **日志标号**

      ?> 在日志信息中，进程ID前会加一些标号，表示日志产生的线程/进程类型。

        * `#` Master进程
        * `$` Manager进程
        * `*` Worker进程
        * `^` Task进程

    * **重新打开日志文件**

      ?> 在服务器程序运行期间日志文件被`mv`移动或`unlink`删除后，日志信息将无法正常写入，这时可以向`Server`发送`SIGRTMIN`信号实现重新打开日志文件。

      * 仅支持`Linux`平台
      * 不支持[UserProcess](/server/methods?id=addProcess)进程

  * **注意**

    !> `log_file`不会自动切分文件，所以需要定期清理此文件。观察`log_file`的输出，可以得到服务器的各类异常信息和警告。

### log_level

?> **设置`Server`错误日志打印的等级，范围是`0-6`。低于`log_level`设置的日志信息不会抛出。**【默认值：`SWOOLE_LOG_INFO`】

对应级别常量参考[日志等级](/consts?id=日志等级)

  * **注意**

    !> `SWOOLE_LOG_DEBUG`和`SWOOLE_LOG_TRACE`仅在编译为[--enable-debug-log](/environment?id=debug参数)和[--enable-trace-log](/environment?id=debug参数)版本时可用；  
    在开启`daemonize`守护进程时，底层将把程序中的所有打印屏幕的输出内容写入到[log_file](/server/setting?id=log_file)，这部分内容不受`log_level`控制。

### log_date_format

?> **设置`Server`日志时间格式**，格式参考 [strftime](https://www.php.net/manual/zh/function.strftime.php) 的`format`

```php
$server->set([
    'log_date_format' => '%Y-%m-%d %H:%M:%S',
]);
```

### log_date_with_microseconds

?> **设置`Server`日志精度，是否带微秒**【默认值：`false`】

### log_rotation

?> **设置`Server`日志分割**【默认值：`SWOOLE_LOG_ROTATION_SINGLE`】

| 常量                             | 说明   | 版本信息 |
| -------------------------------- | ------ | -------- |
| SWOOLE_LOG_ROTATION_SINGLE       | 不启用 | -        |
| SWOOLE_LOG_ROTATION_MONTHLY      | 每月   | v4.5.8   |
| SWOOLE_LOG_ROTATION_DAILY        | 每日   | v4.5.2   |
| SWOOLE_LOG_ROTATION_HOURLY       | 每小时 | v4.5.8   |
| SWOOLE_LOG_ROTATION_EVERY_MINUTE | 每分钟 | v4.5.8   |

### display_errors

?> 开启 / 关闭 `Swoole` 错误信息。

```php
$server->set([
  'display_errors' => true
])
```

### dns_server

?> 设置`dns`查询的`ip`地址。

### socket_dns_timeout

?> 域名解析超时时间，如果在服务端启用协程客户端，该参数可以控制客户端的域名解析超时时间，单位为秒。

### socket_connect_timeout

?> 客户端连接超时时间，如果在服务端启用协程客户端，该参数可以控制客户端的连接超时时间，单位为秒。

### socket_write_timeout / socket_send_timeout

?> 客户端写超时时间，如果在服务端启用协程客户端，该参数可以控制客户端的写超时时间，单位为秒。   
该配置也能用于控制`协程化`之后的`shell_exec`或者[Swoole\Coroutine\System::exec()](/coroutine/system?id=exec)的执行超时时间。   

### socket_read_timeout / socket_recv_timeout

?> 客户端读超时时间，如果在服务端启用协程客户端，该参数可以控制客户端的读超时时间，单位为秒。

### max_coroutine / max_coro_num :id=max_coroutine

?> **设置当前工作进程最大协程数量。**【默认值：`100000`，Swoole版本小于`v4.4.0-beta` 时默认值为`3000`】

?> 超过`max_coroutine`底层将无法创建新的协程，服务端的Swoole会抛出`exceed max number of coroutine`错误，`TCP Server`会直接关闭连接，`Http Server`会返回Http的503状态码。

?> 在`Server`程序中实际最大可创建协程数量等于 `worker_num * max_coroutine`，task进程和UserProcess进程的协程数量单独计算。

```php
$server->set(array(
    'max_coroutine' => 3000,
));
```

### enable_deadlock_check

?> 打开协程死锁检测。

```php
$server->set([
  'enable_deadlock_check' => true
]);
```

### enable_preemptive_scheduler

?> 设置打开协程抢占式调度，避免其中一个协程执行时间过长导致其他协程饿死，协程最大执行时间为`10ms`。

```php
$server->set([
  'enable_preemptive_scheduler' => true
]);
```

### c_stack_size / stack_size

?> 设置单个协程初始 C 栈的内存尺寸，默认为 2M。

### aio_core_worker_num

?> 设置`AIO`最小工作线程数，默认值为`cpu`核数。

### aio_worker_num 

?> 设置`AIO`最大工作线程数，默认值为`cpu`核数 * 8。

### aio_max_wait_time

?> 工作线程等待任务最大时间，单位为秒。

### aio_max_idle_time

?> 工作线程最大空闲时间，单位为秒。

### iouring_entries

?> 设置`io_uring`的队列大小，默认为`8192`，如果传入的值不是`2的次方数`，内核会修改为最接近的，大于该值的`2的次方数`。

!> 如果传入的值过大，内核会抛出异常并且终止程序。

!> 当系统安装了`liburing`和编译`Swoole`开启了`--enable-iouring`之后才能使用。

### iouring_workers

?> 设置`io_uring`的工作线程数，默认值是`CPU 核数 * 4`。

!> 如果传入的值过大，内核会抛出异常并且终止程序。

!> 当系统安装了`liburing`和编译`Swoole`开启了`--enable-iouring`之后才能使用。

### iouring_flag

?> 设置`io_uring`的工作模式，默认值为`SWOOLE_IOURING_DEFAULT`。

- `SWOOLE_IOURING_DEFAULT`，中断驱动模式，可通过系统调用`io_uring_enter`提交`I/O`请求，然后直接检查完成队列状态判断是否完成。
- `SWOOLE_IOURING_SQPOLL`，内核轮询模式，内核会创建内核线程用于提交和收割`I/O`请求，几乎完全消除用户态内核态上下文切换，性能较好。

!> 如果传入的模式不正确，内核会统一使用`SWOOLE_IOURING_DEFAULT`中断驱动模式。

### reactor_num

?> **设置启动的 [Reactor](/learn?id=reactor线程) 线程数。**【默认值：`CPU`核数】

?> 通过此参数来调节主进程内事件处理线程的数量，以充分利用多核。默认会启用`CPU`核数相同的数量。  
`Reactor`线程是可以利用多核，如：机器有`128`核，那么底层会启动`128`线程。  
每个线程能都会维持一个[EventLoop](/learn?id=什么是eventloop)。线程之间是无锁的，指令可以被`128`核`CPU`并行执行。  
考虑到操作系统调度存在一定程度的性能损失，可以设置为CPU核数*2，以便最大化利用CPU的每一个核。

  * **提示**

    * `reactor_num`建议设置为`CPU`核数的`1-4`倍
    * `reactor_num`最大不得超过 [swoole_cpu_num()](/functions?id=swoole_cpu_num) * 4

  * **注意**

  !> -`reactor_num`必须小于或等于`worker_num` ；  
-如果设置的`reactor_num`大于`worker_num`，会自动调整使`reactor_num`等于`worker_num` ；  
-在超过`8`核的机器上`reactor_num`默认设置为`8`。
	
### worker_num

?> **设置启动的`Worker`进程数。**【默认值：`CPU`核数】

?> 如`1`个请求耗时`100ms`，要提供`1000QPS`的处理能力，那必须配置`100`个进程或更多。  
但开的进程越多，占用的内存就会大大增加，而且进程间切换的开销就会越来越大。所以这里适当即可。不要配置过大。

  * **提示**

    * 如果业务代码是全[异步IO](/learn?id=同步io异步io)的，这里设置为`CPU`核数的`1-4`倍最合理
    * 如果业务代码为[同步IO](/learn?id=同步io异步io)，需要根据请求响应时间和系统负载来调整，例如：`100-500`
    * 默认设置为[swoole_cpu_num()](/functions?id=swoole_cpu_num)，最大不得超过[swoole_cpu_num()](/functions?id=swoole_cpu_num) * 1000
    * 假设每个进程占用`40M`内存，`100`个进程就需要占用`4G`内存。

### max_request

?> **设置`worker`进程的最大任务数。**【默认值：`0` 即不会退出进程】

?> 一个`worker`进程在处理完超过此数值的任务后将自动退出，进程退出后会释放所有内存和资源

!> 这个参数的主要作用是解决由于程序编码不规范导致的PHP进程内存泄露问题。PHP应用程序有缓慢的内存泄漏，但无法定位到具体原因、无法解决，可以通过设置`max_request`临时解决，需要找到内存泄漏的代码并修复，而不是通过此方案，可以使用Swoole Tracker发现泄漏的代码。

  * **提示**

    * 达到max_request不一定马上关闭进程，参考[max_wait_time](/server/setting?id=max_wait_time)。
    * [SWOOLE_BASE](/learn?id=swoole_base)下，达到max_request重启进程会导致客户端连接断开。

  !> 当`worker`进程内发生致命错误或者人工执行`exit`时，进程会自动退出。`master`进程会重新启动一个新的`worker`进程来继续处理请求

### max_conn / max_connection

?> **服务器程序，最大允许的连接数。**【默认值：`ulimit -n`】

?> 如`max_connection => 10000`, 此参数用来设置`Server`最大允许维持多少个`TCP`连接。超过此数量后，新进入的连接将被拒绝。

  * **提示**

    * **默认设置**

      * 应用层未设置`max_connection`，底层将使用`ulimit -n`的值作为缺省设置
      * 在`4.2.9`或更高版本，当底层检测到`ulimit -n`超过`100000`时将默认设置为`100000`，原因是某些系统设置了`ulimit -n`为`100万`，需要分配大量内存，导致启动失败

    * **最大上限**

      * 请勿设置`max_connection`超过`1M`

    * **最小设置**
    
      * 此选项设置过小底层会抛出错误，并设置为`ulimit -n`的值。
      * 最小值为`(worker_num + task_worker_num) * 2 + 32`

    ```shell
    serv->max_connection is too small.
    ```

    * **内存占用**

      * `max_connection`参数不要调整的过大，根据机器内存的实际情况来设置。`Swoole`会根据此数值一次性分配一块大内存来保存`Connection`信息，一个`TCP`连接的`Connection`信息，需要占用`224`字节。

  * **注意**

  !> `max_connection`最大不得超过操作系统`ulimit -n`的值，否则会报一条警告信息，并重置为`ulimit -n`的值

  ```shell
  WARN swServer_start_check: serv->max_conn is exceed the maximum value[100000].

  WARNING set_max_connection: max_connection is exceed the maximum value, it's reset to 10240
  ```

### task_worker_num

?> **配置 [Task进程](/learn?id=taskworker进程)的数量。**

?> 配置此参数后将会启用`task`功能。所以`Server`务必要注册[onTask](/server/events?id=ontask)、[onFinish](/server/events?id=onfinish) 2 个事件回调函数。如果没有注册，服务器程序将无法启动。

  * **提示**

    *  [Task进程](/learn?id=taskworker进程)是同步阻塞的

    * 最大值不得超过[swoole_cpu_num()](/functions?id=swoole_cpu_num) * 1000
    
    * **计算方法**
      * 单个`task`的处理耗时，如`100ms`，那一个进程1秒就可以处理`1/0.1=10`个task
      * `task`投递的速度，如每秒产生`2000`个`task`
      * `2000/10=200`，需要设置`task_worker_num => 200`，启用`200`个Task进程

  * **注意**

    !> - [Task进程](/learn?id=taskworker进程)内不能使用`Swoole\Server->task`方法

### task_ipc_mode

?> **设置 [Task进程](/learn?id=taskworker进程)与`Worker`进程之间通信的方式。**【默认值：`1`】 
 
?> 请先阅读[Swoole下的IPC通讯](/learn?id=什么是IPC)。

模式 | 作用
---|---
1 | 使用`Unix Socket`通信【默认模式】
2 | 使用`sysvmsg`消息队列通信
3 | 使用`sysvmsg`消息队列通信，并设置为争抢模式

  * **提示**

    * **模式`1`**
      * 使用模式`1`时，支持定向投递，可在[task](/server/methods?id=task)和[taskwait](/server/methods?id=taskwait)方法中使用`dst_worker_id`，指定目标 `Task进程`。
      * `dst_worker_id`设置为`-1`时，底层会判断每个 [Task进程](/learn?id=taskworker进程)的状态，向当前状态为空闲的进程投递任务。

    * **模式`2`、`3`**
      * 消息队列模式使用操作系统提供的内存队列存储数据，未指定 `mssage_queue_key` 消息队列`Key`，将使用私有队列，在`Server`程序终止后会删除消息队列。
      * 指定消息队列`Key`后`Server`程序终止后，消息队列中的数据不会删除，因此进程重启后仍然能取到数据
      * 可使用`ipcrm -q`消息队列`ID`手动删除消息队列数据
      * `模式2`和`模式3`的不同之处是，`模式2`支持定向投递，`$serv->task($data, $task_worker_id)` 可以指定投递到哪个 [task进程](/learn?id=taskworker进程)。`模式3`是完全争抢模式， [task进程](/learn?id=taskworker进程)会争抢队列，将无法使用定向投递，`task/taskwait`将无法指定目标进程`ID`，即使指定了`$task_worker_id`，在`模式3`下也是无效的。

  * **注意**

    !> -`模式3`会影响[sendMessage](/server/methods?id=sendMessage)方法，使[sendMessage](/server/methods?id=sendMessage)发送的消息会随机被某一个 [task进程](/learn?id=taskworker进程)获取。  
    -使用消息队列通信，如果 `Task进程` 处理能力低于投递速度，可能会引起`Worker`进程阻塞。  
    -使用消息队列通信后task进程无法支持协程(开启[task_enable_coroutine](/server/setting?id=task_enable_coroutine))。  

### task_max_request

?> **设置 [task进程](/learn?id=taskworker进程)的最大任务数。**【默认值：`0`】

设置task进程的最大任务数。一个task进程在处理完超过此数值的任务后将自动退出。这个参数是为了防止PHP进程内存溢出。如果不希望进程自动退出可以设置为0。

### task_tmpdir

?> **设置task的数据临时目录。**【默认值：Linux `/tmp` 目录】

?> 在`Server`中，如果投递的数据超过`8180`字节，将启用临时文件来保存数据。这里的`task_tmpdir`就是用来设置临时文件保存的位置。

  * **提示**

    * 底层默认会使用`/tmp`目录存储`task`数据，如果你的`Linux`内核版本过低，`/tmp`目录不是内存文件系统，可以设置为 `/dev/shm/`
    * `task_tmpdir`目录不存在，底层会尝试自动创建

  * **注意**

    !> -创建失败时，`Server->start`会失败



### task_use_object/task_object :id=task_use_object

?> **使用面向对象风格的Task回调格式。**【默认值：`false`】

?> 设置为`true`时，[onTask](/server/events?id=ontask)回调将变成对象模式。

  * **示例**

```php
<?php

$server = new Swoole\Server('127.0.0.1', 9501);
$server->set([
    'worker_num'      => 1,
    'task_worker_num' => 3,
    'task_use_object' => true,
//    'task_object' => true, // v4.6.0版本增加的别名
]);
$server->on('receive', function (Swoole\Server $server, $fd, $tid, $data) {
    $server->task(['fd' => $fd,]);
});
$server->on('Task', function (Swoole\Server $server, Swoole\Server\Task $task) {
    //此处$task是Swoole\Server\Task对象
    $server->send($task->data['fd'], json_encode($server->stats()));
});
$server->start();
```

### dispatch_mode

?> **数据包分发策略。**【默认值：`2`】

模式值 | 模式 | 作用
---|---|---
1 | 轮循模式 | 收到会轮循分配给每一个`Worker`进程
2 | 固定模式 | 根据连接的文件描述符分配`Worker`。这样可以保证同一个连接发来的数据只会被同一个`Worker`处理
3 | 抢占模式 | 主进程会根据`Worker`的忙闲状态选择投递，只会投递给处于闲置状态的`Worker`
4 | IP分配 | 根据客户端`IP`进行取模`hash`，分配给一个固定的`Worker`进程。<br>可以保证同一个来源IP的连接数据总会被分配到同一个`Worker`进程。算法为 `inet_addr_mod(ClientIP, worker_num)`
5 | UID分配 | 需要用户代码中调用 [Server->bind()](/server/methods?id=bind) 将一个连接绑定`1`个`uid`。然后底层根据`UID`的值分配到不同的`Worker`进程。<br>算法为 `UID % worker_num`，如果需要使用字符串作为`UID`，可以使用`crc32(UID_STRING)`
7 | stream模式 | 空闲的`Worker`会`accept`连接，并接受[Reactor](/learn?id=reactor线程)的新请求

  * **提示**

    * **使用建议**
    
      * 无状态`Server`可以使用`1`或`3`，同步阻塞`Server`使用`3`，异步非阻塞`Server`使用`1`
      * 有状态使用`2`、`4`、`5`
      
    * **UDP协议**

      * `dispatch_mode=2/4/5`时为固定分配，底层使用客户端`IP`取模散列到不同的`Worker`进程
      * `dispatch_mode=1/3`时随机分配到不同的`Worker`进程
      * `inet_addr_mod`函数

```
    function inet_addr_mod($ip, $worker_num) {
        $ip_parts = explode('.', $ip);
        if (count($ip_parts) != 4) {
            return false;
        }
        $ip_parts = array_reverse($ip_parts);
    
        $ip_long = 0;
        foreach ($ip_parts as $part) {
            $ip_long <<= 8;
            $ip_long |= (int) $part;
        }
    
        return $ip_long % $worker_num;
    }
```
  * **Base模式**
    * `dispatch_mode`配置在 [SWOOLE_BASE](/learn?id=swoole_base) 模式是无效的，因为`BASE`不存在投递任务，当收到客户端发来的数据后会立即在当前线程/进程回调[onReceive](/server/events?id=onreceive)，不需要投递`Worker`进程。

  * **注意**

    !> -`dispatch_mode=1/3`时，底层会屏蔽`onConnect/onClose`事件，原因是这2种模式下无法保证`onConnect/onClose/onReceive`的顺序；  
    -非请求响应式的服务器程序，请不要使用模式`1`或`3`。例如：http服务就是响应式的，可以使用`1`或`3`，有TCP长连接状态的就不能使用`1`或`3`。

### dispatch_func

?> 设置`dispatch`函数，`Swoole`底层内置了`6`种[dispatch_mode](/server/setting?id=dispatch_mode)，如果仍然无法满足需求。可以使用编写`C++`函数或`PHP`函数，实现`dispatch`逻辑。

  * **使用方法**

```php
$server->set(array(
  'dispatch_func' => 'my_dispatch_function',
));
```

  * **提示**

    * 设置`dispatch_func`后底层会自动忽略`dispatch_mode`配置
    * `dispatch_func`对应的函数不存在，底层将抛出致命错误
    * 如果需要`dispatch`一个超过8K的包，`dispatch_func`只能获取到 `0-8180` 字节的内容

  * **编写PHP函数**

    ?> 由于`ZendVM`无法支持多线程环境，即使设置了多个[Reactor](/learn?id=reactor线程)线程，同一时间只能执行一个`dispatch_func`。因此底层在执行此PHP函数时会进行加锁操作，可能会存在锁的争抢问题。请勿在`dispatch_func`中执行任何阻塞操作，否则会导致`Reactor`线程组停止工作。

    ```php
    $server->set(array(
        'dispatch_func' => function ($server, $fd, $type, $data) {
            var_dump($fd, $type, $data);
            return intval($data[0]);
        },
    ));
    ```

    * `$fd`为客户端连接的唯一标识符，可使用`Server::getClientInfo`获取连接信息
    * `$type`数据的类型，`0`表示来自客户端的数据发送，`4`表示客户端连接建立，`3`表示客户端连接关闭
    * `$data`数据内容，需要注意：如果启用了`HTTP`、`EOF`、`Length`等协议处理参数后，底层会进行包的拼接。但在`dispatch_func`函数中只能传入数据包的前8K内容，不能得到完整的包内容。
    * **必须**返回一个`0 - (server->worker_num - 1)`的数字，表示数据包投递的目标工作进程`ID`
    * 小于`0`或大于等于`server->worker_num`为异常目标`ID`，`dispatch`的数据将会被丢弃

  * **编写C++函数**

    **在其他PHP扩展中，使用swoole_add_function注册长度函数到Swoole引擎中。**

    ?> C++函数调用时底层不会加锁，需要调用方自行保证线程安全性

    ```c++
    int dispatch_function(swServer *serv, swConnection *conn, swEventData *data);

    int dispatch_function(swServer *serv, swConnection *conn, swEventData *data)
    {
        printf("cpp, type=%d, size=%d\n", data->info.type, data->info.len);
        return data->info.len % serv->worker_num;
    }

    int register_dispatch_function(swModule *module)
    {
        swoole_add_function("my_dispatch_function", (void *) dispatch_function);
    }
    ```

    * `dispatch`函数必须返回投递的目标`worker`进程`id`
    * 返回的`worker_id`不得超过`server->worker_num`，否则底层会抛出段错误
    * 返回负数`（return -1）`表示丢弃此数据包
    * `data`可以读取到事件的类型和长度
    * `conn`是连接的信息，如果是`UDP`数据包，`conn`为`NULL`

  * **注意**

    !> -`dispatch_func`仅在[SWOOLE_PROCESS](/learn?id=swoole_process)模式下有效，[UDP/TCP/UnixSocket](/server/methods?id=__construct)类型的服务器均有效  
    -返回的`worker_id`不得超过`server->worker_num`，否则底层会抛出段错误

### message_queue_key

?> **设置消息队列的`KEY`。**【默认值：`ftok($php_script_file, 1)`】

?> 仅在[task_ipc_mode](/server/setting?id=task_ipc_mode) = 2/3时使用。设置的`Key`仅作为`Task`任务队列的`KEY`，参考[Swoole下的IPC通讯](/learn?id=什么是IPC)。

?> `task`队列在`server`结束后不会销毁，重新启动程序后， [task进程](/learn?id=taskworker进程)仍然会接着处理队列中的任务。如果不希望程序重新启动后执行旧的`Task`任务。可以手动删除此消息队列。

```shell
ipcs -q 
ipcrm -Q [msgkey]
```

### daemonize

?> **守护进程化**【默认值：`false`】

?> 设置`daemonize => true`时，程序将转入后台作为守护进程运行。长时间运行的服务器端程序必须启用此项。  
如果不启用守护进程，当ssh终端退出后，程序将被终止运行。

  * **提示**

    * 启用守护进程后，标准输入和输出会被重定向到 `log_file`
    * 如果未设置`log_file`，将重定向到 `/dev/null`，所有打印屏幕的信息都会被丢弃
    * 启用守护进程后，`CWD`（当前目录）环境变量的值会发生变更，相对路径的文件读写会出错。`PHP`程序中必须使用绝对路径

    * **systemd**

      * 使用`systemd`或者`supervisord`管理`Swoole`服务时，请勿设置`daemonize => true`。主要原因是`systemd`的机制与`init`不同。`init`进程的`PID`为`1`，程序使用`daemonize`后，会脱离终端，最终被`init`进程托管，与`init`关系变为父子进程关系。
      * 但`systemd`是启动了一个单独的后台进程，自行`fork`管理其他服务进程，因此不需要`daemonize`，反而使用了`daemonize => true`会使得`Swoole`程序与该管理进程失去父子进程关系。

### backlog

?> **设置`Listen`队列长度**

?> 如`backlog => 128`，此参数将决定最多同时有多少个等待`accept`的连接。

  * **关于`TCP`的`backlog`**

    ?> `TCP`有三次握手的过程，客户端 `syn=>服务端` `syn+ack=>客户端` `ack`，当服务器收到客户端的`ack`后会将连接放到一个叫做`accept queue`的队列里面（注1），  
    队列的大小由`backlog`参数和配置`somaxconn` 的最小值决定，可以通过`ss -lt`命令查看最终的`accept queue`队列大小，`Swoole`的主进程调用`accept`（注2）  
    从`accept queue`里面取走。 当`accept queue`满了之后连接有可能成功（注4），  
    也有可能失败，失败后客户端的表现就是连接被重置（注3）  
    或者连接超时，而服务端会记录失败的记录，可以通过 `netstat -s|grep 'times the listen queue of a socket overflowed` 来查看日志。如果出现了上述现象，你就应该调大该值了。 幸运的是`Swoole`的SWOOLE_PROCESS模式与`PHP-FPM/Apache`等软件不同，并不依赖`backlog`来解决连接排队的问题。所以基本不会遇到上述现象。

    * 注1:`linux2.2`之后握手过程分为`syn queue`和`accept queue`两个队列, `syn queue`长度由`tcp_max_syn_backlog`决定。
    * 注2:高版本内核调用的是`accept4`，为了节省一次`set no block`系统调用。
    * 注3:客户端收到`syn+ack`包就认为连接成功了，实际上服务端还处于半连接状态，有可能发送`rst`包给客户端，客户端的表现就是`Connection reset by peer`。
    * 注4:成功是通过TCP的重传机制，相关的配置有`tcp_synack_retries`和`tcp_abort_on_overflow`。

### open_tcp_keepalive

?> 在`TCP`中有一个`Keep-Alive`的机制可以检测死连接，应用层如果对于死链接周期不敏感或者没有实现心跳机制，可以使用操作系统提供的`keepalive`机制来踢掉死链接。
在 [Server->set()](/server/methods?id=set) 配置中增加`open_tcp_keepalive => true`表示启用`TCP keepalive`。
另外，有`3`个选项可以对`keepalive`的细节进行调整。

  * **选项**

     * **tcp_keepidle**

        单位秒，连接在`n`秒内没有数据请求，将开始对此连接进行探测。

     * **tcp_keepcount**

        探测的次数，超过次数后将`close`此连接。

     * **tcp_keepinterval**

        探测的间隔时间，单位秒。

  * **示例**

```php
$serv = new Swoole\Server("192.168.2.194", 6666, SWOOLE_PROCESS);
$serv->set(array(
    'worker_num' => 1,
    'open_tcp_keepalive' => true,
    'tcp_keepidle' => 4, //4s没有数据传输就进行检测
    'tcp_keepinterval' => 1, //1s探测一次
    'tcp_keepcount' => 5, //探测的次数，超过5次后还没回包close此连接
));

$serv->on('connect', function ($serv, $fd) {
    var_dump("Client:Connect $fd");
});

$serv->on('receive', function ($serv, $fd, $reactor_id, $data) {
    var_dump($data);
});

$serv->on('close', function ($serv, $fd) {
  var_dump("close fd $fd");
});

$serv->start();
```

### heartbeat_check_interval

?> **启用心跳检测**【默认值：`false`】

?> 此选项表示每隔多久轮循一次，单位为秒。如 `heartbeat_check_interval => 60`，表示每`60`秒，遍历所有连接，如果该连接在`120`秒内（`heartbeat_idle_time`未设置时默认为`interval`的两倍），没有向服务器发送任何数据，此连接将被强制关闭。若未配置，则不会启用心跳, 该配置默认关闭。

  * **提示**
    * `Server`并不会主动向客户端发送心跳包，而是被动等待客户端发送心跳。服务器端的`heartbeat_check`仅仅是检测连接上一次发送数据的时间，如果超过限制，将切断连接。
    * 被心跳检测切断的连接依然会触发[onClose](/server/events?id=onclose)事件回调

  * **注意**

    !> `heartbeat_check`仅支持`TCP`连接

### heartbeat_idle_time

?> **连接最大允许空闲的时间**

?> 需要与`heartbeat_check_interval`配合使用

```php
array(
    'heartbeat_idle_time'      => 600, // 表示一个连接如果600秒内未向服务器发送任何数据，此连接将被强制关闭
    'heartbeat_check_interval' => 60,  // 表示每60秒遍历一次
);
```

  * **提示**

    * 启用`heartbeat_idle_time`后，服务器并不会主动向客户端发送数据包
    * 如果只设置了`heartbeat_idle_time`未设置`heartbeat_check_interval`底层将不会创建心跳检测线程，`PHP`代码中可以调用`heartbeat`方法手动处理超时的连接

### open_eof_check

?> **打开`EOF`检测**【默认值：`false`】，参考[TCP数据包边界问题](/learn?id=tcp数据包边界问题)

?> 此选项将检测客户端连接发来的数据，当数据包结尾是指定的字符串时才会投递给`Worker`进程。否则会一直拼接数据包，直到超过缓存区或者超时才会中止。当出错时底层会认为是恶意连接，丢弃数据并强制关闭连接。  
常见的`Memcache/SMTP/POP`等协议都是以`\r\n`结束的，就可以使用此配置。开启后可以保证`Worker`进程一次性总是收到一个或者多个完整的数据包。

```php
array(
    'open_eof_check' => true,   //打开EOF检测
    'package_eof'    => "\r\n", //设置EOF
)
```

  * **注意**

    !> 此配置仅对`STREAM`(流式的)类型的`Socket`有效，如 [TCP 、Unix Socket Stream](/server/methods?id=__construct)   
    `EOF`检测不会从数据中间查找`eof`字符串，所以`Worker`进程可能会同时收到多个数据包，需要在应用层代码中自行`explode("\r\n", $data)` 来拆分数据包

### open_eof_split

?> **启用`EOF`自动分包**

?> 当设置`open_eof_check`后，可能会产生多条数据合并在一个包内 , `open_eof_split`参数可以解决这个问题，参考[TCP数据包边界问题](/learn?id=tcp数据包边界问题)。

?> 设置此参数需要遍历整个数据包的内容，查找`EOF`，因此会消耗大量`CPU`资源。假设每个数据包为`2M`，每秒`10000`个请求，这可能会产生`20G`条`CPU`字符匹配指令。

```php
array(
    'open_eof_split' => true,   //打开EOF_SPLIT检测
    'package_eof'    => "\r\n", //设置EOF
)
```

  * **提示**

    * 启用`open_eof_split`参数后，底层会从数据包中间查找`EOF`，并拆分数据包。[onReceive](/server/events?id=onreceive)每次仅收到一个以`EOF`字串结尾的数据包。
    * 启用`open_eof_split`参数后，无论参数`open_eof_check`是否设置，`open_eof_split`都将生效。

    * **与 `open_eof_check` 的差异**
    
        * `open_eof_check` 只检查接收数据的末尾是否为 `EOF`，因此它的性能最好，几乎没有消耗
        * `open_eof_check` 无法解决多个数据包合并的问题，比如同时发送两条带有 `EOF` 的数据，底层可能会一次全部返回
        * `open_eof_split` 会从左到右对数据进行逐字节对比，查找数据中的 `EOF` 进行分包，性能较差。但是每次只会返回一个数据包

### package_eof

?> **设置`EOF`字符串。** 参考[TCP数据包边界问题](/learn?id=tcp数据包边界问题)

?> 需要与 `open_eof_check` 或者 `open_eof_split` 配合使用。

  * **注意**

    !> `package_eof`最大只允许传入`8`个字节的字符串

### open_length_check

?> **打开包长检测特性**【默认值：`false`】，参考[TCP数据包边界问题](/learn?id=tcp数据包边界问题)

?> 包长检测提供了固定包头+包体这种格式协议的解析。启用后，可以保证`Worker`进程[onReceive](/server/events?id=onreceive)每次都会收到一个完整的数据包。  
长度检测协议，只需要计算一次长度，数据处理仅进行指针偏移，性能非常高，**推荐使用**。

  * **提示**

    * **长度协议提供了3个选项来控制协议细节。**

      ?> 此配置仅对`STREAM`类型的`Socket`有效，如[TCP、Unix Socket Stream](/server/methods?id=__construct)

      * **package_length_type**

        ?> 包头中某个字段作为包长度的值，底层支持了10种长度类型。请参考 [package_length_type](/server/setting?id=package_length_type)

      * **package_body_offset**

        ?> 从第几个字节开始计算长度，一般有2种情况：

        * `length`的值包含了整个包（包头+包体），`package_body_offset` 为`0`
        * 包头长度为`N`字节，`length`的值不包含包头，仅包含包体，`package_body_offset`设置为`N`

      * **package_length_offset**

        ?> `length`长度值在包头的第几个字节。

        * 示例：

        ```c
        struct
        {
            uint32_t type;
            uint32_t uid;
            uint32_t length;
            uint32_t serid;
            char body[0];
        }
        ```
        
    ?> 以上通信协议的设计中，包头长度为`4`个整型，`16`字节，`length`长度值在第`3`个整型处。因此`package_length_offset`设置为`8`，`0-3`字节为`type`，`4-7`字节为`uid`，`8-11`字节为`length`，`12-15`字节为`serid`。

    ```php
    $server->set(array(
      'open_length_check'     => true,
      'package_max_length'    => 81920,
      'package_length_type'   => 'N',
      'package_length_offset' => 8,
      'package_body_offset'   => 16,
    ));
    ```

### package_length_type

?> **长度值的类型**，接受一个字符参数，与`PHP`的 [pack](http://php.net/manual/zh/function.pack.php) 函数一致。

目前`Swoole`支持`10`种类型：

字符参数 | 作用
---|---
c | 有符号、1字节
C | 无符号、1字节
s | 有符号、主机字节序、2字节
S | 无符号、主机字节序、2字节
n | 无符号、网络字节序、2字节
N | 无符号、网络字节序、4字节
l | 有符号、主机字节序、4字节（小写L）
L | 无符号、主机字节序、4字节（大写L）
v | 无符号、小端字节序、2字节
V | 无符号、小端字节序、4字节

### package_length_func

?> **设置长度解析函数**

?> 支持`C++`或`PHP`的`2`种类型的函数。长度函数必须返回一个整数。

返回数 | 作用
---|---
返回0 | 长度数据不足，需要接收更多数据
返回-1 | 数据错误，底层会自动关闭连接
返回包长度值（包括包头和包体的总长度）| 底层会自动将包拼好后返回给回调函数

  * **提示**

    * **使用方法**

    ?> 实现原理是先读取一小部分数据，在这段数据内包含了一个长度值。然后将这个长度返回给底层。然后由底层完成剩余数据的接收并组合成一个包进行`dispatch`。

    * **PHP长度解析函数**

    ?> 由于`ZendVM`不支持运行在多线程环境，因此底层会自动使用`Mutex`互斥锁对`PHP`长度函数进行加锁，避免并发执行`PHP`函数。在`1.9.3`或更高版本可用。

    !> 请勿在长度解析函数中执行阻塞`IO`操作，可能导致所有[Reactor](/learn?id=reactor线程)线程发生阻塞

    ```php
    $server = new Swoole\Server("127.0.0.1", 9501);
    
    $server->set(array(
        'open_length_check'   => true,
        'dispatch_mode'       => 1,
        'package_length_func' => function ($data) {
          if (strlen($data) < 8) {
              return 0;
          }
          $length = intval(trim(substr($data, 0, 8)));
          if ($length <= 0) {
              return -1;
          }
          return $length + 8;
        },
        'package_max_length'  => 2000000,  //协议最大长度
    ));
    
    $server->on('receive', function (Swoole\Server $server, $fd, $reactor_id, $data) {
        var_dump($data);
        echo "#{$server->worker_id}>> received length=" . strlen($data) . "\n";
    });
    
    $server->start();
    ```

    * **C++长度解析函数**

    ?> 在其他PHP扩展中，使用`swoole_add_function`注册长度函数到`Swoole`引擎中。
    
    !> C++长度函数调用时底层不会加锁，需要调用方自行保证线程安全性
    
    ```c++
    #include <string>
    #include <iostream>
    #include "swoole.h"
    
    using namespace std;
    
    int test_get_length(swProtocol *protocol, swConnection *conn, char *data, uint32_t length);
    
    void register_length_function(void)
    {
        swoole_add_function((char *) "test_get_length", (void *) test_get_length);
        return SW_OK;
    }
    
    int test_get_length(swProtocol *protocol, swConnection *conn, char *data, uint32_t length)
    {
        printf("cpp, size=%d\n", length);
        return 100;
    }
    ```

### package_max_length

?> **设置最大数据包尺寸，单位为字节。**【默认值：`2M` 即 `2 * 1024 * 1024`，最小值为`64K`】

?> 开启[open_length_check](/server/setting?id=open_length_check)/[open_eof_check](/server/setting?id=open_eof_check)/[open_eof_split](/server/setting?id=open_eof_split)/[open_http_protocol](/server/setting?id=open_http_protocol)/[open_http2_protocol](/http_server?id=open_http2_protocol)/[open_websocket_protocol](/server/setting?id=open_websocket_protocol)/[open_mqtt_protocol](/server/setting?id=open_mqtt_protocol)等协议解析后，`Swoole`底层会进行数据包拼接，这时在数据包未收取完整时，所有数据都是保存在内存中的。  
所以需要设定`package_max_length`，一个数据包最大允许占用的内存尺寸。如果同时有1万个`TCP`连接在发送数据，每个数据包`2M`，那么最极限的情况下，就会占用`20G`的内存空间。

  * **提示**

    * `open_length_check`：当发现包长度超过`package_max_length`，将直接丢弃此数据，并关闭连接，不会占用任何内存；
    * `open_eof_check`：因为无法事先得知数据包长度，所以收到的数据还是会保存到内存中，持续增长。当发现内存占用已超过`package_max_length`时，将直接丢弃此数据，并关闭连接；
    * `open_http_protocol`：`GET`请求最大允许`8K`，而且无法修改配置。`POST`请求会检测`Content-Length`，如果`Content-Length`超过`package_max_length`，将直接丢弃此数据，发送`http 400`错误，并关闭连接；

  * **注意**

    !> 此参数不宜设置过大，否则会占用很大的内存

### open_http_protocol

?> **启用`HTTP`协议处理。**【默认值：`false`】

?> 启用`HTTP`协议处理，[Swoole\Http\Server](/http_server)会自动启用此选项。设置为`false`表示关闭`HTTP`协议处理。

### open_mqtt_protocol

?> **启用`MQTT`协议处理。**【默认值：`false`】

?> 启用后会解析`MQTT`包头，`worker`进程[onReceive](/server/events?id=onreceive)每次会返回一个完整的`MQTT`数据包。

```php
$server->set(array(
  'open_mqtt_protocol' => true
));
```

### open_redis_protocol

?> **启用`Redis`协议处理。**【默认值：`false`】

?> 启用后会解析`Redis`协议，`worker`进程[onReceive](/server/events?id=onreceive)每次会返回一个完整的`Redis`数据包。建议直接使用[Redis\Server](/redis_server)

```php
$server->set(array(
  'open_redis_protocol' => true
));
```

### open_websocket_protocol

?> **启用`WebSocket`协议处理。**【默认值：`false`】

?> 启用`WebSocket`协议处理，[Swoole\WebSocket\Server](websocket_server)会自动启用此选项。设置为`false`表示关闭`websocket`协议处理。  
设置`open_websocket_protocol`选项为`true`后，会自动设置`open_http_protocol`协议也为`true`。

### open_websocket_close_frame

?> **启用websocket协议中关闭帧。**【默认值：`false`】

?> （`opcode`为`0x08`的帧）在`onMessage`回调中接收

?> 开启后，可在`WebSocketServer`中的`onMessage`回调中接收到客户端或服务端发送的关闭帧，开发者可自行对其进行处理。

```php
$server = new Swoole\WebSocket\Server("0.0.0.0", 9501);

$server->set(array("open_websocket_close_frame" => true));

$server->on('open', function (Swoole\WebSocket\Server $server, $request) {});

$server->on('message', function (Swoole\WebSocket\Server $server, $frame) {
    if ($frame->opcode == 0x08) {
        echo "Close frame received: Code {$frame->code} Reason {$frame->reason}\n";
    } else {
        echo "Message received: {$frame->data}\n";
    }
});

$server->on('close', function ($server, $fd) {});

$server->start();
```

### open_tcp_nodelay

?> **启用`open_tcp_nodelay`。**【默认值：`false`】

?> 开启后`TCP`连接发送数据时会关闭`Nagle`合并算法，立即发往对端TCP连接。在某些场景下，如命令行终端，敲一个命令就需要立马发到服务器，可以提升响应速度，请自行Google Nagle算法。

### open_cpu_affinity 

?> **启用CPU亲和性设置。** 【默认 `false`】

?> 在多核的硬件平台中，启用此特性会将`Swoole`的`reactor线程`/`worker进程`绑定到固定的一个核上。可以避免进程/线程的运行时在多个核之间互相切换，提高`CPU` `Cache`的命中率。

  * **提示**

    * **使用taskset命令查看进程的CPU亲和设置：**

    ```bash
    taskset -p 进程ID
    pid 24666's current affinity mask: f
    pid 24901's current affinity mask: 8
    ```

    > mask是一个掩码数字，按`bit`计算每`bit`对应一个`CPU`核，如果某一位为`0`表示绑定此核，进程会被调度到此`CPU`上，为`0`表示进程不会被调度到此`CPU`。示例中`pid`为`24666`的进程`mask = f` 表示未绑定到`CPU`，操作系统会将此进程调度到任意一个`CPU`核上。 `pid`为`24901`的进程`mask = 8`，`8`转为二进制是 `1000`，表示此进程绑定在第`4`个`CPU`核上。

### cpu_affinity_ignore

?> **IO密集型程序中，所有网络中断都是用CPU0来处理，如果网络IO很重，CPU0负载过高会导致网络中断无法及时处理，那网络收发包的能力就会下降。**

?> 如果不设置此选项，swoole将会使用全部CPU核，底层根据reactor_id或worker_id与CPU核数取模来设置CPU绑定。  
如果内核与网卡有多队列特性，网络中断会分布到多核，可以缓解网络中断的压力

```php
array('cpu_affinity_ignore' => array(0, 1)) // 接受一个数组作为参数，array(0, 1) 表示不使用CPU0,CPU1，专门空出来处理网络中断。
```

  * **提示**

    * **查看网络中断**

```shell
[~]$ cat /proc/interrupts 
           CPU0       CPU1       CPU2       CPU3       
  0: 1383283707          0          0          0    IO-APIC-edge  timer
  1:          3          0          0          0    IO-APIC-edge  i8042
  3:         11          0          0          0    IO-APIC-edge  serial
  8:          1          0          0          0    IO-APIC-edge  rtc
  9:          0          0          0          0   IO-APIC-level  acpi
 12:          4          0          0          0    IO-APIC-edge  i8042
 14:         25          0          0          0    IO-APIC-edge  ide0
 82:         85          0          0          0   IO-APIC-level  uhci_hcd:usb5
 90:         96          0          0          0   IO-APIC-level  uhci_hcd:usb6
114:    1067499          0          0          0       PCI-MSI-X  cciss0
130:   96508322          0          0          0         PCI-MSI  eth0
138:     384295          0          0          0         PCI-MSI  eth1
169:          0          0          0          0   IO-APIC-level  ehci_hcd:usb1, uhci_hcd:usb2
177:          0          0          0          0   IO-APIC-level  uhci_hcd:usb3
185:          0          0          0          0   IO-APIC-level  uhci_hcd:usb4
NMI:      11370       6399       6845       6300 
LOC: 1383174675 1383278112 1383174810 1383277705 
ERR:          0
MIS:          0
```

`eth0/eth1`就是网络中断的次数，如果`CPU0 - CPU3` 是平均分布的，证明网卡有多队列特性。如果全部集中于某一个核，说明网络中断全部由此`CPU`进行处理，一旦此`CPU`超过`100%`，系统将无法处理网络请求。这时就需要使用 `cpu_affinity_ignore` 设置将此`CPU`空出，专门用于处理网络中断。

如图上的情况，应当设置 `cpu_affinity_ignore => array(0)`

?> 可以使用`top`指令 `->` 输入 `1`，查看到每个核的使用率

  * **注意**

    !> 此选项必须与`open_cpu_affinity`同时设置才会生效

### tcp_defer_accept

?> **启用`tcp_defer_accept`特性**【默认值：`false`】

?> 可以设置为一个数值，表示当一个`TCP`连接有数据发送时才触发`accept`。

```php
$server->set(array(
  'tcp_defer_accept' => 5
));
```

  * **提示**

    * **启用`tcp_defer_accept`特性后，`accept`和[onConnect](/server/events?id=onconnect)对应的时间会发生变化。如果设置为`5`秒：**

      * 客户端连接到服务器后不会立即触发`accept`
      * 在`5`秒内客户端发送数据，此时会同时顺序触发`accept/onConnect/onReceive`
      * 在`5`秒内客户端没有发送任何数据，此时会触发`accept/onConnect`

### ssl_cert_file / ssl_key_file :id=ssl_cert_file

?> **设置SSL隧道加密。**

?> 设置值为一个文件名字符串，指定cert证书和key私钥的路径。

  * **提示**

    * **`PEM`转`DER`格式**

    ```shell
    openssl x509 -in cert.crt -outform der -out cert.der
    ```

    * **`DER`转`PEM`格式**

    ```shell
    openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
    ```

  * **注意**

    !> -`HTTPS`应用浏览器必须信任证书才能浏览网页；  
    -`wss`应用中，发起`WebSocket`连接的页面必须使用 `HTTPS` ；  
    -浏览器不信任`SSL`证书将无法使用 `wss` ；  
    -文件必须为`PEM`格式，不支持`DER`格式，可使用`openssl`工具进行转换。

    !> 使用`SSL`必须在编译`Swoole`时加入[--enable-openssl](/environment?id=编译选项)选项

    ```php
    $server = new Swoole\Server('0.0.0.0', 9501, SWOOLE_PROCESS, SWOOLE_SOCK_TCP | SWOOLE_SSL);
    $server->set(array(
        'ssl_cert_file' => __DIR__.'/config/ssl.crt',
        'ssl_key_file' => __DIR__.'/config/ssl.key',
    ));
    ```

### ssl_method

!> 此参数已在 [v4.5.4](/version/bc?id=_454) 版本移除，请使用`ssl_protocols`

?> **设置OpenSSL隧道加密的算法。**【默认值：`SWOOLE_SSLv23_METHOD`】，支持的类型请参考[SSL 加密方法](/consts?id=ssl-加密方法)

?> `Server`与`Client`使用的算法必须一致，否则`SSL/TLS`握手会失败，连接会被切断

```php
$server->set(array(
    'ssl_method' => SWOOLE_SSLv3_CLIENT_METHOD,
));
```

### ssl_protocols

?> **设置OpenSSL隧道加密的协议。**【默认值：`0`，支持全部协议】，支持的类型请参考[SSL 协议](/consts?id=ssl-协议)

!> Swoole版本 >= `v4.5.4` 可用

```php
$server->set(array(
    'ssl_protocols' => 0,
));
```

### ssl_sni_certs

?> **设置 SNI (Server Name Identification) 证书**

!> Swoole版本 >= `v4.6.0` 可用

```php
$server->set([
    'ssl_cert_file' => __DIR__ . '/server.crt',
    'ssl_key_file' => __DIR__ . '/server.key',
    'ssl_protocols' => SWOOLE_SSL_TLSv1_2 | SWOOLE_SSL_TLSv1_3 | SWOOLE_SSL_TLSv1_1 | SWOOLE_SSL_SSLv2,
    'ssl_sni_certs' => [
        'cs.php.net' => [
            'ssl_cert_file' => __DIR__ . '/sni_server_cs_cert.pem',
            'ssl_key_file' => __DIR__ . '/sni_server_cs_key.pem',
        ],
        'uk.php.net' => [
            'ssl_cert_file' =>  __DIR__ . '/sni_server_uk_cert.pem',
            'ssl_key_file' => __DIR__ . '/sni_server_uk_key.pem',
        ],
        'us.php.net' => [
            'ssl_cert_file' => __DIR__ . '/sni_server_us_cert.pem',
            'ssl_key_file' => __DIR__ . '/sni_server_us_key.pem',
        ],
    ]
]);
```

### ssl_ciphers

?> **设置 openssl 加密算法。**【默认值：`EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH`】

```php
$server->set(array(
    'ssl_ciphers' => 'ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP',
));
```

  * **提示**

    * `ssl_ciphers` 设置为空字符串时，由`openssl`自行选择加密算法

### ssl_verify_peer

?> **服务SSL设置验证对端证书。**【默认值：`false`】

?> 默认关闭，即不验证客户端证书。若开启，必须同时设置 `ssl_client_cert_file` 选项

### ssl_allow_self_signed

?> **允许自签名证书。**【默认值：`false`】

### ssl_client_cert_file

?> **根证书，用于验证客户端证书。**

```php
$server = new Swoole\Server('0.0.0.0', 9501, SWOOLE_PROCESS, SWOOLE_SOCK_TCP | SWOOLE_SSL);
$server->set(array(
    'ssl_cert_file'         => __DIR__ . '/config/ssl.crt',
    'ssl_key_file'          => __DIR__ . '/config/ssl.key',
    'ssl_verify_peer'       => true,
    'ssl_allow_self_signed' => true,
    'ssl_client_cert_file'  => __DIR__ . '/config/ca.crt',
));
```

!> `TCP`服务若验证失败，会底层会主动关闭连接。

### ssl_compress

?> **设置是否启用`SSL/TLS`压缩。** 在[Co\Client](/coroutine_client/client)使用时，它有一个别名`ssl_disable_compression`

### ssl_verify_depth

?> **如果证书链条层次太深，超过了本选项的设定值，则终止验证。**

### ssl_prefer_server_ciphers

?> **启用服务器端保护, 防止 BEAST 攻击。**

### ssl_dhparam

?> **指定DHE密码器的`Diffie-Hellman`参数。**

### ssl_ecdh_curve

?> **指定用在ECDH密钥交换中的`curve`。**

```php
$server = new Swoole\Server('0.0.0.0', 9501, SWOOLE_PROCESS, SWOOLE_SOCK_TCP | SWOOLE_SSL);
$server->set([
    'ssl_compress'                => true,
    'ssl_verify_depth'            => 10,
    'ssl_prefer_server_ciphers'   => true,
    'ssl_dhparam'                 => '',
    'ssl_ecdh_curve'              => '',
]);
```

### user

?> **设置`Worker/TaskWorker`子进程的所属用户。**【默认值：执行脚本用户】

?> 服务器如果需要监听`1024`以下的端口，必须有`root`权限。但程序运行在`root`用户下，代码中一旦有漏洞，攻击者就可以以`root`的方式执行远程指令，风险很大。配置了`user`项之后，可以让主进程运行在`root`权限下，子进程运行在普通用户权限下。

```php
$server->set(array(
  'user' => 'Apache'
));
```

  * **注意**

    !> -仅在使用`root`用户启动时有效  
    -使用`user/group`配置项将工作进程设置为普通用户后，将无法在工作进程调用`shutdown`/[reload](/server/methods?id=reload)方法关闭或重启服务。只能使用`root`账户在`shell`终端执行`kill`命令。

### group

?> **设置`Worker/TaskWorker`子进程的进程用户组。**【默认值：执行脚本用户组】

?> 与`user`配置相同，此配置是修改进程所属用户组，提升服务器程序的安全性。

```php
$server->set(array(
  'group' => 'www-data'
));
```

  * **注意**

    !> 仅在使用`root`用户启动时有效

### chroot

?> **重定向`Worker`进程的文件系统根目录。**

?> 此设置可以使进程对文件系统的读写与实际的操作系统文件系统隔离。提升安全性。

```php
$server->set(array(
  'chroot' => '/data/server/'
));
```

### pid_file

?> **设置 pid 文件地址。**

?> 在`Server`启动时自动将`master`进程的`PID`写入到文件，在`Server`关闭时自动删除`PID`文件。

```php
$server->set(array(
    'pid_file' => __DIR__.'/server.pid',
));
```

  * **注意**

    !> 使用时需要注意如果`Server`非正常结束，`PID`文件不会删除，需要使用[Swoole\Process::kill($pid, 0)](/process/process?id=kill)来侦测进程是否真的存在

### buffer_input_size / input_buffer_size :id=buffer_input_size

?> **配置接收输入缓存区内存尺寸。**【默认值：`2M`】

```php
$server->set([
    'buffer_input_size' => 2 * 1024 * 1024,
]);
```

### buffer_output_size / output_buffer_size :id=buffer_output_size

?> **配置发送输出缓存区内存尺寸。**【默认值：`2M`】

```php
$server->set([
    'buffer_output_size' => 32 * 1024 * 1024, //必须为数字
]);
```

  * **提示**

    !> Swoole 版本 >= `v4.6.7` 时，默认值为无符号INT最大值`UINT_MAX`

    * 单位为字节，默认为`2M`，如设置`32 * 1024 * 1024`表示，单次`Server->send`最大允许发送`32M`字节的数据
    * 调用`Server->send`，`Http\Server->end/write`，`WebSocket\Server->push`等发送数据指令时，`单次`最大发送的数据不得超过`buffer_output_size`配置。

    !> 此参数只针对[SWOOLE_PROCESS](/learn?id=swoole_process)模式生效，因为PROCESS模式下Worker进程的数据要发送给主进程再发送给客户端，所以每个Worker进程会和主进程开辟一块缓冲区。[参考](/learn?id=reactor线程)

### socket_buffer_size

?> **配置客户端连接的缓存区长度。**【默认值：`2M`】

?> 不同于 `buffer_output_size`，`buffer_output_size` 是 worker 进程`单次`send 的大小限制，`socket_buffer_size`是用于设置`Worker`和`Master`进程间通讯 buffer 总的大小，参考[SWOOLE_PROCESS](/learn?id=swoole_process)模式。

```php
$server->set([
    'socket_buffer_size' => 128 * 1024 *1024, //必须为数字，单位为字节，如128 * 1024 *1024表示每个TCP客户端连接最大允许有128M待发送的数据
]);
```

- **数据发送缓存区**

    - Master 进程向客户端发送大量数据时，并不能立即发出。这时发送的数据会存放在服务器端的内存缓存区内。此参数可以调整内存缓存区的大小。
    
    - 如果发送数据过多，数据占满缓存区后`Server`会报如下错误信息：
    
    ```bash
    swFactoryProcess_finish: send failed, session#1 output buffer has been overflowed.
    ```
    
    ?>发送缓冲区塞满导致`send`失败，只会影响当前的客户端，其他客户端不受影响
    服务器有大量`TCP`连接时，最差的情况下将会占用`serv->max_connection * socket_buffer_size`字节的内存
    
    - 尤其是外往通信的服务器程序，网络通信较慢，如果持续连续发送数据，缓冲区很快就会塞满。发送的数据会全部堆积在`Server`的内存里。因此此类应用应当从设计上考虑到网络的传输能力，先将消息存入磁盘，等客户端通知服务器已接受完毕后，再发送新的数据。
    
    - 如视频直播服务，`A`用户带宽是 `100M`，`1`秒内发送`10M`的数据是完全可以的。`B`用户带宽只有`1M`，如果`1`秒内发送`10M`的数据，`B`用户可能需要`100`秒才能接收完毕。这时数据会全部堆积在服务器内存中。
    
    - 可以根据数据内容的类型，进行不同的处理。如果是可丢弃的内容，如视频直播等业务，网络差的情况下丢弃一些数据帧完全可以接受。如果内容是不可丢失的，如微信消息，可以先存储到服务器的磁盘中，按照`100`条消息为一组。当用户接受完这一组消息后，再从磁盘中取出下一组消息发送到客户端。

### enable_unsafe_event

?> **启用`onConnect/onClose`事件。**【默认值：`false`】

?> `Swoole`在配置 [dispatch_mode](/server/setting?id=dispatch_mode)=1 或`3`后，因为系统无法保证`onConnect/onReceive/onClose`的顺序，默认关闭了`onConnect/onClose`事件；  
如果应用程序需要`onConnect/onClose`事件，并且能接受顺序问题可能带来的安全风险，可以通过设置`enable_unsafe_event`为`true`，启用`onConnect/onClose`事件。

### discard_timeout_request

?> **丢弃已关闭链接的数据请求。**【默认值：`true`】

?> `Swoole`在配置[dispatch_mode](/server/setting?id=dispatch_mode)=`1`或`3`后，系统无法保证`onConnect/onReceive/onClose`的顺序，因此可能会有一些请求数据在连接关闭后，才能到达`Worker`进程。

  * **提示**

    * `discard_timeout_request`配置默认为`true`，表示如果`worker`进程收到了已关闭连接的数据请求，将自动丢弃。
    * `discard_timeout_request`如果设置为`false`，表示无论连接是否关闭`Worker`进程都会处理数据请求。

### enable_reuse_port

?> **设置端口重用。**【默认值：`false`】

?> 启用端口重用后，可以重复启动监听同一个端口的 Server 程序

  * **提示**

    * `enable_reuse_port = true` 打开端口重用
    * `enable_reuse_port = false` 关闭端口重用

!> 仅在`Linux-3.9.0`以上版本的内核可用 `Swoole4.5`以上版本可用

### enable_delay_receive

?> **设置`accept`客户端连接后将不会自动加入[EventLoop](/learn?id=什么是eventloop)。**【默认值：`false`】

?> 设置此选项为`true`后，`accept`客户端连接后将不会自动加入[EventLoop](/learn?id=什么是eventloop)，仅触发[onConnect](/server/events?id=onconnect)回调。`worker`进程可以调用 [$server->confirm($fd)](/server/methods?id=confirm)对连接进行确认，此时才会将`fd`加入[EventLoop](/learn?id=什么是eventloop)开始进行数据收发，也可以调用`$server->close($fd)`关闭此连接。

```php
//开启enable_delay_receive选项
$server->set(array(
    'enable_delay_receive' => true,
));

$server->on("Connect", function ($server, $fd, $reactorId) {
    $server->after(2000, function() use ($server, $fd) {
        //确认连接，开始接收数据
        $server->confirm($fd);
    });
});
```

### reload_async

?> **设置异步重启开关。**【默认值：`true`】

?> 设置异步重启开关。设置为`true`时，将启用异步安全重启特性，`Worker`进程会等待异步事件完成后再退出。详细信息请参见 [如何正确的重启服务](/question/use?id=swoole如何正确的重启服务)

?> `reload_async` 开启的主要目的是为了保证服务重载时，协程或异步任务能正常结束。 

```php
$server->set([
  'reload_async' => true
]);
```

  * **协程模式**

    * 在`4.x`版本中开启 [enable_coroutine](/server/setting?id=enable_coroutine)时，底层会额外增加一个协程数量的检测，当前无任何协程时进程才会退出，开启时即使`reload_async => false`也会强制打开`reload_async`。

### max_wait_time

?> **设置 `Worker` 进程收到停止服务通知后最大等待时间**【默认值：`3`】

?> 经常会碰到由于`worker`阻塞卡顿导致`worker`无法正常`reload`, 无法满足一些生产场景，例如发布代码热更新需要`reload`进程。所以，Swoole 加入了进程重启超时时间的选项。详细信息请参见 [如何正确的重启服务](/question/use?id=swoole如何正确的重启服务)

  * **提示**

    * **管理进程收到重启、关闭信号后或者达到`max_request`时，管理进程会重起该`worker`进程。分以下几个步骤：**

      * 底层会增加一个(`max_wait_time`)秒的定时器，触发定时器后，检查进程是否依然存在，如果是，会强制杀掉，重新拉一个进程。
      * 需要在`onWorkerStop`回调里面做收尾工作，需要在`max_wait_time`秒内做完收尾。
      * 依次向目标进程发送`SIGTERM`信号，杀掉进程。

  * **注意**

    !> `v4.4.x`以前默认为`30`秒

### tcp_fastopen

?> **开启TCP快速握手特性。**【默认值：`false`】

?> 此项特性，可以提升`TCP`短连接的响应速度，在客户端完成握手的第三步，发送`SYN`包时携带数据。

```php
$server->set([
  'tcp_fastopen' => true
]);
```

  * **提示**

    * 此参数可以设置到监听端口上，想深入理解的同学可以查看[google论文](http://conferences.sigcomm.org/co-next/2011/papers/1569470463.pdf)

### request_slowlog_file

?> **开启请求慢日志。** 从`v4.4.8`版本开始[已移除](https://github.com/swoole/swoole-src/commit/b1a400f6cb2fba25efd2bd5142f403d0ae303366)

!> 由于这个慢日志的方案只能在同步阻塞的进程里面生效，不能在协程环境用，而Swoole4默认就是开启协程的，除非关闭`enable_coroutine`，所以不要使用了，使用 [Swoole Tracker](https://business.swoole.com/tracker/index) 的阻塞检测工具。

?> 启用后`Manager`进程会设置一个时钟信号，定时侦测所有`Task`和`Worker`进程，一旦进程阻塞导致请求超过规定的时间，将自动打印进程的`PHP`函数调用栈。

?> 底层基于`ptrace`系统调用实现，某些系统可能关闭了`ptrace`，无法跟踪慢请求。请确认`kernel.yama.ptrace_scope`内核参数是否`0`。

```php
$server->set([
  'request_slowlog_file' => '/tmp/trace.log',
]);
```

  * **超时时间**

```php
$server->set([
    'request_slowlog_timeout' => 2, // 设置请求超时时间为2秒
    'request_slowlog_file' => '/tmp/trace.log',
]);
```

!> 必须是具有可写权限的文件，否则创建文件失败底层会抛出致命错误
    
### enable_coroutine

?> **是否启用异步风格服务器的协程支持**

?> `enable_coroutine` 关闭时在[事件回调函数](/server/events)中不再自动创建协程，如果不需要用协程关闭这个会提高一些性能。参考[什么是Swoole协程](/coroutine)。

  * **配置方法**
    
    * 在`php.ini`配置 `swoole.enable_coroutine = 'Off'` (可见 [ini配置文档](/other/config.md) )
    * `$server->set(['enable_coroutine' => false]);`优先级高于ini

  * **`enable_coroutine`选项影响范围**

      * onWorkerStart
      * onConnect
      * onOpen
      * onReceive
      * [setHandler](/redis_server?id=sethandler)
      * onPacket
      * onRequest
      * onMessage
      * onPipeMessage
      * onFinish
      * onClose
      * tick/after 定时器

!> 开启`enable_coroutine`后在上述回调函数会自动创建协程

* 当`enable_coroutine`设置为`true`时，底层自动在[onRequest](/http_server?id=on)回调中创建协程，开发者无需自行使用`go`函数[创建协程](/coroutine/coroutine?id=create)
* 当`enable_coroutine`设置为`false`时，底层不会自动创建协程，开发者如果要使用协程，必须使用`go`自行创建协程，如果不需要使用协程特性，则处理方式与`Swoole1.x`是100%一致的
* 注意，这个开启只是说明Swoole会通过协程去处理请求，如果事件中含有阻塞函数，那需要提前配置`hook_flags`或者开启[一键协程化](/runtime)，将`sleep`，`mysqlnd`这些阻塞的函数或者扩展开启协程化

```php
$server = new Swoole\Http\Server("127.0.0.1", 9501);

$server->set([
    //关闭内置协程
    'enable_coroutine' => false,
]);

$server->on("request", function ($request, $response) {
    if ($request->server['request_uri'] == '/coro') {
        go(function () use ($response) {
            co::sleep(0.2);
            $response->header("Content-Type", "text/plain");
            $response->end("Hello World\n");
        });
    } else {
        $response->header("Content-Type", "text/plain");
        $response->end("Hello World\n");
    }
});

$server->start();
```

### hook_flags

?> **设置`一键协程化`Hook的函数范围。**【默认值：不hook】

!> Swoole版本为 `v4.5+` 或 [4.4LTS](https://github.com/swoole/swoole-src/tree/v4.4.x) 可用，详情参考[一键协程化](/runtime)

```php
$server->set([
    'hook_flags' => SWOOLE_HOOK_SLEEP,
]);
```
底层支持以下协程化项，可使用`SWOOLE_HOOK_ALL`表示协程化全部：

* `SWOOLE_HOOK_TCP`
* `SWOOLE_HOOK_UNIX`
* `SWOOLE_HOOK_UDP`
* `SWOOLE_HOOK_UDG`
* `SWOOLE_HOOK_SSL`
* `SWOOLE_HOOK_TLS`
* `SWOOLE_HOOK_SLEEP`
* `SWOOLE_HOOK_FILE`
* `SWOOLE_HOOK_STREAM_FUNCTION`
* `SWOOLE_HOOK_BLOCKING_FUNCTION`
* `SWOOLE_HOOK_PROC`
* `SWOOLE_HOOK_CURL`
* `SWOOLE_HOOK_NATIVE_CURL`
* `SWOOLE_HOOK_SOCKETS`
* `SWOOLE_HOOK_STDIO`
* `SWOOLE_HOOK_PDO_PGSQL`
* `SWOOLE_HOOK_PDO_ODBC`
* `SWOOLE_HOOK_PDO_ORACLE`
* `SWOOLE_HOOK_PDO_SQLITE`
* `SWOOLE_HOOK_ALL`

### send_yield

?> **当发送数据时缓冲区内存不足时，直接在当前协程内[yield](/coroutine?id=协程调度)，等待数据发送完成，缓存区清空时，自动[resume](/coroutine?id=协程调度)当前协程，继续`send`数据。**【默认值：在[dispatch_mod](/server/setting?id=dispatch_mode) 2/4时候可用，并默认开启】

* `Server/Client->send`返回`false`并且错误码为`SW_ERROR_OUTPUT_BUFFER_OVERFLOW`时，不返回`false`到`PHP`层，而是[yield](/coroutine?id=协程调度)挂起当前协程
* `Server/Client`监听缓冲区是否清空的事件，在该事件触发后，缓存区内的数据已被发送完毕，这时[resume](/coroutine?id=协程调度)对应的协程
* 协程恢复后，继续调用`Server/Client->send`向缓存区内写入数据，这时因为缓存区已空，发送必然是成功的

改进前

```php
for ($i = 0; $i < 100; $i++) {
    //在缓存区塞满时会直接返回 false，并报错 output buffer overflow
    $server->send($fd, $data_2m);
}
```

改进后

```php
for ($i = 0; $i < 100; $i++) {
    //在缓存区塞满时会 yield 当前协程，发送完成后 resume 继续向下执行
    $server->send($fd, $data_2m);
}
```

!> 此项特性会改变底层的默认行为，可以手动关闭

```php
$server->set([
    'send_yield' => false,
]);
```

  * __影响范围__

    * [Swoole\Server::send](/server/methods?id=send)
    * [Swoole\Http\Response::write](/http_server?id=write)
    * [Swoole\WebSocket\Server::push](/websocket_server?id=push)
    * [Swoole\Coroutine\Client::send](/coroutine_client/client?id=send)
    * [Swoole\Coroutine\Http\Client::push](/coroutine_client/http_client?id=push)

### send_timeout

设置发送超时，与`send_yield`配合使用，当在规定的时间内，数据未能发送到缓存区，底层返回`false`，并设置错误码为`ETIMEDOUT`，可以使用 [getLastError()](/server/methods?id=getlasterror) 方法获取错误码。

> 类型为浮点型，单位为秒，最小粒度为毫秒

```php
$server->set([
    'send_yield' => true,
    'send_timeout' => 1.5, // 1.5秒
]);

for ($i = 0; $i < 100; $i++) {
    if ($server->send($fd, $data_2m) === false and $server->getLastError() == SOCKET_ETIMEDOUT) {
      echo "发送超时\n";
    }
}
```

### hook_flags

?> **设置`一键协程化`Hook的函数范围。**【默认值：不hook】

!> Swoole版本为 `v4.5+` 或 [4.4LTS](https://github.com/swoole/swoole-src/tree/v4.4.x) 可用，详情参考[一键协程化](/runtime)

```php
$server->set([
    'hook_flags' => SWOOLE_HOOK_SLEEP,
]);
```

### buffer_high_watermark

?> **设置缓存区高水位线，单位为字节。**

```php
$server->set([
    'buffer_high_watermark' => 8 * 1024 * 1024,
]);
```

### buffer_low_watermark

?> **设置缓存区低水位线，单位为字节。**

```php
$server->set([
    'buffer_low_watermark' => 1 * 1024 * 1024,
]);
```

### tcp_user_timeout

?> TCP_USER_TIMEOUT选项是TCP层的socket选项，值为数据包被发送后未接收到ACK确认的最大时长，以毫秒为单位。具体请查看man文档

```php
$server->set([
    'tcp_user_timeout' => 10 * 1000, // 10秒
]);
```

!> Swoole版本 >= `v4.5.3-alpha` 可用

### stats_file

?> **指定[stats()](/server/methods?id=stats)内容写入的文件路径。设置后会自动在[onWorkerStart](/server/events?id=onworkerstart)时设置一个定时器，定时将[stats()](/server/methods?id=stats)的内容写入指定文件中**

```php
$server->set([
    'stats_file' => __DIR__ . '/stats.log',
]);
```

!> Swoole版本 >= `v4.5.5` 可用

### event_object

?> **设置此选项后，事件回调将使用[对象风格](/server/events?id=回调对象)。**【默认值：`false`】

```php
$server->set([
    'event_object' => true,
]);
```

!> Swoole版本 >= `v4.6.0` 可用

### start_session_id

?> **设置起始 session ID**

```php
$server->set([
    'start_session_id' => 10,
]);
```

!> Swoole版本 >= `v4.6.0` 可用

### single_thread

?> **设置为单一线程。** 启用后 Reactor 线程将会和 Master 进程中的 Master 线程合并，由 Master 线程处理逻辑，在PHP ZTS下，如果使用`SWOOLE_PROCESS`模式，一定要设置该值为`true`。

```php
$server->set([
    'single_thread' => true,
]);
```

!> Swoole版本 >= `v4.2.13` 可用

### max_queued_bytes

?> **设置接收缓冲区的最大队列长度。** 如果超出，则停止接收。

```php
$server->set([
    'max_queued_bytes' => 1024 * 1024,
]);
```

!> Swoole版本 >= `v4.5.0` 可用

### admin_server

?> **设置admin_server服务，用于在 [Swoole Dashboard](http://dashboard.swoole.com/) 中查看服务信息等。**

```php
$server->set([
    'admin_server' => '0.0.0.0:9502',
]);
```

!> Swoole版本 >= `v4.8.0` 可用

### bootstrap

?> **多线程模式下的程序入口文件，默认是当前执行的脚本文件名。**

!> Swoole版本 >= `v6.0` ， `PHP`为`ZTS`模式，编译`Swoole`时开启了`--enable-swoole-thread`可用

```php
$server->set([
    'bootstrap' => __FILE__,
]);
```

### init_arguments

?> **设置多线程的数据共享数据，该配置需要一个回调函数，服务器启动时会自动执行该函数**

!> Swoole内置了许多线程安全容器，[并发Map](/thread/map)，[并发List](/thread/arraylist)，[并发队列](/thread/queue)，不要在函数中返回不安全的变量。

!> Swoole版本 >= `v6.0` ， `PHP`为`ZTS`模式，编译`Swoole`时开启了`--enable-swoole-thread`可用

```php
$server->set([
    'init_arguments' => function() { return new Swoole\Thread\Map(); },
]);

$server->on('request', function($request, $response) {
    $map = Swoole\Thread::getArguments();
});
```
