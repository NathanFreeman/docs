# 多端口监听

`Swoole`支持单服务多端口监听，每个端口可独立配置协议（如 HTTP、TCP、WebSocket）与 SSL/TLS 加密，实现协议隔离与安全通信，无需启动多个服务进程。

## 监听新端口

- `Swoole`通过[Swoole\Server->listen()](/server/methods?id=listen)方法监听多个端口。该方法返回[Swoole\Server\Port](/server/swoole_server_port) 对象。这意味着你可以同时处理 HTTP、TCP、UDP 等不同协议，实现高效的端口复用。

* **示例**

```php
<?php
use Swoole\Server;

$server = new Server('127.0.0.1', 9501);

// 监听 9502 端口，纯 TCP 协议
$port1 = $server->listen("127.0.0.1", 9502, SWOOLE_SOCK_TCP);

// 监听 9503 端口，纯 TCP 协议
$port2 = $server->listen("127.0.0.1", 9503, SWOOLE_SOCK_TCP);

// 监听 9504 端口，纯 TCP 协议
$port3 = $server->listen("127.0.0.1", 9504, SWOOLE_SOCK_TCP);

$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$server->start();
```

## 协议配置和事件回调

- 当你通过[Swoole\Server->listen()](/server/methods?id=listen)添加一个新端口时，它就像一个“克隆体”，默认会继承主端口的协议配置（如 `HTTP`、`WebSocket`）和事件回调。假设主端口配置为`HTTP`服务时，新监听的端口若未显式配置，将直接继承主端口的`HTTP`协议设置，自动以HTTP服务模式运行。

- 如果你想让新端口处理不同的协议（例如主端口是 `HTTP`，新端口是 `TCP`），你必须显式地调用 `Swoole\Server\Port->set()` 和 `Swoole\Server\Port->on()` 来修改协议配置，并重新绑定事件。

* **示例一：TCP 端口配置覆盖，主端口 9501 是 TCP，子端口 9502 也是 TCP，但处理逻辑和包长规则不同。**

```php
<?php
use Swoole\Server;

$server = new Server('127.0.0.1', 9501);

// 1. 主端口配置：假设用于处理某种定长包
$server->set([
    'open_length_check' => true,
    'package_length_type' => 'C',
    'package_length_offset' => 100, // 假设第100字节是包长度的值
    'package_max_length' => 1000,  // 发送到9501端口的数据一定不会超过1000字节
]);

// 主端口接收事件
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
    echo "9501 端口收到数据\n";
});

// 2. 监听子端口 9502
$port = $server->listen("127.0.0.1", 9502, SWOOLE_SOCK_TCP);

// 【关键】必须重写配置，否则 9502 也会使用上面的 package_length_type=C，package_length_offset=100和package_max_length=1000
$port->set([
    'open_length_check' => true,
    'package_length_type' => 'N',
    'package_length_offset' => 200, // 假设第200字节是包长度的值
    'package_max_length' => 800000, // 发送到9502端口的数据一定不会超过800000字节
]);

// 【关键】必须为子端口单独绑定事件，否则还是触发主端口的receive事件
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
    echo "9502 端口收到数据，使用不同的解析规则\n";
});

$server->start();
```


* **示例二：主端口 9501 是 HTTP 服务，想加一个 9502 端口专门做 TCP 透传。**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$server = new Server('127.0.0.1', 9501);

// 主端口 HTTP 逻辑
$server->on('request', function(Request $request, Response $response) {
    $response->end("Hello HTTP");
});

// 监听子端口 9502
$port = $server->listen('127.0.0.1', 9502, SWOOLE_SOCK_TCP);

// 【关键步骤 1】重置协议配置
// 如果不写这一步，9502 端口收到的数据会被当成 HTTP 请求解析，导致失败
$port->set([
    'open_http_protocol' => false, // 关闭 HTTP 协议
    // 这里可以添加 TCP 相关的配置，如 open_length_check 等
]);

// 【关键步骤 2】在 $port 对象上绑定 receive 事件
// 注意：不能写在 $server->on('receive') 里，因为主服务器是 HTTP 模式，不支持 receive
$port->on('receive', function($server, $fd, $reactorId, $data) {
    $server->send($fd, "TCP Port 9502 received: $data");
});

$server->start();
```

- `Swoole\Server\Port->set()`可以设置的协议有：

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


- `Swoole\Server\Port->on()`可以监听的事件有：


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

## 注意

!> `Swoole\Http\Server` 是通过继承 `Swoole\Server` 实现的，`Swoole\WebSocket\Server` 是通过继承 `Swoole\Http\Server`。因此，如果你创建了一个普通的 `TCP` 服务器，无法通过 `Swoole\Server->listen()` 方法给它添加 `HTTP` 或 `WebSocket` 子端口。 

```php
// HTTP服务器的request事件
function (Swoole\Http\Request $request, Swoole\Http\Reponse $response) {}

// Websocket服务器的message事件
function (Swoole\WebSocket\Server $server,  Swoole\WebSocket\Frame $frame) {}
```

- 从上述两个函数签名可以看出，如果通过 `Swoole\Server` 在主服务器之外的新端口监听 `WebSocket` 协议，当触发 [message](/server/events?id=message) 事件时，回调函数期望接收一个 `Swoole\WebSocket\Server` 类型的对象。然而，此时的主服务器类型为 `Swoole\Server`，实际传入的却是 `Swoole\Server` 对象，导致参数类型不匹配而报错。

  虽然当前 [request](/server/events?id=request) 事件不需要传入 `$server` 参数，但考虑到未来 API 的扩展性，后续版本可能会为 `Swoole\Http\Server` 引入专属方法，届时同样会出现类似的类型不一致问题。

  因此，**禁止**由父类（`Swoole\Server`）在新端口监听一个子类服务器（如 `Swoole\WebSocket\Server` 或 `Swoole\Http\Server`），推荐改为由子类在新端口监听父类服务器，以确保回调参数类型始终正确。

* **错误示例**

```php
<?php
use Swoole\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\WebSocket\Server as WebsocketServer;
use Swoole\WebSocket\Frame;


// 创建一个 TCP 服务器
$server = new Server('127.0.0.1', 9501);  // 主端口监听TCP
$server->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {
  echo "Hello TCP!!!";
});

// 尝试在子端口 9502 监听 HTTP 协议 —— 无效
$port1 = $server->listen("127.0.0.1", 9502, SWOOLE_SOCK_TCP);
$port1->set([
  'open_http_protocol' => true,
  'package_max_length' => 800000,
]);
$port1->on('request', function(Request $request, Response $reponse) {
  echo "Hello HTTP!!!";
});

// 尝试在子端口 9503 监听 Websocket 协议 —— 无效
$port2 = $server->listen("127.0.0.1", 9503, SWOOLE_SOCK_TCP);
$port2->set([
  'open_websocket_protocol' => true,
  'package_max_length' => 800000,
]);
$port2->on('message', function (WebsocketServer $server,  Frame $frame) {
  echo "Hello Websocket!!!";
});

$server->start();
```

* **正确做法**

- 场景：你的主要功能是 `HTTP/WebSocket`，但还想提供一个简单的 TCP 管理接口。

- 解决方案：先创建 `HTTP 或 WebSocket` 服务器，再通过 `Swoole\Server->listen()` 添加 `TCP` 子端口。

```php
<?php
use Swoole\Server;
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
// 先起启动一个 HTTP 服务器
$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $response->header("Content-Type", "text/html; charset=utf-8");
  $response->end("<h1>Hello Swoole. #".rand(1000, 9999)."</h1>");
});

// 监听 TCP 服务器
$port = $http->listen('127.0.0.1', 9502, SWOOLE_TCP);
// 重置从HTTP服务器继承过来的协议配置
$port->set(['open_http_protocol' => false]); 
// 重新设置事件监听
$port->on('receive', function(Server $server, int $fd, int $reactorId, string $data) {

});

$http->start();
```

| 主服务器类型 | 能否添加 HTTP 子端口 | 能否添加 WebSocket 子端口 | 能否添加 TCP 子端口 |
|------------|-------------------|-------------------------|-------------------|
| TCP        | ❌ 不行 | ❌ 不行 | ✅ 可以 |
| HTTP       | ✅ 可以 | ❌ 不行 | ✅ 可以 |
| WebSocket  | ✅ 可以 | ✅ 可以 | ✅ 可以 |



