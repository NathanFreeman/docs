# Swoole\Server\Port

- [Swoole\Server->listen()](/server/methods?id=listen)方法用于为服务器添加多个监听端口，每个端口对应一个 `Swoole\Server\Port` 对象。本文档将对 `Swoole\Server\Port` 类进行全面讲解。

!> `Swoole\Server\Port`对象只能通过 [Swoole\Server->listen()](/server/methods?id=listen) 方法获得，不能直接实例化。单独 `new Swoole\Server\Port()` 没有任何实际意义，这种对象无法正常工作。

```php
$port = new Port();  // ❌ 无效，无法使用
```


## 属性

### host

- 返回当前端口监听的主机地址的`host`，该属性是一个`string`类型的字符串。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_TCP);
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) use ($port) {
  var_dump($port->host); // 输出127.0.0.1
});

$server->start();
```

### port

- 返回当前端口监听的端口的`port`，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_TCP);
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) use ($port) {
  var_dump($port->port); // 输出9502
});

$server->start();
```

### type

- 返回当前服务器的 `Socket` 类型，该属性是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_SOCK_TCP);
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) use ($port) {
  var_dump($port->type); // 输出 SWOOLE_SOCK_TCP
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
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_TCP | SWOOLE_SSL);
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) use ($port) {
  var_dump($port->ssl); // 输出true
});

$server->start();
```



### setting

- 通过 `Swoole\Server\Port->set()` 方法配置的所有参数，最终都会被保存在 `Swoole\Server\Port->$setting` 属性中。该属性是一个数组（`array`），允许开发者在回调函数中随时访问和读取当前的运行时配置。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_TCP);
$port->set([
    'open_length_check' => true,
    'package_length_type' => 'N',
    'package_length_offset' => 0,
    'package_max_length' => 800000,
]);
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) use ($port) {
  var_dump($port->setting);
});

$server->start();
```

### connections

- 这是一个由连接的文件描述符（fd）组成的数组迭代器。你可以使用 foreach 遍历当前端口所有的连接客户端连接。

* **示例**

```php
<?php
use Swoole\Server;
$server = new Server('127.0.0.1', 9501);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_TCP);

$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) use ($port) {
  foreach($port->connections as $fd) {
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

## 方法

### on

- 为当前监听端口添加回调事件。

```php
Swoole\Server\Port->on(string $name, callable $callback): bool {}
```

* **参数**

  * **`string $name`**
    * **功能**：事件名。
    * **默认值**：无
    * **其它值**：无

  * **`callable $callback`**
    * **功能**：回调函数。
    * **默认值**：无
    * **其它值**：无

* **返回值**

  * 执行成功返回`true`，否则返回`false`。


---

* **示例**

```php
<?php
use Swoole\Server;

$server = new Server('127.0.0.1', 9501, SWOOLE_BASE, SWOOLE_SOCK_TCP);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_SOCK_TCP);
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});


$server->start();
```

- 可以监听的事件有：

  | 事件                                                                   | 说明                       |
  |----------------------------------------------------------------------|--------------------------|
  | [connect](/server/events?id=connect)                                 | 客户端连接建立                  |
  | [close](/server/events?id=close)                                     | 客户端连接关闭                  |
  | [disconnect](/server/events?id=disconnect)                           | 连接断开（通常用于长连接）            |
  | [receive](/server/events?id=receive)                                 | 接收数据流                    |
  | [packet](/server/events?id=packet)                                   | 接收 UDP 数据包               |
  | [message](/server/events?id=message)                                 | 接收消息（通常用于 WebSocket）     |
  | [request](/server/events?id=request)                                 | HTTP 请求                  |
  | [handShake](/server/events?id=handShake)                             | 握手事件                     |
  | [beforeHandshakeResponse](/server/events?id=beforeHandshakeResponse) | 握手响应前                    |
  | [open](/server/events?id=open)                                       | 连接开启（通常指 WebSocket 握手成功） |


---

!> 服务启动之后，不能再添加事件。底层会抛出`can't register event callback function after server started`。

!> 添加不存在的事件，底层会抛出`unknown event types[%s]`。

### set

- 对当前监听端口设置协议相关的配置。

```php
Swoole\Server\Port->set(array $setting): bool {}
```

* **参数**

  * **`array $setting`**
    * **功能**：协议相关的配置。
    * **默认值**：无
    * **其它值**：无

* **返回值**

  * 执行成功返回`true`，否则返回`false`。

---

* **示例**

```php
use Swoole\Server;

$server = new Server('127.0.0.1', 9501, SWOOLE_BASE, SWOOLE_SOCK_TCP);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$port = $server->listen('127.0.0.1', 9502, SWOOLE_SOCK_TCP);
$port->set([
  'open_length_check' => true,
  'package_length_type' => 'N',
  'package_length_offset' => 200,
  'package_max_length' => 800000,
]);
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});


$server->start();
```

---

- 可以设置的配置有

| 配置项                                                                         | 说明                       |
|-----------------------------------------------------------------------------|--------------------------|
| [backlog](/server/setting?id=backlog)                                       | 监听队列长度                   |
| [socket_buffer_size](/server/setting?id=socket_buffer_size)                 | 配置客户端连接的缓存区长度            |
| [heartbeat_idle_time](/server/setting?id=heartbeat_idle_time)               | 连接最大允许空闲的时间              |
| [buffer_high_watermark](/server/setting?id=buffer_high_watermark)           | 缓存区高水位线                  |
| [buffer_low_watermark](/server/setting?id=buffer_low_watermark)             | 缓存区低水位线                  |
| [max_idle_time](/server/setting?id=max_idle_time)                           | 最大空闲时间                   |
| [open_tcp_nodelay](/server/setting?id=open_tcp_nodelay)                     | 启用 TCP_NODELAY           |
| [tcp_defer_accept](/server/setting?id=tcp_defer_accept)                     | 启用 TCP_DEFAT_ACCEPT      |
| [open_tcp_keepalive](/server/setting?id=open_tcp_keepalive)                 | 启用 TCP keepalive         |
| [tcp_keepidle](/server/setting?id=tcp_keepidle)                             | keepalive 空闲探测时间         |
| [tcp_keepinterval](/server/setting?id=tcp_keepinterval)                     | keepalive 探测间隔           |
| [tcp_keepcount](/server/setting?id=tcp_keepcount)                           | keepalive 探测次数           |
| [tcp_user_timeout](/server/setting?id=tcp_user_timeout)                     | TCP 数据在确认对方未响应时，等待的最大时间（毫秒） |
| [tcp_fastopen](/server/setting?id=tcp_fastopen)                             | 开启 TCP 快速握手特性            |
| [open_eof_check](/server/setting?id=open_eof_check)                         | 开启 EOF 检测                |
| [open_eof_split](/server/setting?id=open_eof_split)                         | 开启 EOF 自动分包              |
| [package_eof](/server/setting?id=package_eof)                               | 设置 EOF 字符串               |
| [open_length_check](/server/setting?id=open_length_check)                   | 开启长度检测                   |
| [package_length_type](/server/setting?id=package_length_type)               | 长度值的类型，接受一个字符参数          |
| [package_length_offset](/server/setting?id=package_length_offset)           | 包的长度值在包头的第几个字节           |
| [package_body_offset](/server/setting?id=package_body_offset)               | 从第几个字节开始包体计算长度           |
| [package_length_func](/server/setting?id=package_length_func)               | 设置包长度计算函数                |
| [package_max_length](/server/setting?id=package_max_length)                 | 设置最大数据包尺寸，单位为字节                  |
| [open_http_protocol](/server/setting?id=open_http_protocol)                 | 开启 HTTP 协议               |
| [open_websocket_protocol](/server/setting?id=open_websocket_protocol)       | 开启 WebSocket 协议          |
| [open_http2_protocol](/server/setting?id=open_http2_protocol)               | 开启 HTTP2 协议              |
| [open_mqtt_protocol](/server/setting?id=open_mqtt_protocol)                 | 开启 MQTT 协议               |
| [open_redis_protocol](/server/setting?id=open_redis_protocol)               | 开启 Redis 协议              |
| [ssl_compress](/server/setting?id=ssl_compress)                             | 设置是否启用 SSL/TLS 压缩                 |
| [ssl_protocols](/server/setting?id=ssl_protocols)                           | 设置 OpenSSL 隧道加密的协议               |
| [ssl_verify_peer](/server/setting?id=ssl_verify_peer)                       | 服务 SSL 设置验证对端证书                  |
| [ssl_allow_self_signed](/server/setting?id=ssl_allow_self_signed)           | 允许自签名证书                  |
| [ssl_client_cert_file](/server/setting?id=ssl_client_cert_file)             | 根证书，用于验证客户端证书                 |
| [ssl_cafile](/server/setting?id=ssl_cafile)                                 | CA 证书文件                  |
| [ssl_capath](/server/setting?id=ssl_capath)                                 | CA 证书目录                  |
| [ssl_verify_depth](/server/setting?id=ssl_verify_depth)                     | 如果证书链条层次太深，超过了本选项的设定值，则终止验证|
| [ssl_prefer_server_ciphers](/server/setting?id=ssl_prefer_server_ciphers)   | 启用服务器端保护，防止 BEAST 攻击|
| [ssl_ciphers](/server/setting?id=ssl_ciphers)                               | 设置 openssl 加密算法。                   |
| [ssl_ecdh_curve](/server/setting?id=ssl_ecdh_curve)                         | 指定用在 ECDH 密钥交换中的 curve                 |
| [ssl_dhparam](/server/setting?id=ssl_dhparam)                               | 指定 DHE 密码器的 Diffie-Hellman 参数                 |
| [ssl_sni_certs](/server/setting?id=ssl_sni_certs)                           | 设置 SNI (Server Name Identification) 证书|

### getCallback

- 获取当前端口监听的回调事件。

```php
Swoole\Server\Port->getCallback(string $name): \Closure|false {}
```

* **参数**

  * **`string $name`**
    * **功能**：事件名字，该参数不区分大小写
    * **默认值**：无
    * **其它值**：无

* **返回值**

  * 执行成功返回一个回调函数，否则返回`false`。

---

* **示例**

```php
<?php
use Swoole\Server;

$server = new Server('0.0.0.0', 9501, SWOOLE_BASE, SWOOLE_SOCK_TCP);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

$port = $server->listen('0.0.0.0', 9502, SWOOLE_SOCK_UDP);
$port->on('packet', function(Server $server, mixed $data, array $clientInfo) {
  echo $data . PHP_EOL;
});

var_dump($port->getCallback('packet'));

$server->start();
```

### getSocket

- 当前端口监听的服务端文件描述符（fd）导出为 `PHP` 底层的 `Socket` 对象，便于使用 `PHP` 的 `sockets` 扩展进行更底层的操作。

```php
Swoole\Server\Port->getSocket(): \Socket|false {}
```

---

* **示例：为 UDP 端口加入组播组**

```php
<?php
use Swoole\Server;

$server = new Server('0.0.0.0', 9501, SWOOLE_BASE, SWOOLE_SOCK_TCP);
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo $data . PHP_EOL;
});

// 监听一个 UDP 端口，用于接收组播数据
$port = $server->listen('0.0.0.0', 9502, SWOOLE_SOCK_UDP);

// 获取这个 UDP 端口的底层 Socket 对象，目的是为了使用 PHP 原生函数进行 Swoole 未封装的高级设置
$socket = $port->getSocket();
// 使用原生 Socket 扩展，让服务器加入一个“组播组”，就像收音机调频到了 224.10.20.30 这个频道
socket_set_option(
    $socket,
    IPPROTO_IP,
    MCAST_JOIN_GROUP,
    array(
        'group' => '224.10.20.30', // 表示组播地址
        'interface' => 'eth0' // 网卡名称
    )
);
    
$port->on('packet', function(Server $server, mixed $data, array $clientInfo) {
  echo "收到组播数据: " . $data . PHP_EOL;
});

$server->start();
```

* **测试命令**

```shell
# 执行下列的命令就可以看到输出了
echo "Hello Multicast" | socat - UDP-DATAGRAM:224.10.20.30:9502
```


---


!> 此方法需要依赖 `PHP` 的 `sockets` 扩展，并且编译 `Swoole` 时需要开启 `--enable-sockets` 选项。
