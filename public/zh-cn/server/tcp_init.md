# TCP/UDP 服务器

你可以通过实例化 `Swoole\Server` 对象来创建 TCP 或 UDP 服务器。`Swoole\Server` 是所有异步风格服务器的基类，后续章节介绍的 `Swoole\Http\Server`、`Swoole\WebSocket\Server`、`Swoole\Redis\Server` 等类都继承自它。

#### 示例

- 示例1：创建一个简单的 TCP 服务器

- 在服务器上监听 `127.0.0.1` 的 `9501` 端口，实现一个基本的 TCP 服务器：

```php
<?php
use Swoole\Server;

$server = new Server('127.0.0.1', 9501);

$server->on('receive', function (Server $server, int $fd, int $reactorId, string $data) {
    $server->send($fd, 'Hello World');
});

$server->start();
```

- 测试服务器打开另一个终端，使用 `telnet` 连接服务器：

```bash
telnet 127.0.0.1 9501
```

- 连接成功后，输入任意字符并回车，服务器将返回响应内容：

```
Hello World
```

- 示例2：创建一个简单的 UDP 服务器

- 在服务器上监听 `127.0.0.1` 的 `9502` 端口，实现一个基本的 UDP 服务器：

```php
<?php
use Swoole\Server;

$server = new Server('127.0.0.1', 9502, SWOOLE_BASE, SWOOLE_SOCK_UDP);

$server->on('packet', function (Server $server, mixed $data, array $clientInfo) {
    $server->sendto($clientInfo['address'], $clientInfo['port'], "Hello World");
});

$server->start();
```

- 测试服务器打开另一个终端，使用 `nc` 连接服务器：

```bash
echo "Hello" | nc -u 127.0.0.1 9502
```

- 连接成功后，输入任意字符并回车，服务器将返回响应内容：

```
Hello World
```
