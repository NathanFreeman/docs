# WebSocket服务器

- 我们可以通过实例化 `Swoole\WebSocket\Server` 来创建一个 WebSocket 服务器。`Swoole\WebSocket\Server`、`Swoole\WebSocket\Frame` 和 `Swoole\WebSocket\CloseFrame` 三者共同构成了完整的 WebSocket 服务器体系。其中，[Frame对象](/server/swoole_websocket_frame)用于封装客户端发送的数据帧，而 [CloseFrame对象](/server/swoole_websocket_closeframe)则代表关闭帧，用于处理连接关闭的逻辑。以下是一个简单示例，展示了它们的基本用法。

> `Swoole\WebSocket\Server` 是 `Swoole\Http\Server` 的子类。

#### 示例

- 示例1：创建一个简单的 WebSocket 服务器

- 在服务器上监听 `127.0.0.1` 的 `9501` 端口，实现一个基本的 WebSocket 服务器：

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;

$server = new Server('127.0.0.1', 9501);
$server->on('message', function(Server $server, Frame $frame) {
  $response = new Frame();
  $response->data = $frame->data;
  $response->opcode = WEBSOCKET_OPCODE_TEXT;
  $response->finish = true;
  
  $server->push($frame->fd, $response);
});

$server->start();
```

- 测试服务器打开另一个终端，使用 `uwsc` 发送 WebSocket 请求：

```bash
uwsc ws://127.0.0.1:9501/
```

- 输入Hello Swoole，回车后我们会看到终端会输出：

```
Server message: 'Hello Swoole'
```

---

- 示例2：创建一个简单的 WebSocket 服务器，发送连续帧

- 在服务器上监听 `127.0.0.1` 的 `9501` 端口，向客户端发送连续帧：

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;

$server = new Server('127.0.0.1', 9501);
$server->on('message', function(Server $server, Frame $frame) {
  $response = new Frame();
  
  for ($i = 0; $i < 100; $i++) {
    $response->data = base64_encode(random_bytes(200));
    if ($i == 0) {
      $response->opcode = SWOOLE_WEBSOCKET_OPCODE_TEXT;
    } else {
      $response->opcode = SWOOLE_WEBSOCKET_OPCODE_CONTINUATION;
    }
    
    $response->finish = $i == 99;
    $server->push($frame->fd, $response);
  }
});

$server->start();
```

- 测试服务器打开另一个终端，使用 `uwsc` 发送 WebSocket 请求：

```bash
uwsc ws://127.0.0.1:9501/
```

- 随便输入一些信息，回车后我们会看到终端会输出：

```
Server message: 大量的随机信息
```

---


- 示例3：使用 WebSocket 服务器监听 HTTP 请求

- 在服务器上监听 `127.0.0.1:9501` 启动 WebSocket 服务时，由于 `Swoole\WebSocket\Server` 继承自 `Swoole\Http\Server`，因此该服务能够同时处理 WebSocket 连接和普通 HTTP 请求。

```php
<?php
use Swoole\WebSocket\Server;
use Swoole\WebSocket\Frame;
use Swoole\Http\Request;
use Swoole\Http\Response;

$server = new Server('127.0.0.1', 9501);
$server->on('message', function(Server $server, Frame $frame) {
  $server->push($frame->fd, $frame->data);
});

$server->on('request', function(Request $request, Response $reponse) {
  $reponse->end("<h1>Hello World</h1>");
});

$server->start();
```

- 测试服务器打开另一个终端，使用 `curl` 发送 HTTP 请求：

```bash
curl http://127.0.0.1:9501/
```

- 我们会看到终端会输出：

```
<h1>Hello Swoole</h1>
```
